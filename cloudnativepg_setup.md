# CloudNativePG Setup — PostgreSQL-Operator

## Ziel

CloudNativePG stellt PostgreSQL-Datenbanken für Apps im Cluster bereit (Nextcloud, Immich, …).
Nutzt die `longhorn` StorageClass — die Daten liegen also repliziert auf den SSDs der
Dell-Nodes, der NAS ist nur Backup-Ziel (nicht Host), siehe `nas_setup.md`.

> Nicht verwechseln mit der (zurückgestellten) Migration des **k3s-internen** Datastores auf
> PostgreSQL (`nas_setup.md` Abschnitt 6) — das ist ein komplett anderes Thema. CloudNativePG
> betrifft ausschließlich App-Datenbanken.

## Architektur-Entscheidung: 1 Instanz statt 3 pro Cluster

CloudNativePG-Beispiele/Defaults gehen oft von `instances: 3` aus (HA mit automatischem
Failover). Bei 8GB RAM pro Node und nur 2 Nodes, auf denen Workloads laufen (Master ist
getaintet, siehe `k3s_setup.md`), ist das für ein Homelab unverhältnismäßig teuer — jede
zusätzliche Instanz kostet RAM/CPU dauerhaft, nicht nur im Fehlerfall.

Stattdessen: **`instances: 1`** pro App-Datenbank. Ausfallsicherheit kommt über zwei andere
Ebenen ab, die ohnehin vorhanden sind:
- **Longhorn** repliziert den zugrundeliegenden Storage (2 Replicas, siehe `longhorn_setup.md`) —
  ein Node-Ausfall verliert also nicht die Daten, nur kurzzeitig die Verfügbarkeit bis der Pod
  neu geschedult ist
- **Velero + CloudNativePG-eigene Backups** gegen MinIO auf dem NAS (Abschnitt weiter unten /
  `backup_strategy.md`)

Falls später mehr RAM verfügbar ist (NAS-Upgrade, weitere Nodes), kann pro Cluster einzeln auf
`instances: 3` umgestellt werden — das ist eine reine Spec-Änderung, kein Re-Deploy.

---

## 1. Operator installieren

Von einem Rechner mit Cluster-Zugriff:

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update

helm install cnpg cnpg/cloudnative-pg \
  --namespace cnpg-system --create-namespace
```

Prüfen:

```bash
kubectl -n cnpg-system get pods
# cnpg-cloudnative-pg-... sollte 1/1 Running sein
```

> ✅ **Status (2026-07-07):** erledigt, Operator läuft.

---

## 2. Postgres-Cluster anlegen (Beispiel/Vorlage)

Pro App eine eigene `Cluster`-Resource, `instances: 1` (siehe Architektur-Entscheidung oben),
`storageClass: longhorn`. Ressourcen-Requests/-Limits explizit setzen — sonst laufen
Postgres-Instanzen bei 8GB-Nodes schnell dem Node-RAM davon:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: <app>-postgres
  namespace: <app-namespace>
spec:
  instances: 1
  storage:
    size: 2Gi              # je nach App anpassen
    storageClass: longhorn
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "512Mi"
      cpu: "500m"
```

```bash
kubectl apply -f <app>-postgres.yaml
kubectl get cluster -n <app-namespace>
```

Erwartete `STATUS`: `Cluster in healthy state`, `PRIMARY` zeigt den Pod-Namen (`<name>-1`).

CloudNativePG legt automatisch ein Secret `<name>-app` mit den Zugangsdaten für die
Anwendung an (`kubectl get secret <name>-app -o yaml`) — darüber verbinden sich die App-Pods,
kein manuelles Passwort-Handling nötig.

---

## 3. Smoke-Test

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: cnpg-smoketest
  namespace: default
spec:
  instances: 1
  storage:
    size: 1Gi
    storageClass: longhorn
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "512Mi"
      cpu: "500m"
EOF

# Warten bis "Cluster in healthy state"
kubectl get cluster cnpg-smoketest

# Schreib-/Lesetest
kubectl exec cnpg-smoketest-1 -c postgres -- psql -U postgres \
  -c "CREATE TABLE smoketest (msg text); INSERT INTO smoketest VALUES ('cnpg-ok'); SELECT * FROM smoketest;"
```

Aufräumen (löscht auch die PVC über den Owner-Reference-Mechanismus):

```bash
kubectl delete cluster cnpg-smoketest
```

> ✅ **Status (2026-07-07):** erledigt. Cluster wurde `healthy`, PVC `Bound` auf `longhorn`,
> Schreib-/Lesetest über `psql` erfolgreich (`cnpg-ok`), Cluster + PVC danach vollständig entfernt.

---

## 4. Backups (später, vor Produktiv-Einsatz)

CloudNativePG kann direkt gegen S3-kompatible Ziele sichern (Continuous WAL-Archiving +
Basebackups) — passt zum bereits vorhandenen MinIO-Bucket `cnpg-backups` auf dem NAS
(`nas_setup.md` Abschnitt 7). Wird eingerichtet, sobald die erste echte App-Datenbank
(Nextcloud) läuft — siehe `backup_strategy.md`.

---

## Nächste Schritte

Laut Roadmap in `nas_setup.md` Abschnitt 10:

1. ~~CloudNativePG~~ ✅ Operator erledigt (2026-07-07)
2. **Nextcloud neu deployen** — mit echtem CloudNativePG-Cluster (`instances: 1`, `longhorn`-PVC) statt SQLite-Smoke-Test
3. Velero einrichten
4. Immich / Jellyfin deployen
