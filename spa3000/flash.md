# SPA3000 flash contents

Status: draft, 2026-10-30. Dirty-side findings written as facts for the
clean side, from reading every part of the stock firmware that reads or
writes the flash. Contains no vendor code. No real dump of this unit
exists yet: the first dump is the test of everything here, above all the
configuration-store record format. See [`../clean-room.md`](../clean-room.md)
for the process.

Marks as in [`../common/firmware-image-format.md`](../common/firmware-image-format.md):
**Observed** (seen in the firmware), **Datasheet**, **Hypothesis**.
All multi-byte values are big-endian.

The image sections themselves, the boot records and the boot selector
are in the format spec §5. This document covers what else is on the chip,
what must survive our firmware, and how to check a dump.

## 1. Chip

Am29LV081B, 1 MB, sixteen uniform 64 KB sectors (0x00000, 0x10000, …,
0xF0000); autoselect ID manufacturer 0x01, device 0x38; AMD command set
with unlock addresses 0x555/0x2AA (only A10–A0 are decoded, so the
firmware's 0x5555/0x2AAA work too); sector erase confirm 0x30; byte
program 0xA0; status by DQ7 polling with DQ5 as the timeout flag. There
is no software sector protection on this part: protection needs high
voltage on A9 or RESET#. (Datasheet; the firmware uses exactly these
commands, Observed.)

The stock firmware never issues chip erase, sector protect, unlock
bypass or erase suspend. It verifies every sector erase by reading the
sector back, retries erases and byte programs, and kicks the watchdog
inside its erase and program loops (Observed). It also probes for a
second flash chip at 0x18000000 before using 0x1C000000; whether
0x18000000 is another alias of this chip on this board is not known
(F1).

## 2. Map

Offsets in the 1 MB part (CPU addresses 0x0C000000 + offset).

| Range | Use | Preserve? |
|---|---|---|
| 0x00000–0x0FFFF, 0x10000–0x1FFFF | Two alternating configuration sectors (§4): boot record at +0, image-header copy at +0x400, event log at +0x800–+0xFFF, configuration store at +0x1000–+0xFFFF | Keep. Losing them loses the user's settings (a factory reset), not the unit: any install rebuilds the boot records |
| 0x20000 upward | Recovery program (section B) | Keep: it is the network recovery path |
| up to 0xEFFFF | Application | Ours to replace |
| **0xF0000–0xF1FFF** | **Factory identity record** (§3) | **Never erase or write** |
| 0xF2000–0xFFFFF | Boot program (section C) | Keep |

The factory record and the configuration store cannot be rebuilt from any
download, and the vendor recovery protocol does not restore them. There
is **no calibration data** in flash: nothing for the Si3210 or Si3050
was found in the factory record or elsewhere, so their settings come from
the firmware's own tables and the configuration store (Observed, Medium:
absence of evidence over every reader of the factory record).

## 3. Factory identity record (0xF0000)

Observed, High unless marked. **No checksum, magic or version field
covers it**, so a dump can only be checked for plausibility.

| Offset | Size | Content | If blank or bad, the firmware uses |
|---|---|---|---|
| +0x00 | 6 | MAC address | 00:0E:08:00:01:00, if bytes 0–2 are all 0xFF |
| +0x06 | 0x1A | Not read | — |
| +0x20 | 32 | Serial number, NUL-terminated ASCII, at most 31 characters (the recovery protocol's identifier, format spec §7.1) | `00000000000000`, if the first byte is 0xFF or the string is too long |
| +0x40 | 32 | Model name, as above | `SPA-3000` |
| +0x60 | 32 | Hardware version, as above | `0.1.0(manuf)` |
| +0x80, +0x84 | 4 + 4 | Offset (from 0xF0000) and length of the TLS private key | no key, if the offset is ≤ 0 |
| +0x88, +0x8C | 4 + 4 | Offset and length of the TLS certificate | no certificate, if the offset is ≤ 0 |
| +0x90, +0x94 | 4 + 4 | Offset and length of a provider customisation table | not customised (a length of 0xFFFFFFFF means: look in the configuration store instead) |
| up to +0x1FFF | | The three blobs, where the pairs above point | |

- **Key**: DER, obfuscated by a fixed permutation within each whole
  256-byte block: plain byte *j* (0–255) of a block is stored at position
  ((0x89 × (*j* + 1)) mod 0x101) − 1 of that block. A final partial block
  is stored as is.
- **Certificate**: plain DER.
- With either missing, the firmware uses a built-in generic key and
  certificate and reports the certificate as not installed.
- **Customisation table** (Medium): scrambled with the configuration
  store's scrambler (§4, one run over the whole blob). Plain, it starts
  with an 8-byte magic, `LINKSYSC` or `SIPURA C`, then 8-byte entries
  from +0x08 (a 16-bit parameter id, 16 bits not understood, a 32-bit
  offset from the blob start to a NUL-terminated default value), ended
  by id 0. A provider's default or locked profile.

A unit with an erased factory record still runs, with the defaults
above; two such units on one LAN would clash on the MAC. This unit's
case label gives the MAC to expect (OUI 00:18:F8, a Linksys block, not
the default's 00:0E:08).

## 4. Configuration sectors

Sectors 0 and 1 have the same layout. Which one is active, and the boot
record itself, are in the format spec §5 ("Boot record" and "Boot-record
blocks").

**Event log** (Observed, High). 32 slots of 0x40 bytes, slot *k* at
+0xFC0 − 0x40·*k*: a 32-bit sequence number, then up to 60 bytes of ASCII
text. New entries go in the next free slot of **both** sectors. When the
log is full it stops growing until the next compaction, which keeps the
latest 16.

**Configuration store** (Observed, High for the record format; Medium
for the flags byte and the exact compaction triggers).

- An append-only log of records from +0x1000 to +0x87FF of the active
  sector, with an identical copy (the mirror) from +0x8800 to +0xFFFF.
  Each append writes the primary, then the mirror.
- Each record is scrambled byte by byte (below). Descrambled:

  | Offset | Size | Content |
  |---|---|---|
  | 0 | 1 | Flags (bit 0 is read; meaning unknown) |
  | 1 | 2 | Parameter id, 1 to 0xC83 |
  | 3 | 2 | Value length L |
  | 5 | L | Value |
  | 5 + L | 2 | CRC-16 (below) of bytes 0 to 4 + L |

  Records are packed with no alignment, L + 7 bytes each, at most 0x1000.
  A later record for an id supersedes an earlier one. The log ends at the
  first 7 bytes that are all 0xFF, and everything after it must be
  erased. A record with id 0, a length past the end or a bad CRC is an
  error.
- **Scrambler.** An 8-bit state *s*, starting at 0xF3 for each record.
  For each stored byte *c*: plain = *c* XOR *s* XOR (*c* >> 7) XOR
  (*c* >> 6); then *s* = (*c* XOR (*c* << 1) XOR (*c* << 2)) mod 256.
- **CRC-16.** Polynomial 0x1021, register initialised to 0xFFFF, message
  bits shifted into the low end of the register (the augmented form) but
  without appending zero bytes, no final XOR; the result is the register.
  Check value for the ASCII string `123456789`: **0xA69D**.
- **Compaction.** When a record would end past +0x8400, or when at load
  the active sector has an error or only one sector is valid, the store
  erases the other sector and rewrites it: the latest 16 log entries, one
  record per live parameter, the first 0x80 bytes of the image-header
  copy (not the full 0x180), and finally the boot record with the
  sequence number increased, first byte last. That sector then becomes
  active. So the sectors alternate only on compaction, not on every save.
- If a record is bad in one copy, the firmware takes it from the other
  and compacts. If it is bad in both, parsing stops there and the
  compaction keeps only what came before it.

## 5. Who erases what in the stock firmware

| Writer | Erases | Factory sector? |
|---|---|---|
| Configuration save | nothing (append) | no |
| Compaction, boot-record update (application and recovery program) | the inactive configuration sector | no |
| Install of the application (either program) | from below the new application up to 0xEFFFF | no |
| Install of the recovery program (application only) | 0x20000 upward | no |
| **Install of the boot program (application only)** | **0xF0000**, after saving its 8 KB to RAM; then writes them back and the new boot program | **yes, saved and restored** |

**Hazard**: a power cut during the boot-program install, between the
erase and the write-back, loses the unit's identity for good (Observed,
High). Two further routines that would erase or write the factory sector
exist but were found to have no callers (Observed, Medium).

## 6. What our firmware must preserve

- Never erase sector 15 (0xF0000). If the boot program must ever be
  replaced, do it as the stock firmware does (save 8 KB, erase, restore)
  and accept the power-cut risk, or better, leave it alone.
- Treat 0xF0000–0xF1FFF as read-only identity: read the MAC and serial
  number there.
- Leave sectors 0 and 1 in the vendor format, or migrate the parameters
  deliberately.
- Keep the recovery program at 0x20000 and a valid type 1 slot in the
  boot record (format spec §5, §6).

## 7. Checking a dump

A good dump has:

- two reads that are identical;
- for each section reached through the boot records, a table entry whose
  byte sum holds and data whose MD5 matches (format spec §3, §5);
- boot records whose complements hold;
- a configuration store in which every record passes its CRC, followed
  only by erased bytes, with the mirror equal to the primary;
- a factory record with a non-blank, unicast MAC whose OUI is a Sipura
  or Linksys one and which matches the case label; printable strings at
  +0x20, +0x40 and +0x60; blob pairs inside the 8 KB; a key that
  unpermutes to DER and a certificate that parses as DER.

Signs of a bad read: the dump repeats every 2^*n* bytes (an open address
line), a data bit never changes, or the whole dump is 0xFF or 0x00.

## 8. Open questions

| # | Question | How to answer |
|---|---|---|
| F1 | Is 0x18000000 another alias of the flash on this board? | Read the autoselect ID there from our own code, or watch the chip selects |
| F2 | The meaning of the configuration record's flags byte, and whether a real unit's records match §4 | The first real dump |
| F3 | Does anything sit in the factory record between +0x98 and the first blob? | The first real dump |
| F4 | Is anything stale between section B's end and 0x50000, or above the application? | The first real dump |
