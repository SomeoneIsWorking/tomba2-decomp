# Text fonts

Every text drawer of `MAIN.EXE` samples the 4bpp page at (960, 256), DR_TPAGE 0x1F, through a CLUT row at x 1008
(colour < 16: row 496 + colour) or x 992 (row 480 + colour). Read from `FUN_80078CA8`, `FUN_80078988`, `FUN_8007C940`,
`FUN_8007CC00` and `FUN_8007CDD4` and checked against the page in VRAM (area 8 pause menu).

## The page

| v | cells | what |
|---|---|---|
| 0-39 | 8x8, 32 a row | small font, cell = ASCII - 0x20; row 3 (v 24) starts circle, cross, triangle, square |
| 40-119 | 8x16, 32 a row | text font, cell = ASCII - 0x20; row 3 (v 88) starts circle, cross, triangle, square |
| 120-151 | 16x16, 16 a row | icons: circle, cross, triangle, square, 8 arrows, diamond, club, heart, spade; row 1 `!?`, `!!`, (c), marks |
| 152-175 | 8x8, 32 a row | a third, unshadowed font in indices 1-5; nothing draws it (see below) |
| 176- | | pre-drawn words (Items, Event, Status, Help, Load, Save) and digits |

The two letter fonts draw ASCII 0x5C as the yen sign and 0x2A as the reference mark; the 8x16 font's 0x5F is blank.
Glyphs are CLUT index 1 with indices 3, 4 and 5 on their edges, over a shadow in index 6 one texel right and below. The
icons use indices 1-4 and 6-D and have no shadow. The four buttons, in all three fonts, are a disc in indices 1-4 with
the symbol in its own indices (circle 7/8, cross 9/A, triangle B/C, square D), which their CLUT rows colour.

Read from the page in VRAM (area 8 Options), the symbol cells are one colour (index 1, edges 3-5, shadow 6) except:
in the 8x16 font the arrows 0x64-0x6B, ◇ 0x82, □ 0x84, △ 0x86 and ▽ 0x88 are filled in index 4 inside their outline,
where the 8x8 font draws them hollow; the box 0x98 is filled in index 3 in both. The 8x8 arrows 0x64-0x67 are white
(hollow) arrows, 0x68-0x6B hollow diagonal ones. Cells 0x7D and 0x7E, which the token table names 【 and 】, are drawn
as a square ring around a smaller square ring in both fonts.

Rows 3 and 4 of both letter fonts (cells 0x60-0x98) are the symbols the token table below names, in its order; in
the US build only `Font_iconGlyphEmit` reaches them; the US strings found draw the buttons 0x60-0x63 (the glyphEmit
bytes 0x01-0x04, the Options screens) and ■ 0x85 (the Options volume bars).

## The unshadowed font at v 152

No drawer samples it. `Font_glyphEmit` reaches v 136 at most (byte 0xFF on the 8x16 font), `Font_iconGlyphEmit`
v 32 (code 0x98) and the mark at v 64; no immediate 0x98 or 0x98xx outside the cursor template index
(`FUN_8007e998`), screen coordinates and SOP's tile map (`FUN_8010c26c`, a u offset) appears in `MAIN.EXE` or any
overlay; none of the UI templates under `PTR_DAT_800ecf58` (read at the title and the area 8 pause menu) samples
v 152-175 on the page. Only a `DialogText_layout` code 0xE0-0xEF would, as an 8x16 half-cell, which no font use
needs.

## Drawers

- `Font_glyphEmit` (`FUN_80078CA8`), reached through `Font_drawText` (`FUN_80079374`: a1 0x00100008, 8x16 sprites,
  scratch 0x1F800180 = 32) and `Font_drawTextSmall` (`FUN_80079324`: a1 0x00080008, 8x8, 0x1F800180 = -32). A byte
  b >= 0x20 other than space draws one SPRT (code 0x65) with n = b + (s16)0x1F800180, u = (n mod 32) * width,
  v = (n div 32) * height, plus 8 when the height is 16: the 8x16 font's cell b - 0x20 from v 40, or the 8x8 font's
  from v 0. Bytes 0x01-0x04 call `Font_iconGlyphEmit` with the token strings at 0x80016DA8, 0x80016DAC, 0x80016DA4,
  0x80016DA0.
- `Font_iconGlyphEmit` (`FUN_80078988`), also called by `FUN_8007F078`, `FUN_8007F104`, `FUN_8007F250` and
  `FUN_8007F498`: Shift-JIS pairs. Full-width A-Z, a-z and 0-9 map to their ASCII cell (0x8260 'Ａ' to 0x21); other
  pairs go through the token table at 0x800A55E0 ({string, u16 code}, stride 8). Each glyph is an 8x8 SPRT (code 0x75)
  at u = (code & 31) * 8, v = ((code & 0xFFF) >> 5) * 8: the small font. A code with bit 15 adds a mark at
  (56 or, with bit 12, 64; 64).

## The token table (0x800A55E0)

90 entries, NULL-terminated; each string is one Shift-JIS pair. A miss draws nothing and advances 8.

| codes | pairs |
|---|---|
| 0xFF02 | U+3000 ideographic space (advance only) |
| 0x01-0x0F, 0x1A-0x20, 0x3B-0x40, 0x5B-0x5E | full-width `!″#$%&'()*+,−./:;<=>?@[¥]^_` `{|}〜`, the ASCII cells |
| 0x60-0x63 | 決 不 三 四: the circle, cross, triangle and square buttons |
| 0x64-0x67 | → ← ↑ ↓ |
| 0x68-0x6B | 北 東 西 南: four diagonal arrows |
| 0x6C-0x78 | 、 。 ， ー ‐ ＼ 波 (a wave dash) … ‥ ‘ ’ “ ” |
| 0x79-0x90 | 「 」 ☆ ★ 【 】 ○ ● ◎ ◇ ◆ □ ■ △ ▲ ▽ ▼ ♪ ♂ ♀ ℃ ※ 『 』 |
| 0x91-0x98 | 菱 葉 心 剣 疑 驚 商 送: diamond, club, heart and spade suits, `!?`, `!!`, (c) and a box |

The kanji are private tokens for art: their cells are the shapes in the last column. The token strings at
0x80016DA0-0x80016DAC are four of these pairs: 0x80016DA0 四 (square), 0x80016DA4 三 (triangle), 0x80016DA8 不
(cross), 0x80016DAC 決 (circle), which `Font_glyphEmit`'s bytes 0x04, 0x03, 0x01 and 0x02 draw. The Options screens
(`FUN_8007F078`, `FUN_8007F104`, `FUN_8007F250`, `FUN_8007F498`) draw 決, 三 and 不 at 0x800172A4-0x800172AC, ■ for
the volume bars (0x800172F4) and `・` (0x800172B0), which the table misses.
- `DialogText_layout` (`FUN_8007C940`) lays a dialog node's string into `g_Glyphs` (0x800ECB88, 8 bytes each: s16 x,
  u8 y, u8 colour | 0x80 wide, u16 u, u16 v; count s16 0x1F80017E). Codes 0xF0-0xF7 set the colour, 0xFA breaks the
  line (y + 16), 0xFB advances 8, 0xC0-0xD0 call the inline-item layouts. A code c in 0x60-0x80 is wide: the icon
  cell c - 0x60 (u = ((c - 0x60) mod 16) * 16, v = ((c - 0x60) div 16) * 16 + 120), advancing 16; any other is
  u = (c & 31) * 8, v = (c >> 5) * 16 + 40, the text font's cell c, so a dialog code is ASCII - 0x20.
- `DialogGlyphs_emit` (`FUN_8007CC00`) draws `g_Glyphs` as SPRT 0x65 into bucket 2, 8x16 or, wide, 16x16.
- `DialogGlyphs_prompt` (`FUN_8007CDD4`) draws the continue prompt, a 16x16 SPRT at v 120, u 0 (circle) or 16
  (cross) by the button layout byte 0x800BFE4C.
