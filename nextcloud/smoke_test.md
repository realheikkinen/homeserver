# Nextcloud Smoke-Test

## Voraussetzungen

Helm installieren (falls noch nicht geschehen):

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## Installation

```bash
# 1. Nextcloud Helm-Repo hinzufügen
helm repo add nextcloud https://nextcloud.github.io/helm/
helm repo update

# 2. Namespace anlegen
kubectl create namespace nextcloud

# 3. Installieren
helm install nextcloud nextcloud/nextcloud \
  --namespace nextcloud \
  --values ~/github/homeserver/nextcloud/values.yaml

# 4. Warten bis Pods laufen (MariaDB startet zuerst, dauert ~2 Min)
kubectl get pods -n nextcloud -w
```

Erwarteter Endzustand:

```
NAME                                    READY   STATUS    RESTARTS
nextcloud-mariadb-0                     1/1     Running   0
nextcloud-XXXX                          1/1     Running   0
```

## Zugriff

```bash
# NodePort prüfen (sollte 30080 sein)
kubectl get svc -n nextcloud
```

Im Browser: `http://192.168.188.89:30080` (k8s-worker-01)
Login: `admin` / `smoketest123`

## Aufräumen

```bash
helm uninstall nextcloud --namespace nextcloud
kubectl delete namespace nextcloud
```
