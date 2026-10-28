# SPA3000 firmware image format

Status: draft, from analysis of a single image, version 3.1.20(GW)
(775,376 bytes). Clean-side document: it holds facts about the format only,
with no code or text from the vendor firmware. See
[`../clean-room.md`](../clean-room.md) for the process.

Each fact is marked:

- **Verified**: checked by computation against the image (checksums match,
  lengths add up, streams decompress).
- **Observed**: the value is in the image, but its meaning is unknown or
  inferred.
- **Hypothesis**: a plausible explanation that has not been confirmed.

All multi-byte integers are **big-endian** unless stated otherwise. Offsets
are in hex. "Section data" means the bytes counted by a section's length
field.

## 1. Overall layout

```
0x000  global header                 0x80 bytes
0x080  section table                 3 entries × 0x80 bytes
0x200  section A: entry copy (0x80) + data
       section B: entry copy (0x80) + data
       section C: entry copy (0x80) + data
EOF    (sections are contiguous; the last one ends exactly at end of file)
```

Verified: the table entry lengths plus 0x80 per section, starting at 0x200,
add up exactly to the file length.

## 2. Global header (0x000–0x07F)

| Offset | Size | Value in 3.1.20(GW) | Meaning | Status |
|---|---|---|---|---|
| 0x00 | 16 | ASCII `SpA3000 FiRmWaRe` (no terminator) | Magic | Observed |
| 0x10 | 32 | zero | Reserved | Observed |
| 0x30 | 16 | MD5 digest | MD5 over file bytes 0x000–0x1FF, computed with this 16-byte field set to zero | **Verified** |
| 0x40 | 16 | zero | Reserved | Observed |
| 0x50 | 4 | 0x00000080 | Probably the global header size | Hypothesis |
| 0x54 | 4 | 0x00000080 | Probably the table entry size | Hypothesis |
| 0x58 | 4 | 0x000BD4D0 | Total file length in bytes | **Verified** (equals file size) |
| 0x5C | 32 | `3.1.20(GW)`, NUL-padded | Image version string | Observed |
| 0x7C | 4 | 0x00000003 | Number of sections | Hypothesis (matches the table) |

The global MD5 covers the header and the whole section table, but not the
section data. The section data is protected by the per-section MD5 below.

## 3. Section table entry (0x80 bytes)

Entries start at 0x080, 0x100 and 0x180. Each section's data is preceded
in the file by a byte-for-byte copy of its table entry (verified for all
three).

| Offset | Size | Meaning | Status |
|---|---|---|---|
| 0x00 | 4 | Zero | Observed |
| 0x04 | 4 | Tag for the section type: 0xC6F4B6AF (type 0), 0xB8B4D4AD (type 1), 0xCACAD3C3 (type 2). The bootloader compares it with a per-type table in its parameters (§5, Section C), up to the first zero byte of the table entry. | Observed |
| 0x08 | 1 | Section type: 2, 1 or 0 | Observed |
| 0x09 | 3 | Zero | Observed |
| 0x0C | 4 | 0x00000080: offset from the start of the entry to the section data. The bootloader uses it to find the data. | Observed |
| 0x10 | 16 | MD5 of the section data (not including the entry copy). The bootloader checks it. | **Verified** for all three; check Observed |
| 0x20 | 4 | Section data length in bytes. The bootloader rejects lengths above 0xFFFFF. | **Verified**; limit Observed |
| 0x24 | 4 | Zero | Observed |
| 0x28 | 0x56 | Section version string, NUL-padded | Observed |
| 0x7E | 2 | Sum of entry bytes 0x00–0x7D as unsigned bytes, mod 0x10000. The bootloader reads the 32-bit word at 0x7C and compares it with the sum of bytes 0x00–0x7B; the two agree while bytes 0x7C–0x7D (the end of the version field) are zero. Keep them zero. | **Verified** for all three; check Observed |

Sections in 3.1.20(GW), in file order:

| # | Type | Version | Data offset in file | Data length | Encoding | Likely role |
|---|---|---|---|---|---|---|
| A | 2 | 3.1.20(GW) | 0x00280 | 0x9A794 | Packed (§4) | Main application (Observed: web server, SIP, crypto) |
| B | 1 | 1.0.6(3k) | 0x9AA94 | 0x14A3C | Packed (§4) | Recovery and upgrade program (Observed; §5) |
| C | 0 | 1.1.1 | 0xAF550 | 0x0DF80 | Raw, not compressed | Bootloader (Hypothesis, strongly supported; §5) |

## 4. Packed section format (types 1 and 2)

Offsets in this section are relative to the start of the section data.

### 4.1 Section header (0x00–0x1F)

| Offset | Size | Section A | Section B | Meaning | Status |
|---|---|---|---|---|---|
| 0x00 | 4 | 0x00060803 | 0x00060803 | Constant; perhaps a format version | Observed |
| 0x04 | 4 | 0x0009A290 | 0x000145C4 | Offset of the load directory (§4.4) | **Verified** |
| 0x08 | 4 | 12 | 5 | Number of load directory records | **Verified** |
| 0x0C | 4 | 0x001FFFF0 | 0x001FFF00 | Unknown. Hypothesis: initial stack pointer near the top of RAM. The board has 2 MB of SDRAM ([`hardware.md`](../spa3000/hardware.md)), so this is supported. | Hypothesis |
| 0x10 | 4 | 0x0009A794 | 0x00014A3C | Section data length (same as the table entry) | **Verified** |
| 0x14 | 4 | 0x0009A784 | 0x00014A2C | Section data length − 0x10 | Observed |
| 0x18 | 8 | zero | zero | Reserved | Observed |

### 4.2 Auxiliary chunks (from 0x20)

At offset 0x20 there is a list of zero or more chunks:

```
u8   0x01         chunk marker
u8   type
u16  0
u32  length       (big-endian)
u8   data[length]
```

The list ends at the first position whose first byte is not 0x01. In
section B the list is empty. In section A there are three chunks:

| Type | Length | Contents | Status |
|---|---|---|---|
| 1 | 0x1775C | Concatenation of 85 variable-size blobs. The data is high entropy and is not deflate. | Observed. Hypothesis: compressed voice prompts for the phone menu (IVR). |
| 2 | 0x2A8 | Index into chunk 1: 85 pairs of (u32 size, u32 offset). Offsets are cumulative; some entries have size 0. | **Verified** (offsets plus sizes add up to the chunk 1 length) |
| 3 | 0x138 | Short byte sequences | Observed. Hypothesis: phrase definitions that combine chunk 1 blobs. |

New firmware would supply its own prompts, so these chunks matter only if
you want to stay compatible with the original layout.

### 4.3 Compressed stream records

After the chunk list come the stream records. Each record is found by its
offset in the load directory and is laid out as:

```
u8   pre[8]        meaning unknown (see Q3)
...  raw deflate stream (RFC 1951, no zlib or gzip header)
u32  CRC-32 of the uncompressed data   (LITTLE-endian; the standard
                                        CRC-32 used by zlib and gzip)
u32  uncompressed length               (LITTLE-endian)
...  zero padding to a 4-byte boundary
```

Verified for all eight streams: each decompresses cleanly and its CRC-32
and length match. The trailer is the same as a gzip member trailer.

### 4.4 Load directory

The directory sits at the offset given in header field 0x04 and holds
`count` records of 20 bytes (5 × u32). The first word is the record type.

| Type | Word 1 | Word 2 | Word 3 | Word 4 | Meaning | Status |
|---|---|---|---|---|---|---|
| 0x10 | Offset of the stream record (§4.3) | Load address | Unknown (see Q4) | 1 | Decompress stream to load address | **Verified** (offsets point to streams; load addresses tile memory without overlap, see §5) |
| 0x02 | Address | Length | Varies | 0 | Zero-fill (BSS). Each region starts exactly where a loaded stream ends. | Hypothesis, strongly supported |
| 0x07 | Word address: 0x00016D34 (A), 0x00014000 (B) | Varies | Varies | 0 | Initial value of register r23, the code base register used for calls and far jumps. It equals the word address of the first code stream plus 0x10000. **Not the entry point.** | **Verified** for A; B fits the same rule (Observed) |
| 0x05 | Word address: 0x0004BC00 (A only) | Varies | Varies | 0 | Initial value of r24, a second code base register. It equals the word address of the second code stream (byte 0xEF000) plus 0x10000. | **Verified** |
| 0x06 | 0x00010000 (A only) | Varies | Varies | 0 | Probably the initial value of r25 | Hypothesis |
| 0x03 | 0x400070A6 (A), 0x4000CB6A (B) | Varies | Varies | 0 | Entry point; always the last record. Bits 29–0 are a **word** address (A: 0x70A6, byte 0x1C298; B: 0xCB6A, byte 0x32DA8). Bit 30 is a flag of unknown meaning. | A: **Observed** (the address follows a return and, in the first instructions, sets the stack pointer from memory before jumping into the application). B: Hypothesis, by the same rule. |

Code addresses in the directory and in the processor are word addresses
(byte address = word address × 4); see [`processor-es3890.md`](processor-es3890.md)
§8, which records how the loader stub's handling of these records was
established. The 0x07, 0x05 and 0x06 words are therefore not locations in
memory to load anything: they are register values supplied to the
application.

"Varies" means the word does not look meaningful: its value changes
irregularly and seems to be leftover buffer contents from the vendor's
build tool. Hypothesis: the loader ignores these words. When generating an
image, set them to zero, but test this first (see Q5).

### 4.5 Trailing area

After the directory, all three sections end with the same 0x414 bytes of
MIPS-X code: a loader that walks the load directory (Observed). It does
not contain a decompressor. For each type 0x10 record it calls a
decompression service in the boot section (§4.6). So a packed section with
type 0x10 records depends on the stock boot section being in flash. New
firmware can provide its own loader, and need not use the service (§4.6).

On type 0x03 (entry) the loader jumps to the entry word address with r4
holding the address of the section's packed header (Observed).

### 4.6 Decompression service in the boot section

The stock boot section (section C) provides raw-deflate decompression to
the loaders of the other sections. Its interface, as the loader uses it
(all Observed from the loader's code; the service's own entry conditions
beyond these are not known):

**Location.** A descriptor at flash address 0x0CFF2400, which is section
C data offset 0x0380 when section C sits at the top of the flash window
(§5, Section C):

| Offset | Size | Content |
|---|---|---|
| 0x00 | 4 | Length in bytes of a data block to copy to RAM (0x310 in 1.1.1). 0 means nothing to copy. |
| 0x04 | 0x1C | Not used by the loader. In 1.1.1 the next word is 0x221C (unknown meaning; possibly the code length). |
| 0x20 | length | Data block: copied word by word to RAM at 0x00180000. In 1.1.1 it holds the deflate tables (§5). |
| 0x20 + length | — | Code. The loader calls its first word; it executes in place from flash. |

**Call, per type 0x10 record:**

| Register | Value on entry |
|---|---|
| r4 | Source: address of the stream record (§4.3) |
| r5 | Destination: the record's load address |
| r24 | Code base: word address of the service's first code word + 0x10000 |
| r25 | 0x00180000, the copied data block |
| r19 | 0x00187FF0 |
| r29 | 0x001EFFF0 (stack pointer) |
| r31 | Return address (the call is a `jspci` through the code word address) |

The loader keeps its own state in RAM at byte addresses 0x0C–0x33 across
the call (it saves registers there), so the service must not use that
range. The loader does not look at any return value, so a corrupt stream
is not detected by the loader itself; whether the service checks the
stream's CRC-32 trailer is not known. The data block is re-copied before
every stream.

The running application does not use this service: it contains its own
deflate decoder, and no reference to the 0x0CFFxxxx range was found in
it (Observed, by search only; a computed address would not show).

Consequences for new firmware:

- An image in this format that uses type 0x10 records, including any
  vendor section B kept as a fallback, needs a boot section that provides
  this service at this address with this interface.
- Our own sections can avoid it: store code uncompressed, as section C's
  embedded program does (type 0x01 records, §5), or carry a decompressor
  in their own loader.
- A replacement boot section must provide the service if it is to start
  vendor images.

## 5. Load map

### Section A (application, 3.1.20(GW))

| Load address | End (exclusive) | Uncompressed size | Content (strings, then instruction decoding) |
|---|---|---|---|
| 0x00001000 | 0x00009000 | 0x8000 | Does not decode cleanly as code (see below). Strings suggest network start-up and diagnostics. |
| 0x00009000 | 0x000096D6 | 0x6D6 | Zero-fill |
| 0x00010000 | 0x00018F38 | 0x8F38 | Does not decode cleanly as code (see below). Strings suggest the embedded web server. |
| 0x00018F38 | 0x0001989E | 0x966 | Zero-fill |
| 0x0001B4D0 | 0x00097C68 | 0x7C798 | Code (the r23 region) |
| 0x00097C68 | 0x000A6D5C | 0xF0F4 | Not clean code: mixed code and data; includes crypto tables (AES) |
| 0x000EF000 | 0x0015CC80 | 0x6DC80 | Code (the r24 region) |
| 0x0015CC80 | 0x0017112C | 0x144AC | Data tables |

Entry point (type 0x03 record): word 0x70A6, which is byte address
0x0001C298, inside the first code stream (Observed; see §4.4). The
0x00016D34 value earlier listed here as the entry point is the r23 code
base, not an entry point.

Code status: the streams at 0x1B4D0 and 0xEF000 decode completely as
MIPS-X (Verified; [`processor-es3890.md`](processor-es3890.md)). The earlier
guess, from strings alone, that the 0x1000 and 0x10000 regions are code
was wrong or incomplete: they do not decode cleanly as code. What they are
is unknown (Q9). The stream at 0x97C68 is likewise mixed or data.

Contiguous stream ends (0x97C68, 0x15CC80) are **verified**: one stream
ends exactly where the next begins. The highest address used is about
1.45 MB, which fits under the 0x1FFFF0 value in header field 0x0C.

### Section B (1.0.6(3k))

| Load address | End (exclusive) | Uncompressed size | Content |
|---|---|---|---|
| 0x00000100 | 0x00001B18 | 0x1A18 | Code |
| 0x00001B18 | 0x00002AF8 | 0xFE0 | Zero-fill |
| 0x00010000 | 0x0003D7E4 | 0x2D7E4 | Code |

Entry point (type 0x03 record): word 0xCB6A, byte address 0x00032DA8,
inside the stream at 0x10000 (Hypothesis, by the same rule as section A).
The type 0x07 value 0x00014000 is the r23 code base, not the entry point.
It equals the word address of the stream at 0x10000 (0x4000) plus 0x10000,
which fits the section A rule and suggests that stream is code (Observed).
The processor spec does not otherwise classify section B's streams.

Decompilation confirms the stream at 0x10000 is code (724 functions, with
the entry point at 0x32DA8) and shows that the stream at 0x100 is its
initialised data: strings, constants and pointer tables (Observed).

#### What section B does

Established by decompiling section B (see the clean-room log). All
Observed unless marked.

Section B is a **network recovery and upgrade program**. It has a small
real-time kernel, the Ethernet driver, an IP stack with DHCP and DNS, a
TFTP client, an HTTP client and the flash driver. It has no telephony, no
web server and no user interface beyond the LEDs. Once started it never
returns and never starts another section itself. It leaves by restarting
the unit, so that the boot selector chooses again (§5, Section C).

**Start-up.**

1. Creates its kernel objects and checks the flash chip, retrying once.
   If the flash does not respond it stops with an error LED pattern.
2. Reads the factory record at flash offset 0xF0000 (the start of the
   last 64 KB sector): MAC address at +0x00 (6 bytes), an identifier
   string at +0x20 (Hypothesis: the serial number), the model name at
   +0x40 and a further version string at +0x60. Each field falls back to
   a built-in default if it is blank (first byte 0xFF) or longer than 31
   characters. The defaults include the model name `SPA-3000`, so a
   unit with an empty factory record still runs.
3. Reads the version of the installed image from the active boot-record
   block (+0x400 + 0x5C, the global header's version field, §2).
4. Brings up the network: either the stored address, or DHCP. If the
   stored address is already in use it tries other addresses in the same
   subnet. If DHCP fails it probes 192.168.1.100–200, 192.168.2.100–200
   and 192.168.3.100–200 for a free address.

**Main loop.** It does two things in turn:

- **Answers a recovery protocol** on UDP port 4810, joined to multicast
  group 234.4.8.10 (§7).
- **Retries stored upgrade rules.** Three parameters from flash
  parameter storage hold upgrade rules, probably the application's
  provisioning rules (Hypothesis). The rules use the same kind of
  conditional syntax, with variables for software and hardware version,
  an upgrade condition, a scheme and a server address. Section B
  evaluates each non-empty rule on a timer. The interval starts short,
  doubles on each attempt and is reset when it passes a limit. If the
  rule's condition holds, it resolves the server name and fetches the
  image over `tftp` or `http` from each resolved address in turn. After
  a successful upgrade by rule, it restarts the unit.

**Installing an image** (both paths use the same routine):

1. Download the whole image into RAM.
2. Reject it if it is shorter than 0x100 bytes or does not start with
   the magic `SpA3000 FiRmWaRe` (§2). So it only accepts SPA3000 images.
3. Reject it if it is shorter than its header and table say it should be.
4. Check the global header MD5 (§2).
5. Parse the image version (§2, field 0x5C) as numbers. Reject it if the
   version cannot be parsed, has major version 0, or is 1.0.x or 2.0.x.
   This refuses downgrades to old firmware generations; 2.1 and later
   are accepted.
6. Check every section (Hypothesis: the same table entry and MD5 checks
   as the boot selector).
7. Write a short upgrade log entry naming the new version (see the
   boot-record blocks under Section C).
8. Write only the sections of **type 2 or above** to flash, back to back,
   so that they end at flash offset 0xF0000, just below the factory
   record and section C. Each is written as its table entry followed by
   its data, rounded up to a multiple of 4 bytes. The 64 KB sectors in that
   range are erased first. After writing, everything is read back, and
   the erase and write are repeated until the read-back matches. Types 0
   and 1 in the image are **not** written, so section B cannot replace
   itself or the boot section.
9. Build a new boot record: the type 2 slot points to the new section's
   table entry in flash (at 0x0C000000 + offset), with its complement
   as the check word. Copy the image's first 0x180 bytes (global header
   and section table) beside it. Then write both to the two boot-record
   blocks in turn (see Section C).

Error codes from this routine were seen, but their reporting path was
not traced.

**Restart.** A restart is requested by setting flags that the main loop
acts on. A normal restart writes a small message block to RAM, protected
by a checksum, and then resets the chip through peripheral registers.
Whether the boot selector or section A reads that block is not known.
After an upgrade started by the recovery protocol, the program does not
restart. It stops in a loop with a fixed LED state, so a power cycle (or
perhaps a watchdog) is needed before the new image runs (Observed;
whether a watchdog is armed is not known).

### Section C (1.1.1)

Stored raw, 0xDF80 bytes, with no packed header at the start. It is
MIPS-X code and data (Observed), and is probably the bootloader
(Hypothesis, strongly supported; see below). It runs in place from the
top of the flash window (Hypothesis, strongly supported; see "Placement"
below).

The earlier finding that it "does not decode as MIPS-X" was an artefact:
exactly two-thirds of its 32-bit words are zero, and a zero word decodes
as a branch with condition 000. The non-zero regions have the same mix of
instruction types as section A's code, including the application's no-op
0x60000019 (Observed).

Layout (offsets within the section data):

| Range | Content | Status |
|---|---|---|
| 0x0000–0x0380 | Zero | Observed |
| 0x0380–0x28CC | The decompression service of §4.6: descriptor at 0x380 (data length 0x310, then 0x221C), data block 0x3A0–0x6B0 copied to RAM, code from 0x6B0. The data block holds the standard deflate decoder tables (at 0x3B4: bit masks 1, 3, 7 … 0xFFFF; the code-length order 16, 17, 18, 0, 8, 7, 9, 6 …; length bases 3, 4, 5 … 258). The code is a deflate decoder handling stored, fixed-Huffman and dynamic-Huffman blocks. | Observed |
| 0x28CC–0xB988 | Zero | Observed |
| 0xB988–0xDF80 | An embedded program in the packed container format (below) | Observed |

The embedded program at 0xB988 has a §4.1 header: 0x00060803, directory
offset 0x2180, 4 records, stack 0x001FFFF0, length 0x25F8 and length −
0x80 (0x2578). Its length runs exactly to the end of section C
(Verified). Unlike A and B it has no deflate streams: its code is stored
uncompressed from header offset 0x20. Its directory:

| Type | Word 1 | Word 2 | Word 3 | Word 4 | Notes |
|---|---|---|---|---|---|
| 0x01 | 0x1F58 | 0x100 | 0x8A | 1 | Record type not seen in A or B: copy stored data. Word 1 is a header offset (section offset 0xD8E0), word 2 the load address, word 3 the length in **words** (0x228 bytes, to 0x328). The program reads its boot parameters from this range (below). Observed. |
| 0x02 | 0x328 | 0x15 | Varies | 0 | Zero-fill of 0x15 **words**, 0x328–0x37C; the program's variables live there (Observed). So this record's length is in words here, unlike the byte lengths assumed in §4.4; whether A and B use words too is unchecked. |
| 0x07 | 0x10008 | Varies | Varies | 4 | r23 = word 8 + 0x10000, the §4.4 rule for a code stream at header offset 0x20 |
| 0x03 | 0x40000008 | Varies | Varies | 4 | Entry at word 8 (byte 0x20, the first word after the header); bit 30 set as in A and B |

After the directory, the section ends with the same 0x414-byte loader stub
that ends sections A and B (§4.5; Observed, matching runs of 64 bytes).

Evidence for the bootloader role:

- Sections A and B are deflate-compressed, and their 0x414-byte stubs
  call the decompressor at 0x0380 to unpack them (§4.6, Observed).
- The embedded program holds the MD5 starting values and first round
  constant (section offsets 0xDA00, 0xDA0C), and sixteen near-identical
  repeated blocks between 0xCC84 and about 0xD75C, which are probably
  MD5's four rounds written out in full (Hypothesis). The image and its
  section table use MD5 (§2, §3).
- The embedded program finds, checks and starts the type 1 and type 2
  sections in flash (Observed; next subsection).
- Section C is the only section stored uncompressed, and has type 0.

The code at 0x0380 uses r24 and r25 but never sets them; the loader stub
of each section sets them before calling it (§4.6, Observed).

#### Placement

Section C sits at the top of the flash window, with its data at
0x0CFF2080–0x0CFFFFFF and its section table entry just before it at
0x0CFF2000 (Hypothesis, strongly supported):

- The loaders of A and B find the decompression service at 0x0CFF2400,
  which is exactly section C data offset 0x380 at that placement (§4.6),
  and the code there executes in place.
- The loader stub, when run as section C's own, reads the word at
  0x1CFEFFFC and subtracts it from 0x0D000000 to find its packed header.
  Section C's last word is 0x25F8, the embedded program's length, and
  0x0D000000 − 0x25F8 is exactly where the embedded program starts at
  this placement (Observed arithmetic; why the read uses 0x1CFEFFFC rather
  than 0x0CFFFFFC is not known, perhaps an alias).
- The boot selector's scans cover 15 slots from 0x0C000000 (§5 below),
  which leaves out the sixteenth 64 KB sector, 0x0C0F0000.

- Section A's installer writes a type 0 section so that it ends at the
  top of flash: flash offset 0xF2000 for this section's 0xE000 bytes
  (Observed; §7.2).

Flash offset 0xF2000 is therefore confirmed. Seeing it at 0x0CFF2000
still relies on the 1 MB flash repeating through the window (address
bits above 19 ignored), which is not confirmed. This supersedes the
earlier guess that section C occupies sector 0.

#### Boot selection by the embedded program

Established by decompiling the embedded program (see the clean-room log).
All Observed unless marked.

Boot parameters, read from the range filled by the 0x01 record:

| Address | Value | Use |
|---|---|---|
| 0x15C, 0x160, 0x164 | Pointers to the type 0, 1 and 2 tags: 0xC6F4B6AF, 0xB8B4D4AD, 0xCACAD3C3 | Compared with entry field 0x04 (§3) |
| 0x16C | 0x000FFFFF | Largest accepted section length |
| 0x170–0x17C | 0x0C000000 and 0x0C010000, each twice | The two boot-record blocks |
| 0x188, 0x18C | 0x0C000000, step 0x10000 | Type 1 scan: start and step |
| 0x194, 0x198 | 0x0C000000, step 0x10000 | Type 2 scan: start and step |
| 0x180, 0x184, 0x190, 0x19C | 0x0FFFFFFF | Mask applied to the found data address before the jump |

Flash is therefore at 0x0C000000 (Hypothesis, strongly supported): the
board's flash is 1 MB in sixteen 64 KB sectors ([`hardware.md`](../spa3000/hardware.md)),
which matches the 0x10000 step, the fifteen-slot scans and the 0xFFFFF
length limit.

**Boot record.** A block starting with the 8 bytes `SIPURA *`, then a
sequence number at +0x08 (compared between blocks: the higher one wins)
with its complement at +0x0C. Section A writes the halfword 0x0004 at
+0x10 and 0x0010 at +0x14, each followed by its complement (+0x12, +0x16).
Their meaning is not known (Hypothesis: a format version and a slot
size). Section A checks all four pairs when validating a record; the boot
selector was not seen to. Then comes a slot for the type 1 section at
+0x20 and for the type 2 section at +0x30. Section A puts sections of
higher types, if an image has any, in further slots from +0x40. Each
slot is again `SIPURA *`, then the flash
address of a section table entry at +0x08 and a check word at +0x0C. A
slot is used only if the address is not 0xFFFFFFFF and the address OR the
check word is 0xFFFFFFFF (so the check word is presumably the complement).

**Section check.** At a section table entry (§3) in flash:

1. The type byte at 0x08 must be the wanted type.
2. If the type's tag in the parameters is not empty, entry field 0x04 must
   match it.
3. The word at 0x7C must equal the sum of entry bytes 0x00–0x7B.
4. The length at 0x20 must be at most 0xFFFFF.
5. The MD5 of the data (at entry + field 0x0C, for the field 0x20 length)
   must equal the digest at 0x10.

**Order.**

1. Check `SIPURA *` at both boot-record blocks, 0x0C000000 and
   0x0C010000; use the one with the higher +0x08 word, or the only valid
   one.
2. If there is one: try its type 2 slot, then its type 1 slot.
3. Otherwise, or if both slots fail: scan 15 positions from 0x0C000000 in
   steps of 0x10000 for a valid type 1 section, then the same for type 2.
4. Jump to: data start + packed header field 0x14 (data length − 0x10).
   In sections A and B, that word is a branch back into the loader stub
   (§4.5), so this is how the stub starts (Verified for A and B).
5. If nothing is found: initialise peripherals through base register r27
   (offsets 0x4000–0x8630) and loop forever. No serial or network recovery
   was seen on this path.

Where the boot records come from: the image itself contains no
`SIPURA *` record. Sections A and B write them when they install an
image (§5, Section B; §7.2), with identical code for the block update
below. Section A also rewrites the record, with the sequence number
increased, every time it saves parameters (Observed). An earlier guess,
that section C occupies sector 0 and its zero first 0x380 bytes hold the
first boot record, is superseded by the placement above.

**Boot-record blocks**, as sections A and B read and write them (Observed).
Flash sectors 0 and 1 (0x0C000000 and 0x0C010000) are a pair of
alternating 64 KB blocks:

| Offset in block | Size | Content |
|---|---|---|
| 0x0000 | 0xA0 | Boot record (above) |
| 0x0400 | 0x180 | Copy of the installed image's global header and section table (§2, §3) |
| 0x0800–0x0FFF | 32 × 0x40 | Event log, filled downwards from 0x0FC0. Each entry is a sequence number followed by a short text. Upgrades add entries naming the stage, the program doing the upgrade (application or recovery) and the new version. Section A also logs parameter-storage errors here. |
| 0x1000–0xFFFF | — | The application's parameter storage (Observed: section A's parameter writes switch blocks in the same way). Copied across unchanged when only the record changes. |

To update the record, either section chooses the active block, erases the
other one and copies 0x1000–0xFFFF across. It then writes the header copy
and the record, with the sequence number increased by one. The record's
first byte is written last, so a block whose `SIPURA *` is incomplete is
never taken as valid. Finally it marks that block as active. An install
does this twice, so both blocks end up holding the new record.

Flash layout implied so far (offsets in the 1 MB part):

| Offset | Content | Status |
|---|---|---|
| 0x00000, 0x10000 | Boot-record blocks | Observed |
| 0x20000 | Section B: its table entry, then its data | Observed (section A writes type 1 sections here) |
| up to 0xF0000 | Sections of type 2 and above (the application), written so they end at 0xF0000 | Observed |
| 0xF0000–0xF1FFF | Factory record (MAC, identifier, model, version) at 0xF0000. Section A preserves these 8 KB when it rewrites the last sector. | Observed |
| 0xF2000–0xFFFFF | Section C (entry at 0xF2000, data from 0xF2080) | Observed (section A writes type 0 sections so that they end at the top of flash; this section's 0x80 + 0xDF80 bytes then start at 0xF2000) |

Section A works these offsets out from the top of flash, which it finds
from the flash chip's ID. The values above are for this unit's 1 MB part.
The boot-record blocks and the boot selector's scan use fixed addresses.

Section A and section B place the application differently. Section A
rounds each section up to a whole 64 KB sector. Section B packs them to
4-byte boundaries, so an application installed by section B usually
starts partway through a sector (for 3.1.20(GW): flash 0x557EC, against
0x50000 from section A). The boot selector reaches it through the boot
record. Its fallback scan only looks at 64 KB boundaries, so it would not
find an application installed by section B (Observed arithmetic).

## 6. Building a valid image

To produce an image the device might accept, a writer would:

1. Build each packed section: header (§4.1), optional chunks (§4.2),
   stream records (§4.3), directory (§4.4), and whatever loader occupies
   the trailing area (§4.5). A loader that unpacks type 0x10 records
   through the boot section's service must follow §4.6; one that brings
   its own decompressor, or a section stored uncompressed, does not
   depend on it.
2. Fill each table entry: type, length, version, MD5 of the section data,
   field 0x04 (the type's tag), then the 16-bit byte sum at 0x7E, with
   bytes 0x7C–0x7D zero.
3. Write the global header and table, set the file length at 0x58,
   compute the MD5 over 0x000–0x1FF with the MD5 field zeroed, and store
   it at 0x30.
4. Write each section as its entry copy followed by its data, starting at
   0x200.

The bootloader checks each section's type, tag, entry byte sum, length
and MD5 (§5, Section C); whether anything checks the global header MD5
at boot is not known. It falls back to another valid section in flash if
the preferred one fails, but if none is valid it stops (Q7). **Do not flash an
experimental image until the recovery question (Q7) is answered**, and
keep a valid section of the other type in flash.

## 7. Upgrade transport

From the device's public administration documentation and the vendor
upgrader's behaviour:

- The device fetches an image when an administrator requests
  `http://<device>/admin/upgrade?<url>`, where `<url>` is a `tftp://` URL
  of the image. Third-party guides say `http://` URLs also work
  (unverified).
- The vendor's Windows upgrader runs its own TFTP server on the PC (from
  UDP port 69 upwards), then makes that HTTP request so the device pulls
  the image from it. The image it serves is byte-identical to the `.bin`
  file (**Verified**: the `.bin` is embedded unchanged in the upgrader).
- If an admin password is set, the request must be authenticated. The
  device's web server uses HTTP Digest authentication (Observed; confirm
  on the wire).

### 7.1 Recovery protocol (section B)

When section B is running (§5), it accepts upgrades over a UDP protocol
of its own. Observed from decompilation. The field layout is partly
traced; confirm on the wire.

- It listens on **UDP port 4810**, joined to multicast group
  **234.4.8.10**. Replies use port 4811. Whether they go to the multicast
  group or to the sender is not certain, because the decompiler hides the
  address argument.
- Messages are **160 bytes**. Byte 0 is the message type. The 32-bit word
  at offset 4 is the magic 0x5245434D (`RECM`). From offset 8 is a target
  identifier, NUL-terminated. It matches if it is `FFFFFFFFFFFF` (any
  unit) or the unit's own identifier from the factory record (§5,
  Section B). Messages of other lengths, or with the wrong magic or
  target, are ignored.
- **Type 0, discovery.** The unit answers with type 1. The reply holds
  the identifier, model name, MAC address, version strings and current
  IP address.
- **Type 2, upgrade.** The unit acknowledges, then fetches an image by
  TFTP. The file name comes from offset 0x20 of the request and a port
  number from offset 0x1C. The server is probably the requester
  (Hypothesis). It installs the image as in §5. If that fails it goes on
  listening. If it succeeds it stops and waits for a power cycle.

**Public description.** A third party published the same protocol as the
"S.O.S. firmware recovery protocol" of Cisco SPA IP phones (Dan Lukes,
Cisco Community knowledge base, 2013-09-15). The description is from
observation, tested on an SPA504G only, and claimed to apply to SPA ATAs
too. It agrees with everything above: the ports, group, 160-byte
requests, `RECM` magic, message types 0, 1 and 2, the wildcard
identifier, and a TFTP port at 0x1C and file name at 0x20 in the upgrade
request. It adds, all unconfirmed on the SPA3000:

- The client sends from port 4811 to 234.4.8.10:4810, and the unit
  replies to **the group** at port 4811, not to the sender.
- Replies are **100 bytes**. The identifier field is 20 bytes.
- Discovery reply: identifier at 0x08 (20 bytes), model at 0x1C (20),
  MAC at 0x30 (6, binary), recovery program version at 0x36 (20),
  hardware version at 0x4A (20), IPv4 address at 0x5E (4, binary), then
  2 bytes of unknown meaning at 0x62. Strings are NUL-terminated.
- The upgrade acknowledgement (type 2 again) echoes the request cut to
  100 bytes. Only the first 67 bytes of the file name are used.
- The unit sends its TFTP read request **to 234.4.8.10**, at the port
  from the request, not to the requester's unicast address. This
  replaces the Hypothesis above, pending a capture.
- The image is the vendor's normal upgrade file, and the version
  restrictions of a normal upgrade still apply (§5 step 5).
- Clients should wait 2.5 s for replies.

So a unit whose application is corrupt, but whose boot section and
section B are intact, can be recovered over Ethernet without opening it,
with a tool that speaks this protocol and serves the image by TFTP.

### 7.2 Installing an image from the application (section A)

Observed from decompiling section A. Section A downloads the image (§7)
and installs it with a routine that has the same structure as section
B's (§5), but differs in what it accepts and writes:

- **Accepted magics.** `SpA3000 FiRmWaRe` (§2). It also accepts
  `SpA1001 FiRmWaRe` when a hardware check passes (Hypothesis: on SPA1001
  hardware). An older magic is recognised but refused, and a separate
  manufacturing-image magic is refused with its own message.
- **Version checks.** The global MD5 must match (§2). The image version
  must not be below a per-model floor. It must also not be below a
  minimum held in a parameter, which lets a provider forbid downgrades. A
  further check refuses an image if, after installing it, the upgrade
  rule would trigger the same upgrade again.
- **All section types are written**, each only if it is present in the
  image:
  - **Type 0 (boot section):** compared byte for byte with flash and
    rewritten only if different. It goes at the top of flash. The
    factory record's 8 KB below it are saved first and written back after
    the sector is erased. If power fails during this step, the unit is
    probably left with no boot section (Hypothesis).
  - **Type 1 (recovery):** compared byte for byte and rewritten only if
    different, at flash 0x20000. The boot record's type 1 slot is pointed
    at it.
  - **Type 2 and above:** as in §5, but each section is rounded up to a
    64 KB sector (§5, flash layout).
- **Afterwards.** It writes the boot record to both blocks, increases an
  install counter held in a parameter and logs the new version. The
  caller logs the outcome and restarts.

So a stock image upgrade can replace section B and the boot section,
which section B itself never does.

**Restart.** Section A restarts in the same way as section B: it sets
flags, writes the small checksummed message block to RAM and resets the
chip. Its start-up message reports a "reboot reason" (Hypothesis: read
from that block). Nothing was found in section A that clears or spoils
the type 2 slot, marks the application invalid, or otherwise asks for
section B to run. The boot selector chooses only from flash contents
(§5), so the application cannot restart the unit into section B. The
search followed the string references, the install routine, the
boot-record writers and the restart path. A path through computed
addresses cannot be ruled out.

## 8. Processor

The chip is an **ESS Technology ES3890F** (ESS sold these as Video CD
processors with two 32-bit RISC cores). **Read** from the chip on a
`Q3000 REV F` board; see [`hardware.md`](../spa3000/hardware.md). The 0cpm project
found the same part on the related SPA901 and reports that the Linksys SPA
line uses a dual MIPS-X chip.

The main core's instruction set is **MIPS-X**, the Stanford RISC design
described in Paul Chow, *MIPS-X Instruction Set and Programmer's Manual*,
Stanford CSL-TR-86-289 (1986). It is not the commercial MIPS architecture.
[`processor-es3890.md`](processor-es3890.md) is the full description. In
summary:

- The instruction set is **Verified** against the code: the main
  application decodes completely, with two delay slots after every branch
  and jump. This replaces the earlier statistical Hypothesis.
- Instructions are 32-bit and big-endian. Code addresses are word
  addresses; data addresses are byte addresses.
- The ES3890 has byte and halfword loads and stores, which it encodes with
  the opcodes the manual gives to coprocessor and floating-point memory
  operations (Observed, strongly supported).
- The application calls through base registers r23 and r24, which the
  loader sets from the load directory (§4.4).

Ruled out: **Ubicom32** (including the IP3000 family). Its no-op encoding
is 0xC8000000, and decoding the image with Ubicom's published opcode table
gives an implausible mix of instructions. The "3k" suffix in section B's
version more likely refers to the SPA3000 than to the IP3000.

Section C has been decompiled (its boot selector, §5) and its loader stub
disassembled (§4.5, §4.6); section A has been decompiled in full. The
chip's second core has not been examined.

Toolchain: no mainstream compiler or assembler supports MIPS-X (none in
GNU binutils, GCC or LLVM as of 2026-10). Ghidra has no stock support
either; the [ghidra-mipsx](https://github.com/openmipsx/ghidra-mipsx)
processor extension, built from the public manual and the processor spec,
makes the decoded code readable. A compiler and assembler for new firmware
are being built from the instruction set manual in openmipsx's
[LLVM fork](https://github.com/openmipsx/llvm-project/tree/mipsx).

## 9. Open questions

| # | Question | Why it matters |
|---|---|---|
| Q1 | ~~What is table entry field 0x04?~~ Answered: a fixed tag per section type, which the bootloader checks (§3). | — |
| Q2 | Global header fields 0x50, 0x54 and 0x7C: confirm their meaning. | Needed to generate headers confidently. |
| Q3 | Meaning of the 8 bytes before each deflate stream. Most are zero; some equal the stream's load address. | Generator correctness. |
| Q4 | Directory type 0x10, word 3 (the other record types are resolved, see §4.4). Values (0x1472, 0xC46, 0xD1D0, 0xCAAB, 0x16EA, 0x35A8, 0x4EB8, 0x2B1) are not the 16-bit byte sum or either half of the CRC-32 of the uncompressed data. | Possibly a checksum the loader checks. |
| Q5 | Does the loader ignore the "Varies" words in non-stream directory records? | Generator correctness. |
| Q6 | Section C's embedded program is the boot selector: it checks and starts a type 2 or type 1 section from flash at 0x0C000000 (§5). The code at 0x0380 is the decompression service of §4.6. Sections A and B both write the boot records into flash sectors 0 and 1, with identical code; the +0x08 word is a sequence number increased on every write. Section A writes type 0 sections at flash 0xF2000, which confirms section C's placement. What section B does when started is in §5. All of these are answered. Still open: what runs before the boot selector; whether flash really repeats through the window (the boot selector's 0x0CFF2400 against flash offset 0xF2400); and the meaning of the record's +0x10 and +0x14 halfwords. | Boot chain and recovery. |
| Q7 | Mostly answered. Section B is a recovery image (§5, Section B). If the application fails its checks, the boot selector falls back to section B. Section B brings up the network and accepts a new application over the multicast recovery protocol (§7.1), or fetches one using the stored upgrade rules. It never overwrites itself or the boot section. So the unit can be recovered over Ethernet as long as sectors 0–1, section B (at flash 0x20000) and section C are intact. A stock upgrade from the application can rewrite both section B and the boot section, so those are not untouchable (§7.2). The application has no way to restart into section B: the boot selector chooses only from flash, and section A never spoils its own slot (§7.2). An application that passes the boot checks and then hangs is therefore started every time. If both A and B are invalid, nothing was found: the boot selector loops forever. Still open: whether the reset button or anything that runs before the boot selector can force section B; and confirming the protocol on the wire. | Avoiding a permanently unusable unit. |
| Q8 | Is the format the same in other firmware versions? | Only 3.1.20(GW) has been examined. |
| Q9 | What are the stream regions at 0x1000 and 0x10000 in section A, which do not decode cleanly as code? Also: what is bit 30 of the entry record, and is type 0x06 really r25? | Load map correctness; whether they are second-core code, data or something else. |
