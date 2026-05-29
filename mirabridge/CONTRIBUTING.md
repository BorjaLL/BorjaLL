<!-- SPDX-License-Identifier: GPL-3.0-or-later -->
# Contributing to MiraBridge

Thanks for helping! This is an early, experimental project — the most valuable contributions right now are **real-world test reports**, not just code.

## The #1 way to help: compatibility reports

We need data on which **TVs/sinks**, **Wi-Fi Direct adapters**, and **capture devices** actually work. Even a "it didn't work" report is gold.
- Open a **[Compatibility report](./.github/ISSUE_TEMPLATE/compatibility_report.md)** issue.
- Include: sink make/model, Wi-Fi adapter (chipset), capture device (chipset), Pi/board, OS/kernel, and what happened.

## Code contributions

- Discuss non-trivial changes in an issue first.
- Keep changes scoped to a milestone (see [`ROADMAP.md`](./ROADMAP.md)); don't pull M3 work into M1.
- Match the existing style; keep configs in [`pipeline/`](./pipeline/), docs in [`docs/`](./docs/).
- Add an `SPDX-License-Identifier: GPL-3.0-or-later` header to new source files.

## Dev setup (M1)

Follow `../reports/m1-spec.md` for the bench bring-up. In short: Raspberry Pi OS (NetworkManager), the reference Wi-Fi adapter and capture device, and GNOME Network Displays.

## Reporting bugs

Use the **[Bug report](./.github/ISSUE_TEMPLATE/bug_report.md)** template. Always include OS/kernel, GND version, Wi-Fi adapter + driver (`dmesg | grep`), and the `v4l2-ctl --list-formats-ext` output for your capture device.

## Scope & conduct

- Stay within the project scope in [`ROADMAP.md`](./ROADMAP.md).
- Be kind and constructive. Assume good faith.

## Legal

By contributing you agree your contributions are licensed under **GPL-3.0-or-later**. Don't paste code from incompatible licenses.
