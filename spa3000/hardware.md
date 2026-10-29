# SPA3000 hardware

Status: draft, from photos of both sides of one board and the case
(2026-10-27) and close-ups (2026-10-29), continuity tests on the unpowered board (2026-10-28), and DC
voltages on the powered board running the stock firmware (2026-10-28). Clean-side document: chip markings and public datasheets
only. See [`../clean-room.md`](../clean-room.md) for the process.

Each fact is marked:

- **Read**: printed on the part, board or case and legible in the photos.
- **Datasheet**: from the manufacturer's public datasheet for that part.
- **Public source**: from public open-source code or documentation about
  a related device (§2.6, §2.7). True of that device; to be checked on this one.
- **Measured**: tested on the board with a multimeter (continuity or
  diode test, unpowered unless stated).
- **Hypothesis**: inferred from placement or typical use; not confirmed.

Photos are in [`photos/`](photos/):

| File | Shows |
|---|---|
| `20261027-SPA3000-Board-Top.jpg` | Whole top side, in the case |
| `20261027-SPA3000-Board-Underside.jpg` | Whole underside: flash, RAM, glue logic |
| `20261027-SPA3000-Board-Telephony-Closeup.jpg` | Si3210 and DAA area |
| `20261027-SPA3000-DAA-Closeup.jpg` | Si3050, Si3019 and test points |
| `20261027-SPA3000-Power-Relay-Closeup.jpg` | Relay, regulators, inductor, LEDs |
| `20261027-SPA3000-U2-Y1-Closeup.jpg` | U2 footprint and CPU crystal |
| `20261027-SPA3000-Case-Rear-Labels.jpg` | Port labels on the case |
| `20261027-SPA3000-CPU-Ethernet-Closeup.jpg` | ES3890F, RTL8019AS and the Y2 marking |
| `20261027-SPA3000-Inductor-Bridge-Closeup.jpg` | L2, Q1, bridge D14 and the LEDs |
| `20261027-SPA3000-Regulators-Closeup.jpg` | REG1, REG2 and relay K1 |
| `20261029-SPA3000-JP1-JP2-Edge-Closeup.jpg` | Board edge between the power jack and U2: C25, JP2 (two holes), JP1 (three holes), `TWS` |
| `20261029-SPA3000-Underside-Rear-Closeup.jpg` | Underside, rear half: RJ11 jack pins, D2, Q19/Q20, the transistor rows |
| `20261029-SPA3000-Underside-Glue-Logic-Closeup.jpg` | Underside: U13, U3, U6, U10 and the parts around them |
| `20261029-SPA3000-Underside-DAA-Closeup.jpg` | Underside, right side: L7–L10, Q4–Q9, Q26, Z1 and the large resistors |

A cartoon map of both sides, drawn from these photos, for finding parts.
Positions are approximate. Source:
[`20261029-SPA3000-Board-Map.svg`](20261029-SPA3000-Board-Map.svg) (a
PNG copy is beside it).

![SPA3000 board map: top side with the rear ports at the top, and the underside flipped front-over-rear, colour-coded by function](20261029-SPA3000-Board-Map.svg)

Public datasheets used:

| Part | Document | Where |
|---|---|---|
| ES3890 | ESS Technology, *ES3890 Video CD Processor Product Brief*, SAM0487-031704, 6 pages. Pinout and pin description only; no registers, memory map or timing. No full datasheet has been found in public. | HTML copy at alldatasheet.com (`/html-pdf/126592/ETC1/ES3890F/`); PDF at insidegadgets.com (`/wp-content/uploads/2018/03/ES3890F.pdf`), behind a browser check |
| RTL8019AS | Realtek, *RTL8019AS Specification*, LS003.8 (20 August 2000). Used for the pin configuration (§3) and pin descriptions (§4). | Local `reference/` folder, `20261029-RTL8019AS_Datasheet.pdf`, from a university course page (fdi.ucm.es); source and hash in [`../clean-room.md`](../clean-room.md) |
| Si3210 | Silicon Labs, *Si3210/Si3211 ProSLIC Data Sheet*, rev 1.61. Used here for the external parts lists: Figure 10 and Table 13 (BJT/inductor DC-DC converter) and Figure 13 and Table 16 (discrete linefeed). Local `reference/` folder, `20261029-Si3210_Si3211_Datasheet_rev1_61.pdf`; source and hash in [`../clean-room.md`](../clean-room.md) |
| ES3880, ES3883 | Philips, *Service Manual FW-V355 & FW-M355* (3139 785 30029), chapter 8, "VCD – MPEG-01B module", pages 8-2 to 8-8: circuit notes, block diagram, ES3880 and ES3883 pin tables and the board schematic. Not a datasheet, but the only public pin tables found for these two parts. Used for §2.7. | manuals.plus (`/m/4c81edb8bef9b8d9e240dcf8a33d02d68d289db64084d8ec4cbf846ae15a6de2`), behind a browser check; read in the browser 2026-10-29, not saved |

Public source code used:

| Project | Files | Where |
|---|---|---|
| MAME (BSD-3-Clause) | `src/mame/skeleton/pap2.cpp` (Linksys PAP2 machine, by Joseph Khilfeh, added in pull request 12726, September 2024) and `src/devices/cpu/mipsx/mipsx.cpp` (MIPS-X CPU skeleton, by David Haywood) | github.com/mamedev/mame, master branch, read 2026-10-27 |

## 1. Board and case

| Item | Value | Status |
|---|---|---|
| Board name | `Q3000 REV F` | Read |
| PCB flammability rating | 94V-0, maker's mark "TJ-1" | Read |
| Versions reported by the stock web UI (Info page, 2026-10-28) | Software 3.1.20(GW), the version the format spec was written from; hardware 3.3.1(1448); client certificate installed; customization Open (not provider-locked) | Observed |
| Enclosure | Two-part grey plastic case. The board is held by screws through mounting holes (M2 at the lower-left corner, and one at the left edge). | Read (from photos) |
| Power input | DC 5 V, centre positive | Read (case label and polarity symbol) |
| Rear ports, left to right seen from behind | LINE (RJ11, FXO), PHONE (RJ11, FXS), INTERNET (RJ45), power jack | Read (case label) |
| Rear ports, left to right seen from above with the ports at the top | Power jack, RJ45, PHONE, LINE. LINE is the outer RJ11, next to the Si3050/Si3019 corner of the board. | Read |
| LEDs | Surface-mount, along the front edge of the top side: D1, D4, D7, D8 | Read |
| Front panel | Four light windows, left to right seen from the front: Power (⏻ symbol), INTERNET, PHONE, LINE | Read (case label) |
| LED positions | Left to right seen from the front: D1 (Power), D4 (INTERNET), D8 (PHONE), D7 (LINE). R20 sits beside D1 and R79 beside D4; D8 and D7 have no resistor beside them. | Read (from a photo with the board above the case) |
| LED wiring | D4, D7 and D8 share one trace on their board-edge side, which is their anode side; with the board unpowered, diode-testing any one of them lights all three at about 1.8 V. D1 (Power) is not on that trace: it reads 1.2 V one way and 0.9 V the other and does not light, so something conducts in parallel with it. | Measured |
| Test points | TP2, TP3, TP6 and T1 are plated holes. TP4's label is above the top-left corner of the RTL8019AS, with no hole beside it; not yet probed. A small pad labelled `TWS` (the name of ES3890F pin 59, §2.5) is below TP4, beside C73. | Read |
| Edge footprints between the power jack and U2 | Along the left board edge, going from the power jack towards U2: C25 (the large electrolytic), a single plated hole beside C25's label, JP2, a two-hole footprint (one pad square), then JP1, an unpopulated three-hole footprint, and then U2. Each has its own silkscreen box, with its label just outside it; an earlier reading took both labels for the two-hole footprint. JP1's square pad is its pin 1, at the JP2 end, with a thick bar beside it; pin 3 is at the U2 end. JP1 is in line with U2's outer row. The `TWS` pad and a second small unlabelled pad sit just inboard of JP1's pin 1; C73, R80, C32 and R23 sit between there and the RTL8019AS. | Read (close-up, 2026-10-29; labels re-read 2026-10-29) |
| JP1 wiring | Pins 2 and 3 have continuity to each other and to ES3890F pin 19 (CS1#, §2.5), unpowered. Pin 19 was counted by hand; pins 18 (CS0#) and 20 (CS3#) are its neighbours, so it is at least one of the bus chip selects. In the photos: pin 1 has a top-side trace to R80, passing C73's lower pad; pin 3 has a top-side trace that runs up past a via beside C32's lower end and on across the top of the board to ES3890F pin 19 (Read on the board, 2026-10-29). Pin 3 has no continuity to that via, so the trace only passes it (Measured, 2026-10-29). The via instead has continuity to the RTL8019AS's 9th pin on the edge facing U2, counted from the end away from its pin 1 (Measured, 2026-10-29). With the numbering in §2.2 that is pin 89; a short trace from the via to that pin is visible in a photo, and the tick marks agree (pin 85 and pin 90 ticks either side). Pin 89 is VDD (Datasheet), so the via is a supply via; it has nothing to do with JP1. On the underside the via has continuity to C21 (Measured, 2026-10-29), so C21 is probably a decoupling capacitor for the RTL8019AS's supply. The datasheet gives VDD as +5 V, and this board's logic is on 3.3 V (§2.4), so which rail feeds the RTL8019AS is worth measuring; on the underside, pins 2 and 3 each have a trace running along the board edge outside U2's outer row, towards U2's far end. | Continuity Measured (2026-10-29); traces Read (photos, 2026-10-29) |
| JP1 role | A selector jumper with its default hard-wired: pin 2 is the common, joined to pin 3 by a trace or inner layer, and pin 1 is the alternative. It routes a bus chip select, so it is not a serial console. Pin 1's R80 and C73 suggest either an alternative signal or an RC network. | Hypothesis |
| Test point voltages, powered and idle (no call), to REG2 pin 1 | TP2 0.11 V, TP3 0.95 V, TP6 0 V, T1 0 V, all steady. TP2 and T1 are beside the LINE jack, TP3 beside the Si3210 and TP6 beside the Si3050. None sits at a logic-high level, so none looks like an idle serial transmit line; TP2 and TP3 look like analog nodes in the line circuits. `TWS` pad 2.46 V, steady: held high, not a running word-select clock (which would average about half the rail). | Measured (powered); roles Hypothesis |

## 2. Main components

### 2.1 Processor, memory and glue logic

| Ref | Marking | Part | Role | Status |
|---|---|---|---|---|
| — | ESS `Visba3` `ES3890F` `F075` | ESS Technology ES3890F, a dual 32-bit RISC media processor sold for Video CD players; 128-pin PQFP. Pins in §2.5. | Main CPU. Runs the MIPS-X code described in the firmware spec. | Read; package Datasheet |
| Y1 | `TXC 27.0…` | 27.0 MHz crystal (TXC), HC-49/US can with its leads formed onto surface pads. Two unused plated holes about 4.9 mm apart on the underside, under Y1, look like an alternative through-hole HC-49 footprint; seen from below they sit just inboard of U2's inner row, level with roughly its 8th to 10th positions from pin 1 (Read, 2026-10-29). | CPU clock. 27 MHz is the standard reference for video SoCs of this kind. Routing: two traces run from Y1 to vias, cross to the underside, and come back up through two more vias with short traces to ES3890F pins 30 and 31 (XIN, XOUT; §2.5); each end of Y1 has continuity to one of them (Measured). Hypothesis: the underside part of the path goes through the load capacitors, probably C5 and C7 (a matched pair stacked just below the flash on the underside, C5 nearer the flash, with C12 above them and R5 beside them; each has a trace to a via), and perhaps a feedback resistor. | Read; pins 30 and 31 Measured; parts Hypothesis |
| U3 (underside) | `AM29LV081B-90EC` `0331GBM J` `© 1997 AMD` | AMD Am29LV081B: 8 Mbit (1 MB) parallel NOR flash, 3 V, 8-bit data bus only, 90 ns, 40-pin TSOP, sixteen uniform 64 KB sectors | Program storage. A 775 KB upgrade image fits with room to spare. | Read; details Datasheet |
| U13 (underside) | `SAMSUNG` `022` `K4E151612C-TC60` (read from a magnified crop of the underside photo; an earlier reading of `K4S161622…` was wrong) | Samsung K4E151612C: EDO DRAM, not SDRAM. 1M × 16 (16 Mbit, 2 MB), 3.3 V, TSOP-II; -60 is a 60 ns access time. EDO has no clock: it is driven by RAS/CAS strobes, which matches the ES3890F's DRAM pins (CAS#, DRAS0/1#, DWE#, DOE#, no clock; §2.5). 10 row and 10 column bits on MA[9:0] address exactly 1M words. `022` is probably a year-and-week date code. | Main RAM. 2 MB matches the 0x001FFFF0 value in the firmware spec's packed-section header (§4.1), which looks like a stack pointer at the top of RAM. | Read; type, size and speed from Samsung's part numbering, Datasheet still to check; row/column split Hypothesis |
| U6 (underside) | ST `74HC32` | Quad 2-input OR gate | Glue logic. Hypothesis: combines bus strobes and chip selects for the Ethernet chip or flash. | Read; role Hypothesis |
| U10 (underside) | ST `74HC74` | Dual D flip-flop | Glue logic. Hypothesis: clock division or bus timing for the Ethernet chip. | Read; role Hypothesis |
| Around U6 and U10 (underside) | — | C20, C81 and R21 beside U10; R14–R18 between U10 and U6; C22 and C34 beside U6; C21 between U2 and U10 (one end is on RTL8019AS pin 89, VDD, through the via by C32 on the top side; Measured, 2026-10-29); D6, a small glass diode, next to the rear end of U2. | Decoupling and pull-ups for the glue logic, by placement | Read (close-up, 2026-10-29); roles Hypothesis |
| U2 | Unpopulated footprint: two rows of 18 plated, tinned holes (36 in all). Pitch 2.54 mm, scaled from the HC-49/US body of Y1 (17 pitches = 43.2 mm). The two rows are about 2.54 mm apart (Measured, roughly, 2026-10-29), so this is a 2×18 pin-header outline, not a DIP-36 (which would have rows 15.24 mm apart). Pin 1 is the square pad at the top-row end nearest R6 and the M2 mounting hole. On the top side, a fan of about 15 parallel traces runs straight from the ES3890F's nearer pin row into the U2 pads, with no series resistors; a few more go to the R6–R9 group (R7 and R8 marked `472`, 4.7 kΩ; R9 marked `201`, 200 Ω). On the underside, the flash's bus traces run past U2 towards the CPU and do not visibly enter its pads. The ES3890F edge that faces U2 is pins 39–64 (§2.5). | — | U2 carries the 5 V input, so it is a connector, not an alternative memory footprint (Measured, see below). Its `U` prefix, rather than `J` or `P`, hints that the designer meant a module soldered on through pin headers (a daughterboard or debug pod) rather than a cable header, though ODMs often apply designators loosely. Hypothesis: a development port wired straight to ES3890F pins 39–64: address lines, the CD-ROM serial input and the audio serial output (§2.5). The general-purpose AUX pins are mostly on the far edge, so a serial console, if any, is less likely to be on U2 than first thought. Inner layers cannot be seen. Powered and idle, to REG2 pin 1, numbering each row 1–18 from the square-pad end (the square pad is inner 1; the inner row is nearer the CPU): inner 1 0.2 V and changing (an active signal, mostly low); inner 2 3.05 V steady (probably pulled up); outer 1 4.9 V steady (the 5 V input); outer 2 0.13 V steady. Positions 3–18 of both rows not yet measured. | Read (from photos); 5 V pin and voltages Measured (powered); role Hypothesis |

### 2.2 Ethernet

| Ref | Marking | Part | Role | Status |
|---|---|---|---|---|
| — | Realtek `RTL8019AS` `7526954` `L721F TAIWAN` | Realtek RTL8019AS | Ethernet controller: 10 Mbit/s only, NE2000-compatible register set, ISA-style 8/16-bit bus, 100-pin PQFP. Pin 1 is at the corner nearest both the ES3890F and U2: a larger silkscreen dot sits beside it, with smaller tick marks every five pins, and the package's own pin-1 dimple is in that corner. Counting anticlockwise, pins 1–30 run along the long edge facing the ES3890F, 31–50 along the short edge facing the PHONE side, 51–80 along the long edge facing the JXD, and 81–100 along the short edge facing U2. The datasheet's pin diagram has the same 30/20/30/20 split, and the layout fits it: the ISA address bus SA0–SA19 and the IORB/IOWB strobes (pins 5–30) face the ES3890F; the crystal pins X1 and X2 (50, 51) are at the corner next to Y2; the twisted-pair pins TPOUT± (45, 46) and TPIN± (58, 59) are either side of that corner, towards the JXD. The chip has no chip-select input: it decodes its I/O range from AEN (pin 34) and SA0–SA9, and the host reads and writes with IORB (29) and IOWB (30). | Read; role Datasheet; pin 1 Read (2026-10-29); numbering Datasheet, fit with the board Read |
| Y2 | `20R000` | 20.000 MHz crystal | RTL8019AS reference clock (the datasheet calls for 20 MHz) | Read |
| — | `JXD 1531H` `TF-1606DG` | JXD TF-1606DG | 10Base-T magnetics (isolation transformers and filters) between the RTL8019AS and the RJ45 | Read; role Hypothesis |
| — (underside, in front of the RJ45) | — | Two rows of six through-hole pins, each row in two groups of three, with one square pad at the U2 end of the row nearer the front. C23, C24 and C72 sit between the rows. | Hypothesis: the JXD TF-1606DG's pins, from their position under it | Read (close-up, 2026-10-29); identity Hypothesis |
| — (underside, behind the INTERNET jack) | — | Eight pins in two staggered rows of four | The RJ45's contacts | Read (close-up, 2026-10-29) |
| Q19, Q20 (underside) | — | Two SOT-23 transistors between the JXD pins and the PHONE jack pins, with R78, R81, R84 and D5 beside them | Unknown | Read (close-up, 2026-10-29) |

### 2.3 Telephony

| Ref | Marking | Part | Role | Status |
|---|---|---|---|---|
| U9 | `Si3210-KT` `0407ECCAAA` `PHILIPPINES` | Silicon Labs Si3210 ProSLIC | PHONE (FXS) port: line feed, ringing, codec and DTMF detection, plus a controller for an external DC-DC converter that makes the line battery voltage. PCM bus for audio, SPI for control. | Read; role Datasheet |
| U11 | `Si3050-KT` | Silicon Labs Si3050, system-side DAA | LINE (FXO) port, processor side: PCM and SPI to the processor | Read |
| U12 | `3019-FT` `0638FF` `FA09` | Silicon Labs Si3019, line-side DAA | LINE (FXO) port, line side. Linked to the Si3050 through Silicon Labs' capacitive isolation barrier. | Read |
| D14 | 4-pin SMD bridge rectifier, marking not legible | Diode bridge | Hypothesis: the polarity bridge in the Si3019's line circuit, so the DAA works whichever way round tip and ring are wired | Read (designator); role Hypothesis |
| Q22–Q25, FB1, FB2, RV1 | Transistors with large copper pads, ferrite beads and a surge protector, all beside the Si3019 | — | Hypothesis: the rest of the Si3019's external line circuit (hookswitch transistors, EMI filtering and surge protection) | Hypothesis |
| K1 | `EC2-5NU` `NEC` `JAPAN` | NEC EC2-5NU: DPDT signal relay with a 5 V coil, non-latching | Hypothesis: the power-fail or lifeline relay. Its two poles would switch tip and ring so that PHONE connects straight to LINE when the unit is off or the relay is released. | Read; role Hypothesis |
| L2 with Q1 | L2 `330`; Q1 `FZT955` | 33 µH power inductor; Q1 is a Zetex FZT955 high-voltage PNP transistor (SOT-223) | Hypothesis, strongly supported: the Si3210's DC-DC converter, which makes the negative line battery voltage. The Si3210 datasheet's BJT/inductor converter (Figure 10, Table 13) uses a 120 V PNP switch (FZT953 or FZT955 listed), a 1 A inductor, a 2222-type NPN driver and an ES1D rectifier into VBAT; this board has Q1 `FZT955`, L2 and an ES1D (below). | Read; parts list Datasheet; role Hypothesis |
| Q2–Q9 (both sides) | Small transistors near the Si3210. On the underside, Q4–Q7 are a row of SOT-23 parts marked `2L`, in the order Q7, Q4, Q6, Q5 going away from U6, with R38, R27, R40, C42 and C43 below them. Q8 and Q9 are larger three-legged parts marked `G1`, each with a row of three unused pads beside it, beside the flash on the side away from U2, behind L7–L10. | `2L` is the SOT-23 marking commonly used for the MMBT5401, a 120 V PNP transistor. `G1` is the marking commonly used for the MMBT5551, its 120 V NPN partner; Q8 and Q9 are larger than SOT-23, so this reading is less certain. | Hypothesis, strongly supported: the Si3210's discrete linefeed. The datasheet's discrete circuit (Figure 13, Table 16) uses four 120 V PNP 5401s (its Q1–Q4, SOT-23) and two 120 V NPN 5551s (its Q5 and Q6, SOT-223). That matches the four `2L` parts and the two larger `G1` parts here. | Read (underside close-ups, 2026-10-29); parts from the markings Hypothesis; parts list Datasheet; role Hypothesis |
| L7–L10 (underside) | — | Four two-terminal parts in a row, in the order L8, L7, L10, L9, towards the front edge, with R87 beside them. C102 and C103, two larger capacitors, sit beyond them on the side away from U2. | Unknown | Read (close-up, 2026-10-29) |
| Q26, Z1 and the large resistors (underside) | Q26: SOT-23 beside a large bare copper pad. Z1: glass diode. R97 marked `1071` (1.07 kΩ); R89 marked `000` (0 Ω). | — | Hypothesis: more of the Si3019's line-side circuit, from their position under the DAA. Also on that side: R90, R94, R95, R99, R106, R111, C94, C95, C98, C101, C104 and C111. | Read (close-up, 2026-10-29); role Hypothesis |
| Jack pins (underside) | — | The PHONE and LINE jacks' pins are numbered 1–8 as one set on the silkscreen: 1–4 under PHONE and 5–8 under LINE, each jack's four pins in two staggered rows of two. C40 is beside pin 8. | — | Read (close-up, 2026-10-29) |
| — (underside) | `ES1D` | ES1D, a general-purpose 1 A, 200 V fast-recovery rectifier in an SMA package | Near the rear edge, between the locating pegs of the PHONE and LINE jacks. No label is visible beside it; the nearby `D2` label sits by a two-hole footprint with a square pad, just in front of the PHONE jack pins. Hypothesis: the rectifier of the Si3210's DC-DC converter, the ES1D that the datasheet's converter uses (see L2 with Q1 above). It sits near the rear and L2 and Q1 near the front, so a trace or inner-layer path would join them. To confirm, check continuity from its pads to Q1's collector and to L2, unpowered. | Read (close-up, 2026-10-29); part Datasheet; role Hypothesis |

### 2.4 Power

| Ref | Marking | Part | Role | Status |
|---|---|---|---|---|
| REG2 | `AS1117L-33` | AS1117 3.3 V low-dropout regulator, SOT-223 | 3.3 V rail from the 5 V input, for the flash, RAM and logic. Powered: 3.4 V out, 4.9 V in. Pin 1 is ground; with the marking upright and the legs nearest, pins 1, 2, 3 run left to right, and the tab is the output. | Read; voltages Measured (powered); loads Hypothesis |
| REG1 | `1117C` `38 ADC` | Adjustable 1117-family low-dropout regulator, SOT-223. Powered: 2.53 V out (tab), 4.9 V in (pin 3), 1.27 V on the adjust pin (pin 1). Output minus adjust is 1.26 V, the 1117's reference, so it is the adjustable version, set to 2.5 V by two resistors; "38" is not a voltage. It runs from the 5 V input, not the 3.3 V rail. | Hypothesis: the ES3890F's 2.5 V core, PLL and video DAC rail (§2.5). To confirm, check continuity from REG1's tab to ES3890F VCC pins with the board unpowered. | Read; voltages Measured (powered); load Hypothesis |
| L12, L13 (underside) | `000` on both (0 Ω) | Zero-ohm links, beside the power jack's two solder tabs | Hypothesis: links in the 5 V input, perhaps positions for filter inductors | Read (close-up, 2026-10-29); role Hypothesis |

### 2.5 ES3890F pins

From the public product brief (see the datasheet table above). Pin numbers
are the package pins; signal names are ESS's.

| Pins | Signals | Function | Status |
|---|---|---|---|
| 1–9 | Analog ground, current-set, reference and filter pins, MIC1, MIC2, 5 V analog supply | Microphone ADC. Probably unused on this board. | Datasheet |
| 10–15 | AUX0[7:5], AUX3[2:0] | General-purpose I/O | Datasheet |
| 16–20 | LWR#, LOE#, CS0#, CS1#, CS3# | External bus write enable, output enable and three chip selects (active low) | Datasheet |
| 21–28 | LD[7:0] | External bus data, 8 bits | Datasheet |
| 30, 31 | XIN, XOUT | 27 MHz crystal (Y1) | Datasheet |
| 33–40, 43–54 | LA[19:0] | External bus address, 20 bits (1 MB, the size of U3). The brief does not say which bit is on which pin. | Datasheet |
| 55–57 | TDMFS, TDMDR, TDMCLK | Serial data input, meant for a CD-ROM drive: frame, data and clock | Datasheet |
| 58–61 | TBCK, SEL_PLL1/TWS, SEL_PLL0/TSD, MCLK | I²S-style audio serial output (bit clock, word select, data) and master clock. Pins 59 and 60 are also read as PLL mode straps and want pull-ups for normal operation. The brief gives no strap table; the ES3880's is in §2.7. | Datasheet |
| 62, 63, 67–69 | CAS#, DRAS1#, DRAS0#, DWE#, DOE# | DRAM control | Datasheet |
| 70–78 (and MA9) | MA[9:0] | DRAM multiplexed address. The brief gives no pin number for MA9. On the ES3880, MA9 and DOE# share one pin (§2.7), so MA9 is probably on the DOE# pin here too. | Datasheet; MA9 Hypothesis |
| 79–94 | DBUS[15:0] | DRAM data, 16 bits | Datasheet |
| 96 | REST# | External reset input (active low) | Datasheet |
| 100–115, 118–122 | AUX2[7:0], AUX1[7:0], AUX0[4:0] | General-purpose I/O; pin 120 (AUX0[2]) is input only. The brief suggests VCD uses (front-panel display, CD subcode) for some of them. | Datasheet |
| 125, 126 | VDAC, YDAC | Video DAC outputs. Unused on this board. | Datasheet |
| Supplies | VCC (core, 2.5 V) on 29, 42, 66, 95, 116; VSS on 32, 41, 65, 97, 117; VCC_P/VSS_P (PLL, 2.5 V) on 99/98; VCCV/VSSV (video DAC, 2.5 V) on 127, 128/123, 124; VPP (5 V) on 64; VDDA (analog, 5 V) on 8 | Power | Datasheet |

No UART or JTAG pins are listed. A serial console would have to use AUX
pins, through a function the brief does not describe or in software.

**Orientation on the board** (Measured, 2026-10-28): pin 1 is at the
corner of the package marked by the small round dot, the corner furthest
from U2 on the RTL8019AS side. With the usual anticlockwise numbering, pins
1–38 run along the long edge facing the RTL8019AS, 39–64 along the short
edge facing U2, 65–102 along the other long edge and 103–128 along the
short edge furthest from U2. Three things agree:

- the small dot is in the corner without an ejector-pin mark, where the pin
  1 marker usually is;
- the crystal Y1's two nets come back up from the underside through two
  vias whose traces land on the 8th and 9th pins from the U2 corner of the
  long edge facing the RTL8019AS. Counting from that corner as pin 38, these
  are pins 31 and 30 (XOUT, XIN). Numbering the other way round would make
  them pins 8 and 9, the analog supply and ground, so this also confirms the
  direction;
- R7 and R8 (4.7 kΩ) sit at the corner where pins 59 and 60 would be, which
  need pull-ups.

Confirmed with a continuity tester on the unpowered board:

- each end of Y1 has continuity to one of pins 30 and 31;
- pins 32, 41, 65, 97 and 117 (VSS) all have continuity to ground.

The ground pins are spread over all four edges, so the numbering is right
all the way round the package.

### 2.6 Related board: Linksys PAP2, from MAME

MAME's PAP2 machine (see "Public source code used" above) describes a
two-port adapter from the same family: ES3890F at 27 MHz, RTL8019AS, 2 MB
RAM, 1 MB flash (SST39VF080 there, not the AMD part) and two Si3210s. Its
comments say the SPA-3000 is "essentially the same hardware". The machine
is marked not working: it has a memory map but no peripherals, and its
CPU core does not execute instructions. Everything below is **Public
source** for the PAP2, unless marked otherwise.

| Item | PAP2 value | Fit with this board |
|---|---|---|
| LEDs | Four of the five LEDs are on ES3890F GPIOs: the power LEDs on AUX1[4] and AUX1[5] (pins 112 and 113), Phone 1 on AUX1[6] (pin 114), Phone 2 on AUX1[7] (pin 115). The Ethernet LED is driven from the RTL8019AS. | The pin numbers agree with §2.5 if AUX1[0] is pin 108. This board has four LEDs (§1); candidates for H4 |
| RTL8019AS bus | Pin 96 tied to ground, so the chip shares the flash's 8-bit data bus. Pin 65 tied to VDD, which disables Plug and Play. Its registers are selected when ES3890F pin 19 (CS1#) is low. | Consistent with LD[7:0] being 8 bits wide (§2.5); answers part of H3 if confirmed. In the RTL8019AS datasheet, pin 96 is IOCS16B, which at reset reads as SLOT16: low means an 8-bit slot. Pin 65 is JP: high selects jumper mode, where the I/O base, interrupt line and medium are set by pins read at reset instead of the EEPROM (Datasheet) |
| RTL8019AS EEPROM pins | Perhaps connected to AUX2[0–3]. The MAME author marks this as uncertain. | Hypothesis |
| RAM | 0x00000000–0x001FFFFF (2 MB) | Matches U13's size and the 0x001FFFF0 stack value in the format spec §4.1 |
| Flash | 0x1C000000–0x1C0FFFFF (1 MB), mirrored through 0x1CFFFFFF | Matches the 0x1CFEFFFC read in the processor spec §8 |
| Reset PC | Byte address 0x1CFFFF80, the last 128 bytes of the mirrored flash window. The author notes it might be 0x7FFFFF80 with another mirror. | Consistent with the boot section sitting at the top of the flash window (format spec §5, Placement) |
| On-chip registers | Guessed at 0x20000000–0x2000FFFF. The author suggests ESS's ES6008 datasheet may help. | Hypothesis; a lead for the peripheral specs ([stack plan](https://github.com/openmipsx/.github/blob/main/plan/open-source-stack.md) §3.2) |
| CPU clock | 81 MHz, guessed as three times the 27 MHz crystal, from the ES3880 datasheet | The ES3880's strap table (§2.7) gives 81 MHz when both SEL_PLL straps are high. Hypothesis, stronger than before; relevant to [stack plan](https://github.com/openmipsx/.github/blob/main/plan/open-source-stack.md) P2 |

### 2.7 Related chips: ES3880 and ES3883, from a Philips service manual

The ES3880 is ESS's earlier Video CD processor, a 100-pin PQFP, and the
ES3883 is its "Video CD Companion". Philips used the pair in the VCD
module of the FW-V355 micro system; its service manual (see "Public
datasheets used") has their pin tables and the board schematic.
Everything below is **Public source** for that board, unless marked
otherwise.

**The ES3890F looks like the two chips in one package** (Hypothesis).
Between them they have what the ES3890F brief lists (§2.5): the ES3880
has the RISC processor, external bus (LA, LD, LCS0#/1#/3#, LOE#, LWR#),
DRAM interface, TDM input and audio serial ports. The ES3883 has the
27 MHz crystal oscillator, the video encoder and DACs, the microphone
ADC and further AUX ports. The ES3890F brief has all of these, with the
crystal on the processor.

| Item | ES3880 / ES3883 | Fit with this board |
|---|---|---|
| PLL straps | ES3880 SEL_PLL[1:0] set the CPU clock: 00 bypass (CPUCLK pin used instead), 01 54 MHz, 10 67.5 MHz, 11 81 MHz. They share pins with the audio transmit data (AOUT, SEL_PLL0) and frame sync (ATFS, SEL_PLL1), as on the ES3890F. | The `TWS` pad (pin 59, SEL_PLL1) idles high (§1), and R7 and R8 sit by pins 59 and 60 (§2.5). If both straps read high, the CPU runs at 81 MHz. Pin 60 not yet measured. Hypothesis |
| DRAM | ES3880 pin 92 is DA9/DOE#: one pin is both the tenth DRAM address line and the DRAM output enable. The DRAM data bus is 16 bits, DBUS[15:0]. | Explains the missing MA9 pin in the ES3890F brief (§2.5). U13 needs ten address lines (§2.1). Check continuity from U13's A9 and OE# pins to the ES3890F's DOE# pin. Hypothesis |
| AUX pins | ES3880 AUX[7:0]; AUX0 and AUX1 are open-collector. ES3883 AUX0–AUX15, each with a VCD servo or display role or general control; AUX11–AUX15 can also be interrupt inputs or outputs. | The ES3890F has AUX0–AUX3 ports (§2.5). Which of them are open-collector is unknown |
| Companion register port | The ES3883 has an 8-bit register port named DSC: data DSC_D[7:0], clock DSC_C and strobe DSC_S, "for programming to access internal registers". On the Philips board DSC_D[7:0] are wired to the ES3880's LD[7:0], DSC_C to LCS1# (through a link, 4002, drawn dashed) and DSC_S to the ES3880's AUX5. | So in that design, the companion's registers sit in the CS1 window on the 8-bit data bus. On the PAP2 and here, CS1# also selects the RTL8019AS (§2.6, H3). If the ES3890F kept this arrangement inside the package, its companion registers are reached through CS1 too. Hypothesis |
| Reset | The ES3883 has a reset output, RSTOUT_B, which resets the ES3880 on the Philips board; its own reset input is RESET_B. | The ES3890F's REST# (pin 96) is an input (§2.5). Hypothesis: the merged chip has the reset output internally |
| Internal memories | ES3880 block diagram: a RISC processor with 2K×32 ROM and 512×32 SRAM; a separate "serial MPEG processor" with 64×32 ROM and 32×32 SRAM; a Huffman decoder; a DRAM DMA controller; on-screen display and video output registers. | Suggests a small on-chip ROM on the RISC side and a second, smaller processor. Hypothesis for the ES3890F |
| Clocks | ES3883: 27 MHz crystal on XIN/XOUT; its own PLL straps (on RSD and RWS) select the MPEG decoder clock: bypass, 27, 32.4 or 40.5 MHz. | The ES3890F has the crystal pins (30, 31; Measured, §2.5) |

## 3. What this means for new firmware

- **CPU**: confirms the ES3890F hypothesis in the firmware spec, §8.
- **Memory**: 1 MB of flash on an 8-bit bus and 2 MB of EDO DRAM. New firmware
  and its data must fit in 1 MB of flash, including any space kept for a
  fallback image and stored configuration.
- **Flash programming**: the Am29LV081B uses the standard AMD/JEDEC command
  set (unlock sequence, sector erase, byte program), all in its public
  datasheet. The firmware could write its own flash, and an external
  programmer could rewrite the chip if a bad image left the unit unusable.
  The chip is soldered down, so that means hot-air removal or a TSOP clip.
- **Ethernet**: the RTL8019AS is NE2000-compatible, so the driver can be
  written from Realtek's public datasheet and the well-known NE2000
  register model. The link is 10 Mbit/s only.
- **Telephony**: both ports use Silicon Labs parts with public datasheets.
  The FXS (Si3210) and FXO (Si3050 and Si3019) all connect to a host over a
  PCM audio bus and an SPI-style control bus. Hypothesis: both share one
  PCM bus and one SPI bus to the ES3890F, with separate chip selects.

## 4. Open questions

| # | Question | How to answer |
|---|---|---|
| H1 | Is there a serial console? The three-hole footprint by U2 is jumper JP1, and two of its pins are on an ES3890F bus chip select (§1), so it is not a serial header. TP2, TP3, TP6 and T1 sit in the telephony area and none idles high (§1). U2, TP4 and the AUX GPIO pins (§2.5) remain candidates. | Powered and idle, sweep U2's 36 holes and TP4 to ground for one idling at about 3.3 V. Then probe it with a scope or logic analyser at boot, looking for UART-like bursts at common baud rates |
| H2 | What is the U2 footprint? It carries 5 V (outer 1) and is a 2×18 header outline with rows 2.54 mm apart, so it is a connector of some kind, probably for a plug-on module given the `U` prefix, not alternative memory. It faces ES3890F pins 39–64 (§2.5). Next: finish the powered voltages for positions 3–18 of both rows (§2.1). | Use a continuity tester between U2 pins and ES3890F pins 39–64 first, then the flash and RTL8019AS pins |
| H3 | How are the SLIC, DAA and Ethernet wired to the ES3890F (bus, chip selects, interrupts, PCM)? On the PAP2 the RTL8019AS is on CS1# (pin 19) and the 8-bit data bus (§2.6). Here pin 19 reaches jumper JP1's pins 2 and 3 (§1). The RTL8019AS has no chip-select pin (§2.2), so CS1# must reach it through AEN (pin 34) or by gating its IORB/IOWB strobes (pins 29, 30). Hypothesis: U6, the 74HC32, ORs CS1# with the ES3890F's LOE# and LWR#, so IORB and IOWB go low only when CS1# is also low. | Confirm pin 19 against pins 18 and 20. Then, unpowered: ES3890F pin 19 and JP1 pin 2 against U6's inputs (pins 1, 2, 4, 5, 9, 10, 12, 13) and RTL8019AS pin 34 (AEN); U6's outputs (pins 3, 6, 8, 11) against RTL8019AS pins 29 and 30; ES3890F pins 16 and 17 (LWR#, LOE#) against U6's inputs. Then trace the rest, or probe with a logic analyser |
| H4 | GPIOs for the LEDs, the reset button and relay K1. On the PAP2 the LEDs are on AUX1[4–7], pins 112–115 (§2.6). | Continuity from each LED's resistor to pins 112–115 first; otherwise trace the board, or probe while the original firmware runs |
| H5 | Is there a reset button? None is fitted. The two-hole footprint is jumper JP2, not a switch footprint (§1). What JP2 selects is unknown: it could be a factory reset, a boot option or a test mode. It sits near the `TWS` pad, which is ES3890F pin 59 (SEL_PLL1/TWS, a PLL mode strap; §2.5) and idles at 2.46 V (§1), so a clock or PLL mode setting is another possibility (Hypothesis, from placement only). What JP1's pin 1 connects to is also open (§1). Hypothesis: factory reset is done only from the phone menu (IVR), as the public administration guide describes for SPA devices. | Unpowered, check continuity from both JP2 pads and JP1 pin 1 to ground, 3.3 V, the `TWS` pad, R80, C73 and the ES3890F's pins, starting with pins 59, 60 and 96 (REST#); check the administration guide |
| H6 | Does REG1 feed the ES3890F's VCC pins? Its output is measured: 2.5 V, adjustable (§2.4). U13's full marking is now read (§2.1). | Continuity from REG1's tab to ES3890F pin 29 or 95, unpowered |
| H7 | Can the flash be read in circuit, to back up the original contents before any experiment? | Check whether a TSOP clip fits, or desolder. This would produce dirty-side material, so do it on the dirty side. |
