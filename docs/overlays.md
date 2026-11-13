# Images and overlays

The disc boots `SCUS_944.54`, which loads `MAIN.EXE` over the same range; the game then loads its code overlays
from `BIN/` into three fixed slots. Every overlay is a raw image (no header) loaded at its slot address. The image
set and slots come from psx-tomba's loader and module-load owners (`game/core/overrides/native_override_catalog.h`,
`game/core/engine/engine_frame_ticks.cpp`, `game/scene/demo.cpp`).

| slot | address | images | loaded by |
|---|---|---|---|
| STAGE | `0x80106228` | START, DEMO, GAME | `FUN_800450BC` → `FUN_8001DB8C(0x80106228, lba, size)`; START first from task 0, then the stage swap |
| MODE | `0x80108F9C` | SOP, A00..A0L (area code, A00..A0L in area-index order) | the area load |
| AREA | `0x8018A000` | OPN, CRD | `FUN_80045558(idx)`: 0 OPN, 1 CRD; shares the slot with area and texture staging data |

## Executables

| image | load | text | rodata | code | data from | SHA-1 |
|---|---|---|---|---|---|---|
| `SCUS_944.54` | `0x80010000` | `0x28800` | `0x80010000` | `0x80010D54` | `0x8001FB5C` | `9f68ffd6e4dfa9f9ff6ba6818760ee4c625be421` |
| `MAIN.EXE` | `0x80010000` | `0xAE800` | `0x80010000` | `0x8001CAC0` | `0x8009D090` | `f41c4065788e8c838cbd8b22f0fff6d4bc7de49a` |

Both run with `gp = 0x800BE0D4` (splat's `create_config` finds it in `MAIN.EXE`; the headers carry 0). MAIN's bss
starts at `0x800BE800` and runs up to the STAGE slot; its size is not recorded yet.

## Overlays

File offsets of each split; the first data run is the image's table (its first word is a small per-file number:
START 6, DEMO 7, GAME 8, A00..A0L 0x0A..0x1F, SOP 0x20, OPN 0x22, CRD 0x23).

| image | slot | size | code from | trailing data from | SHA-1 |
|---|---|---|---|---|---|
| `BIN/START.BIN` | 0x80106228 | 0x670 | 0x274 | - | `5eb7efe5e5aff8a567c30367a7031a8243d26cca` |
| `BIN/DEMO.BIN` | 0x80106228 | 0x14FC | 0xBC | 0x14DC | `e3988a70ed218b392a245301e4b20f34343cb7d1` |
| `BIN/GAME.BIN` | 0x80106228 | 0x2D74 | 0x154 | 0x2D38 | `2e1181fe670f27581f95e7fe4d658361e87f2882` |
| `BIN/SOP.BIN` | 0x80108F9C | 0x44FC | 0x1C8 | 0x39C0 | `31a0fbd54612836912ff7e991334f063fe0df7f2` |
| `BIN/OPN.BIN` | 0x8018A000 | 0x351C | 0x2DC | 0x3378 | `0462c26910a5285eb231cf0d95499f034ae136f4` |
| `BIN/CRD.BIN` | 0x8018A000 | 0x61E4 | 0x1E8 | 0x5EC4 | `1cfb06cdda46ea7d48fc81790dc437543e62b032` |
| `BIN/A00.BIN` | 0x80108F9C | 0x459A8 | 0x13A0 | 0x3DA20 | `c34a290816d05433ca402fbaecde2bf424b4e664` |
| `BIN/A01.BIN` | 0x80108F9C | 0x316DC | 0xFE0 | 0x2B724 | `5bdd26f4e39009786d682221afb80f87e2bb1a47` |
| `BIN/A02.BIN` | 0x80108F9C | 0x3112C | 0x158C | 0x1F428 | `c0232987d14263d5e61b763dd1322e27b87429cf` |
| `BIN/A03.BIN` | 0x80108F9C | 0x1326C | 0x80 | 0x89C0 | `8bd4bcc23f1c41b778e1c5733269d9e45701c86a` |
| `BIN/A04.BIN` | 0x80108F9C | 0x3D3A4 | 0x1578 | 0x36AD8 | `7c940d9434cb724c363a098312c2c84ba83511e5` |
| `BIN/A05.BIN` | 0x80108F9C | 0x392B4 | 0x1398 | 0x322B0 | `05ea0e4071a358cff97a5c146b46489e1d61c970` |
| `BIN/A06.BIN` | 0x80108F9C | 0x456BC | 0x157C | 0x38DFC | `27332ed9f09df70cfcbed5bd3046b44035bb328b` |
| `BIN/A07.BIN` | 0x80108F9C | 0x38158 | 0x16C4 | 0x28604 | `4c5163c1a5c51955b6bf3e17fe9db7a468baa89d` |
| `BIN/A08.BIN` | 0x80108F9C | 0x40F8C | 0x1580 | 0x38498 | `abddb36655ecc72406a2b497c212596c6db4d67c` |
| `BIN/A09.BIN` | 0x80108F9C | 0x566D | 0xDC | 0x5068 | `1ba8966b0b36c38b386ad3050f3b672ba43408a7` |
| `BIN/A0A.BIN` | 0x80108F9C | 0x1EE38 | 0x378 | 0xFEC0 | `5a582ac84ecf40074c7e4d65061b8ba82cd0dc08` |
| `BIN/A0B.BIN` | 0x80108F9C | 0x1C368 | 0x320 | 0xF2D8 | `75269941dacd0b68102076181d0463f274e1ded6` |
| `BIN/A0C.BIN` | 0x80108F9C | 0x1E1C0 | 0x338 | 0xFC90 | `c00f2b9fe2cf34a75086a65e1e29f7597900b0c6` |
| `BIN/A0D.BIN` | 0x80108F9C | 0x1D328 | 0x368 | 0xE53C | `305a5a4ed6587734767719e4ae4a1a27dc8156ac` |
| `BIN/A0E.BIN` | 0x80108F9C | 0x1E6F0 | 0x36C | 0xF098 | `9b9fa2b41c129826fbd9292a7965bdf47755d854` |
| `BIN/A0F.BIN` | 0x80108F9C | 0x2114C | 0x384 | 0x11A30 | `959318a353097c3bf7da0c1c9f41cbc07218f60f` |
| `BIN/A0G.BIN` | 0x80108F9C | 0x463C | 0x1E4 | 0x4310 | `d07947d2fe3969137f6171c94eb18bb80d57cde6` |
| `BIN/A0H.BIN` | 0x80108F9C | 0x36C0 | 0x140 | 0x35F0 | `47c2c86518bee63d4668e0ce75c4446969f34919` |
| `BIN/A0I.BIN` | 0x80108F9C | 0x3B30 | 0x154 | 0x3A74 | `501db41b522102fbdb698161997a390a87d0a79d` |
| `BIN/A0J.BIN` | 0x80108F9C | 0x4A7C | 0x128 | 0x42EC | `9a7152203c47f1120007784f9277f7bbad8a90e6` |
| `BIN/A0K.BIN` | 0x80108F9C | 0x16CC4 | 0x62C | 0xF724 | `9dfaa004c493bd474842379b8129a18baab1b702` |
| `BIN/A0L.BIN` | 0x80108F9C | 0x12450 | 0x264 | 0xA220 | `b5f09d338c809957134cb1fe8cc9828503479931` |

## Boundaries

The code ranges are a first split, not file splits: code starts at the first function (the first `addiu sp, sp, -n`
prologue, or the leaf before it that ends in an earlier `jr ra`) and ends after the last `jr ra` and its delay slot.
Refine a range when a function is named that falls outside it. `START` has no `jr ra`: its code from `0x274` runs to
the end of the file.

An overlay config reads `config/symbols/main.txt` for the resident names its code calls; splat lists those as
symbols it cannot place in the overlay's segment, which is expected.
