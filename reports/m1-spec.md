# Milestone M1 — First Working Prototype Spec

**Project:** MiraBridge (working name) — turn commodity hardware into a Mac → Miracast bridge.
**Companion to:** `diy-options-and-product-exploration.md`, `pi4-miracast-bridge-build-guide.md`
**Milestone owner:** _you_  ·  **Status:** not started

---

## 1. Goal (one sentence)

**On a bench (Pi with monitor/keyboard or SSH), get a MacBook's screen onto one Miracast sink via the bridge, with Mirror and Extend both working — once, reliably, on known hardware.**

This is a *proof it works at all*. Headless operation, auto-start, a pairing web-UI, and multi-TV support are **explicitly out of scope** (those are M2–M3).

---

## 2. Definition of done (acceptance criteria)

M1 is complete when **all** of these are demonstrated and recorded:

- [ ] **AC1 — Miracast leg alone:** The Pi's own desktop appears on the reference Miracast sink via GNOME Network Displays.
- [ ] **AC2 — Capture leg alone:** The MacBook detects the capture device as an external display, and its screen is visible on the Pi (e.g. via `mpv`).
- [ ] **AC3 — Full chain:** With the capture shown fullscreen on the Pi and GND casting, the **MacBook's screen is visible on the reference sink**.
- [ ] **AC4 — Mirror & Extend:** Both macOS display modes work (Extend at the capture device's native resolution, typically 1080p).
- [ ] **AC5 — Recovery:** Unplugging and replugging the Mac's HDMI restores the image **without** re-pairing the Miracast link.
- [ ] **AC6 — Recorded results:** Measured glass-to-glass latency, the working resolution/fps, and a known-issues log are written down (template in §7).

> Latency target is **informational only** for M1 (use case is slides/video). Record it; don't gate on it.

---

## 3. Reference hardware (exact BOM)

| Role | Exact part | Why this one | Approx |
|---|---|---|---|
| Board | **Raspberry Pi 4, 4 GB** | Has HW H.264 encoder; well-trodden. | $55 |
| Storage | 32 GB A1 microSD | | $8 |
| **Wi-Fi P2P adapter** | **Alfa AWUS036ACM** (MediaTek **MT7612U**) | **Mainline `mt76x2u` driver (kernel ≥4.19), no out-of-tree build; exposes `P2P-client`/`P2P-GO`.** | $35 |
| **HDMI capture** | **MS2130-based USB 3.0 capture** (e.g. generic "4K HDMI to USB3") | UVC/UAC, no drivers, 1080p60 uncompressed via `v4l2`. | $20 |
| Mac cable | USB-C → HDMI adapter + short HDMI | Feeds the capture device. | $12 |
| Pi bench output | Any HDMI monitor (or 1080p HDMI dummy plug) | GND casts the Pi's *desktop*, so the Pi needs a display/output. | — |
| Power | Official Pi 4 PSU (5V/3A) for the bench | Battery comes later (M2). | $10 |

**Reference Miracast sink (controllable, reproducible):** a cheap **MiraScreen / AnyCast-class Miracast dongle** plugged into any HDMI monitor. *(MiraScreen is in GND's tested-sink list, so it's a low-risk first target.)* **Also validate against the real target** (the school's display) once the reference sink works.

**Alternate Wi-Fi adapter** (if AWUS036ACM unavailable): an RTL8812AU dongle — but expect an out-of-tree DKMS driver (`morrownr/8812au-20210820`), so prefer the MT7612U for M1.

---

## 4. M1 architecture (Route A — uses GND as-is)

```
 MacBook ─USB-C→HDMI─► [MS2130 capture] ─USB3─► Pi 4 ─────────────► (Pi desktop)
   (sees external                              shows capture            │
    monitor: Mirror/Extend)                    fullscreen via mpv       │ GNOME Network Displays
                                                                        │ screencasts the desktop
                                                                        ▼
                                                              Wi-Fi Direct (Miracast)
                                                                        ▼
                                                             Reference Miracast sink
```

We deliberately reuse GND (which casts a *desktop*) rather than building the low-level `v4l2 → WFD` pipeline — that optimization is M3.

---

## 5. Bring-up steps (in order — each maps to an AC)

### 5.0 Base OS
1. Flash **Raspberry Pi OS (64-bit, with desktop)** via Raspberry Pi Imager; preset hostname, SSH, locale, your home Wi-Fi (onboard).
2. `sudo apt update && sudo apt full-upgrade -y`
3. Switch networking to **NetworkManager** (GND needs NM + wpa_supplicant, **not** iwd): `sudo raspi-config` → Advanced → Network Config → NetworkManager → reboot.

### 5.1 Wi-Fi P2P adapter (→ enables AC1)
1. Plug in the AWUS036ACM. Confirm driver + modes:
   ```bash
   dmesg | grep -i mt76
   iw list | grep -A8 "Supported interface modes"   # expect P2P-client and P2P-GO
   nmcli device                                       # expect the adapter + a p2p-dev-* entry
   ```
2. *(Pi 4 + MT7612U 5 GHz quirk):* if the P2P/AP link is unstable, disable Scatter-Gather on the adapter and/or prefer a 2.4 GHz operating channel (documented MT7612U-on-VL805 issue).

### 5.2 Capture device (→ AC2)
```bash
sudo apt install -y v4l-utils mpv
v4l2-ctl --list-devices
v4l2-ctl -d /dev/video0 --list-formats-ext    # confirm 1080p modes
mpv av://v4l2:/dev/video0 --profile=low-latency --untimed   # should show the Mac
```
At this point macOS shows the capture as a display — toggle **Mirror/Extend** in System Settings → Displays (banks AC4 later).

### 5.3 Miracast leg alone (→ AC1)
```bash
flatpak install -y flathub org.gnome.NetworkDisplays   # or: apt install gnome-network-displays
```
Run **GNOME Network Displays** in the Pi's desktop session → choose **Wi-Fi (Miracast)** → select the reference sink → accept any PIN/Allow on the sink. **Confirm the Pi desktop shows on the sink.**

> If GND errors with `No such interface ScreenCast`, the screencast portal backend is missing/incompatible. Fallback: run a **minimal GNOME-on-Wayland** session (which ships the portal GND targets) instead of the default Pi desktop. Log which worked.

### 5.4 Full chain (→ AC3, AC4, AC5)
1. Show the capture fullscreen on the Pi desktop:
   ```bash
   mpv av://v4l2:/dev/video0 --profile=low-latency --untimed --fullscreen --no-osc
   ```
2. With GND still casting, the **sink now shows the Mac**. Verify Mirror and Extend on the Mac (AC4).
3. Unplug/replug the Mac's HDMI; confirm the image returns without re-pairing GND (AC5).

---

## 6. Test plan

| Test | Method | Pass condition |
|---|---|---|
| Latency (informational) | Run an online ms stopwatch fullscreen on the Mac; photograph Mac + sink together; subtract. | Record the number. |
| Resolution/fps | `v4l2-ctl --list-formats-ext` + what the Mac reports for the external display. | Record actual (target 1080p). |
| Stability | Leave running 30 min on a static slide + a 1080p video clip. | No disconnect; no crash. |
| Recovery | Replug Mac HDMI 3× | Image returns each time, no GND re-pair. |
| Real sink | Repeat AC3 against the school display | Works, or failure mode documented. |

---

## 7. Results log (fill in)

```
Date:
Pi OS version / kernel:
GND version + install method (flatpak/apt):
Worked with default Pi desktop? (Y/N)  | If N, session used:
Wi-Fi adapter + driver (dmesg line):
P2P modes present? (Y/N):
Capture device + chip + /dev/videoN:
Capture format/fps used:
Reference sink (make/model):
AC1 ___ AC2 ___ AC3 ___ AC4(mirror) ___ AC4(extend) ___ AC5 ___
Measured latency (ms):
Real (school) sink result:
Top 3 issues hit + fixes:
```

---

## 8. Risk register & fallbacks

| Risk | Likelihood | Fallback |
|---|---|---|
| GND portal error on non-GNOME desktop | **High** | Minimal GNOME-on-Wayland autologin session; or move to low-level pipeline (M3). |
| MT7612U 5 GHz instability on Pi 4 (VL805) | Medium | Disable Scatter-Gather; force 2.4 GHz P2P channel. |
| Reference/real sink demands a PIN every connect | Medium | Pair manually in GND on the bench; note if sink offers "always allow." |
| Capture caps Extend resolution below desktop | Medium | Accept 1080p for M1; document. |
| School sink is Miracast-only AND blocks P2P region/channel | Low | Test the controllable MiraScreen sink first to isolate. |

---

## 9. Out of scope (defer)

- Headless / no-monitor operation, auto-start on boot, battery power → **M2**.
- First-pairing web-UI, auto-reconnect to last sink → **M2**.
- Low-level `v4l2src → v4l2h264enc → WFD/RTSP` pipeline (drop GND/portal dependency) → **M3**.
- Multi-TV compatibility matrix, prebuilt flashable image → **M3**.
- Optional pre-flashed CM4 device → **M4**.
