# Research & Plan: Miracast Desktop Sharing from a MacBook to a Miracast TV

**Status:** Research / planning draft
**Date:** 2026-05-29
**Confirmed constraint (2026-05-29):** The target TV supports **Miracast only** — no AirPlay 2, no Google Cast. This rules out Options A & B below and makes a true-Miracast path mandatory.
**Goal:** Get an app that mirrors a MacBook's desktop to a Miracast-certified TV (the wireless display standard used by most Windows/Android devices and by "Wireless Display" / Smart View TVs).

---

## 1. TL;DR

- **macOS has no native Miracast support and never has.** Apple uses its own proprietary **AirPlay** protocol instead. ([Apple Community](https://discussions.apple.com/thread/6064005), [PigeonCast](https://pigeoncast.com/blogs/miracast-macbook))
- Miracast is built on **Wi-Fi Direct (Wi-Fi P2P)**, and **macOS exposes no public Wi-Fi Direct / P2P API** (CoreWLAN does not provide it). This is the single biggest blocker.
- Even the leading open-source Miracast project, **MiracleCast**, has **never implemented the sender/source side** — only the receiver (sink). Writing a from-scratch macOS sender is a very large, research-grade effort. ([MiracleCast](https://github.com/albfan/miraclecast), [issue #4](https://github.com/albfan/miraclecast/issues/4))
- Therefore the realistic paths are **bridging/translation** or **hardware**, not a pure-native Miracast stack. Building a "true" native Miracast sender on macOS is likely infeasible without private APIs or a custom Wi-Fi driver.

**Recommendation:** Pursue a **bridge architecture** (Mac → AirPlay/standard stream → small Linux helper or adapter → Miracast TV), or simply use an **off-the-shelf Miracast HDMI dongle that also speaks AirPlay**. Treat a native macOS Miracast sender as a stretch/research goal only.

---

## 2. What Miracast actually is (technical background)

Miracast is a Wi-Fi Alliance standard (Wi-Fi Display, spec v1.0 released Aug/Sep 2012) for wireless screen mirroring. ([Wikipedia](https://en.wikipedia.org/wiki/Miracast), [Wi-Fi Alliance](https://www.wi-fi.org/discover-wi-fi/miracast))

The pipeline:

1. **Transport / connection:** A direct device-to-device link over **Wi-Fi Direct (Wi-Fi P2P)** — no shared Wi-Fi network or router required. Discovery + capability negotiation happen first.
2. **Session control:** **RTSP**-style negotiation of session parameters between source and sink.
3. **Video:** Captured frames encoded with **H.264** (up to 1080p60 in v1.0; later versions add 4K/HEVC).
4. **Audio:** Stereo (LPCM/AAC).
5. **Packaging:** A/V multiplexed into **MPEG2-TS**, then carried over **RTP / UDP / IP**.

Key implication: a sender must (a) negotiate and hold a **Wi-Fi Direct P2P** link, (b) run the **RTSP** capability handshake, and (c) produce a conformant **H.264 + MPEG2-TS over RTP** stream. Step (a) is the part macOS won't let you do through public APIs.

Sources: [Wi-Fi CERTIFIED Miracast Technical Overview (PDF)](https://tools.barco.com/kb-downloads/4814/Wi-Fi_CERTIFIED_Miracast_Technical_Overview_20170725.pdf), [Copperpod IP](https://www.copperpodip.com/post/understanding-miracast-as-a-wireless-display-technology).

---

## 3. Why this is hard on a MacBook

| Blocker | Detail |
|---|---|
| No native Miracast | macOS has never implemented Miracast at the OS level; it uses AirPlay. ([PigeonCast](https://pigeoncast.com/blogs/miracast-macbook)) |
| No public Wi-Fi Direct API | **CoreWLAN** ([Apple docs](https://developer.apple.com/documentation/corewlan)) covers scanning/associating with normal networks, not Wi-Fi P2P group ownership/negotiation needed for Miracast. |
| Hardware/driver question | Even on Linux, Miracast needs a **P2P-capable Wi-Fi chipset** + `wpa_supplicant`. Apple's Wi-Fi stack doesn't expose P2P group operations to userspace. ([MiracleCast reqs](https://github.com/albfan/miraclecast)) |
| Sender side is unsolved in OSS | MiracleCast = receiver only; the source role is explicitly "not implemented yet." ([issue #4](https://github.com/albfan/miraclecast/issues/4)) |

How existing commercial Mac apps sidestep this: tools like **AirParrot** and **Mirroring360** generally do **protocol translation / bridging** (Mac emits AirPlay/Cast-style streams; the app or a companion receiver bridges to the target) rather than implementing real Miracast P2P on the Mac. ([AirParrot](https://www.airsquirrels.com/airparrot/), [Dr.Fone guide](https://drfone.wondershare.com/mirror-emulator/miracast-mac.html))

---

## 4. Candidate approaches (ranked by feasibility)

### Option A — Buy, don't build: dual-protocol hardware dongle ⭐ easiest
Use a Miracast HDMI dongle that **also supports AirPlay** (many do). The Mac mirrors via native AirPlay; the dongle drives the TV.
- **Pros:** Works today, zero code, native macOS mirroring UX.
- **Cons:** Not a "Miracast from the Mac" solution per se; needs compatible hardware; the TV's *built-in* Miracast isn't what's used.
- **Use when:** The real requirement is "Mac on the big screen," and "Miracast" was just the assumed mechanism.

### Option B — App that bridges to existing AirPlay/Cast (commercial-style) ⭐ most realistic to ship
Build/configure software that captures the Mac screen and sends it over a protocol the **TV already supports natively** (AirPlay 2 on many modern TVs, or Google Cast). This is what AirServer/AirParrot/Mirroring360 effectively do.
- **Pros:** Achievable with public macOS APIs (ScreenCaptureKit for capture, VideoToolbox for H.264) + existing AirPlay/Cast sink libraries.
- **Cons:** It's *not* Miracast — only works if the TV also speaks AirPlay/Cast. ([AirServer](https://www.airserver.com/Overview))

### Option C — Mac → Raspberry Pi (or Linux box) bridge → Miracast TV (true Miracast) ⭐ recommended for a Miracast-only TV you can't touch
A small Linux device sits between the Mac and the TV and does the actual Miracast over the air to the TV's built-in receiver. **No hardware is plugged into the TV.** It's a **two-hop bridge**:

1. **Mac → Pi (over the LAN, via AirPlay):** The Pi runs an AirPlay receiver (e.g. **UxPlay** / RPiPlay) so the Mac mirrors to it natively, with no Mac-side software. The Pi shows the Mac's screen.
2. **Pi → TV (via Miracast):** The Pi runs **GNOME Network Displays**, which casts the Pi's screen to the TV's built-in Miracast sink over Wi-Fi Direct.

- **Update / correction:** A working open-source Miracast **source** *does* exist — **[GNOME Network Displays](https://gitlab.gnome.org/GNOME/gnome-network-displays)** (an experimental Wi-Fi Display implementation), tested against LG WebOS TVs, MiraScreen, Measy/MontoView receivers, etc. (My earlier note that "no OSS sender exists" applied only to MiracleCast, which is sink-only.)
- **Pros:** Genuine Miracast to the TV; nothing attached to the TV; nothing installed on the Mac (uses native AirPlay for the first hop).
- **Cons / caveats — this is a fiddly DIY project, not plug-and-play:**
  - **Wi-Fi Direct hardware is the gating factor.** Needs `wpa_supplicant` built with `CONFIG_P2P` + `CONFIG_WIFI_DISPLAY`, managed by NetworkManager. The Pi's **built-in Wi-Fi P2P is unreliable** (common init failures); a known-good **USB Wi-Fi P2P dongle** (e.g. RTL-based) is often required.
  - **Likely needs two radios.** Hop 1 (AirPlay) uses the normal LAN station connection; hop 2 (Miracast) uses Wi-Fi Direct P2P. A single Pi radio generally can't do station + P2P concurrently — plan on **built-in Wi-Fi for LAN + USB dongle for P2P**.
  - **Two encode/decode hops** (Mac→AirPlay→Pi, then Pi→Miracast→TV) add **latency**; fine for slides/video, poor for gaming/interactive. Use a **Pi 4/5** for the H.264 encode load.
  - Setup is involved (compiling/configuring wpa_supplicant, NetworkManager P2P, UxPlay + GNOME Network Displays).
- **Reality check:** Feasible and the *only* path that uses the TV's own Miracast without touching the TV — but budget real tinkering time, and treat the Wi-Fi adapter choice as make-or-break.

### Option D — Native macOS Miracast sender (from scratch) ❌ likely infeasible
Implement Wi-Fi Direct P2P + RTSP + encode on the Mac directly.
- **Blockers:** No public P2P API; would require private frameworks, a custom kernel Wi-Fi driver, or a dedicated P2P-capable USB Wi-Fi adapter with its own userspace driver. Research-grade, fragile, and likely breaks across macOS updates.
- **Verdict:** Not recommended unless this is explicitly a research project.

---

## 4b. "Can I just add a dongle to the Mac?" — the hardware question

Short answer: **There is no dongle you plug into a Mac that turns it into a working Miracast *sender* for an arbitrary Miracast TV.** Here's why each tempting option fails or succeeds:

| Dongle idea | Verdict | Why |
|---|---|---|
| **USB Wi-Fi adapter** that adds Wi-Fi Direct to the Mac | ❌ Doesn't work | Even with P2P-capable hardware, you still need **Miracast *source* software on macOS — which does not exist.** Hardware alone sends nothing. Marketing claims of "USB dongle + Mac software" don't correspond to a real shipping product. |
| **ScreenBeam USB Transmitter** (and similar HW transmitters) | ❌ Doesn't fit | It's **Windows-only** and pairs with **ScreenBeam's own proprietary receivers**, not a generic Miracast TV. ([ScreenBeam USB Transmitter](https://www.screenbeam.com/nl/products/screenbeam-usb-transmitter-2/)) |
| **"Miracast dongle"** off Amazon | ⚠️ Wrong direction | ~Almost all of these are **receivers** that plug into a TV's HDMI. Your TV is *already* a Miracast receiver, so these are redundant. ([example](https://www.amazon.com/Wireless-Miracast-Mirroring-Receiver-Projector/dp/B08JQ5Z3K1)) |
| **HDMI-input → Miracast-source transmitter** (box that takes Mac HDMI and emits standard Miracast to your TV) | ⚠️ Effectively nonexistent | This specific product category isn't reliably available; "wireless HDMI transmitters" almost always ship with their *own* matched receiver and don't speak standard Miracast to a third-party sink. |
| **Wireless HDMI kit** (transmitter + its *own* receiver) | ✅ Works — but bypasses Miracast | Transmitter plugs into the Mac (USB-C→HDMI); the kit's **own receiver** plugs into the **TV's HDMI** port. The Mac sees a normal external display; **no drivers, no software.** It ignores the TV's Miracast entirely and just uses an HDMI input. Requires a free HDMI port + carrying the small receiver. |

**Takeaway for a Miracast-only TV:** the only true *plug-and-play hardware* answer is a **wireless HDMI kit** into the TV's HDMI port (Option A′ below). If you must use the **TV's built-in Miracast** specifically, there is no Mac dongle for that — you need the **Linux-helper software bridge (Option C)**, which is an engineering project, not a purchase.

### Option A′ — Wireless HDMI kit into the TV's HDMI port ⭐ easiest hardware fix
- **Pros:** Zero software/drivers; works regardless of what wireless protocol the TV supports; Mac treats it as a plain monitor.
- **Cons:** Needs a free **HDMI port** on the TV; you carry a small receiver; doesn't use the TV's Miracast at all; quality/latency varies by kit.
- **Use when:** "Mac on the big screen wirelessly" is the real goal and an HDMI port is available.

---

## 5. Recommended plan (given a Miracast-only TV)

**Phase 0 — Resolved.** TV is **Miracast-only** *and* must not be touched (no plugging anything into the TV). This eliminates Options A, A′ and B. **Option C (Pi bridge) is the only path** — and it's viable thanks to GNOME Network Displays.

**Phase 1 — De-risk the Wi-Fi hardware first (this is make-or-break):**
1. Get a **Wi-Fi Direct / P2P-capable USB adapter** known to work with `wpa_supplicant` `CONFIG_P2P` (e.g. RTL-based). On a Pi, plan for **built-in Wi-Fi = LAN/AirPlay, USB dongle = P2P/Miracast**.
2. Prove **Pi → TV Miracast** alone: install **GNOME Network Displays**, confirm it discovers and casts the Pi desktop to the TV. If this fails, the whole approach fails — stop and reassess.

**Phase 2 — Add the Mac → Pi hop:** Install an AirPlay receiver (**UxPlay**) on the Pi; mirror the Mac to it natively; confirm the Mac screen shows on the Pi.

**Phase 3 — Chain the hops & tune:** Cast the Pi's (AirPlay-fed) screen to the TV via GNOME Network Displays; measure end-to-end latency/quality; use a Pi 4/5 for encode headroom.

**Phase 3 — Build the chosen path** (most likely Option B as a shippable app, with Option A as the no-code fallback). Reserve Option C/D for a true-Miracast requirement and budget research time accordingly.

---

## 6. Key risks & open technical questions

- **Miracast *source* in OSS is experimental, not mainstream.** MiracleCast is sink-only, but **GNOME Network Displays** does implement the source role (experimental). Expect rough edges, not a polished tool.
- **Wi-Fi Direct hardware compatibility is the #1 risk.** Pi built-in Wi-Fi P2P is flaky; a specific USB P2P dongle is often required, and station + P2P usually can't share one radio (plan for two interfaces).
- **macOS sandbox/entitlements:** ScreenCaptureKit needs Screen Recording permission; an App Store build adds further constraints.
- **Latency** stacks up with each bridge hop — may not suit interactive/gaming use.
- **"Miracast" may be a proxy requirement.** Often the user just wants "Mac → TV wirelessly," and the TV supports AirPlay/Cast, making real Miracast unnecessary.

---

## 7. Decisions needed before building

1. ~~What does the TV support?~~ **Resolved: Miracast only.**
2. **Does the TV have a free HDMI port, and is using its *built-in Miracast specifically* a hard requirement?** *(If an HDMI port is fine → Option A′ wireless HDMI kit, done. If Miracast-the-protocol is mandatory → Option C.)*
3. **Acceptable to add a small companion device** (Pi / mini-PC) for Option C? *(Gates the only true-Miracast path.)*
4. **Distribution target:** personal tool, open-source, or App Store product? *(Affects entitlements & architecture.)*
5. **Latency tolerance:** presentation/video (lenient) vs interactive/gaming (strict)?

---

## 8. Sources

- macOS & Miracast: [Apple Community](https://discussions.apple.com/thread/6064005) · [PigeonCast](https://pigeoncast.com/blogs/miracast-macbook) · [Dr.Fone](https://drfone.wondershare.com/mirror-emulator/miracast-mac.html) · [Alibaba/Electronics guide](https://electronics.alibaba.com/buyingguides/miracast-for-macbook-pro-what-works-(and-what-doesn%E2%80%99t))
- Protocol: [Wikipedia – Miracast](https://en.wikipedia.org/wiki/Miracast) · [Wi-Fi Alliance](https://www.wi-fi.org/discover-wi-fi/miracast) · [Barco Technical Overview (PDF)](https://tools.barco.com/kb-downloads/4814/Wi-Fi_CERTIFIED_Miracast_Technical_Overview_20170725.pdf) · [Copperpod IP](https://www.copperpodip.com/post/understanding-miracast-as-a-wireless-display-technology)
- Implementation/OSS: [GNOME Network Displays (Miracast source)](https://gitlab.gnome.org/GNOME/gnome-network-displays) · [gnome-network-displays mirror](https://github.com/benzea/gnome-network-displays) · [MiracleCast](https://github.com/albfan/miraclecast) (sink-only) · [MiracleCast sender issue #4](https://github.com/albfan/miraclecast/issues/4) · [piracast](https://github.com/codemonkeyricky/piracast) · [lazycast](https://github.com/homeworkc/lazycast) · [Apple CoreWLAN docs](https://developer.apple.com/documentation/corewlan)
- Pi Wi-Fi Direct / P2P: [wpa_supplicant P2P module](https://w1.fi/wpa_supplicant/devel/p2p.html) · [Pi wpa_supplicant P2P troubleshooting](https://industrialmonitordirect.com/blogs/knowledgebase/raspberry-pi-wpa-supplicant-wi-fi-direct-p2p-not-starting)
- Commercial bridging apps: [AirParrot 3](https://www.airsquirrels.com/airparrot/) · [AirServer](https://www.airserver.com/Overview) · [Mirroring360](https://www.mirroring360.com/android-faq)
