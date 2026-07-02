# k3s Cluster Setup — Anleitung

## Ziel

k3s-Cluster mit drei Nodes aufsetzen:

| Hostname | Rolle | IP (Beispiel) |
|---|---|---|
| k8s-master-01 | Server (Control Plane) | 192.168.1.10 |
| k8s-worker-01 | Agent | 192.168.1.11 |
| k8s-worker-02 | Agent | 192.168.1.12 |

> Die IPs sind Platzhalter — an das eigene Netzwerk anpassen.

---

## 1. Voraussetzungen (alle drei Nodes)

### 1.1 Statische IPs vergeben

Kubernetes-Nodes brauchen feste IPs. Unter Debian 13 über `/etc/network/interfaces` oder NetworkManager konfigurieren.

```bash
# Prüfen, welches Netzwerk-Interface aktiv ist
ip addr show
```

Alternativ: Statische DHCP-Leases im Router vergeben (einfacher, funktioniert genauso).

### 1.2 Hostnames und /etc/hosts

Auf **allen drei Nodes** die Hostnamen eintragen, damit sie sich gegenseitig auflösen können:

```bash
# Hostname setzen (falls noch nicht geschehen)
sudo hostnamectl set-hostname k8s-master-01  # bzw. k8s-worker-01, k8s-worker-02
```

In `/etc/hosts` auf **allen Nodes**:

```
192.168.1.10  k8s-master-01
192.168.1.11  k8s-worker-01
192.168.1.12  k8s-worker-02
```

### 1.3 SSH-Zugriff einrichten

Vom eigenen Rechner SSH-Keys auf alle Nodes verteilen — erleichtert die Administration:

```bash
# Vom eigenen Rechner aus
ssh-keygen -t ed25519 -C "homeserver"  # falls noch kein Key vorhanden
ssh-copy-id user@k8s-master-01
ssh-copy-id user@k8s-worker-01
ssh-copy-id user@k8s-worker-02
```

### 1.4 System aktualisieren

Auf **allen drei Nodes**:

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

### 1.5 Firewall-Ports

Falls `ufw` oder `nftables` aktiv ist, folgende Ports öffnen:

| Port | Protokoll | Richtung | Zweck |
|---|---|---|---|
| 6443 | TCP | Zu Master | Kubernetes API |
| 10250 | TCP | Zwischen allen Nodes | Kubelet |
| 8472 | UDP | Zwischen allen Nodes | Flannel VXLAN |
| 51820 | UDP | Zwischen allen Nodes | Flannel WireGuard (falls aktiviert) |

```bash
# Prüfen ob eine Firewall aktiv ist
sudo ufw status
sudo nft list ruleset
```

> Wenn keine Firewall aktiv ist und die Nodes nur im Heimnetz hängen, kann man diesen Schritt überspringen.

---

## 2. k3s Server installieren (k8s-master-01)

### 2.1 Installation

```bash
curl -sfL https://get.k3s.io | sh -s - server \
  --write-kubeconfig-mode 644 \
  --tls-san k8s-master-01 \
  --tls-san 192.168.1.10
```

Erklärung der Flags:
- `--write-kubeconfig-mode 644` — Kubeconfig ist ohne sudo lesbar (praktisch für Homelab, in Produktion nicht empfohlen)
- `--tls-san` — Zusätzliche SANs im TLS-Zertifikat, damit man von außen per Hostname oder IP zugreifen kann

### 2.2 Installation prüfen

```bash
# Service-Status
sudo systemctl status k3s

# Node sollte Ready sein
sudo kubectl get nodes

# Token auslesen — wird für die Worker-Installation benötigt
sudo cat /var/lib/rancher/k3s/server/node-token
```

### 2.3 Kubeconfig sichern

Die Kubeconfig liegt unter `/etc/rancher/k3s/k3s.yaml`. Auf den eigenen Rechner kopieren:

```bash
# Vom eigenen Rechner aus
scp user@k8s-master-01:/etc/rancher/k3s/k3s.yaml ~/.kube/config-homeserver
```

In der kopierten Datei `server: https://127.0.0.1:6443` ersetzen durch `server: https://192.168.1.10:6443`.

```bash
# Testen
export KUBECONFIG=~/.kube/config-homeserver
kubectl get nodes
```

---

## 3. k3s Agents installieren (Worker-Nodes)

### 3.1 Installation auf k8s-worker-01 und k8s-worker-02

Den Token von Schritt 2.2 verwenden:

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://k8s-master-01:6443 K3S_TOKEN=<TOKEN> sh -
```

### 3.2 Cluster prüfen

Zurück auf dem Master oder vom eigenen Rechner:

```bash
kubectl get nodes
```

Erwartete Ausgabe:

```
NAME             STATUS   ROLES                  AGE   VERSION
k8s-master-01   Ready    control-plane,master   5m    v1.31.x+k3s1
k8s-worker-01   Ready    <none>                 1m    v1.31.x+k3s1
k8s-worker-02   Ready    <none>                 1m    v1.31.x+k3s1
```

### 3.3 Worker-Nodes labeln

```bash
kubectl label node k8s-worker-01 node-role.kubernetes.io/worker=worker
kubectl label node k8s-worker-02 node-role.kubernetes.io/worker=worker
```

---

## 4. Grundkonfiguration

### 4.1 Workloads vom Master fernhalten

Der Master hat wie die Worker nur 8GB RAM. Control-Plane-Last und Workloads gleichzeitig ist eng. Taint setzen, damit reguläre Pods nur auf den Workern laufen:

```bash
kubectl taint nodes k8s-master-01 node-role.kubernetes.io/master=:NoSchedule
```

> Falls du später RAM-Engpässe auf den Workern hast, kannst du den Taint jederzeit entfernen und den Master als dritten Worker mitnutzen.

### 4.2 Helm installieren

Helm ist der Standard-Paketmanager für Kubernetes. Auf dem eigenen Rechner (nicht auf den Nodes):

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### 4.3 Traefik prüfen

k3s bringt Traefik als Ingress-Controller mit. Prüfen ob er läuft:

```bash
kubectl get pods -n kube-system | grep traefik
```

Traefik reicht für den Anfang. Später kann man auf einen anderen Ingress-Controller wechseln, falls nötig.

### 4.4 Local-Path-Provisioner prüfen

k3s bringt einen lokalen Storage-Provisioner mit. Volumes werden unter `/var/lib/rancher/k3s/storage/` auf dem jeweiligen Node angelegt:

```bash
kubectl get storageclass
```

`local-path` sollte als Default-StorageClass angezeigt werden. Das reicht für den Start — Longhorn kommt später.

---

## 5. Smoke-Test

Einen einfachen Nginx deployen, um den Cluster zu testen:

```bash
kubectl create deployment nginx-test --image=nginx --replicas=2
kubectl expose deployment nginx-test --port=80 --type=NodePort
kubectl get svc nginx-test
```

Im Browser `http://192.168.1.11:<NodePort>` aufrufen. Wenn die Nginx-Standardseite erscheint, funktioniert der Cluster.

Aufräumen:

```bash
kubectl delete deployment nginx-test
kubectl delete svc nginx-test
```

---

## 6. Nächste Schritte

Nach erfolgreichem Setup in dieser Reihenfolge weiter:

1. **k3s-Snapshot-Backup einrichten** (siehe `backup_strategy.md`)
2. **Jellyfin deployen** mit NFS- oder Host-Path-Storage für Medien
3. **Monitoring-Stack** (kube-prometheus-stack via Helm)
4. **Longhorn** für replizierte Volumes (wenn Apps mit Datenbanken dazukommen)

---

## Wichtige Hinweise

### Ressourcen im Blick behalten

Bei 8GB pro Node zählt jedes MB. Nützliche Befehle:

```bash
# RAM- und CPU-Verbrauch pro Node
kubectl top nodes

# RAM- und CPU-Verbrauch pro Pod
kubectl top pods -A

# Metrics-Server muss dafür laufen (k3s bringt ihn mit)
```

### k3s Updates

k3s aktualisiert sich **nicht** automatisch. Updates manuell einspielen:

```bash
# Auf jedem Node einzeln, Master zuerst
curl -sfL https://get.k3s.io | sh -
```

Zwischen Master- und Worker-Update kurz warten und prüfen, dass der Master wieder Ready ist.

### k3s deinstallieren (falls nötig)

```bash
# Auf dem Server
/usr/local/bin/k3s-uninstall.sh

# Auf den Agents
/usr/local/bin/k3s-agent-uninstall.sh
```

### Troubleshooting

```bash
# k3s-Logs auf dem Master
sudo journalctl -u k3s -f

# k3s-Logs auf einem Worker
sudo journalctl -u k3s-agent -f

# Pod-Status und Events
kubectl get events -A --sort-by='.lastTimestamp'
```
