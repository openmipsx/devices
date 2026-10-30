# Linksys PAP2T

Status: unit arrived 2026-10-30. The closed case and the LEDs at
power-on are recorded in [`hardware.md`](hardware.md), with photos in
[`photos/`](photos/), and so are the stock web UI's versions: software
5.1.6(LS), hardware 0.1.5, not provider-locked. Case not yet opened.

What is known so far, from other sources:

- Its firmware (5.1.1(LS) and 5.1.6(LS)) uses the flash container and the
  SPA2002's layout and base-register values, and its section 0 is
  byte-identical to the SPA2002's (see the provenance log in
  [`../clean-room.md`](../clean-room.md)).
- MAME's PAP2 driver describes the related PAP2 board; see
  [`../spa3000/hardware.md`](../spa3000/hardware.md) §2.6.

The teardown will follow the SPA3000's: `hardware.md` and `photos/` here,
with anything shared with the SPA3000 going into [`../common/`](../common/).

## First pass

In this order, so that nothing is lost before it is recorded:

1. **Before powering it** (done 2026-10-30; the label has no hardware
   version or FCC ID): photograph the case, front and rear, and the
   underside label. Record the label's model, hardware version and FCC ID;
   leave the MAC address and serial number out of the documents.
2. **Powered, case closed, stock firmware**: note the LED behaviour at
   power-on, then read the web UI's Info page (software and hardware
   versions, customization or provider lock, certificate); both done
   2026-10-30, `hardware.md` §1 and §1.1. Do not upgrade, downgrade or
   factory-reset yet: the installed version and any provider settings
   are evidence.
3. **Case open, unpowered**: photograph the whole top side and the whole
   underside, then close-ups of the processor and Ethernet chip, flash,
   RAM, each telephony chip, the regulators, and every header, jumper
   footprint and test point. Read each chip marking.
4. **Unpowered, multimeter**: trace the headers and test points first
   (reset, chip selects, any serial console candidates), as on the
   SPA3000 (§1 of its `hardware.md`).

Photos go in `photos/` here, named `YYYYMMDD-PAP2T-<What>.jpg`, for
example `20261030-PAP2T-Board-Top.jpg`.
