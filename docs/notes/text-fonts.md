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
| 152-175 | 8x8, 32 a row | a third, unshadowed font in indices 1-5 (no drawer above samples it) |
| 176- | | pre-drawn words (Items, Event, Status, Help, Load, Save) and digits |

The two letter fonts draw ASCII 0x5C as the yen sign and 0x2A as the reference mark; the 8x16 font's 0x5F is blank.
Glyphs are CLUT index 1 with index 5 on their edges, over a shadow in index 6 one texel right and below. The icons
use indices 1-4, 7 and 8 and have no shadow.

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
- `DialogText_layout` (`FUN_8007C940`) lays a dialog node's string into `g_Glyphs` (0x800ECB88, 8 bytes each: s16 x,
  u8 y, u8 colour | 0x80 wide, u16 u, u16 v; count s16 0x1F80017E). Codes 0xF0-0xF7 set the colour, 0xFA breaks the
  line (y + 16), 0xFB advances 8, 0xC0-0xD0 call the inline-item layouts. A code c in 0x60-0x80 is wide: the icon
  cell c - 0x60 (u = ((c - 0x60) mod 16) * 16, v = ((c - 0x60) div 16) * 16 + 120), advancing 16; any other is
  u = (c & 31) * 8, v = (c >> 5) * 16 + 40, the text font's cell c, so a dialog code is ASCII - 0x20.
- `DialogGlyphs_emit` (`FUN_8007CC00`) draws `g_Glyphs` as SPRT 0x65 into bucket 2, 8x16 or, wide, 16x16.
- `DialogGlyphs_prompt` (`FUN_8007CDD4`) draws the continue prompt, a 16x16 SPRT at v 120, u 0 (circle) or 16
  (cross) by the button layout byte 0x800BFE4C.
