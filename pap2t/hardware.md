# PAP2T hardware

Status: draft, from photos of the closed case, a video of the front
panel during power-on, the stock web UI's Info page, and one photo of
each side of the bare board, square on and at an angle (all
2026-10-30), and close-ups of the flash and of the regulators U11 and U12. Clean-side document:
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
| `20261030-PAP2T-Board-Top.jpg` | Whole top side, out of the case, rear ports at the top as in the SPA3000's top-side photo |
| `20261030-PAP2T-Board-Underside.jpg` | Whole underside, turned over top to bottom so the rear ports are at the bottom, as in the SPA3000's underside photo |
| `20261030-PAP2T-Board-Top-Angled.jpg` | Top side at an angle, lit from one side, so that the chip markings read |
| `20261030-PAP2T-Board-Underside-Angled.jpg` | Underside at an angle, the same way |
| `20261030-PAP2T-U51-Flash-Closeup.jpg` | Underside, U51 (the flash) and the parts round it, rear ports towards the bottom |
| `20261030-PAP2T-U11-Regulator-Closeup.jpg` | U11, the RAM (U52A) and the board marking, at the left edge |
| `20261030-PAP2T-U12-Regulator-Closeup.jpg` | U12 and the parts round it, between the ES3890F, X1, L200 and LED D4 |

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

## 2. Main components

Read from the whole-board photos, square on and angled. Left, right, top and bottom are
as in `20261030-PAP2T-Board-Top.jpg`: rear ports along the top edge, the
front (LED) edge at the bottom.

The board carries the same core as the SPA3000 (its `hardware.md` §2):
ES3890F with a 27 MHz crystal, RTL8019AS with a 20 MHz crystal, the same
JXD magnetics and the same pair of glue-logic types (a '32 OR gate and a
'74 flip-flop). The FXO port's DAA and relay are gone, and the telephony
is two near-identical blocks, one per PHONE port, each built round a
Silicon Labs Si3215. MAME's PAP2 has two Si3210s instead (SPA3000
`hardware.md` §2.6).

### 2.1 Board

| Item | Value | Status |
|---|---|---|
| Board marking | Top side, left edge beside the RAM: `U54V016` / `REV.0 GP` / `500141`. `B-3` in the top right corner. | Read |
| PCB maker's marking | Underside: `DYnamic` `MO-V0` `E150630` `94V-0`, with a UL recognised-component mark and a blank white patch | Read |
| M-numbered features | M1–M10, in three kinds. M1, M2, M3 and M4: large plated holes, each ringed by stitching vias (M4 partly under C164). M7 (the round pad beside V015, by U52A) and M8 (beside SW2) on the top side, and M9 and M10 on the underside: bare copper discs about 2 mm across in a mask-free ring, with no visible trace or hole, in a diagonal pair on each side. M5 and M6: small plain holes with no visible copper ring. | Read (photos) |
| M-numbered roles | M1–M4 are mounting holes, grounded through their vias. M7–M10 are fiducials (pick-and-place alignment marks), not test points: the underside has its own pair because it carries SMT parts. M5 and M6 are tooling or locating holes. The shared `M` prefix is probably the CAD library's for mechanical parts. A fiducial should have continuity to no net; a reading on any of M7–M10 would make it a test point after all. | Hypothesis |
| Headers | None. Nothing like the SPA3000's 2×18 U2 footprint (its `hardware.md` §2.1) is on either side, and no TP-labelled test points have been found. | Read (photos and by eye) |
| Fixing in the case | As received, no screws held the board: it was kept in place mainly by the Ethernet and PHONE jacks through the rear panel, so it could shift against the underside of the case | Observed |
| SW2 | Footprint by the PHONE 2 jack on the right edge: plated holes, no part fitted as far as the photo shows | Read; "unfitted" to check by eye |
| V015 | Two small pads near M7, by the left edge beside the RAM | Read; role unknown |

### 2.2 Processor, memory and Ethernet

| Ref | Marking | Part | Role | Status |
|---|---|---|---|---|
| U13 | ESS `Visba3` `ES3890F` `F333` `AUVF25000` | ESS ES3890F, 128-pin PQFP, as on the SPA3000 (`F075` there) | Main CPU | Read |
| X1 | `27.000`, with a maker's mark read as `VXC` (perhaps `TXC`) above and a code below | 27.000 MHz crystal, HC-49 can with its leads formed onto surface pads, as Y1 on the SPA3000 | CPU clock, as on the SPA3000 and in MAME's PAP2 | Read (angled photo) |
| U52A, inside a larger U52 outline | `SEC` `KOREA` `628Y` `KM416C1204…-6`; one or two characters before `-6` not legible | Samsung (`SEC`) KM416C1204, TSOP-II. Hypothesis, from Samsung's part numbering, to check against a datasheet: 1M × 16 (2 MB) EDO DRAM, `C` for a 5 V part (`V` would be 3.3 V), `-6` for 60 ns. The SPA3000 has a 3.3 V K4E151612C. The longer U52 pad rows around it look like a second footprint for a longer package. | RAM. 2 MB fits MAME's PAP2 memory map. Whether it really runs from 5 V, and so how the ES3890F's 3.3 V bus drives it, is to measure. | Read; part details and supply Hypothesis |
| U51 (underside) | `MX` (Macronix logo) `06…` / `29LV160…B …` / `2R8…700…` / `TAIWAN`. The top is worn and scuffed, probably from rubbing on the case while the board was loose (§2.1); the character after `160` looks like `C`, and the rest of the second line is lost. | Macronix MX29LV160 (C or D revision), `B` for the bottom-boot version: 16 Mbit (2 MB) parallel NOR flash, 3 V, 48-pin TSOP, ×8 or ×16 selected by its BYTE# pin. Details to confirm against the datasheet. | Program storage. Twice the 1 MB that the SPA2002 layout uses and that MAME gives the PAP2 (an SST39VF080). Hypothesis: only the first 1 MB is mapped or used (an address line not wired, or the firmware ignores the rest), and BYTE# is tied for ×8, since the ES3890F's flash bus is 8 bits wide (SPA3000 `hardware.md` §2.5). Bottom boot puts the small sectors at the low end, which matters for where the firmware can erase and write. | Read (partial); part and size from the part number; use Hypothesis |
| U104 | Realtek `RTL8019AS` `F5C39G5` `LF20 TAIWAN` | Realtek RTL8019AS, 100-pin PQFP | Ethernet controller, as on the SPA3000 | Read |
| X2 | `20.000H20` | 20.000 MHz crystal, HC-49 | RTL8019AS reference clock (the datasheet calls for 20 MHz) | Read; role Datasheet |
| U105 | `JXD 2501H` `TF-1606DG` | JXD TF-1606DG | 10Base-T magnetics between the RTL8019AS and the RJ45, the same part as on the SPA3000 | Read; role Hypothesis |
| U15 | `…T32` (first character hidden) `AE482` `…330D`, 14-pin TSSOP | Hypothesis: a 74-series '32, quad 2-input OR gate, from the type number | Glue logic, the counterpart of the SPA3000's U6 (74HC32) | Read (partial); part Hypothesis |
| U151 | `…4AHC74` (first character hidden) `L1E9Z610` `UmG1150C`, with a maker's logo; 14-pin SOIC | 74AHC74, dual D flip-flop | Glue logic, the counterpart of the SPA3000's U10 (74HC74) | Read; part from the type number |

### 2.3 Telephony

Two blocks in the right half, each with the same parts, numbered in the
200s and the 150s. The 200 block (nearer the RJ45) has U200, inductor L200,
diode D200 and SOT-223 transistors Q216 and Q217; the 150 block has the
second Si3215, L150, D150, Q163 and Q164. Which block serves which PHONE
port is not known.

| Ref | Marking | Part | Role | Status |
|---|---|---|---|---|
| U200, and its twin in the 150 block | `Si3215-FM` `0946CFK0LM` `KR` and `Si3215-FM` `0644CCKABW` `KR`, each with a circled e3 (lead-free) mark | Silicon Labs Si3215 ProSLIC, QFN | One FXS (PHONE) port each: line feed, ringing and codec, PCM bus for audio, SPI for control. How the Si3215 differs from the SPA3000's Si3210 (and MAME's PAP2's) is to read from the datasheet before writing a driver. | Read; role Hypothesis until the datasheet is read |
| L150, L200 | `330` on both | 33 µH power inductors, the same marking as L2 on the SPA3000 | Hypothesis: each SLIC's DC-DC converter for the line battery voltage, as L2 on the SPA3000, each with a large electrolytic beside it | Read; role Hypothesis |
| D150, D200 (underside) | `ES1D` on D200; D150 reads `ES1D` or `ES1B` | ES1-series fast-recovery rectifiers, SMA | Hypothesis: each converter's rectifier, as the ES1D on the SPA3000's underside; their numbers pair them with L150 and L200 | Read; role Hypothesis |
| Q163, Q164, Q216, Q217 | `CZT5551` `CENTRAL` `K5D` on all four | Central Semiconductor CZT5551, a high-voltage NPN transistor in SOT-223, two per block | Hypothesis: part of each line's drive or the converter. The SPA3000's converter switch is a PNP (FZT955), so these do a different job or the Si3215 uses a different circuit; to check against the datasheet. | Read; role Hypothesis |
| Q157, Q159, Q210, Q212 (underside) | `2L` | SOT-23 transistors, two under each block, the same marking as Q4–Q7 on the SPA3000's underside | Hypothesis: part of each line's circuit, by placement | Read; role Hypothesis |
| Q209, Q211, and two more by the 150 block | Not read | SOT-23 transistors, two on the top side of each block | Unknown | Read (position) |
| — | — | No DAA, no relay | The PAP2T has no LINE port (§1) | Read |

### 2.4 Power and LEDs

| Ref | Marking | Part | Role | Status |
|---|---|---|---|---|
| U11 | `AMS1117` `ADJ` `G1068` | AMS1117 adjustable low-dropout regulator, SOT-223, at the left edge between the RAM and the board marking, with C29, C30, C35 and C36 round it | One supply rail, its voltage set by two resistors. Hypothesis: one of 3.3 V (flash, RAM, logic) and the ES3890F's core (2.5 V on the SPA3000, from an adjustable 1117 there too). Measuring the outputs powered settles which. | Read; role Hypothesis |
| U12 | `AMS1117` `ADJ` `G1068` | AMS1117 adjustable low-dropout regulator, SOT-223, beside the ES3890F, with R23 and another resistor marked `121` (120 Ω) next to it, C32, C37 and C39 | The other rail. The 120 Ω parts fit the usual upper resistor of a 1117 divider; the lower resistor, which fixes the voltage, is not yet identified. | Read; role Hypothesis |
| D1 | Not legible | SMD diode beside U12 | Unknown | Read |
| D4, D5, D101, LED2 | — | Four SMD LEDs along the front edge, left to right. LED2 is a larger part showing two dies. | Hypothesis: PHONE 1, PHONE 2, ETHERNET and Power, in the order of the front-panel windows (§1), with LED2 the bicolour Power LED (§1.1) | Read; roles Hypothesis |

## 3. Open questions

1. The markings §2 could not read: two characters of the RAM's part
   number, and the flash's revision letter and speed. The flash's manufacturer and device ID, read
   electrically, would settle its exact part; a first dump needs the
   flash read anyway (as for the SPA3000, its `flash.md` §7).
2. How much of the 2 MB flash is wired and used: whether the ES3890F's
   flash window reaches the second megabyte (trace the top address line,
   A19 in ×8 terms, from U51 to the ES3890F), and how BYTE# is tied.
3. What the Si3215 changes from the Si3210 that the SPA3000 and MAME's
   PAP2 use, from its datasheet; and the role of the four CZT5551 NPNs.
4. Whether the RAM is a 5 V part (KM416**C**), and if so how it is
   powered and driven.
5. What the LED patterns in §1.1 mean (see the readings there), and
   whether the four LEDs are driven by the ES3890F's pins or something else.
