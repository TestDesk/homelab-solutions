# Debian Kernel-Versionsnummern verstehen

**Version:** Mai 2026  
**YunoHost:** 12.1.39 (stable)  
**Debian:** 12 Bookworm

---

## Das Problem

Wenn Debian eine Sicherheitslücke im Linux-Kernel patcht, veröffentlicht es die gepatchten Versionen im [Debian Security Tracker](https://security-tracker.debian.org) als **Paketversionen** – zum Beispiel `6.1.170-3`.

Der übliche Befehl um die eigene Kernel-Version zu prüfen lautet:

```bash
uname -r
```

Ausgabe:
```
6.1.0-47-amd64
```

Wer jetzt `6.1.0-47` mit `6.1.170-3` vergleicht, denkt er ist noch verwundbar – ist er aber nicht. Die beiden Nummern beschreiben dasselbe, sprechen aber eine völlig andere Sprache.

---

## Zwei verschiedene Versionsschemata

### Kernel-Build-Name (`uname -r`)

```
6.1.0-47-amd64
│   │  │   └── Architektur (amd64 = 64-Bit)
│   │  └────── Build-Nummer (von Debian vergeben)
│   └───────── Patch-Level des Upstream-Kernels
└───────────── Upstream-Kernel-Version (vom Linux-Projekt)
```

Diese Nummer kommt vom Linux-Projekt und Debian kombiniert – sie sagt nichts über den Sicherheitsstatus aus.

### Debian-Paketversion (`dpkg`)

```
6.1.170-3
│   │   └── Debian-Revision (wie oft Debian das Paket überarbeitet hat)
│   └─────── Upstream-Version des Kernels
└─────────── Haupt-Kernel-Version
```

Diese Nummer ist die offizielle Debian-Paketversion – sie steht im Security Tracker und ist der richtige Vergleichswert.

---

## Die Zuordnung

Die Verbindung zwischen beiden Nummern sieht man mit:

```bash
dpkg -l | grep linux-image
```

Beispiel-Ausgabe:

```
ii  linux-image-6.1.0-42-amd64    6.1.159-1    amd64    Linux 6.1 for 64-bit PCs (signed)
ii  linux-image-6.1.0-47-amd64    6.1.170-3    amd64    Linux 6.1 for 64-bit PCs (signed)
ii  linux-image-amd64              6.1.170-3    amd64    Linux for 64-bit PCs (meta-package)
```

| Kernel-Build-Name | Paketversion |
|-------------------|-------------|
| `6.1.0-42-amd64` | `6.1.159-1` |
| `6.1.0-47-amd64` | `6.1.170-3` |

`6.1.0-47-amd64` = Paket `6.1.170-3` – beide gehören zusammen.

---

## Richtig prüfen

Nicht `uname -r` mit dem Security Tracker vergleichen, sondern die Paketversion direkt prüfen:

```bash
apt-cache policy linux-image-amd64
```

Ausgabe:
```
linux-image-amd64:
  Installed: 6.1.170-3
  Candidate: 6.1.170-3
```

Wenn `Installed` und `Candidate` identisch sind, ist das System auf dem neuesten Stand. Den Wert dann mit dem Security Tracker vergleichen – nicht `uname -r`.

---

## Warum erklärt das niemand?

Debian dokumentiert diese Zuordnung nirgends direkt. Sie ist nur indirekt aus der `dpkg`-Ausgabe ablesbar. Security-Advisories – auch von YunoHost – verweisen auf `uname -r` ohne auf diesen Unterschied hinzuweisen, was zu unnötiger Verwirrung führt.
