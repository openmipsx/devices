# Clean-room process

This project aims to write new firmware for Sipura and Linksys analogue
telephone adapters, starting with the SPA3000, without copying any of the
vendor's code. The usual way to do that is a
clean-room split: one side studies the original firmware and writes
specifications that hold only facts, and the other side writes the new
firmware from those specifications alone.

This is an engineering process, not legal advice. Clean-room separation is
evidence of independent creation, and that is all; it does not settle
questions of patents, trademarks, licence terms or the law where you live.
Take advice if you plan to distribute anything.

## Roles

**Dirty side (analysis).** May handle the original image and anything
derived from it: unpacked sections, decompressed code, disassembly, strings,
the vendor's upgrade tool, captures of the original firmware running. Its
only output to the clean side is the specification documents in this
repository.

**Clean side (implementation).** Writes the new firmware. Reads only the
specifications in this repository, public standards (RFCs, ITU-T recommendations,
chip datasheets) and the device's public user documentation. Never opens
the dirty-side material listed below, and never asks the dirty side for
code, disassembly or verbatim text.

One person can do both roles only by keeping them separate in time and
material. Two people is much stronger evidence.

## What may go into a specification

Allowed, because they are facts needed for interoperability:

- File and structure layouts: offsets, field sizes, byte order, magic
  values, which checksum algorithm covers which bytes.
- Load addresses, sizes and the memory map.
- Protocols and on-the-wire behaviour, described in your own words.
- Names of standard algorithms (MD5, CRC-32, raw deflate) and where they
  are used.
- Hardware facts: chip part numbers, pin assignments, peripheral registers.
- Which parts are still unknown, and confidence levels for hypotheses.

Not allowed:

- Disassembly, decompiled code, pseudocode that follows the original's
  structure, or code snippets of any kind from the original.
- Verbatim text from the image (web pages, messages, format strings)
  beyond short identifiers that must match to interoperate, such as a
  magic number.
- Data tables, audio prompts, keys or certificates taken from the image.
- Text from the vendor's release notes (they carry a confidentiality and
  copyright notice).

When in doubt, describe behaviour ("the image header is rejected if the
MD5 does not match") rather than implementation ("the loader calls a
routine at 0x1234 that...").

## Dirty-side material

None of it is in this repository. The dirty side keeps it apart, and the
clean side never opens it:

| Material | What it is |
|---|---|
| Vendor downloads | Firmware images, upgraders and release notes, as downloaded (hashes below) |
| Unpacked sections | Sections and decompressed code produced by the unpacker |
| Ghidra projects and decompilations | Disassembly and decompiled C of the vendor code |

The unpacker (`unpack_spa3000.py` in the log below) was written from the
format analysis and contains no vendor code, but it produces dirty-side
output, so treat its *output* as dirty.

## Clean-side material

| Path | What it is |
|---|---|
| [`common/firmware-image-format.md`](common/firmware-image-format.md) | Upgrade image container format, checksums, load map, boot selection and recovery protocol. Written from the SPA3000 3.1.20(GW) image |
| [`common/processor-es3890.md`](common/processor-es3890.md) | Instruction set as used by the ES3890, with corrections to the load directory description. Derived from disassembly (see the 2026-10-27 provenance row), but holds facts only |
| [`spa3000/hardware.md`](spa3000/hardware.md) | SPA3000 board components from chip markings and public datasheets |
| [`spa3000/photos/`](spa3000/photos/) | Photos of the unit and board (no firmware content) |
| [`pap2t/`](pap2t/) | Linksys PAP2T: not started |
| [ghidra-mipsx](https://github.com/openmipsx/ghidra-mipsx) | Ghidra processor extension for MIPS-X, built from the public manual and the specs only; it contains no vendor code. The tool is clean-side, but **the analysis it enables is dirty-side**: anyone who reads disassembly of the firmware in Ghidra is dirty-side for that material |
| [Open-source stack plan](https://github.com/openmipsx/.github/blob/main/plan/open-source-stack.md) | Proposal for building the new firmware from open-source components, with obstacles and order of work. Names public projects and refers to the specs; one hypothesis comes from the decompilation reading in the provenance log |

Public reference documents (the MIPS-X manual, user guides, datasheets)
are not copied here. The provenance log says where each came from, and the
hashes below identify the copies that were used.

## Validating the simulator on vendor code

The [mipsx-sdk](https://github.com/openmipsx/mipsx-sdk) simulator was
written from the specs, and the compiler was tested against the
simulator, so a wrong fact in a spec would leave every test passing. The
vendor's firmware ran on real hardware, which makes it the best evidence
short of the hardware itself. Running it on the simulator is allowed,
but it is dirty-side work, in the same way as reading it in Ghidra.

**The simulator stays clean.** Running vendor code on it does not change
that, as long as nobody edits the simulator from what a vendor run
showed. Anything a run produces is dirty: traces, hazard reports (they
print the PC, the instructions and nearby symbols), `--disassemble`
output, and ELF files built from unpacked sections. Keep all of it on
the dirty side, never in `mipsx-sdk`.

**Results cross as spec facts.** What reaches the clean side is a change
to a spec, with a provenance row and a confidence level, for example
"byte loads with opcode 101 sign-extend: confirmed by running vendor
code" or "vendor code uses a loaded register two instructions after the
load, so loads have one delay slot". Addresses, instruction sequences
and trace excerpts do not cross. Counts are fine ("no P1 hazards in N
million instructions").

**Fixes go through the spec.** If a run shows the simulator is wrong,
the dirty side corrects the spec. The clean side then changes the
simulator and writes its own regression test from the spec text, not
from the instruction sequence that showed the bug. One person, or one
Claude session, should not both read a vendor trace and edit the
simulator; the edit cites the spec change.

**Good checks** need an oracle that does not depend on reading the code:

- Routines whose interface is already in a spec, with outputs that can be
  checked independently: the boot section's decompression service
  ([`firmware-image-format.md`](common/firmware-image-format.md) §4.6)
  against zlib, and image validation against a good image and one with
  a broken checksum.
- Variants (`mipsx-sim --variant`): each replaces a Hypothesis with an
  alternative. Vendor code that works under the spec's reading and fails
  under a variant is evidence for the spec. One result per hypothesis
  crosses.
- Hazard checks: the vendor's compiler made code that works, so a hazard
  reported in vendor code means the simulator or the rule is wrong, and
  vendor code that breaks a rule variant (`load-delay-2`, `md-delay-2`)
  is evidence that the stricter rule does not exist.

Code that only runs with the device's peripherals is out of reach until
the simulator models them. Features that help, such as loading a raw
blob at an address, setting registers before the start, or ignoring a
range of I/O addresses, belong in the simulator only if they are general
and written from the specs, not shaped to what the firmware was seen to
touch. A vendor run shows what the vendor's toolchain relied on, not
everything the chip does: constructs it never emits remain unconfirmed.

Each run gets a row in the provenance log naming the image, the
sections and routines run, the simulator version and options, and the
spec facts that came out of it.

## Provenance log

Record each piece of analysis so it can be shown later where every fact in
the specs came from. Newest entries at the bottom. Paths are as they were
on the date of the entry; on 2026-10-27 this folder and `ghidra-mipsx`
moved from `~/src/home/` and `~/src/` to `~/src/mipsx/`. On 2026-10-28 the
specs and photos were published in this repository: `docs/spec/` became
`common/` and `spa3000/`, `docs/photos/` became `spa3000/photos/` (with
photo metadata removed), and `docs/plan/open-source-stack.md` moved to the
organisation's `.github` repository. Later that day `~/src/mipsx/` was renamed
`~/src/openmipsx/`.

| Date | Who | Source examined | Method | Output |
|---|---|---|---|---|
| 2026-10-27 | Tim, with Claude (AI assistant) as analyst | `spa3000-3-1-20GW.bin` (SHA-256 `e32d18fa…8e70`), from the Cisco Community thread "SPA3000 firmware?" | Byte statistics, header inspection, checksum and decompression tests against hypotheses. No disassembly. | `docs/spec/firmware-image-format.md` |
| 2026-10-27 | Same | Decompressed streams from the image | Printable strings sampled, only to classify what each stream holds (web server, crypto tables and so on). No disassembly. | Stream descriptions in the format spec |
| 2026-10-27 | Same | `upg-spa3000-3-1-20GW.exe` | Printable strings only, to learn the upgrade transport | Upgrade-transport section of the format spec |
| 2026-10-27 | Same | `rn-spa3000-3-1-20(GW).txt` (release notes) | Read in full | Nothing carried over. The analyst has seen it, so the clean side should not. |
| 2026-10-27 | Same | Section C (`sec0_1.1.1.raw`) and decompressed section A and B code; public sources: the 0cpm SPA notes (reverse.0cpm.org/linksys), Ubicom's GPL binutils opcode tables (github.com/jameshilliard/ubicom32) | Word-frequency statistics, instruction-type and delay-slot counts, and matching against Ubicom32 opcodes. No disassembly or decompilation. | Processor section of the format spec (MIPS-X, probably ESS ES3890-family; Ubicom32 ruled out) |
| 2026-10-27 | Tim (photos), Claude (reading) | The unit itself, opened; two photos of the top of the board | Chip markings read from photos; roles from public datasheets | `docs/spec/hardware.md`; CPU confirmed as ES3890F in the format spec |
| 2026-10-27 | Same | The unit: five more photos (underside, close-ups, case labels) | Chip and case markings read from photos; roles from public datasheets | Flash, RAM, glue logic, relay, regulators and port assignment in `docs/spec/hardware.md` |
| 2026-10-27 | Same | The unit: three more close-ups | Markings read from photos | Y2, Q1, REG1, REG2 and D14 in `docs/spec/hardware.md` |
| 2026-10-27 | Claude | Public document: Paul Chow, *MIPS-X Instruction Set and Programmer's Manual*, Stanford CSL-86-289, from eecg.utoronto.ca/~pc/research/publications/mipsx.isetman.pdf | Downloaded only; no firmware material involved | `docs/reference/mipsx.isetman.pdf` (102-page scan; title page dates it May 1988, inner page May 1986) |
| 2026-10-27 | Claude as analyst | Section A code streams and the loader stub at the end of the packed sections; section B stream; section C | Instruction decoding with an ad-hoc decoder written from the public MIPS-X manual; dataflow and alignment statistics; inspection of the loader stub's handling of the load directory. **This is disassembly**, unlike the earlier rows; the decoder's output was read, but no listings are kept in the specs. | `docs/spec/processor-es3890.md`; corrections to the load directory, load maps and processor section of `docs/spec/firmware-image-format.md` |
| 2026-10-27 | Claude as analyst | `spa3000-3.1.20(GW)/decompiled/decompiled.c`: Ghidra decompilation of section A, 3,117 functions | Read the decompiled C to classify subsystems, start-up order and use of peripheral registers. **This is decompilation.** No code, names or text carried over. | One hypothesis in `docs/plan/open-source-stack.md` §3.3 (voice processing is handed to the second core or a dedicated peripheral block) |
| 2026-10-27 | Claude as analyst | Fifteen further vendor downloads in `firmware/sipura/` (SPA1001, SPA2000, SPA2002, SPA2100, SPA2102, SPA3000, SPA841, SPA941, SPA9000 and three recovery upgraders), unpacked in a scratch directory | Header and table inspection, checksum and decompression tests; call-target statistics to match load directory records to registers; Ghidra import and auto-analysis of each application to test the extension (**this is disassembly**, but no listings were read beyond summary counts) | Generalised format and processor notes in `~/src/ghidra-mipsx/docs/`: the flash-block container, record types 0x0A (r22) and 0x13 (r26), and the per-type value of entry field 0x04 (answers Q1 and Q8 of the format spec) |
| 2026-10-27 | Claude as analyst | Section C (`sec0_1.1.1.raw`), with section A's stream 0 and the tails of sections A and B for comparison | Entropy and zero-run profile; instruction-type statistics per region; search for known constants (deflate tables, MD5, CRC-32, SHA-1); comparison of the section C tail with the A and B loader stubs; reading the embedded program's header and directory. About ten instructions near the embedded program's entry were decoded by hand to read a byte comparison (**this is disassembly**); no listings kept. | Section C in §5, the section table and Q6 of `docs/spec/firmware-image-format.md`; correction to `docs/spec/processor-es3890.md` (section C is MIPS-X) |
| 2026-10-27 | Claude as analyst | Section C: whole section loaded at 0, and the embedded program at 0xB988 imported on its own at 0 with r23 = 0x10008 | ghidra-mipsx headless import, auto-analysis, `MipsXFindFunctions`, then decompilation to C (`spa3000-3.1.20(GW)/decompiled-sec0/`, Ghidra project in `spa3000-3.1.20(GW)/ghidra-sec0/`). The embedded program's start-up, image selection and validation functions were read. **This is decompilation.** No code, names or text carried over. | `docs/spec/firmware-image-format.md`: boot selection under Section C in §5, directory record types 0x01 and 0x02, §3 entry fields 0x04, 0x0C, 0x10, 0x20 and 0x7E, §6 notes, Q1, Q6 and Q7 |
| 2026-10-27 | Claude as analyst | Section C's trailing 0x414-byte loader stub (offsets 0xDB6C–0xDF80, identical to the A and B stubs), disassembled in the `ghidra-sec0` project with a throwaway headless script; the section C decompilation's routines at 0x0380–0x28CC; text search of section A's decompilation for references to the 0x0CFFxxxx range and for its own deflate decoder | **This is disassembly and decompilation.** The listing was read and not kept; no code, names or text carried over. | `docs/spec/firmware-image-format.md`: §4.5 rewritten, new §4.6 (the boot section's decompression service and its entry interface), section C layout row for 0x0380, Placement subsection (top of the flash window, superseding the sector 0 guess), §6 step 1, §8 note, Q6; note in `docs/spec/processor-es3890.md` §8 |
| 2026-10-27 | Claude as analyst | Section B (`sec1_1.0.6(3k)`), via the image's type 1 section | ghidra-mipsx headless import (`SipuraLoader`, section type 1), auto-analysis, `MipsXFindFunctions`, second analysis pass, then decompilation to C: 724 functions, none failed, 93.0% of executable memory disassembled (`spa3000-3.1.20(GW)/decompiled-sec1/`, Ghidra project in `spa3000-3.1.20(GW)/ghidra-sec1/`). Only summary counts were read; the C has not been read yet. **This is decompilation.** | Nothing yet |
| 2026-10-27 | Claude as analyst | `spa3000-3.1.20(GW)/decompiled-sec1/decompiled.c` (section B); printable strings of section B's data stream with their load addresses; a few constant words from that stream | Read the start-up path, network set-up, main loop, the image download, check and flash-writing routine, the boot-record and upgrade-log writers, the factory-record reader and the restart paths. **This is decompilation.** No code, names or text carried over. Protocol constants (port, multicast group, magic, message length), the image magic and the flash offsets are interface facts. | `docs/spec/firmware-image-format.md`: section B role in §3, "What section B does" in §5, boot-record blocks and flash layout under Section C in §5, new §7.1 (recovery protocol), Q6 and Q7; recovery note in `docs/plan/open-source-stack.md` §3.5 |
| 2026-10-27 | Claude as analyst | `spa3000-3.1.20(GW)/decompiled/decompiled.c` (section A); printable strings of section A's stream at 0x1000 with their load addresses; a few constant words from that stream | Searched the decompilation for references to boot-record, log, image-magic and restart strings. Read the image install routine and its version checks, the boot-record block writer and record validator, the parameter-store block switch, the flash-size detection and the restart path, and compared each with section B's. **This is decompilation.** No code, names or text carried over. | `docs/spec/firmware-image-format.md`: boot record fields under Section C in §5, boot-record blocks, flash layout and section placement, the Placement subsection, new §7.2 (installing from the application), Q6 and Q7 |
| 2026-10-27 | Claude (reading) | Public document: ESS *ES3890 Video CD Processor Product Brief* (SAM0487-031704), read as HTML pages on alldatasheet.com; the unit's photos already in `docs/photos/` | Pin table summarised in own words; package orientation and U2 traces read from photos. No firmware material involved. Not downloaded, so no hash below. | `docs/spec/hardware.md`: datasheet table, §2.5 (ES3890F pins and orientation), U2 and REG1 rows, H2 and H6 |
| 2026-10-27 | Claude as analyst | Linksys PAP2T firmware 5.1.6(LS) and 5.1.1(LS) (`firmware/sipura/pap2t-*/`, see `firmware/sipura/PAP2T-PROVENANCE.md`), unpacked in a scratch directory | `sipura_unpack.py` (all checksums verified); block and section comparison with SPA2002 3.1.5 and SPA2102 5.1.6; ghidra-mipsx headless import and auto-analysis of sections 2, 1 and 0 (**this is disassembly**, but only summary counts were read) | Tested-firmware rows and PAP2T mentions in `~/src/ghidra-mipsx/README.md` and `docs/`. The PAP2T image uses the flash container and the SPA2002's layout and base-register values, and its section 0 is byte-identical to the SPA2002's |
| 2026-10-27 | Claude (reading) | Public documents: *Sipura SPA-3000 Simplified Users Guide* v1.1b (JMG Technology, 2005, from callvoip.nl); Sipura Technology *SPA User Guide* (July 2004, from provu.co.uk); ProVu SPA3000 datasheet (2006, from provu.co.uk) | Downloaded and read for upgrade and recovery procedures. No firmware material involved. They describe the PC upgrader, the `/upgrade?` URL and the Upgrade Rule parameters only; nothing on recovery. | `docs/reference/` (the three `20261027-*.pdf` files; since moved to a local `reference/` folder, not redistributed) |
| 2026-10-27 | Claude (reading) | Public web page: "S.O.S Firmware Recovery Protocol Specification", Cisco Community knowledge base (Dan Lukes, 2013), read in the browser; not saved | Compared with §7.1 of the format spec; facts restated in own words. No firmware material involved. | "Public description" in §7.1 of `docs/spec/firmware-image-format.md` |
| 2026-10-27 | Claude (reading) | Public source code: MAME (github.com/mamedev/mame, master branch, BSD-3-Clause): `src/mame/skeleton/pap2.cpp`, `src/devices/cpu/mipsx/mipsx.cpp`, the start of `mipsxdasm.cpp`, and the file list and commit history of pull request 12726. Read with `curl` from GitHub; not saved | Read the PAP2 driver's comments and memory map, and checked whether the CPU core executes instructions (it does not). Facts restated in own words. The driver names a PAP2 ROM dump; it was not obtained. No vendor firmware handled | `docs/spec/hardware.md`: "Public source" status mark, public source table, §2.6, hints in H3 and H4 |
| 2026-10-29 | Claude as analyst | Sipura SPA942 firmware 4.1.15 and 4.1.12(a) (`dirty-side/firmware/sipura/spa942-*/`, see `dirty-side/firmware/sipura/SPA942-PROVENANCE.md`), downloaded from Wayback Machine copies of sipura.com and unpacked in a scratch directory | `sipura_unpack.py` (all checksums verified); flash block occupancy and section comparison with SPA941 4.1.15; ghidra-mipsx `test/run-corpus.sh` on sections 2 and 1 (**this is disassembly**, but only summary counts were read) | Tested-firmware rows in `ghidra-mipsx/README.md`; SPA942 mentions in its `docs/` (magic, header field 0x40, r22 and r26 records). The SPA942 uses the flash container with the SPA2100/SPA2102/SPA9000 layout: entry record present, no uncompressed code outside the sections |
| 2026-10-29 | Claude as analyst (separate session) | SPA3000 3.1.20(GW) section C, run on mipsx-sim 0.1 (mipsx-sdk `debe872`, unmodified), with the 8 type 0x10 stream records of sections A and B and the real sections A and B in a simulated flash image | **Running vendor code** (see "Validating the simulator on vendor code"). (1) The decompression service called through a trampoline that sets only the §4.6 entry registers; output compared with zlib for all 8 streams, under the spec reading and every `--variant`; pipeline-rule events counted from full traces. (2) The loader and boot selector run black-box on 25 flash images built from §5 (good, single corruptions, no boot record, synthetic sections of chosen lengths), with the outcome being which section starts. Harness workarounds: RAM extended over the flash window and peripheral block; in the harness copy only, stored jump offsets adjusted for the 2³⁰ wrap. The decompilation was not read. Traces and harness kept only in `dirty-side/simval/sec0/` | `processor-es3890.md`: new Simulated mark, 30-bit code addresses (§1, §6), opcode 100 zero-extends (§3), SQ (§4), `bic` (§5), entry bit 30 (§8), 0x1CFFFFFC (§8), new §9 (pipeline rules). `firmware-image-format.md`: Simulated mark, §4.4 bit 30, §4.6 entry values sufficient, §5 Placement address, section check results, the unresolved disagreement with the boot order, §6, Q7 and Q9 |
| 2026-10-29 | Claude as analyst (separate session) | Section B of SPA3000 3.1.20(GW) and 3.1.10(GWd), SPA2102 5.1.6, PAP2T 5.1.6(LS), SPA942 4.1.15, SPA941 4.1.15, SPA9000 3.2.5(c), SPA2100 3.3.6 and SPA2002 3.1.5; section A of both SPA3000s, SPA2102, SPA942, SPA9000 and SPA2100; unpacked with `sipura_unpack.py` (all checksums verified) | **Running vendor code.** Each section's start-up run on mipsx-sim 0.1 (unmodified) with `--hazards=warn` under the default rules and every variant, with peripherals unmapped and again backed by RAM, until the first stop. Traces streamed and read only to triage stop reasons; nothing kept outside `dirty-side/simval/sweep/` | `processor-es3890.md`: jump target wrap (§6), OP 001 encodings other than `hsc` in later firmware (§6), first peripheral range (§8), pipeline rules (§9). `firmware-image-format.md`: the loader stub takes its packed header in r9 (§4.5) |
| 2026-10-29 | Claude as analyst (separate session) | `spa3000-3.1.20(GW)/decompiled-sec0/embedded/decompiled.c` (the boot selector and its two section-check routines); the earlier boot-selector harness and flash images in `dirty-side/simval/sec0/`, rerun on mipsx-sim 0.1 (mipsx-sdk `debe872`, unmodified) | **This is decompilation and running vendor code.** Re-read the selection order, the fallback, both check routines and the jump against the failing simulated cases; traced the no-record case through the selector's control code and the type 1 check (trace read, not kept). Found a harness fault, not a simulator fault: the harness put section A at flash 0x30000, over the end of section B, so B failed its MD5 whenever A was present. Rebuilt with A at flash 0x50000 and reran all cases, plus new ones for the tag check on each path and for odd lengths against misaligned start addresses. No code, names or text carried over; harness changes and results only in `dirty-side/simval/sec0/` | `firmware-image-format.md` §5 (Section C): the boot order and fallback to section B confirmed by simulation, the "Simulation disagrees" paragraph replaced; tag checked only on the scan path; length-limit word 0x168 for the record path; start address must be a multiple of 4 or the selector stops. §6 rewritten to match, Q7 updated. No processor change: the simulator ran the selector correctly |

Hashes of the original files:

```
sha256 5d61ce775a8c2aba4f39a5c21c3156ddd9a9382f19758bd503175ac8c8259bf2  spa3000-3.1.20(GW).zip
sha256 e32d18fab22dcc2676c681987ec46e2f72432d1d8ca03233b3fe0f35ea558e70  spa3000-3-1-20GW.bin
sha256 251c4633e3fc704f47a3b7d2a77cea855187b33056bda3fede1009afdda4078a  upg-spa3000-3-1-20GW.exe
sha256 d9d5c386962ba8889cb83edb1a09efa079765319d40f6592cb64745bace1f6f0  reference/mipsx.isetman.pdf
sha256 d54af8446cf6c45de0376abc3a40a8a8708a966083f0893193d998eca1ccd043  reference/20261027-ProVu_SPA3000_Datasheet.pdf
sha256 e5c53af6685b7ad44b50e0f6fb8e8a8b6ae54dfa424ad6dc32ba5dd002f31caf  reference/20261027-Sipura_SPA3000_Simplified_Users_Guide_v1_1b.pdf
sha256 3b49ec346f5e30c10bf0c814eeb51d8ee78be5fe7c02b20ead70b51956566388  reference/20261027-Sipura_SPA_User_Guide_v2_0_9.pdf
```

## Suggested next dirty-side tasks

Each should end in a spec document, not in shared code.

1. **Identify the CPU.** Done: ES3890F, read from the chip (see
   `spa3000/hardware.md`), and the instruction set is confirmed against
   the code (`common/processor-es3890.md`). Next: understand the
   remaining unclassified regions and section C. Reading disassembly in
   Ghidra ([ghidra-mipsx](https://github.com/openmipsx/ghidra-mipsx)) is dirty-side work, and whoever does it
   is dirty-side for that material.
2. **Resolve the unknown header fields** listed under "Open questions" in
   the format spec, especially anything the bootloader checks before it
   accepts an image.
3. **Hardware map.** Peripherals, GPIOs for LEDs and the reset button, the
   SLIC (FXS) and DAA (FXO) chips and how they are wired, flash layout and
   size, RAM size.
4. **Boot and recovery.** How the bootloader starts the application, and
   whether there is a recovery path if an upgrade fails. Learn this before
   flashing any experimental image.
5. **External behaviour.** Configuration parameters, provisioning, web UI
   and IVR functions, documented from the public administration guide and
   from watching the device on the network, not from the image.
