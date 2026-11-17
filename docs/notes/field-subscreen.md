# The field subscreen and its Items tab

The page the field opens from play (not Start's pause menu): four tabs over a darkened field, Items first. Read
from `FUN_8010810C`, `FUN_80108624`, `FUN_800346BC`, `FUN_80036DFC`, `FUN_80037E44`, `FUN_800368D0` and
`FUN_80036240` with `decomp_pipeline.py` (every target's body present).

## Pages of the GAME stage

`GameStage_pageStep` (`FUN_8010810C`, GAME) runs the page in `sm[0x6B]` (sm = `*0x1F800138`); `0x1F800136` holds the
field draw (0 play, 1 the pause menu over it, 2 and up a page that holds it off) and `sm[0x50]` is the page's own step.

| page | what |
|---|---|
| 0 | play: Start (`0x800E7E68 & 8`) opens page 1; the field button `*0x1F800176` opens page 6 when `0x800E7E84` is 1, no event holds (`0x800BF80D`, `0x1F800137`, `0x800BF839`) and `0x800BF818` is not 1 or 3. It writes `sm[0x6B] = 6`, `0x1F800136 = 0x800E7E84` and `sm[0x50] = 0` |
| 1 | the pause menu (`PauseMenu_step`, `FUN_801084F8`): three entries in `0x800BF808`; Cross opens page `entry + 3` |
| 2, 9, 11 | back to play |
| 3 | Options (`FUN_8007B45C`) |
| 4, 8 | the card pages (`FUN_8007BF20` with 0 or 1) |
| 5 | the leave-game prompt |
| 6 | the subscreen (`GameStage_subscreenPage`, `FUN_80108624`; areas 3, 5 and 20 have their own) |
| 7 | leave to the title |
| 10 | `FUN_80121040` |

`GameStage_subscreenPage` waits two steps, opens it (`Subscreen_open`, `FUN_8007AC6C`, zeroes the 0x30-byte block
`g_Subscreen` at `0x80100660`) and then runs `Subscreen_step` (`FUN_800346BC`) each frame until it returns true.

## `g_Subscreen` (0x80100660)

| offset | field |
|---|---|
| +0x00 | phase: 0 opening (two frames: the template base, then each tab's list), 1 running, 2 closing |
| +0x01 | the open tab's step |
| +0x04 | the UI template base `FUN_8007E1B8` draws from (`PTR_DAT_800ECF58`) |
| +0x08 | the tab, 0..3; 0 Items |
| +0x09 | entered: the cursor is inside the tab, not on the tab row |
| +0x0C, +0x10, +0x14 | per tab: x and y offsets and ordering-table bucket (100 for the open tab, 0x84 for the others) |
| +0x18 | the held-direction repeat count |
| +0x1C | Items: the cursor's row on screen |
| +0x1D | Items: the column, 0 or 1 |
| +0x1E | Items: the first row shown (seven are) |
| +0x1F | Items: the cursor's row in the list |
| +0x20 | Items: the hint's kind (the selected item's kind, 5 on the Sort cell) |
| +0x22 | Items: the list's rows |
| +0x24, +0x25, +0x26 | Items: the count of kind-0, kind-1 and other items (`FUN_80036240`) |

While not entered, Left/Right choose the tab and Cross or Down enters it (tab 2 refuses, tab 3 only with
`0x80100683`); Triangle and Circle close the subscreen. Entered, Triangle closes it and the tab's step takes the rest.

## The Items tab

- `ItemsTab_buildLists` (`FUN_80036240`) sorts the owned items (`0x800BFAB4` count non-zero) by `g_ItemText[id]`
  kind (+0): kind 0 (equipment: the suits, `0x0E`-`0x16`, among them) fills the list's first rows, two to a row, then
  kinds 1 and the rest. `0x800BFBB4[id]` is each item's place within its kind.
- `ItemsTab_itemAt` (`FUN_800368D0`, a leaf that only reads) gives the item in (row, column): its id, 999 for the
  Sort cell after the last item, negative for an empty cell.
- `ItemsTab_step` (`FUN_80036DFC`) is the tab's step machine on `+0x01`: 1 browses (Up/Down/Left/Right move, L2/R2
  page, Cross acts on the item through `FUN_80035FAC`, Square sets the hint to the item's, Circle leaves the tab),
  2 and 3 choose a map place for the warp items `0x5F` and `0x60`, 4 (Cross on Sort) and 5 (L1/R1) flip the sort
  order `0x800BF8A0`. Select (`0x0001`) is not read on this tab.
- `ItemsTab_draw` (`FUN_80037E44`) draws the tab: its frame, the cursor (template `*0x800174C8` at x 0x54 or 0xE4,
  y `screenRow * 20 + 0x35`, 0x39 for a non-equipment row), seven rows from y 0x30 at 20 a row, names in the 8x8 font
  (`FUN_800376EC`: x 0x2C or 0xBC, colour 8 for an item that can be worn now, 4 for one that cannot) into bucket
  `+0x14 + 3`, and the worn mark beside the worn suit (`0x800BF88C`).

## The hint box

`SubscreenHint_draw` (`FUN_80033FF4`, `(state, bucket 0x65)`) draws the box (three parts from `0x800A241C` at y 200,
the first skipped while the Help tab is entered, and 18 strip quads from `0x800A2430`, 16 pixels apart from x 0x1A)
and then one string in the 8x8 font at (0x28, 0xBF), colour 0x12, bucket 0x61. Not entered, the string is the tab's
(`0x800A3A68`, `0x800A3A7C`, `0x800A3AC0`, `0x800A3AC4`); entered, Events has `0x800A3A80` and Items the hint of
`+0x20`: kind 0, 1 or 2 (`0x800A3A6C`, `0x800A3A70`, `0x800A3A74`) or 4, the selected item's note
(`g_ItemText[id] + 8`). The font's CLUT rows read on this screen: 0x12 grey, 0x13 dark brown, 0x16 white, 0x11 yellow,
0x10 pink.
