<!-- SPDX-License-Identifier: GPL-3.0-or-later -->
# pipeline/ — runtime configs

The actual "make video go from capture to TV" plumbing lives here.

## Planned contents

| File | Purpose | Milestone |
|---|---|---|
| `wpa_supplicant-p2p.conf` | Wi-Fi Direct (P2P) config for the Miracast leg | M1 |
| `capture-fullscreen.sh` | Show the V4L2 capture fullscreen (mpv) for GND to cast | M1 |
| `gnd-autostart.desktop` | Launch GNOME Network Displays in the session | M1/M2 |
| `mirabridge.gst` / `pipeline.sh` | Low-level `v4l2src → v4l2h264enc → mpegtsmux → RTP` source | M3 |
| `mirabridge.service` | systemd unit to start the chain on boot | M2 |

## M1 quick reference (manual, on the bench)

```bash
# 1. Verify capture
v4l2-ctl --list-devices
v4l2-ctl -d /dev/video0 --list-formats-ext

# 2. Show capture fullscreen (what GND will cast)
mpv av://v4l2:/dev/video0 --profile=low-latency --untimed --fullscreen --no-osc

# 3. Cast the desktop with GNOME Network Displays (GUI) → pick the sink
```

See `../../reports/m1-spec.md` for the full bring-up. Configs land here as M1 is validated.
