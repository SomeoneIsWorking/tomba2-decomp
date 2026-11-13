# Hazard falls: water, death floors and the respawn

Tomba's block G is `0x800E7E80`. Vitality is the halfword `G+0x16E` (`0x800E7FEE`), its shown target `G+0x170`,
its maximum the byte `0x800BF87D`. `Tomba_damage` (`0x80022C0C`, G, n) subtracts n (doubled while grown,
`G+0x17E & 0x8000`) and at 0 latches `0x800BF80D = 0xFF`, the game-over request. The spawn `0x800783DC` sets
vitality to the maximum, or to `0x1F800194` when the carry flag `0x800BF880` is set (a respawn after a fall).

## Floor attribute

`Tomba_groundCheck` (`0x8005444C`) copies the attribute of the floor line under Tomba into `G+0x6A`. Bit 0x8000
marks a hazard floor; `& 0xF00` is its type: 0x100 and 0x200 water, 0x700 death. Bits 12..14 are the water
kind the area tick branches on. Type 0x500 is snow, only in areas 4 (`0x15xx`) and 5 (`0x25xx`): `Tomba_runSpeed`
(`0x80055FBC`) cuts the start speed to an eighth and the brake to half there while the suit word `G+0x17E & 0x30`
(no suit 0x10, Bird Clothes 0x20) is set. `0x800532A0` sends the same suits on type 0x500 with bits 12..15 = 4 to
mode 0x35, but no area's grid has such a line. Floor lines live in the area's collision grid: the layer table at
`*0x1F8001C8`: entry 0 is the sector count and sector s (1-based) is at the u16 halfword offset in entry s
(`FUN_800490E4` scans them; `FUN_80048ECC` loads one). Each sector record `{x0, x1, z0, z1, ?, links, cells, cellLinks,
lines, verts}` (offsets in halfwords from the record) followed by 4-byte rows `{first z cell, first cell}`, one per 64-unit X step from min(x0, x1), cells 64 units in Z
from min(z0, z1) (`FUN_80047CBC`);
a cell is `{tag, first line, line count, ?}`, a line `{attribute, y, slope, vertex}`.

Hazard floors at each area's entry 0 (sector, x, z, floor y):

| area | type | where |
|---|---|---|
| 0 | water (0xC101, 0xD101) | sector 22: x 14272..14912, z 7744..11072; sector 34: x 16320..19904, z 7168..10752; y -128 |
| 1 | death (0x8701) | sectors 17..24, y -2300; sector 24: x 7488..9600, z 14400 |
| 4 | death | sectors 2..4, y -6016 |
| 6 | water (0xD101) | sectors 1..3, 7, y -1280 |
| 8 | water (0xF101, 0xF201, 0x8101, 0x8201) | sectors 37, 42 |
| 11 | death | sector 1, y -6208 |
| 12 | death | sectors 9..16, y -800 |
| 13 | water (0xD101) | sectors 1, 13, y -1024 |
| 14 | water (0xB101) | sectors 16..30, y -2752 |
| 15 | death | sector 1, y -5312 |

## Entering the hazard state

The area's own Tomba tick decides. A00's `TombaTick_hazardFloor` (`0x8010E408`), on a water floor with
`G+4 < 2` and no swimming suit (`G+0x17E & 0x100`), writes `G+4 = 2`, `G+6 = G+7 = 0` and by water kind: 2..3
`G+5 = 3` (thrown back to a point from `0x80146EF8` or the area's entry table), 4..5 `G+0 = 3, G+5 = 4`. A01's
`TombaTick_deathFloor` (`0x8010C7BC`), on a death floor, writes `G+0 = 3, G+4 = 2, G+5 = 5, G+6 = G+7 = 0`. The
hit knockback `Tomba_hazardKnockback` (`0x80066878`, substate 1) lands on water as `(4, 1)` and on a death floor
as `(5, 1)`. Every site writes the same `G+6` for both.

## The substates

`Tomba_hazardDispatch` (`0x80067CA4`) runs on `G+5`:

- 4 `Tomba_waterFall` (`0x8006762C`): case 0 `Tomba_waterFallDamage` (one point, sound 0x23 or 0x24 at zero),
  then the flounder animation, the area's splash (areas 0, 6, 8, 13, 14), a 30-frame hold, the fade,
  `Tomba_hazardRespawnEntry`, and the bob `Tomba_waterFallBob` until the reload.
- 5 `Tomba_deathFloorFall` (`0x800678E8`): case 0 zeroes vitality and its target, sound 0x24, then the fall,
  `Tomba_hazardRespawnEntry(G, 0)` and the fade; at zero vitality the reload is the game over.
- 3 `Tomba_hazardThrowBack` (`0x80066FA8`), 7 `Tomba_longFall` (`0x80067A40`).

`Tomba_hazardRespawnEntry` (`0x800662D8`) sets the entry `0x800BF871` the reload spawns at: area 6 and area 0
by their own handlers, area 8 by the swimming state, otherwise `g_HazardRespawnEntry[area]` (`0x800A47E8`) when
it is not 0xFF (0xFF keeps the entry Tomba arrived by).

Measured (psx-tomba, spot warp over area 0 water, sector 22): f+4 `G+4..7 = 2, 4, 0, 0`, vitality 4 → 3 at the
same frame, substate steps 6, 7, 8, 9, 10, respawn at area 0 entry 8 66 frames after the fall.

Measured over the area 1 death floor (sector 24, dropped from y -2600 onto y -2300): `G+4..7 = 2, 5, 0, 0`,
vitality 4 → 0 in the same frame, the respawn entry 0 → 1 (`g_HazardRespawnEntry[1]`), then `0x800BF80D = 3`. The
field run (`0x80106F48`) reads vitality 0 and steps the scene machine (`*0x1F800138`) to `+0x4A = 2, +0x4C = 0`, the
game-over sub-mode. With vitality left after a fall it instead sets the carry `0x800BF880` and `0x1F800194` and
reloads the area at the picked entry.

Entering substate 4 where the death floor enters 5 (same `G+6`) gives the water fall in area 1: vitality 4 → 3,
substates 6..9, respawn at area 1 entry 1 with 3 vitality; at 1 vitality it reaches 0 and the same game-over
sub-mode.

Area 4's death floor (sector 2, from entry 1) measures the same: substate 5 and vitality 0 retail, the water fall
and respawn at area 4 entry 1 (`g_HazardRespawnEntry[4]`) with the redirect.

## Area 5's cold field

Past Z 0x1928 in area 5 the cold reaches Tomba unless the suit word carries 0x640 (Squirrel Clothes, Clear Fruit, Evil
Flame Pig Robe). A05's Tomba tick `TombaTick_coldField` (`0x8010A854`), with no event flag `0x800BFE56 & 0x20` and
`G+0x177 & 2` clear, starts hazard substate 6 (`G+4 = 2, G+5 = 6`, `G+7 = 1` with Cold Powder `0x800BFAD5` or when
`0x80022D08(G, 0, 1, 1)` is 0). Substate 6 takes a point and hands over to mode 0x1A, which sets `G+0x177 & 2` while
Tomba is frozen; the crust actor `IceCrust_step` (`0x8012E454`) grows up to three layers on him (`IceCrust_addLayer`
`0x8012E340`, scale at layer+0x3A) and shatters them when he leaves the field or gains 0x640. Measured walking right
from area 5 entry 0 with no suit: a point lost at f227, f347, f443 and the game-over substate at f547; with the
Squirrel worn he walks on to Z 0x22C5.

## Other areas' floor ticks

Each test water against the swim bit `G+0x17E & 0x100` the same way: with it, mode 0x1D (or 0x32 out of mode 6),
`G+4 = 1`; without it, `G+0 = 3` and substate 4.

- A06 `TombaTick_floor` (`0x8010B334`): swims only once the area flag `0x800BFE56 & 0x40` is set; type 0x900 throws
  Tomba back to the near or far point at `0x801422A0` by Z against 15000.
- A08 `TombaTick_floor` (`0x8010B328`): after `FUN_8010B1E8`; skipped while `0x800BF873` or `0x1F800137` is set; a
  hang (mode 0x2A) or a held mode 3 sets `G+0x181` instead of starting the swim, and `FUN_8012DE84(G, y)` picks the
  surface height `0x1F80019E`. Measured from entry 1, spot sector 1 (8064, -3900, 7808): no suit falls in at f18 with
  vitality 4 → 3; Swimming Pig worn swims at f14.
- A08 `Waterfall_tombaContact` (`0x8010D334`, a1 the waterfall, actor kind 0x5A, the contact dispatcher
  `0x8010E7C0`'s table at `0x80109120`): by the actor's shape byte +3 (1 cylinder, 2 and 4 the box test
  `0x80022F04`, 3 `0x8001F830`) a swimmer (`G+0x17E & 0x100`) is lifted up the falling water to its top, up to 6 or
  0x10 a frame, and anyone else is pushed out along the contact angle `0x1F80009C` into hazard substate 2 (the
  knockback, `G+0x172 = 0x78`). The shape 1 fall over the pool at sector 6 (15221, -6240, 4160) is loaded from
  entry 9: no suit is knocked out at f66, Swimming Pig worn stays in the pool in mode 0x10.
- A0E `TombaTick_floor` (`0x8010967C`): while `0x800BF850 == 0x80` holds Tomba at the ridden actor `*0x1F800260`'s X
  plus his radius. Entering the area plays the Water Pig Bag scene (Tomba in group 4 mode 0x22) unless
  `0x800BFA1A & 0x10`; with it set, spot sector 5 (10240, -3070, 11840) drops Tomba onto water at f18.

## Area 4's ice monster

A04's Tomba contact loop `TombaContacts_loop` (`0x8010F1CC`) walks `*0x1F80013C` (count `0x1F800144`, left
`0x1F800182`) and dispatches on the actor kind through the table at `0x801092E8` (kind - 8). Kind 0x28 is the ice
monster's touch: on `0x8001F830` contact Tomba faces the contact angle and, when `0x80022D08(G, 0, 1, 1)` takes,
`G+0x177 & 2` is clear and the suit word has 0x30 (no suit or the Bird), he freezes (mode 0 substate 4,
`G+0 |= 8`, then `G+0x177 = 2`). Heavier suits are knocked back (mode 0x34). Reached from area 4 entry 1, spot
sector 23 (3392, -10500, 10600): no suit freezes at f66 with vitality 4 → 3. `0x80110A30` repeats case 0x28 and has no
reference.

A04's Tomba tick `TombaTick_snowField` (`0x8010B024`): at entry 7 a death-type floor (0x700) leaves the area
(`0x1F800137 = 2`, `0x800BF80F = 2`, `0x800BF839 = 3`, `0x8005082C(0, 0, 0)`, `0x800BF83A` 0x408 for floor bits
12..15 == 1, else 0x402); elsewhere a hazard death floor is substate 5. Off hazard floors, with `G+0x16A != 6` (6 is
the snow-slope slide of `0x800532A0`), a winged suit (`G+0x17E & 0x60`), `G+0x78 == 0` and collision sector
`G+0x2A >= 0x27`, it sets `0x800BF819 = 4`: `Engine_frameStartTick` (`0x80059D28`) spends it one per frame,
masking `0x800FCF54` and `0x800E7E68` to 12 bits. Sectors 0x27..0x2A are slope (0x3501), so the first reach is
sector 43 (x 4736, z 7424..8768, floor y -7043). `G+0x2A` is the collision sector the grid walk last stored.
`0x8010F75C`'s kind 1 test needs `G+0x16A == 8`, which `0x800532A0` sets only on a 0x500 floor with nibble 4: no
area has one.

## Area 1's steam vents and A0B's updraft

A01's contact loop `0x8010F40C` dispatches on the actor kind through the table at `0x80109190`; kind 0x12 is a steam
vent, `SteamVent_tombaContact` (`0x8010D90C`). The vents are spawned by `0x80133574` (shape = index, update
`0x80132FD0`) at the positions of the s16 xyz table `0x80138AF0`, gated per index by the bits of `0x800BF9F4`; vent
0 stands at (10496, -7488, 9040). While Tomba is tested the vent's extents +0x84/+0x86 grow by 200, or by 400 with
`G+0x17E & 0x40` (the Squirrel), unless he holds something (`G+0x78`). On a `0x80022F04` overlap the depth
`d = G84 + vent84 + 300 - (vent32 - G32)` kicks him: `0x800BF848` = the box test's floor `0x1F800090` (or 1 and
sound 0x25), x speed `G+0x44` loses a quarter, and `G+0x4A -= 2d`, clamped to 0x280 and to -0x3E00 (d < 500) or
-0x2800; a side hit (`0x1F800094`) instead zeroes `G+0x4A`/`G+0x50` and pushes `G+0x30` back by `d << 12`. Poses
0x13..0x15, 0xCA, 0xE5, 0xE6 switch to pose 0x12 unless the floor kind is set and `G+0x145 == 1` (then
`0x800BF848 = 4`). Shape 8 only pushes Tomba down by the overlap. Standing on vent 0 (spot sector 2) Tomba floats
at y -8828..-8696 with no suit and about 200 higher with the Squirrel.

A0B is the Evil Ice Pig battle. Its kind 0x55 actor, spawned by `0x80117FD8` 0x12C in front of its parent's facing, runs the
same lift in `IcePigBattle_updraftContact` (`0x8010A93C`, from the contact loops' kind 0x55 cases at `0x8010B2B8` and
`0x8010B63C`), always growing the extents and leaving them grown when nothing touches (it zeroes the touch count
+0x64). After 0x3D frames of touches, with `G+0x16E` set, `0x1F800137`, `G+0x177 & 2` and `G+0x78` clear and no
Evil Flame Pig Robe (`G+0x17E & 0x400`), Tomba freezes: hazard substate 6 (`G+0 |= 0xC`, phase 1 if Cold Powder
`0x800BFAD5` is held or `0x80022D08(G, 0, 1, 1)` fails), the actor's +0 = 2. Not yet reached in a run: warp 11/1
opens with the battle's dialogue (circle advances it) and no kind 0x55 touch came before Tomba fell. The frozen states `0x8010B73C` and
`0x8010B938` set `G+0x177 |= 2`, which the frost actor `0x80114608` shows until the robe is worn.

A01's Tomba tick `PipeArea_tombaTick` (`0x8010C7BC`): at entries 15 and up it does nothing. While `0x800BFE56 & 2`
and `0x800BF839 == 0`, a death floor takes the exit `0x8010C6EC` picks by `G+0x2A` (only when `G+0x164 == 1`, else
the fall) and so does any climb above y -0x8FB. Otherwise a death floor is substate 5, and when `0x800BF8BE != 0xFF`
a fluttering Bird (`G+0x17E & 0x20`, `G+0x179` set by the flutter `0x80056F3C`) in x 0x28B5..0x2D69 below y -0x1A8F
sinks 0x20 a frame.

`Tomba_climbStep` (`0x80062A30`, a0 Tomba, a1 the climbable, a2 the mode bits; called from A00, A01, A04, A0A, A0C,
A0F) moves `G+0x156`, his place along the climbable. Mode 3 lets go once `Tomba_groundCheck` says he stands. The
step is s8 `0x800A4700[mode & 1]` (+3 with `G+0x165`), or for mode bit 2 `G+0x4A` growing by 4 up to 0x1E. Down
(bit 0 clear) clears the floor attribute, follows a moving climbable (kind 0x17, part +0x47 != 2) by its velocity
`+0x4A >> 8` (bit 1: a 0x30 drop), undoes the step if `0x800541F4(G, 1)` hits and stops at `-climbable84`. Up past
the top (`+0x86 - +0x84 + G+0x68`, +0x50 when grown) on bit 7 or a moving climbable he jumps off: `G+0x156 += 0x2C`,
`0x80056D44(G, 0)`, and off a kind 0x17 he takes its velocity, rises 0x2C more and, in a winged suit
(`G+0x17E & 0x60`), another `(velocity & 0x1F00) >> 5`. Otherwise the place is capped at the height; then, without
bit 6, standing lets go (mode 0x85 turns him round, mode 0x35).
