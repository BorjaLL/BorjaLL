<!-- SPDX-License-Identifier: GPL-3.0-or-later -->
# Architecture

## The problem in one diagram

```
 MacBook ─USB-C→HDMI─► [USB HDMI capture] ─USB─► MiraBridge ─Wi-Fi Direct (Miracast)─► TV sink
   no software           UVC/V4L2 device          Linux board                    TV's built-in
   (sees a monitor)      (Mac = "monitor")        capture→encode→Miracast        Miracast receiver
```

Two hops, two reasons:
1. **Mac → box (wired capture).** macOS can't send Miracast and has no Wi-Fi Direct API, but it *will* drive any HDMI monitor. The capture device presents EDID, so the Mac sees a normal display → **Mirror/Extend both work**. The box receives it as a `v4l2` source.
2. **Box → TV (Miracast).** The box is the Miracast **source** over Wi-Fi Direct to the TV's sink.

## Why a separate box (not a Mac app)

macOS exposes **no public Wi-Fi Direct / P2P API**, so a Mac cannot be a Miracast source in software. The bridge moves that job onto Linux, where `wpa_supplicant` provides Wi-Fi Direct P2P.

## Runtime, current (M1) — reuse GNOME Network Displays

GND casts a **desktop/monitor** via the screencast portal + PipeWire. So M1 shows the capture **fullscreen** on the box's desktop and lets GND cast that. Simple, but:
- Needs a graphical session + screencast portal (**not headless** — that's why M2 adds an autologin session, and M3 replaces this).

## Runtime, target (M3) — low-level pipeline

Drop GND/portal and run the source directly:

```
v4l2src (capture) → v4l2h264enc (Pi 4 HW encode) → mpegtsmux → RTP/UDP  ──►  TV (WFD/RTSP)
        Wi-Fi Direct group + RTSP capability negotiation via wpa_supplicant / WFD
```

This is appliance-friendly (no desktop) but is the hardest engineering work; building blocks: MiracleCast `wifid` (P2P), a GStreamer WFD/RTSP source.

## Components & where they live

| Concern | Tech | Path |
|---|---|---|
| Wi-Fi Direct P2P | `wpa_supplicant` (CONFIG_P2P) + NetworkManager | `pipeline/` |
| Capture | `uvcvideo` / `v4l2` | `pipeline/` |
| Encode + transport (M3) | GStreamer (`v4l2h264enc`, `mpegtsmux`, RTP) | `pipeline/` |
| Miracast source (M1) | GNOME Network Displays | (external dep) |
| Pairing / status UI (M2) | small web app | `webui/` |
| Image build (M3) | pi-gen / scripts | `image/` |

## Known hard parts (see issues)
- GND requires a screencast portal → not headless (M1 limitation).
- Wi-Fi Direct hardware support is uneven → pinned adapter shortlist.
- Some sinks require a PIN per connect → pairing UX (M2).
