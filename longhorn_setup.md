# Longhorn Setup — replizierter Block-Storage

## Ziel

Longhorn stellt replizierten Block-Storage für App-Datenbanken (CloudNativePG) und andere
kritische Daten bereit — auf den lokalen 250GB-SSDs der drei Dell-Nodes. Ergänzt die bereits
vorhandenen StorageClasses:

| StorageClass | Backend | Geeignet für |
|---|---|---|
| `local-path` (Default) | Lokale SSD auf dem Node | Temporäre Daten, Development |
| `longhorn` | Replizierte SSDs (Dell-Nodes) | Datenbanken (CloudNativePG), kritische App-Daten |
| `nfs-nas` | HDD auf dem NAS | Medien, große Dateien, unkritische Daten |

## Architektur-Entscheidung: Replica-Anzahl 2 statt 3

`k8s-master-01` trägt den Taint `node-role.kubernetes.io/master:NoSchedule` (siehe `k3s_setup.md`)
— reguläre Workloads inkl. Longhorn-Replicas laufen also nur auf den beiden Workern. Longhorns
Standard-Replica-Anzahl ist 3; mit nur 2 verfügbaren Nodes würden Volumes dauerhaft als
"degraded" markiert. Deshalb wird die Default-Replica-Anzahl beim Install auf **2** gesetzt.

> Falls der Master-Taint später entfernt wird (z.B. bei RAM-Engpässen auf den Workern), kann die
> Replica-Anzahl nachträglich auf 3 erhöht werden — betrifft dann aber auch die
> Control-Plane-Last auf dem Master.

---

## 1. Voraussetzungen (alle drei Nodes)

Longhorn braucht `open-iscsi`, um Volumes anzubinden — auf **allen drei Nodes**, auch auf dem
getainteten Master (falls der Taint später entfernt wird):

```bash
sudo apt update && sudo apt install -y open-iscsi
sudo systemctl enable --now iscsid
systemctl status iscsid --no-pager
```

> ✅ **Status (2026-07-07):** erledigt auf allen drei Nodes, `iscsid` läuft.

---

## 2. Installation per Helm

Von einem Rechner mit Cluster-Zugriff (z.B. `KUBECONFIG=~/.kube/config-homeserver/k3s.yaml`):

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system --create-namespace \
  --set defaultSettings.defaultReplicaCount=2 \
  --set persistence.defaultClass=false \
  --set persistence.defaultClassReplicaCount=2
```

Erklärung der Flags:
- `defaultSettings.defaultReplicaCount=2` — globale Replica-Anzahl (siehe Architektur-Entscheidung oben)
- `persistence.defaultClass=false` — Longhorns StorageClass wird **nicht** zur Cluster-Default (das bleibt `local-path`)
- `persistence.defaultClassReplicaCount=2` — Replica-Anzahl auch auf der StorageClass selbst konsistent gesetzt

Die `int64`-Format-Warnungen von `helm install` sind ein bekanntes, harmloses CRD-Schema-Problem
bei Longhorn — kein Fehler.

### Installation prüfen

```bash
kubectl -n longhorn-system get pods
```

Warten, bis alle Pods `Running`/`Completed` sind — dauert ein paar Minuten (CSI-Sidecars,
Instance-Manager, Engine-Images). `longhorn-manager` darf dabei kurzzeitig 1-2 Restarts haben,
das ist während der Init-Phase normal.

Erwartete Komponenten (u.a.): `longhorn-manager` (DaemonSet, 1 pro Worker), `longhorn-driver-deployer`,
`longhorn-ui`, `csi-attacher`/`csi-provisioner`/`csi-resizer`/`csi-snapshotter`, `longhorn-csi-plugin`,
`instance-manager`, `engine-image-*`.

> ✅ **Status (2026-07-07):** erledigt, alle Pods `Running`.

---

## 3. Verifizieren

### StorageClass prüfen

```bash
kubectl get storageclass
```

Erwartung: `local-path` weiterhin `(default)`, zusätzlich `longhorn` und `longhorn-static` (nicht default).

### Smoke-Test mit echtem Schreibzugriff

Reine PVC-Bindung reicht bei Longhorn nicht als Test — das Volume wird erst beim ersten Pod, der
es mountet, tatsächlich angelegt. Daher Test mit PVC **und** Pod:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: longhorn-smoketest
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: longhorn-smoketest
  namespace: default
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "echo longhorn-ok > /data/test.txt && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: longhorn-smoketest
EOF

# Warten bis Running, dann prüfen
kubectl get pvc longhorn-smoketest
kubectl exec longhorn-smoketest -- cat /data/test.txt   # sollte "longhorn-ok" ausgeben
```

Aufräumen:

```bash
kubectl delete pod longhorn-smoketest
kubectl delete pvc longhorn-smoketest
```

> ✅ **Status (2026-07-07):** erledigt. PVC sofort `Bound`, Pod konnte schreiben und lesen
> (`longhorn-ok`), Test-Ressourcen wieder gelöscht.

---

## 4. Longhorn-UI (optional, für Storage-Übersicht/Snapshots)

Kein Ingress eingerichtet — Zugriff bei Bedarf per Port-Forward:

```bash
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80
```

Danach im Browser: `http://localhost:8080`

---

## 5. Ressourcen im Blick behalten

Bei 8GB RAM pro Node lohnt es sich, Longhorns Overhead im Auge zu behalten — `instance-manager`
und `engine`-Prozesse laufen pro angebundenem Volume zusätzlich zum eigentlichen App-Pod:

```bash
kubectl top pods -n longhorn-system
kubectl top nodes
```

---

## Nächste Schritte

Laut Roadmap in `nas_setup.md` Abschnitt 10:

1. ~~Longhorn deployen~~ ✅ erledigt (2026-07-07)
2. **CloudNativePG** — PostgreSQL-Operator, nutzt die `longhorn` StorageClass
3. Nextcloud neu deployen (mit CloudNativePG + Longhorn PVC)
4. Velero einrichten
5. Immich / Jellyfin deployen
