# A0K's drawers and their emitter pairs

Area 20 (The Mouse Village) draws through two A0K drawers, both reached at `warp 20/4` (the minigame floor and its
cards); the other entries draw only through the resident pair. Each pair takes a0 = record list, a1 = OT base,
a2 = count.

## 0x80116B14 and 0x801163E0 / 0x80116708

While the byte `0x800BF871` is below 4 the drawer calls the resident wrapper `0x800803DC`; otherwise it zeroes the far
colour and calls `0x801163E0`/`0x80116708`. The pair is A0A's flagged copy (`0x801103F4`/`0x80110698`: 0 average,
1 farthest, anything else nearest, the GT4 dropping a sorted record) with one addition: once the bucket is known, each
corner's colour goes through DPCS with IR0 = bucket * 2 and is written back. The GT3 cues after its last UV store and
before the link, the GT4 after the link and before its last UV store.

## 0x80115F1C and 0x80114F1C / 0x8011562C

The drawer stores -rcos and -rsin of the halfword `0x800E7ED6` at `0x1F8000C0`/`0x1F8000C2`, zeroes the far colour and
calls `0x80114F1C`/`0x8011562C`. Per record:
- SOP's projection and screen cull (all X 320 or more, or all Y 240 or more, drops the record);
- the farthest SZ (SZ1-3 for GT3, SZ0-3 for GT4, staged at the 0x38 frame for every record) / 4, then A0G's near clamp
  (depth + 0x18F < 0x1B7 -> 0x28), the log compression and its range test; the flag byte picks nothing;
- the last UV store, then each corner's colour by DPCS with IR0 from that corner's model vertex and the focus point
  `0x1F800160`/`162`/`164`:
  - dx, dz = the vertex less the focus; along = (dx * C2 + dz * C0) >> 12, across = (dz * C2 - dx * C0) >> 12;
  - along * 2, across * 2 and again * 2 if negative;
  - x = (along * C2 - across * C0) >> 12, z = (along * C0 + across * C2) >> 12, hi/lo = the larger/smaller of |x|, |z|;
  - m = lo + lo / 2; d = hi - hi >> 5 - hi >> 7 + m >> 2 + m >> 6, then d - d >> 2;
  - IR0 = max(d + |vy - focus y - 0x100|, 0);
- the link (GT3 9 words, GT4 12).

The GT4 also stores a1 at its caller's argument area (`0x3c(sp)` in its 0x38 frame) before the count test.
