# A00 pig basket — FUN_80117228

A type-05 object whose handler is FUN_80117228 (A00). Byte +3 is its variant and +4 its state; quest byte
`0x800BF9E5` records how far its story has gone.

## Variant 1, the basket a pig carries

- State 0: collision box (+0x80..+0x86 = 0x28, 0x50, 0x4B, 0x73). With quest byte 4 already set (a reload after the
  pig was disturbed), +0x5E = 1, so it starts at rest.
- State 1, FUN_80116E3C, switches on +0x5E:
  - 0, FUN_801168C4: rides its carrier (`+0x10` → its `+0xE4` position). Once the carrier's state (+4) passes 1,
    the pig is disturbed: quest byte = 4, scene camera flags 0x800BF841/0x800BF809 set. The basket then flies 0x5F
    frames toward (0x39E0, 0xF2A4, 0x20E0) under gravity and drops for 0xF more. After that, +0x5E = 1 and the
    camera flags clear.
  - 1, FUN_80116CC4: placed at (0x39E0, 0xF286, 0x20E0) = (14816, -3450, 8416). While FUN_8007778C (bounds cull)
    passes, it falls under gravity through FUN_8004766C until it lands (+0x29). A hard floor stops it at
    y = 0xFAA1 (-1375); +6 = 2 is at rest.
  - 2: spins (+0x54 = 0x800); quest byte 7 ends it (state 3).
- State 2: with scene event 6 armed (FUN_80040C00), it arms event 7, plays sound 0x29, sets +0x5E = 2 and quest
  byte 6.

The resting place, (14816, -1375, 8416) at the foot of the cliff beside the door slot, is retail's own. A basket
seen there after the pig was disturbed is the game's design.

## Variant 0

FUN_80116F64 follows quest byte `0x800BF9E0` (values 2, 7, 10, 0x11, 0x14, 0x15) as a scripted prop.
