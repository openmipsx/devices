# SPA3000 peripherals and start-up

Status: draft, 2026-10-30. Dirty-side findings written as facts for the
clean side. Contains no vendor code. See [`../clean-room.md`](../clean-room.md)
for the process and the provenance rows of 2026-10-29 and 2026-10-30.

This is how the stock firmware drives the ES3890F's on-chip registers and
the board's devices: addresses, values, order and timing. It is what the
first replacement firmware (the LED milestone) needs, alongside
[`hardware.md`](hardware.md) for the wiring and
[`../common/processor-es3890.md`](../common/processor-es3890.md) §10 for
interrupt entry and return.

Each fact is marked:

- **Observed**: seen in the vendor firmware's use of the register (what
  it writes, reads and waits for). The role is often inferred; where it
  is, it says Hypothesis.
- **Simulated**: seen when the vendor application ran on the simulator
  with a peripheral model built on the dirty side. The model answers only
  what the firmware needed to get past each wait, so a Simulated fact is
  only as good as the model. The simulator counts one cycle per
  instruction with no cache or bus waits, so **times set by busy-wait
  loops are lower bounds**. Times set by the 10 ms timer tick are
  reliable. Times are at 81 MHz, from the first instruction of the
  application unless marked "from reset".
- **Measured**: tested on the board ([`hardware.md`](hardware.md)).
- **Hypothesis**: inferred; not confirmed.

## 1. Address map

| Address | What | Status |
|---|---|---|
| 0x00000000–0x001FFFFF | DRAM, 2 MB | Observed |
| 0x02000000, 0x12000000 (+ 2 MB) | DRAM again. The application uses the 0x12000000 form for audio buffers and writes its fatal-error marker at both 0x001FFFF0 and 0x121FFFF0 | Observed; that these are aliases is Hypothesis (address bit 28 cached or uncached, as for flash) |
| 0x0C000000, 0x1C000000 | Flash, 1 MB, repeating through a 16 MB window ([`flash.md`](flash.md)) | Observed |
| 0x14000000–0x14000003 | System-control register file (§3). Address bits 1–0 are probably not decoded | Observed |
| 0x14000030 + 0x40·*n* | RTL8019AS register *n* (§8) | Observed |
| 0x18000000 | Probed for a second flash chip; nothing answers on this board | Observed probe; Simulated "nothing there" |
| 0x20000000–0x2000FFFF | On-chip registers (§2). The reset code sets r27 to 0x20000000 and nothing else writes r27, so firmware reaches them as offsets from r27 | Observed |

## 2. On-chip registers (0x20000000 + offset)

Only what the firmware's use makes reasonably clear. Offsets are from
0x20000000.

| Offset | Use | Status |
|---|---|---|
| 0x4000, 0x4004 | Timer or clock set-up. Reset code writes 0x4000 = 1, 0x4004 = 0x1BDF3E | Observed values; role Hypothesis |
| 0x4008 | Written 0xFF at start-up | Observed |
| 0x400C | Timer count (§4) | Observed; Hypothesis, strong |
| 0x4010 | Timer interrupt acknowledge (any write) | Observed |
| 0x4024 | Acknowledge for interrupt bit 13 (the system-control block's interrupt), after that block's own status is cleared | Observed; Hypothesis, strong |
| 0x4028 | Written 0, 6, 9 or 0xE before (and often after) nearly every GPIO and system-control access. Meaning unknown | Observed |
| 0x402C | GPIO port; only bit 2 ever changes, always as a low-then-high pulse: the **watchdog kick** (§5). Initial value 0x36 | Observed; Hypothesis, strong |
| 0x4030 | GPIO port carrying the SLIC and DAA control bus (§8). Initial value 0x44 | Observed |
| 0x4034 | Interrupt enable (§4) | Observed |
| 0x4038 | Interrupt status (§4) | Observed |
| 0x6000–0x610C | Cleared at start-up (§7) | Observed |
| 0x8000, 0x8100, 0x8104 | Clock or bus timing; reset code writes 0x8100 = 0x384, 0x8104 = 0x92 | Observed values; role Hypothesis |
| 0x8200–0x822C | A command engine: command words to 0x8214, then a wait for 0x8224 bit 0; 0x822C is a FIFO level (`& 0x3F`) | Observed; role Hypothesis |
| 0x8400–0x86FF | DMA columns at stride 0x100: each column offset (+0x28, +0x30, +0x38) holds count, address low and address high in the 0x84, 0x85 and 0x86 pages. Writing address high starts the column | Observed; start on address-high write Simulated |
| 0x87C0 | DMA column busy bits: bit 10 for column +0x28, bit 12 for column +0x30 (§8) | Observed use; meaning Hypothesis, strong |
| 0x87CC | DMA column interrupt enables (same bit positions) | Observed |
| 0x87D0–0x87EC | Engine control; 0x87E0 gets 0x6E23 then 0x6E21 (a pulse on bit 1) | Observed values; role unknown |
| 0x87F4 | Write 1s to stop DMA columns | Observed; Hypothesis, strong |
| 0xD008–0xD018, 0xF008–0xF0D0 | Set up once at start-up (§7); roles unknown | Observed |

## 3. System-control register file (0x14000003)

A byte-wide device outside the chip, reached through one address with a
two-step protocol (Observed):

1. Write an **address byte**: bits 7–1 are the register number, bit 0 is
   1 for a read and 0 for a write.
2. Write or read **one data byte** at the same address.

The firmware writes 0x20004028 = 0 before each access. Address bits 1–0
are probably not decoded (the boot code uses 0x14000000 for the same
thing).

| Reg | Use | Status |
|---|---|---|
| 0 | Control; written 0x00 at start-up | Observed |
| 1 | Read at start-up: bit 4 is a board-mode strap. Written as the read value OR 0x21 (0x31 on this board) | Observed; strap Hypothesis |
| 2 | Read at start-up: bit 0 is a second board-mode strap. Written 0x39 | Observed; strap Hypothesis |
| 3 | **Watchdog control**: bit 7 enables it (§5). The firmware writes 0xBC; it writes 0xFE and stops kicking to force a reset | Observed; Hypothesis, strong |
| 4, 5 | **Port 0** data and direction (§6) | Observed |
| 6, 7 | Port 1 data and direction | Observed |
| 0x15 | Interrupt enable mask for this block; bit 3 set when interrupts are enabled. Written 0x08 | Observed |
| 0x16 | Interrupt status, write 1 to clear. Written 0xFF at enable | Observed; Hypothesis, strong |
| 0x22, 0x23, 0x40, 0x63, 0x64 | Mode-dependent set-up (§7 step 11) | Observed values; roles unknown |
| 0x43, 0x46, 0x47 | **Clock/PLL settings**, from a per-mode table; latched by writing reg 0x70 = 0x00 then 0x80 | Observed; Hypothesis, strong |
| 0x50, 0x51 | Port 2 data and direction; unused | Observed |

**Board mode.** If reg 1 bit 4 reads 0 the firmware uses one mode;
otherwise reg 2 bit 0 picks between two more. One of them also
reprograms 0x20004000 = 0x81 and 0x20004004, which suggests a different
CPU clock. On this board both of the ES3890F's PLL straps (pins 59 and 60)
are pulled up (Measured, [`hardware.md`](hardware.md) §2.5), and the
firmware's values below are for the mode where reg 1 bit 4 = 1 and reg 2
bit 0 = 1 (Simulated: chosen to match the straps; which register bit is
which strap is not known).

## 4. Interrupt controller and timer

**Controller** (Observed). Pending = 0x20004038 (status) & 0x20004034
(enable) & 0x3FFF: 14 sources.

| Bit | Source |
|---|---|
| 1 | Timer |
| 7 | DMA columns (cascaded: 0x87C0 and 0x87CC, §8) |
| 13 | System-control block (§3 regs 0x15 and 0x16; acknowledge 0x20004024) |

The CPU side (vector at word 0, PSW mask bit, PC chain) is in
[`processor-es3890.md`](../common/processor-es3890.md) §10.

**Timer** (Observed sequence; counting direction and rate Hypothesis,
strong; Simulated). 0x2000400C counts **up by one per CPU cycle** and the
timer interrupt fires when it passes from negative to zero. Start-up
writes 0x2000400C = 0xFFF3A3F0 (−810,000). In each tick the handler
writes 0x20004010 (acknowledge), reads 0x2000400C and writes back
0x2000400C − 809,930, so the overshoot is kept and the period is
**809,930 cycles = 10.000 ms at 81 MHz**. That makes the CPU clock 81 MHz
in this board's mode (Hypothesis, strong: the handler's arithmetic is
exact only then).

## 5. Watchdog

- **Kick**: a low-then-high pulse on **0x2000402C bit 2**. The firmware
  writes 0x32 then 0x36, writing 0x20004028 = 9 before each. Observed;
  Hypothesis, strong.
- **Enable**: system-control reg 3 bit 7. The firmware writes 0xBC.
  Writing 0xFE there and then not kicking is how the firmware reboots
  itself: it masks interrupts and spins until the watchdog resets the
  chip. Observed; Hypothesis, strong.
- **Kick interval** (Simulated). Start-up kicks three times in its first
  11 µs, before the watchdog is enabled; enables it at about 33 µs and
  kicks twice at once; kicks once in the first timer tick. After that the
  timer tick kicks once every **501 ticks (5.01 s)**, and only if the main
  task has checked in. The longest gap seen is **5.032 s**.
- **Timeout: not known.** The firmware relies on it being longer than
  5.03 s (Hypothesis, strong). Whether the watchdog runs before reg 3
  bit 7 is set is not known either. The boot code never writes reg 3 and
  never kicks (Simulated, from reset).
- **For our firmware**: kick at least every 2–3 s, half the firmware's
  own interval, and do not set reg 3 bit 7 until the kick is running. If
  the board resets anyway without our code ever setting the bit, the
  watchdog is on from reset.

## 6. Port 0 (system-control regs 4 and 5)

Reg 4 is the data latch and reg 5 the direction: 1 = driven output,
0 = released (input). The firmware keeps shadows of both. To drive a pin
it **writes the data register first, then the direction register**, so
no pin glitches (Observed). Its pin helper takes a level: 0 clears the
data bit and sets the direction bit, 1 sets both, and 2 sets the data
bit and clears the direction bit, releasing the pin with its latch high
(Observed).

**Port 0 is the ES3890F's AUX1 port, bit *n* on AUX1[*n*] (pin 108 + *n*).**
The firmware's use of bits 0, 1, 2, 3 and 5 matches the board wiring of
pins 108, 109, 110, 111 and 113 exactly (Measured,
[`hardware.md`](hardware.md) §2.2, §2.3, H4). Bit 4 on pin 112 is a
prediction.

| Bit | Pin | Board net | State after the first write | Then (Simulated) |
|---|---|---|---|---|
| 0 | 108 | RTL8019AS RSTDRV | driven 1 (Ethernet held in reset) | → 0 at 0.111 s, after ≥ 3.0 ms high; stays 0 |
| 1 | 109 | Si3210 RESET# | driven 1 | 0 at 0.19 s for ≥ 12 µs, then 1 |
| 2 | 110 | Si3050 RESET# | driven 1 | 0 at 0.19 s for about 0.197 s (about 20 ticks), then 1, a short low pulse, 1 |
| 3 | 111 | Status LEDs D4, D7, D8 (through Q21) | released, latch 0 | driven 1 at 0.20 s. Then **0 = LEDs on, 1 = off**; first on at 0.73 s, then toggled every 50 ticks (0.50 s) |
| 4 | 112 (predicted) | Relay K1 driver | released, latch 0 | driven 1 at 1.44 s |
| 5 | 113 | Si3050 CS# | driven 1 (idle) | low only during Si3050 frames |
| 6 | 114 | unknown | driven 1 | never changes |
| 7 | 115 | — | never written | — |

The first write is data 0x67, then direction 0x67, at 0.108 s
(Simulated). Before it the direction is presumably all released
(Hypothesis). The LED polarity fits Q21 as a high-side switch
([`hardware.md`](hardware.md) H4), and the simulated timing matches the
board: the LEDs first light about half a second into start-up and then
blink at 0.5 s.

## 7. Start-up order

### 7.1 Reset code, from reset (Simulated; matches the code)

Before anything else, PSW = 0xF3 and r27 = 0x20000000. Then, in this
order, all word writes:

| # | Register | Value | Believed role |
|---|---|---|---|
| 1 | 0x20008100 | 0x384 | clock or bus timing (Hypothesis) |
| 2 | 0x200087EC | 0x20 | engine control |
| 3 | 0x20004000 | 0x1 | timer control |
| 4 | 0x20004004 | 0x1BDF3E | timer or clock setting |
| 5 | 0x200087D0 | 0x80 | engine control |
| 6 | 0x20008104 | 0x92 | clock or bus timing (Hypothesis) |
| 7–9 | 0x20008438, 0x20008538, 0x20008638 | 0x1FF, 0x1000, 0 | DMA column +0x38: count, address low, address high |
| 10–11 | 0x200087E0 | 0x6E23, then 0x6E21 | a pulse on bit 1; DRAM or engine set-up is a candidate (Hypothesis) |

No other peripheral access happens before the boot selector starts a
program, and the boot path does not touch the watchdog. No DRAM
controller programming shows beyond these writes, so either DRAM works
from reset or these writes set it up (Hypothesis). The program the boot
path starts repeats rows 1–11 itself.

### 7.2 Application start, first 35 µs (Simulated; about 2,600 instructions)

1. Interrupt enable 0x20004034 = 0.
2. One watchdog kick (not yet enabled).
3. Read the board-mode straps (§3). In the mode that reprograms them,
   write 0x20004000 = 0x81 and 0x20004004; not on this board.
4. Clear 0x20006000–0x20006068 (13 pairs at stride 0x10);
   0x20006104 = 0x3F, 0x20006108 = 0, 0x2000610C = 0.
5. 0x2000F0C4 = 0x3FF, 0x200087CC = 0, 0x20008000 = 0, 0x20008F00 = 0,
   0x20008204 = 0x400, 0x20008208 = 0x8000, 0x200087E0 = 0x6E23 then
   0x6E21, 0x20008000 = 0, 0x200087F4 = 0xFFBFFF (stop all columns),
   0x200087D0 = 0x80, 0x200087D4 = 0x2C, 0x200087D8 = 0x21,
   0x20008104 = 0x92, column +0x38 = 0x1FF / 0x1000 / 0.
6. 0x2000F0C0 = 0xFC then 0, 0x2000F0C4 = 0x3FF, 0x2000F00C = 0xFF0,
   0x2000F008 = 0xDE0, DRAM addresses into 0x2000F028, 0x2000F02C,
   0x2000F044 and 0x2000F080–0x2000F088, 0x2000F0C4 = 0x200,
   0x2000F0D0 = 0x800 then 0x1AE.
7. 0x20004008 = 0xFF, 0x2000400C = 0, 0x20004014 = 0,
   **0x2000402C = 0x36**, **0x20004030 = 0x44**.
8. DMA column +0x30 loaded with count 0xFFF6 and started.
9. 0x2000D014 = 0x620, 0x2000D018 = 0x77A, 0x2000D00C = 0xB8,
   0x2000D010 = a DRAM address, 0x2000D008 = 0x30.
10. Watchdog kick.
11. System control, in this order: reg 0 = 0x00, reg 0x23 = 0x31,
    reg 0x22 = 0x02, reg 2 = 0x39, reg 0x63 = 0x00, reg 0x40 = 0x60,
    reg 0x64 = 0x01, **reg 0x43 = 0xE4, reg 0x46 = 0xE0, reg 0x47 = 0x3F,
    reg 0x70 = 0x00 then 0x80**, reg 1 = 0x31. The 0x43/0x46/0x47 values
    belong to this board's mode.
12. 0x20004030 = 0x44 again, then a watchdog kick.
13. **Watchdog enabled: system-control reg 3 = 0xBC**, at about 33 µs,
    followed at once by two kicks.

### 7.3 Timer and interrupts (about 72 µs)

14. 0x20004010 written, 0x2000400C = 0xFFF3A3F0, 0x20004034 = 0x0002
    (timer only). PSW bit 4 is cleared straight after; tasks run with
    interrupts unmasked from then on.
15. First tick at 10.07 ms (§4).

### 7.4 Devices, after the scheduler starts (Simulated; lower bounds)

| Time | What |
|---|---|
| 0.16 ms | Flash identification at 0x18000000 (nothing) and 0x1C000000 ([`flash.md`](flash.md)) |
| 0.07–0.10 s | Flash maintenance. With erased boot-record sectors the application erases sector 0 and writes a new boot record (§9) |
| 0.106 s | 0x20008000 = 0 then 1, command words to 0x20008214, wait for 0x20008224 bit 0 |
| 0.108 s | Port 0 set-up: data 0x67, then direction 0x67 (§6) |
| 0.111 s | Port 0 bit 0 → 0 (Ethernet reset released) |
| 0.114–0.117 s | RTL8019AS reset and NE2000 initialisation (§8). System control reg 6 = 0x10, reg 7 = 0, reg 0x15 = 0x08, reg 0x16 = 0xFF. 0x20004034 = **0x2002** (adds bit 13) |
| 0.19 s | DMA column +0x30 restarted; 0x200087CC = 0x1000. Port 0 bits 1 and 2 → 0; bit 1 → 1 about 12 µs later |
| 0.20 s | Port 0 bit 3 driven high (LEDs off) |
| 0.22 s | Si3210 traffic starts on 0x20004030 (§8) |
| 0.37 s | 0x200087CC = 0x1400, 0x20004034 = **0x2082** (adds bit 7) |
| 0.39 s | Port 0 bit 2 → 1 (Si3050 reset released); Si3050 frames with port 0 bit 5 as chip select |
| 0.73 s | Status LEDs first on, then toggled every 0.50 s |
| 1.44 s | Port 0 bit 4 driven high (relay K1 driver) |

## 8. Devices

**DMA columns** (Hypothesis, strong; Simulated). 0x200087C0 bits 10 and
12 read 1 while column +0x28 or +0x30 is **busy**. A column's interrupt
(cascaded into 0x20004038 bit 7) stands while the column is idle and
enabled in 0x200087CC. Loading the column (count, address low, address
high) is the acknowledge. The opposite reading, bit set = done, gives an
interrupt storm.

**RTL8019AS** (Observed). Register *n* of the NE2000 register set is at
0x14000030 + 0x40·*n*, so CPU address bits 10–6 drive the chip's SA4–SA0
(Hypothesis, strong). Writes store the byte in both halves of a halfword;
reads are bytes. The firmware polls page 3 CONFIG0 bit 2 from the timer
tick, as a link check (Observed; Datasheet for the register).

**Si3210 and Si3050 control bus** (Observed; Simulated). Bit-banged on
0x20004030: bits 1, 2 and 3 are data, Si3210 chip select (active low) and
clock, and bits 5–7 are written alongside them as output enables. Frames
are the datasheet's: a read/write bit, a 7-bit register number and 8 data
bits, MSB first, sampled on the clock's rising edge. The Si3210 is
addressed without the daisy-chain prefix throughout. The firmware
identifies it by direct register 8 reading 0x02 and register 11 reading
0x33, the datasheet reset values, and stops with a fatal error if they do
not match. The Si3050 uses the same lines with port 0 bit 5 as its chip
select (§6).

**Flash**: identified by the AMD autoselect sequence; see
[`flash.md`](flash.md).

**Voice coding** (Simulated). The application does G.711 µ-law and A-law
coding itself, on the MIPS-X core, to and from 16-bit linear samples; no
peripheral is involved. Its encoder is bit-exact with ITU-T G.711 for all
65,536 inputs of both laws, keeping the top 14 (µ-law) or 13 (A-law) bits
of each sample. Its decoder returns the G.711 output values scaled to
16 bits. So the Si3210's PCM interface is presumably used in a linear
mode (Hypothesis), and new firmware needs its own G.711 coder.

## 9. Side effects to know before running stock firmware

- If both boot-record sectors are erased, the stock application rewrites
  sector 0 during start-up (Simulated). Do not run it against a flash
  whose boot sectors are blank unless that is wanted.
- The firmware's fatal-error path masks interrupts, writes 0xFE to
  system-control reg 3 and spins: a fatal error looks like a hang
  followed by a watchdog reset (Observed).

## 10. Open questions

| # | Question | How to answer |
|---|---|---|
| P1 | The watchdog's timeout, and whether it runs before system-control reg 3 bit 7 is set | Our own code on the board: kick, then stop kicking for a measured time, with and without setting the bit |
| P2 | What 0x20004028 does | Board experiments once our code runs |
| P3 | Real timing of the DMA columns, the 0x8200 engine and the counters at 0x2000E014 and 0x2000F0A8 | Board, logic analyser |
| P4 | Which system-control bits are which PLL strap, and what the other board modes change | Board, or a reading taken in reset |
| P5 | The Si3050's register traffic (not modelled) | Board, logic analyser on the SPI lines |
