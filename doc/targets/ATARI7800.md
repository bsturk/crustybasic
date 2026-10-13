# crustyBASIC on Atari 7800

Use `@OPTION TARGET atari7800` for Atari 7800 cartridges. The system
name is `atari7800.orig`. Output is a 32K `.a78` cartridge image.

Use `@OPTION OUTPUT_TYPE cart-32k-ram` for a 32K cartridge with 16K
RAM at `$4000-$7FFF`. The cartridge image selects SuperGame RAM hardware.

Portable runtime calls are documented in [`../API.md`](../API.md).
This page covers Atari 7800 modes and hardware notes.

## Basics

| Item | Value |
| --- | --- |
| CPU | 6502 |
| Output | `.a78` |
| Code start | `$8000` |
| RAM | 4K built in, split between program data and MARIA display data |
| Text grid | 20x24 MARIA backed text |
| `REAL` | Not supported |
| Joysticks | 2 digital ports |
| Buttons | 2 per port, with one-button fallback |
| Paddles | 4 analog axes and 4 triggers |
| Sound | 2 TIA voices |
| Recursive PROCs | Yes |

The target provides MARIA text, portable CELL text access,
palette/background control, frame hooks, joystick, console, and paddle
input, random bytes, TIA sound, RAM bitmap graphics, a full-height
MARIA tile display, MARIA sprites, and MARIA display-list raster
interrupts.

## Memory Regions

| Range | Use |
| --- | --- |
| `$0000-$001F` | TIA registers (sound, 2600-mode input) |
| `$0020-$003F` | MARIA registers |
| `$0040-$00FF` | Zero page; `$40-$68` is compiler scratch/state and `$69-$FF` is program data |
| `$0140-$01BF` | Program data |
| `$01C0-$01FF` | 6502 stack (64-byte budget, grows down from `$01FF`) |
| `$0280-$0297` | RIOT (switches and timers) |
| `$1800-$203F` | Main program data, buffers, scratch RAM |
| `$2040-$20FF` | Mirror of zero-page RAM |
| `$2100-$213F` | Program data |
| `$2140-$21FF` | Mirror of stack-page RAM |
| `$2200-$241F` | Program data |
| `$2420-$27FF` | MARIA DLL, display lists, and text screens |
| `$8000-$FF7F` | Cartridge ROM (code, const data) |
| `$FF80-$FFF7` | Cartridge signature |
| `$FFF8-$FFFF` | Startup flags and 6502 vectors |

The base 7800 has 4K RAM at `$1800-$27FF`, with parts mirrored into
page zero and the stack page. crustyBASIC places separate data blocks
only in the unique RAM ranges and keeps the stack area clear. The MARIA
display list list (DLL) starts at `$2420`; graphics display lists and
the two text screens use the remaining upper RAM. The startup footer
fails the build if code grows into the signature or vector area.

## Text

`SCREEN_TEXT` selects the 20x24 MARIA text screen. `PRINT`,
`POSITION`, `CLS`, `PUT_CHAR`, `CELL_PUTC`/`CELL_GETC`, and
`CELL_MEMMOVE` all operate on it. `PRINT` scrolls the screen up when it
passes the bottom row. The charmap covers ASCII `$20-$5F`; lowercase
letters print as uppercase. `CELL_CODE` maps a character to its cell code.

`COLOR color` changes the foreground color used by `PRINT` and
the CELL text helpers. It updates all three colors in MARIA palette 0.

Each text row has its own display list, so rows can point at different
MARIA palettes.
`SET_PALETTE index, c1, c2, c3` writes one of the eight 3-color MARIA
palettes and `SET_BACKGROUND color` sets the shared background
register.

## Graphics

Portable modes map to these MARIA modes:

| Portable mode | MARIA mode | Surface | Colors |
| --- | --- | --- | --- |
| `CELL` | text | 20x24 cells | shared foreground and background |
| `CELL_MULTICOLOR` | text | 20x24 cells | background plus 3 foreground colors |
| `BITMAP_HIRES` | `MARIA_320A` | 248x80, 1 bit per pixel | 2 |
| `BITMAP_LORES` | `MARIA_160B` | 62x80, 4 bits per pixel | background plus 12 |
| `BITMAP_MULTICOLOR` | `MARIA_160A` | 124x80, 2 bits per pixel | background plus 3 |

All six native MARIA names are also available directly:

| Native mode | Surface | Colors |
| --- | --- | --- |
| `MARIA_160A` | 124x80, 2 bits per pixel | 4 |
| `MARIA_160B` | 62x80, 4 bits per pixel | 13 |
| `MARIA_320A` | 320x192, 40x24 indirect tiles | 2 per tile row |
| `MARIA_320B` | 124x80, 2 bits per pixel | 4 |
| `MARIA_320C` | 124x80, paired palette selection | 9 |
| `MARIA_320D` | 248x80, uniform 1 bit pixel mapping | 2 |

`MARIA_320A` is the full-height tile display. It uses 8x8 tiles instead
of a writable pixel bitmap, so use the TILE API with it. `PLOT`,
`POINT`, and the other pixel calls are not available in this mode.
`PRINT`, `POSITION`, `TEXT_COLUMNS`, and the CELL calls use all 40 columns.

The other five native modes use RAM bitmap surfaces. Each logical row
is displayed over two scanlines, giving a 160-scanline bitmap between
the blank top and bottom areas. The 320 bitmap modes use true 320-pixel
timing, but the writable surface is narrower than 320 pixels. One
normal MARIA object holds at most 31 bytes, or 248 one-bit pixels. The
bitmap rows are split across the unique RAM ranges, but the graphics
calls still present one surface. `GFX_PAGE_BASE(0)` returns `0`
because there is no single linear framebuffer address.

`MARIA_320C` can use eight palette colors plus the background across
the display. One row can select four of those foreground colors because
its bitmap is one MARIA object. `MARIA_320D` has unusual alternating
palette rules; the portable pixel API gives it a predictable two-color
mapping.

`PLOT`, `UNPLOT`, `POINT`, `COLOR`, `CLS`, and the portable
`LINE`/`HLINE`/`VLINE`/`BOX`/`CIRCLE` helpers work in the five RAM
bitmap modes. Every portable and native name in the tables works with
both a literal `DISPLAY` call and a runtime mode value.

`DISPLAY_MIXED` supports `BITMAP_HIRES`, `BITMAP_LORES`, and
`BITMAP_MULTICOLOR`. Its text region can start on any of the 24 text
rows and can contain any number of rows that fit below it. Cell row
arguments remain absolute, so a region starting at row 10 uses rows 10
and onward. `CELL_CLS` clears only the active text region. Bitmap rows
behind that region are hidden, while bitmap drawing keeps its normal
coordinates. Passing zero text rows selects the requested bitmap mode
without a text region.

In a plain bitmap mode `PRINT` output is dropped. In `DISPLAY_MIXED`,
`PRINT` writes to the text region and starts at its first row.

## Tiles

The hardware TILE backend drives the full 320x192 display and is the
default, so the TILE API needs no backend option:

```basic
@OPTION TILE_ACTIVE_COUNT 16

@INCLUDE "api/tile.cbi"

PROC MAIN
	DISPLAY TILE_DEFAULT_DISPLAY_MODE
	TILE_BEGIN 1, 1, ADDR MY_TILES_ROW7
ENDPROC
```

With a `1, 1` span, `TILE_COLUMNS` is 40 and `TILE_ROWS` is 24. The
960-byte map stays in RAM. Tile graphics and other constant data stay
in cartridge ROM.

The tile bank starts on a 256-byte page and occupies eight pages.
MARIA needs the pages in this order:

| Page | Contents |
| --- | --- |
| 0 | Row 7 of tile IDs 0 through 255 |
| 1 | Row 6 of tile IDs 0 through 255 |
| 2 through 6 | Rows 5 through 1 |
| 7 | Row 0 of tile IDs 0 through 255 |

Pass the address of page 0 to `TILE_BEGIN`. `TILE_PRINT` writes ASCII
codes directly, so a custom bank that supports text must put glyphs in
their ASCII slots.

`TILE_ACTIVE_COUNT` sizes the portable tile tables at compile time. It
defaults to 32 on Atari 7800. Lower it when the program uses fewer tile
IDs to leave more RAM for game data. It does not change the 256 slots
available in the ROM tile bank. Embedded tilesets generate their required
active count automatically.

### PNG tile projects

Atari 7800 declares the `png_a7800_8x8` tile format. Import the pictures
and colors from an indexed PNG. Its placement map is generated automatically:

```basic
@INCLUDE_TILESET SCENE_TILES "scene.png" AS png_a7800_8x8

PROC MAIN
	OK AS U8

	OK = DISPLAY(MARIA_320A)
	TILE_BEGIN 1, 1
	TILE_BLIT ADDR SCENE_TILES_MAP_DATA, 0, 0, SCENE_TILES_MAP_WIDTH, SCENE_TILES_MAP_HEIGHT
ENDPROC
```

The PNG dimensions must be multiples of 8. Each tile has one background
and one foreground color. The importer maps the foregrounds to the eight
MARIA palette slots and keeps the color with each unique tile. Matching
8x8 pictures with matching colors are stored once and reused by the
generated `SCENE_TILES_MAP_*` data.

`TILE_BEGIN` binds the pictures and colors. `TILE_PLOT` and `TILE_BLIT`
apply the palette spans automatically in the hardware tile display. The
same imported tiles work with `TILE_BITMAP`; their colors are applied
through `TILE_COLOR`. All tiles shown together by the hardware backend
should use the same background because MARIA has one shared background
color.

Background tiles are one bit, so each palette span has one foreground
color plus the shared background. A tile row can contain several spans
using different MARIA palettes. Sprites use separate objects and can add
color over the same row.

`TILE_SPAN_PALETTE row, start_col, count, palette` selects one of the
eight MARIA palettes for a horizontal span. Row is 0 through 23, columns
are 0 through 39, and palette is 0 through 7. Later calls replace the
palette in overlapping columns. Set the palette colors first with
`SET_PALETTE`. This API is specific to the Atari 7800 `MARIA_320A`
hardware tile display.

Each palette run uses a five-byte MARIA object, and runs longer than 32
columns use more than one object. The row display list has 26 bytes for
tile and sprite objects, or 38 with more than four sprite slots; each sprite object uses four bytes. The program
is responsible for keeping the combined objects within that space. When
changing a visible row, call `FRAME_WAIT` first.

With more than four sprite slots, the hardware tile display uses
`$22D8-$241F` for additional display lists.
These bytes are unavailable for program data while that display is used.

### MARIA Tile Addresses

`MARIA_TILE_ROW_ADDR(row)` returns the address of the first tile code in
a row. Each row contains 40 tile codes.

`MARIA_TILE_DISPLAY_LIST_ADDR(row)` returns the address of the first
MARIA display list byte for a row. Use it when a program needs to build
or edit MARIA objects directly.

Both functions return `U16`. The row must be 0 through 23. Use them after
`DISPLAY MARIA_320A`. Call `FRAME_WAIT` before writing to a visible row
or its display list.

## Sprites

Sprites are 8x8 MARIA display-list objects. `SPRITES_ON` keeps an
active `MARIA_320A` tile display, or selects it when another mode is
active:

| Item | Value |
| --- | --- |
| Count | 4 by default, up to 7 (`SPRITE_MAX_COUNT`) |
| Size | 8x8 (`SPRITE_DATA_8X8`) |
| Position | X 0 through 312, Y 0 through 184 |
| X granularity | 2 pixels (`SPRITE_ALIGN_X`) |
| Palettes | 8 MARIA palettes (`SPRITE_PALETTE`, `SPRITE_PALETTE_SET`) |

`@OPTION SPRITE_COUNT N` selects 1 through 7 sprite slots.
Fewer slots leave more RAM for program data.

Updates are staged and pushed with `SPRITES_FLUSH` once per frame
(`SPRITE_NEEDS_VBLANK_FLUSH = TRUE`). Call `FRAME_WAIT` before the flush
so the display list changes happen during blanking.

`@OPTION SPRITE_DOUBLE_BUFFER TRUE` keeps a second copy of the display
lists. `SPRITES_FLUSH` can then run at any point in the frame, and its
changes appear at the next `FRAME_WAIT`, so call the flush before
`FRAME_WAIT`. A second flush before `FRAME_WAIT` waits for the next
frame. This uses about 1K more RAM unless the program also uses a bitmap
display. After a program calls `MARIA_TILE_DISPLAY_LIST_ADDR`, sprites
update the one copy in place.

`SPRITE_COLOR`
gives each portable sprite its own palette by default, and
`SPRITE_PALETTE_SET` fills all three colors of a MARIA palette with `c0`
because a 320A sprite pixel has one color. MARIA has no
hardware collision registers. `SPRITE_HIT` checks the visible 8x8
sprite rectangles in software, while `SPRITE_HIT_BG` returns 0.
`SPRITE_PRIORITY id, 1` places that sprite behind the tile background;
zero places it in front. MARIA supplies this priority through display
list order. `SPRITE_FLIP_*` and `SPRITE_EXPAND` are accepted but do
nothing on this target.

## Raster Interrupts

MARIA display-list interrupts drive the portable raster helpers:

- `RASTER_MARK_ROW row` / `RASTER_MARK_ROWS start, count, step` flag
  display rows to interrupt. Rows count from the first visible row: a
  text row in the text and `MARIA_320A` displays, a bitmap row in the
  bitmap displays.
- `RASTER_LINE_INSTALL ADDR(hook)` runs the hook at each marked row;
  `RASTER_INDEX` counts hook calls within the frame.
  `RASTER_LINE_INSTALL ADDR(hook), count` supplies the number of marked
  rows per frame when using a custom display list with interrupt flags
  already set.
- `RASTER_FRAME_INSTALL ADDR(hook)` runs the hook once per frame, at the
  first marked row, or from the bottom blank zone when no row is marked.
- `RASTER_STOP` stops the hooks but keeps their addresses and marked rows.
- `RASTER_START` resumes a stopped raster program.
- `RASTER_UNINSTALL` stops the hooks and clears their setup.
- `RASTER_WSYNC` waits for the current scanline to finish.

Hooks are no argument procedures called from the NMI; keep them short.
`DISPLAY_MIXED` drives its text region with the line hook and two marked
rows, so only the frame hook is free in that display.

## Sound

Two TIA voices are available. Portable `PLAY_NOTE`, `PLAY_NOTE_SHAPE`,
`PLAY_NOTE_FOR`, and `PLAY_NOTE_SHAPE_FOR` map MIDI notes 60-95 through
a TIA tuning table. `BEEP`, `SOUND_OFF voice`, and `SOUND_ALL_OFF` are
also available.

Exact TIA effects use:

```basic
TIA.SOUND VOICE, AUDF, AUDC, VOLUME
```

This writes `AUDF`, `AUDC`, and `AUDV` directly. `AUDC` 4 is tonal;
8 and 12 provide noise and buzz variants. TIA pitch resolution is
coarse, so intervals may sound detuned compared with POKEY or SID.

## Input

`JOY(port)` reads a digital stick. `JOY0` is the short form for port 0
and `JOY1` is the short form for port 1. It returns a mask containing
`JOY_UP`, `JOY_DOWN`, `JOY_LEFT`, and `JOY_RIGHT`. Use the named masks
instead of numeric values. A mask check also works when two directions
are pressed together:

```basic
IF JOY0 & JOY_UP THEN SELECTED = 1
IF JOY0 & JOY_DOWN THEN SELECTED = 2
```

`JOY_BUTTON(port, button)` returns 1 when a button is pressed. Button 0
is the right 7800 button and button 1 is the left button. `FIRE0` is the
short form for port 0, button 0. `FIRE1` is the short form for port 1,
button 0:

```basic
IF FIRE0 THEN CHOICE = SELECTED
```

Use `JOY_BUTTON(0, 1)` for the second button on port 0. If a
2600-compatible one-button stick is detected, that port automatically
falls back to its single fire button. `STRIG` is an Atari 8-bit computer
call and is not part of the Atari 7800 input API.

`CONSOLE_SWITCHES` returns an active-high mask: bit 0 is Start, bit 1 is Select,
bit 3 is Pause, and bits 6 and 7 are the two difficulty switches.
`CONSOLE_START`, `CONSOLE_SELECT`, and `CONSOLE_PAUSE` return their
individual mask bits. `CONSOLE_P0_DIFFICULTY` and
`CONSOLE_P1_DIFFICULTY` return `CONSOLE_DIFFICULTY_A` (`1`) or
`CONSOLE_DIFFICULTY_B` (`0`).

`PADDLE(0)` through `PADDLE(3)` return raw positions from 0 through
255. `PTRIG(0)` through `PTRIG(3)` return the raw trigger state: 0 is
pressed and 1 is released. The first paddle or paddle-trigger read
switches both controller ports out of two-button pad mode. Do not mix
paddles and two-button pads in the same program.

## Timing

`FRAME_WAIT` synchronizes to MARIA vertical blank via `MSTAT` and
increments `FRAME_COUNTER`. `FRAME_INSTALL ADDR(HOOK)` installs a
no-argument hook that runs once at the end of each `FRAME_WAIT`;
`FRAME_UNINSTALL` removes it. This is a regular call, not an interrupt.
The `TICKS` API counts frames. `RAND n` returns fast LFSR-based random
bytes.

## Images

Native image display supports:

| `AS` format | Image constant | Size | Colors |
| --- | --- | --- | --- |
| `a7800-160a` | `IMAGE_FMT_A7800_160A` | 124x80 | 4 |
| `a7800-160b` | `IMAGE_FMT_A7800_160B` | 62x80 | 13 |
| `a7800-320a` | `IMAGE_FMT_A7800_320A` | 248x80 | 2 |
| `a7800-320b` | `IMAGE_FMT_A7800_320B` | 124x80 | 4 |
| `a7800-320c` | `IMAGE_FMT_A7800_320C` | 124x80 | 9 |
| `a7800-320d` | `IMAGE_FMT_A7800_320D` | 248x80 | 2 |
| `a7800-multi` | `IMAGE_FMT_A7800_MULTI` | 124x48 | 4 |

An indexed PNG or PCX must name its layout:

```basic
@INCLUDE_IMAGE TITLE "title.png" AS a7800-320c
```

The selected layout is the only one stored in the image descriptor.
`IMAGE_LOAD` selects the matching MARIA display and loads its pixels,
row palettes, and colors. The six full bitmap layouts have a 2545 byte
payload before optional RLE8 packing.

`a7800-multi` remains the compact 124x48 layout. `cb-image` also accepts
its 1488 byte raw screen data or 1492 byte screen and color data. Runtime
image file loading is not available.

All image layouts support wipes, slides, palette fades, and dissolves.
Fades use 16 brightness steps; dissolves use 1, 2, 4, or 8 pixel blocks.
Effects use unpacked images.

## Assembler

The configured assembler is `vasm6502_oldstyle` through `vasm`. The
build tool signs the raw cartridge bytes and wraps them as `.a78` (v3
header) using the `vasm_atari7800_a78_32k` recipe.
