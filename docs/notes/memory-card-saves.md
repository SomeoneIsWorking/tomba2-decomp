# Memory card saves

CRD (`0x8018A000`) holds the browser, the card I/O and the save/load code; MAIN holds the card init, the BIOS
wrappers and the page driver. Addresses are from the decompiled C of both images
(`decomp_pipeline.py --image-name tomba2_crd`, `tomba2`).

## Files and blocks

- One save per file, one 8 KB block each: `bu<port<<4 as %02X>:BASCUSP94454T2-<NN>`, NN = 00..14 (the writer
  refuses NN >= 15). So 15 saves per card at most; the number is the browser's slot, not a directory frame.
- File layout (0x980 bytes written by `CardSave_writeFile`): 0x000..0x37F the BIOS title/icon header
  (`CardSave_buildHeader`), 0x380..0x97F the 0x600-byte state image. Image byte 0x5FE = 3 (format tag), 0x5FF = XOR
  of image bytes 0x000..0x5FE.
- The image is RAM `0x800BF870..0x800BFE63` (0x5F4 bytes, `g_GblAreaId` onward), padded to 0x5FE. The writer
  clears `0x800BF9EB` before copying.

## BIOS calls

| wrapper | call |
|---|---|
| `0x80080840` | B0:0B TestEvent |
| `0x800808B0` / `C0` / `D0` / `E0` / `F0` | B0:32 open, 33 lseek, 34 read, 35 write, 36 close |
| `0x80080940` / `0x80080900` | firstfile (libc device hook, B0:42), B0:43 nextfile |
| `0x8009BA90` / `0x8009BAA0` | A0:AB `_card_info`, A0:AC `_card_load` |
| `0x8009BAB0` | `_card_clear`: B0:50 `_new_card`, then B0:4E `_card_write(port, 0x3F, 0)` |
| `0x8009C3F4` | `_card_format` |

`CardSys_init` (`0x8001CC00`, boot) zeroes the browser block, opens SwCARD events IOE/ERROR/TIMEOUT/NEWCARD into
`0x800BF4B4/B8/BC/C0` and HwCARD into `0x800BF4C4/C8/CC/D0`, enables them, and clears both directory tables.
Every async card op is finished by testing the four SwCARD events in that order: 1 done, 2 error, 3 timeout,
4 new card.

## The browser block `g_CardBrowser` (`0x800BF498`)

| off | |
|---|---|
| +0x00 | page state (`CardBrowser_step` switch) |
| +0x01, +0x02 | page step, op step |
| +0x03 | page to return to after a message page |
| +0x04 | open fd |
| +0x08 | list buffer, `0x800E7AD8`: s16 count, then 0x3C-byte entries from +4 |
| +0x14 | file buffer, `0x800D34E0` (0x980 bytes) |
| +0x3C | mode argument: bit 0 save (else load), bit 7 return after save |
| +0x3D | port the check is polling (alternates 0/1) |
| +0x3E | op result: 0 busy, 1 ok, 2 failed, 3 timeout, 4 new card |
| +0x3F | op code run by `CardBrowser_runOp` |
| +0x40 | 0/1 port check phase, 2 op running (check paused) |
| +0x41 | last port check result, 0xFF while one is in flight (ops wait on it) |
| +0x46 | chosen port, 0xFF none; +0x45 = port + 1 (cursor) |
| +0x47 | chosen list index |
| +0x48 + port | file number last loaded/saved on that port |
| +0x4A + port | port state: 0 card ok, 1 new card, 0xFE none/error, 0xFC unchecked |
| +0x4E + port | directory count, 0xFF not read |
| +0x50 | frame countdown |
| +0x51 | op delay: open/read/write run once it passes 6 |
| +0x5A | retry count |

`CardMenu_step` (`0x8007BE18`) switches on sm[0x50]: 0 `CardBrowser_init(mode)` (sets sm[0x50] = 1, list and file
buffers), 1 `CardBrowser_step`, 2 cancel (sm[0x6B] = 1), 3 sm[0x6B] = 2, 4 sm[0x6B] = 7 (load done),
5 sm[0x6B] = 9 (save done, mode bit 7). Each resets sm[0x50] to 0.

## Finding saves

1. Port page (state 1, `0x8018DD38`): confirm (pad 0x4000) picks a port whose +0x4A is < 3, then state 2 with op 0x10.
2. Op 0x10/0x11 `CardPort_infoLoad` (`0x8018BF74`): `_card_info`, then `_card_load` (or `_card_clear` on a new
   card) on port << 4. State 2 (`0x8018DFEC`) on result 1: directory not read (+0x4E+port = 0xFF) goes to state 3
   with op 0x0C, else straight to `CardBrowser_buildList`.
3. Op 0x0C/0x0D `CardDir_scan` (`0x8018BA68`), step 0, synchronous: firstfile/nextfile over `bu%02X:*`; for each
   name matching the 15-byte prefix `BASCUSP94454T2-` it opens the file, seeks 0x380 and reads 0x80 bytes (the
   head of the image), then fills the entry for slot atoi(name[15..16]). Free blocks (15 - blocks used) fill the
   first empty slots as entries with +2 = 0. Present entries are packed in slot order into the directory table
   `g_CardDirTable` (`0x800BED90` + port * 0x384, 15 x 0x3C) and +0x4E+port = their count. Steps 1-2 run
   `_card_info` and wait for its event.
4. `CardBrowser_buildList` (`0x8018D74C`): load mode copies entries with +1 = 1 and +2 = 1 into the list buffer and
   goes to state 10 (`0x8018ED94`); save mode copies all of them (saves and free blocks) and goes to state 6. The
   cursor +0x47 starts on the entry whose file number is +0x48+port.

Directory entry (0x3C bytes), with the image offset each summary field comes from:

| off | | image |
|---|---|---|
| +0x00 | file number NN | |
| +0x01 | present | |
| +0x02 | 1 Tomba save, 0 free block | |
| +0x03 | path string `bu00:BASCUSP94454T2-NN` | |
| +0x21 | byte | 0x11 |
| +0x22, +0x23 | area, sub (`AreaName_lookup` 0x80078798) | 0x00, 0x01 |
| +0x24 | u16 | 0x2E |
| +0x27 | count drawn beside an icon | 0x13 |
| +0x28..+0x2F | flags, bit 7 counted for a second icon | 0x14..0x1B |
| +0x30, +0x32 | u16 pair, drawn `EVENT %3d/%3d` | 0x3A, 0x38 |
| +0x34 | u32 play time in frames, drawn `%03d:%02d:%02d` (/108000, /1800, /30) | 0x08 (`0x800BF878`) |
| +0x38 | u32 drawn `AP %7d` | 0x04 (`0x800BF874`) |

`CardBrowser_drawSave` (`0x8018C790`) draws one entry; `NEW` marks a free block.

## Loading a save

State 10 confirm -> state 11 (`0x8018EFDC`, yes/no) -> yes sets state 12 (`0x8018F280`) and op 1. Ops run from
`CardBrowser_runOp` (`0x8018F660`) at the top of every `CardBrowser_step`, only while +0x41 != 0xFF:

- op 1: +0x5A = 0, +0x5B = 0xF, +0x51 = 0, +0x40 = 2, op 2.
- op 2 `CardLoad_openRead` (`0x8018B020`), once +0x51 has passed 6 (the 8th call after op 1): open `bu%02X:BASCUSP94454T2-%02d` (port << 4, list
  entry +0x47's file number) with 0x8001 (read, async), lseek 0x380, clear the SwCARD events, read 0x600 bytes into
  the file buffer; op 3. Seven failed opens: `CardSys_resetPorts`, result 2.
- op 3 `CardLoad_verifyApply` (`0x8018B1A4`): tests the SwCARD events; none fired keeps op 3. On done it closes
  the fd and checks image[0x5FE] = 3 and the XOR at 0x5FF. A good image: `0x1F8001FF` = the old `0x800BF870` byte,
  `0x1F800278` = the old u16 `0x800BFE56`, then image 0x000..0x5F3 is copied to `0x800BF870`; result 1. A bad
  image, error or timeout is result 2 at once; a new-card event runs `_card_clear` and retries from op 2, up to
  0x32 times.

State 12 on result 1 stores the file number into +0x48+port and goes to state 13: 30 frames (pad 0x6000 cuts it
short), then sm[0x50] = 4, and the next `CardMenu_step` sets sm[0x6B] = 7. Nothing else is applied by CRD: the
loaded state is the RAM image at `0x800BF870`, which the caller's GAME start reads.

## A direct load

The smallest guest path is the two load ops on the browser block. Required: CRD resident (`FUN_80045558(1)`), the
block initialised by `CardBrowser_init(0)` (sm[0x50] = 0 then one `CardMenu_step`), and the SwCARD events from
`CardSys_init`. Then:

1. +0x46 = port (0/1), +0x47 = 0, list buffer byte `0x800E7ADC` = NN, +0x51 = 7, +0x5A = 0, +0x3E = 0.
2. Call `CardLoad_openRead(0x800BF498)`; +0x3F = 3 on success, +0x3E = 2 on failure.
3. Call `CardLoad_verifyApply(0x800BF498)` until +0x3E != 0 (with an instant card it settles on the first call).
   +0x3F = 2 means a retry: set +0x51 = 7 and go to 2. +0x3E = 1 is a loaded image.
4. sm[0x50] = 4 and `CardMenu_step` (or sm[0x6B] = 7 directly): the load-accepted answer the DEMO page reads.

To pick the newest save the port first needs the directory: run op 0x10 then 0x0C through `CardBrowser_runOp`
with +0x46 = port (or call `CardDir_scan` step 0), then read `g_CardDirTable` entries with +2 = 1.

## Ordering

No save count and no timestamp: the card directory carries none, and the image has no counter beyond play time.
The only time-like field is the u32 at image +0x08 (`0x800BF878`), shown as play time at 30 per second. The
largest value is the furthest-played save, not necessarily the last written (two playthroughs on one card). Its
writer in MAIN or the overlays is not found statically: no absolute reference to `0x800BF878` outside CRD.
