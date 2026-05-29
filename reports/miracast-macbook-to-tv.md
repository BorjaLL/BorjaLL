# Research & Plan: Miracast Desktop Sharing from a MacBook to a Miracast TV

**Status:** Research / planning draft
**Date:** 2026-05-29
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

### Option C — Mac → companion Linux helper → Miracast TV (true Miracast, bridged) ⭐ true Miracast, moderate effort
Run a tiny Linux box (Raspberry Pi, mini-PC, or even a Linux VM with a USB Wi-Fi P2P dongle passed through) as the **Miracast source**. The Mac streams its desktop to the helper over the LAN; the helper does the actual Wi-Fi Direct + RTSP + H.264/MPEG2-TS handshake to the TV.
- **Pros:** Delivers genuine Miracast to the TV; uses a Linux stack where P2P/`wpa_supplicant` is available.
- **Cons:** The Miracast *source* role still isn't off-the-shelf in OSS (MiracleCast only sinks today), so the helper-side sender is itself an implementation project. Requires extra hardware (P2P-capable USB Wi-Fi). Added latency from the extra hop.
- **Reality check:** This is the most promising path to *real* Miracast, but it inherits the "no open-source sender exists" problem — see §6 risks.

### Option D — Native macOS Miracast sender (from scratch) ❌ likely infeasible
Implement Wi-Fi Direct P2P + RTSP + encode on the Mac directly.
- **Blockers:** No public P2P API; would require private frameworks, a custom kernel Wi-Fi driver, or a dedicated P2P-capable USB Wi-Fi adapter with its own userspace driver. Research-grade, fragile, and likely breaks across macOS updates.
- **Verdict:** Not recommended unless this is explicitly a research project.

---

## 5. Recommended plan

**Phase 0 — Define the real requirement (decision needed, see §7).**
Confirm whether the TV is *Miracast-only*, or whether it also supports **AirPlay 2 / Google Cast**. This single answer determines whether Option B (easy) or Option C (hard) is needed.

**Phase 1 — Spike each viable option (1–2 days each):**
1. **Option A spike:** Try a dual-protocol (AirPlay+Miracast) dongle with native macOS mirroring. Validate latency/quality. *Cheapest answer if it satisfies the goal.*
2. **Option B spike:** Prototype Mac screen capture (**ScreenCaptureKit**) → H.264 (**VideoToolbox**) → AirPlay/Cast sink. Confirms the "build an app" path if TV speaks AirPlay/Cast.
3. **Option C spike:** Stand up a Linux helper, verify a USB Wi-Fi adapter is **Wi-Fi Direct capable** (MiracleCast ships a hardware test script), and confirm Miracast *sink* works. This de-risks the hardware before attempting the unimplemented *source* role.

**Phase 2 — Pick the architecture** based on spike results and the Phase 0 decision.

**Phase 3 — Build the chosen path** (most likely Option B as a shippable app, with Option A as the no-code fallback). Reserve Option C/D for a true-Miracast requirement and budget research time accordingly.

---

## 6. Key risks & open technical questions

- **Open-source Miracast *sender* does not exist** (MiracleCast is sink-only). Any "true Miracast" path requires writing the source role — the hardest, least-documented part.
- **Wi-Fi Direct hardware compatibility** is not guaranteed; many chipsets/dongles silently lack working P2P.
- **macOS sandbox/entitlements:** ScreenCaptureKit needs Screen Recording permission; an App Store build adds further constraints.
- **Latency** stacks up with each bridge hop — may not suit interactive/gaming use.
- **"Miracast" may be a proxy requirement.** Often the user just wants "Mac → TV wirelessly," and the TV supports AirPlay/Cast, making real Miracast unnecessary.

---

## 7. Decisions needed before building

1. **What exactly does the target TV support?** Miracast only, or also AirPlay 2 / Google Cast? *(Determines Option B vs C.)*
2. **Is "Miracast specifically" a hard requirement, or is "wireless Mac→TV" the real goal?** *(Determines whether Option A/B suffice.)*
3. **Acceptable to add a small companion device** (Pi / dongle / mini-PC) if it's the only way to get real Miracast? *(Gates Option C.)*
4. **Distribution target:** personal tool, open-source, or App Store product? *(Affects entitlements & architecture.)*
5. **Latency tolerance:** presentation/video (lenient) vs interactive/gaming (strict)?

---

## 8. Sources

- macOS & Miracast: [Apple Community](https://discussions.apple.com/thread/6064005) · [PigeonCast](https://pigeoncast.com/blogs/miracast-macbook) · [Dr.Fone](https://drfone.wondershare.com/mirror-emulator/miracast-mac.html) · [Alibaba/Electronics guide](https://electronics.alibaba.com/buyingguides/miracast-for-macbook-pro-what-works-(and-what-doesn%E2%80%99t))
- Protocol: [Wikipedia – Miracast](https://en.wikipedia.org/wiki/Miracast) · [Wi-Fi Alliance](https://www.wi-fi.org/discover-wi-fi/miracast) · [Barco Technical Overview (PDF)](https://tools.barco.com/kb-downloads/4814/Wi-Fi_CERTIFIED_Miracast_Technical_Overview_20170725.pdf) · [Copperpod IP](https://www.copperpodip.com/post/understanding-miracast-as-a-wireless-display-technology)
- Implementation/OSS: [MiracleCast](https://github.com/albfan/miraclecast) · [MiracleCast sender issue #4](https://github.com/albfan/miraclecast/issues/4) · [Apple CoreWLAN docs](https://developer.apple.com/documentation/corewlan)
- Commercial bridging apps: [AirParrot 3](https://www.airsquirrels.com/airparrot/) · [AirServer](https://www.airserver.com/Overview) · [Mirroring360](https://www.mirroring360.com/android-faq)
