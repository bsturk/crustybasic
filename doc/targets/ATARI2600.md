# crustyBASIC on Atari 2600

Use `@OPTION TARGET atari2600` for Atari 2600 / VCS cartridges. The
system name is `atari2600.orig`. Output is a raw `.a26` cartridge image.

The Atari 2600 has no operating system, keyboard, text screen, or frame
buffer. It has 128 bytes of RAM and draws the display one scanline at a
time. Keep data small and draw or wait for one frame on every game loop.

Portable runtime calls are documented in [`../API.md`](../API.md). This
page covers Atari 2600 display features, options, and hardware limits.

## Basics

| Item | Value |
| --- | --- |
| CPU | 6507, 6502 family |
| Output | `.a26` |
| Code start | `$F000` for 4K carts |
| RAM | `$80-$FF`, 128 bytes total |
| User data | `$A4-$F3`, 80 bytes shared with display state |
| Timing | NTSC 262 scanlines, or PAL 312 with `@OPTION REGION REGION_PAL` |
| `REAL` | Not supported |
| String encoding | ASCII |
| Recursive PROCs | Yes |
| Optimize | `default` or `speed` |

The CPU stack shares the same 128 bytes of RAM. Keep globals, arrays,
and strings tiny. A plain `STRING` uses 8 bytes and holds seven
characters. The display features a program selects take their state from
the same 80 bytes, so the RAM table further down matters.

The byte `LFSR` generator shares its state with `RAND` to save RAM. Seed it
after any `RAND` calls when a repeatable sequence matters.

## Choose a Display

Three facts pick the display kernel at compile time: how wide the
playfield is, how many sprites the program wants, and whether it draws
its own frames.

```basic
@OPTION TARGET atari2600
@OPTION A2600_PLAYFIELD_COLUMNS 20
@OPTION A2600_PLAYFIELD_ROWS 16
@OPTION A2600_PLAYFIELD_RAM_ROWS 16
@OPTION SPRITE_COUNT 2
```

| Option | Values | Default |
| --- | --- | --- |
| `A2600_PLAYFIELD_COLUMNS` | `20`, `40` | `20` |
| `SPRITE_COUNT` | `0`, `2`, `4`, `6` | `0` |
| `A2600_CUSTOM_KERNEL` | `TRUE`, `FALSE` | `FALSE` |

| Columns | Sprites | Kernel |
| --- | --- | --- |
| 20 | 0 or 2 | Playfield kernel: rows in RAM or a static playfield, two pixel sprites, missiles, ball, information rows, per row colors, timed paddles |
| 40 | 0 | Wide kernel: 40 unique cells per row, a RAM row window with ROM fill rows, the ball, and row sized playfield players |
| 20 | 4 or 6 | Multisprite kernel: a static playfield and reused sprites |
| any | 0 | `A2600_CUSTOM_KERNEL TRUE`: the program draws its own frames |

Everything else is an option on the chosen kernel. Combinations the
kernel cannot draw fail during preprocessing with a message that names
the options. `KERNEL_PROFILE` reports the chosen kernel for `@IF` blocks:
`KERNEL_NONE`, `KERNEL_PLAYFIELD`, `KERNEL_WIDE`, or `KERNEL_MULTISPRITE`.

With no display options the program gets the playfield kernel with a
static playfield: `PLAYFIELD PF0, PF1, PF2` sets bits that repeat on every
row, and missiles, the ball, and collision reads are available. TILE,
playfield text, images, and CELL need playfield rows in RAM. Set
`A2600_PLAYFIELD_RAM_ROWS` to use them; a program that calls them without
RAM rows fails to assemble on the symbol
`TILE_NEEDS_A2600_PLAYFIELD_RAM_ROWS`.

## Rows

Both playfield kernels split the 192 visible scanlines into logical rows.

| Option | Meaning | Default |
| --- | --- | --- |
| `A2600_PLAYFIELD_ROWS` | Logical row count | `8` |
| `A2600_PLAYFIELD_ROW_SCANLINES` | Lines per row | `192 \ ROWS`, always `8` with 8 pixel cells |
| `A2600_PLAYFIELD_RAM_ROWS` | Rows kept in RAM | `0` on 20 columns, every row on 40 |
| `A2600_PLAYFIELD_RAM_ROW_FIRST` | First RAM row, 40 columns only | `0` |

Any row count works while rows times lines stays at or under 192. A
shorter picture is centered with blank lines above and below. Rows are
at least 2 lines tall, 5 with missiles or the ball, and 6 with two
sprites.

Logical rows carry the per row colors, the missile and ball rows, and
the information rows, so a static playfield can still have 24 rows for
those. The `PLAYFIELD_ROWS`, `PLAYFIELD_ROW_SCANLINES`,
`PLAYFIELD_RAM_ROWS`, `PLAYFIELD_RAM_ROW_FIRST`, `PLAYFIELD_COLUMNS`, and
`PLAYFIELD_BLANK_SCANLINES` constants report the resolved geometry.

## The 20 Column Kernel

Rows `0` to `A2600_PLAYFIELD_RAM_ROWS - 1` keep three playfield bytes
each in RAM and are the TILE surface. Rows outside that window draw the
static `PLAYFIELD` bytes. `A2600_PLAYFIELD_LAYOUT` chooses how the right
half of every row is drawn:

| Layout | Surface |
| --- | --- |
| `PLAYFIELD_REPEAT` | 20 cells repeated on the right |
| `PLAYFIELD_REFLECT` | 20 cells mirrored on the right, the default |

```basic
@OPTION TARGET atari2600
@OPTION A2600_PLAYFIELD_ROWS 16
@OPTION A2600_PLAYFIELD_RAM_ROWS 16
@OPTION SPRITE_COUNT 2

PROC MAIN
	TILE_BEGIN 1, 1
	TILE_CLS
	TILE_BOX 0, 0, TILE_COLUMNS - 1, TILE_ROWS - 1, 1

	WHILE 1
		FRAME_WAIT 1
	ENDWHILE
ENDPROC
```

Two sprites draw with the rows: 8, 16, or 32 line shapes positioned to
the pixel, see [Portable Sprites](#portable-sprites). Missiles and the
ball are row objects positioned to the pixel with `MISSILE0_X X`,
`MISSILE1_X X`, and `BALL_X X`, and shown on whole rows.
`A2600_OBJECT_ROWS` picks how the rows are chosen for all three:

| Mode | Calls | Shows on |
| --- | --- | --- |
| `OBJECT_ROWS_SINGLE`, the default | `MISSILE0_ROW ROW`, `MISSILE0_Y Y`, `MISSILE0_ENABLE FLAG` | one row |
| `OBJECT_ROWS_RANGE` | `MISSILE0_ROWS FIRST, COUNT`, `MISSILE0_HIDE` | `COUNT` rows from `FIRST` |
| `OBJECT_ROWS_GAP` | `MISSILE0_GAP FIRST, COUNT`, `MISSILE0_HIDE` | every row but those |

The same calls exist for missile 1 and the ball (`BALL_ROW`, `BALL_Y`,
`BALL_ENABLE`, `BALL_ROWS`, `BALL_GAP`, `BALL_HIDE`). A gap gated ball
or missile is a wall with a hole in it. Range modes cost one more RAM
byte per object.
The objects change on the second to fourth line of a row, so they draw
from the row's middle lines on.

Per row colors and heights are RAM tables filled at run time:

| Option | Calls |
| --- | --- |
| `A2600_PLAYFIELD_COLORS PLAYFIELD_COLORS_PER_ROW` | `PLAYFIELD_ROW_COLOR ROW, C` |
| `A2600_BACKGROUND_COLORS BACKGROUND_COLORS_PER_ROW` | `BACKGROUND_ROW_COLOR ROW, C` |
| `A2600_PLAYER_COLORS PLAYER_COLORS_PLAYER1_PER_ROW` or `PLAYER_COLORS_PER_ROW` | `PLAYER1_ROW_COLOR ROW, C`, `PLAYER0_ROW_COLOR ROW, C` |
| `A2600_PLAYFIELD_HEIGHTS PLAYFIELD_HEIGHTS_PER_ROW` | `PLAYFIELD_ROW_HEIGHT ROW, H` |

With per row heights, make the row heights total 192 scanlines. Rows
left unset use the default height, and every row needs at least the
minimum height `A2600_PLAYFIELD_ROW_SCANLINES` accepts. Per row
colors next to two sprites need the rows to fill the 192 lines exactly.

## The 40 Column Kernel

`A2600_PLAYFIELD_COLUMNS 40` draws 40 unique cells per row with no
sprites. `A2600_PLAYFIELD_CELL_PIXELS` picks the cell width:

| Cells | Rows | Row storage | Objects |
| --- | --- | --- | --- |
| `4` pixels, the default | Any count, at least 2 lines each | 6 bytes per RAM row | Ball, one playfield player |
| `8` pixels | Up to 24 rows of 8 lines | 2.5 bytes per row, every row in RAM | Two playfield players, a score row |

With 4 pixel cells every row starts with one setup line that shows only
the background color, so a 12 line row shows 11 lines of cells. The RAM
row window can sit anywhere: `A2600_PLAYFIELD_RAM_ROW_FIRST F` and
`A2600_PLAYFIELD_RAM_ROWS K` keep rows `F` to `F + K - 1` in RAM. Rows
above and below the window draw fixed ROM rows when
`A2600_PLAYFIELD_FILL_ROW_ABOVE` or `A2600_PLAYFIELD_FILL_ROW_BELOW` is
`TRUE`, and nothing otherwise. The program then declares the matching
six byte `CONST` array in register order:

```basic
@OPTION A2600_PLAYFIELD_FILL_ROW_ABOVE TRUE

' PF0, PF1, PF2 for the left half, then the right half. PF0 draws its
' high nibble with bit 4 at the left, PF1 draws bit 7 first, and PF2
' draws bit 0 first
CONST PLAYFIELD_FILL_ROW_ABOVE(5) AS U8 = $F0, $FF, $FF, $F0, $FF, $7F
```

`A2600_PLAYFIELD_ROW_COLORS TRUE` colors every row from a `CONST`
table with one entry per row:

```basic
CONST PLAYFIELD_ROW_COLORS(15) AS U8 = ORANGE, RED, ORANGE, YELLOW, ...
```

`TILE_PLOT` and `TILE_GET` address rows by their logical row number; a
plot on a row outside the RAM window is undefined. The ball uses the
same `BALL_X`, `BALL_ROW`, `BALL_Y`, `BALL_ENABLE`, and `BALL_SIZE` calls
as the 20 column kernel.

Playfield players are the 40 column kernel's moving objects: quad width
players, 32 pixels wide, positioned to the pixel once per frame and
drawn on whole rows. `A2600_PLAYFIELD_PLAYERS 1` or `2` enables them,
and `A2600_PLAYER_ROWS N` sets how many rows a player covers. With 4
pixel cells the second player takes a second setup line per row, so
rows are then at least 3 lines and show two lines less of cells. Draw a
frame with the players through the frame call instead of `FRAME_WAIT`;
the call also advances `FRAME_COUNTER`:

```basic
PLAYFIELD_FRAME_OVERLAY X, ROW, ADDR MASKS, OFFSET
PLAYFIELD_FRAME_OVERLAY X0, ROW0, X1, ROW1, ADDR MASKS, OFFSET0, OFFSET1
```

`X` is a pixel, `ROW` the first row, and the masks start at
`MASKS + OFFSET` with one byte per row: each bit is a 4 pixel column of
the player, so `%XXXXXX..` is 24 pixels wide. Set the colors with
`P0COLOR` and `P1COLOR`. The players keep no RAM between frames.

With 8 pixel cells the TILE surface groups two playfield cells into one
8x8 tile, twenty per row, and `TILE_BEGIN 2, 1` selects it. Fewer than 24
rows are centered. Row 0 can be a four digit score: set
`A2600_INFO1 INFO_SCORE` with `A2600_INFO1_ROW 0`, then `INFO1_SET LEFT,
RIGHT`, `INFO1_ADD LEFT, RIGHT`, or `SCORE_SET VALUE`. The score is
stored in the row's own two bytes. `A2600_SCORE_LAYOUT` splits or
centers it.

`TILE_ROW_COPY SOURCE_Y, DEST_Y` copies one complete row on every
playfield surface.

## Information Rows

The 20 column kernel can replace up to two logical rows with a score or
text row:

```basic
@OPTION TARGET atari2600
@OPTION A2600_PLAYFIELD_ROWS 16
@OPTION SPRITE_COUNT 2

@OPTION A2600_INFO1 INFO_SCORE
@OPTION A2600_INFO1_ROW 0
@OPTION A2600_INFO2 STATIC_TEXT
@OPTION A2600_INFO2_ROW 10
@OPTION A2600_INFO2_TEXT "LIFE"
@OPTION A2600_SCORE_DIGITS 4
@OPTION A2600_SCORE_LAYOUT SCORE_LAYOUT_CENTERED
```

Information rows need:

- 20 columns with zero or two sprites
- single colors for players, playfield, and background, and fixed row
  heights
- `A2600_INPUT_MODE INPUT_STANDARD`
- rows of at least 8 lines, 12 for a six digit score
- with two slots, either no RAM rows or every row in RAM

Each slot accepts `INFO_NONE`, `INFO_SCORE`, `STATIC_TEXT`, or
`DYNAMIC_TEXT`. `INFO_NONE` disables the slot. `A2600_INFO1_ROW` and
`A2600_INFO2_ROW` select zero based logical rows from `0` through
`PLAYFIELD_ROWS - 1`. Two enabled slots must use different rows. An
information row replaces the playfield row at that position and keeps
the row height; the glyphs use five lines and the rest is blank.

`A2600_SCORE_DIGITS` is the total number of decimal digits across a score
row. It accepts `4` or `6` and defaults to `4`. Six means `000` on each
side. The two values are independent and keep their leading zeroes.

`A2600_SCORE_LAYOUT` controls four digit score placement. The default
`SCORE_LAYOUT_SPLIT` draws two digits on each side of the screen.
`SCORE_LAYOUT_CENTERED` draws the four digits together in the center.
For example, `INFO1_SET 12, 34` displays `1234` in the centered layout.

Score setters are available only for slots using `INFO_SCORE`:

| Digits | Slot 1 | Slot 2 |
| --- | --- | --- |
| `4` | `INFO1_SET(LEFT_VALUE AS U8, RIGHT_VALUE AS U8)` | `INFO2_SET(LEFT_VALUE AS U8, RIGHT_VALUE AS U8)` |
| `6` | `INFO1_SET(LEFT_VALUE AS U16, RIGHT_VALUE AS U16)` | `INFO2_SET(LEFT_VALUE AS U16, RIGHT_VALUE AS U16)` |

`SCORE_SET VALUE` fills slot 1 with one value. Four digit setters
display each value modulo 100. Six digit setters display each value
modulo 1000.

Every text row displays up to four characters and pads shorter text with
spaces. Text glyphs cover ASCII space through `Z`. `A2600_INFO1_TEXT` and
`A2600_INFO2_TEXT` provide quoted text for static rows. Dynamic rows use
`INFO1_TEXT_SET(TEXT_VALUE AS STRING * 4)` or
`INFO2_TEXT_SET(TEXT_VALUE AS STRING * 4)`.

Each score slot uses 4 persistent RAM bytes with four digits or 6 bytes
with six digits. Each dynamic text slot uses 4 persistent bytes. A static
text slot keeps its text in ROM and uses no dedicated persistent RAM.

Resolved constants `INFO1_TYPE`, `INFO2_TYPE`, and `SCORE_DIGITS` report
the selected slot modes and score digit count for `@IF` blocks.

## Portable TILE

TILE is a binary playfield surface on Atari 2600:

- Tile value `0` clears a cell
- Any nonzero tile value sets a cell
- Plot, clear, lines, boxes, fill, dimensions, wait, and image display
  are supported
- Tile definitions, tile blocks, and per tile colors are not supported

The related capability values are `TILE_CUSTOM_SHAPE_SUPPORTED = FALSE`,
`TILE_CUSTOM_SHAPE_DYNAMIC_SUPPORTED = FALSE`, and
`TILE_BLOCK_SUPPORTED = FALSE`. `TILE_COLOR_AVAILABLE` remains FALSE on this
tile surface.

`TILE_SUPPORTED` is true when the display has RAM rows: 20 columns with
`A2600_PLAYFIELD_RAM_ROWS` set, or 40 columns. The surface is 20 by
`PLAYFIELD_ROWS` cells on the 20 column kernel, 40 by `PLAYFIELD_ROWS`
with 4 pixel cells, and 20 by `PLAYFIELD_ROWS` with 8 pixel cells.

Every surface also provides target only `TILE_GET(X AS U8, Y AS U8) AS
U8`, which reads the displayed playfield and returns `0` or `1`, and
`TILE_ROW_COPY(SOURCE_Y AS U8, DEST_Y AS U8)`.

The CELL calls draw on the same surface. A cell value of `0` is clear
and any other value is set. `CELL_MEMMOVE` moves cells within one row.

For 8x8 logical tiles on the 20 column surface, group two playfield
cells into each TILE:

```basic
@OPTION TILE_FIXED_SPAN_W 2
@OPTION TILE_FIXED_SPAN_H 1

TILE_BEGIN 2, 1
```

This provides ten columns. `TILE_PLOT`, `TILE_GET`, and `TILE_HLINE`
then use one 8x8 tile per logical position. The 40 column kernel with 8
pixel cells uses this grouping on its own.

Use score colors to show only the left half of a repeated playfield:

```basic
@OPTION A2600_PLAYFIELD_LAYOUT PLAYFIELD_REPEAT
@OPTION A2600_PLAYFIELD_RAM_ROWS 8

P0COLOR YELLOW
P1COLOR BLACK
BGCOLOR BLACK
PLAYFIELD_CTRL PF_SCORE
```

## Portable Sprites

The portable SPRITE API supports 8x8 one-bit shapes stored as eight
bytes. The usual setup is:

```basic
@OPTION SPRITE_COUNT 2

CONST SHIP(7) AS U8 = %
	...XX...
	..XXXX..
	.XXXXXX.
	XXXXXXXX
	XXXXXXXX
	.XXXXXX.
	..XXXX..
	...XX...
END%

PROC MAIN
	SPRITES_ON
	SPRITE_DATA_8X8 0, ADDR SHIP
	SPRITE_COLOR 0, YELLOW
	SPRITE_MOVE 0, 40, 80
	SPRITE_SHOW 0
	SPRITES_FLUSH

	WHILE 1
		FRAME_WAIT
	ENDWHILE
ENDPROC
```

| Call | Purpose |
| --- | --- |
| `SPRITES_ON` | Prepare sprites; sprites start hidden |
| `SPRITE_DATA_8X8 ID, ADDR` | Set an eight-byte shape |
| `SPRITE_MOVE ID, X, Y` | Set pixel position |
| `SPRITE_COLOR ID, C` | Set sprite color |
| `SPRITE_SHOW ID`, `SPRITE_HIDE ID` | Change visibility |
| `SPRITES_FLUSH` | Sort four/six-sprite reuse state by visibility and Y; no-op with two sprites |
| `SPRITES_RESET` | Clear reuse state, or hide and reset stored two-sprite display state |
| `SPRITES_OFF` | Hide all configured sprites |

With two sprites, both hardware players draw their own shape and color.
They can overlap on a scanline. This works on the 20 column kernel with
a static playfield or with RAM rows. Sprite setters take effect immediately,
and `SPRITES_FLUSH` does nothing. `SPRITES_RESET` hides both players,
clears their stored Y, visibility, and reflection, and returns each player
to one copy. It preserves X, shape pointers, colors, and missile sizes.

Two sprite displays support horizontal flipping with `SPRITE_FLIP_X`
and horizontal doubling with `SPRITE_EXPAND`, which keeps missile sizes.
Sprite priority is shared by all sprites on every sprite display. With collisions enabled, `SPRITE_HIT` checks player to
player contact and `SPRITE_HIT_BG` checks player to playfield contact.

Portable sprite positions use X `0..149` when the full 8x8 shape must
remain on screen. Two-sprite Y positions use `0..184`; four/six-sprite
reuse uses `3..184` because its aligned setup and two positioning
scanlines precede the shape.

With four or six sprites, the target reuses the two hardware players.
Call `SPRITES_FLUSH` after changing Y or visibility. It places visible
logical sprites in Y order for the next frame. Adjacent sprites need an
11-scanline slot: eight shape scanlines, one clear/setup scanline, and
two positioning scanlines. Place requested Y positions at least 11
scanlines apart to preserve them. Closer sprites are delayed and may
clip near the bottom.
Only one logical sprite may occupy a scanline, reported by
`SPRITE_HW_PER_SCANLINE = 1`.

`SPRITE_COLOR_MODE_SHARED` uses the two TIA player colors across reused
sprites. `SPRITE_COLOR_MODE_PER_SPRITE` stores and applies a color for
each of the four or six logical sprites. The two-sprite display already
has one hardware color per sprite, so leave this option at its default.

The target specific `PLAYER0_*` and `PLAYER1_*` calls control the two
hardware players: `PLAYER0_X`, `PLAYER0_Y`, `PLAYER0_SHAPE ADDR`,
`PLAYER0_HIDE`, `PLAYER0_REFLECT FLAG`, `PLAYER0_CTRL VALUE`, and the same
for player 1. Shapes are `A2600_SPRITE_HEIGHT` bytes tall (8, 16, or 32).
They do not update four or six sprite reuse state, so use the portable
SPRITE calls for reused sprites. Missile, ball, and collision helpers are
available only when `MISSILES_ENABLED`, `BALL_ENABLED`, and
`COLLISIONS_ENABLED` resolve true.

## Images and Text

Atari 2600 images use `IMAGE_FMT_PF_ROWS`. Nonzero pixels become set
playfield bits. Source PNG files must be 8 bit indexed and at most 40x64
pixels. `--mirror` requires a width of 20 pixels or less.

```sh
tools/bin/cb-image --target atari2600 --name A2600 --mirror pf.png -o pf.cbi
```

Images draw into the RAM rows. Rows and columns beyond the configured
grid are clipped. Converted images use RLE8 when it makes them smaller
and otherwise keep the original row payload. Image data is linked into
the cartridge.

Wipes, slides, four step dither fades, and dissolves draw into the RAM
playfield rows. Dissolve block sizes of 1, 2, 4, and 8 are measured in
playfield pixels. Effects use unpacked images. Image loads and effects
draw frames whenever their work would not fit in the time between
frames, so a delay of `0` runs as fast as the display allows.

The target has no general console text output, so `PRINT` is not a
screen renderer and `TEXT_OUTPUT_SUPPORTED` remains false. It does have
two narrow target renderers:

- Playfield text draws 3x5 glyphs into the binary TILE surface
- Sprite text uses the two player objects for two 8x8 characters

| Call | Purpose |
| --- | --- |
| `PLAYFIELD_TEXT X, Y, S$` | Draw playfield text |
| `PLAYFIELD_TEXT_CLEAR X, Y, COUNT` | Clear playfield text |
| `SPRITE_TEXT X, Y, S$` | Draw two sprite characters |
| `SPRITE_TEXT_CLEAR X, Y, COUNT` | Hide both sprite characters |
| `TEXT X, Y, S$` | Use the selected target renderer |
| `TEXT_CLEAR X, Y, COUNT` | Clear using the selected target renderer |
| `PRINT_AT X, Y, S$` | Same renderer as `TEXT` |

`A2600_TEXT_RENDERER` selects the renderer for `TEXT` at compile time.
Use `A2600_FONT` and `A2600_SPRITE_FONT` only when the program needs
custom fonts. `FONT`, `PLAYFIELD_FONT`, and `SPRITE_FONT` select fonts at
runtime. Keep strings short: playfield text accepts `STRING * 5`, while
sprite text accepts `STRING * 2`. Playfield text requires RAM rows.
Sprite text requires `SPRITE_COUNT 2`. `TEXT_SUPPORTED` reports
whether the selected renderer works with the resolved display.

Information row text is separate from these renderers. Select it with
`A2600_INFO1` or `A2600_INFO2` as described in
[Information Rows](#information-rows).

## Frames and Timing

With a built in display kernel, `FRAME_WAIT` draws one frame and
increments the portable `FRAME_COUNTER`. `FRAME_WAIT COUNT` and
`FRAME_DELAY COUNT` draw frames while waiting, so the television signal
stays active.

```basic
WHILE 1
	' update game state
	FRAME_WAIT
ENDWHILE
```

`FRAME_COUNTER` is the shared 16 bit frame count and wraps every 65536
drawn frames. `TICKS` uses the same counter and wraps after 65536 frames
since `TICKS_RESET`.

`FRAME_INSTALL ADDR` runs a PROC once per frame at the start of
overscan, right after the picture, and `FRAME_UNINSTALL` removes it.
The hook runs inside the kernel's frame, so keep it short; a few
hundred cycles fit before the next frame starts.

VBLANK and overscan are timed with the RIOT timer rather than counted,
so positioning objects anywhere on the line, including the right edge,
never changes the frame length.

`REGION` selects the television standard. NTSC, the default, draws 3
VSYNC, 37 VBLANK, 192 visible, and 30 overscan scanlines (262) with
`TICKS_HZ` 60. `@OPTION REGION REGION_PAL` draws 3, 63, 192, and 54
scanlines (312) with `TICKS_HZ` 50 and PAL color values behind the same
color names. The picture, rows, RAM, and TILE contracts are identical in
both regions.

## RAM

| Feature | RAM cost |
| --- | --- |
| Every kernel | 1 byte of playfield control, 2 bytes of frame hook, joystick shadow, frame counter, random state |
| Static playfield | 3 shadow bytes |
| 20 column RAM rows | 3 bytes per row |
| 40 column RAM rows, 4 pixel cells | 6 bytes per row |
| 40 column rows, 8 pixel cells | 2.5 bytes per row |
| Two sprites | X, first and last line, and shape address per player, 5 bytes each, plus portable Y and visibility |
| Missiles or ball | X and row, 2 bytes per object, 3 with row runs or gaps |
| Four sprites | 24 byte table, 28 with per sprite color |
| Six sprites | 36 byte table, 42 with per sprite color |
| Per row colors or heights | One byte per row and table |
| Four digit score row | 4 glyph bytes |
| Six digit score row | 6 glyph bytes |
| Static information text | No persistent row state |
| Dynamic information text | 4 glyph bytes |
| Timed paddles | 2 bytes |
| Playfield players | None between frames |

The four and six sprite figures cover persistent reuse tables only.
Linked SPRITE routine locals add RAM, so the total varies with the calls
used.

Row storage is the largest share of the 128 bytes. Prefer a small RAM
row window with fill rows on the 40 column kernel, fewer rows, shared
colors, and only the hardware tables the game uses. The compiler
reports RAM overflow with the feature that caused it.

## Other Atari 2600 Options

These source options refine a supported feature combination:

| Option | Values |
| --- | --- |
| `A2600_PLAYFIELD_LAYOUT` | `PLAYFIELD_REPEAT`, `PLAYFIELD_REFLECT` (default); 20 columns only |
| `A2600_PLAYFIELD_ROWS` | Any count whose height fits 192 lines; default `8` |
| `A2600_PLAYFIELD_ROW_SCANLINES` | Lines per row; default `192 \ ROWS`, `8` with 8 pixel cells |
| `A2600_PLAYFIELD_RAM_ROWS` | Rows kept in RAM; default `0` on 20 columns, all rows on 40 |
| `A2600_PLAYFIELD_RAM_ROW_FIRST` | First RAM row on the 40 column kernel with 4 pixel cells; default `0` |
| `A2600_PLAYFIELD_FILL_ROW_ABOVE`, `A2600_PLAYFIELD_FILL_ROW_BELOW` | `TRUE` names a six byte `CONST` row for the rows outside the window |
| `A2600_PLAYFIELD_ROW_COLORS` | `TRUE` colors 40 column rows from the `CONST` table `PLAYFIELD_ROW_COLORS` |
| `A2600_PLAYFIELD_CELL_PIXELS` | `4` (default), `8`; 40 columns only |
| `A2600_PLAYFIELD_PLAYERS` | `0` (default), `1`, `2`; 40 columns only |
| `A2600_PLAYER_ROWS` | Rows a playfield player covers; default `1` |
| `A2600_SPRITE_HEIGHT` | `8` (default), `16`, `32` |
| `A2600_INFO1` | `INFO_NONE` (default), `INFO_SCORE`, `STATIC_TEXT`, `DYNAMIC_TEXT` |
| `A2600_INFO1_ROW` | `0` through `PLAYFIELD_ROWS - 1`; required when slot 1 is enabled |
| `A2600_INFO1_TEXT` | Quoted text up to four characters; required with slot 1 `STATIC_TEXT` |
| `A2600_INFO2` | `INFO_NONE` (default), `INFO_SCORE`, `STATIC_TEXT`, `DYNAMIC_TEXT` |
| `A2600_INFO2_ROW` | `0` through `PLAYFIELD_ROWS - 1`; required when slot 2 is enabled |
| `A2600_INFO2_TEXT` | Quoted text up to four characters; required with slot 2 `STATIC_TEXT` |
| `A2600_SCORE_DIGITS` | `4` (default), `6` |
| `A2600_SCORE_LAYOUT` | `SCORE_LAYOUT_SPLIT` (default), `SCORE_LAYOUT_CENTERED` |
| `A2600_MISSILES` | `TRUE` (default), `FALSE`; 20 columns |
| `A2600_BALL` | `TRUE` (default), `FALSE`; not with 8 pixel cells |
| `A2600_COLLISIONS` | `TRUE` (default), `FALSE` |
| `A2600_SPRITE_COLORS` | `SPRITE_COLOR_MODE_SHARED`, `SPRITE_COLOR_MODE_PER_SPRITE`; four or six sprites |
| `A2600_PLAYER_COLORS` | `PLAYER_COLORS_SINGLE`, `PLAYER_COLORS_PLAYER1_PER_ROW`, `PLAYER_COLORS_PER_ROW` |
| `A2600_PLAYFIELD_COLORS` | `PLAYFIELD_COLORS_SINGLE`, `PLAYFIELD_COLORS_PER_ROW` |
| `A2600_BACKGROUND_COLORS` | `BACKGROUND_COLORS_SINGLE`, `BACKGROUND_COLORS_PER_ROW` |
| `A2600_PLAYFIELD_HEIGHTS` | `PLAYFIELD_HEIGHTS_FIXED`, `PLAYFIELD_HEIGHTS_PER_ROW` |
| `A2600_INPUT_MODE` | `INPUT_STANDARD`, `INPUT_TIMED_PADDLE` |
| `A2600_OBJECT_ROWS` | `OBJECT_ROWS_SINGLE` (default), `OBJECT_ROWS_RANGE`, `OBJECT_ROWS_GAP`; 20 columns |
| `A2600_TEXT_RENDERER` | `TEXT_PLAYFIELD`, `TEXT_SPRITE` |
| `A2600_FONT` | Playfield font path |
| `A2600_SPRITE_FONT` | Sprite font path |

The per row tables, information rows, missiles, and timed paddles belong
to the 20 column kernel. Fill rows, `PLAYFIELD_ROW_COLORS`, cell width,
and playfield players belong to the 40 column kernel. Four and six
sprite displays draw a static playfield only, without missiles, the
ball, collisions, information rows, or per row tables.

Resolved constants include `PLAYFIELD_LAYOUT`, `PLAYFIELD_COLUMNS`,
`PLAYFIELD_ROWS`, `PLAYFIELD_ROW_SCANLINES`, `PLAYFIELD_RAM_ROWS`,
`PLAYFIELD_CELL_PIXELS`, `MISSILE0_SUPPORTED`, `MISSILE1_SUPPORTED`,
`MISSILES_ENABLED`, `BALL_SUPPORTED`, `BALL_ENABLED`,
`COLLISIONS_ENABLED`, and the portable capability constants. Use those
values in `@IF` blocks when a source needs to adapt to the chosen
features. The `A2600_MISSILES`, `A2600_BALL`, and `A2600_COLLISIONS`
options request features; the resolved constants say whether the chosen
display can provide them.

## Playfield and Hardware Objects

| Call | Purpose |
| --- | --- |
| `BGCOLOR C` | Set background color |
| `PFCOLOR C` | Set playfield and ball color |
| `PLAYFIELD PF0, PF1, PF2` | Set the static playfield bits for rows outside the RAM window |
| `PLAYFIELD_CTRL FLAGS` | Set score, priority, and ball size bits; the kernel stores them once per frame |
| `MISSILE0_ENABLE FLAG`, `MISSILE1_ENABLE FLAG` | Show or hide missiles when available |
| `MISSILE0_X X`, `MISSILE1_X X` | Position missiles |
| `MISSILE0_ROW ROW`, `MISSILE0_Y Y`, and the same for missile 1 | Pick the row a missile appears on |
| `MISSILE0_ROWS FIRST, COUNT`, `MISSILE0_GAP FIRST, COUNT`, `MISSILE0_HIDE` | Row runs and gaps with `A2600_OBJECT_ROWS` |
| `BALL_ENABLE FLAG`, `BALL_SIZE SIZE` | Show the ball and set its size |
| `BALL_X X`, `BALL_ROW ROW`, `BALL_Y Y`, `BALL_ROWS`, `BALL_GAP`, `BALL_HIDE` | Position the ball |
| `COLLISION_CLEAR` | Clear TIA collision latches |

Playfield flags are `PF_REFLECT`, `PF_SCORE`, and `PF_PRIORITY`; the
kernels set reflection from `A2600_PLAYFIELD_LAYOUT`. Ball sizes are
`BALL_SIZE_1`, `BALL_SIZE_2`, `BALL_SIZE_4`, and `BALL_SIZE_8`. Extra
color names are `BRIGHT_ORANGE`, `DARK_BLUE`, `DARK_ORANGE`, `DARK_RED`,
`DARK_YELLOW`, and `LIGHT_ORANGE`.

Read collision helpers after a frame, then call `COLLISION_CLEAR` before
the next frame. The `HIT_*` helpers read individual collision latches.
`ASM` blocks can use the TIA and RIOT register names directly.

## Custom Kernels

`A2600_CUSTOM_KERNEL TRUE` with `SPRITE_COUNT 0` links no display
kernel. The program owns the display: it writes its own PROC that draws
a frame and calls it instead of `FRAME_WAIT`. The shared runtime still
expects `FRAME_COUNTER` counting frames and the background color in
`_vcs_color`. `FRAME_WAIT` does nothing, and
TILE, images, text, sprites, and the frame hook are unavailable.

## Input

| Capability | Value |
| --- | --- |
| Keyboard | No |
| Joystick ports | 2 |
| Buttons | 1 per port |
| Paddle axes | 2 with `INPUT_TIMED_PADDLE`, otherwise 0 |
| Keypad ports | 0 |

`JOY(0)` and `JOY(1)` return direction bits. `JOY_BUTTON(PORT, 0)` reads
fire. `PAD1` and `PAD2` combine directions and fire in one target mask.
`CONSOLE_SWITCHES`, `CONSOLE_RESET`, `CONSOLE_SELECT`, `CONSOLE_BW`,
`CONSOLE_P0_DIFFICULTY`, and `CONSOLE_P1_DIFFICULTY` expose the console
switches. The difficulty helpers return `CONSOLE_DIFFICULTY_A` (`1`) or
`CONSOLE_DIFFICULTY_B` (`0`).

Paddles are timing devices. `PADDLE` requires `A2600_INPUT_MODE
INPUT_TIMED_PADDLE`: the 20 column kernel times paddles 0 and 1 while it
draws, and `PADDLE(0)` and `PADDLE(1)` return positions. `PADDLE_MAX` is
the position both paddles reach at full turn for the chosen row layout;
one paddle can count a little past it. Timed paddles are not available
with information rows. Paddle positions update while frames are drawn.

`PTRIG(0)` through `PTRIG(3)` read the paddle buttons and return `0`
while a button is pressed. Joystick directions, fire buttons, and paddle
buttons are read when called.

## Sound

`SOUND_SUPPORTED` is `TRUE` and `SOUND_VOICES` is `2`.
`SOUND_SHAPE_SUPPORTED` and `SOUND_NOISE_SUPPORTED` are `TRUE`.
`SOUND_VOLUME_MODEL` is `SOUND_VOLUME_MODEL_VOICE`, and
`SOUND_VOLUME_MAX` is `15`.

Exact TIA sound control uses the chip interface:

```basic
TIA.SOUND CHAN, AUDF, AUDC, VOLUME
```

| Argument | Atari 2600 meaning |
| --- | --- |
| `CHAN` | TIA voice `0` or `1` |
| `AUDF` | Native divider `0..31` |
| `AUDC` | Native control `0..15` |
| `VOLUME` | Native volume `0..15` |

For example, this starts a tone on voice 0:

```basic
TIA.SOUND 0, 12, 4, 8
```

The call writes `AUDC`, `AUDF`, and `AUDV` directly and returns. The
voice keeps playing until another `TIA.SOUND` call changes it,
`SOUND_OFF CHAN` stops it, or `SOUND_ALL_OFF` or `SILENCE` stops both
voices.

The portable shape constants map to TIA control values:
`SOUND_SHAPE_DEFAULT`, `SOUND_SHAPE_TONE`, `SOUND_SHAPE_TRIANGLE`, and
`SOUND_SHAPE_SAW` are `4`; `SOUND_SHAPE_PULSE` is `12`; and
`SOUND_SHAPE_NOISE` is `8`.

Portable note playback covers `NOTE_C4` through `NOTE_B6`. `PLAY_NOTE`
uses the paired TIA pitch and control table. `PLAY_NOTE_SHAPE` uses the
requested control with the closest pitch that control reaches; controls
`4` and `5` go no lower than `NOTE_B4`, so lower notes play at that
pitch. Note pitches follow `REGION`. Exact effects use `TIA.SOUND`.

`PLAY_NOTE_FOR` and `PLAY_NOTE_SHAPE_FOR` block for their duration in
ticks, one per drawn frame, and then stop the voice.

`BEEP DURATION` uses voice `0`, divider `12`, control `4`, and volume
`8`. It calls `FRAME_WAIT DURATION`, keeping a built in display active,
and then stops voice `0`. Its duration is also measured in drawn frames.
`BEEP PERIOD, DURATION` works the same way with `PERIOD` as the divider.

Timed sound scheduling is unavailable: `SOUND_TIMED_SUPPORTED` and
`SOUND_TIMED_FREE_RUNNING` are `FALSE`, and `SOUND_TIMED_VOICES` is `0`. Timed
sound calls do nothing and their status queries return `0`.

## Cartridge Sizes

Default builds try 2K, then 4K. Larger banked cartridges use
`@OPTION MAPPER`:

| Mapper | Size | Banks |
| --- | --- | --- |
| `f8` | 8K | 2 |
| `f6` | 16K | 4 |
| `f4` | 32K | 8 |

Use `@BANK N` for code in a switched bank. See
[`../USAGE.md#banked-builds`](../USAGE.md#banked-builds).
