# DIY Options, Budget Tiers & "Is There a Product Here?" — Mac → Miracast

**Companion to:** `miracast-macbook-to-tv.md`, `pi4-miracast-bridge-build-guide.md`
**Purpose:** Explore DIY approaches at different budgets, and honestly assess the "Kickstarter dongle" and "open-source hardware" ideas.

---

## 0. The one-sentence reframe

**The missing piece is *software*, not *hardware*.** Commodity boards (Pi, mini-PC) and HDMI capture dongles already exist; what doesn't exist is a **polished, plug-and-play image that turns them into a "Mac → Miracast" appliance with auto-pairing.** That makes a *software/appliance* project far more tractable — and more likely to actually fill the need — than a custom hardware Kickstarter.

---

## 1. Is the need real? (Honest market reality)

**The gap is real but narrow.** Here's the precise unmet need:

> "Send a **Mac's** screen to a **Miracast-only** sink that you **can't add hardware to**."

Why it's niche:
- **Most TVs/receivers aren't Miracast-only.** Modern smart TVs and especially **school/enterprise receivers (ScreenBeam, etc.) are multi-protocol — they already accept AirPlay** from Macs (see `school-connection-checklist.md`). In those cases the Mac just connects natively; no product needed.
- **Where you *can* touch the display**, a $15–60 multi-protocol receiver (or a wireless-HDMI kit) solves it.
- So the true addressable need shrinks to: **Miracast-only displays you don't control** — e.g. some corporate/education/hotel/rental setups, or older Miracast-only panels. That's a *thin* market.

**What genuinely doesn't exist commercially:** an **HDMI-input device that acts as a Miracast *source* to a third-party (TV's own) Miracast sink.** Every "wireless HDMI transmitter" on the market ([SC02](https://proscreencast.com/products/sc02-wireless-hdmi-transmitter-and-receiver-kit), [JTECH WDEX](https://www.jtechdigital.com/products/jtech-wdex-50m2-jtd-2867-1080p-wireless-hdmi-extender-w-airplay-smart-view-screen-mirroring-miracast-98ft)) ships with **its own paired receiver** — it does *not* drive an arbitrary Miracast TV. That's the hole. But note the hole is small *because* the workaround (add a cheap multi-protocol receiver) is usually available.

**Takeaway:** Validate the AirPlay shortcut first. If your school's receivers do AirPlay, the "need" largely evaporates for you personally — but the general gap (Mac → Miracast-only, no display access) remains for a niche, and is interesting as an OSS project more than a mass-market product.

---

## 2. DIY budget tiers

| Tier | Cost | What you build | Pros | Cons |
|---|---|---|---|---|
| **T0 — Software / existing gear** | **$0** | Use the AirPlay shortcut (if the sink supports it), or repurpose a **spare laptop running Linux** + GNOME Network Displays + a capture dongle you own. | Free; no new hardware. | Only works if sink does AirPlay, or you already have a Linux-capable spare machine. |
| **T1 — Minimalist bridge** | **~$40–60** | Used **Pi 4** + cheap **MS2109** capture + budget **RTL8812AU** Wi-Fi dongle + existing power. | Cheapest real bridge. | MS2109 adds 1–2 s lag, 1080p30; fiddlier; flaky. |
| **T2 — Recommended portable kit** ⭐ | **~$90–140** | **Pi 4 (4GB)** + **USB3 MS2130 / Cam Link** capture + **MT7612U/RTL8812AU** Wi-Fi + USB-C power bank + case. | Portable, HW H.264 encode, best $/reliability. | Still a DIY kit; experimental software. |
| **T3 — Reliability appliance** | **~$160–260** | **Intel N100 mini-PC** + USB3 capture + Wi-Fi P2P adapter. | QuickSync encode, mature x86 drivers, most reliable. | Bulkier, needs mains power, not pocketable. |
| **T4 — Custom hardware / product** | **$$$ (thousands NRE)** | A purpose-built board: HDMI-in chip + SoC + Wi-Fi → Miracast-source firmware. | Single sleek device; productizable. | Real embedded HW + firmware + RF + regulatory; see §4. |

**For your stated goal (small, portable, slides/video): T2 (the Pi 4 kit) is the sweet spot.** T0 wins if the AirPlay shortcut pans out.

---

## 3. ⭐ Best path if you want to *build something for others*: an open-source software appliance

Rather than designing hardware, ship a **flashable image / installer** ("plug commodity parts together, flash this, done"). This is the high-leverage gap.

**Product concept ("MiraBridge"-style):**
- Input: USB HDMI capture (UVC/V4L2) — the Mac sees a normal monitor (Mirror/Extend both work).
- Output: Miracast **source** to the TV's sink.
- UX: auto-start on boot, auto-reconnect to last sink, optional phone web-UI to pick a display and handle first-time PIN.

**Scope / the hard parts (be realistic):**
1. **The Miracast *source* pipeline** is the core engineering. Options:
   - Build on **GNOME Network Displays** (works, but needs a graphical screencast portal — not headless-friendly).
   - Or a **low-level stack**: `wpa_supplicant` P2P (Wi-Fi Direct) + a GStreamer **WFD/RTSP source** (`v4l2src → v4l2h264enc → mpegtsmux → RTP`). Cleaner for an appliance but more custom; today's OSS WFD-source tooling (MiracleCast `wifid` for P2P, `gst-rtsp-server-wfd`) is immature.
2. **Wi-Fi Direct hardware abstraction** — pin a small list of known-good USB adapters (RTL8812AU/8814AU, MT7612U) and bundle drivers.
3. **First-pairing UX** — headless can't enter a TV PIN; need a one-time web-UI or button flow.
4. **Auto-discovery + reconnection** robustness across TV brands.

**Why this is the right OSS bet:** no manufacturing, no certification, no inventory; anyone reproduces it for ~$90 in parts; it directly fills the *software* gap that nobody has polished. Milestones:
- M1: Reliable HDMI-capture → Miracast-source on one reference TV + one Wi-Fi adapter.
- M2: Auto-start + reconnect + web-UI pairing.
- M3: Compatibility matrix (TVs × Wi-Fi adapters), prebuilt image, docs.
- M4 (stretch): optional pre-flashed device for non-builders.

---

## 4. The Kickstarter hardware dongle — honest assessment

**Concept:** a single dongle: HDMI-in → Miracast-out to the TV's sink. Sleek, no parts to assemble.

**What it actually takes (this is a real product, not a weekend hack):**
- **HDMI input** needs a dedicated receiver chip (e.g. **Lontium LT6911**, **Toshiba TC358743**) feeding a SoC — passive adapters can't do it.
- **SoC** with hardware **H.264 encode** + a **Wi-Fi P2P-capable** radio, running custom **Miracast-source firmware** (the part nobody ships as a turnkey module).
- **RF + antenna design**, thermals, power, enclosure.
- **Regulatory:** FCC / CE / RED radio certification (per-region, $$$).
- **Branding:** can't legally call it "Miracast" in marketing without **Wi-Fi Alliance membership + certification** (annual fees + test labs); implementing Wi-Fi Display *without* the trademark is allowed. ([Wi-Fi Alliance](https://www.wi-fi.org/discover-wi-fi/miracast))
- **Manufacturing, inventory, fulfillment, support.**

**Risk verdict:** ⚠️ **High risk, modest market.**
- **NRE in the tens of thousands**, 9–18 months, hardware/firmware expertise required.
- **Narrow TAM** (§1): the need only bites for Miracast-only sinks you can't touch.
- **Incumbent risk:** ScreenBeam et al. own the *receiver* side and could close the gap with a firmware/AirPlay update.
- **Better as:** a *follow-on* to a proven software project — sell an **optional pre-flashed device** (CM4-based, below) once the software is solid and demand is demonstrated, instead of leading with bespoke silicon.

---

## 5. The "soldering / open-source hardware" middle ground

Realistically this isn't hand-soldering — it's a **PCB / carrier-board** project:
- **Raspberry Pi Compute Module 4/5 carrier board** with an onboard **HDMI-input capture** path and a vetted **Wi-Fi P2P module**. You reuse the CM's proven SoC + the software stack from §3, and only design the carrier.
- Pros: far less risk than full custom silicon; leverages the Pi software ecosystem; open-hardware friendly (KiCad, JLCPCB).
- Cons: still PCB design, HDMI-input chip integration (LT6911/TC358743), RF layout, and small-batch assembly; regulatory still applies if sold.
- **Sequence:** prove the **software** on off-the-shelf Pi + capture (§3) → *then* consider collapsing it into a CM4 carrier for a tidier device.

---

## 6. Recommendation

1. **First, kill or confirm the need:** run `school-connection-checklist.md`. If the school receivers do AirPlay, you're done — and the "product" is only worth pursuing as a general-interest OSS project.
2. **To scratch your own itch:** build the **T2 Pi 4 kit** (`pi4-miracast-bridge-build-guide.md`).
3. **If you want to build something others use:** do the **open-source software appliance (§3)** — it's the real gap, low risk, reproducible for ~$90. Lead with software.
4. **Hardware dongle / CM4 board (§4–5):** only *after* the software is proven and demand is real; start with a CM4 carrier (open hardware) before bespoke silicon, and budget for regulatory + (optional) Wi-Fi certification.

---

## 7. Sources
- Wireless-HDMI kits ship matched receivers: [ProScreenCast SC02 kit](https://proscreencast.com/products/sc02-wireless-hdmi-transmitter-and-receiver-kit) · [JTECH WDEX-50M2](https://www.jtechdigital.com/products/jtech-wdex-50m2-jtd-2867-1080p-wireless-hdmi-extender-w-airplay-smart-view-screen-mirroring-miracast-98ft) · [Tom's Guide: screen-mirroring devices](https://www.tomsguide.com/us/best-miracast-screen-mirroring,review-2286.html)
- Multi-protocol EDU receivers (AirPlay+Miracast+Cast): [ScreenBeam 1000 EDU](https://www.screenbeam.com/products/screenbeam-1000-edu/)
- Certification/trademark: [Wi-Fi Alliance — Miracast](https://www.wi-fi.org/discover-wi-fi/miracast) · [Is Wi-Fi Alliance certification mandatory?](https://support.tuya.com/en/help/_detail/Ke5d9p8747q6a)
- Miracast-source software building blocks: [GNOME Network Displays](https://gitlab.gnome.org/GNOME/gnome-network-displays) · [gst-rtsp-server-wfd](https://github.com/albfan/gst-rtsp-server-wfd) · [MiracleCast](https://github.com/albfan/miraclecast)
