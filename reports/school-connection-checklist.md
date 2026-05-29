# At School: How to Connect Your MacBook to the Classroom TV

**Goal:** Find out (in ~2 minutes) whether your Mac can already cast to the classroom display with **no extra hardware**.

The TVs show **"press Win+K to connect"**. That's just the *Windows* instruction — the same box very likely also accepts **AirPlay** from a Mac. Let's check.

---

## Step 1 — Get on the school Wi-Fi
Connect the MacBook to the same school Wi-Fi network the room uses. (If there's a separate "staff" vs "student" network, try the one you're meant to use.)

## Step 2 — Try Screen Mirroring (the main test)
1. Click **Control Center** (the two-toggles icon in the top-right menu bar).
2. Click **Screen Mirroring**.
3. Wait ~10 seconds. **Does the room/display name appear in the list?**
   - ✅ **Yes →** click it. If it asks for a code, type the number shown on the TV. **You're done — that's all you ever need to do.**
   - ❌ **No →** go to Step 3.

## Step 3 — Look at the TV screen and note what it says
Read the text on the TV and **take a photo**. Note especially:
- Any **brand name** (e.g. "ScreenBeam", "EZCast", "Microsoft").
- Any **device or room name** (e.g. "Room 204", "Apple TV", "SBWD-...").
- Whether it mentions **AirPlay**, **Miracast**, or **Cast**.

## Step 4 — Quick discovery check (optional, harmless)
Open **Terminal** (Spotlight → type "Terminal") and run these one at a time. Each prints for a few seconds — press **Control+C** to stop, then run the next:

```bash
dns-sd -B _airplay._tcp
```
```bash
dns-sd -B _googlecast._tcp
```

If a device name shows up under either, **write it down** (or screenshot).

---

## What to report back
Send these four things and the right next step becomes clear:
1. Did the display appear in **Screen Mirroring**? (yes/no)
2. The **photo** of the TV screen / the brand + name shown.
3. Any names that appeared from the `dns-sd` commands.
4. Whether there's a separate guest/staff/student Wi-Fi.

---

## What each outcome means (quick reference)
| What you saw | What it means | Next step |
|---|---|---|
| Display showed in **Screen Mirroring** | Native **AirPlay** works | **Done** — just connect, no hardware needed |
| Showed in **`dns-sd`** but not Screen Mirroring | It advertises AirPlay but the connection is blocked | Often fixable (firewall/AWDL) — report back |
| **Nothing** appeared anywhere | School Wi-Fi likely **isolates devices**, or the display is Miracast-only | Fall back to the DIY bridge plan |

## Please don't
- Don't run network *scanners* (e.g. nmap) against school equipment — not needed, and likely against the school's acceptable-use policy.
- The steps above are exactly what any Mac does normally to find a display, so they're fine.
