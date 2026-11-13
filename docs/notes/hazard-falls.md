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
- A0E `TombaTick_floor` (`0x8010967C`): while `0x800BF850 == 0x80` holds Tomba at the ridden actor `*0x1F800260`'s X
  plus his radius. Entering the area plays the Water Pig Bag scene (Tomba in group 4 mode 0x22) unless
  `0x800BFA1A & 0x10`; with it set, spot sector 5 (10240, -3070, 11840) drops Tomba onto water at f18.
