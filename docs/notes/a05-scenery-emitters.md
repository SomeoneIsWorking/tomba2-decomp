# A05's scenery walker and its emitter pairs

A05's scenery walker draws each part through one of three GT3/GT4 pairs (a0 = record list, a1 = OT base, a2 = count):
- `0x8013544C`/`0x8013590C`, the lit pair (the call sites at `0x80136158` and `0x80136368`);
- at `0x801361A0`: while bit 0x20 of the halfword at the walker's record + 0x5E6 is set, or the byte `0x800E7EAA` is 12
  or more, `0x8013484C`/`0x80134B74`; otherwise (`0x8013623C`) `0x8013AC90`/`0x8013AF0C`, then `0x801351DC`.

## 0x8013AC90 / 0x8013AF0C, the fogged pair

A04's fogged pair `0x8013F418`/`0x8013F6E4` instruction for instruction, except for two things. IR0 is
clamp(AVSZ * 4 - 0x40, 0, 0x1000) where A04 subtracts 0x200. A05 also has no hide bits: A04 hides a record without
flag bit 2 by bit 3 while `0x1F800207` is 15-17 and by bit 5 while `0x1F80009C` is set. The screen cull is the shared
one: each pair drops a record only when all its X are 320 or more, or all its Y are 240 or more.

## 0x8013484C / 0x80134B74

A0G's near-clamped body (`0x8010BC40`/`0x8010BF28`) with these differences:
- the frame is 0x10 bytes, and SZ1-3 (GT3) or SZ0-3 (GT4) go to sp for every record;
- the mode is the flag byte's low 2 bits: 0 is AVSZ, 2 is the nearest SZ / 4, and 1 and 3 are the farthest;
- a nearest record stores its depth and skips its link and its packet advance, so it is never drawn;
- after the log compression and its range test, flag bit 2 subtracts 10 from the bucket before the sign test.
The near clamp (depth + 0x18F < 0x1B7 → 0x28) is A0G's.
