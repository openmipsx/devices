# SPA3000 peripherals and start-up

Status: draft, 2026-10-31; first Measured facts from our own code on the
board added 2026-11-02 (§4, §5, §6). Dirty-side findings written as facts for the
clean side. Contains no vendor code. See [`../clean-room.md`](../clean-room.md)
for the process and the provenance rows of 2026-10-29 to 2026-10-31.

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
| 0x3800–0x39FC | 128 words written once at start-up: entries 0, 1, 2 = 0x03F07F00, 0x03700202, 0x07F00200, the rest 0x03F07F00. With the Si3210 on PCM slots 0 (receive) and 2 (transmit), read as the **TDM highway's 128-slot map**, slot 1 being the Si3050 | Observed values; role Hypothesis |
| 0x3A00–0x3A7C | 32 words written with it: ten control words, then a 26-entry reciprocal table (32768 / (1 + k/32) for k = 0 to 25), then zeros | Observed; role unknown |
| 0x3B80–0x3B90 | Control of the two blocks above: 0x3B90 pulsed 0xF then 0 twice; 0x3B84 stepped through 1, 0x11, 1, 5, 1, 3, 1, 2, 0, 8; 0x3B88 = 0x10; 0x3B80 = 0, then 0x81 last | Observed |
| 0x4000, 0x4004 | Timer or clock set-up. Reset code writes 0x4000 = 1, 0x4004 = 0x1BDF3E | Observed values; role Hypothesis |
| 0x4008 | Written 0xFF at start-up | Observed |
| 0x400C | Timer count (§4) | Observed; Hypothesis, strong |
| 0x4010 | Timer interrupt acknowledge (any write) | Observed |
| 0x4024 | Acknowledge for interrupt bit 13 (the system-control block's interrupt), after that block's own status is cleared | Observed; Hypothesis, strong |
| 0x4028 | **Interrupt suppress** (§4). Writing S (bits 3–0) holds off interrupts for the next 15 − S instructions: 0 = 15, 0xE = 1, 0xF = none. The firmware writes 0 before its PSW read-modify-write, before each two-step system-control access (§3), in the idle loop, in the context switch and as the last store of every interrupt handler before `jpcrs`; it writes 9 or 6 before each GPIO read-modify-write group and again before the next | Observed use; meaning Hypothesis, strong (the firmware's handler tail is the one a public description of the predecessor chip's register shows; see §4) |
| 0x402C | GPIO port; only bit 2 ever changes, always as a low-then-high pulse: the **watchdog kick** (§5). Initial value 0x36 | Observed; Hypothesis, strong |
| 0x4030 | GPIO port carrying the SLIC and DAA control bus (§8). Initial value 0x44 | Observed |
| 0x4034 | Interrupt enable (§4) | Observed |
| 0x4038 | Interrupt status (§4) | Observed |
| 0x6000–0x610C | Cleared at start-up (§7) | Observed |
| 0x8000, 0x8100, 0x8104 | Clock or bus timing; reset code writes 0x8100 = 0x384, 0x8104 = 0x92 | Observed values; role Hypothesis |
| 0x8000 | Also the DSP's **run control** (§8, "Program loading"): 0 then 1 at DSP init, 3 while a program is being loaded, 1 to run | Observed; role Hypothesis, strong |
| 0x8200–0x822C | The port of an on-chip **DSP coprocessor** (§8, "Audio path"): command words to 0x8214, then a wait for 0x8224 bit 0; 0x822C is a FIFO level (`& 0x3F`) | Observed; role Hypothesis |
| 0x8400–0x86FF | DMA columns at stride 0x100: each column offset (+0x28, +0x30, +0x38) holds count, address low and address high in the 0x84, 0x85 and 0x86 pages. Writing address high starts the column. The count is a negated length. Address high holds bits 31–11 of the RAM address; address low holds bits 10–2 as a word offset in its bits 8–0 (the d2a column also sets bit 11 there, meaning unknown) | Observed; start on address-high write Simulated; address split Hypothesis, strong (read this way, the DMA columns and the DSP's block moves, §8, name the same buffers) |
| 0x87C0 | DMA column busy bits: bit 10 for column +0x28, bit 12 for column +0x30 (§8) | Observed use; meaning Hypothesis, strong |
| 0x87CC | DMA column interrupt enables (same bit positions) | Observed |
| 0x87D0–0x87EC | Engine control; 0x87E0 gets 0x6E23 then 0x6E21 (a pulse on bit 1) | Observed values; role unknown |
| 0x87F4 | Write 1s to stop DMA columns | Observed; Hypothesis, strong |
| 0xD008–0xD018 | Set up once at start-up (§7); roles unknown | Observed |
| 0xF008–0xF088, 0xF0C0, 0xF0C4 | Written once at start-up: 0xF008 = 0xDE0, 0xF00C = 0xFF0, 0xF028 = 0x3DF80, 0xF02C = 0x3FFC0, 0xF044 = 0x1FE0, 0xF080 = 0x33C0, 0xF084 = 0x37D0, 0xF088 = 0x7FE0; 0xF0C0 = 0xFC then 0; 0xF0C4 = 0x3FF then 0x200. Four of the values are addresses inside ordinary application code and data (not entry points), and the sizes involved are 0x2000 − 0x20 and 0x8000 − 0x20. Look like base and limit pairs of some unit; **not** a second core's program windows. | Observed; role unknown |

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
| 6, 7 | Port 1 data and direction. Port 1 bit 4 is pin 104, AUX2[4], so port 1 is probably AUX2 (§8) | Observed; pin Measured; port 1 = AUX2 Hypothesis |
| 0x15 | Interrupt enable mask for this block; bit 3 set when interrupts are enabled. Written 0x08 | Observed |
| 0x16 | Interrupt status, write 1 to clear. Written 0xFF at enable. Bit 5 is the RTL8019AS interrupt (§8) | Observed; Hypothesis, strong |
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
| 13 | System-control block (§3 regs 0x15 and 0x16; acknowledge 0x20004024). The RTL8019AS arrives here as reg 0x16 bit 5 (§8) |

The CPU side (vector at word 0, PSW mask bit, PC chain) is in
[`processor-es3890.md`](../common/processor-es3890.md) §10.

**Interrupt suppress, 0x20004028** (Hypothesis, strong). A 4-bit
register: writing S suppresses interrupts for the next 15 − S
instructions (0 gives 15, 0xE gives 1, 0xF none). The ES3890's
predecessor, IIT's Vision Controller, had the same register in its
RISC-interface block, with these semantics and the same handler tail,
in a public account by one of its engineers (Hedley Rainnie, "Stanford
Mips-X @IIT", hrrzi.com, 2017; its listing shows the register at 0x4014
while its machine word encodes 0x4028, so which offset that chip used
is unresolved; the ES3890 firmware uses 0x4028 everywhere and writes
0x4014 once, with 0, at reset). On the SPA3000 the firmware writes 0
before every sequence that must not be split: the PSW mask update that
is its critical-section primitive, the address-then-data accesses to
the system-control file (§3), the idle loop, the context switch and the
return from every interrupt handler (`st 0x4028, r0`, two no-ops, then
`jpcrs`, `jpc`, `jpc`). The GPIO drivers write 9 (6 instructions) or 6
(9 instructions) before each read-modify-write-store on 0x402C or
0x4030 and re-arm before the next group; the groups between re-arms are
5 to 8 instructions. Any handler we write needs the same store before
`jpcrs`; see `processor-es3890.md` §10 for why.

**Timer** (Observed sequence; counting direction and rate Hypothesis,
strong; Simulated). 0x2000400C counts **up by one per CPU cycle** and the
timer interrupt fires when it passes from negative to zero. Start-up
writes 0x2000400C = 0xFFF3A3F0 (−810,000). In each tick the handler
writes 0x20004010 (acknowledge), reads 0x2000400C and writes back
0x2000400C − 809,930, so the overshoot is kept and the period is
**809,930 cycles = 10.000 ms at 81 MHz**. That makes the CPU clock 81 MHz
in this board's mode (Hypothesis, strong: the handler's arithmetic is
exact only then).

**Measured, 2026-11-02** (our own code, mipsx-sdk `examples/blink`): a
delay loop that waited for 0x2000400C to advance by 81,000,000 counted
1.00 s by stopwatch (ten 2.000 s LED periods in 20 s), after only the
three plain timer writes of §7.2 step 7 and with no interrupt enabled.
So the counter runs from those writes alone, counts up, and at 81 MHz
within the stopwatch's few per cent.

## 5. Watchdog

- **Kick**: a low-then-high pulse on **0x2000402C bit 2**. The firmware
  writes 0x32 then 0x36, writing 0x20004028 = 9 before each. Observed;
  Hypothesis, strong.
- **Enable**: system-control reg 3 bit 7. The firmware writes 0xBC.
  Writing 0xFE there and then not kicking is how the firmware reboots
  itself: it masks interrupts and spins until the watchdog resets the
  chip. Observed; Hypothesis, strong. Both installers write reg 3 with
  bit 7 **clear** (0x3C, the start-up shadow value) before they erase
  and program flash, and 0xBC again when the install is over or refused;
  no kick happens in between (Observed in the code of A and B;
  [`flash.md`](flash.md) §1). So the stock firmware treats bit 7 as an
  on/off switch it can use at run time, and a stock install of the
  application runs for several seconds with the watchdog off.
- **Kick interval** (Simulated). Start-up kicks three times in its first
  11 µs, before the watchdog is enabled; enables it at about 33 µs and
  kicks twice at once; kicks once in the first timer tick. After that the
  timer tick kicks once every **501 ticks (5.01 s)**, and only if the main
  task has checked in. The longest gap seen is **5.032 s**.
- **Timeout: not known.** The firmware relies on it being longer than
  5.03 s (Hypothesis, strong). Whether the watchdog runs before reg 3
  bit 7 is set is not known either. The boot code never writes reg 3 and
  never kicks (Simulated, from reset).
- **Measured, 2026-11-02** (our own code, mipsx-sdk `examples/blink`
  variant 1): a program that kicked as above at least once a second and
  never wrote reg 3 ran for more than 10 minutes without a reset. So
  either the watchdog is off until bit 7 is set, or the kick as
  described satisfies it; which of the two is still open (P1).
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

**Measured, 2026-11-02** (our own code, mipsx-sdk `examples/blink`): the
two-step protocol of §3, with 0x20004028 = 0 before each access and
none of the stock firmware's other system-control or on-chip start-up
writes (§7.2 steps 3–6, 8, 9, 11), reached port 0: writing data then
direction with only bit 3 set left D4, D7 and D8 dark, and toggling
the data bit then blinked all three together, with D1 unaffected. So
reg 4 and 5 are port 0 as described, bit 3 drives the three status
LEDs through Q21, and nothing else needs setting up for it.

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
| 0.07–0.10 s | Flash maintenance. With erased boot-record sectors the application erases sector 0 and writes a new boot record (§9). With a valid boot record it writes nothing, and the first DHCPDISCOVER comes at 0.083 s instead of 0.117 s |
| 0.106 s | 0x20008000 = 0 then 1, command words to 0x20008214, wait for 0x20008224 bit 0 |
| 0.108 s | Port 0 set-up: data 0x67, then direction 0x67 (§6) |
| 0.111 s | Port 0 bit 0 → 0 (Ethernet reset released) |
| 0.114–0.117 s | RTL8019AS reset and NE2000 initialisation (§8). System control reg 6 = 0x10, reg 7 = 0, reg 0x15 = 0x08, reg 0x16 = 0xFF. 0x20004034 = **0x2002** (adds bit 13) |
| 0.117 s | One transmit with a byte count of 0, then the first DHCPDISCOVER (§8, "Network behaviour"), without waiting for the link |
| 0.19 s | DMA column +0x30 restarted; 0x200087CC = 0x1000. Port 0 bits 1 and 2 → 0; bit 1 → 1 about 12 µs later |
| 0.20 s | Port 0 bit 3 driven high (LEDs off) |
| 0.22 s | Si3210 traffic starts on 0x20004030 (§8) |
| 0.37 s | 0x200087CC = 0x1400, 0x20004034 = **0x2082** (adds bit 7) |
| 0.39 s | Port 0 bit 2 → 1 (Si3050 reset released); Si3050 set-up with port 0 bit 5 as chip select, ending with the line side powered up (§8, "Si3050 DAA") |
| 0.73 s | Status LEDs first on, then toggled every 0.50 s |
| 1.39 s | Si3050 line check: LINE port off hook for about 150 ms (§8) |
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

**RTL8019AS receive** (Hypothesis, strong; set-up Simulated). The stock
firmware takes received frames by interrupt, not by polling. It sets
PSTART = 0x46, PSTOP = 0x60, BNRY = 0x46, CURR = 0x46 (pages 0x40–0x45
are the transmit buffer, TPSR = 0x40), DCR = 0x4A, RCR = 0x04 and
IMR = 0x11 (PRX and OVW), and keeps its own next-packet pointer,
starting at 0x46. The chip's interrupt reaches the CPU through the
system-control block: reg 0x16 **bit 5** is the Ethernet interrupt,
cascaded into 0x20004038 **bit 13** (§4). The firmware enables it with
reg 0x15 = 0x08, reg 0x16 = 0xFF and 0x20004034 bit 13, and makes port 1
bit 4 an input. The chip's INT0 interrupt output goes to ES3890F pin 104, AUX2[4] (Measured; see
[`hardware.md`](hardware.md) §2.2), so port 1 bit 4 is that pin if port 1
is AUX2 (Hypothesis, strong: port 0 is AUX1, §6). Which reg 0x15 bit masks reg 0x16
bit 5 is not known. The handler reads reg 0x16, writes the value
back, writes 0x20004024 = 0, and then, while ISR & 0x1F is non-zero,
writes the ISR value back to clear it and acts:

- **OVW**: stops the chip and resets the ring to the start-up values.
  The ring is emptied, not drained: frames still in it are lost.
- **PRX**: reads CURR from page 1 and, for each frame from its pointer
  up to CURR, remote-DMA-reads the 4-byte header, checks the length
  (64–1522) and the next-page byte, reads the frame without its CRC, and
  sets BNRY to the next page minus one (0x5F after a wrap). A bad header
  seen twice skips one page.

While transmitting it holds IMR = 0.

**Network behaviour** (Simulated: the application on a Linux TAP with a
DHCP server, from reset). For a replacement firmware these are what
the stock firmware does, not requirements:

- **A zero-length transmit at start-up.** Before its first DHCPDISCOVER
  the firmware issues one transmit command with a byte count of 0. What
  the RTL8019AS puts on the wire for that is not known (Hypothesis:
  nothing, or a runt). A simulator's Ethernet model should accept it
  without failing; new firmware need not copy it.
- **DHCP does not wait for the link.** The first DHCPDISCOVER (342
  bytes) goes out at 0.117 s (0.083 s when the flash has a valid boot
  record, §7.4) even while the CONFIG0 link check reports no link (as
  the model's did), and
  is retried at about 2.8 s if unanswered. An offer is answered with a
  DHCPREQUEST within about 0.5 ms. After the lease it sends two
  gratuitous ARPs for its own address.
- **The packet that causes an ARP miss is dropped.** When the firmware
  has to reply to a host whose MAC it does not know, it sends an ARP
  request and discards the packet it was answering; it does not queue
  it. So the first echo request from a host not yet in its ARP cache
  goes unanswered (4 of 5 pings answered, about 0.6 ms each).
  Presumably the same holds for the first packet of any other exchange
  with such a host (Hypothesis).
- The web server was answering on port 80 by 1.7 s, the first time it
  was asked.
- **An inbound call to the FXS port** (Simulated: a SIP peer on the TAP,
  the firmware registered on line 1, an off-hook phone modelled on the
  Si3210; vendor behaviour, not a requirement). An INVITE offering PCMU
  and telephone-event to the registered contact was answered 100 Trying
  and 180 Ringing at about 8.03 s simulated, without an authentication
  challenge, and the Si3210 linefeed entered ringing in the same tick
  (about 8 s after the INVITE was sent, as the peer's clock and the
  board's run at different speeds). With the phone taken off hook 1 s
  into ringing, the firmware saw ring trip, set the linefeed to forward
  active and saw loop closure within about 45 ms, and sent 200 OK from
  its DHCP lease address 59 ms after the hook. The answer chose PCMU,
  advertised a packetisation time of 30 ms and offered the
  telephone-event and a vendor named-signal-event payload. RTP then went both
  ways at 20 ms and 160 bytes per packet from the board's lease address
  (contiguous sequence numbers, one SSRC, marker never set), regardless of
  the advertised 30 ms; the board's payload was mu-law zero (silent)
  because the samples come from the DSP, which is not modelled. 140
  packets went out in the 2.8 s of simulated speech, all received.
  Packets to the board were received by the Ethernet chip; what the
  firmware did with them is not visible. A BYE from the peer was answered
  200 OK within about 60 ms; the board did not hang up on its own.

**Si3210 and Si3050 control bus** (Observed; Simulated). Bit-banged on
0x20004030: bits 1, 2 and 3 are data, Si3210 chip select (active low) and
clock, and bits 5–7 are written alongside them as output enables. Frames
are the datasheet's: a read/write bit, a 7-bit register number and 8 data
bits, MSB first, sampled on the clock's rising edge. The Si3210 is
addressed without the daisy-chain prefix throughout. The firmware
identifies it by direct register 8 reading 0x02 and register 11 reading
0x33, the datasheet reset values, and stops with a fatal error if they do
not match. The Si3050 uses the same lines with port 0 bit 5 as its chip
select (§6; details below).

**Si3210 reset and restart** (Simulated; Datasheet for the chip). The
firmware pulses RESET# (port 0 bit 1, §6) once per start, at 0.19 s,
before the identification check above. Its set-up later writes direct
register 8 to 0x00, so the check passes on a second start only because
that start's RESET# pulse puts the register back to 0x02. A soft
restart of the unit (§9) leaves the Si3210 powered, and the restarted
firmware relies on the pulse: in the simulation, before RESET# was
modelled, the restarted firmware read DR8 = 0x00 and stopped in its
fatal-error path. New firmware should pulse RESET# on every start and
not assume the SLIC's registers are at their reset values otherwise.
Whether the ES3890F's own reset also resets the Si3210 on the board
is not known (pin 109 is a plain AUX pin, [`hardware.md`](hardware.md)
§2.3).

**Si3210 set-up** (Simulated, from the registers the firmware leaves
set; meaning from the datasheet):

- PCM: DR1 = 0x3D, **16-bit linear** samples with PCM enabled; transmit
  start count (DR2, DR3) 2 and receive start count (DR4, DR5) 0, in PCLK
  periods.
- Loop-closure debounce DR69 (LCDI) = 0x18, 30 ms; the loop-closure
  threshold indirect register LCRT is written 0x1000.
- After the set-up the linefeed (DR64) is 1, forward active.
- **The firmware polls for the hook switch**: it reads DR68 (LCR) from
  its tick and does not take the chip's interrupt. It never clears LCIP
  in DR19, so if INT is wired, it would stay asserted after the first
  hook change (it enables that interrupt: LCIP did set). When the phone
  goes off hook, LCR sets after the 30 ms debounce, and the firmware then
  writes DR92 = 0xFF and writes the linefeed as forward active.
- When the LINE port shows a ring (Si3050 register 5 ring-detect bits),
  the firmware rings the PHONE port: DR64 = 4 (ringing) for about 2 s at
  a time, and DR64 = 2 (forward on-hook transmission) once between
  rings. Seen only in a run where every DAA read returned 0xFF, so the
  ring was not real and the cadence means little.

**Si3050 DAA** (Hypothesis for the wiring, from the firmware's traffic
alone; Datasheet for the chip; Simulated for the sequence). The CS# and
RESET# pins are Measured ([`hardware.md`](hardware.md) §2.3).

- It shares the Si3210's data line (bit 1) and clock (bit 3) on
  0x20004030, with bit 5 as the data line's output enable as before. Its
  chip select is port 0 bit 5 and its RESET# port 0 bit 2, both active
  low (§6).
- **SDI and SDO are wired together** on bit 1: for a read, the firmware
  releases the line (bit 5 = 0) for the data byte and samples the chip's
  output there. With both chips on one released line, a line nobody
  drives reads 1 (Hypothesis: a pull-up or the Si3210's released SDO).
- The firmware drives it as an **8-bit controller** in the datasheet's
  sense: CS# rises between the control, address and data bytes of each
  operation. Control byte 0x20 for a write and 0x60 for a read, channel
  0, no broadcast.
- **Set-up sequence**: RESET# low from 0.19 s for about 0.197 s, high,
  then a short second low pulse; a software reset (register 1, SR); the
  configuration writes (line interface, ring validation, gains, PCM start
  counts); then register 6 = 0x00, which clears PDL and powers up the
  line side, at 0.39 s. Register 33 ends at 0x39 (PCM enabled). In a
  14.8 s run the firmware wrote 21 times in all, to registers 1, 5, 6,
  16–18, 22–24, 26, 30, 31 and 33–39.
- It **never reads the revision registers** (11 and 13); it reads only
  registers 5, 28 (LCS2) and 29 (LVS). So nothing in the firmware tells
  us the board's chip revisions.
- **Line check**: at 1.39 s it takes the line off hook (register 5),
  reads LCS2 about 150 ms later, then goes back on hook with ONHM set
  (register 5 = 0x08). From about 1.55 s, after a burst of about 110
  reads, it reads LCS2, LVS and register 5 once per 10 ms tick; 4,324
  reads in a 14.8 s run.
- Not seen: the DAA's interrupt, ring handling past the ring-detect
  bits, caller ID, or PCM audio through it (none of these were modelled).

Times on this bus are set by busy-wait loops between the 10 ms ticks, so
the bit timing is a lower bound (see the status marks at the top).

**Flash**: identified by the AMD autoselect sequence; see
[`flash.md`](flash.md).

**Voice coding** (Simulated). The application does G.711 µ-law and A-law
coding itself, on the MIPS-X core, to and from 16-bit linear samples; no
peripheral is involved. Its encoder is bit-exact with ITU-T G.711 for all
65,536 inputs of both laws, keeping the top 14 (µ-law) or 13 (A-law) bits
of each sample. Its decoder returns the G.711 output values scaled to
16 bits. The Si3210 is programmed for 16-bit linear PCM (above, Simulated),
which fits, and new firmware needs its own G.711 coder.

**Audio path and the 0x8200 DSP** (Hypothesis, strong; register traffic
Simulated). The 0x20008200 block is the port of a **DSP coprocessor**
inside the ES3890F, not a plain command engine, and it, not the MIPS-X
core, writes the samples into the audio buffers that the DMA columns
carry to and from the codecs:

| Offset | Role |
|---|---|
| 0x8214 | Command word. High byte classes seen: 0x40, 0x41, 0x48, 0x50, 0x51 and 0x58 are **block moves** between RAM and the DSP's own memory, with an offset in that memory in the low byte; 0x18 and 0x19 look like "run" commands; 0x08 is the **program load** class (class 0x08 loads program memory, while 0x40/0x50 moves address data memory); the pair 0x0100 then 0x0800 opens a program load and 0x0106 then 0x0200 closes it (see "Program loading" below); 0x01, 0x02 and 0x10 are also used, roles unknown |
| 0x8200 | Block-move length, written as a negated count (for example −40 or −160); for a program load, 0xFE00 \| (512 − words) |
| 0x8208, 0x820C | RAM address of a block move, split as for the DMA columns' address low and high (§2) |
| 0x8204, 0x8218 | Written often; roles unknown |
| 0x8224 | Ready: bit 0, polled after commands |
| 0x822C | FIFO level, read `& 0x3F` |

- **Cadence**: the block moves run from the 10 ms tick. In a 14.8 s run
  there were about 53,500 block moves, about 36 per tick. Among them are
  moves of 160 words into the d2a (playback) buffers.
- **The audio buffers**: the d2a column +0x30 alternates between two buffers 0x500
  bytes apart with a count of 672; column +0x28 (a2d, capture) cycles
  four buffers of 80 words, one word per 8 kHz frame for 10 ms
  (Hypothesis: the count unit is a 32-bit word; 672 does not divide
  evenly into 80 frames, so d2a's layout is not known).
- **Program loading** (Observed in code; the loads Simulated). The DSP
  runs code supplied by the firmware, not from on-chip ROM. Its program
  memory is 512 words of 32 bits (Hypothesis, strong: the load length
  field allows at most 512 words and no program is longer). The
  application image holds **27 programs** at the start of its data-table
  stream (wherever the image loader places it; program sizes run from 58 to
  512 words, 40,452 bytes in all, in two contiguous banks: ids 1 to 0x11
  at stream offsets 0x0000 to 0x5918 in id order, then ids 0x13, 0x12 and
  0x14 to 0x1B at offsets 0xB9CC to 0xFEB8). One program is resident at a
  time; the firmware swaps programs by id and keeps the resident id in
  RAM. A load is: wait for the FIFO level (0x822C & 0x3F) to be at most
  0x3C and for 0x8224 bit 0; write 3 to the run control (0x20008000);
  command words 0x0100 then 0x0800 to 0x8214; 0x8200 = 0xFE00 \|
  (512 − words); 0x8208 = bits 10 to 2 of the program's RAM address as a
  word offset in bits 8 to 0, with bit 14 set; 0x820C = the RAM address
  >> 11 (the 0x12000000 alias, so bits 31 to 11 of 0x12xxxxxx); wait
  again; write 1 to the run control; command words 0x0106 then 0x0200.
  The source is the loaded image data itself (no copy, no relocation).
  So the run control is 0 and then 1 at init, 3 while a program is being
  loaded, and 1 to run.
- **Which programs run** (loads Simulated; selection Hypothesis). The
  first load is program 0x12 (512 words) at about 0.39 s, after the DSP
  init of §7.4 (run control 0 then 1; commands 0x106, 0x200, 0x1005,
  0x1052; ready wait). The periodic block moves and run commands begin
  only after it. From about 5 s, every 10 ms tick loads 0x12 and runs it,
  then loads 0x1B (96 words) and runs that, so two programs time-share
  the DSP. Ids 1 to 0x11 appear to be selected by the codec set-up code
  (the codec name table follows the program table in the image), and ids
  0x12 to 0x1B by the tone, caller-ID and FXO code (Hypothesis).
- **Instruction format** (Hypothesis, from statistics over all 10,113
  program words). Fixed 32-bit words; a word with bit 14 set carries one
  following operand word (2,542 of 7,571 instructions); the top nibble
  of an instruction is 4 in about two thirds of cases and 5 in a tenth;
  the commonest first bytes are 0x40, 0x48, 0x50, 0x4F, 0x41 and 0x44.
- **Public description** (Hypothesis; from public patents, read
  2026-11-01, see [`../common/processor-es3890.md`](../common/processor-es3890.md)
  §1 for the lineage, and from a dirty-side statistical test of the 1992
  format against the 27 programs, counts only). The ES3890's cores
  descend from IIT/8x8's Vision Processor, and the 8x8 patents US
  6,121,998 (filed 1998) and US 5,379,351 (filed 1992) describe its DSP
  from the host side: a RISC core drives it by writing command words to
  a command queue with a fill-level status, an "init" housekeeping
  command that is a soft reset, and I/O commands that stream blocks
  between system memory and the DSP's data memory; the program lives in
  a local SRAM beside an on-chip ROM of common routines, a program too
  big for the SRAM is kept in DRAM and paged in while it runs, and the
  DSP runs the G.711, G.723 and G.729 coders, echo cancellation and DTMF
  detection. That is the shape of the loading and block-move protocol
  above, and the 1992 command word fits the 0x8214 word once its type is
  read as the three bits 14 to 12 and its address as the twelve bits 11
  to 0: bit 15 is a wait bit (hold the command until the datapath is
  idle; clear on every run command seen); type 001 is a datapath
  subroutine call, and every 0x18xx/0x19xx run command's entry 0x800|A
  falls inside the resident 512-word program if that program sits at
  address 0x800, while the init commands 0x1005 and 0x1052 call entries
  below 0x800, consistent with the ROM of common routines; types 100
  and 101 are I/O write and read with a 12-bit DSP data address, so
  0x40xx and 0x48xx (and 0x50xx and 0x58xx) address one data memory with
  bit 11 set or clear, not two memories; type 000 is housekeeping, which
  covers the loader's 0x0800, 0x0100, 0x0106 and 0x0200, sub-commands
  unknown. The 1992 part's 64-bit microcode word is ruled out for the
  ES3890: 11 of the 27 programs have an odd word count and 4 of the 10
  run entries are odd, so the program is a stream of 32-bit words. The
  top three bits of each program word behave like the 1992 sequencer
  field (010 "next" on about 81% of instruction words; 000 ends 18 of
  the 27 programs and most host-run routines, 001 ends 8), but the jump
  codes do not pair with target loads as the 1992 design needs, so only
  "000 = done, 001 = return" is supported. The status registers differ
  from the 1992 part: 0x822C & 0x3F is a 6-bit fill count (the loader
  waits for at most 0x3C), not a 2-bit encoding, and 0x8224 bit 0 is a
  ready bit, so the queue is deeper. None of this has been tested on a
  board.
- **Data moves**. Commands 0x40xx (RAM to DSP data memory at offset xx)
  and 0x50xx (DSP to RAM), and also 0x48xx and 0x58xx, carry sample
  buffers (80 and 160 words), 40-word blocks and 1 to 3 word parameters.
  In 14.8 s the only block taken from the image is 20 words of
  constants, moved to data offset 0 once a tick. Program and data
  memories are separate.
- **Consequence**: with the DSP's registers answering "ready" and "FIFO
  empty" and no data moving, the d2a buffers stay all zero after the
  phone goes off hook, and no tone appears anywhere in RAM. So tones
  (dial tone at least) are presumably generated in the DSP. The audio path cannot be
  simulated without a model of the DSP, which needs its instruction
  set; the programs are in the image (above), but the instruction set is
  not known. A capture of the PCM
  highway on the board (P6) is the way to learn the frame layout.
  New firmware has two choices: drive the DSP as the stock firmware does
  (the loading protocol above is understood; what the programs do is not), or bypass it and fill the DMA
  buffers from the MIPS-X core (whether the columns work without the DSP
  is not known).

## 9. Side effects to know before running stock firmware

- If both boot-record sectors are erased, the stock application rewrites
  sector 0 during start-up (Simulated). Do not run it against a flash
  whose boot sectors are blank unless that is wanted. The record it
  writes then has **no section slots** (Simulated), which matters at the
  next restart (below).
- **New configuration is applied by restarting** (Simulated). Saving
  settings from the web UI (line 1's proxy, user and password were
  tested) stores them in flash at once, but the firmware applies them
  only by restarting the unit, and only once it is **idle**: no line in
  use, no ring. With the telephony ports idle, the restart came about
  3 s after the save. While the LINE port appeared to ring (every DAA
  read returning 0xFF), it never came, and a reboot requested from the
  web UI was likewise deferred. The restart goes through the boot code
  and the boot selector from the start, like a reset
  ([`../common/firmware-image-format.md`](../common/firmware-image-format.md)
  §5), and the restarted firmware repeats the whole start-up in §7,
  including the Si3210 and Si3050 resets.
- **A record without slots makes that restart start the recovery
  program** (Simulated). With the slot-less record above, the selector
  finds no slot, scans, and the type 1 section (section B, the recovery
  program) wins the scan. After the restart the unit did DHCP and ARP
  but refused port 80 and sent no SIP. With a record holding slots for B
  at 0x20000 and A (as an installed unit has; built from the format spec
  §5), the restart came back to the application. So stock firmware run
  from a flash with blank boot sectors works until its first
  configuration change and then drops into recovery.
- **The LINE port goes off hook at each start** (Simulated): for about
  150 ms at 1.39 s, as the Si3050 line check (§8). On a real line that is
  a brief seizure.
- The firmware's fatal-error path masks interrupts, writes 0xFE to
  system-control reg 3 and spins: a fatal error looks like a hang
  followed by a watchdog reset (Observed).

## 10. Open questions

| # | Question | How to answer |
|---|---|---|
| P1 | The watchdog's timeout, and whether it runs before system-control reg 3 bit 7 is set. Narrowed 2026-11-02 (§5, Measured): kicking without setting the bit runs indefinitely, so the watchdog is off from reset or the kick works | Our own code on the board: kick, then stop kicking for a measured time, with and without setting the bit |
| P2 | What 0x20004028 does | Board experiments once our code runs |
| P3 | Real timing of the DMA columns, the 0x8200 engine and the counters at 0x2000E014 and 0x2000F0A8 | Board, logic analyser |
| P4 | Which system-control bits are which PLL strap, and what the other board modes change | Board, or a reading taken in reset |
| P5 | The Si3050's wiring and set-up as given in §8 (shared data and clock, SDI and SDO tied, 8-bit framing), all from the firmware's traffic | Board, logic analyser on the SPI lines and CS# pins during start-up |
| P6 | The PCM highway: frame layout, slots and which DMA buffer words carry which port's samples; whether the DMA columns work without the 0x8200 DSP (how to load and run it is answered in §8) | Board, logic analyser or scope on the Si3210's DRX and DTX with the phone off hook (dial tone) |
| P7 | Whether the ES3890F's own reset (watchdog or JP2) also resets the Si3210 and Si3050, or only the firmware's RESET# pulses do | Board: scope on Si3210 pin 7 and Si3050 pin 13 through a watchdog reset |
| P8 | The DSP's instruction set. A board capture cannot answer it; only decoding the program words in the image or a vendor document can | Decode the 27 programs (§8) against a vendor document, if one is found |
