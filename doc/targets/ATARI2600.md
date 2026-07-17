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
| User data | `$B4-$F3`, 64 bytes before feature storage |
| Built-in timing | NTSC, 262 scanlines |
| `REAL` | Not supported |
| String encoding | ASCII |

The CPU stack shares the same 128 bytes of RAM. Keep globals, arrays,
and strings tiny. A plain `STRING` uses 8 bytes and holds seven
characters.

The byte `LFSR` generator shares its state with `RAND` to save RAM. Seed it
after any `RAND` calls when a repeatable sequence matters.

## Choose Display Features

Select the features the program needs. The target chooses the display
kernel from them at compile time.

```basic
@OPTION TARGET atari2600
@OPTION A2600_PLAYFIELD_MODE PLAYFIELD_MODE_ROWS
@OPTION A2600_PLAYFIELD_COLUMNS 20
@OPTION A2600_PLAYFIELD_ROWS 16
@OPTION A2600_SPRITE_COUNT 2
```

The main options are:

| Option | Values | Default |
| --- | --- | --- |
| `A2600_PLAYFIELD_MODE` | `PLAYFIELD_MODE_NONE`, `PLAYFIELD_MODE_STATIC`, `PLAYFIELD_MODE_ROWS` | `PLAYFIELD_MODE_STATIC` |
| `A2600_SPRITE_COUNT` | `0`, `2`, `4`, `6` | `0` |
| `A2600_SPRITE_COLORS` | `SPRITE_COLOR_MODE_SHARED`, `SPRITE_COLOR_MODE_PER_SPRITE` | `SPRITE_COLOR_MODE_SHARED` |

With no feature options, the target provides a static playfield without
player sprites. The static display still makes both missiles, the ball,
and collision reads available. The default 20x8 TILE display remains
available; `TILE_BEGIN`, playfield text, or image display switches to it
at runtime. Missiles and the ball are not drawn in this TILE view. An
explicit playfield mode trims that fallback. Select
`PLAYFIELD_MODE_ROWS` for a row playfield, or set
`@OPTION TILE_BACKEND TILE_KERNEL` and let the target select row mode
when no playfield mode was given.

Supported combinations:

| Playfield | Sprites | Result |
| --- | --- | --- |
| None | 0 | No built-in display kernel, for a custom kernel |
| None | 2 | Two hardware sprites without a requested playfield |
| Static | 0 or 2 | Static playfield with up to two hardware sprites |
| None or static | 4 or 6 | Sprite reuse display with one logical sprite per scanline |
| 20-column rows | 0 or 2 | Binary TILE playfield with up to two hardware sprites |
| 40-column rows | 0 | Unique full-width binary TILE playfield with optional ball |

Unsupported combinations fail during preprocessing with the conflicting
options in the message.

Information rows require a static playfield with zero or two sprites. See
[Information Rows](#information-rows) for their source options.

Do not select a kernel name. `KERNEL_PROFILE` is read-only metadata that
reports the private profile chosen by the target. It is useful for target
conditionals and diagnostics, not configuration.

## Portable Sprites

The portable SPRITE API supports 8x8 one-bit shapes stored as eight
bytes. The usual setup is:

```basic
@OPTION A2600_SPRITE_COUNT 2

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
	SPRITES_BEGIN
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
| `SPRITES_BEGIN` | Call `SPRITES_RESET` and begin with hidden sprites |
| `SPRITE_DATA_8X8 ID, ADDR` | Set an eight-byte shape |
| `SPRITE_MOVE ID, X, Y` | Set pixel position |
| `SPRITE_COLOR ID, C` | Set sprite color |
| `SPRITE_SHOW ID`, `SPRITE_HIDE ID` | Change visibility |
| `SPRITES_FLUSH` | Sort four/six-sprite reuse state by visibility and Y; no-op with two sprites |
| `SPRITES_RESET` | Clear reuse state, or hide and reset stored two-sprite display state |
| `SPRITES_OFF` | Hide all configured sprites |

With two sprites, both hardware players draw their own shape and color.
They can overlap on a scanline. This works with a static playfield and
with 20-column row playfields. Sprite setters take effect immediately,
and `SPRITES_FLUSH` does nothing. `SPRITES_RESET` hides both players,
clears their stored Y and visibility, and returns each player to one
copy. It preserves X, shape pointers, and colors.

Two sprite displays support horizontal flipping with `SPRITE_FLIP_X`
and horizontal doubling with `SPRITE_EXPAND`. Sprite priority is shared
by both players. With collisions enabled, `SPRITE_HIT` checks player to
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

The target-specific `PLAYER0_*` and `PLAYER1_*` calls control the two
hardware players. They do not update four/six-sprite logical reuse
state, so use the portable SPRITE calls for reused sprites. Missile,
ball, and collision helpers are available only when `MISSILES_ENABLED`,
`BALL_ENABLED`, and `COLLISIONS_ENABLED` resolve true.

## Portable TILE

TILE is a binary playfield surface on Atari 2600:

- Tile value `0` clears a cell
- Any nonzero tile value sets a cell
- Plot, clear, lines, boxes, fill, dimensions, wait, and image display
  are supported
- Tile definitions, tile blocks, and per-tile colors are not supported

The related capability values are `TILE_DEFINE_AVAILABLE = 0` and
`TILE_BLOCK_AVAILABLE = 0`. `TILE_COLOR_AVAILABLE` remains 0 on this
tile surface.

`A2600_PLAYFIELD_LAYOUT` controls the right half:

| Layout | Surface |
| --- | --- |
| `PLAYFIELD_REPEAT` | 20 cells repeated on the right |
| `PLAYFIELD_REFLECT` | 20 cells mirrored on the right |
| `PLAYFIELD_ASYMMETRIC` | 40 unique cells across the display |

The 20 column surface and the fixed 2x1 asymmetric surface also provide
target only `TILE_GET(X AS U8, Y AS U8) AS U8`. It reads the displayed
playfield and returns `0` or `1`.

`TILE_ROW_COPY(SOURCE_Y AS U8, DEST_Y AS U8)` copies one complete displayed
row. Use it when moving a row between frames.

As an example, for 8x8 logical tiles, group two playfield cells into each TILE:

```basic
@OPTION TILE_FIXED_CELL_W 2
@OPTION TILE_FIXED_CELL_H 1

TILE_BEGIN 2, 1
```

On the 20 column surface this provides ten columns. `TILE_PLOT`,
`TILE_GET`, and `TILE_HLINE` then use one 8x8 tile per logical position.

For a full width 20x24 logical TILE surface, use the standard asymmetric
playfield options:

```basic
@OPTION A2600_PLAYFIELD_MODE PLAYFIELD_MODE_ROWS
@OPTION A2600_PLAYFIELD_LAYOUT PLAYFIELD_ASYMMETRIC
@OPTION A2600_PLAYFIELD_COLUMNS 40
@OPTION A2600_PLAYFIELD_ROWS 24
@OPTION A2600_SPRITE_COUNT 0
@OPTION TILE_FIXED_CELL_W 2
@OPTION TILE_FIXED_CELL_H 1

TILE_BEGIN 2, 1
```

This groups the 40 playfield cells into 20 logical 8x8 tiles and uses 60
bytes for all 24 rows. This surface also provides target only:

`TILE_FRAME_P0_4X4(X AS U8, Y AS U8, SRC_ADDR AS U8 ADDR, SRC_OFFSET AS U8)`
draws one 4x4 tile mask from `SRC_ADDR + SRC_OFFSET` over the TILE surface
for one frame. `X` and `Y` use logical TILE positions, and `P0COLOR` sets
the mask color. The source is four bytes, one byte per tile row. Each pair
of bits is one 8x8 tile, so `..XXXX..` draws the middle two tiles. The
source can be one mask or a table of masks. Use X `0..16` and Y `0..20`
when the full mask must remain visible.

Use score colors to show only the left half of a repeated playfield:

```basic
@OPTION A2600_PLAYFIELD_MODE PLAYFIELD_MODE_ROWS
@OPTION A2600_PLAYFIELD_LAYOUT PLAYFIELD_REPEAT
@OPTION A2600_PLAYFIELD_COLUMNS 20
@OPTION A2600_SPRITE_COUNT 0

P0COLOR YELLOW
P1COLOR BLACK
BGCOLOR BLACK
PLAYFIELD_CTRL PF_SCORE
```

The default is `PLAYFIELD_REFLECT`. `PLAYFIELD_MODE_ROWS` defaults to 20
columns and 24 rows. The 20-column surface supports 8, 12, 16, or 24
rows and up to two sprites. The 40-column surface normally uses 8 rows and
no sprites. The fixed 2x1 asymmetric combination above supports 24 rows.
By default, every supported row count fills the 192 scanline display. Rows
with sprites require
`A2600_PLAYFIELD_ROW_SCANLINES` of at least 6. Sprite-free rows may use
lower valid nonzero heights. In all cases, rows multiplied by row height
must not exceed 192 scanlines.

The 40-column 8-row surface can also draw the TIA ball. Use `BALL_X`,
`BALL_Y`, `BALL_SIZE`, and `BALL_ENABLE`; `BALL_Y` uses pixel Y to choose
the row.

```basic
@OPTION TARGET atari2600
@OPTION A2600_PLAYFIELD_MODE PLAYFIELD_MODE_ROWS
@OPTION A2600_PLAYFIELD_COLUMNS 20
@OPTION A2600_PLAYFIELD_ROWS 16
@OPTION A2600_SPRITE_COUNT 2

PROC MAIN
	TILE_BEGIN 1, 1
	TILE_CLS
	TILE_BOX 0, 0, TILE_COLUMNS - 1, TILE_ROWS - 1, 1

	WHILE 1
		TILE_WAIT 1
	ENDWHILE
ENDPROC
```

## Information Rows

The standard display can replace up to two logical playfield rows with
score or text:

```basic
@OPTION TARGET atari2600
@OPTION A2600_PLAYFIELD_MODE PLAYFIELD_MODE_STATIC
@OPTION A2600_PLAYFIELD_ROWS 16
@OPTION A2600_SPRITE_COUNT 2

@OPTION A2600_INFO1 INFO_SCORE
@OPTION A2600_INFO1_ROW 0
@OPTION A2600_INFO2 STATIC_TEXT
@OPTION A2600_INFO2_ROW 10
@OPTION A2600_INFO2_TEXT "LIFE"
@OPTION A2600_SCORE_DIGITS 4
@OPTION A2600_SCORE_LAYOUT SCORE_LAYOUT_CENTERED
```

Information rows require:

- `A2600_PLAYFIELD_MODE PLAYFIELD_MODE_STATIC`
- `A2600_SPRITE_COUNT 0` or `A2600_SPRITE_COUNT 2`
- `A2600_PLAYER_COLORS PLAYER_COLORS_SINGLE`,
  `A2600_PLAYFIELD_COLORS PLAYFIELD_COLORS_SINGLE`, and
  `A2600_BACKGROUND_COLORS BACKGROUND_COLORS_SINGLE`
- `A2600_PLAYFIELD_HEIGHTS PLAYFIELD_HEIGHTS_FIXED`
- `A2600_INPUT_MODE INPUT_STANDARD`

Each slot accepts `INFO_NONE`, `INFO_SCORE`, `STATIC_TEXT`, or
`DYNAMIC_TEXT`. `INFO_NONE` disables the slot. `A2600_INFO1_ROW` and
`A2600_INFO2_ROW` select zero based logical rows from `0` through
`PLAYFIELD_ROWS - 1`. Two enabled slots must use different rows. An
information row replaces the normal playfield row at that position and
keeps the fixed `PLAYFIELD_ROW_SCANLINES` height.

The row count multiplied by the fixed row height must equal 192. When
setting `A2600_PLAYFIELD_ROW_SCANLINES`, use `8 x 24`, `12 x 16`,
`16 x 12`, or `24 x 8`, with the row count first. Information rows require
at least 8 scanlines. A six digit score requires at least 12. The 24 row
layout uses 8 scanlines per row and cannot use six digit scores. Select 8,
12, or 16 rows instead.

`A2600_SCORE_DIGITS` is the total number of decimal digits across a score
row. It accepts `4` or `6` and defaults to `4`. Six means `000` on each
side. The two values are independent and keep their leading zeroes.

`A2600_SCORE_LAYOUT` controls four-digit score placement. The default
`SCORE_LAYOUT_SPLIT` draws two digits on each side of the screen.
`SCORE_LAYOUT_CENTERED` draws the four digits together in the center.
For example, `INFO1_SET 12, 34` displays `1234` in the centered layout.

Score setters are available only for slots using `INFO_SCORE`:

| Digits | Slot 1 | Slot 2 |
| --- | --- | --- |
| `4` | `INFO1_SET(LEFT_VALUE AS U8, RIGHT_VALUE AS U8)` | `INFO2_SET(LEFT_VALUE AS U8, RIGHT_VALUE AS U8)` |
| `6` | `INFO1_SET(LEFT_VALUE AS U16, RIGHT_VALUE AS U16)` | `INFO2_SET(LEFT_VALUE AS U16, RIGHT_VALUE AS U16)` |

Four digit setters display each value modulo 100. Six digit setters
display each value modulo 1000.

Every text row displays up to four characters and pads shorter text with
spaces. Text glyphs cover ASCII space through `Z`. `A2600_INFO1_TEXT` and
`A2600_INFO2_TEXT` provide quoted text for static rows. Dynamic rows use
`INFO1_TEXT_SET(TEXT_VALUE AS STRING * 4)` or
`INFO2_TEXT_SET(TEXT_VALUE AS STRING * 4)`.

Each score slot uses exactly 4 persistent RAM bytes with four digits or 6
bytes with six digits. Each dynamic text slot uses 4 persistent bytes. A
static text slot keeps its text in ROM and uses no dedicated persistent
RAM.

Resolved constants `INFO1_TYPE`, `INFO2_TYPE`, and `SCORE_DIGITS` report
the selected slot modes and score digit count for `@IF` blocks.

The selected row types insert small assembly snippets into the normal
display row code. They do not use separate display kernels. Only the
selected row code and required font data are linked into the cartridge.

## Images and Text

Atari 2600 images use `IMAGE_FMT_PF_ROWS`. Nonzero pixels become set
playfield bits. Source PNG files must be 8-bit indexed and at most 40x64
pixels. `--mirror` requires a width of 20 pixels or less.

```sh
tools/bin/cb-image --target atari2600 --name A2600 --mirror pf.png -o pf.cbi
```

Images use row playfield mode. Rows and columns beyond the configured
grid are clipped. Image data is linked into the cartridge.

The target has no general console text output, so `PRINT` is not a
screen renderer and `TEXT_OUTPUT_AVAILABLE` remains false. It does have
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
runtime. Keep strings short: playfield text accepts
`STRING * 5`, while sprite text accepts `STRING * 2`. Playfield text
requires `TILE_AVAILABLE`. Sprite text requires `A2600_SPRITE_COUNT 2`.
`TEXT_AVAILABLE` reports whether the selected renderer works with the
resolved display.

Information row text is separate from these renderers. Select it with
`A2600_INFO1` or `A2600_INFO2` as described in
[Information Rows](#information-rows).

## Frames and Timing

With a built-in display kernel, `FRAME_WAIT` draws one frame and
increments the portable `FRAME_COUNTER`. `FRAME_WAIT COUNT`,
`FRAME_DELAY COUNT`, and `TILE_WAIT COUNT` draw frames while waiting, so
the television signal stays active.

```basic
WHILE 1
	' update game state
	FRAME_WAIT
ENDWHILE
```

`DRAWSCREEN` remains the direct target call. `FRAME_COUNT` returns the
target's 8-bit frame count and wraps every 256 drawn frames.
`TICKS_HZ` is 60; `TICKS` advances only when frames are drawn and also
wraps every 256 frames.

There is no frame callback or VBI hook. `FRAME_INSTALL_AVAILABLE` is
false. The built-in display kernels currently use NTSC timing: 3 VSYNC,
37 VBLANK, 192 visible, and 30 overscan scanlines. Each scanline has 76
CPU cycles.

## RAM and Scanline Tradeoffs

| Feature | RAM cost | Display limit |
| --- | --- | --- |
| Static playfield | Four shadow bytes | Same playfield bits for all visible lines |
| Two sprites | Base X, Y, and raw shape pointers, plus portable Y and visibility state | Two complete 8x8 shapes may share a scanline |
| 20-column rows | 20, 30, 40, or 60 row bytes; sprite overlay adds persistent top, end, X, and saved shape pointers | Row writes consume visible scanline time |
| 40-column, 8 rows | 40 bytes; ball adds 3 bytes | Mid-scanline playfield rewrite leaves no sprite time |
| 40-column, fixed 2x1, 24 rows | 60 row bytes | 20x24 logical 8x8 TILE surface with one 4x4 tile frame overlay |
| Four sprites | 24-byte persistent table, 28 bytes with per-sprite color | One logical sprite per scanline; 11 scanlines per sprite |
| Six sprites | 36-byte persistent table, 42 bytes with per-sprite color | One logical sprite per scanline; 11 scanlines per sprite |
| Per-row colors or heights | One byte per enabled row and table | Less visible-loop time for other hardware work |
| Four digit score row | Four glyph bytes | Two decimal digits on each side |
| Six digit score row | Six glyph bytes | Three decimal digits on each side |
| Static information text | No persistent row state | Four characters stored in ROM |
| Dynamic information text | Four glyph bytes | Four runtime changeable characters |

The four/six-sprite figures cover persistent reuse tables only. Linked
SPRITE routine locals add RAM, so the total varies with the calls used.
For 20-column rows with two sprites, the overlay keeps 10 persistent
bytes for top, end, X, and saved shape pointers. Portable Y and
visibility state and linked routine locals are additional costs.

The playfield row storage is a large share of the 128-byte RAM. Prefer
fewer rows, shared colors, and only the hardware tables the game uses.
The compiler rejects conflicting options or a row height that makes the
display taller than 192 scanlines.

## Other Atari 2600 Options

These source options refine a supported feature combination:

| Option | Values |
| --- | --- |
| `A2600_PLAYFIELD_LAYOUT` | `PLAYFIELD_REPEAT`, `PLAYFIELD_REFLECT`, `PLAYFIELD_ASYMMETRIC` |
| `A2600_PLAYFIELD_COLUMNS` | `20`, `40` |
| `A2600_PLAYFIELD_ROWS` | `8`, `12`, `16`, `24` |
| `A2600_PLAYFIELD_ROW_SCANLINES` | Nonzero; at least `8` for information rows and `12` for six digit scores |
| `A2600_INFO1` | `INFO_NONE` (default), `INFO_SCORE`, `STATIC_TEXT`, `DYNAMIC_TEXT` |
| `A2600_INFO1_ROW` | `0` through `PLAYFIELD_ROWS - 1`; required when slot 1 is enabled |
| `A2600_INFO1_TEXT` | Quoted text up to four characters; required with slot 1 `STATIC_TEXT` |
| `A2600_INFO2` | `INFO_NONE` (default), `INFO_SCORE`, `STATIC_TEXT`, `DYNAMIC_TEXT` |
| `A2600_INFO2_ROW` | `0` through `PLAYFIELD_ROWS - 1`; required when slot 2 is enabled |
| `A2600_INFO2_TEXT` | Quoted text up to four characters; required with slot 2 `STATIC_TEXT` |
| `A2600_SCORE_DIGITS` | `4` (default), `6` |
| `A2600_SCORE_LAYOUT` | `SCORE_LAYOUT_SPLIT` (default), `SCORE_LAYOUT_CENTERED` |
| `A2600_MISSILES` | `TRUE`, `FALSE` |
| `A2600_BALL` | `TRUE`, `FALSE` |
| `A2600_COLLISIONS` | `TRUE`, `FALSE` |
| `A2600_PLAYER_COLORS` | `PLAYER_COLORS_SINGLE`, `PLAYER_COLORS_PLAYER1_PER_ROW`, `PLAYER_COLORS_PER_ROW` |
| `A2600_PLAYFIELD_COLORS` | `PLAYFIELD_COLORS_SINGLE`, `PLAYFIELD_COLORS_PER_ROW` |
| `A2600_BACKGROUND_COLORS` | `BACKGROUND_COLORS_SINGLE`, `BACKGROUND_COLORS_PER_ROW` |
| `A2600_PLAYFIELD_HEIGHTS` | `PLAYFIELD_HEIGHTS_FIXED`, `PLAYFIELD_HEIGHTS_PER_ROW` |
| `A2600_INPUT_MODE` | `INPUT_STANDARD`, `INPUT_TIMED_PADDLE` |
| `A2600_TEXT_RENDERER` | `TEXT_PLAYFIELD`, `TEXT_SPRITE` |
| `A2600_FONT` | Playfield font path |
| `A2600_SPRITE_FONT` | Sprite font path |

Important tradeoffs:

- Information rows require the standard static display with zero or two
  sprites, single color settings for players, playfield, and background,
  fixed row heights, `INPUT_STANDARD`, and exactly 192 visible scanlines
- Per-row player colors reuse RAM otherwise available to missile state
- Timed paddles reuse missile 0 state; use static with zero/two sprites or
  none with two sprites
- Per-row background color requires per-row playfield color or height
- Four and six sprite displays do not support information rows,
  per-row tables, timed paddles, missiles, the ball, or collisions

Calls for the optional row display tables are:

| Call | Purpose |
| --- | --- |
| `PLAYER0_ROW_COLOR ROW, C`, `PLAYER1_ROW_COLOR ROW, C` | Set player colors by row |
| `PLAYFIELD_ROW_COLOR ROW, C`, `BACKGROUND_ROW_COLOR ROW, C` | Set display colors by row |
| `PLAYFIELD_ROW_HEIGHT ROW, H` | Set a row height in scanlines |

These calls take effect only when their matching option is selected.
With per-row heights, make the displayed row heights total 192 scanlines.
Information rows keep the fixed
`PLAYFIELD_ROW_SCANLINES` height, so include them in that total.

Resolved constants include `PLAYFIELD_LAYOUT`, `PLAYFIELD_COLUMNS`,
`PLAYFIELD_ROWS`, `PLAYFIELD_ROW_SCANLINES`,
`MISSILE0_AVAILABLE`, `MISSILE1_AVAILABLE`, `MISSILES_ENABLED`,
`BALL_AVAILABLE`, `BALL_ENABLED`, `COLLISIONS_ENABLED`, and the portable
capability constants. Use those values in `@IF` blocks when a source
needs to adapt to the chosen features. The `A2600_MISSILES`,
`A2600_BALL`, and `A2600_COLLISIONS` options request features; the
resolved constants say whether the chosen display can provide them.

## Static Playfield and Hardware Objects

The static playfield and direct TIA calls remain useful for small games:

| Call | Purpose |
| --- | --- |
| `BGCOLOR C` | Set background color |
| `PFCOLOR C` | Set playfield and ball color |
| `PLAYFIELD PF0, PF1, PF2` | Set static playfield bits |
| `PLAYFIELD_CTRL FLAGS` | Set reflect, score, priority, and ball size bits |
| `MISSILE0_ENABLE FLAG`, `MISSILE1_ENABLE FLAG` | Show or hide missiles when available |
| `MISSILE0_X X`, `MISSILE1_X X` | Position missiles when available |
| `BALL_ENABLE FLAG`, `BALL_SIZE SIZE` | Show the ball and set its size when available |
| `BALL_X X` | Position the ball when available |
| `COLLISION_CLEAR` | Clear TIA collision latches |

Playfield flags are `PF_REFLECT`, `PF_SCORE`, and `PF_PRIORITY`.
Extra color names are `BRIGHT_ORANGE`, `DARK_BLUE`, `DARK_ORANGE`,
`DARK_RED`, `DARK_YELLOW`, and `LIGHT_ORANGE`.
Missiles, the ball, row colors, row heights, information rows, timed
paddles, and collisions are target specific features. Use
`MISSILES_ENABLED`, `BALL_ENABLED`, and `COLLISIONS_ENABLED` for resolved
overall availability. `MISSILE0_AVAILABLE` and `MISSILE1_AVAILABLE`
describe the individual missile objects.

Read collision helpers after a frame, then call `COLLISION_CLEAR` before
the next frame. The `HIT_*` helpers read individual collision latches.
Direct TIA and RIOT register names are also available for custom hardware
code.

## Input

| Capability | Value |
| --- | --- |
| Keyboard | No |
| Joystick ports | 2 |
| Buttons | 1 per port |
| Paddle axes | 4 |
| Keypad ports | 0 |

`JOY(0)` and `JOY(1)` return direction bits. `JOY_BUTTON(PORT, 0)` reads
fire. `PAD1` and `PAD2` combine directions and fire in one target mask.
`CONSOLE`, `CONSOLE_RESET`, `CONSOLE_SELECT`, `CONSOLE_BW`,
`CONSOLE_P0_DIFFICULTY`, and `CONSOLE_P1_DIFFICULTY` expose the console
switches. The difficulty helpers return `CONSOLE_DIFFICULTY_A` (`1`) or
`CONSOLE_DIFFICULTY_B` (`0`).

Paddles are timing devices. In `INPUT_STANDARD` mode, `PADDLE(0)` through
`PADDLE(3)` return the raw `INPT0` through `INPT3` state. Use
`A2600_INPUT_MODE INPUT_TIMED_PADDLE` for position values. Timed paddles
support a static playfield with zero or two sprites, or no playfield with
two sprites. Joystick directions and timed paddles update while frames
are drawn.

## Sound

`SOUND_AVAILABLE` is `1` and `SOUND_VOICES` is `2`.
`SOUND_HAS_SHAPE`, `SOUND_HAS_NOISE`, and `SOUND_HAS_NATIVE` are `1`;
`SOUND_HAS_ENVELOPE`, `SOUND_HAS_FILTER`, and `SOUND_HAS_PULSEWIDTH`
are `0`.
`SOUND_VOLUME_MODEL` is `SOUND_VOLUME_MODEL_VOICE`, and
`SOUND_VOLUME_MAX` is `15`.

`SOUND` is the immediate, target specific interface:

```basic
SOUND CHAN, FREQ, CTRL, VOL
```

| Argument | Atari 2600 meaning |
| --- | --- |
| `CHAN` | TIA voice `0` or `1` |
| `FREQ` | Native `AUDF` divider `0..31` |
| `CTRL` | Native `AUDC` control `0..15` |
| `VOL` | Voice volume `0..15` |

The call returns immediately and the voice keeps playing until another
`SOUND` call changes it, `SOUND_OFF CHAN` stops it, or
`SOUND_ALL_OFF`/`SILENCE` stops both voices. `SOUND_RAW` performs the
same operation as `SOUND`.

The portable shape constants map to TIA control values:
`SOUND_SHAPE_DEFAULT`, `SOUND_SHAPE_TONE`, `SOUND_SHAPE_TRIANGLE`, and
`SOUND_SHAPE_SAW` are `4`; `SOUND_SHAPE_PULSE` is `12`; and
`SOUND_SHAPE_NOISE` is `8`.

`SOUND_FREQ(NOTE)` returns a packed Atari value with `AUDC` in the high
byte and `AUDF` in the low byte. When this packed value is passed as
`FREQ`, its high byte overrides the `CTRL` argument. Portable note
playback covers `NOTE_C4` through `NOTE_B6`; unsupported notes return
zero and the portable note calls silence the selected voice.

`PLAY_NOTE` starts a note and leaves it playing. `PLAY_NOTE_FOR` blocks,
converts its duration to drawn frames using `DURATION \ 100`, and then
stops the voice. Because the packed note value supplies `AUDC`,
`PLAY_NOTE_SHAPE` currently uses the note table's control rather than
the requested shape. Use `SOUND` when an exact TIA control is required.

`BEEP DURATION` uses voice `0`, divider `12`, control `4`, and volume
`8`. It calls `FRAME_DELAY DURATION`, keeping a built in display active,
and then stops voice `0`. Its duration is therefore measured directly
in drawn frames, unlike `PLAY_NOTE_FOR`.

Timed sound scheduling is unavailable: `SOUND_TIMED_AVAILABLE`,
`SOUND_TIMED_FREE_RUNNING`, and `SOUND_TIMED_VOICES` are `0`. Timed
sound calls do nothing and their status queries return `0`.

## Cartridge Sizes

Default builds try 2K, then 4K. Larger banked cartridges use
`@OPTION MAPPER`:

| Mapper | Size | Banks |
| --- | --- | --- |
| `f8` | 8K | 2 |
| `f6` | 16K | 4 |
| `f4` | 32K | 8 |

Use `@BANK PRG N` for code in a switched bank. See
[`../USAGE.md#banked-builds`](../USAGE.md#banked-builds).

## Not Supported

| Feature | Reason |
| --- | --- |
| General text I/O | No text screen or keyboard |
| `REAL` | No floating-point ROM |
| Bitmap graphics | The display is drawn one scanline at a time |
| Large strings and arrays | RAM is only 128 bytes |
| `ON ERROR` | Not available on this target |
| PAL built-in kernels | Current kernels and note tables are NTSC |

## Examples

See [`../../examples/atari2600/`](../../examples/atari2600/). Portable
SPRITE and TILE examples are under
[`../../examples/atari2600/crustybasic/`](../../examples/atari2600/crustybasic/).
Two row score and text setup is shown in
[`standard_info_rows.cbs`](../../examples/atari2600/crustybasic/standard_info_rows.cbs).
