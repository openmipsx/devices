# PAP2T hardware

Status: draft, from photos of the closed case, a video of the front
panel during power-on, and the stock web UI's Info page (2026-10-30). The case has not been opened. Clean-side document:
case and chip markings and public datasheets only. See
[`../clean-room.md`](../clean-room.md) for the process.

Each fact is marked as in the SPA3000's
[`hardware.md`](../spa3000/hardware.md): **Read**, **Datasheet**,
**Public source**, **Measured** or **Hypothesis**.

Photos are in [`photos/`](photos/). The serial number and MAC address are
blacked out in the label photo, and metadata is stripped from all of them.

| File | Shows |
|---|---|
| `20261030-PAP2T-Case-Underside-Label.jpg` | Underside: label, rubber feet, keyhole wall-mount slots |
| `20261030-PAP2T-Case-Rear-Ports.jpg` | Rear panel: port labels and power rating |
| `20261030-PAP2T-Case-Front-LEDs.jpg` | Front panel: light windows and their labels |
| `20261030-PAP2T-LEDs-Power-Amber.jpg` | Front panel 1.9 s after power-on: Power amber, the other three green (a frame from the video in §1.1) |

The video itself (74 s, 186 MB) is not published; its hash is in the
provenance log.

## 1. Board and case

| Item | Value | Status |
|---|---|---|
| Model | `PAP2T`, "Internet Phone Adapter with 2 Ports for Voice-over-IP", Linksys (a division of Cisco Systems) with the Cisco Systems logo | Read (label) |
| Hardware version | Not on the label | Read (label) |
| Versions reported by the stock web UI (Info page, 2026-10-30) | Product name `PAP2T-NA`; software 5.1.6(LS), the later of the two PAP2T images examined (see the [README](README.md)); hardware 0.1.5; client certificate installed; customization Open (not provider-locked). The page title and header call it a PAP2. | Observed |
| Clock | With no time server configured, the Info page's current time read 1/1/2003 13:05:58 at an elapsed time of 01:05:58: the clock starts at noon on 1 January 2003 at power-on, so the unit keeps no time across a power cycle. | Observed |
| State when read | Address from DHCP; both lines unregistered (no account set), both phones on hook; uptime about 66 minutes. The LEDs were all on steadily (§1.1) although neither line was registered. | Observed |
| Serial number and MAC address | Printed on the label as text and barcodes; not recorded here | Read |
| Date of manufacture | The label has a box for it, left blank | Read (label) |
| Compliance marks | FCC logo (no FCC ID printed), Industry Canada ICES-003, a stylised R mark (not identified), CE, the Australian A-Tick with supplier code `N13171`, cULus listed ITE `E213095`, WEEE bin; made in China | Read (label) |
| Enclosure | Two-tone plastic case: silver top and underside, grey band round the sides, with vent slots on at least the right-hand side. The top has a recessed Linksys / Cisco Systems badge. The underside has four rubber feet and two keyhole slots for wall mounting, through which the green board is visible. | Read (photos) |
| Power input | DC 5 V, 2 A, centre positive | Read (rear label and polarity symbol) |
| Rear ports, left to right seen from behind | PHONE 1 (RJ11), PHONE 2 (RJ11), both in one green surround; INTERNET (RJ45) in a blue surround; POWER jack. No line (FXO) port. The RJ45 has no visible built-in LEDs. A green part is visible inside the RJ45 opening; not identified. | Read (rear photo) |
| Front panel | Four light windows, left to right seen from the front: PHONE 1, PHONE 2, ETHERNET, Power (⏻ symbol). "Internet Phone Adapter" is printed to their left, beside a logo cut off at the edge of the photo. | Read (front photo) |
| LED behaviour at power-on | See §1.1. Power is bicolour: amber, then green. | Seen in a video |

### 1.1 Front-panel LEDs during start-up

From a 74 s phone video at 30 frames a second, read frame by frame
(2026-10-30). Powered from its own supply, running the stock firmware
(5.1.6(LS), read afterwards), Ethernet unplugged at first, nothing on the
phone ports. Times are from power-on; durations are to within about a
frame (33 ms) where the LEDs were in view, and the camera moves, so the
brightness of one LED against another cannot be compared.

| Time | All four LEDs (PHONE 1, PHONE 2, ETHERNET, Power) | Status |
|---|---|---|
| 0 | Power applied. All four come on together at once: PHONE 1, PHONE 2 and ETHERNET green, Power amber. | Seen |
| 2.4 s | Power changes from amber to green. All four stay on. | Seen |
| 3.9 s | All four start flashing together: about 0.5 s on, 0.5 s off (1 Hz), eleven or twelve times. | Seen |
| 14.6 s | The pattern changes to a repeating one, all four together: on about 0.5 s, off 0.3 s, on 0.1 s, off 0.1 s, on 0.1 s, then off 2.0 s. One cycle every 3.1 s, seen six times. | Seen |
| about 34 s | Ethernet cable plugged in, to a live network. The LEDs are out of view from 30.7 s to 35.4 s while this is done. | Seen |
| 35.4 s | All four flash together at 1 Hz again (0.5 s on, 0.5 s off), five or six times. | Seen |
| 41.5 s to the end (68 s) | All four on steadily, with occasional dropouts of all four at once for one to three frames (30 to 100 ms), a few seconds apart and irregular. | Seen |

No LED was seen to act on its own: every change, including the dropouts,
was on all four at once. So during start-up the firmware drives the four
together, and the PHONE and ETHERNET lights say nothing yet about their
ports.

Readings (Hypothesis): the solid amber Power light is the reset or boot
stage before the application runs. The 1 Hz flashing is the application
waiting for the network, perhaps for a DHCP reply. The long-short-short
pattern is an error or "no network" code reached when that times out with
no link. Plugging in the cable restarts the wait, which ends once the unit
has an address. The brief dropouts may be network activity or a
supervisory task. The administration guide's LED table and a capture of
the Ethernet traffic would test these readings.

Compared with the SPA3000 (its `hardware.md` §1): two FXS ports and no
FXO port in place of one of each, and the same 5 V centre-positive supply.
The SPA3000's Power LED comes on alone first and its other three then
flash together; the PAP2T lights all four at once from power-on, with a
bicolour Power LED. The SPA3000's Power-position LED turned out to be
driven by the Ethernet controller; on the PAP2T, Power follows the same
start-up patterns as the other three.
The front-panel order is reversed: the SPA3000 has Power on the left, the
PAP2T on the right.

## 2. Open questions

1. Board name and revision, and the main chips: expected to be an ES3890F
   with SPA2002-like peripherals, from the firmware (see the
   [README](README.md)), and to be close to MAME's PAP2 (SPA3000
   `hardware.md` §2.6). To read once the case is open.
2. What the LED patterns in §1.1 mean (see the readings there), and
   whether the four LEDs are driven by the ES3890F's pins or something else.
