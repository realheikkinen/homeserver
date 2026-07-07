# Nextcloud Setup — Produktiv-Deployment

## Ziel

Nextcloud mit echter Persistenz: CloudNativePG als Datenbank (statt SQLite) und Longhorn als
Storage für die Nutzdaten (statt kein Storage). Löst den früheren Smoke-Test ab
(`nextcloud/smoke_test.md`, `nextcloud/values.yaml`) — der war nur ein Chart-Sanity-Check ohne
Datenerhalt, kein Vorläufer dieser Instanz.

---

## 1. Alten Smoke-Test entfernen

```bash
helm uninstall nextcloud -n nextcloud
```

> ✅ **Status (2026-07-07):** erledigt, Namespace danach leer (`kubectl get all -n nextcloud`).

---

## 2. CloudNativePG-Cluster für Nextcloud

Eigener Postgres-Cluster pro App (siehe `cloudnativepg_setup.md`), `instances: 1`,
`storageClass: longhorn`:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: nextcloud-postgres
  namespace: nextcloud
spec:
  instances: 1
  storage:
    size: 4Gi
    storageClass: longhorn
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  bootstrap:
    initdb:
      database: nextcloud
      owner: nextcloud
EOF

kubectl get cluster nextcloud-postgres -n nextcloud
```

CloudNativePG legt automatisch das Secret `nextcloud-postgres-app` mit den DB-Zugangsdaten an
(Keys: `host`, `username`/`user`, `password`, `dbname`, …) — kein manuelles Passwort nötig.

> ✅ **Status (2026-07-07):** erledigt, Cluster `healthy`, Secret vorhanden.

---

## 3. Admin-Zugangsdaten als Secret (statt Klartext in values.yaml)

```bash
ADMIN_PW=$(openssl rand -base64 24)
kubectl create secret generic nextcloud-admin -n nextcloud \
  --from-literal=nextcloud-username=admin \
  --from-literal=nextcloud-password="$ADMIN_PW"
```

> ⚠️ Das Passwort nur einmalig anzeigen und sicher aufbewahren (Passwortmanager) — es steht
> **nicht** im Git-Repo. Falls vergessen, neu generieren und Secret aktualisieren:
> `kubectl create secret generic nextcloud-admin -n nextcloud --from-literal=... --dry-run=client -o yaml | kubectl apply -f -`
>
> Perspektivisch: Sealed Secrets oder SOPS+age einführen (siehe `backup_strategy.md`), damit
> auch Secret-Definitionen versioniert im Git-Repo liegen können.

> ✅ **Status (2026-07-07):** erledigt, Secret `nextcloud-admin` angelegt.

---

## 4. Nextcloud deployen

Werte in `nextcloud/values-prod.yaml` (im Repo):

```yaml
nextcloud:
  host: 192.168.188.89:30080
  existingSecret:
    enabled: true
    secretName: nextcloud-admin
    usernameKey: nextcloud-username
    passwordKey: nextcloud-password

service:
  type: NodePort
  nodePort: 30080

ingress:
  enabled: false

persistence:
  enabled: true
  storageClass: longhorn
  size: 10Gi

internalDatabase:
  enabled: false

mariadb:
  enabled: false

externalDatabase:
  enabled: true
  type: postgresql
  existingSecret:
    enabled: true
    secretName: nextcloud-postgres-app
    usernameKey: username
    passwordKey: password
    hostKey: host
    databaseKey: dbname

resources:
  requests:
    memory: 256Mi
    cpu: 100m
  limits:
    memory: 768Mi
    cpu: 750m

startupProbe:
  enabled: true
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 60
  successThreshold: 1
```

```bash
helm install nextcloud nextcloud/nextcloud \
  --namespace nextcloud \
  --values nextcloud/values-prod.yaml
```

> Die Helm-NOTES-Ausgabe nach dem Install warnt pauschal vor SQLite ("did not provide an
> external database host") — das ist ein statisches Chart-Template, das nicht prüft, ob
> `externalDatabase` tatsächlich gesetzt ist. Mit `kubectl exec ... -- env | grep POSTGRES`
> lässt sich die echte Konfiguration verifizieren (siehe Abschnitt 6).

### Bekannte Falle: Liveness-Probe killt die Erstinstallation

Die Chart-Defaults für `livenessProbe`/`readinessProbe` (`initialDelaySeconds: 10,
periodSeconds: 10, failureThreshold: 3` → Kill nach ~40s) reichen **nicht** für die
Erstinstallation (DB-Schema anlegen, `occ install`) auf 8GB-Nodes mit Longhorn-Storage. Ohne
`startupProbe` landet der Pod in `CrashLoopBackOff`, weil er mitten in der Installation
abgeschossen wird, bevor sie fertig ist — bei uns 7 Restarts, bis das auffiel.

**Fix:** `startupProbe.enabled: true` mit großzügigem `failureThreshold` (oben: 60 × 10s =
10 Minuten) — danach greifen `liveness`/`readiness` mit den normalen, kurzen Intervallen für den
laufenden Betrieb. Bereits in `values-prod.yaml` enthalten.

> ✅ **Status (2026-07-07):** erledigt nach Behebung der Probe-Falle. Log zeigt
> `Installing with PostgreSQL database` → `Nextcloud was successfully installed`.

---

## 5. Verifizieren

### DB-Anbindung prüfen

```bash
POD=$(kubectl get pods -n nextcloud -l app.kubernetes.io/name=nextcloud -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n nextcloud "$POD" -- env | grep -i postgres
# POSTGRES_HOST sollte nextcloud-postgres-rw sein, kein sqlite/mysql-Bezug
```

### Erreichbarkeit

```bash
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: 192.168.188.89:30080" http://192.168.188.89:30080/status.php
# 200 erwartet
```

Im Browser: `http://192.168.188.89:30080`

### Persistenz-Test (Unterschied zum alten Smoke-Test)

```bash
kubectl delete pod -n nextcloud -l app.kubernetes.io/name=nextcloud
# warten bis der neue Pod Ready ist, dann Logs prüfen:
kubectl logs -n nextcloud -l app.kubernetes.io/name=nextcloud
# sollte KEIN "Initializing"/"Installing" mehr zeigen, sondern direkt Apache starten
```

> ✅ **Status (2026-07-07):** erledigt. Nach Pod-Neustart keine Neuinstallation, Login/Status
> weiterhin `200` — Daten und Konfiguration überleben den Neustart (anders als beim
> SQLite-Smoke-Test, der bei jedem Restart alles verlor).

---

## Nächste Schritte

Laut Roadmap in `nas_setup.md` Abschnitt 10:

1. ~~Nextcloud neu deployen~~ ✅ erledigt (2026-07-07)
2. **Velero einrichten** — Backups gegen MinIO/`velero`-Bucket auf dem NAS
3. Immich / Jellyfin deployen
4. CloudNativePG-Backups gegen `cnpg-backups`-Bucket einrichten, bevor Nextcloud produktiv mit echten Daten befüllt wird
