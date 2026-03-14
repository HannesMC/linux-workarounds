# Alte NVIDIA GPUs unter Linux – Workarounds & Lösungen

> 🌐 **English:** [brave-fermi-gpu-workaround.md](brave-fermi-gpu-workaround.md)

---

### Das Problem

Viele ältere Laptops (ThinkPad W520, Dell Precision M4600, HP EliteBook 8560w usw.) enthalten NVIDIA-GPUs auf Basis der **Fermi-Architektur** (z.B. Quadro 1000M, Quadro 2000M, GeForce 540M). NVIDIA hat den **proprietären Treibersupport** für diese GPUs eingestellt:

| NVIDIA-Treiberzweig | Letzte unterstützte Architektur | Support-Ende |
|---|---|---|
| 340.xx | Tesla/GT2xx (GeForce 8–300) | 2019 |
| 390.xx | Fermi (GeForce 400/500, Quadro x000M) | Ende 2022 |

Auf modernen Linux-Distributionen (Ubuntu 22.10+, Pop!\_OS 24.04, Linux Mint 22+) steht für diese GPUs **nur noch der Open-Source-Treiber Nouveau** zur Verfügung. Nouveau hat erhebliche Einschränkungen:

- **Keine Hardware-Videodekodierung** (VA-API liefert nur `VAProfileNone`)
- Instabile OpenGL/EGL-Unterstützung mit modernen Chromium-basierten Browsern
- GPU-Process-Abstürze in Brave, Chrome, Edge und Electron-Apps

---

### Was in Brave passiert (und anderen Chromium-Browsern)

Wenn Brave mit aktivierter Hardware-Beschleunigung unter Nouveau startet:

1. Der GPU-Process versucht OpenGL über EGL/ANGLE zu initialisieren
2. Nouveau kann keinen kompatiblen GL-Kontext bereitstellen → **GPU-Process stürzt ab**
3. Brave erkennt den Absturz und **deaktiviert GPU-Beschleunigung** für die gesamte Sitzung
4. Alles wird per Software gerendert → Menüs flackern, laden langsam oder öffnen sich gar nicht
5. YouTube und andere Videoseiten laufen komplett auf der CPU → **ruckeliges Abspielen**

Dies lässt sich über `brave://gpu` nachvollziehen – die meisten Features erscheinen dort als "Software only" oder "Disabled".

---

### Lösung: NVIDIA-GPU im BIOS deaktivieren

Wenn das Laptop **hybride Grafik** unterstützt (Intel iGPU + NVIDIA dGPU), ist die sauberste Lösung, die NVIDIA-GPU im BIOS vollständig zu deaktivieren. Damit übernimmt die Intel-GPU alles – und die hat auf allen modernen Linux-Distributionen **vollständige Mesa/VA-API-Unterstützung**.

**BIOS-Pfad (Beispiel ThinkPad W520):**

    Config → Display → Graphics Device

| Option | Beschreibung |
|---|---|
| **Integrated Graphics** ✅ | Nur Intel – empfohlen |
| Discrete Graphics | Nur NVIDIA – verursacht die beschriebenen Probleme |
| NVIDIA Optimus | Beide GPUs, dynamische Umschaltung – unter Linux mit Legacy-Treibern unzuverlässig |

> ℹ️ Der BIOS-Pfad variiert je nach Hersteller. Typische Stellen: *Display*, *Advanced*, *GPU Settings*, *Video*.

**Nach dem Neustart prüfen:**

    glxinfo | grep "OpenGL renderer"
    # Sollte anzeigen: Mesa Intel(R) HD Graphics ...

    vainfo
    # Sollte mehrere VAProfile-Einträge zeigen (H264, MPEG2, VC1, ...)

---

### Brave als native System-App installieren (APT – empfohlen)

> ⚠️ **Brave NICHT als Flatpak installieren auf dieser Hardware.** Die Flatpak-Sandbox blockiert den VA-API-Zugriff auf `/dev/dri` und verhindert Hardware-Videodekodierung – auch mit Workaround-Flags. Immer die native APT-Installation verwenden.

    sudo curl -fsSLo /usr/share/keyrings/brave-browser-archive-keyring.gpg \
      https://brave-browser-apt-release.s3.brave.com/brave-browser-archive-keyring.gpg

    echo "deb [signed-by=/usr/share/keyrings/brave-browser-archive-keyring.gpg arch=amd64] \
      https://brave-browser-apt-release.s3.brave.com/ stable main" | \
      sudo tee /etc/apt/sources.list.d/brave-browser-release.list

    sudo apt update
    sudo apt install brave-browser

Falls Brave vorher als Flatpak installiert war, dieses zuerst entfernen:

    flatpak uninstall com.brave.Browser

Nach der nativen Installation ist **keine Flags-Konfiguration notwendig** – Hardware-Videodekodierung funktioniert direkt. Überprüfbar unter `brave://gpu` im Abschnitt *Video Acceleration Information*:

    Decode h264 baseline: 16x16 to 2048x2048 pixels
    Decode h264 main    : 16x16 to 2048x2048 pixels
    Decode h264 high    : 16x16 to 2048x2048 pixels

> 💡 Hardware-Dekodierung auf YouTube bestätigen: DevTools öffnen → Drei-Punkte-Menü → **More Tools → Media** → dort nach `VDAVideoDecoder` und `hardware_accelerated: true` suchen.

---

### Benötigte Pakete (Intel VA-API)

    sudo apt install i965-va-driver vainfo

Auf manchen Distributionen wird stattdessen `intel-media-va-driver` benötigt (ab Gen 8 / Broadwell und neuer). Für Sandy Bridge / Ivy Bridge (wie beim W520) ist `i965-va-driver` das richtige Paket.

---

### Betroffene Hardware (Beispiele)

- Lenovo ThinkPad W520 / W530 (Quadro 1000M / 2000M)
- Lenovo ThinkPad T420s (NVIDIA NVS 4200M)
- Dell Precision M4600 / M6600 (Quadro 1000M / 2000M)
- HP EliteBook 8560w (Quadro 1000M)
- Alle Laptops mit GeForce 400M / 500M Serie

---

### Falls BIOS-Deaktivierung nicht möglich ist

Manche Laptops bieten keine Option zur Deaktivierung der dGPU. In diesem Fall:

**Firefox verwenden:**
Firefox nutzt eine andere GPU-Pipeline und ist unter Nouveau deutlich stabiler. Hardware-Dekodierung in `about:config` aktivieren:

    media.hardware-video-decoding.force-enabled = true

**YouTube zu H.264 zwingen:**
Die Browser-Extension **"h264ify"** oder **"Enhanced H.264ify"** installieren. Damit wird verhindert, dass YouTube VP9/AV1-Streams liefert, die auf älteren CPUs in Software deutlich schwerer zu dekodieren sind.

---

## Lizenz

CC BY 4.0 – [HannesMC](https://github.com/HannesMC)
