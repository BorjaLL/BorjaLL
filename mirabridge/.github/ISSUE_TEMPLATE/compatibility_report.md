---
name: Compatibility report
about: Tell us what hardware/sink you tried and whether it worked (even failures help!)
title: "[compat] <sink> + <wifi adapter> — works? "
labels: compatibility
---

<!-- The most valuable contribution to MiraBridge right now. Thank you! -->

## Result
- [ ] ✅ Worked (Mac visible on the sink)
- [ ] 🟡 Partial (describe below)
- [ ] ❌ Did not work

## Hardware
- **Sink (TV / Miracast receiver):** make + model
- **Does the sink also support AirPlay?** (yes/no/unknown) — *if yes, you may not need MiraBridge*
- **Wi-Fi Direct adapter:** product + chipset (e.g. Alfa AWUS036ACM / MT7612U)
- **Capture device:** product + chipset (e.g. MS2130)
- **Board:** (e.g. Raspberry Pi 4 4GB)

## Software
- **OS + version:**
- **Kernel** (`uname -r`):
- **GNOME Network Displays version + install method** (flatpak/apt):

## What happened
- Which M1 acceptance criteria passed? (AC1 Miracast-leg / AC2 capture / AC3 full chain / AC4 mirror+extend / AC5 recovery)
- **Measured latency** (if any):
- **Working resolution/fps:**

## Diagnostics (paste output)
```
iw list | grep -A8 "Supported interface modes"
dmesg | grep -i -E "mt76|8812|8814"
v4l2-ctl --list-formats-ext
```

## Notes / errors
<!-- PIN prompts, disconnects, GND portal errors, etc. -->
