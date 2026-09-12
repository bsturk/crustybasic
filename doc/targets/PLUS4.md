# crustyBASIC on Commodore Plus/4

Use `@OPTION TARGET plus4` for the default Commodore Plus/4 profile.
It builds a BASIC-loaded `.prg`.

Portable runtime calls are documented in [`../API.md`](../API.md).
This page covers Plus/4 systems, modes, formats, and hardware notes.

| System | CPU | Output | Text | Host ROM | Default assembler |
| ------ | --- | ------ | ---- | -------- | ----------------- |
| `plus4.orig` | 6502 | `.prg` | 40x25 | yes | `vasm` |

crustyBASIC strings use direct ASCII letter values. `A-Z` maps to
`$41-$5A`, `a-z` maps to `$61-$7A`, and a newline maps to `$0D`.
The CBM BASIC dialect uses VICE petcat listing conversion instead:
`A-Z` maps to `$C1-$DA`, `a-z` maps to `$41-$5A`, and `~` maps to
`$FF`. Use `{$NN}` for an exact PETSCII byte. `REAL` uses the BASIC
3.5 ROM floating point routines.

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
record the native value in `DISPLAY_MODE`.

Bitmap modes use the bitmap at `$4000`, luma attributes at `$0800`,
and chroma attributes at `$0C00`.

`DISPLAY_MIXED mode, first_row, rows` puts full width text rows over a
bitmap. The text rows use color memory at `$6000` and character memory at
`$6400`, just above the bitmap, so leave that area free. Page flipped soft
sprites use the same area and cannot be combined with mixed mode. A later
`DISPLAY` call leaves it.

`CELL_MULTICOLOR` uses the text screen at `$0C00` and color RAM at
`$0800`. `CELL_COLORS fg, bg, color2, color3` sets the default
foreground, background, and two shared multicolor text colors. The
per-cell `CELL_COLOR col, row, color` sets that cell's foreground color.
The other three colors are shared across the screen.

## Scrolling

`SCROLL_BEGIN 8, 8` enables smooth cell scrolling in both directions.
Use zero for either step to disable that axis. The backing screen is
40x25 cells; scrolling reduces the visible area to 38 columns and 24 rows
when both axes are enabled.

Scrolling uses the normal text screen and a second screen at `$6000-$67FF`,
which is reserved automatically. Set the cell colors before `SCROLL_BEGIN`.
Individual cell colors do not scroll. Bitmap scrolling and fixed regions
are not supported.

See [`scroll.cbs`](../../examples/__portable__/scroll.cbs).

## Images

Native image display supports `IMAGE_FMT_TED_HIRES` and
`IMAGE_FMT_TED_MULTI` for Botticelli and Multi Botticelli files
(10050 bytes, load address `$7800`). KERNAL-backed runtime image loading
is available.

Both formats support wipes, slides, and eight step fades. Effects use
unpacked images in memory; fade out works on the displayed bitmap.

## Sprites

TED has no hardware sprites. The portable API draws software sprites
over the bitmap:

| `SPRITE_DEFAULT_MODE` | Shape | Visible colors | X range | Data |
| --- | --- | --- | --- | --- |
| `SPRITE_MODE_HIRES` (default) | 8x8 one bit | 1 | 0 through 312 | 8 bytes |
| `SPRITE_MODE_MULTICOLOR` | 4x8 two bit | 3 | 0 through 156 | 8 bytes |

`SPRITE_DATA` copies target native bytes without conversion. In
multicolor mode each byte holds four logical pixels. The software path
treats `00` as transparent, `01` as `SPRITE_COLOR2`, `10` as the slot's
`SPRITE_COLOR`, and `11` as `SPRITE_COLOR3`. `SPRITE_BG` sets the bitmap
backdrop.

`SPRITE_DATA_TILES` accepts any runtime tile width and height. Tiles are
stored left to right, then top to bottom. Each tile covers 8x8 hires
pixels or 4x8 multicolor pixels. Saved bitmap and cell color storage is
allocated from the supplied dimensions.

With page flipping enabled, `SPRITES_FLUSH` waits before showing the
finished page. Use it once per game update without a separate `FRAME_WAIT`.
Sprite colors are refreshed when their covered cells or colors change.

`SPRITES_ON` selects the matching bitmap mode. See
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

The CBM clock calls are specific to this target family:

| Call | Purpose |
| --- | --- |
| `TI` | Read the 24 bit KERNAL jiffy clock as `U32`. |
| `TI_SET jiffies` | Set the KERNAL jiffy clock from the low 24 bits. |
| `TI$` | Return the clock as a six character `HHMMSS` string. |

`TICKS` uses `TI` as its source. `TICKS_RESET` records a new starting
value without changing the KERNAL clock.

## Files

`FILE_LOAD_PROGRAM(path$)` loads a PRG from disk device 8 using its
stored address. `FILE_LOAD_PROGRAM(path$, dst)` uses `dst` instead.
Both skip the two address bytes and load the remaining file through
KERNAL LOAD. They return zero on success or the KERNAL error code,
without running the loaded code.

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
