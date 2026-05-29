<!-- SPDX-License-Identifier: GPL-3.0-or-later -->
# image/ — flashable SD image build

Scripts to produce a ready-to-flash MiraBridge image so non-builders can just write it to a card.

**Status: M3.** Not started — M1/M2 are validated by hand first; we only automate what's proven.

## Planned approach
- Base on `pi-gen` (Raspberry Pi OS image builder) or a `debootstrap`/Ansible post-install.
- Preinstall: NetworkManager, GNOME Network Displays (or the M3 low-level pipeline), `v4l-utils`, `mpv`, the pinned Wi-Fi adapter driver, and the `pipeline/` configs + systemd units.
- First-boot: launch the `webui/` pairing flow.
- Output: `mirabridge-<version>.img.xz` + checksum.

## Planned contents
| File | Purpose |
|---|---|
| `build.sh` | Produce the image |
| `stage-mirabridge/` | pi-gen stage adding our packages + configs |
| `config.example` | Build options (board, locale, Wi-Fi adapter set) |
