# Research & Plan: Miracast Desktop Sharing from a MacBook to a Miracast TV

**Status:** Research / planning draft
**Date:** 2026-05-29
**Confirmed constraint (2026-05-29):** The target TV supports **Miracast only** — no AirPlay 2, no Google Cast. This rules out Options A & B below and makes a true-Miracast path mandatory.
**Goal:** Get an app that mirrors a MacBook's desktop to a Miracast-certified TV (the wireless display standard used by most Windows/Android devices and by "Wireless Display" / Smart View TVs).
**Chosen direction:** Portable Raspberry Pi 4 bridge (Option C1). **→ See the step-by-step [Pi 4 Build Guide](./pi4-miracast-bridge-build-guide.md)** for parts, setup, and the non-technical workflow/timing.
**Companion docs:** [School connection checklist](./school-connection-checklist.md) (try the AirPlay shortcut first) · [DIY options, budget tiers & product/Kickstarter exploration](./diy-options-and-product-exploration.md).

---

## 0b. ⭐ POSSIBLE SHORTCUT — check before building anything (school context)

The school TVs show **"press Win+K to connect" / ready and waiting**. That is the signature of an **enterprise/EDU wireless-display receiver** (very commonly **ScreenBeam**, e.g. ScreenBeam 1000 EDU / 1100). These boxes **natively support AirPlay AND Miracast AND Google Cast simultaneously** — "Win+K" is just the *Windows* instruction. ([ScreenBeam 1000 EDU](https://www.screenbeam.com/products/screenbeam-1000-edu/))

**Implication:** the MacBook may connect **natively via AirPlay** over the school Wi-Fi — **no Pi, no bridge, nothing to build.**

**2-minute test on the Mac (on school Wi-Fi):**
1. **Control Center → Screen Mirroring** — does the room/display appear as an AirPlay target? If yes → connect → done.
2. Read/photograph the TV's on-screen text (brand/model/room name — likely "ScreenBeam").
3. If not visible, in Terminal: `dns-sd -B _airplay._tcp` and `dns-sd -B _googlecast._tcp` (few seconds each, then Ctrl+C).

**Interpreting results:**
- In Screen Mirroring → **native AirPlay works; the whole Pi project is unnecessary.**
- In `dns-sd` but not Screen Mirroring → advertises AirPlay but handshake blocked (AWDL/firewall) — usually fixable.
- Nothing → likely **Wi-Fi client isolation** (common on school networks; blocks device-to-device for *everyone*) or a Miracast-only sink → fall back to the Pi bridge (Option C).

**Caveats:** client isolation is a network-policy block the Mac can't bypass; keep to normal client discovery (Control Center / `dns-sd`), not port scans of school infrastructure.

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

### Option C — Mac → Linux bridge box → Miracast TV (true Miracast) ⭐ recommended for a Miracast-only TV you can't touch
A small Linux device sits between the Mac and the TV and does the actual Miracast over the air to the TV's built-in receiver. **No hardware is plugged into the TV.** It's a **two-hop bridge**, and there are two ways to do the first hop.

**Board choice depends on the priority. For a *small, portable, latency-tolerant* box (slides/video), a Raspberry Pi 4 is the pick; for a fixed, reliable, low-latency appliance, x86 wins.** Two things drive it:
- **The Wi-Fi Direct adapter is the make-or-break part, and it's board-independent.** Even Intel AX200/AX210 "fail silently" with GNOME Network Displays; the best-tested chipsets are **Realtek RTL8812AU/RTL8814AU** or **MediaTek MT7612U** USB adapters. Plan to buy one of these regardless of board.
- **The Miracast leg needs H.264 *encode*. Among Pis, the Pi 4 has a hardware H.264 encoder; the Pi 5 *removed* it (CPU-only).** So the **Pi 4 is the better Pi for this job.** x86 (N100/QuickSync) still encodes best of all, but isn't pocket-portable and needs mains power.

| Board | Verdict for this job |
|---|---|
| **Raspberry Pi 4** | ✅ Best for *portability*: smallest self-contained Linux box, **has a HW H.264 encoder**, runs off a USB-C power bank. Encode/latency limits are fine for slides/video. |
| **Intel N100 mini-PC** | ✅ Best for a *fixed appliance*: QuickSync HW encode, x86 driver maturity — but bulkier and needs mains power, so less portable. |
| **Old laptop / PC (Linux)** | ✅ Cheapest if owned, x86 encode + mature drivers — but not portable (defeats "don't carry two laptops"). |
| **Raspberry Pi 5** | ⚠️ Worse than the Pi 4 here: **no HW H.264 encoder**, and needs more power (5V/5A). |

**Hop 2 (same for any board) — bridge → TV via Miracast:** Run **GNOME Network Displays**, which casts the box's screen to the TV's built-in Miracast sink over Wi-Fi Direct (via the USB P2P adapter).

**Hop 1 — getting the Mac's screen onto the box. Two options:**

- **C1 — Wired HDMI capture (recommended).** Mac → USB-C-to-HDMI → **USB HDMI capture dongle** → box's USB. The capture dongle presents EDID, so **the Mac treats it as a real external monitor** (no Mac software), and the box sees it as a standard **UVC / V4L2** video device.
  - **Watch:** capture dongle quality dictates latency — cheap **MS2109** sticks can add **1–2 s** and cap at 1080p30; use a **USB 3.0 MS2130 / Elgato Cam Link-class** device for low latency + 1080p60. GNOME Network Displays casts the *desktop*, so either show the capture fullscreen and cast that, or patch its GStreamer pipeline to use `v4l2src`.

- **C2 — AirPlay (no extra capture hardware).** The box runs an AirPlay receiver (**UxPlay**); the Mac mirrors to it natively. Simpler hardware, but adds a wireless encode/decode hop and AirPlay's own variability.

- **Update / correction:** A working open-source Miracast **source** *does* exist — **[GNOME Network Displays](https://gitlab.gnome.org/GNOME/gnome-network-displays)** (an experimental Wi-Fi Display implementation), tested against LG WebOS TVs, MiraScreen, Measy/MontoView receivers, etc. (My earlier note that "no OSS sender exists" applied only to MiracleCast, which is sink-only.)
- **Pros:** Genuine Miracast to the TV; nothing attached to the TV; nothing installed on the Mac.
- **Cons / caveats — this is a fiddly DIY project, not plug-and-play:**
  - **Wi-Fi Direct hardware is the gating factor** (see board note above). Needs `wpa_supplicant` built with `CONFIG_P2P` + `CONFIG_WIFI_DISPLAY`, managed by NetworkManager (not iwd).
  - **C1 uses two USB peripherals** (HDMI capture + Wi-Fi P2P). **C2 needs two radios** (onboard Wi-Fi for LAN/AirPlay + USB dongle for P2P), since one radio generally can't do station + P2P concurrently.
  - **Two encode/decode hops** add **latency** either way; fine for slides/video, poor for gaming/interactive.
  - Setup is involved (wpa_supplicant/NetworkManager P2P, GNOME Network Displays, + capture pipeline or UxPlay).
- **Biggest project risk (board-independent):** whether GNOME Network Displays will actually *pair with this specific TV*. Test that first on any Linux box you already have + a known-good USB P2P adapter, before buying an appliance.

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

**Phase 0 — Resolved constraints.** TV is **Miracast-only** *and* must not be touched. The bridge must be **very small/portable** (no second laptop). Use case is **slides/video/browsing**, so **latency is not a concern**. → Options A, A′, B are out; **Option C is the path**, on a **Raspberry Pi 4** (smallest self-contained Linux box that still has a HW H.264 encoder; latency-tolerant use makes its limits irrelevant).

**Portable kit (fits in a small pouch):**
- **Raspberry Pi 4** + small case + microSD
- **USB Wi-Fi Direct adapter** (RTL8812AU/8814AU or MT7612U) — the make-or-break part
- **Input (C1, recommended):** small **USB HDMI capture dongle** + **USB-C→HDMI** cable. *(C2 alt: no capture dongle; Pi runs an AP + UxPlay for AirPlay — fewer parts, more fragile setup.)*
- **USB-C power bank** (Pi 4 = 5V/3A; easier on a battery than a Pi 5)

This replaces "carry a second laptop" with "carry a phone-sized box + a cable." It's a **DIY kit, not a sleek single dongle**, and the software is experimental.

**Phase 1 — De-risk the two unknowns first (before buying everything):**
1. Get the **USB Wi-Fi P2P adapter** and prove **box → TV Miracast** alone with **GNOME Network Displays** — confirm it discovers and pairs with *this specific TV*. If this fails, the whole approach fails — stop and reassess.
2. (Can be done on any Linux machine you have; the board doesn't matter for this test.)

**Phase 2 — Add the Mac → box hop (wired, C1):** Plug a **USB HDMI capture dongle** into the box and the Mac (via USB-C→HDMI); confirm the Mac sees it as an external monitor and the box sees a V4L2 device.

**Phase 3 — Chain the hops & tune:** Show the capture fullscreen on the box and cast that screen to the TV via GNOME Network Displays (or wire `v4l2src` into its pipeline); confirm quality. Latency isn't a target for this use case.

**Phase 4 — Package for portability:** Auto-start the pipeline on boot so the kit "just works" when powered from the battery; no keyboard/monitor needed at the venue.

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
- Pi HDMI capture / ports: [HDMI-to-USB capture on Pi (Hackaday)](https://hackaday.com/2020/12/21/heavy-raspberry-pi-user-keep-an-hdmi-to-usb-capture-device-around/) · [Adafruit HDMI→USB capture (UVC)](https://www.adafruit.com/product/4669) · [HDMI input via USB dongles (RPi forums)](https://forums.raspberrypi.com/viewtopic.php?t=291063) · [Pi 5 USB-C is power-only, no video alt mode](https://forums.raspberrypi.com/viewtopic.php?t=375782)
- Commercial bridging apps: [AirParrot 3](https://www.airsquirrels.com/airparrot/) · [AirServer](https://www.airserver.com/Overview) · [Mirroring360](https://www.mirroring360.com/android-faq)
