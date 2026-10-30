# devices

Teardowns and specifications of the telephone adapters that openmipsx
targets: what is on each board, how their firmware images are laid out,
and how the ES3890F's MIPS-X core behaves. These are the documents the
replacement firmware is written from.

| Path | What |
|---|---|
| [`common/`](common/) | Specs shared across the Sipura and Linksys family: the [firmware image format](common/firmware-image-format.md) and the [ES3890 processor](common/processor-es3890.md) |
| [`spa3000/`](spa3000/) | Linksys (Sipura) SPA3000: [hardware](spa3000/hardware.md), [peripherals and start-up](spa3000/peripherals.md), [flash contents](spa3000/flash.md) and [photos](spa3000/photos/) |
| [`pap2t/`](pap2t/) | Linksys PAP2T: [hardware](pap2t/hardware.md) (case only so far) and [photos](pap2t/photos/) |
| [`clean-room.md`](clean-room.md) | How these documents are made, and the provenance log for every source examined |

The format spec was written from the SPA3000's 3.1.20(GW) image. Other
models use the same container with their own addresses and layouts; these
will be separated out as more devices are documented.

## Status

Drafts. Each fact is marked as read from the part, verified, observed or a
hypothesis; see the top of each document.

## Clean room

These specifications hold facts only: layouts, addresses, protocols and
hardware details needed for interoperability. They contain no vendor code,
disassembly or vendor text beyond short identifiers that must match, such
as magic numbers. The firmware images, decompilations and other material
they were derived from are not published. See
[`clean-room.md`](clean-room.md).

Linksys, Cisco and Sipura are trademarks of their owners. This project is
not affiliated with or endorsed by them.

## Licence

Licensed under the Apache License 2.0; see [`LICENSE`](LICENSE).

## Related

- [openmipsx/mipsx-sdk](https://github.com/openmipsx/mipsx-sdk): ABI,
  simulator, start-up code and the firmware image packer
- [openmipsx/ghidra-mipsx](https://github.com/openmipsx/ghidra-mipsx):
  Ghidra processor module and SPA image loader
- [Open-source stack plan](https://github.com/openmipsx/.github/blob/main/plan/open-source-stack.md):
  how the replacement firmware is to be assembled
