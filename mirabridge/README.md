<!-- SPDX-License-Identifier: GPL-3.0-or-later -->
# MiraBridge

**Turn a cheap Linux board into a Mac → Miracast bridge.**
Cast a MacBook's screen to a **Miracast-only** display — with **nothing installed on the Mac** and **nothing plugged into the TV**.

> ⚠️ **Status: pre-alpha / experimental.** This is a young project. The core Miracast-source path works but is rough. See the [Roadmap](./ROADMAP.md) — we are at **M1** (first working prototype).

---

## Why this exists

macOS has **no Miracast support** (Apple uses AirPlay) and **no public Wi-Fi Direct API**. So a Mac cannot talk to a Miracast-only screen — the kind found in some schools, offices, hotels, and on older wireless displays you can't modify.

Existing tools don't fill this gap:
- Commercial "wireless HDMI" kits ship their **own** receiver — they don't drive a third-party Miracast TV.
- Open-source Miracast tools are mostly **sinks** (receivers); the **source** side is experimental.

**MiraBridge** is a small box between the Mac and the TV that does the Miracast for the Mac.

## How it works

```
 MacBook ─USB-C→HDMI─► [USB HDMI capture] ─USB─► MiraBridge box ─Wi-Fi Direct (Miracast)─► TV
   (sees a normal external monitor:                 (Linux board: capture →
    Mirror OR Extend, your choice)                   H.264 encode → Miracast source)
```

The capture device makes the Mac think it's a **normal external monitor**, so **Mirror and Extend both work** natively in macOS. The box re-transmits that video to the TV's built-in Miracast receiver.

## Before you build: check the free shortcut first

If your target display is a modern smart TV or an **enterprise/EDU receiver (e.g. ScreenBeam)**, it probably already accepts **AirPlay** — in which case your Mac connects natively and **you don't need MiraBridge at all.** Test that first (Control Center → Screen Mirroring). MiraBridge is for the **Miracast-only, can't-touch-the-display** case.

## Quick start

> Full build steps live in [`docs/`](./docs/). Short version:

1. Get the [reference hardware](./docs/hardware-compatibility.md) (Raspberry Pi 4 + MT7612U Wi-Fi adapter + MS2130 USB3 capture).
2. Flash the image (see [`image/`](./image/) — _not published yet; M3_).
3. Plug capture → box, Mac → capture (USB-C→HDMI).
4. Put the TV in screen-mirroring mode; pair once.
5. On the Mac, pick **Mirror** or **Extend**.

## Project layout

| Path | What |
|---|---|
| [`docs/`](./docs/) | Architecture, build guide, hardware compatibility matrix |
| [`pipeline/`](./pipeline/) | wpa_supplicant / GStreamer / capture configs (the runtime) |
| [`image/`](./image/) | Scripts to build the flashable SD image (M3) |
| [`webui/`](./webui/) | First-pairing & status web UI (M2) |
| [`ROADMAP.md`](./ROADMAP.md) | Milestones M1–M4 |
| [`CONTRIBUTING.md`](./CONTRIBUTING.md) | How to help (esp. hardware/TV compatibility reports) |

## How you can help

The single most useful contribution right now is a **compatibility report**: what TV/sink + Wi-Fi adapter you tried and whether it worked. Open a [compatibility report issue](./.github/ISSUE_TEMPLATE/compatibility_report.md).

## License

[GPL-3.0-or-later](./LICENSE) — chosen for compatibility with the GPL components this builds on (GNOME Network Displays, wpa_supplicant, GStreamer).

## Acknowledgements / prior art

[GNOME Network Displays](https://gitlab.gnome.org/GNOME/gnome-network-displays) (Miracast source), [MiracleCast](https://github.com/albfan/miraclecast) (sink + P2P daemon), the `mt76` driver project, and the `morrownr` USB-Wi-Fi driver maintainers.
