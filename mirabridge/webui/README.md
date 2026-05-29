<!-- SPDX-License-Identifier: GPL-3.0-or-later -->
# webui/ — first-pairing & status UI

A small, phone-friendly web app so a non-technical user can set up a headless box: pick a sink, enter a TV PIN, see status.

**Status: M2.** Not started.

## Why it's needed
A headless appliance can't show GND's GUI or type a TV PIN. The web UI fills that gap on first run.

## Planned scope (M2)
- List discovered Miracast sinks; let the user pick one.
- Enter a **PIN** when the sink requires it.
- Show connection status (capture present? sink connected? last error?).
- "Remember this sink" → auto-reconnect on boot.
- Reachable via the box's hostname/IP on the LAN, or a fallback setup hotspot.

## Likely stack
Tiny Python (Flask/FastAPI) or Go service talking to the pairing backend; static mobile-first frontend. Keep it dependency-light for the image.
