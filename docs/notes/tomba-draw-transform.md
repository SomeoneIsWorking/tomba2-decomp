# Tomba's draw transform

`Tomba_draw` (`0x8003B588`) calls `FUN_800597AC(G)` with Tomba's block G = `0x800E7E80`, which rebuilds every one of
his render records' world transforms from G inside the field draw. Everything it reads is therefore an input of the
picture, not of the logic.

## Inputs of FUN_800597AC

| field | what | written by |
|---|---|---|
| `G+0x2E/32/36` | position, the high halves of the 16.16 words at `+0x2C/30/34` | the update |
| `G+0x54/56/58` | body angles, built by `RotMatrix` (`0x80085480`) | the update |
| `G+0x88/8A/8C` | root offset, s16 x3, turned by the body matrix (`FUN_80084470`) and added to the position | `FUN_80076904` loads it at a keyframe whose flags have bit 7 (12-bit triple); `FUN_80075F0C` adds the per-tick delta `G+0x90/92/94` |
| `G+0xB8/BA/BC` | scale, `FUN_800517BC` into the block at `0x1F800000` | constant 0x1000 in walk and jump |
| `G+0x14E` | pan about the world vertical, a 12-bit angle read only while `G+0x177` bit 0 is set (`FUN_800851F0`, a YXZ builder fed `(0, pan, 0)`); 0 otherwise | the idle sequence `FUN_8005A39C` (below); outside it `+0x14C/14E/150` hold other state, e.g. `FUN_80024548`'s probe point |
| `G+0x164 == 5` | attach: the result is multiplied onto the matrix of the node at `G+0x10` (`+0x18`) | |
| `G+0x145`, `G+0x146 & 3` | a second body matrix without roll at `0x1F800060`, used by the first record with a negative parent | |
| record `+6` | parent index; negative means G's matrix | |
| record `+8/A/C` | part angles (`RotMatrix`) | `FUN_80076904` at a keyframe, `FUN_80075F0C` adds `+0x10/12/14` each tick |
| record `+0x38/3A/3C` | part offset | `FUN_80076904` only for keyframes with flags bit 6 (stored `<< 3`) |

The part angles are a pose computed each tick: `FUN_80076904` writes a keyframe's absolute 12-bit angles and
`FUN_80075F0C` integrates the delta between keyframes without masking, so the halfwords carry whole turns above 12
bits. An animation switch (the cursor store at `0x80054D14`, then `FUN_80076904`) writes the new animation's first frame the same
way. Two consecutive ticks' angles are both absolute poses, so their short-way blend is the pose between them, also
across a keyframe or animation switch.

## Observed (village, running then jumping)

Store watch on `0x800E7F08..0x800E7F0E` while Tomba runs: `G+0x8A` reads -0x1E, -0x28, -0x1C, -0x10, -4, +0xA, 0,
-0xA ... on consecutive ticks, reloaded from the keyframe every few ticks: the run's bob. `G+0x88` and `G+0x8C` stay 0. Over the same
run and a jump, record 2's part offsets `+0x38..+0x3C` are never written, and `G+0xB8` reads 0x1000, `G+0x177` 0.

## Idle sequence: pan and camera shake

`FUN_8005A970` is Tomba's stand state. On entry it zeroes the pan `G+0x14E`; while he stands still it runs
`FUN_8005A39C`, which sets `G+0x177` bit 0 every tick (`FUN_80059D28` clears the byte at the start of each tick) and
steps through the table `0x800A4610`, four bytes an entry: animation, loop count (counted in `G+0x14C`), pan index,
flags. Each tick it turns the pan toward `(index + 2 * facing) * 0x400` by at most 0x80 (`FUN_800776F8`), so a turn to
face the camera takes 8 ticks. Entries 4, 8-11, 14-17, 21-23 turn him; entries 9, 11 and 22 (animation 0xBF, flags 4)
write the camera shake state `0x800E807E`: 6 (height pulse, capture) when the animation reaches frame 0xE, then 7
each tick of the stomp; entry 17 (flags 3) writes 8 and 9, the small pulse.

The shake reaches the picture a tick late. `CutsceneCamera_shakeTail` writes the eye height `0x1F8000D6` as
`anchor - 32 + rand(0..63)` after the mode's `lookAt` built this tick's view, and returns to state 0; the next tick
`mainFollow` skips `heading` (state 7 is set again before the camera runs), so its `lookAt` builds the view from the
jittered eye. A drawn view therefore carries the previous tick's jitter, and no view is kept without it.

What a drawn view is built from. `FUN_8006D02C(a0 = camera object 0x800E8008)` is the look-at: it reads the whole
units of the eye (the high halves of the 16.16 words at `0x1F8000D0/D4/D8`) and of the target (`0x1F8000DC/E0/E4`),
writes pitch and yaw to `0x1F8000F0/F2`, the distances to `cam+0x5C/0x60`, the view rotation to `0x1F8000F8` and
its translation `R * -eye` to `0x1F80010C`, and copies the matrix to `0x1F800118`. `mainFollow` (`FUN_8006E0F0`)
then subtracts `cam+0x28 + cam+0x34` from the translation Z `0x1F800114`; nothing else moves the view before the
field draw in the village. The jitter arms write `eye = anchor + offset`: the height arms (states 5, 6/7, 8/9) only
Y, offset `-32 + (rand & 0x3F)` or `-16 + (rand & 0x1F)` from `cam+0x88`; the axis arm (state 2) X and Z by
`-16 + (rand & 0x1F)` from `cam+0x86/0x8A` and Y by `-8 + (rand & 0xF)` from `cam+0x88`. The view the next tick
draws is the look-at of the jittered eye exactly when no step moved the eye in between (`heading` skipped), so
its unshaken view is the same look-at with the offset taken off the eye.

Observed (village `warp 0/0`, standing): the pan runs 0, 0x80 ... 0x400 and back; a stomp shakes the camera for 12
ticks (entries 9 and 11) and up to 117 ticks (entry 22), the view's translation Y moving by up to 0x65 a tick.
