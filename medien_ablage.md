# Medien-Ablage für Jellyfin — Workflow von DVD-Rip bis Bibliothek

## Übersicht

`tower01` ist der Ripping-Rechner (eigener PC mit CD/DVD-Laufwerk, MakeMKV läuft dort lokal in
Docker). Von dort werden die gerippten Filme auf den NAS kopiert, wo Jellyfin sie über den
bestehenden NFS-Export liest.

```
tower01 (CD/DVD-Laufwerk + MakeMKV-Docker)
  └── rippt Disc → Rohdateien lokal, z.B. /home/user/media/<FILM_NAME>/*.mkv
        │
        │  rsync über NFS-Mount
        ▼
nas-01: /data/bulk/media  (NFS-Export, siehe nas_setup.md Abschnitt 4)
        │
        │  NFS (statische PV, siehe jellyfin_setup.md)
        ▼
Jellyfin-Pod: /media
```

Der Export `/data/bulk/media` ist bereits eingerichtet und in Jellyfin als `/media` gemountet —
hier geht es nur noch um den Weg der Datei von `tower01` bis dorthin.

---

## 1. Namenskonvention (wichtig für Jellyfins Metadaten-Scraper)

```
/data/bulk/media/
├── Filme/
│   └── Filmname (Jahr)/
│       └── Filmname (Jahr).mkv
└── Serien/
    └── Serienname/
        └── Season 01/
            └── Serienname S01E01.mkv
```

Jahr in Klammern und `Season NN` sind notwendig, damit Jellyfin (TMDb o.ä.) den Titel automatisch
korrekt erkennt, statt manuell nachbessern zu müssen.

---

## 2. NFS-Mount auf `tower01` — Automount statt Dauer-Mount

**Entscheidung:** Kein permanenter `_netdev`-Mount in `/etc/fstab`, sondern ein
**systemd-Automount**.

**Warum:** `tower01` läuft nicht dauerhaft, sondern wird für Ripping-Sessions hochgefahren. Ein
klassischer Boot-Zeit-Mount würde den Start blockieren/verzögern, falls der NAS gerade nicht
erreichbar ist. Ein Automount dagegen mountet erst beim ersten tatsächlichen Zugriff auf den
Ordner und hängt sich nach einer Leerlaufzeit automatisch wieder aus — kein manuelles
`mount`/`umount` mehr nötig, aber auch kein Boot-Risiko.

### Voraussetzung

```bash
apt update && apt install -y nfs-common
```

### Einrichtung

Mountpoint anlegen:

```bash
mkdir -p /mnt/jellyfin-media
```

Eintrag in `/etc/fstab`:

```
192.168.188.64:/data/bulk/media  /mnt/jellyfin-media  nfs  noauto,x-systemd.automount,x-systemd.idle-timeout=600,_netdev  0  0
```

- `noauto` — wird beim Boot nicht aktiv gemountet
- `x-systemd.automount` — systemd erzeugt daraus automatisch eine Automount-Unit
- `x-systemd.idle-timeout=600` — nach 10 Minuten Inaktivität wird wieder ausgehängt
- `_netdev` — als netzwerkabhängiges Gerät markiert (relevant falls doch mal automatisch gemountet wird)

Aktivieren:

```bash
systemctl daemon-reload
systemctl start "$(systemd-escape --path --suffix=automount /mnt/jellyfin-media)"
```

> **Warum `systemd-escape` statt den Unit-Namen von Hand zu tippen** (z. B.
> `mnt-jellyfin\x2dmedia.automount`): Der Bindestrich in `jellyfin-media` wird von systemd intern
> als `\x2d` kodiert. Tippt man das direkt in die Bash, frisst die Shell den Backslash selbst auf
> (Backslash + `x` → nur `x`), und `systemctl` bekommt einen falschen Unit-Namen
> (`mnt-jellyfinx2dmedia.automount`) — Fehler `Unit ... not found`. `systemd-escape` erzeugt den
> korrekten Namen zuverlässig, unabhängig vom Pfad.

Ab jetzt reicht `cd /mnt/jellyfin-media` oder ein `rsync` dorthin — der Mount passiert
transparent im Hintergrund, kein manuelles Ein-/Aushängen mehr wie bisher.

### Zustand der Automount-Unit prüfen und zurücksetzen

```bash
systemctl status "$(systemd-escape --path --suffix=automount /mnt/jellyfin-media)"
```

Erwarteter Normalzustand: `Active: active (waiting)` — die Unit "lauert" auf den ersten Zugriff.

**Zwei bekannte Fallstricke:**

1. **`Active: inactive (dead)` statt `active (waiting)`**
   Passiert z. B., wenn ein vorheriger Start-Versuch fehlgeschlagen ist (siehe Punkt 2) und sich
   die Unit danach nicht von selbst neu aktiviert. In diesem Zustand passiert bei `ls
   /mnt/jellyfin-media` gar nichts, weil niemand auf den Zugriff wartet. Fix: einfach erneut
   starten (Befehl oben, `systemctl start ...`).

2. **`Job failed` / `Path ... is already mounted`**
   Passiert, wenn der Pfad zum Startzeitpunkt schon uneins (z. B. durch einen alten, manuell mit
   `mount -t nfs ...` gesetzten Mount, der nie mit `umount` beendet wurde) belegt ist — der
   Automount-Mechanismus kann sich nicht auf einen bereits aktiv gemounteten Pfad legen. Fix:
   ```bash
   umount /mnt/jellyfin-media
   mount | grep jellyfin-media   # sollte jetzt leer sein
   systemctl start "$(systemd-escape --path --suffix=automount /mnt/jellyfin-media)"
   ```

**Funktionstest (kompletter Zyklus):**

```bash
ls /mnt/jellyfin-media           # löst den Mount aus
mount | grep jellyfin-media      # sollte jetzt eine NFS-Zeile zeigen
sleep 600                        # 10 Min. warten (= idle-timeout)
mount | grep jellyfin-media      # sollte jetzt wieder leer sein (automatisch ausgehängt)
ls /mnt/jellyfin-media           # löst erneut aus
mount | grep jellyfin-media      # NFS-Zeile wieder da
```

> ✅ **Status (2026-07-09):** eingerichtet und verifiziert auf `tower01`. Automount-Unit lief
> zwischenzeitlich in `inactive (dead)` (nach einem fehlgeschlagenen Start-Versuch wegen bereits
> belegtem Pfad) — nach `umount` + erneutem `systemctl start` korrekt auf `active (waiting)`
> gewechselt, `ls`-Zugriff hat den Mount danach zuverlässig ausgelöst.

---

## 3. Workflow: von der Disc bis zur Jellyfin-Bibliothek

### Schritt 1 — Rippen

DVD einlegen, MakeMKV (Docker auf `tower01`) rippen lassen. Ergebnis landet lokal, z. B.:

```
/home/user/media/LITTLE_MISS_SUNSHINE/
├── B1_t00.mkv
└── C2_t01.mkv
```

MakeMKV rippt oft mehrere Titel derselben Disc (Hauptfilm + Extras/Trailer/Kapitel-Fragmente).

### Schritt 2 — Hauptfilm identifizieren

```bash
ls -lh /home/user/media/LITTLE_MISS_SUNSHINE/
```

Der Hauptfilm ist in der Regel die deutlich größere/längere Datei. Im Zweifel Laufzeit prüfen:

```bash
ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1 <datei>.mkv
```

### Schritt 3 — Kopieren und benennen

```bash
mkdir -p "/mnt/jellyfin-media/Filme/Little Miss Sunshine (2006)"

rsync -avh --progress \
  /home/user/media/LITTLE_MISS_SUNSHINE/B1_t00.mkv \
  "/mnt/jellyfin-media/Filme/Little Miss Sunshine (2006)/Little Miss Sunshine (2006).mkv"
```

Dank Automount (Abschnitt 2) muss vorher nichts manuell gemountet werden — der erste Zugriff auf
`/mnt/jellyfin-media` löst den Mount automatisch aus.

### Schritt 4 — Rohdaten aufräumen (optional)

Nicht benötigte Titel (Extras, Duplikate) lokal auf `tower01` löschen, sobald der Hauptfilm
erfolgreich auf dem NAS liegt:

```bash
rm -rf /home/user/media/LITTLE_MISS_SUNSHINE/
```

### Schritt 5 — Jellyfin-Bibliothek aktualisieren

Jellyfin scannt periodisch automatisch, für sofortiges Ergebnis manuell anstoßen:

*Dashboard → Bibliotheken → Alle Bibliotheken scannen*

---

## Offene Punkte / Ausblick

- Titel-Erkennung (Schritt 2) ist aktuell manuell — bei Bedarf später automatisierbar
  (z. B. per Skript, das die größte `.mkv`-Datei eines Rip-Ordners automatisch verschiebt),
  aber nicht vorgezogen, solange das Volumen überschaubar ist
