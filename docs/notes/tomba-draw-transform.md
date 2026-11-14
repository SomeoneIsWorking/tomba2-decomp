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
| `G+0x14E` | pan about the world vertical, read only while `G+0x177` bit 0 is set (`FUN_800851F0`, a YXZ builder fed `(0, pan, 0)`) | area events: MAIN `0x80024660`, A04, A05, A06, A0I stores at `+0x14E` |
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
