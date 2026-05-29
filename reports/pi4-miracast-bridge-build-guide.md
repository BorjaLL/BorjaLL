# Build Guide: Portable Pi 4 → Miracast TV Bridge for a MacBook

**Companion to:** `miracast-macbook-to-tv.md`
**Goal:** A pocket-pouch device that takes the MacBook's video and shows it on a **Miracast-only TV** — nothing installed on the Mac, nothing plugged into the TV.
**Use case assumed:** slides / video / browsing (latency-tolerant).

---

## 0. Read this first — honest expectations

This is a **DIY, experimental** build, not a polished product. Two hard truths shape everything below:

1. **The open-source Miracast *source* (GNOME Network Displays, "GND") needs a graphical session.** It uses the desktop ScreenCast portal (xdg-desktop-portal + PipeWire). It will **not** run on a truly headless Pi — so the appliance runs a *minimal auto-login Wayland session* whose only job is to display the captured Mac screen and cast it.
2. **The TV always needs a manual nudge.** Every session you must put the TV into "Screen Mirroring / Screen Share" mode with its remote, and on first pairing the TV usually shows a **PIN or "Allow" prompt**. There is no way around this with Miracast — it's how the protocol works.

So the realistic end-user experience is **"a couple of remote-control taps + plug in a cable,"** not "zero interaction." That's still far smaller than carrying a second laptop.

**Before buying everything, do Phase 1 (the pairing test).** If your specific TV won't pair, nothing else matters.

---

## 1. Parts list

| Part | Recommended | Notes |
|---|---|---|
| Board | **Raspberry Pi 4 (4 GB)** | Has a **hardware H.264 encoder** (Pi 5 removed it); 5V/3A power is battery-friendly. |
| Storage | 32 GB+ A1 microSD (or USB SSD) | |
| **Wi-Fi Direct adapter** | **USB adapter with RTL8812AU / RTL8814AU or MediaTek MT7612U** | **The make-or-break part.** Must do Wi-Fi Direct P2P with `wpa_supplicant`. Onboard Pi Wi-Fi P2P is unreliable — use this for the Miracast link. |
| **HDMI capture** | **USB 3.0 capture (MS2130 chipset) or Elgato Cam Link** | Presents EDID so the Mac sees a real monitor. Avoid the cheapest MS2109 sticks (1–2 s lag, 1080p30). |
| Cable | **USB-C → HDMI** cable/adapter (Mac side) + short HDMI (into capture) | |
| Power | **USB-C PD power bank** (≥ 5V/3A) | For true portability. |
| Case | Any Pi 4 case with USB access | |

Two USB peripherals hang off the Pi: the **capture dongle** and the **Wi-Fi P2P adapter**.

---

## 2. The data path

```
 MacBook ──USB-C→HDMI──► [USB HDMI capture] ──USB──► Pi 4 ──Wi-Fi Direct (Miracast)──► TV
   (sees a normal              (UVC / V4L2          (fullscreen view of capture,
    external monitor:           video device)        cast by GNOME Network Displays
    Mirror OR Extend)                                over the USB Wi-Fi P2P adapter)
```

Because the Mac treats the capture dongle as a genuine external display, **Mirror vs Extend is chosen on the Mac** in System Settings → Displays, just like any monitor. The bridge transports whatever the Mac sends.

---

## 3. Phase 1 — Validate TV pairing FIRST (cheapest possible test)

Do this on **any Linux machine you already have** (laptop dual-boot, a live USB, or the Pi later) **+ the USB Wi-Fi P2P adapter**. You're only testing whether GND can drive *your* TV.

1. Put the TV into screen-mirroring mode (see §7 per brand).
2. On Linux (GNOME session), install GND:
   ```bash
   flatpak install flathub org.gnome.NetworkDisplays
   ```
   (or `sudo apt install gnome-network-displays` on Debian/Ubuntu, `sudo dnf install gnome-network-displays` on Fedora).
3. Plug in the USB Wi-Fi P2P adapter. Confirm P2P capability:
   ```bash
   iw list | grep -A6 "Supported interface modes"   # expect "P2P-client" and "P2P-GO"
   ```
4. Launch **GNOME Network Displays**, pick **Wi-Fi (Miracast)**, select your TV, accept the PIN/allow prompt on the TV.

**Decision gate:** if the desktop shows on the TV → proceed. If not (no P2P adapter found, no sink discovered, or pairing fails) → fix the Wi-Fi adapter / TV mode before spending more. This is the single most likely failure point.

---

## 4. Phase 2 — Raspberry Pi 4 base setup

1. Flash **Raspberry Pi OS (64-bit, "with desktop")** using Raspberry Pi Imager. In Imager's advanced options set hostname, enable SSH, set Wi-Fi/locale.
2. First boot, then update:
   ```bash
   sudo apt update && sudo apt full-upgrade -y
   ```
3. Ensure NetworkManager is the network backend (GND requires NM + wpa_supplicant, **not** iwd):
   ```bash
   sudo raspi-config        # Advanced Options → Network Config → NetworkManager
   sudo reboot
   ```
4. Verify `wpa_supplicant` has P2P support (Debian's build normally does). Plug in the USB Wi-Fi adapter and check:
   ```bash
   iw list | grep -A6 "Supported interface modes"   # look for P2P-client / P2P-GO
   nmcli device                                       # the adapter should appear, plus a p2p-dev-* device
   ```
   If your adapter needs an out-of-tree driver (common for RTL8812AU/8814AU), install via DKMS:
   ```bash
   sudo apt install -y dkms git build-essential raspberrypi-kernel-headers
   git clone https://github.com/morrownr/8812au-20210820   # match to your chipset
   cd 8812au-20210820 && sudo ./install-driver.sh
   ```

---

## 5. Phase 3 — Capture device

1. Plug the Mac (via USB-C→HDMI) into the capture dongle, capture into the Pi's USB 3.0 port.
2. On the Pi:
   ```bash
   sudo apt install -y v4l-utils
   v4l2-ctl --list-devices
   v4l2-ctl -d /dev/video0 --list-formats-ext   # confirm 1080p resolutions/fps
   ```
3. Quick visual test (over the Pi's HDMI or VNC):
   ```bash
   sudo apt install -y mpv
   mpv av://v4l2:/dev/video0 --profile=low-latency --untimed
   ```
   You should see the Mac's screen. (At this point macOS already lets you pick **Mirror/Extend** for this "monitor.")

---

## 6. Phase 4 — Integration (the experimental part)

GND casts a **graphical session's output**, so the appliance runs a tiny Wayland session that shows the capture fullscreen, and GND casts that.

**Route A — GND on a minimal session (recommended first; uses GND as-is):**

1. Install GND + a lightweight Wayland compositor and the portal stack:
   ```bash
   sudo apt install -y gnome-network-displays \
       labwc wlr-randr xdg-desktop-portal xdg-desktop-portal-wlr \
       pipewire wireplumber gstreamer1.0-plugins-{good,bad,ugly} mpv
   ```
   *(On Raspberry Pi OS, `xdg-desktop-portal-wlr` provides the ScreenCast portal for wlroots compositors like labwc. If GND rejects it, fall back to a minimal GNOME session, which ships the portal GND is built against.)*
2. Auto-login to that session and, on start, launch the fullscreen capture viewer:
   ```bash
   # ~/.config/labwc/autostart
   mpv av://v4l2:/dev/video0 --profile=low-latency --untimed --fullscreen --no-osc &
   gnome-network-displays &
   ```
3. In GND, choose the TV (Wi-Fi/Miracast). It screencasts the session output (the fullscreen Mac capture) to the TV.

> ⚠️ **This is the fiddly bit.** GND + portal + PipeWire on a non-GNOME/headless Pi is the least-proven link (see the report's risk section). Budget iteration here. If the wlroots portal won't satisfy GND, use a minimal GNOME-on-Wayland autologin session instead.

**Route B — Low-level pipeline (cleaner appliance, more custom):** skip GND/portal entirely. Use `wpa_supplicant`/`wpa_cli` P2P to form the Wi-Fi Direct group with the TV, then a GStreamer WFD/RTSP source pushing `v4l2src → v4l2h264enc (Pi 4 HW encode) → mpegtsmux → RTP`. Building blocks: MiracleCast's `wifid` (P2P management) + a WFD GStreamer source (e.g. `gst-rtsp-server-wfd`). This is genuinely DIY/research-grade today; pursue only if Route A's portal issues prove intractable.

---

## 7. Phase 5 — Make it portable & "appliance-like"

- **Auto-start on boot:** enable autologin to the session above (`raspi-config` → System → Boot/Auto Login → Desktop Autologin), with the autostart entries from Phase 4.
- **Remember the TV:** GND reconnects faster to a previously paired sink; do the first PIN pairing once at home.
- **No keyboard/screen at the venue:** once configured, it boots straight into the cast attempt. Optional: a tiny status LED script, or SSH from your phone for diagnostics.
- **Power:** run from the USB-C battery; Pi 4 needs ≥ 5V/3A.

---

## 8. Per-TV: putting the TV into receive mode (§ referenced above)

- **Samsung:** Source/Home → **Screen Mirroring**, or Settings → General → Network → Screen Mirroring. May prompt **Allow/PIN** on first connect.
- **LG:** **Screen Share** app, or Home Dashboard → Screen Share. Confirm the popup on the TV.
- **Sony / Android TV:** **Screen mirroring** input (or the "Wi-Fi Direct/Cast" tile).
- **Generic Miracast dongles built into the TV:** select the "Wireless Display / Miracast" input.

---

## 9. End-to-end workflow & timing (non-technical user)

After the one-time setup above, here's the actual sequence and timing at a venue:

| Step | Action | Time |
|---|---|---|
| 1 | **Power on the box** from the battery (or leave it on). It boots and auto-starts. | ~45–60 s (once) |
| 2 | **TV: switch to Screen Mirroring** mode with the remote (§8). | ~10–20 s |
| 3 | The box **auto-connects** to the TV over Wi-Fi Direct. First time: **accept the Allow/PIN prompt** on the TV. | ~5–20 s |
| 4 | **Plug the cable into the Mac** (USB-C→HDMI→capture). The Mac instantly detects an external display. | 1–2 s |
| 5 | On the Mac: **Mirror or Extend** (System Settings → Displays; it can remember the choice). | instant |
| 6 | **The TV shows the Mac.** Glass-to-glass delay ~0.3–1 s — fine for slides/video. | — |

**Cold start total: ~1–2 minutes**, dominated by the TV mode-switch and the Wi-Fi Direct handshake. Once connected, it stays up; you can unplug/replug the Mac freely (step 4–5) without re-pairing.

**What the user actually does, in plain terms:**
> "Press the TV remote's screen-mirroring button, plug the little box's cable into my Mac, and pick Mirror or Extend like I would with any monitor."

**Mirror vs Extend:** fully your choice on the Mac — the bridge doesn't care. Extend works because the Mac sees a true second monitor (at the capture device's resolution, typically 1080p).

---

## 10. Honest caveats

- **Experimental software.** GND + the Wayland portal on a Pi is the least-proven link; expect setup iteration and occasional reconnect retries.
- **TV interaction is unavoidable** (mode switch every time; PIN/allow at least once).
- **Resolution** is capped by the capture device (commonly 1080p) — the extended display won't exceed that.
- **Not for gaming/interactive** — the double encode adds delay (irrelevant for the stated slides/video use).
- **Two USB devices + power + a cable** = a small pouch, not a single sleek dongle.
