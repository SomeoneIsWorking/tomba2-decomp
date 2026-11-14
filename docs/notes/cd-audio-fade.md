# The CD audio fade

The XA channel block at `0x800BE1F8` carries the CD fade: target `0x800BE222` (+0x2A) and current `0x800BE224`
(+0x2C), s16. The mixer `FUN_80075824`, run every frame with that block, steps the current toward the target by
0x100 a frame (0x400 while the cut mode `0x1F800137` is 2), scales the channel volume by it and stores the volume
pair at +0x10/+0x12; with the audio state `0x1F80019A` other than 2 it skips the ramp.

The only writers of the two words in MAIN.EXE (`decomp_pipeline.py --refs`):
- `FUN_80075240`, the channel init: both 0x7FFF;
- `FUN_80075CEC(level)`: a negative level sets both to `-level` (a cut), otherwise only the target, clamped to
  0x7FFF (a ramp). Its callers are `FUN_80026470`, `FUN_800264BC`, `FUN_80026510`,
  `FUN_8002655C`, `FUN_80075070` and the mixer's own low-volume branch.

A clip start does not touch the fade. `FUN_8001D2A8(channel, start, end, flags)` stores the clip in
`0x801FE134/138/146/147`, the volume source bits in `0x800BE0E4` and spawns the stream task `FUN_8001CFC8` in slot
2, which drives the CD commands and never writes `0x800BE222/224`. Area music therefore starts at whatever level
the fade's callers left, with no fade-in of its own.
