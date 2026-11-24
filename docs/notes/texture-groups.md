# Texture groups

`FUN_80044F58` loads texture group `set` (task bytes +0x6E set, +0x6D mode) from two files whose first sectors the file
table leaves at `0x800BE0F0` and `0x800BE0F8`:

- `/CD/TOMBA2.IDX` (LBA 6565, 48 sectors): one 2 KB header per set. Words 0 and 1 are the group's start and end
  byte offsets in `TOMBA2.IMG`; 42 words from +0x100 are copied to `0x800FB170`. Sets 26, 29, 35 and 41..47 are empty
  (end not after start). Mode 2 reads the header 4 or 26 sectors further on, by the set's bit in `0x800BFE56`.
- `/CD/TOMBA2.IMG` (LBA 6613): the groups, each read whole to `0x8018A000`. Word 0 is the image count; from +4,
  12-byte entries `{x, y, w, h: s16; packed length: u32}` (a VRAM rectangle in 16-bit words); the packed images
  follow from +0x800, one after the other.

`FUN_80044E84` decompresses each image with `FUN_80044D8C` into scratch ending at `0x801FD000` and uploads it with
`FUN_80081218` (LoadImage) to its rectangle.

`FUN_80044D8C(desc, dst, src, length)` is a byte LZ. A control byte gives a count (bits 3..7) and a mode (bits 0..2).
Mode 0 copies `count` literal bytes, and a count of 0 ends the image. Modes 1..7 copy `count` bytes from earlier
output at `base + 2 * factor * w` before the write position (`w` the entry's width in words). The (base, factor)
pairs are the table at `0x800153C8`: (0,0), (-1,0), (0,-1), (-1,-1), (-2,-1), (-3,-1), (1,-1), (2,-1).

## The title screen

Set 2 is the title screen: an 8-bit image at (640, 256), 256 x 240 words (512 x 240 pixels), and five 256-colour
palettes at (640, 507). The picture is its left 320 x 240 through the palette on row 511 (the fifth); the right 192
columns hold the menu labels (New Game, Load Game, Options, Start Game).
