# ES3890 processor: MIPS-X as used in the SPA3000

Status: draft, 2026-10-27. Dirty-side findings written as facts for the
clean side. Contains no vendor code. Same status marks as
[`firmware-image-format.md`](firmware-image-format.md): **Verified**,
**Observed**, **Hypothesis**.

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
- Two delay slots after every branch and jump. **Verified**.
- The ES3890 reuses the manual's coprocessor and floating-point memory
  opcodes as byte and halfword loads and stores. **Observed**, strongly
  supported (§3).
- The application calls functions through base registers r23 and r24,
  set by the image loader from the load directory. **Verified** (§6).

## 2. Instruction formats

| Field | Bits | Notes |
|---|---|---|
| TY | 31–30 | 00 branch, 01 compute, 10 memory, 11 compute-immediate |
| OP / COND | 29–27 | |
| Src1 | 26–22 | |
| Src2 | 21–17 | Holds Dest for loads, `addi` and `jspci` (manual Appendix II "starred" instructions) |
| Dest | 16–12 | Compute instructions and `movfrs` |
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
| 101 | `movfrc` | Load byte, sign-extended | Any offset; Src1 is a normal pointer register, not r0; almost never followed by sign extension |
| 001 | `ldt` | Load halfword, zero-extended | Offsets always even |
| 111 | `movtoc` | Load halfword, sign-extended | Offsets always even; the register behaves as a destination |
| 011 | `stt` | Store byte (low 8 bits) | Any offset |
| 110 | `stf` | Store halfword (low 16 bits) | Offsets always even |

Load or store: **Observed** from dataflow (whether the register is
defined just before or used just after). Sizes: **Observed** from offset
alignment. Signedness: **Hypothesis**. The manual's byte insert and
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
  the branch is taken.
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
says.

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
| 000 | `jspci Src1,#imm,Dest` | Jump to word address Src1 + SignExtend(imm17), after two delay slots that always execute. Dest (Src2 field) = jspci word address + 3, which is the return address |
| 011 | `movfrs Spec,Dest` | Dest (bits 16–12) = special register; Spec in Func bits 2–0 |
| 010 | `movtos Src1,Spec` | Special register = Src1 |
| 101 | `jpc` | Func = 0x003. Jump to PC-4 (exception return chain) |
| 111 | `jpcrs` | Func = 0x003. As `jpc`, and restores the PSW |
| 001 | `hsc` | Word 0xCFC00000 (Src1 = r31, rest 0). Halt, from the manual and MAME; not seen in the SPA3000 application |
| 110 | `trap vector` | Func bits 2–0 = 011, bit 11 = 0; bits 10–3 hold the one's complement of the vector. Jumps to vector × 8 (word address) |

Special registers: 1 = PSW, 2 = MD, 4 = PC-4 when read and PC-1 when
written.

How the compiler uses `jspci`, all **Verified** in the application:

- **Return:** `jspci r31,#0,r0`.
- **Call:** Dest = r31. Src1 is almost always r23 or r24, a base register
  that holds a constant code word address. With the base values from §7,
  10,366 of 10,377 calls through r23, and all 3,435 calls through r24,
  land on the instruction just after a return, which is to say on a
  function start.
- **Jump:** Dest = r0 with Src1 other than r31. Used for far jumps
  through r23 and r24 and for computed jumps.

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
| 0x03 | 0x400070A6 (A), 0x4000CB6A (B) | Entry point. Bits 29–0 are a word address (A: word 0x70A6, byte 0x1C298). Bit 30 is a flag of unknown meaning. | **Observed**: word 0x70A6 follows a return and sets r29 from memory before jumping into the application |

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
0x14000000 (perhaps a UART or status port) and reads from 0x1CFEFFFC.
How the stub uses the 0x1CFEFFFC value is consistent with it being
section C's last word in flash (Hypothesis); see the format spec §5 (Section C,
Placement). The stub's calls into the boot section's decompressor, and
the registers it sets for them, are in the format spec §4.6.
