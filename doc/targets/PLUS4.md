# crustyBASIC on Commodore Plus/4

Use `@OPTION TARGET plus4` for the default Commodore Plus/4 profile.
It builds a BASIC-loaded `.prg`.

Portable runtime calls are documented in [`../API.md`](../API.md).
This page covers Plus/4 systems, modes, formats, and hardware notes.

| System | CPU | Output | Text | Host ROM | Default assembler |
| ------ | --- | ------ | ---- | -------- | ----------------- |
| `plus4.orig` | 6502 | `.prg` | 40x25 | yes | `vasm` |

Strings encode to PETSCII. `REAL` uses the BASIC 3.5 ROM floating
point routines.

## Graphics

The default graphics mode is `BITMAP_HIRES`.

| Mode | Plus/4 mode | Size | Colors |
| --- | --- | --- | --- |
| `CELL` | TED 40 column text | 40x25 | text |
| `BITMAP_HIRES` | TED hi res bitmap | 320x200 | 2 |
| `BITMAP_LORES` | not supported | | |
| `BITMAP_MULTICOLOR` | TED multicolor bitmap | 160x200 | 4 |
| `CELL_MULTICOLOR` | TED multicolor text | 40x25 | 4 |

Native TED mode constants are also available: `TED_TEXT`,
`TED_BITMAP_HIRES`, `TED_BITMAP_MULTICOLOR`, and
`TED_CELL_MULTICOLOR`. They select the same hardware modes and
record the native value in `GFX_MODE`.

Bitmap modes use the bitmap at `$4000`, luma attributes at `$0800`,
and chroma attributes at `$0C00`.

`CELL_MULTICOLOR` uses the text screen at `$0C00` and color RAM at
`$0800`. `CELL_COLOR fg, bg, color2, color3` sets the default
foreground, background, and two shared multicolor text colors. The
per-cell `CELL_COLOR col, row, color` sets that cell's foreground color.
The other three colors are shared across the screen.

## Images

Native image display supports `IMAGE_FMT_TED_HIRES` and
`IMAGE_FMT_TED_MULTI` for Botticelli and Multi Botticelli files
(10050 bytes, load address `$7800`). KERNAL-backed runtime image loading
is available.

## Sprites

TED has no hardware sprites. The portable API draws software sprites
over the bitmap:

| `MULTICOLOR_SPRITE` | Shape | Visible colors | X range | Data |
| --- | --- | --- | --- | --- |
| `FALSE` (default) | 8x8 one bit | 1 | 0 through 312 | 8 bytes |
| `TRUE` | 4x8 two bit | 3 | 0 through 156 | 8 bytes |

`SPRITE_DATA` copies target native bytes without conversion. In
multicolor mode each byte holds four logical pixels. The software path
treats `00` as transparent, `01` as `SPRITE_COLOR2`, `10` as the slot's
`SPRITE_COLOR`, and `11` as `SPRITE_COLOR3`. `SPRITE_BG` sets the bitmap
backdrop.

`SPRITES_BEGIN` selects the matching bitmap mode. See
[`multicolor_sprite.cbs`](../../examples/plus4/crustybasic/multicolor_sprite.cbs).

## Input

| Capability | Value |
| --- | --- |
| Keyboard | Yes |
| Individual held keys | Yes |
| Joystick ports | 2 |
| Buttons | 1 per port |
| Stick type | Digital |
| Paddle axes | 0 |
| Keypad ports | 0 |
| Keypad `INPUT` | No |
| Mouse buttons | 0 |

`KEY_HELD` polls the keyboard matrix without consuming typed input.

## Timing

The runtime tick source is the KERNAL jiffy clock. It runs at about
60 Hz on PAL and NTSC because the KERNAL calibrates the TED interrupt by
video standard. The 24-hour wrap is compensated for one crossing.

## Files

File channels default to disk device 8 and use the logical channel as
the secondary address. Plus/4-specific opens can choose another device
or secondary address.

The lower KERNAL channel helpers are also callable by target-specific
programs: `OPEN_CH`, `CLOSE_CH`, `GET_CH`, `PUT_CH`, `PRINT_CH_STR`,
`ST`, and `CMD_CH`.

`cbm_basic_v3_5` targets `plus4` by default.

## Examples

Plus/4 specific examples live under:

- [`../../examples/plus4/`](../../examples/plus4/)
