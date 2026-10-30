# ES3890 processor: MIPS-X as used in the SPA3000

Status: draft, 2026-10-27. Dirty-side findings written as facts for the
clean side. Contains no vendor code. Same status marks as
[`firmware-image-format.md`](firmware-image-format.md): **Verified**,
**Observed**, **Simulated**, **Hypothesis**. Simulated facts come from
running vendor code on the openmipsx simulator, with outputs checked
against an independent oracle (see the clean-room log, 2026-10-29).

The base reference is Paul Chow, *MIPS-X Instruction Set and Programmer's
Manual*, Stanford CSL-86-289
([PDF](https://www.eecg.utoronto.ca/~pc/research/publications/mipsx.isetman.pdf)).
Section numbers below ("manual §x") refer to it. This document records
where the ES3890 matches the manual and where it differs.

Bits are numbered here from the least significant bit (bit 0) up, which is
the opposite of the manual's convention.

## 1. Summary

- Instructions are 32-bit, big-endian. **Verified**: the main application
  code decodes completely with the encodings below.
- Data addresses are **byte** addresses. Code addresses (the PC, branch
  displacements, jump targets and link values) are **word** addresses:
  byte address = word address × 4. **Verified** (§4, §5).
- Code addresses are 30 bits wide: bits 31–30 of a jump target are
  ignored, so word addresses wrap modulo 2³⁰. **Simulated** (§6).
- Two delay slots after every branch and jump. **Verified**.
- The ES3890 reuses the manual's coprocessor and floating-point memory
  opcodes as byte and halfword loads and stores. **Observed**, strongly
  supported; the extension done by three of the four loads is
  **Simulated** (§3).
- The application calls functions through base registers r23 and r24,
  set by the image loader from the load directory. **Verified** (§6).

## 2. Instruction formats

| Field | Bits | Notes |
|---|---|---|
| TY | 31–30 | 00 branch, 01 compute, 10 memory, 11 compute-immediate |
| OP / COND | 29–27 | |
| Src1 | 26–22 | |
| Src2 | 21–17 | Holds Dest for loads, `addi`, `jspci` and `movfrs` (manual Appendix II "starred" instructions) |
| Dest | 16–12 | Compute instructions |
| Comp Func | 11–0 | Compute instructions |
| Offset / Immed | 16–0 | 17-bit signed: memory and compute-immediate instructions |
| SQ | 16 | Branches: 1 = squash the delay slots if the branch is not taken |
| Disp | 15–0 | Branches: 16-bit signed, in words |

r0 always reads as zero.

## 3. Memory instructions (TY = 10)

Address = Reg(Src1) + SignExtend(Offset), in bytes. The register in the
Src2 field is the destination for loads and the data source for stores.

| OP | Manual name | ES3890 behaviour | Evidence |
|---|---|---|---|
| 000 | `ld` | Load word | Offsets always a multiple of 4 |
| 010 | `st` | Store word | Offsets always a multiple of 4 |
| 100 | `ldf` | Load byte, zero-extended | Any offset; result used as a register; sometimes followed by explicit sign extension |
| 101 | `movfrc` | Load byte, sign-extended | Any offset; Src1 is a normal pointer register, not r0; in 20 images, almost never followed by sign extension and never by zero extension |
| 001 | `ldt` | Load halfword, zero-extended | Offsets always even. Simulated: a loaded value ORed into the low half of a word without masking gives correct results only if zero-extended |
| 111 | `movtoc` | Load halfword, sign-extended | Offsets always even; the register behaves as a destination. Simulated: signed comparisons of loaded samples give correct results only if sign-extended |
| 011 | `stt` | Store byte (low 8 bits) | Any offset |
| 110 | `stf` | Store halfword (low 16 bits) | Offsets always even |

Load or store: **Observed** from dataflow (whether the register is
defined just before or used just after). Sizes: **Observed** from offset
alignment.

Signedness:

- Opcode 100 zero-extends (**Simulated**: with it sign-extending, the
  boot section's decompressor gives wrong output for every stream and
  every image fails its MD5 check; the application's G.711 decoder (next
  item) also fails for every code from 0x80 up).
- Opcode 111 sign-extends a halfword and opcode 001 zero-extends one
  (**Simulated**). Routines of the SPA3000 application were run on test
  input whose results can be checked without reading the code. A routine
  that finds the largest of a set of signed 16-bit samples is right for
  all 64 test sets under this reading. With the two opcodes' extensions
  swapped, it is wrong for every one of the 56 sets that hold a negative
  sample. The application's G.711 decoder, checked against ITU-T G.711
  for all 256 codes of both laws, is right under this reading and wrong
  for about a quarter of the codes when they are swapped. The
  application's G.711 encoder is also right on all 65,536 inputs, but it
  extends its inputs again itself, so it does not tell the readings
  apart.
- Opcode 101 sign-extends a byte (**Observed**, high confidence). In 20
  vendor images from four code families, a value loaded with opcode 100
  is followed by explicit sign extension 248 to 414 times per image, and
  one loaded with opcode 101 once. Tests for "less than zero" on
  the loaded value are four times as common after 101 as after 100. No
  simulated vendor code has yet loaded a byte with bit 7 set through it.

The same scan supports the halfword results: after opcode 111 there is
explicit sign extension once or twice per image and zero extension never, and "less than zero" tests are about
20 times as common as after 001 or 100.

The manual's byte insert and
extract sequences (`rotlb`, `rotlcb`) never appear in the application,
which fits the chip having real byte access.

Suggested mnemonics, since the manual's names are misleading here:
`ldbu`, `ldb`, `ldhu`, `ldh`, `stb`, `sth`.

## 4. Branches (TY = 00)

Condition from Reg(Src1) − Reg(Src2):

| COND | Mnemonic | Taken when |
|---|---|---|
| 001 | `beq` | equal |
| 101 | `bne` | not equal |
| 010 | `bhs` | unsigned ≥ |
| 110 | `blo` | unsigned < |
| 011 | `blt` | signed < |
| 111 | `bge` | signed ≥ |
| 000, 100 | — | Unused; not seen in application code |

Target (word address) = branch address + 2 + Disp. **Verified**: a
displacement of −1 or 0 (which would land in the branch's own delay slots)
never occurs, −2 (branch to itself) occurs only in a few wait loops, and
loops in the loader stub close correctly with this rule. It matches the
manual's "PCcurrent", the instruction being fetched during the branch's
ALU cycle.

- SQ = 0: both delay-slot instructions always execute.
- SQ = 1 (`beqsq` and so on): the delay-slot instructions execute only if
  the branch is taken. **Simulated**: with the opposite reading, vendor
  code never finishes.
- `beq r0,r0,label` is an unconditional branch.

## 5. Compute instructions (TY = 01)

| OP | Func | Instruction | Operation |
|---|---|---|---|
| 100 | 0x019 | `add` | Dest = Src1 + Src2. `nop` is `add r0,r0,r0` (word 0x60000019); `mov` is `add` with Src2 = r0. MAME's disassembler says ROMs for its ESS chips use Src1 = r0 instead; that form is also accepted as `mov`, though it has not been checked against the SPA3000 application |
| 100 | 0x066 | `sub` | Dest = Src1 − Src2 |
| 100 | 0x026 | `subnc` | Dest = Src1 + ~Src2 |
| 100 | 0x023 | `and` | Dest = Src1 & Src2 |
| 100 | 0x00B | `bic` | Dest = Src2 & ~Src1 |
| 100 | 0x00F | `not` | Dest = ~Src1 |
| 100 | 0x03B | `or` | Dest = Src1 \| Src2 |
| 100 | 0x01B | `xor` | Dest = Src1 ^ Src2 |
| 000 | 0x0E6 | `mstart` | If MSB(MD): Dest = 0 − Src2, else Dest = 0. Then MD <<= 1 |
| 000 | 0x099 | `mstep` | If MSB(MD): Dest = (Src1 << 1) + Src2, else Dest = Src1 << 1. Then MD <<= 1 |
| 000 | 0x166 | `dstep` | a = (Src1 << 1) \| MSB(MD); o = a − Src2. If MSB(o): Dest = a, MD <<= 1; else Dest = o, MD = (MD << 1) \| 1 |
| 001 | 0x0C0 | `rotlb` | Dest = Src1 rotated left by (Src2 & 3) bytes |
| 001 | 0x080 | `rotlcb` | Dest = Src1 rotated left by (~Src2 & 3) bytes |
| 001 | 0x200 \| amount | `sh` | Funnel shift: Dest = low 32 bits of ((Src2 : Src1) >> n) |
| 001 | 0x100 \| amount | `asr` | Dest = Src1 >> n, arithmetic. Src2 = r0 |

The logic functions follow from the manual's description of the Func bits
(Appendix II); `bic` is ~Src1 & Src2, whatever the manual's comment column
says. **Simulated**: the vendor MD5 code relies on `bic`, and only this
reading gives digests that match a reference MD5.

**Shift amounts.** Func bits 6–4 hold `bbb` and bits 3–0 hold a one-hot
`dddd` (0001 = 0, 0010 = 1, 0100 = 2, 1000 = 3). The 5-bit value e =
`bbb` × 4 + that number. The shift right is n = 32 − e. **Verified**
against the code: `sh` with Src1 = r0 is then a left shift of Src2 by e,
and the most common such case is e = 2 (multiply by 4), as you'd expect
for array indexing. Assemblers should show n, as in the manual's examples
(`sh r3,r0,r3,#24` extracts the top byte).

## 6. Compute-immediate instructions (TY = 11)

| OP | Instruction | Operation |
|---|---|---|
| 100 | `addi Src1,#imm,Dest` | Dest (Src2 field) = Src1 + SignExtend(imm17) |
| 000 | `jspci Src1,#imm,Dest` | Jump to word address (Src1 + SignExtend(imm17)) modulo 2³⁰, after two delay slots that always execute. Dest (Src2 field) = jspci word address + 3, which is the return address |
| 011 | `movfrs Spec,Dest` | Dest (Src2 field, bits 21–17) = special register; Spec in Func bits 2–0; bits 16–12 are 0. See "`movfrs` destination" below |
| 010 | `movtos Src1,Spec` | Special register = Src1 |
| 101 | `jpc` | Func = 0x003. Jump to PC-4 (exception return chain) |
| 111 | `jpcrs` | Func = 0x003. As `jpc`, and restores the PSW |
| 001 | `hsc` | Word 0xCFC00000 (Src1 = r31, rest 0). Halt, from the manual and MAME; not seen in the SPA3000 application. Other OP 001 encodings are **not** undefined: firmware for the SPA2100, SPA2102, SPA9000 and SPA942 executes them during normal start-up (**Simulated**; thousands of them in each image, against a few hundred in the older images, consistent with chance matches in data). Their function is unknown |
| 110 | `trap vector` | Func bits 2–0 = 011, bit 11 = 0; bits 10–3 hold the one's complement of the vector. Jumps to vector × 8 (word address) |

Special registers: 1 = PSW, 2 = MD, 4 = PC-4 when read and PC-1 when
written.

**`movfrs` destination (Observed; corrected 2026-10-30).** `movfrs` keeps
its destination register in the Src2 field, bits 21–17, like the other
compute-immediate instructions. Bits 16–12 are 0. Earlier versions of this
document put it in bits 16–12, which was wrong. Evidence:

- Every `movfrs` word in the SPA3000 application (58 of them) has
  bits 16–12 = 0 and a register number in bits 21–17. For example
  0xD8040001 is `movfrs psw` into r2.
- The code only makes sense this way: the interrupt-mask sequence reads
  the PSW, ORs in bit 4 and writes it back with `movtos` through the
  register in bits 21–17 (§10).
- **Simulated**: run on a simulator with the old decoding, the
  application corrupts its PSW within its first 60,000 instructions; with
  this decoding it runs normally for 15 simulated seconds. About 66,000
  `movfrs` were executed in its first 20 million cycles.

Assemblers, disassemblers, compilers and simulators written from the old
text need the same change. `movtos` is unaffected (Src1 is the source).

How the compiler uses `jspci`, all **Verified** in the application:

- **Return:** `jspci r31,#0,r0`.
- **Call:** Dest = r31. Src1 is almost always r23 or r24, a base register
  that holds a constant code word address. With the base values from §7,
  10,366 of 10,377 calls through r23, and all 3,435 calls through r24,
  land on the instruction just after a return, which is to say on a
  function start.
- **Jump:** Dest = r0 with Src1 other than r31. Used for far jumps
  through r23 and r24 and for computed jumps.

**Target wrap (Simulated).** Jump targets are taken modulo 2³⁰ words:
bits 31–30 of Src1 + SignExtend(imm17) are ignored. Vendor jump tables
hold 30-bit offsets added to a code base register, so a negative entry
gives a sum with bit 30 set, which must land on bits 29–0. Seen in
section B of five images and in the boot section's decompression
service, which only produces correct output with the wrap.

## 7. Calling convention (Observed; Hypothesis where marked)

- Stack pointer: r29. Functions adjust it with `addi r29,#-n,r29`. The
  image's loader stub sets it before entry.
- Return address: r31.
- Arguments: r4, r5, r6 and so on, in that order (Hypothesis). These are
  the registers most often read before being written at function entry.
- Return value: r2 (Hypothesis). r17 and r16 are also often written just
  before a return, which suggests callee-saved registers restored in the
  epilogue.
- Probably callee-saved: r16 to r22 (Hypothesis).
- Reserved: r23 and r24, the code base registers, and probably r25 (see
  the load directory notes in the format spec).

## 8. Corrections to the format spec's load directory (§4.4)

The loader stub at the end of each packed section reads the load
directory. Its behaviour, and the code, show the following:

| Type | Word 1 | Meaning | Status |
|---|---|---|---|
| 0x07 | 0x00016D34 (A), 0x00014000 (B) | Value for r23: the code base word address, equal to the first code stream's word address + 0x10000. **Not the entry point.** | **Verified** for A |
| 0x05 | 0x0004BC00 (A) | Value for r24: the second code stream's word address (0xEF000 / 4) + 0x10000 | **Verified** |
| 0x06 | 0x00010000 (A) | Probably the value for r25 | Hypothesis |
| 0x03 | 0x400070A6 (A), 0x4000CB6A (B) | Entry point. Bits 29–0 are a word address (A: word 0x70A6, byte 0x1C298). Bit 30 has no effect on where execution starts: the jump drops it (§6, Simulated). Whether anything else reads it is unknown. | **Observed**: word 0x70A6 follows a return and sets r29 from memory before jumping into the application |

Section A's streams, in file order, map to load addresses as follows. The
stream files are numbered by their order in the section.

| Stream | Load address (bytes) | Size | Content |
|---|---|---|---|
| 0 | 0x0001B4D0 | 0x7C798 | Code (r23 region) |
| 1 | 0x000EF000 | 0x6DC80 | Code (r24 region) |
| 2 | 0x00097C68 | 0xF0F4 | Not clean code: mixed or data |
| 3 | 0x0015CC80 | 0x144AC | Data |
| 4 | 0x00001000 | 0x8000 | Not clean code |
| 5 | 0x00010000 | 0x8F38 | Not clean code |

Section C is MIPS-X (Observed). The earlier finding
that it is not came from its zero padding: exactly two-thirds of its words
are zero, and 0x00000000 decodes as a branch with condition 000. Its
non-zero regions have the same mix of instruction types as section A's
code, including the no-op 0x60000019. It contains a deflate decompressor
and an embedded program in the packed container format; see
[`firmware-image-format.md`](firmware-image-format.md) §5. Lesson for
statistics like these: exclude zero words first.

**Memory-mapped I/O (Observed):** the loader stub writes bytes to
0x14000000 (perhaps a UART or status port) and reads from 0x1CFFFFFC.
An earlier version of this document gave 0x1CFEFFFC, from treating the
17-bit offset 0xFFFC as negative. 0x1CFFFFFC is section C's last word in
flash, 0x0CFFFFFC, with address bit 28 set: **Simulated**, the stub
starts the embedded program correctly when that word is served there.
That bit 28 is an alias is a Hypothesis, strongly supported. See the
format spec §5 (Section C, Placement).

The first peripheral accesses in application and recovery start-up, in
every image tried, are stores to 0x20000000–0x2000FFFF, within the
first 50 instructions (**Simulated**, location only). The stub's calls into the boot section's decompressor, and
the registers it sets for them, are in the format spec §4.6.

## 9. Pipeline rules seen in vendor code

The vendor's compiler produced code that runs correctly on the device,
so what it does is evidence of what the hardware allows. These facts come
from running about 120 million instructions of vendor code on the
simulator: the boot section's decompression service (output checked
against zlib for 8 streams), its boot selector (MD5 checked against a
reference), the start-up of nine images, and the SPA3000 application's
G.711 encoder and decoder (checked against ITU-T G.711). All
**Simulated**, with counts only. The G.711 run also confirms §4's squash
rule: with the opposite reading the encoder never finishes.

| Rule (manual) | What the vendor code does | Conclusion |
|---|---|---|
| Load delay | Never reads a loaded register in the instruction straight after the load, including across taken branches, calls and returns. Reads it in the second instruction after the load millions of times, at well over 100 places. | A load has **one** delay slot. Whether the hardware interlocks on the first is not known, since the compiler never tests it |
| Load into its own address register (`ld 0[rN],rN`; manual §3.3 forbids it) | Done about 1.5 million times at about 20 places, in the decompression service and section A, with correct results | Works and gives the loaded value, at least outside exceptions. The manual's rule is presumably about restarting a faulted load |
| Memory instruction straight after a store | Common (about 2 million times) | Legal. Whether it stalls a cycle is not tested |
| Load in the second delay slot of a branch or jump | About 70,000 times, at dozens of places | Legal on the hardware |

Not exercised by any vendor path that could be run: `mstart`, `mstep`,
`dstep`, moves to and from MD, `trap`, and shifts by 32. Facts about
those still rest on the manual alone.

## 10. Interrupts (Hypothesis, strongly supported)

From reading the SPA3000 application's interrupt handler and its
critical sections, and from running the application on a simulator with
a model of the peripherals it needs (**Simulated**; the interrupt entry
in that model was written to match the handler, so the simulation shows
consistency, not proof). The interrupt controller and timer registers are
in [`../spa3000/peripherals.md`](../spa3000/peripherals.md).

- **Vector.** Interrupts enter at **word address 0**. The application
  installs four instructions there at start-up: a no-op, a jump to its
  handler, and two no-ops (the jump's delay slots).
- **Mask.** PSW bit 4 (value 0x10) masks interrupts. That is where the
  manual puts its M bit, so this confirms the manual's layout for that
  bit. The reset code sets PSW = 0xF3 before anything else. Setting it,
  with `movfrs psw`, OR 0x10, `movtos psw`, is the application's
  critical-section primitive. Start-up runs with it set and clears it once
  the timer is programmed.
- **PC chain.** On entry the PC chain holds three PCs: the instructions
  that were in the pipeline and did not complete. The handler saves them
  with three `movfrs pcm4` (special register 4, read), restores them with
  three `movtos pcm1` (special register 4, written), and returns with
  `jpcrs`, `jpc`, `jpc`, which restarts the three in order. It also saves
  MD.
- **A store at the head of the chain has already been done.** Before
  returning, the handler checks whether the first saved PC holds a word
  store, byte store or halfword store. If it does, it restarts a no-op in
  its place (the one at word 0). So on the hardware a store at the head
  of the chain has passed its memory stage when the interrupt is taken.
  A handler that restarts it anyway does the store twice: harmless for
  RAM, but not for an I/O register with side effects.
- The handler returns with `jpcrs`, so the PSW saved at entry is
  restored; how the hardware saves it on entry is not known.
- Not known: whether exceptions (other than `trap`) share the vector,
  what the hardware does with the PSW on entry, and how many cycles entry
  takes.
