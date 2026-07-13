# Immich Setup — Foto-/Videobibliothek

## Ziel

Immich als selbstgehostete Foto-/Video-Bibliothek mit KI-Objekterkennung/Gesichtserkennung
(Machine-Learning-Komponente). Originale liegen auf dem NAS (HDD, sequenziell), Thumbnails auf
separater SSD (Random-I/O-intensiv), Metadaten/Vektor-Suche in CloudNativePG.

## Architektur-Übersicht

```
immich-server            ── API/Web, mountet Library (/data) + Thumbs (/data/thumbs)
immich-machine-learning  ── Gesichts-/Objekterkennung, Modell-Cache auf Longhorn
immich-valkey            ── Job-Queue (Redis-kompatibel), Daten unkritisch → emptyDir
immich-postgres (CNPG)   ── PostgreSQL 18 + VectorChord (vchord) statt pgvector
```

Namespace: `immich`. Zwei statische NFS-PV/PVCs nach Jellyfin-Muster (siehe `jellyfin_setup.md`):

| PVC | NFS-Export (nas-01) | Storage | Zweck |
|---|---|---|---|
| `immich-photos` | `/data/bulk/photos` (HDD) | 1700Gi | Originalfotos/-videos |
| `immich-thumbs` | `/data/fast/immich-thumbs` (SSD) | 900Gi | Thumbnails, Random-I/O |

Beide `ReadWriteMany`, `persistentVolumeReclaimPolicy: Retain` (wie bei Jellyfin — beim
PVC-Löschen sollen die echten Dateien nicht mitgelöscht werden).

---

## 1. Statische PV/PVCs

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: immich-photos-pv
spec:
  capacity:
    storage: 1700Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.188.64
    path: /data/bulk/photos
  mountOptions:
    - nfsvers=4.1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: immich-photos
  namespace: immich
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  volumeName: immich-photos-pv
  resources:
    requests:
      storage: 1700Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: immich-thumbs-pv
spec:
  capacity:
    storage: 900Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: 192.168.188.64
    path: /data/fast/immich-thumbs
  mountOptions:
    - nfsvers=4.1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: immich-thumbs
  namespace: immich
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  volumeName: immich-thumbs-pv
  resources:
    requests:
      storage: 900Gi
EOF
```

> ✅ **Status (2026-07-09):** erledigt, beide PVCs `Bound`.

---

## 2. CloudNativePG-Cluster mit VectorChord

**Wichtige Korrektur gegenüber der ursprünglichen Planung:** Immich nutzt inzwischen **nicht
mehr `pgvector`**, sondern **VectorChord** (`vchord`) als Vektor-Extension — anderes Image, andere
Extension als in älteren Immich-Anleitungen im Netz beschrieben.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: immich-postgres
  namespace: immich
spec:
  instances: 1
  imageName: ghcr.io/tensorchord/cloudnative-vectorchord:18.4
  postgresql:
    shared_preload_libraries:
      - vchord.so
  bootstrap:
    initdb:
      database: immich
      owner: immich
      postInitApplicationSQL:
        - "CREATE EXTENSION IF NOT EXISTS vchord CASCADE;"
        - "CREATE EXTENSION IF NOT EXISTS cube;"
        - "CREATE EXTENSION IF NOT EXISTS earthdistance CASCADE;"
  storage:
    size: 10Gi
    storageClass: longhorn
  resources:
    requests:
      memory: 512Mi
      cpu: 250m
    limits:
      memory: 1Gi
      cpu: "1"
EOF
```

`instances: 1` (nicht 3) aus denselben Gründen wie bei Nextcloud — siehe
`cloudnativepg_setup.md`. PG18 passt zur bei Nextcloud bereits verwendeten Major-Version.

Verifizieren:

```bash
kubectl exec -n immich immich-postgres-1 -- psql -U postgres -d immich -c "\dx"
# vchord, cube, earthdistance sollten gelistet sein
```

> ✅ **Status (2026-07-09):** erledigt. Cluster `Cluster in healthy state`, alle drei Extensions
> per `\dx` verifiziert. CloudNativePG legt automatisch das Secret `immich-postgres-app` mit
> App-Zugangsdaten an (Keys `host`, `username`, `dbname`, `password`).

---

## 3. Installation per Helm

```bash
helm install immich oci://ghcr.io/immich-app/immich-charts/immich \
  --version 0.13.1 \
  --namespace immich --create-namespace \
  --values immich/values.yaml
```

`immich/values.yaml` (im Repo) — Kernpunkte:

- `image.tag: v3.0.1` explizit gepinnt (der Chart aktualisiert sich nicht automatisch mit jedem
  Immich-Release, appVersion des Charts war zum Installationszeitpunkt v3.0.0)
- DB-Zugangsdaten per `secretKeyRef` aus `immich-postgres-app` (kein Klartext-Passwort im Repo)
- `immich.persistence.library.existingClaim: immich-photos`
- `server.persistence.thumbs.existingClaim: immich-thumbs` mit `globalMounts.path: /data/thumbs`
  (Details dazu unter Falle 2)
- `valkey.enabled: true`, Default-`emptyDir` (Job-Queue-Daten unkritisch bei Verlust)
- `machine-learning.persistence.cache` auf Longhorn (10Gi, RWX) statt `emptyDir`, damit
  ML-Modelle nicht bei jedem Pod-Neustart neu heruntergeladen werden

> ✅ **Status (2026-07-09):** erledigt. Helm-Release deployed, alle 4 Pods `1/1 Running`:
> `immich-server`, `immich-machine-learning`, `immich-valkey`, `immich-postgres-1`.

---

## 4. Drei nicht-offensichtliche Fallen

### Falle 1: Mount-Pfad der Library

Der Helm-Chart mountet die Library-PVC auf Container-Pfad **`/data`**, nicht auf
`/usr/src/app/upload` — dieser Pfad stammt aus der allgemeinen Docker-Compose-Doku von Immich und
gilt **nicht** für den Helm-Chart. Betrifft `immich.persistence.library` in `values.yaml`.

### Falle 2: `THUMB_LOCATION`-Env-Var wird nicht zuverlässig ausgewertet

Obwohl offiziell dokumentiert, landeten Thumbnails in Immich v3.0.1 trotz gesetzter
`THUMB_LOCATION`-Env-Var weiterhin unter `/data/thumbs` statt am Env-Var-Pfad. Der tatsächlich
funktionierende Mechanismus: die separate SSD-PVC muss **direkt an der Stelle gemountet werden,
wo Immich Thumbnails intern erwartet** (`/data/thumbs`, als zusätzlicher Mount "über" den
bestehenden `/data`-Mount) — kein Env-Var-Redirect nötig oder verlässlich. Umgesetzt über
`server.persistence.thumbs.globalMounts` in `values.yaml` (siehe oben).

### Falle 3: Marker-Datei `.immich` bei frischem PVC

Immich legt in jedem Storage-Unterordner eine Marker-Datei `.immich` an (Inhalt:
Epoch-Millisekunden-Timestamp) und prüft beim Start, ob sie vorhanden ist (System-
Integritätscheck). Wird ein frisches, leeres PVC über einen bestehenden Unterordner gemountet
(wie beim SSD-Split für `/data/thumbs`), fehlt diese Datei → Server crasht beim Start mit
`ENOENT: /data/thumbs/.immich`.

**Fix:** Vor dem ersten Start des Servers mit dem neuen Mount manuell `.immich` mit einem
Epoch-ms-Timestamp in das neue PVC schreiben, z. B. über einen kurzen Debug-Pod:

```bash
kubectl run immich-thumbs-init --rm -it --restart=Never -n immich \
  --image=busybox --overrides='{
    "spec": {
      "containers": [{
        "name": "init",
        "image": "busybox",
        "command": ["sh", "-c", "echo $(( $(date +%s) * 1000 )) > /thumbs/.immich"],
        "volumeMounts": [{"name": "thumbs", "mountPath": "/thumbs"}]
      }],
      "volumes": [{"name": "thumbs", "persistentVolumeClaim": {"claimName": "immich-thumbs"}}]
    }
  }'
```

> ✅ **Status (2026-07-09):** alle drei Fallen erkannt und behoben, im initialen Deployment
> berücksichtigt.

---

## 5. Nachträgliche Thumbnail-Regeneration

Zwei Testfotos wurden hochgeladen, **bevor** der SSD-Split-Fix (Fallen 2+3) fertig war — ihre
Thumbnails landeten dadurch auf dem alten Pfad und waren nach dem Fix nicht mehr sichtbar (die
Originale in `/data/upload` waren davon nie betroffen). Statt die Fotos zu löschen und neu
hochzuladen: **Thumbnails regeneriert** über Administration → Jobs → "Generate Thumbnails"
(Job-Karte "Missing" antriggern).

> ✅ **Status (2026-07-13):** erledigt. Thumbnails für beide Testfotos erfolgreich regeneriert,
> werden in der Immich-UI wieder angezeigt.

---

## 6. Verifizieren

```bash
kubectl get pods -n immich
# 4/4 Pods Running: immich-server, immich-machine-learning, immich-valkey, immich-postgres-1

kubectl get pvc -n immich
# immich-photos, immich-thumbs: Bound (statisch, ""); immich-machine-learning: Bound (longhorn)

curl -s http://192.168.188.89:30283/api/server/ping
# {"res":"pong"}
```

Service: NodePort `immich-server-nodeport` auf `192.168.188.89:30283` (Cluster-IP-Services zeigen
zusätzlich `immich-postgres-rw/-ro/-r`, `immich-machine-learning`, `immich-valkey` intern).

> ✅ **Status (2026-07-13):** vollständig verifiziert. Server erreichbar, Testfotos inkl.
> Thumbnails sichtbar. Immich-Deployment abgeschlossen.

---

## Nächste Schritte

Laut Roadmap in `nas_setup.md` Abschnitt 10:

1. ~~Immich deployen~~ ✅ erledigt (2026-07-09, Doku/Verifizierung abgeschlossen 2026-07-13)
2. **Velero einrichten** (bewusst nach hinten verschoben, siehe `nas_setup.md`)
