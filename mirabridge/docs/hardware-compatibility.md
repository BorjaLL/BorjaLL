<!-- SPDX-License-Identifier: GPL-3.0-or-later -->
# Hardware Compatibility

> ⚠️ Pre-alpha. This table is **aspirational + to-be-verified**. Statuses: ✅ verified · 🧪 expected/under test · ❌ known-bad · ❔ unknown.
> Help fill it in: open a [compatibility report](../.github/ISSUE_TEMPLATE/compatibility_report.md).

## Boards

| Board | H.264 HW encode | Notes | Status |
|---|---|---|---|
| Raspberry Pi 4 | ✅ yes | Reference board for M1. 5V/3A. | 🧪 |
| Raspberry Pi 5 | ❌ no (CPU encode) | Encoder removed; worse for this job. | ❔ |
| Intel N100 mini-PC | ✅ QuickSync | Best reliability; not portable. | ❔ |

## Wi-Fi Direct (P2P) adapters — the make-or-break part

| Adapter | Chipset | Driver | P2P modes | Notes | Status |
|---|---|---|---|---|---|
| **Alfa AWUS036ACM** | **MT7612U** | `mt76x2u` (mainline ≥4.19) | P2P-client/GO | **Reference.** No out-of-tree build. 5 GHz on Pi 4 (VL805) may need Scatter-Gather disabled. | 🧪 |
| Generic RTL8812AU | RTL8812AU | `morrownr/8812au` (DKMS) | P2P | Works but needs out-of-tree driver. | ❔ |
| RTL8814AU | RTL8814AU | DKMS | P2P | High power; bulky. | ❔ |
| Intel AX200/AX210 (onboard) | — | `iwlwifi` | flaky | Reported to "fail silently" with GND P2P. | ❌ |
| Raspberry Pi onboard Wi-Fi | BCM | — | unreliable | Avoid for the Miracast leg. | ❌ |

## HDMI capture devices

| Device | Chipset | Interface | Max | Driver | Status |
|---|---|---|---|---|---|
| **Generic "4K HDMI→USB3"** | **MS2130** | USB 3.0 UVC | 1080p60 uncompressed | `uvcvideo` (none needed) | 🧪 |
| Elgato Cam Link 4K | — | USB 3.0 UVC | 1080p60 | `uvcvideo` | ❔ |
| Cheap "HDMI→USB" stick | MS2109 | USB 2.0 UVC | 1080p30, **high latency** | `uvcvideo` | ❔ (not recommended) |

## Sinks (TVs / Miracast receivers)

| Sink | Type | PIN required? | Notes | Status |
|---|---|---|---|---|
| MiraScreen / AnyCast dongle | Miracast | varies | Suggested **controllable reference** sink for the bench. | 🧪 |
| ScreenBeam (EDU/Enterprise) | Multi-protocol | varies | **Often also does AirPlay → may not need MiraBridge.** | ❔ |
| LG WebOS TV | Miracast (Screen Share) | sometimes | In GND's tested list historically. | ❔ |
| Samsung TV | Miracast (Screen Mirroring) | often (first connect) | | ❔ |

## How to add a row
Open a compatibility report with: exact models, chipsets, OS/kernel, GND version, and the result (which ACs from `../reports/m1-spec.md` passed).
