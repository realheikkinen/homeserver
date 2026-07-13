# Velero Setup — Kubernetes-native Backups gegen MinIO

## Ziel

Velero sichert Cluster-State (Kubernetes-Manifeste) und PV-Daten aller Namespaces gegen den
`velero`-Bucket auf MinIO (`nas-01`). Das ist Ebene 2 der in `backup_strategy.md` beschriebenen
Backup-Strategie (Ebene 1: k3s-Snapshots, Ebene 3: IaC im Git-Repo).

## Architektur-Entscheidung: dedizierter MinIO-User statt Root-Zugangsdaten

Statt der MinIO-Root-Zugangsdaten (`minioadmin`) bekommt Velero einen eigenen MinIO-User
(`velero`) mit einer Policy, die **nur** Zugriff auf den `velero`-Bucket erlaubt
(`s3:GetObject`, `PutObject`, `DeleteObject`, `ListBucket` etc., beschränkt auf
`arn:aws:s3:::velero` und `arn:aws:s3:::velero/*`). Geringeres Blast-Radius, falls das
Velero-Secret im Cluster jemals kompromittiert wird.

> ✅ **Status (2026-07-13):** User `velero` angelegt, Policy `velero-backup-policy` zugewiesen,
> auf `nas-01` verifiziert (`mc admin user info nas velero`).

---

## 1. MinIO-User und Policy anlegen (auf `nas-01`)

```bash
export VELERO_SECRET=$(openssl rand -base64 24 | tr -d '/+=')
mc admin user add nas velero "$VELERO_SECRET"

cat > /tmp/velero-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:ListBucketMultipartUploads",
        "s3:ListMultipartUploadParts",
        "s3:AbortMultipartUpload"
      ],
      "Resource": [
        "arn:aws:s3:::velero",
        "arn:aws:s3:::velero/*"
      ]
    }
  ]
}
EOF

mc admin policy create nas velero-backup-policy /tmp/velero-policy.json
mc admin policy attach nas velero-backup-policy --user=velero
rm /tmp/velero-policy.json
```

> Bei älteren `mc`-Versionen heißt der letzte Schritt `mc admin policy set nas
> velero-backup-policy user=velero` statt `attach`.

Verifizieren:

```bash
mc admin user info nas velero
# AccessKey: velero
# Status: enabled
# PolicyName: velero-backup-policy
```

> ✅ **Status (2026-07-13):** erledigt, `$VELERO_SECRET` wurde ausschließlich lokal (nicht im
> Chat/Repo) in `./velero-credentials` eingetragen.

---

## 2. Velero CLI (lokal, mit Cluster-Zugriff)

```bash
# https://github.com/vmware-tanzu/velero/releases
velero version --client-only
```

> ✅ **Status (2026-07-13):** `velero` CLI v1.18.2 bereits vorhanden.

---

## 3. Credentials-Datei

`./velero-credentials` (nicht im Git-Repo, siehe `.gitignore`):

```ini
[default]
aws_access_key_id=velero
aws_secret_access_key=<VELERO_SECRET>
```

> **Falle:** Beim Anlegen per Heredoc kann der `EOF`-Terminator versehentlich mit in die Datei
> geschrieben werden (z.B. bei copy-paste über mehrere Terminal-Segmente) — Ergebnis ist eine
> zusätzliche Zeile `EOF` nach `aws_secret_access_key=...`. Das lässt `velero install` beim
> Parsen der AWS-Credentials-Datei scheitern. Vor der Verwendung prüfen, dass die Datei
> **genau 3 Zeilen** hat (`[default]`, `aws_access_key_id=...`, `aws_secret_access_key=...`),
> z.B. mit `wc -l` und `awk -F= '{print NR": "$1}'` (zeigt nur die Keys, nicht die Werte).

---

## 4. Installation im Cluster

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.10.0 \
  --bucket velero \
  --secret-file ./velero-credentials \
  --use-volume-snapshots=false \
  --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://nas-01:9000
```

`--use-volume-snapshots=false`, weil MinIO keine nativen Volume-Snapshots unterstützt — Velero
sichert PV-Daten stattdessen über den eingebauten Data-Mover (File System Backup) bzw. je nach
Backup-Konfiguration.

### Installation prüfen

```bash
kubectl get pods -n velero
kubectl get backupstoragelocation -n velero
```

Erwartung: Pod `1/1 Running`, BackupStorageLocation `default` mit `PHASE: Available` (nicht nur
angelegt — `Available` bedeutet, Velero hat erfolgreich gegen MinIO validiert).

> ✅ **Status (2026-07-13):** erledigt. Pod `velero-67b5d579b9-zthhj` `1/1 Running`,
> BackupStorageLocation `default` `Available`.

---

## 5. Backup testen

```bash
velero backup create test-backup --include-namespaces nextcloud
velero backup describe test-backup
```

Erwartung: `Phase: Completed`, `Items backed up` entspricht `Total items to be backed up`.

> ✅ **Status (2026-07-13):** erledigt. `test-backup` `Completed`, 39/39 Items gesichert
> (Namespace `nextcloud`, inkl. CloudNativePG-Cluster-CRD und Deployment). TTL 720h (30 Tage),
> läuft automatisch ab — kein manuelles Aufräumen nötig.

---

## 6. Node-Agent (File System Backup) — echte Volume-Daten sichern

**Wichtige Erkenntnis:** Das ursprüngliche Setup (Abschnitt 4) sicherte mit
`--use-volume-snapshots=false` zwar erfolgreich die Kubernetes-Manifeste, aber **keine echten
Volume-Inhalte** — bei einem Restore wären PVCs leer zurückgekommen (Nextcloud-Dateien,
Postgres-Datenbanken, Immich-/Jellyfin-Medien wären verloren gewesen). Grund: ohne
`--use-node-agent` fehlt Veleros dateibasiertes Backup (kopia-basiertes File System Backup,
läuft als DaemonSet auf jedem Node).

`velero install` ist idempotent nachrüstbar — derselbe Befehl wie in Abschnitt 4, nur um
`--use-node-agent` ergänzt, legt zusätzlich das fehlende `DaemonSet/node-agent` an, ohne
Bestehendes zu verändern:

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.10.0 \
  --bucket velero \
  --secret-file ./velero-credentials \
  --use-volume-snapshots=false \
  --use-node-agent \
  --backup-location-config region=minio,s3ForcePathStyle=true,s3Url=http://nas-01:9000
```

```bash
kubectl get daemonset -n velero
kubectl get pods -n velero -o wide
```

Erwartung: `node-agent` DaemonSet mit **2** Instanzen (nur auf den beiden ungetainteten Workern
`k8s-worker-01`/`k8s-worker-02` — der getaintete Master bekommt korrekterweise keinen Pod).

> ✅ **Status (2026-07-13):** erledigt, `node-agent` läuft `2/2 Ready` auf beiden Workern.

---

## 7. Backup-Schedule

```bash
velero schedule create daily-backup \
  --schedule="0 3 * * *" \
  --exclude-namespaces kube-system,kube-node-lease,kube-public,longhorn-system,velero,cnpg-system \
  --ttl 720h0m0s \
  --default-volumes-to-fs-backup
```

**Design-Entscheidungen:**
- **Exclude- statt Include-Namespaces:** neue Apps (nach dem Muster von Nextcloud/Immich/Jellyfin)
  werden automatisch mitgesichert, ohne den Schedule anzupassen. Ausgeschlossen sind reine
  System-/Infrastruktur-Namespaces ohne eigene Nutzdaten (`kube-system`, `kube-node-lease`,
  `kube-public`, `longhorn-system`, `velero` selbst, `cnpg-system` — der Operator-Namespace,
  nicht die App-DB-Cluster, die liegen in `nextcloud`/`immich`).
- **`0 3 * * *`** — täglich um 3 Uhr nachts, wenig Netzwerk-/Streaming-Last auf dem Cluster.
- **`--ttl 720h0m0s`** — 30 Tage Aufbewahrung (Standardwert, konsistent mit den Test-Backups).
- **`--default-volumes-to-fs-backup`** — aktiviert File System Backup (kopia, siehe Abschnitt 6)
  für alle Pod-Volumes, ohne dass jeder Pod einzeln per Annotation opt-in muss.

Verifiziert per manuellem Trigger nach Schedule-Template (`velero backup create
daily-backup-verify --from-schedule daily-backup`), danach wieder gelöscht (der reguläre
Schedule läuft ab jetzt eigenständig):

> ✅ **Status (2026-07-13):** Verifikations-Backup `Completed`, 121/121 Items. Kopia-Volume-Backups
> für alle App-Daten bestätigt — u.a. Nextcloud-Dateien (~834 MB), Nextcloud- und
> Immich-Postgres-Datenbanken (~528 MB / ~743 MB), Immich-Fotos/Thumbnails (NFS, ~200 MB / ~55 MB),
> **Jellyfin-Mediathek (NFS, ~6,1 GB)**. RAM-Headroom danach weiterhin komfortabel (Worker 31–42%).

```bash
velero schedule describe daily-backup
```

Nächster reguläres Backup: heute Nacht 3 Uhr (`velero get backups` bzw. `velero backup describe
<name>` danach zur Kontrolle).

---

## Nächste Schritte

- **Restore-Test** — bisher nur Backups verifiziert, kein Restore in einen Test-Namespace
  ausprobiert. Sollte vor "produktivem Vertrauen" in die Backups nachgeholt werden.
- Alte SQLite-Cronjobs auf `k8s-master-01` aufräumen (siehe `nas_setup.md` Abschnitt 8) —
  niedrige Priorität, unabhängig von Velero.
- Laut Roadmap (`nas_setup.md` Abschnitt 10) danach optional: PostgreSQL als k3s-Datastore
  (Abschnitt 6, weiterhin zurückgestellt).
