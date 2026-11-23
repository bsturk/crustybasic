# crustyBASIC on Commodore Plus/4

Use `@OPTION TARGET plus4` for the default Commodore Plus/4 profile.
It builds a BASIC-loaded `.prg`.

Portable runtime calls are documented in [`../API.md`](../API.md).
This page covers Plus/4 systems, modes, formats, and hardware notes.

| System | CPU | Output | Text | Host ROM | Default assembler |
| ------ | --- | ------ | ---- | -------- | ----------------- |
| `plus4.orig` | 6502 | `.prg`, `.d64` | 40x25 | yes | `vasm` |

`TEXT_ENCODING` defaults to `UPPER_CASE`: `A-Z` maps to `$41-$5A`,
`a-z` maps to `$61-$7A`, and a newline maps to `$0D`.

For both letter cases, use `@OPTION TEXT_ENCODING MIXED_CASE` and
`PRINT "{LOWER}";`. This encoding maps `A-Z` to `$C1-$DA` and `a-z`
to `$41-$5A`. `UPPER` and `LOWER` string functions follow the encoding.

Startup preserves the selected letter case. Printing `{LOWER}` selects
mixed case; `{UPPER}` selects uppercase and graphics. These controls
affect the whole screen. Returning to BASIC keeps that selection.

See [CBM text escapes](CBM_TEXT.md) for named controls, colors, and graphics.

Use `{$NN}` for an exact PETSCII byte. The CBM BASIC dialect also maps
`~` to `$FF` for VICE petcat listings. By default, `REAL` uses the BASIC 3.5 ROM
floating point routines.

Use `--set optimize=speed --set math-real=target` for faster addition,
subtraction, and multiplication with the same five byte format. The
required routines occupy extra program RAM. Rounding can differ from ROM
results. Division and transcendental functions use ROM;
`optimize=size math-real=target` uses ROM throughout.

## Program memory

Normal builds use `$1010-$7FFF` for code and data.
Select `@OPTION NOBASIC TRUE` or `--set nobasic=true` to extend this
through `$FBFF`, adding 31 KB before runtime overhead. These builds use
built-in math and keep KERNAL I/O and interrupts available. ROM is
restored when the program ends. Screen and bitmap memory used by the
program is reserved automatically.

PRG and D64 disk output support both memory layouts. Select
`@OPTION OUTPUT_TYPE disk` for a D64.

A `PROC` can call itself, directly or through another `PROC`. Each
nested call uses the CPU stack, so keep recursion shallow.

## Graphics

`DISPLAY_HIDE` hides the display without clearing it. `DISPLAY_SHOW`
reveals the current contents. `DISPLAY_VISIBILITY_SUPPORTED` is `TRUE`.
The border stays visible, including while using mixed mode.

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
and chroma attributes at `$0C00`. `@OPTION PLUS4_BITMAP_BASE $8000` moves the
bitmap to `$8000`, beneath the BASIC ROM, so program memory stays in one
piece from `$1010` to `$7FFF`.

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

In `CELL` mode, `CELL_ATTRIB` selects `INVERSE` and `BLINKING` text.

`CHARSET_COPY_DEFAULT` copies the selected ROM font to RAM at `$3800-$3BFF`.
`CHARSET_DEFINE` replaces an 8 byte glyph; `CHARSET_INSTALL` replaces all
128 glyphs. `CHARSET_RESET` restores the previous font. Characters 128-255
display the inverse of characters 0-127.

`@OPTION PLUS4_CHARSET_256 TRUE` uses 256 distinct glyphs at
`$3800-$3FFF`. Character codes use all eight bits, so `INVERSE` leaves
the glyph unchanged. `BLINKING` still applies. Charset installation,
copying, and reset select the matching character mode.

## Scrolling

`SCROLL_BEGIN SCROLL_AXIS_BOTH` enables pixel scrolling in cell, multicolor
cell, hires bitmap, and multicolor bitmap modes. Use `SCROLL_AXIS_X` or
`SCROLL_AXIS_Y` to enable only one axis. `SCROLL_MOVE dx, dy` accepts native
pixel distances, including one pixel. A multicolor pixel spans two hires
dots horizontally. Move one axis per call and cross at most one cell boundary.

`SCROLL_BEGIN_REGION x, y, columns, rows, axes` scrolls a cell rectangle
while keeping cells outside it fixed. Regions move by whole cells;
read `SCROLL_X_GRANULARITY` and `SCROLL_Y_GRANULARITY` for the pixel
steps. Edge refill coordinates are relative to the region.

The backing screen is 40x25 cells; enabling both axes gives a 38x24 cell
view. Cell scrolling uses a second screen at `$6000-$67FF`, reserved
automatically. Characters and their colors move together. Bitmap scrolling
moves the bitmap and both color planes, preserving the selected display mode.
Bitmap copies at cell boundaries can take more than one frame.

Fill the edge returned by `SCROLL_MOVE`, then call `SCROLL_PRESENT`.
`SCROLL_CELL_PUTC col, row, code, fg` sets the new cell's foreground color;
the three argument form and `SCROLL_CELL_COLUMN` use the current foreground.
In bitmap modes these calls draw glyphs from the selected character set.
For multicolor bitmaps, `fg` selects the color of pixel value 1.
`SCROLL_CLOSE` restores the full screen dimensions.

`TED_SCROLL_REGION first_row, rows` applies horizontal fine scrolling to
a band of the display, leaving the rows outside it fixed. Rows run from
0 through 24. `TED_SCROLL_OFFSET offset` sets its pixel offset from 0
through 7, initially 7. The band uses 38 columns; the surrounding rows
use 40. These calls move the display window; the program supplies the
screen contents.

`RASTER_STOP` restores the previous interrupt and scroll settings.
`RASTER_START` resumes the band, and `RASTER_UNINSTALL` removes it.

## Images

Native image display supports `IMAGE_FMT_TED_HIRES` and
`IMAGE_FMT_TED_MULTI` for Botticelli and Multi Botticelli files
(10050 bytes, load address `$7800`). KERNAL-backed runtime image loading
reads these native files and crustyBASIC image descriptors through
`IMAGE_LOAD(path$)`.

Image descriptors use raw data or RLE8 packing. `IMAGE_LOAD` expands
packed planes directly into screen memory from a file or memory address.

Both formats support wipes, slides, eight step fades, and dissolves.
Effects use unpacked images in memory; fade out and dissolve out work on
the displayed bitmap. Dissolves use 1, 2, 4, or 8 pixel square blocks and
preserve the image colors.

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
finished page. Call `FRAME_WAIT` after each update; it counts that wait
without waiting again.
Sprite colors are refreshed when their covered cells or colors change.

Select the matching bitmap mode with `DISPLAY DISPLAY_MODE_SPRITE` before
`SPRITES_ON`.

`SPRITE_FLIP_X` and `SPRITE_FLIP_Y` mirror complete shapes, including
layouts supplied by `SPRITE_DATA_TILES`.
`SPRITE_EXPAND` doubles the shape along either axis. Expansion and
flipping can be combined.

`SPRITE_PRIORITY id, TRUE` places a sprite behind occupied bitmap
pixels. Cell colors remain shared with the bitmap.

`SPRITE_HIT` tests occupied pixels against other visible sprites.
`SPRITE_HIT_BG` tests occupied pixels against the bitmap with all
sprites removed. Both calls use the current shapes and positions.

`SPRITES_BEGIN` and `SPRITES_END` group sprite moves, shape changes,
colors, and visibility changes into one redraw. Individual calls draw
immediately when `SPRITE_DOUBLE_BUFFER` is false.

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

## Sound

TED has two sound voices with a shared master volume. Timed notes and
rests use KERNAL jiffy ticks. `FRAME_WAIT` and timed status queries
service their durations. A new note or `SOUND_OFF` cancels the previous
duration on that voice.

`SOUND_ENVELOPE` selects attack, decay, sustain and release settings or
a table of volume stages for subsequent notes. `FRAME_WAIT` advances
the curve. The most recently started note controls the master volume
and its selected envelope, affecting both voices.

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

`TIMER_START` claims TED timer 2 until `TIMER_STOP`, which returns its
elapsed hardware count modulo 65536 in `TIMER_LO`. `TIMER_HI` is zero.
The counter runs at about 894 kHz on NTSC and 884 kHz on PAL.

`RASTER_LINE_INSTALL` runs a callback at rows selected by
`RASTER_MARK_ROW` or `RASTER_MARK_ROWS`, numbered 0 through 24.
Row N starts at TED raster line N * 8 + 3. `RASTER_MARK_OFFSET` selects
a raw raster line from 0 through 199. `RASTER_FRAME_INSTALL` runs once
per video frame in the bottom border. These callbacks share the raster
service with mixed display and `TED_SCROLL_REGION`.

`RASTER_STOP` pauses the callbacks and `RASTER_START` resumes them.
`RASTER_UNINSTALL` removes them. `RASTER_WSYNC` waits for the next
raster line. Keep interrupt callbacks short.

## Files

`FILE_LOAD_PROGRAM(path$)` loads a PRG from disk device 8 using its
stored address. `FILE_LOAD_PROGRAM(path$, dst)` uses `dst` instead.
Both skip the two address bytes and load the remaining file through
KERNAL LOAD. They return zero on success or the KERNAL error code,
without running the loaded code.

File channels run from 1 to 15. `FILE_OPEN` uses disk device 8, and the
KERNAL allows up to 10 files open at once. Plus/4 specific opens can choose
another device or secondary address.

The lower KERNAL channel helpers are also callable by target-specific
programs: `OPEN_CH`, `CLOSE_CH`, `GET_CH`, `PUT_CH`, `PRINT_CH_STR`,
`ST`, and `CMD_CH`.

`cbm_basic_v3_5` targets `plus4` by default.

Commodore BASIC listings often use POKEs to locations 51/52 and 55/56 to
reserve memory. The dialect recognizes fixed values inside program memory
and reserves from that address to the top of program memory. Calculated or
incomplete values produce a warning requesting an explicit
`@OPTION RAM_TOP $xxxx`.
