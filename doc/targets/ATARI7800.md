# crustyBASIC on Atari 7800

Use `@OPTION TARGET atari7800` for Atari 7800 cartridges. The system
name is `atari7800.orig`. Output is a 32K `.a78` cartridge image.

Use `@OPTION OUTPUT_TYPE cart-32k-ram` for a 32K cartridge with 16K
RAM at `$4000-$7FFF`. The cartridge image selects SuperGame RAM hardware.
Use `@OPTION MEMORY_REGION __cb_atari7800_cart_ram` to place program
data in this RAM.

Use `cart-32k-pokey` or `cart-32k-ram-pokey` for POKEY sound at `$0450`.
The RAM variant supports the same graphics and sprite features as
`cart-32k-ram`. Portable sound calls use the selected chip; `TIA.SOUND`
always accesses TIA directly.

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
| Sound | 2 TIA voices or 4 POKEY voices |
| Recursive PROCs | Yes |

The target provides MARIA text, portable CELL text access,
palette/background control, frame hooks, joystick, console, and paddle
input, random bytes, sound, RAM bitmap graphics, a full-height
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

Using `TICKS` or `@PERIODIC` reserves `$0140-$017D` for the clock.

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

`DISPLAY CELL` selects this screen. `DISPLAY MARIA_320A` selects the
40x24 MARIA tile text grid.

With `@OPTION OUTPUT_TYPE cart-32k-ram`, the CHARSET API replaces the
normal text font in `$6800-$6FFF`. Fonts contain 128 glyphs of 16 bytes
each: two bytes per row, from top to bottom, with four 2-bit pixels per
byte. `@INCLUDE_CHARSET` prepares this layout automatically.
`CHARSET_COPY_DEFAULT` activates an editable copy of the built-in font;
`CHARSET_RESET` selects the built-in font again. The selected font also
serves mixed text bands and persists across display mode changes.

Text glyphs use even cell codes, 0 through 254. Use `CELL_CODE` in text
mode to find a printable character's code. `CHARSET_INSTALL_RANGE`
advances by two codes for each glyph. The TILE API uses its own font bank.

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

Mixed text uses palette 7. Bitmap drawing uses palettes 0 through 6,
so text color changes preserve bitmap colors.

In a plain bitmap mode `PRINT` output is dropped. In `DISPLAY_MIXED`,
`PRINT` writes to the text region and starts at its first row.

## Scrolling

With `@OPTION OUTPUT_TYPE cart-32k-ram`, `SCROLL_SETUP(SCROLL_TYPE_SW)`
selects scrolling in every display mode. `SCROLL_MOVE` takes signed
pixel distances; positive X moves the contents left and positive Y
moves them up. Read the granularity values after starting scrolling.
`SCROLL_BEGIN` uses these steps:

| Display | Horizontal step | Vertical step |
| --- | --- | --- |
| `CELL`, `CELL_MULTICOLOR`, portable bitmap modes | 1 pixel | 1 pixel |
| `MARIA_160A`, `MARIA_160B`, `MARIA_320B`, `MARIA_320D` | 1 pixel | 1 pixel |
| `MARIA_320A` tiles, `MARIA_320C` | 2 pixels | 1 pixel |

Scrolling carries palettes with the picture. The two-pixel steps keep
MARIA tile positioning and 320C color pairs aligned. In mixed displays,
the text band stays fixed. Each bitmap pixel row occupies two scanlines.

`SCROLL_BEGIN_REGION x, y, columns, rows, axes` selects a rectangle in
the display's cell grid. Buffer coordinates are relative to that rectangle.
Text regions move in 8-pixel steps and show the whole selected rectangle.
Bitmap regions use the pixel steps above. Partial-width text and bitmap
regions keep their shared row palettes in place; 320A tile regions carry
each cell's palette with its character. Cells and pixels outside the
rectangle stay fixed.

For `SCROLL_BEGIN` and bitmap regions, each enabled axis reserves one
backing cell beyond the reported view.
Use `SCROLL_CELL_PUTC` or `SCROLL_CELL_COLUMN` to refill the edges reported
by `SCROLL_MOVE`, then call `SCROLL_PRESENT`. The optional foreground
argument selects a palette number in the 320A tile display and a color
in multicolor bitmap modes. One-bit bitmaps keep the row's foreground;
cell text uses its shared foreground. `SCROLL_CLOSE` shows the complete
backing rectangle at a cell boundary. Scrolling reserves cartridge RAM at
`$5000-$5A4F`. Bitmap scrolling copies pixels and can take several frames.

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
960-byte map stays in RAM. Constant tile banks stay in cartridge ROM.

With `@OPTION OUTPUT_TYPE cart-32k-ram`, `TILE_DEFINE` copies an 8x8
pattern from eight bytes into cartridge RAM. Define the same ID again
to update its displayed pattern. The writable bank occupies `$6000-$67FF`
and preserves the starting font's other glyphs. Runtime patterns use
slots 128 through 255, or 128 through 191 when software sprites are enabled.
`TILE_NATIVE_MAX_COUNT` reports this capacity.

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

PNG tiles use the `a7800` output format by default. `TILE_SIZE` gives the
logical picture dimensions; each picture is packed as 8x8 pieces with their
foreground colors. Import placement separately:

```basic
@INCLUDE_TILESET SCENE_TILES "scene.png" TILE_SIZE 16 16
@INCLUDE_TILEMAP SCENE_MAP "scene.cbm" USING SCENE_TILES

PROC MAIN
	OK AS U8

	OK = DISPLAY(MARIA_320A)
	TILE_BEGIN SCENE_TILES_SPAN_W, SCENE_TILES_SPAN_H
	SCENE_TILES_INSTALL
	TILE_BLIT ADDR SCENE_MAP_DATA, 0, 0, SCENE_MAP_WIDTH, SCENE_MAP_HEIGHT
ENDPROC
```

The sheet dimensions divide evenly into its logical tiles. Tile IDs keep
sheet order, including repeated pictures. Matching native patterns may share
storage. Foregrounds are chosen from the artwork for the eight MARIA
palette slots. Every 8x8 piece keeps a three byte record: palette ID,
foreground, and background.

`INSTALL` binds pictures and colors. `TILE_PLOT` and `TILE_BLIT` apply palette
spans in the hardware tile display. Tiles shown together share one background
color. `FORMAT mono` selects single color pictures.

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

IDs 0 through 6 are 8x8 MARIA display-list objects. Select the `MARIA_320A`
tile display with `DISPLAY MARIA_320A` before `SPRITES_ON`. [Sprites on bitmap and cell displays](#sprites-on-bitmap-and-cell-displays)
describes the option that keeps other displays:

| Item | Value |
| --- | --- |
| MARIA objects | 4 by default, up to 7 (`SPRITE_HW_COUNT`) |
| Source size | 8x8 (`SPRITE_DATA_8X8`) |
| Position | X 0 through 312, Y 0 through 184 |
| X granularity | 2 pixels (`SPRITE_ALIGN_X`) |
| Palettes | 8 MARIA palettes (`SPRITE_PALETTE`, `SPRITE_PALETTE_SET`) |

`@OPTION SPRITE_COUNT N` selects the total sprite slots. Values 1 through 7
allocate MARIA objects; larger values add software sprites from ID 7.
Fewer MARIA slots leave more RAM for program data.

With `@OPTION OUTPUT_TYPE cart-32k-ram`, `SPRITE_EXPAND` doubles either
axis of a MARIA sprite, giving an 8x8, 16x8, 8x16, or 16x16 rectangle.
The source data stays 8x8. Expanded patterns use `$7000-$7FFF` and follow
the display-list buffering setting. Keep the displayed rectangle inside
the 320x192 surface. Expansion persists when data changes and clears with
`SPRITES_RESET`. The per-ID expansion queries report support.

For MARIA objects on RAM cartridges, `SPRITE_MODE id, SPRITE_MODE_MULTICOLOR`
selects three colors plus transparency. `SPRITE_DATA` takes eight bytes,
one per row, with four two-bit values from left to right. Each value covers
two screen pixels: 0 is transparent, and 1 through 3 select `c0`, `c1`, and
`c2` from `SPRITE_PALETTE_SET`. Patterns use `$7000-$7FFF`.
Mirroring and expansion apply to this data.
`SPRITE_DATA_8X8` selects the one-bit hires format.

Multicolor sprites share palette group 0 or 4, selected with `SPRITE_PALETTE`.
Group 0 uses MARIA palettes 0 through 2; group 4 uses palettes 4 through 6.
Palettes 3 and 7 remain available for independent hires colors.
`SPRITE_COLOR` changes the group's `c0`, preserving `c1` and `c2`.
`SPRITE_COLOR_SHARED` reports this sharing, and `SPRITE_CAN_SET_MODE`
reports support for each ID.

On `MARIA_320A`, sprites use double buffering by default. Call `GFX_SWAP`
or `SPRITES_FLUSH` after moving the sprites, then `FRAME_WAIT` to show
the update. A second flush before `FRAME_WAIT` waits for the next frame.
The second copy of the display lists uses about 1K more RAM unless the
program also uses a bitmap display.

Use `@OPTION SPRITE_DOUBLE_BUFFER FALSE` to save that RAM. In this mode,
call `FRAME_WAIT` before the flush to update the visible list during
blanking. Calling `MARIA_TILE_DISPLAY_LIST_ADDR` also switches sprite
updates to the visible list.

`SPRITE_COLOR` follows the flush timing above and gives each portable
sprite its own palette by default. `SPRITE_PALETTE_SET` sets three
shared palette colors; 320A sprites use `c0`. MARIA has no
hardware collision registers. `SPRITE_HIT` checks the visible sprite
rectangles, including expansion. `SPRITE_HIT_BG` tests the sprite rectangle
against occupied 8x8 tile cells. A cell is occupied when its glyph contains
set pixels. `SPRITE_HIT_BG_MODEL` is `SPRITE_HIT_BG_MODEL_CELL`.
`SPRITE_PRIORITY id, 1` places that sprite behind the tile background;
zero places it in front. MARIA supplies this priority through display
list order. `SPRITE_FLIP_X` and `SPRITE_FLIP_Y` mirror MARIA objects and
software sprites horizontally and vertically. A nonzero flag enables
the flip; zero restores the original orientation.

### Software sprites

MARIA objects and software sprites can be used together through the
[shared SPRITE API](../API.md#hardware-and-software-sprites).

Use `@OPTION OUTPUT_TYPE cart-32k-ram` and `@OPTION SPRITE_COUNT N` with
`N` above 7. IDs 0 through 6 use MARIA objects; IDs 7 through `N - 1`
draw into the `MARIA_320A` tile display through the same SPRITE calls.
`SPRITE_HW_COUNT` is 7 and `SPRITE_KIND` is `SPRITE_KIND_MIXED`.

`SPRITE_DATA id, width, height, address` takes pixel dimensions and a
row-major, one-bit bitmap, high bit first. Each row uses
`(width + 7) / 8` bytes. The two-argument form and `SPRITE_DATA_8X8` use
8x8 data. `SPRITE_DATA_TILES` takes dimensions in 8x8 tiles, stored
left to right, then top to bottom.

`SPRITE_EXPAND id, x_double, y_double` doubles each selected axis for
software sprites. `SPRITE_CAN_EXPAND_X(id)` and `SPRITE_CAN_EXPAND_Y(id)`
report support for that ID. Expansion applies to drawing and collisions,
persists when image data changes, and clears with `SPRITES_RESET`.

Software positions move by pixels. Keep each positive-size
shape, including expansion, inside the 320x192 surface. Visible software
sprites can cover 64 cells in total, including overlapping cells. Glyphs 192 through 255
are reserved for their composed images. Cartridge RAM `$4000-$47FF`
holds the writable font; `$4800-$487F` holds saved cell codes and palettes.
Palette and sprite updates use `$4880-$48A6`.
Update buffers occupy `$48A8-$4E67` and `$5B00-$5F7F`.

`SPRITE_CAN_SET_SIZE(id)` is true for software IDs. Hardware IDs use 8x8 data.

`SPRITES_FLUSH` restores the old cells and draws behind sprites first,
then foreground sprites, in ID order within each group. It prepares the
update while the current image remains visible, then blanks the display
while copying the result. Software changes appear during the flush,
including when `SPRITE_DOUBLE_BUFFER` stages MARIA object changes for the
next frame. `SPRITES_RESTORE` restores the
background before drawing or scrolling it; the next flush redraws the
sprites. Hiding, resetting, disabling sprites, and changing display mode
also restore covered cells.

`SPRITE_COLOR` selects shared palette 0 and sets its foreground color.
`SPRITE_PALETTE` selects one of the eight shared MARIA palettes. Tile
palette spans share display-list space with MARIA objects.
Software sprites share each 8x8 cell's foreground palette. Cells with
visible background pixels keep the background palette; other cells with
sprite pixels use the sprite palette. `SPRITE_BG` changes the shared backdrop.

`SPRITE_PRIORITY id, 1` keeps the tile background's foreground pixels
in front of the sprite. Zero selects foreground drawing.
`SPRITE_CAN_SET_PRIORITY(id)` is true for both sprite renderers.
`SPRITE_HIT` checks rectangles among visible software sprites.
`SPRITE_HIT_BG` uses the original background beneath drawn sprites and
the current sprite rectangle. Hidden sprites return zero.

Known IDs select the direct handler in speed builds. A variable ID uses
the HW/SW dispatcher. `BIND PROC` can select
`__cb_atari7800_sprite_hw_move` or `__cb_atari7800_sprite_sw_move` directly
for variable IDs; the other per-sprite handlers follow the same names.

### Sprites on bitmap and cell displays

With `@OPTION ATARI7800_SPRITE_DISPLAYS TRUE` and a RAM cartridge,
`SPRITES_ON` keeps the current display when it is `BITMAP_HIRES`,
`BITMAP_LORES`, `BITMAP_MULTICOLOR`, `MARIA_160A`, `MARIA_160B`, `CELL`,
or `CELL_MULTICOLOR`. Every sprite ID, software IDs included, then shows
as a MARIA object over the display. Other displays, and cartridges
without RAM, need `MARIA_320A` as described above. After `SPRITES_ON`,
a change to another of these displays carries the sprites to it at the
next `SPRITES_FLUSH`.

Positions start at 0 at the top left of the display and count display
pixels. Sprite pixels match the display's pixels, so sprites are wider on
160 pixel displays and each bitmap row is two scan lines tall. Hardware IDs
keep their 8x8 data, multicolor mode, expansion, and flips; software IDs
keep their sizes. Palettes work as above.

`SPRITE_PRIORITY id, 1` places the sprite behind the display's set
pixels. `SPRITE_HIT_BG` reports set bitmap pixels, or cells that hold a
character, under the sprite rectangle. Each bitmap row or cell row shows
up to six sprites. Changes appear at `SPRITES_FLUSH`; call `FRAME_WAIT`
first.

Cartridge RAM `$7000-$7FFF` holds the sprite display lists and images.
Images share about 1K of it; on cell displays visible sprites also share
256 bytes of image width. A sprite without room is not shown.

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
  first marked row, or from the top blank zone when no row is marked.
- `RASTER_STOP` stops the hooks but keeps their addresses and marked rows.
- `RASTER_START` resumes a stopped raster program.
- `RASTER_UNINSTALL` stops the hooks and clears their setup.
- `RASTER_WSYNC` waits for the current scanline to finish.

Use `SCREEN_OFF` while building a custom display list, then
`DISPLAY_SHOW` to display it.

Hooks are no argument procedures called from the NMI; keep them short.
`DISPLAY_MIXED` drives its text region with the line hook and two marked
rows, so only the frame hook is free in that display.

## Sound

Default cartridges use two TIA voices with MIDI notes 60-95. POKEY
cartridges use four voices with MIDI notes 48-95. Portable `PLAY_NOTE`,
`PLAY_NOTE_SHAPE`, `PLAY_NOTE_FOR`, and `PLAY_NOTE_SHAPE_FOR` use the
selected chip. Both provide volume levels 0-15. `BEEP`, `SOUND_OFF voice`,
and `SOUND_ALL_OFF` use that chip too.

POKEY uses pure tones for the tone, pulse, triangle, and saw selectors;
`SOUND_SHAPE_NOISE` selects noise. Its register interface includes
`POKEY.AUDF1` through `POKEY.AUDF4`, `POKEY.AUDC1` through `POKEY.AUDC4`,
`POKEY.AUDCTL`, `POKEY.STIMER`, `POKEY.SKCTL`, and `POKEY.RANDOM`.

`PLAY_NOTE_SHAPE` selects the closest pitch in the requested shape's range.

`BEEP duration` uses divider 10. `BEEP period, duration` selects
a divider from 0 through 31 on TIA, or 0 through 255 on POKEY.
Duration is in video frames.

`SOUND_TIMED_NOTE` and `SOUND_TIMED_NOTE_SHAPE` schedule notes on any
voice and return immediately. Notes and timed rests advance once per
`FRAME_WAIT`. `SOUND_TIMED_BUSY` and `SOUND_TIMED_REMAINING` report each
voice's state. `SOUND_TIMED_OFF` cancels a voice; `SOUND_TIMED_UNINSTALL`
stops all selected voices and the service.

`SOUND_ENVELOPE` selects attack, decay, sustain and release settings, or
a table of duration and level pairs, for the next note on a voice.
Envelopes advance once per `FRAME_WAIT`. `SOUND_RELEASE` starts the release
stage; `SOUND_OFF` and note duration limits stop playback immediately.

Exact TIA effects use:

```basic
TIA.SOUND VOICE, AUDF, AUDC, VOLUME
```

This writes `AUDF`, `AUDC`, and `AUDV` directly. `AUDC` 4 and 12 select
different tone ranges; 8 selects noise. TIA pitch resolution is
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
255. `PADDLE PORT, X, Y` samples both axes of a port in one charge cycle.
`PADDLE X0, Y0, X1, Y1` samples all four axes together. These output
variables use `U16`. `PADDLE_BUTTON(0)` through `PADDLE_BUTTON(3)` return 1 when pressed
and 0 when released. The first paddle or paddle-trigger read
switches both controller ports out of two-button pad mode. Do not mix
paddles and two-button pads in the same program.

## Timing

`FRAME_WAIT` synchronizes to MARIA vertical blank via `MSTAT` and
increments `FRAME_COUNTER`. `FRAME_INSTALL ADDR(HOOK)` installs a
no-argument hook that runs once at the end of each `FRAME_WAIT`;
`FRAME_UNINSTALL` removes it. This is a regular call, not an interrupt.
`TICKS` returns a 32-bit elapsed frame count at 60 Hz, independent of
`FRAME_WAIT`. It keeps advancing during display changes and while the
screen is hidden or off. `TICKS_RESET` resets elapsed time without
changing `@PERIODIC` timing. `RAND n` returns fast LFSR-based random
bytes.

## Images

Native image display supports:

| `AS` format | Image constant | Size | Colors |
| --- | --- | --- | --- |
| `a7800-160a` | `IMAGE_FMT_A7800_160A` | 124x80 | up to 25, 4 per row |
| `a7800-160b` | `IMAGE_FMT_A7800_160B` | 62x80 | 13 |
| `a7800-320a` | `IMAGE_FMT_A7800_320A` | 248x80 | 2 |
| `a7800-320b` | `IMAGE_FMT_A7800_320B` | 124x80 | 4 |
| `a7800-320c` | `IMAGE_FMT_A7800_320C` | 124x80 | 9 |
| `a7800-320d` | `IMAGE_FMT_A7800_320D` | 248x80 | 2 |
| `a7800-multi` | `IMAGE_FMT_A7800_MULTI` | 124x48 | 4 |

A PNG or PCX must name its layout:

```basic
@INCLUDE_IMAGE TITLE "title.png" AS a7800-320c
```

The selected layout is the only one stored in the image descriptor.
RGB colors map to the full 256-color NTSC palette. The 160A converter
shares up to eight foreground palettes across rows with one common background.
`IMAGE_LOAD` selects the matching MARIA display and loads its pixels,
row palettes, and colors. The six full bitmap layouts have a 2545 byte
payload before optional RLE8 packing.

`a7800-multi` remains the compact 124x48 layout. `cb-image` also accepts
its 1488 byte raw screen data or 1492 byte screen and color data. Runtime
image file loading is not available.

All image layouts support wipes, slides, palette fades, and dissolves.
Fades use 16 brightness steps; dissolves use 1, 2, 4, or 8 pixel blocks.
Effects accept unpacked and RLE8 images.

## Assembler

The configured assembler is `vasm6502_oldstyle` through `vasm`. The
build tool signs the raw cartridge bytes and wraps them as `.a78` (v3
header) using the `vasm_atari7800_a78_32k` recipe.
