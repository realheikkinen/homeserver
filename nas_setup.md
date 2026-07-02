# NAS/Backup-Server Setup — HP ProDesk 600 G4

## Übersicht

Der HP ProDesk 600 G4 dient als NAS- und Backup-System außerhalb des k3s-Clusters.
Er ist kein Cluster-Node — das ist gewollt: fällt der Cluster aus, läuft das Backup-Ziel weiter.

### Rolle im Gesamtsystem

```
k3s Cluster (Dell Mini 5040s)
├── k3s Control Plane  →  PostgreSQL auf NAS  (Cluster-State, kein SQLite)
├── CloudNativePG Pod  →  Longhorn PVC        →  250GB SSD (Dell-Node, repliziert)
├── Nextcloud Pod      →  Longhorn PVC        →  250GB SSD (Dell-Node, repliziert)
├── Immich Server Pod  →  NFS PVC             →  NAS /data/fast/immich-thumbs
│                      →  NFS PVC             →  NAS /data/bulk/photos
└── Jellyfin Pod       →  NFS PVC             →  NAS /data/bulk/media

NAS (HP ProDesk 600 G4)
├── PostgreSQL         →  k3s-Datastore (Cluster-State, außerhalb des Clusters)
├── NFS Server         →  Medien, Thumbnails, große Daten für Cluster-Pods
├── MinIO              →  S3-Ziel für Velero (PV-Backups) + CloudNativePG-Backups
└── rsync/SSH          →  Backup-Ziel für pg_dump (k3s-Datastore)
```

App-Datenbanken (CloudNativePG) laufen im Cluster auf Longhorn — der NAS ist ihr **Backup-Ziel**, nicht ihr Host.
Der k3s-Datastore-PostgreSQL ist der einzige Dienst auf dem NAS, der direkt für den Cluster kritisch ist.

---

## Hardware

| Komponente | Detail |
|---|---|
| Maschine | HP ProDesk 600 G4 |
| RAM | 8GB DDR4 (ausbaubar bis 64GB — Upgrade empfohlen wenn Longhorn dazukommt) |
| Interne SSD | bereits verbaut → OS-Laufwerk |
| 1TB SATA SSD | neu → `/data/fast` (Random-I/O-intensiv: MinIO, Thumbnails) |
| 2TB SATA HDD | neu → `/data/bulk` (sequenziell: Medien, Originalfotos) |

### Warum nicht TrueNAS?

- TrueNAS lebt von ZFS-Redundanz (Mirror, RAIDZ) — mit einer einzelnen HDD kein Vorteil
- ZFS empfiehlt mind. 16GB RAM für sinnvollen ARC-Cache
- Debian 13 ist konsistent mit den Cluster-Nodes und deutlich leichtgewichtiger

---

## 1. Betriebssystem

Debian 13 auf der internen SSD installieren (Standard-Install, kein Desktop).

Nach der Installation:

```bash
sudo apt update && sudo apt upgrade -y
sudo hostnamectl set-hostname nas-01
```

In `/etc/hosts` auf **allen Cluster-Nodes** eintragen:

```
192.168.188.XXX  nas-01
```

SSH-Key vom eigenen Rechner verteilen:

```bash
ssh-copy-id user@nas-01
```

---

## 2. Festplatten einrichten

### Laufwerke identifizieren

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL
```

Die SATA-Laufwerke erscheinen typischerweise als `/dev/sda`, `/dev/sdb` etc.
Die interne SSD (OS) ist bereits gemountet — die beiden neuen Laufwerke sind es nicht.

### 1TB SSD formatieren und mounten

```bash
sudo mkfs.ext4 -L fast-data /dev/sdX   # X durch korrekten Device-Namen ersetzen
sudo mkdir -p /data/fast
```

In `/etc/fstab` eintragen:

```
LABEL=fast-data  /data/fast  ext4  defaults,noatime  0  2
```

### 2TB HDD formatieren und mounten

```bash
sudo mkfs.ext4 -L bulk-data /dev/sdY   # Y durch korrekten Device-Namen ersetzen
sudo mkdir -p /data/bulk
```

In `/etc/fstab` eintragen:

```
LABEL=bulk-data  /data/bulk  ext4  defaults,noatime  0  2
```

```bash
sudo mount -a
df -h | grep /data   # Prüfen ob beide gemountet sind
```

### Verzeichnisstruktur anlegen

```bash
sudo mkdir -p \
  /data/fast/backups \
  /data/fast/minio \
  /data/fast/immich-thumbs \
  /data/bulk/photos \
  /data/bulk/media \
  /data/bulk/k3s-pvc
```

---

## 3. NFS-Server

### Installation

```bash
sudo apt install nfs-kernel-server -y
```

### Exports konfigurieren

`/etc/exports`:

```
# Immich Thumbnails + ML-Cache (SSD — Random-I/O-intensiv)
/data/fast/immich-thumbs  192.168.188.0/24(rw,sync,no_subtree_check,no_root_squash)

# Immich Originalfotos/-videos (HDD — sequenziell, große Dateien)
/data/bulk/photos         192.168.188.0/24(rw,sync,no_subtree_check,no_root_squash)

# Jellyfin Medien (HDD — sequenzielles Streaming, reicht für 4K)
/data/bulk/media          192.168.188.0/24(rw,sync,no_subtree_check,no_root_squash)

# Allgemeine NFS StorageClass für den Cluster
/data/bulk/k3s-pvc        192.168.188.0/24(rw,sync,no_subtree_check,no_root_squash)
```

```bash
sudo exportfs -ra
sudo systemctl enable --now nfs-kernel-server
sudo showmount -e localhost   # Prüfen ob Exports aktiv sind
```

### NFS vom Cluster aus testen

Auf einem der Dell-Nodes:

```bash
sudo apt install nfs-common -y
sudo mount -t nfs nas-01:/data/bulk/media /mnt
ls /mnt
sudo umount /mnt
```

---

## 4. NFS-Provisioner im k3s-Cluster

Damit Pods NFS-Volumes per PVC anfordern können, statt sie manuell als PV anzulegen.

```bash
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

helm install nfs-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace kube-system \
  --set nfs.server=nas-01 \
  --set nfs.path=/data/bulk/k3s-pvc \
  --set storageClass.name=nfs-nas \
  --set storageClass.defaultClass=false \
  --set storageClass.reclaimPolicy=Retain
```

Danach stehen zwei StorageClasses zur Verfügung:

| StorageClass | Backend | Geeignet für |
|---|---|---|
| `local-path` | Lokale SSD auf dem Node | Temporäre Daten, Development |
| `longhorn` | Replizierte SSDs (Dell-Nodes) | Datenbanken (CloudNativePG), kritische App-Daten |
| `nfs-nas` | HDD auf dem NAS | Medien, große Dateien, unkritische Daten |

---

## 5. PostgreSQL als k3s-Datastore

k3s unterstützt nativ externe Datenbanken als Cluster-State-Speicher via `--datastore-endpoint`.
PostgreSQL auf dem NAS ersetzt SQLite auf dem Master — mit zwei entscheidenden Vorteilen:

- **Master-Node austauschbar:** fällt k8s-master-01 aus, liegt der Cluster-State noch auf dem NAS
- **Spätere HA möglich:** ein zweiter Server-Node zeigt einfach auf dieselbe PostgreSQL →
  kein etcd, kein Cluster-Reset nötig

### PostgreSQL installieren

```bash
sudo apt install postgresql -y
```

### Datenbank und User anlegen

```bash
sudo -u postgres psql <<EOF
CREATE USER k3s WITH PASSWORD 'sicheres-passwort';
CREATE DATABASE k3s OWNER k3s;
EOF
```

### Netzwerkzugriff erlauben

In `/etc/postgresql/*/main/postgresql.conf`:

```
listen_addresses = 'localhost,192.168.188.XXX'   # NAS-IP eintragen
```

In `/etc/postgresql/*/main/pg_hba.conf`:

```
# k3s-Cluster-Nodes dürfen auf die k3s-DB zugreifen
host  k3s  k3s  192.168.188.0/24  scram-sha-256
```

```bash
sudo systemctl restart postgresql
sudo systemctl enable postgresql
```

### Verbindung vom Master testen

```bash
# Auf k8s-master-01
psql "postgres://k3s:sicheres-passwort@nas-01:5432/k3s" -c "\conninfo"
```

### k3s auf dem Master neu installieren

Da eine direkte Migration von SQLite nach PostgreSQL nicht supported ist, wird k3s neu
aufgesetzt. Da alle Manifeste und Helm-Values im Git-Repo liegen, ist das der sauberere Weg.

```bash
# Auf k8s-master-01: k3s stoppen und deinstallieren
sudo systemctl stop k3s
/usr/local/bin/k3s-uninstall.sh

# Mit externem Datastore neu installieren
curl -sfL https://get.k3s.io | sh -s - server \
  --datastore-endpoint="postgres://k3s:sicheres-passwort@nas-01:5432/k3s" \
  --write-kubeconfig-mode 644 \
  --tls-san k8s-master-01 \
  --tls-san 192.168.188.XXX

# Worker neu joinen (Token hat sich geändert)
sudo cat /var/lib/rancher/k3s/server/node-token
```

Auf jedem Worker den neuen Token eintragen:

```bash
# Auf k8s-worker-01 und k8s-worker-02
sudo systemctl stop k3s-agent
/usr/local/bin/k3s-agent-uninstall.sh

curl -sfL https://get.k3s.io | K3S_URL=https://k8s-master-01:6443 K3S_TOKEN=<NEUER-TOKEN> sh -
```

### Backup der PostgreSQL (k3s-Datastore)

Täglicher `pg_dump` per Cronjob auf dem NAS — ersetzt die SQLite-VACUUM-Snapshots:

```bash
# Cronjob auf dem NAS (als postgres-User oder root)
crontab -e
```

```
# k3s-Datastore täglich sichern
0 3 * * * pg_dump postgres://k3s:sicheres-passwort@localhost:5432/k3s \
  | gzip > /data/fast/backups/k3s-pg/k3s-$(date +\%Y\%m\%d).sql.gz

# Dumps älter als 7 Tage löschen
0 4 * * * find /data/fast/backups/k3s-pg/ -name "*.sql.gz" -mtime +7 -delete
```

```bash
mkdir -p /data/fast/backups/k3s-pg
```

### Zweiten Server-Node hinzufügen (optional, später)

Wenn du einen Worker zum zweiten Control-Plane-Node promoten willst:

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --datastore-endpoint="postgres://k3s:sicheres-passwort@nas-01:5432/k3s" \
  --token <CLUSTER-TOKEN>
```

Kein etcd-Quorum, kein Cluster-Reset — beide Server-Nodes teilen dieselbe PostgreSQL.

---

## 6. MinIO (S3-Ziel für Velero und CloudNativePG-Backups)

### Installation

```bash
wget https://dl.min.io/server/minio/release/linux-amd64/minio \
  -O /usr/local/bin/minio
chmod +x /usr/local/bin/minio

useradd -r -s /sbin/nologin minio
chown minio:minio /data/fast/minio
```

### Konfiguration

`/etc/default/minio`:

```bash
MINIO_VOLUMES="/data/fast/minio"
MINIO_ROOT_USER="minioadmin"
MINIO_ROOT_PASSWORD="<sicheres-passwort-hier>"
MINIO_SITE_NAME="nas-minio"
```

### systemd-Service

`/etc/systemd/system/minio.service`:

```ini
[Unit]
Description=MinIO Object Storage
After=network-online.target
Wants=network-online.target

[Service]
User=minio
Group=minio
EnvironmentFile=/etc/default/minio
ExecStart=/usr/local/bin/minio server $MINIO_VOLUMES \
  --console-address ":9001"
Restart=always
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now minio
```

MinIO-Konsole erreichbar unter `http://nas-01:9001`.

### Buckets anlegen

Über die Web-Konsole oder per CLI:

```bash
# mc (MinIO Client) installieren
wget https://dl.min.io/client/mc/release/linux-amd64/mc -O /usr/local/bin/mc
chmod +x /usr/local/bin/mc

mc alias set nas http://localhost:9000 minioadmin <passwort>
mc mb nas/velero
mc mb nas/cnpg-backups
```

---

## 7. SSH/rsync-Ziel (Backup-Eingang vom Master)

Der NAS übernimmt die Rolle des rsync-Ziels vom Worker-01 (wo die Key-Auth fehlschlug).
Nach der Migration auf PostgreSQL entfällt der SQLite-Cronjob auf dem Master — der `backup`-User
bleibt aber nützlich für andere rsync-Transfers.

```bash
useradd -m -s /bin/bash backup
mkdir -p /home/backup/.ssh
chmod 700 /home/backup/.ssh
```

Den Public Key von `root@k8s-master-01` eintragen:

```bash
# Auf k8s-master-01 den Key anzeigen
sudo cat /root/.ssh/id_ed25519.pub

# Den Inhalt in authorized_keys auf dem NAS eintragen
echo "<public-key>" >> /home/backup/.ssh/authorized_keys
chmod 600 /home/backup/.ssh/authorized_keys
chown -R backup:backup /home/backup/.ssh
chmod 755 /home/backup
chown backup:backup /data/fast/backups
```

### Verbindung testen

```bash
# Auf k8s-master-01
ssh backup@nas-01 "echo OK"
```

### Crontab auf k8s-master-01 bereinigen

Nach der Migration auf PostgreSQL (Abschnitt 5) die SQLite-Cronjobs auf dem Master entfernen —
das Backup übernimmt jetzt `pg_dump` direkt auf dem NAS:

```bash
# Auf k8s-master-01: alte SQLite-Cronjobs entfernen
sudo crontab -e
# Zeilen mit sqlite3 und dem alten rsync löschen
```

---

## 8. Velero im k3s-Cluster einrichten

```bash
# Velero CLI installieren (auf eigenem Rechner oder Master)
# https://github.com/vmware-tanzu/velero/releases

velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.10.0 \
  --bucket velero \
  --secret-file ./velero-credentials \
  --use-volume-snapshots=false \
  --backup-location-config \
    region=minio,s3ForcePathStyle=true,s3Url=http://nas-01:9000
```

`./velero-credentials`:

```ini
[default]
aws_access_key_id=minioadmin
aws_secret_access_key=<passwort>
```

Backup testen:

```bash
velero backup create test-backup --include-namespaces nextcloud
velero backup describe test-backup
```

---

## 9. Nächste Schritte (nach NAS-Setup)

1. **PostgreSQL einrichten + k3s migrieren** (Abschnitt 5 — vor allem anderen)
2. **Longhorn deployen** (replizierter Block-Storage für App-Datenbanken)
3. **CloudNativePG** (PostgreSQL-Operator, nutzt Longhorn-Storage)
4. **Nextcloud neu deployen** (mit CloudNativePG + Longhorn PVC)
5. **Velero einrichten** (nach Abschnitt 8)
6. **Immich deployen** (Thumbnails auf NFS-SSD, Originale auf NFS-HDD)
7. **Jellyfin deployen** (NFS-PVC auf `/data/bulk/media`)

---

## Ressourcen im Blick behalten

```bash
# Speicherplatz auf dem NAS
df -h /data/fast /data/bulk

# NFS-Verbindungen
sudo showmount -a

# MinIO-Status
sudo systemctl status minio

# Backup-Log prüfen
tail -f /var/log/k3s-backup.log   # auf k8s-master-01
```
