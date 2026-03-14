# Legacy NVIDIA GPU on Linux – Workarounds & Fixes

> 🌐 **Deutsch:** [brave-fermi-gpu-workaround_de.md](brave-fermi-gpu-workaround_de.md)

---

### The Problem

Many older laptops (ThinkPad W520, Dell Precision M4600, HP EliteBook 8560w, etc.) contain NVIDIA GPUs based on the **Fermi architecture** (e.g. Quadro 1000M, Quadro 2000M, GeForce 540M). NVIDIA has **ended proprietary driver support** for these GPUs:

| NVIDIA Driver Branch | Last supported architecture | End of support |
|---|---|---|
| 340.xx | Tesla/GT2xx (GeForce 8–300) | 2019 |
| 390.xx | Fermi (GeForce 400/500, Quadro x000M) | End of 2022 |

On modern Linux distributions (Ubuntu 22.10+, Pop!\_OS 24.04, Linux Mint 22+), **only the open-source Nouveau driver** is available for these GPUs. Nouveau has significant limitations:

- **No hardware video decode** (VA-API only returns `VAProfileNone`)
- Unstable OpenGL/EGL support with modern Chromium-based browsers
- GPU process crashes in Brave, Chrome, Edge, Electron apps

---

### What Happens in Brave (and other Chromium browsers)

When Brave starts with hardware acceleration enabled on Nouveau:

1. The GPU process attempts to initialize OpenGL via EGL/ANGLE
2. Nouveau cannot provide a compatible GL context → **GPU process crashes**
3. Brave detects the crash and **permanently disables GPU acceleration** for the session
4. All rendering falls back to software → menus flicker, load slowly or fail to open
5. YouTube and other video sites run entirely on CPU → **choppy playback**

You can verify this by visiting `brave://gpu` – you will see most features listed as "Software only" or "Disabled".

---

### Solution: Disable the NVIDIA GPU in BIOS

If your laptop supports **hybrid graphics** (Intel iGPU + NVIDIA dGPU), the cleanest solution is to disable the NVIDIA GPU entirely in the BIOS. This hands everything over to the Intel GPU, which has **full Mesa/VA-API support** on all modern Linux distributions.

**BIOS path (ThinkPad W520 example):**

    Config → Display → Graphics Device

| Option | Description |
|---|---|
| **Integrated Graphics** ✅ | Intel only – recommended |
| Discrete Graphics | NVIDIA only – causes the problems described |
| NVIDIA Optimus | Both GPUs, dynamic switching – unreliable on Linux with legacy drivers |

> ℹ️ The BIOS path varies by manufacturer. Common locations: *Display*, *Advanced*, *GPU Settings*, *Video*.

**After reboot, verify:**

    glxinfo | grep "OpenGL renderer"
    # Should show: Mesa Intel(R) HD Graphics ...

    vainfo
    # Should show multiple VAProfile entries (H264, MPEG2, VC1, ...)

---

### Install Brave as Native System App (APT – Recommended)

> ⚠️ **Do NOT install Brave as a Flatpak on this hardware.** The Flatpak sandbox blocks VA-API access to `/dev/dri`, preventing hardware video decode even with workaround flags. Always use the native APT installation.

    sudo curl -fsSLo /usr/share/keyrings/brave-browser-archive-keyring.gpg \
      https://brave-browser-apt-release.s3.brave.com/brave-browser-archive-keyring.gpg

    echo "deb [signed-by=/usr/share/keyrings/brave-browser-archive-keyring.gpg arch=amd64] \
      https://brave-browser-apt-release.s3.brave.com/ stable main" | \
      sudo tee /etc/apt/sources.list.d/brave-browser-release.list

    sudo apt update
    sudo apt install brave-browser

If you previously had Brave as a Flatpak, remove it first:

    flatpak uninstall com.brave.Browser

After installing the native version, **no flags configuration is needed** – hardware video decode works out of the box. You can verify at `brave://gpu` under *Video Acceleration Information*:

    Decode h264 baseline: 16x16 to 2048x2048 pixels
    Decode h264 main    : 16x16 to 2048x2048 pixels
    Decode h264 high    : 16x16 to 2048x2048 pixels

> 💡 To confirm hardware decoding is active on YouTube: open DevTools → three-dot menu → **More Tools → Media** → look for `VDAVideoDecoder` and `hardware_accelerated: true`.

---

### Required Packages (Intel VA-API)

    sudo apt install i965-va-driver vainfo

On some distributions `intel-media-va-driver` may be needed instead (for Gen 8+ / Broadwell and newer). For Sandy Bridge / Ivy Bridge (like the W520), `i965-va-driver` is the correct package.

---

### Affected Hardware (Examples)

- Lenovo ThinkPad W520 / W530 (Quadro 1000M / 2000M)
- Lenovo ThinkPad T420s (NVIDIA NVS 4200M)
- Dell Precision M4600 / M6600 (Quadro 1000M / 2000M)
- HP EliteBook 8560w (Quadro 1000M)
- Any laptop with GeForce 400M / 500M series

---

### If BIOS Disable Is Not Available

Some laptops do not offer an option to disable the dGPU. In that case:

**Use Firefox instead:**
Firefox uses a different GPU pipeline and is significantly more stable on Nouveau. Enable hardware decoding in `about:config`:

    media.hardware-video-decoding.force-enabled = true

**Force YouTube to use H.264:**
Install the browser extension **"h264ify"** or **"Enhanced H.264ify"**. This prevents YouTube from sending VP9/AV1 streams which are much harder to decode in software on older CPUs.

---

## License

CC BY 4.0 – [HannesMC](https://github.com/HannesMC)
