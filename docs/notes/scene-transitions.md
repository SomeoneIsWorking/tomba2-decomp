# Scene transitions: the field's sub-scene machine and its variants

The field leaves an area through `fieldRun` case 6 (GAME `0x80106B98`): with the pending byte `0x800BF839` other
than 3 it sets sm[0x4A] = 5 and sm[0x4C] = the transition type, the byte `0x1F800236`. `FUN_80108A60` dispatches
sm[0x4C]: 0 and 9 return to the field at once, 1-4 `FUN_80107AFC` (the door corridor), 5 and 6 `FUN_80107D3C`,
7 `FUN_80107E20`, 8 `FUN_80107F3C`. Each ends by setting sm 2/1/1/0, and the area machine reloads the area in
`0x800BF870`.

## Who writes the type

Every store to `0x1F800236` in MAIN.EXE and the 28 overlays (`sb rt, 0x236(rs)` after `lui rs, 0x1F80`):
- 0: field and stage resets (MAIN `0x80026600`, `0x80050A48`, ...; GAME `0x801063AC`, `0x80106F88`, `0x80108770`);
- 1-4: doors and area exits in the overlays (A00 `0x8010EA6C`, A04 `0x8011E474` = 4, A04 `0x80120B38` = 3, ...);
- 5: Tomba's state `FUN_8005FB54` case 8, when its countdown at +0x40 ends;
- 6: `FUN_800527C8` substate 3, after it raised the pending byte in substate 1;
- 7: area objects that walk the camera to them first, then request the area in their byte +0xBF (A00
  `FUN_8013BAB0`: `0x800BF83A` = +0xBF << 4); also A01, A04, A06, A08 and A09;
- 8: the boss arenas' clear step (A0A-A0F). A0A `FUN_80116998` case 5 sets the pending byte 1, the destination
  per arena (area 10 → 6/7, 11 → 8/12, 12 → 1/5, 13 → 4/4, 14 → 0/11), an arena bit in `0x800BFE56`, the type 8
  and `0x1F800240` (s16) = the arena's area.

## The variants

- `FUN_80107D3C` (5, 6): 0 load (`FUN_80044BD4(0x800452C0, area, 0, 1)`), 1 `FUN_8003FB84` (`0x1F80008C` = 0xFF),
  2 done once the load flag `0x1F80019B` is set, else `FUN_8003EA88` (255 coloured strips, the wait picture).
- `FUN_80107E20` (7): 0 song stop `FUN_80074BF8(9)`, `FUN_8003E264` and the load, 1 `FUN_8003E264`, 2
  `FUN_8003E894` then done once loaded.
- `FUN_80107F3C` (8): 0 `FUN_80074BF8(9)`, `FUN_8003E264`, `FUN_80044BD4(0x80044F58, (s16 0x1F800240 + 0x1A) &
  0xFF, 0, 1)` (the arena's card picture, texture group area + 0x1A); 1 and 5 `FUN_8003E264`; 2 `FUN_8003E894` until
  loaded, then the song stop, `FUN_8001D71C(9)` and `FUN_8003FB94`; 3 `FUN_8003EBE0` until it returns true, then
  `FUN_8001CF2C`; 4 `FUN_80074BF8(8)`, `FUN_8003E264` and the destination load; 6 `FUN_8003E894` until loaded.

`FUN_80044BD4(entry, a, b, flag)` seeds sm[0x6E] = a and sm[0x6D] = b, clears the load flag and spawns `entry` on
task slot 1; flag 1 returns without waiting and without the RNG stamp. `FUN_80044F58` reads its set and mode from
sm[0x6E]/sm[0x6D], not from its argument registers; in mode 0 it ends by setting the load flag and closing its task.

## The effects

- `FUN_8003E264`: five 64-wide sprites, 15-bit pages of the displayed buffer (`0x1F800135` picks it), redrawn
  over the frame: the last picture held.
- `FUN_8003E894`: the held picture drawn through `FUN_8003E448` as a swirl, its angle `0x800A3B18` += 0x249.
- `FUN_8003FB94`: `0x1F80008C`/`0x90`/`0x94`/`0x98` = 0 and `0x800A3B0C` = 2.
- `FUN_8003E030`: while `0x800A3B0C` is 2, the held picture wobbles by `0x1F800094`.
- `FUN_8003EBE0`, the boss-clear card: `0x1F800090` counts its frames (u32); `0x1F80008C` = count - 0x1E clamped
  to [0, 0xFF] cross-fades two 8-bit pictures (pages x 0x2C0 and 0x240, y 256, three 128-wide sprites each,
  additive). It returns true on the frame it reads a count above 0x1A3, and then clears `0x800A3B0C`. So the card
  shows for 420 frames, with no pad test.

## The opening

New Game runs SOP's narration, `FUN_80109450` (sm[0x50]): 0 load, 1 fade-in (31 frames, sm[0x6C] counts down), 2
the crawl until the pending byte is set or Start is pressed, 3 fade-out (31 frames), 4 reset to the next intro
step. The scripted opening in area 0 runs with load mode `0x800BF89C` = 2 at `fieldRun` case 9 (a dialog
cutscene); Start there sets case 10, a 31-frame fade from sm[0x6E] = 0x1F, then cases 7, 8 and 6 load area 0 in
ordinary play (load mode 4).
Start skips neither fade in the retail code: the narration's fade-in ignores the pad, and case 10 always runs
its 31 steps.

## Not a transition: FUN_80127798's node[5] = 3

The area-0 object `FUN_80127798` (a node at `0x800F1F58` in area 0) leaves state 1 when `0x1F800207` is 29-31 or
event flag 1 (`0x800BF9B5`) is 1. Its node[5] = 3 phase is an in-area event: `FUN_80042354(1, 1)` plays a jingle
and sets `0x1F800137`, `FUN_80054D14(G, 0x71, 8)` sets Tomba's action 0x71, the camera flies to (0x49B3, 0xFA94,
0x204C), flag 1 goes to 3 then 4 and `0x1F800207` = 0x22. It writes neither the pending byte nor `0x1F800236`.

## The memory-card pages

The game-over screen (sm[0x4A] = 2, GAME `0x80106478`) runs the card machine `0x800BF84A` from its state 7 (load)
and 8 (save): case 0 loads CRD, then `FUN_8007BE18` runs CRD's browser on sm[0x50]. CRD `0x8018FBF0` calls
`0x8018A1E8` on a per-port record: phase (+0x40) 0 calls `_card_info` (A0:0xAB) on port (+0x3D) << 4, sets phase 1,
+0x43 = 0 and +0x41 = 0xFF; phase 1 tests the four card events at `0x800BF498` + 0x1C, 0x20, 0x24 and 0x28 through
`FUN_80080840` and returns 1-4 for the one that fired, else counts +0x43 and returns 2 (timeout) at 0xFF. The
browser shows "Checking MEMORY CARD..." until the check settles and keeps calling `_card_info` on both ports every
few frames while it is up.
