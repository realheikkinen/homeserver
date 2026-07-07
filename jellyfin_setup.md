# Jellyfin Setup — Mediaserver

## Ziel

Jellyfin als Mediaserver, Mediathek direkt vom NAS-Export `/data/bulk/media` (HDD, sequenzielles
Streaming). Konfiguration/Metadaten-DB (SQLite, intern) liegt auf Longhorn — repliziert, aber
unabhängig von der Mediathek selbst.

## Architektur-Entscheidung: statische PV statt `nfs-nas`-StorageClass

Der `nfs-nas`-Provisioner (siehe `nas_setup.md` Abschnitt 5) legt **dynamisch** Unterverzeichnisse
unter `/data/bulk/k3s-pvc` an — das ist ein anderer Export als `/data/bulk/media`, wo die
eigentlichen Mediendateien liegen (bzw. später über SMB/manuell draufkopiert werden). Für den
Zugriff auf einen **bereits bestehenden** NFS-Export braucht es eine **statische** PV, die direkt
auf den Export zeigt — kein dynamisches Provisioning.

```
nfs-nas (dynamisch)     → /data/bulk/k3s-pvc/<random-subdir>   (für generische PVC-Anfragen)
jellyfin-media-pv (statisch) → /data/bulk/media                (fester, bestehender Export)
```

---

## 1. Statische PV/PVC für die Mediathek

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jellyfin-media-pv
spec:
  capacity:
    storage: 1700Gi        # entspricht der Größe von /data/bulk, nur Kapazitäts-Anzeige, kein Limit
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    server: nas-01
    path: /data/bulk/media
  mountOptions:
    - nfsvers=4.1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jellyfin-media
  namespace: jellyfin
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  volumeName: jellyfin-media-pv
  resources:
    requests:
      storage: 1700Gi
EOF
```

`persistentVolumeReclaimPolicy: Retain` ist wichtig — bei `Delete` (Default vieler
StorageClasses) würden beim Löschen der PVC die echten Mediendateien auf dem NAS gelöscht.

Config-PVC (Longhorn, für Jellyfins interne SQLite-DB/Metadaten):

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jellyfin-config
  namespace: jellyfin
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 5Gi
EOF
```

> ✅ **Status (2026-07-07):** erledigt, beide PVCs `Bound`.

---

## 2. Voraussetzung: `nfs-common` auf **jedem** Node, der den Pod ausführen könnte

Anders als der `nfs-nas`-Provisioner (der NFS im Provisioner-Pod selbst mountet) mountet dieses
statische NFS-Volume direkt über den Kubelet des Nodes, auf dem der Pod läuft. Ohne
`mount.nfs`-Helper (Paket `nfs-common`) schlägt der Mount fehl:

```
mount: ... fsconfig() failed: NFS: mount program didn't pass remote address.
```

`nfs-common` war zu diesem Zeitpunkt nur auf `k8s-worker-01` installiert (aus dem NFS-Test in
`nas_setup.md`) — der Jellyfin-Pod landete aber zunächst auf `k8s-worker-02`, wo es fehlte.

```bash
# Auf jedem Node ohne nfs-common:
sudo apt update && sudo apt install -y nfs-common
```

> ✅ **Status (2026-07-07):** erledigt auf `k8s-worker-01` und `k8s-worker-02`. Auf
> `k8s-master-01` (noch) nicht nachgezogen — unkritisch, solange der Master-Taint
> (`NoSchedule`) bestehen bleibt und dort keine Workload-Pods landen. Falls der Taint später
> entfernt wird: hier nachholen.

---

## 3. Installation per Helm

```bash
helm repo add jellyfin https://jellyfin.github.io/jellyfin-helm
helm repo update

helm install jellyfin jellyfin/jellyfin \
  --namespace jellyfin --create-namespace \
  --values jellyfin/values.yaml
```

`jellyfin/values.yaml` (im Repo):

```yaml
service:
  type: NodePort
  port: 8096
  nodePort: 30096

persistence:
  config:
    enabled: true
    existingClaim: jellyfin-config
  media:
    enabled: true
    existingClaim: jellyfin-media
  cache:
    enabled: false   # emptyDir reicht, unkritischer Bild-/Transcoding-Cache

resources:
  requests:
    memory: 512Mi
    cpu: 250m
  limits:
    memory: 2Gi
    cpu: "2"
```

Der offizielle Jellyfin-Chart bringt bereits einen sinnvollen `startupProbe` mit (bis zu 5 Minuten
für den Erststart, danach greifen normale liveness/readiness) — die Falle aus
`nextcloud_setup.md` (fehlender startupProbe → CrashLoopBackOff bei Erstinstallation) tritt hier
nicht auf.

> ✅ **Status (2026-07-07):** erledigt, Chart deployed.

---

## 4. Verifizieren

```bash
kubectl get pods -n jellyfin
# 1/1 Running

curl -s -o /dev/null -w "%{http_code}\n" http://192.168.188.89:30096/health
# 200

curl -s -o /dev/null -w "%{http_code}\n" http://192.168.188.89:30096/
# 302 (Redirect zum Setup-Wizard, normal beim ersten Aufruf)
```

Im Browser: `http://192.168.188.89:30096` — Einrichtungsassistenten durchlaufen, Mediathek unter
`/media` einbinden (aktuell leer, bis Dateien auf `nas-01:/data/bulk/media` liegen).

> ✅ **Status (2026-07-07):** erledigt. Pod `1/1 Running`, HTTP-Checks erfolgreich.
>
> **Nebenbefund:** Beim ersten Pod-Reschedule (nach dem `nfs-common`-Fix) kurzzeitig
> `Multi-Attach error` für die Longhorn-Config-PVC, weil das alte Volume vom vorherigen Node
> noch nicht losgelassen war — hat sich nach ~1 Minute von selbst gelöst (normales Verhalten bei
> RWO-Volumes und Pod-Rescheduling, kein Eingriff nötig).

---

## Nächste Schritte

Laut Roadmap in `nas_setup.md` Abschnitt 10:

1. ~~Jellyfin deployen~~ ✅ erledigt (2026-07-07)
2. **Immich deployen** (CloudNativePG-Cluster mit pgvector, zwei NFS-PVCs: Thumbnails auf SSD, Originale auf HDD)
3. **Velero einrichten** (bewusst nach hinten verschoben, siehe `nas_setup.md`)
4. Mediendateien auf `nas-01:/data/bulk/media` ablegen und Jellyfin-Bibliothek scannen lassen
