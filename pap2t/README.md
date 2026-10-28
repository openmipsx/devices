# Linksys PAP2T

Not started: the unit has not arrived yet.

What is known so far, from other sources:

- Its firmware (5.1.1(LS) and 5.1.6(LS)) uses the flash container and the
  SPA2002's layout and base-register values, and its section 0 is
  byte-identical to the SPA2002's (see the provenance log in
  [`../clean-room.md`](../clean-room.md)).
- MAME's PAP2 driver describes the related PAP2 board; see
  [`../spa3000/hardware.md`](../spa3000/hardware.md) §2.6.

The teardown will follow the SPA3000's: `hardware.md` and `photos/` here,
with anything shared with the SPA3000 going into [`../common/`](../common/).
