# BIOS-Konfiguration — HP ProDesk 600 G4

BIOS öffnen: **F10** beim Boot.

## Kritische Einstellungen

| Einstellung | Pfad im BIOS | Wert |
|---|---|---|
| **After Power Loss** | Advanced → Power Management | **Power On** |
| **SATA Controller Mode** | Advanced → System Options | **AHCI** |

**After Power Loss → Power On** ist die wichtigste Einstellung: ohne sie startet der NAS
nach einem Stromausfall nicht automatisch neu.

AHCI ist auf den meisten Systemen bereits Standard — prüfen lohnt sich trotzdem,
falsche SATA-Modi (RAID, IDE) können Performance kosten oder Linux-Probleme verursachen.

## Empfohlene Einstellungen

| Einstellung | Pfad im BIOS | Wert |
|---|---|---|
| **Wake on LAN** | Advanced → Device Options | **Enabled** |
| **S4/S5 Wake on LAN** | Advanced → Power Management | **Enabled** |
| **Fast Boot** | Boot Options | **Disabled** |

Wake on LAN erlaubt das Einschalten des NAS per Netzwerk.
Fast Boot deaktivieren macht Troubleshooting einfacher (POST-Meldungen sichtbar).

## Für die Debian-Installation

| Einstellung | Pfad im BIOS | Wert |
|---|---|---|
| **Boot Order** | Boot Options | USB an erste Stelle |
| **Secure Boot** | Security | **Disabled** |
| **Legacy Support** | Boot Options | **Disabled** (UEFI reicht) |

Nach der Installation Boot Order wieder auf die interne SSD setzen.

## Nicht ändern

- **Virtualization Technology (VT-x)** — aktiviert lassen
- **Hyperthreading** — aktiviert lassen
- **C-States / CPU Power Management** — Standard lassen, Linux regelt das selbst
