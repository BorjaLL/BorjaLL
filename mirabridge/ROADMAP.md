<!-- SPDX-License-Identifier: GPL-3.0-or-later -->
# Roadmap

MiraBridge is built in milestones, each shippable and testable on its own.

## M1 — First working prototype  ⬅️ current
**Goal:** On a bench, get a Mac's screen onto one Miracast sink via the box, Mirror + Extend both working. Reuses GNOME Network Displays (casts a desktop); no headless, no UI.
- Full spec: see `../reports/m1-spec.md`.
- Exit criteria: AC1–AC6 in the spec all demonstrated and logged.

## M2 — Appliance behavior
**Goal:** Make it usable without a keyboard/monitor.
- Auto-start the capture+cast on boot.
- Auto-reconnect to the last-used sink.
- First-pairing **web UI** (phone-friendly) to pick a sink and enter a TV PIN.
- Battery/USB-C power validated.

## M3 — Robust & reproducible
**Goal:** Anyone can flash and run it; works across hardware.
- Low-level pipeline (`v4l2src → v4l2h264enc → WFD/RTSP`) to drop the desktop-portal dependency.
- Prebuilt **flashable image** (`image/`).
- **Compatibility matrix** (TVs × Wi-Fi adapters × capture devices) in `docs/hardware-compatibility.md`.
- Bundled drivers for the supported Wi-Fi adapter shortlist.

## M4 — Optional hardware (stretch)
**Goal:** A tidier device for non-builders.
- Evaluate a **Raspberry Pi CM4 carrier** with onboard HDMI-in + vetted Wi-Fi (open hardware).
- Regulatory + (optional) Wi-Fi Alliance considerations — see `../reports/diy-options-and-product-exploration.md`.
- Only after M3 demonstrates real demand.

## Explicitly NOT goals
- A macOS app or kext (macOS has no Wi-Fi Direct API — that's why this is a separate box).
- Replacing AirPlay where it already works (use native AirPlay instead).
- Low-latency gaming (the capture + double-encode adds delay).
