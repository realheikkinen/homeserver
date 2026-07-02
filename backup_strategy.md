# Backup-Strategie Homeserver Cluster

## Übersicht

Backups für den k3s-Cluster müssen drei Ebenen abdecken:
1. **Cluster-State** — Kubernetes-Objekte (Deployments, Services, Secrets, ConfigMaps)
2. **Persistent Volumes** — Anwendungsdaten (Datenbanken, Configs, Dashboards)
3. **Cluster-Konfiguration** — Infrastructure-as-Code (Helm-Values, Manifeste, Playbooks)

---

## 1. Cluster-State (k3s-Snapshots)

k3s speichert den gesamten Cluster-State in einer eingebetteten Datenbank (SQLite oder etcd).
Snapshots liegen unter `/var/lib/rancher/k3s/server/db/snapshots/` auf `k8s-master-01`.

### Konfiguration

In `/etc/rancher/k3s/config.yaml`:

```yaml
etcd-snapshot-schedule-cron: "0 */6 * * *"
etcd-snapshot-retention: 5
```

### Sicherung nach extern

Die Snapshots liegen nur lokal auf dem Master — ein Cronjob sollte sie regelmäßig auf einen Worker oder externen Speicher kopieren:

```bash
# Beispiel: Cronjob auf k8s-master-01
0 1 * * * rsync -az /var/lib/rancher/k3s/server/db/snapshots/ user@k8s-worker-01:/backup/k3s-snapshots/
```

---

## 2. Persistent Volumes (Anwendungsdaten)

### Velero (empfohlen für Kubernetes-native Backups)

- Sichert Kubernetes-Ressourcen und PV-Snapshots zusammen
- Kann gegen S3-kompatibles Backend sichern (MinIO, Hetzner Storage Box, Backblaze B2)
- Restore auf Namespace- oder Ressourcen-Ebene möglich
- RAM-Overhead: ~200MB

### Anwendungsspezifische Dumps (ergänzend)

Für Datenbanken zusätzlich klassische Dumps — ein SQL-Dump ist zuverlässiger restaurierbar als ein Volume-Snapshot.

```yaml
# Beispiel: CronJob für PostgreSQL-Dump
apiVersion: batch/v1
kind: CronJob
metadata:
  name: pg-backup
spec:
  schedule: "0 2 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: pg-dump
            image: postgres:16
            command:
            - /bin/sh
            - -c
            - pg_dump -h $PGHOST -U $PGUSER $PGDATABASE | gzip > /backup/dump-$(date +%Y%m%d).sql.gz
            envFrom:
            - secretRef:
                name: pg-credentials
            volumeMounts:
            - name: backup-vol
              mountPath: /backup
          restartPolicy: OnFailure
          volumes:
          - name: backup-vol
            persistentVolumeClaim:
              claimName: backup-pvc
```

### Grafana-Dashboards

Dashboards als JSON in ConfigMaps pflegen (Dashboards-as-Code) oder per Grafana-API exportieren.
Wenn Dashboards im Git-Repo liegen, sind sie automatisch gesichert.

---

## 3. Cluster-Konfiguration (Infrastructure-as-Code)

Alles, was den Cluster beschreibt, gehört ins Git-Repo:
- Helm-Values
- Kubernetes-Manifeste / Kustomize-Files
- Ansible-Playbooks für Node-Setup
- k3s-Konfiguration

Das ist das wichtigste "Backup" — damit lässt sich der Cluster jederzeit von Null neu aufbauen.

---

## Backup-Ziele

| Option | Kosten | Aufwand | Geeignet für |
|---|---|---|---|
| **USB-Festplatte an einem Node** | ~40€ für 2TB | Gering (Cronjob + rsync) | Cluster-Snapshots, DB-Dumps |
| **NAS (Synology, TrueNAS)** | 200–400€ | Mittel | Alles inkl. Medien |
| **S3-kompatibel (Hetzner Storage Box, Backblaze B2)** | ~3–5€/Monat | Mittel (Velero-Config) | Off-Site für kritische Daten |
| **Zweiter Standort (WireGuard-Tunnel)** | Minimal | Hoch | Disaster Recovery |

Ein NAS löst zwei Probleme gleichzeitig: Backup-Ziel für den Cluster und Medien-Storage für Jellyfin.

---

## Secrets und Credentials

Kubernetes Secrets liegen unverschlüsselt in etcd/SQLite. Für die Sicherung im Git-Repo:
- **Sealed Secrets** — verschlüsselt Secrets mit einem Cluster-Zertifikat, nur der Cluster kann entschlüsseln
- **SOPS + age** — verschlüsselt YAML-Dateien mit einem age-Key, flexibler als Sealed Secrets

In beiden Fällen können Secrets sicher ins Git-Repo committed werden.

---

## Empfohlene Reihenfolge

1. **Sofort:** k3s-Snapshots per Cronjob von `k8s-master-01` auf `k8s-worker-01` kopieren
2. **Wenn Cluster steht:** Alle Manifeste, Helm-Values und Playbooks ins Git-Repo
3. **Wenn Applikationen laufen:** Velero aufsetzen mit MinIO oder S3-Backend
4. **Für Datenbanken:** Ergänzend `pg_dump`/`mysqldump` als CronJob
5. **Langfristig:** NAS als zentrales Backup-Ziel und Medien-Storage

---

## Nicht vergessen

- **Restore testen:** Ein Backup ohne getesteten Restore ist kein Backup. Mindestens einmal im Quartal testen.
- **Medien-Backup:** Digitalisierte DVDs sind aufwändig wiederherzustellen — separates Backup-Konzept auf NAS oder extern.
- **Monitoring:** Backup-Jobs in Prometheus/Grafana überwachen, damit fehlgeschlagene Backups nicht unbemerkt bleiben.
