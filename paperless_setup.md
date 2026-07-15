# Paperless-ngx Setup — Dokumentenverwaltung mit OCR

## Ziel

Paperless-ngx als selbstgehostete Dokumentenverwaltung mit OCR (Texterkennung). Dokumente
(Originale/Archiv) liegen auf Longhorn (repliziert, klein im Vergleich zu Jellyfin/Immich),
Metadaten in CloudNativePG. Neue Dokumente kommen über einen NFS-Consume-Ordner auf dem NAS rein
— analog zum `tower01`→Jellyfin-Workflow aus `medien_ablage.md`.

## Architektur-Entscheidung: eigene Manifeste statt Helm-Chart

Anders als Nextcloud/Jellyfin/Immich gibt es **keinen offiziellen Helm-Chart** für
Paperless-ngx — das Projekt pflegt nur eine docker-compose-Referenz. Die bekannteste
Community-Alternative wird von einem Drittanbieter gepflegt, nicht vom Paperless-Team selbst.
Da die App architektonisch simpel ist (ein Container für Webserver + Celery-Worker +
Consumer-Watcher zusammen, siehe unten) und das Repo ohnehin schon CNPG-Cluster/PVs von Hand
schreibt, fiel die Entscheidung auf **eigene Kubernetes-Manifeste** statt Helm — siehe
`paperless/`-Ordner im Repo.

## Architektur-Übersicht

```
paperless (Deployment, 1 Container) ── Webserver (granian) + Celery-Worker + Celery-Beat
                                        + Consumer-Watcher, alles im offiziellen Entrypoint
paperless-valkey                    ── Job-Queue (Redis-kompatibel), Daten unkritisch → emptyDir
paperless-postgres (CNPG)           ── PostgreSQL, Metadaten + Volltextsuche
```

Namespace: `paperless`. Drei Volumes:

| PVC | Backend | Zweck |
|---|---|---|
| `paperless-consume` | NFS (statisch, `nas-01:/data/bulk/paperless-consume`) | Eingangsordner für neue Dokumente |
| `paperless-media` | Longhorn (20Gi) | Dokumente/Archiv (Originale) |
| `paperless-data` | Longhorn (5Gi) | Suchindex, Klassifikations-Modell, Thumbnails |

`paperless-consume` ist `ReadWriteMany`/`Retain` (wie `jellyfin-media-pv`) — statische PV statt
`nfs-nas`-StorageClass, weil ein **bereits bestehender, fester** NFS-Export gebraucht wird
(der dynamische Provisioner legt Unterverzeichnisse mit Zufallsnamen an, ungeeignet für einen
Ordner, in den von außen manuell Dateien reinkopiert werden sollen).

Aktuelle Version: `ghcr.io/paperless-ngx/paperless-ngx:2.20.15` (explizit gepinnt, analog zu
Immichs Versions-Pin, statt `latest`).

---

## 1. NFS-Export auf dem NAS

```bash
sudo mkdir -p /data/bulk/paperless-consume
```

In `/etc/exports` ergänzt:

```
/data/bulk/paperless-consume  192.168.188.0/24(rw,sync,no_subtree_check,no_root_squash)
```

```bash
sudo exportfs -ra
sudo showmount -e localhost
```

> ✅ **Status (2026-07-16):** erledigt, Export aktiv.

---

## 2. Namespace + Storage

`paperless/namespace.yaml`, `paperless/storage.yaml` (im Repo) — statische NFS-PV/PVC für
`paperless-consume` (Kapazität nur Anzeige, kein Limit, wie bei `jellyfin-media-pv`), Longhorn-PVCs
`paperless-media` (20Gi) und `paperless-data` (5Gi).

```bash
kubectl apply -f paperless/namespace.yaml
kubectl apply -f paperless/storage.yaml
kubectl get pvc -n paperless
```

> ✅ **Status (2026-07-16):** erledigt, alle drei PVCs sofort `Bound`.

---

## 3. CloudNativePG-Cluster

`paperless/postgres-cluster.yaml` — `instances: 1`, `storageClass: longhorn` (4Gi), Ressourcen
analog Nextcloud/Immich (siehe `cloudnativepg_setup.md`).

```bash
kubectl apply -f paperless/postgres-cluster.yaml
kubectl get cluster paperless-postgres -n paperless
```

CloudNativePG legt automatisch das Secret `paperless-postgres-app` an (Keys u.a. `host`, `port`,
`dbname`, `username`, `password`) — kein manuelles Passwort-Handling.

> ✅ **Status (2026-07-16):** erledigt, Cluster `Cluster in healthy state` nach ca. 10s.

---

## 4. Valkey (Job-Queue)

`paperless/redis.yaml` — kleines `Deployment`+`Service`, `emptyDir` (Job-Queue-Daten unkritisch
bei Verlust, analog `immich-valkey`).

```bash
kubectl apply -f paperless/redis.yaml
kubectl get pods -n paperless -l app=paperless-valkey
```

> ✅ **Status (2026-07-16):** erledigt, Pod sofort `1/1 Running`.

---

## 5. Secrets

Django-`SECRET_KEY` und Admin-Zugangsdaten werden **nicht** als Datei mit echten Werten
committed (analog `nextcloud-admin`-Muster), sondern direkt per `kubectl create secret generic`
mit `openssl rand`-generierten Werten angelegt:

```bash
SECRET_KEY=$(openssl rand -base64 48 | tr -d '\n')
kubectl create secret generic paperless-secret-key -n paperless \
  --from-literal=secret-key="$SECRET_KEY"

ADMIN_PW=$(openssl rand -base64 18 | tr -d '/+=')
kubectl create secret generic paperless-admin -n paperless \
  --from-literal=paperless-admin-username=admin \
  --from-literal=paperless-admin-password="$ADMIN_PW"
```

> ⚠️ Admin-Passwort nur einmalig angezeigt, im Passwortmanager gesichert — **nicht** im Git-Repo.

> ✅ **Status (2026-07-16):** erledigt, beide Secrets angelegt.

---

## 6. Deployment

`paperless/deployment.yaml` — `Deployment` `paperless` (1 Replica) + `Service` (NodePort 30185).
DB-Zugangsdaten per `secretKeyRef` aus `paperless-postgres-app`, Admin/Secret-Key aus den eigenen
Secrets. `PAPERLESS_OCR_LANGUAGE=deu+eng`, `PAPERLESS_TIME_ZONE=Europe/Berlin`,
`PAPERLESS_URL=http://192.168.188.89:30185` (setzt automatisch `ALLOWED_HOSTS`,
`CORS_ALLOWED_HOSTS`, `CSRF_TRUSTED_ORIGINS`).

```bash
kubectl apply -f paperless/deployment.yaml
kubectl rollout status deployment/paperless -n paperless
```

> ✅ **Status (2026-07-16):** erledigt nach Behebung der beiden Fallen unten.

---

## 7. Zwei nicht-offensichtliche Fallen

### Falle 1: `PAPERLESS_PORT`-Kollision durch Kubernetes-Service-Discovery-Env-Vars

Kubernetes injiziert für **jeden Service im selben Namespace** automatisch Docker-Links-kompatible
Umgebungsvariablen in jeden Pod (`<SERVICE_NAME>_PORT=tcp://<cluster-ip>:<port>` u.a.). Weil der
Service hier `paperless` heißt, landete `PAPERLESS_PORT=tcp://10.43.94.196:8000` im Container —
das kollidiert mit Paperless' eigenem `PAPERLESS_PORT`-Konfigurations-Var (Ziel-Port für den
internen Webserver `granian`). Ergebnis: `granian` crashte in einer Schleife mit
`Error: Invalid value for '--port': 'tcp://10.43.94.196:8000' is not a valid integer.`

**Fix:** `enableServiceLinks: false` im Pod-Spec (`paperless/deployment.yaml`) — unterdrückt die
automatische Env-Var-Injection sauber, ohne den Service umbenennen zu müssen. Betrifft
grundsätzlich jede App, deren eigener Service-Name mit einem von der App selbst verwendeten
Env-Var-Präfix kollidiert — für künftige Eigenbau-Manifeste im Hinterkopf behalten.

### Falle 2: `inotify` erkennt keine Änderungen anderer NFS-Clients

Der Consumer nutzt standardmäßig `inotify`, um neue Dateien im Consume-Ordner zu erkennen. Das
funktioniert **nicht zuverlässig über NFS**, wenn die Datei von einem **anderen** NFS-Client
geschrieben wird als dem, der den Consumer ausführt (bekannte, dokumentierte Einschränkung von
`inotify` auf Netzwerk-Dateisystemen) — genau unser Fall: Dateien kommen von Stefans Rechner oder
anderen Geräten im Netz, nicht vom Paperless-Pod selbst. Ohne Fix bleiben neu hinzugefügte
Dateien im Consume-Ordner liegen, bis der Pod neu startet (dann werden nur die beim Start bereits
vorhandenen Dateien verarbeitet).

**Fix:** `PAPERLESS_CONSUMER_POLLING=30` (Sekunden) im Deployment gesetzt — schaltet auf
Polling statt Dateisystem-Benachrichtigungen um. Erkannt durch einen End-to-End-Test: Testdatei
über einen Debug-Pod (analog `.immich`-Marker-Pod aus `immich_setup.md`) in den Consume-PVC
geschrieben, blieb mit reinem `inotify` dauerhaft unverarbeitet liegen; nach dem Polling-Fix
innerhalb von Sekunden konsumiert.

> ✅ **Status (2026-07-16):** beide Fallen erkannt und behoben, im Deployment berücksichtigt.

---

## 8. Verifizieren

```bash
kubectl get pods -n paperless
kubectl get pvc -n paperless
curl -s -o /dev/null -w "%{http_code}\n" http://192.168.188.89:30185/api/
# 302 (Redirect zum Login, normal ohne Auth)
```

End-to-End-Test (Consume-Pipeline): Testdatei über einen Debug-Pod in `paperless-consume`
geschrieben, nach Polling-Intervall (≤30s) automatisch konsumiert:

```
[paperless.consumer] Consuming paperless-testdokument.txt
[paperless.consumer] Document 2026-07-16 paperless-testdokument consumption finished
[paperless.tasks] ConsumeTaskPlugin completed with: Success. New document id 1 created
```

Login-Test per API (`POST /api/token/` mit Admin-Zugangsdaten) erfolgreich, Dokument über
`/api/documents/` abrufbar. Auch im Web-Frontend sichtbar (von Stefan bestätigt).

> ✅ **Status (2026-07-16):** vollständig verifiziert. Consume-Pipeline, Login und API
> funktionieren. Paperless-Deployment abgeschlossen.

---

## Offene Punkte / bewusst verschoben

- **Export-Ordner / `document_exporter`** — optional, später bei Bedarf ergänzen. Velero sichert
  die Longhorn-PVCs (`paperless-media`, `paperless-data`) ohnehin über den bestehenden
  `daily-backup`-Schedule (Exclude- statt Include-Liste, siehe `velero_setup.md`) — keine
  Velero-Änderung nötig, `paperless` fällt automatisch mit rein.
- **Gotenberg/Tika** (Office-Dokument-Konvertierung, z.B. `.docx`→PDF) — nicht Teil dieses
  Deployments, nur nötig falls auch Office-Dateien eingescannt werden sollen.
