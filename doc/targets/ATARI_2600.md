# crustyBASIC on Atari 2600

Use `@OPTION TARGET atari2600` for Atari 2600 / VCS cartridges.
The system name is `atari2600.orig`. Output is a raw `.a26` cartridge
image.

This target is different from the home-computer targets: there is no
keyboard, no operating system, no `REAL`, and only 128 bytes of RAM.
Programs should be small and should draw a frame every game tick.

For portable calls that exist here, such as `JOY`, `JOY_BUTTON`,
`PADDLE`, `RND`, `PEEK`, `SOUND`, and `IMAGE_DISPLAY`, see
[`../API.md`](../API.md).

## Basics

| Item | Value |
| --- | --- |
| CPU | 6507, 6502 family |
| Output | `.a26` |
| Code start | `$F000` for 4K carts |
| RAM | `$80-$FF`, 128 bytes total |
| User data | `$B4-$F3`, 64 bytes |
| Text | Target playfield text and two-player sprite text |
| `REAL` | Not supported |

The CPU stack shares the same 128 bytes of RAM. Keep globals, arrays,
and strings tiny. A default `STRING` is too large for this target.

## Drawing Frames

`A2600_DRAWSCREEN` draws one frame. Call it once per game loop, or use
`A2600_DELAY` when you need to wait while keeping the video signal
alive. The current frame timing defaults to NTSC. Region options expose
`REGION_NTSC`, `REGION_PAL`, and `REGION`.

```basic
@OPTION TARGET atari2600

PROC MAIN
    COLOR AS U8

    WHILE 1
        A2600_BGCOLOR COLOR
        COLOR = COLOR + 1
        A2600_DRAWSCREEN
    ENDWHILE
ENDPROC
```

Display helpers:

| Call | Purpose |
| --- | --- |
| `A2600_DRAWSCREEN` | Draw one frame |
| `A2600_DELAY COUNT` | Draw `COUNT` frames |
| `A2600_FRAME` | 8-bit frame counter |
| `A2600_BGCOLOR C` | Background color |
| `A2600_PFCOLOR C` | Playfield and ball color |
| `A2600_PLAYFIELD PF0, PF1, PF2` | Set the static playfield shadow |
| `A2600_PLAYFIELD_CTRL FLAGS` | Set reflect, score, priority, and ball size bits |

Useful playfield control flags:

| Constant | Purpose |
| --- | --- |
| `A2600_PF_REFLECT` | Mirror the left playfield half |
| `A2600_PF_SCORE` | Use player colors for left and right playfield halves |
| `A2600_PF_PRIORITY` | Draw playfield over players |

Player 0 and player 1 have matching calls:

| Call | Purpose |
| --- | --- |
| `A2600_P0COLOR C`, `A2600_P1COLOR C` | Player colors |
| `A2600_PLAYER0_X X`, `A2600_PLAYER1_X X` | Player X position |
| `A2600_PLAYER0_Y Y`, `A2600_PLAYER1_Y Y` | Player Y position |
| `A2600_PLAYER0_SHAPE ADDR`, `A2600_PLAYER1_SHAPE ADDR` | 8-byte shape |
| `A2600_PLAYER0_GRAPH V`, `A2600_PLAYER1_GRAPH V` | Direct graphics byte |
| `A2600_PLAYER0_HIDE`, `A2600_PLAYER1_HIDE` | Hide a player |

Shape data belongs in a typed `CONST` byte array:

```basic
CONST SMILEY(7) AS U8 = $3C, $42, $A5, $81, $A5, $99, $42, $3C
A2600_PLAYER0_SHAPE ADDR SMILEY
```

## Kernel Families

Atari 2600 video is selected at compile time. With no TILE or playfield
text APIs, the target uses `KERNEL_STANDARD`. Using TILE or playfield
text selects `KERNEL_PLAYFIELD_TILE`. An explicit option such as
`@OPTION A2600_KERNEL KERNEL_PLAYFIELD_TILE` wins.

| Kernel | Purpose |
| --- | --- |
| `KERNEL_NONE` | No built in display kernel |
| `KERNEL_STANDARD` | Static PF0/PF1/PF2 plus two players, with missiles, ball, score/status, per row color/background/height tables, and analog paddle options (this is the default) |
| `KERNEL_PLAYFIELD_TILE` | TILE and playfield text surface (20 or 40 columns) |
| `KERNEL_MULTISPRITE` | Player reuse profile for more visible sprite entries |

Constants include `KERNEL_NONE`, `KERNEL_STANDARD`,
`KERNEL_PLAYFIELD_TILE`, `KERNEL_MULTISPRITE`, and
`A2600_KERNEL_PROFILE`.

The `standard` kernel also provides `A2600_MISSILE0_X`,
`A2600_MISSILE1_X`, and `A2600_BALL_X` for demos that need stable
missile/ball positioning without inline timing code.

When any per row option is selected the standard kernel switches to a
row based visible loop that draws players, missiles, score and status
bands, and the per row tables together. With per row heights (or an
explicit `A2600_PLAYFIELD_ROW_SCANLINES`) the kernel clamps the row
table to 192 visible scanlines and blank-pads any remainder, so height
tables that do not sum exactly still produce a stable frame.

`KERNEL_STANDARD` usage also has the 20-column TILE kernel so a
program can switch into TILE mode at runtime (`TILE_BEGIN` sets a mode
bit). Set `@OPTION A2600_TILE_RUNTIME OFF` to drop that kernel and
reclaim the ROM when the program never uses TILE or playfield text; a
`TILE_BEGIN` call in such a build silently keeps showing the static
playfield.

## Playfield Layout

`A2600_PLAYFIELD_LAYOUT` controls how logical cells map to TIA
playfield bits:

| Layout | Meaning |
| --- | --- |
| `PLAYFIELD_REPEAT` | 20 cells repeated on the right half |
| `PLAYFIELD_REFLECT` | 20 cells mirrored on the right half |
| `PLAYFIELD_ASYMMETRIC` | 40 unique cells across the full width |

The default layout is `PLAYFIELD_REFLECT`. Unique full width needs
`A2600_PLAYFIELD_COLUMNS 40` or layout `PLAYFIELD_ASYMMETRIC` (either
implies the other).

The TILE path supports 20-column repeat/reflect rows and an 8-row
40-column asymmetric path. The 40-column TILE path has no sprite
overlay but supports the ball, and `A2600_DRAWSCREEN` drives it
directly.

Player sprites on the 20-column TILE kernel are opt-in. Set
`@OPTION A2600_PLAYFIELD_SPRITES ON` to enable the two-player overlay
for `A2600_PLAYER0_X/Y/GRAPH`, `A2600_PLAYER1_*`, and the hide PROCs.
The overlay costs 6 bytes of RAM plus kernel cycles every scanline, so
it is off by default. Calling a player PROC on the TILE kernel without
the option fails at assemble time with an undefined
`__CB_ERROR_SET_OPTION_A2600_PLAYFIELD_SPRITES_ON` reference.

## Images

Atari 2600 images use `IMAGE_FMT_PF_ROWS`: one display-ready TIA
register triplet (`PF0`, `PF1`, `PF2`) per row for images up to 20
pixels wide, or two triplets per row (left half, right half) for wider
images. The aux byte's bit 0 mirrors the left half onto the right.
The converter builds descriptors from 8-bit indexed PNG files up to 40
pixels wide (nonzero pixels become playfield bits):

```sh
tools/bin/cb-image --target atari2600 --name A2600 --mirror pf.png -o pf.cbi
```

Rows draw through `KERNEL_PLAYFIELD_TILE`, so select that kernel and
matching playfield rows and columns before displaying the image:

```basic
@OPTION TARGET atari2600
@OPTION A2600_KERNEL KERNEL_PLAYFIELD_TILE

OK = IMAGE_DISPLAY(ADDR A2600_IMAGE)
```

Rows and columns beyond the configured kernel grid are clipped. See
`image.cbs` in the examples directory.

## Kernel Options

Useful options:

| Option | Values |
| --- | --- |
| `A2600_PLAYFIELD_COLUMNS` | `20`, `40` |
| `A2600_PLAYFIELD_ROWS` | `8`, `12`, `16`, `24` |
| `A2600_PLAYFIELD_ROW_SCANLINES` | Row height in scanlines |
| `A2600_SCORE` | `ON`, `OFF` |
| `A2600_STATUS_BAND` | `BAND_NONE`, `BAND_SCORE`, `BAND_TEXT` |
| `A2600_MISSILES` | `ON`, `OFF` |
| `A2600_BALL` | `ON`, `OFF` |
| `A2600_COLLISIONS` | `ON`, `OFF` |
| `A2600_PLAYER_COLORS` | `PLAYER_COLORS_SINGLE`, `PLAYER_COLORS_PLAYER1_PER_ROW`, `PLAYER_COLORS_PER_ROW` |
| `A2600_PLAYFIELD_COLORS` | `PLAYFIELD_COLORS_SINGLE`, `PLAYFIELD_COLORS_PER_ROW` |
| `A2600_BACKGROUND_COLORS` | `BACKGROUND_COLORS_SINGLE`, `BACKGROUND_COLORS_PER_ROW` |
| `A2600_PLAYFIELD_HEIGHTS` | `PLAYFIELD_HEIGHTS_FIXED`, `PLAYFIELD_HEIGHTS_PER_ROW` |
| `A2600_INPUT_MODE` | `INPUT_STANDARD`, `INPUT_TIMED_PADDLE` |
| `A2600_TILE_RUNTIME` | `ON`, `OFF` (default `ON`) |
| `A2600_MULTISPRITE_COUNT` | `4`, `6` |
| `A2600_MULTISPRITE_COLORS` | `ON`, `OFF` (default `OFF`) |

Standard game tradeoffs:

| Option | Tradeoff |
| --- | --- |
| `A2600_PLAYER_COLORS PLAYER_COLORS_PLAYER1_PER_ROW` | Missile 1 state is unavailable |
| `A2600_PLAYER_COLORS PLAYER_COLORS_PER_ROW` | Missile 0 and missile 1 state are unavailable |
| `A2600_INPUT_MODE INPUT_TIMED_PADDLE` | Standard kernel only; missile 0 state is unavailable |
| `A2600_BACKGROUND_COLORS BACKGROUND_COLORS_PER_ROW` | Requires per-row playfield colors or per-row playfield heights |
| `A2600_PLAYFIELD_COLORS PLAYFIELD_COLORS_PER_ROW` | Adds a playfield color table |
| `A2600_PLAYFIELD_HEIGHTS PLAYFIELD_HEIGHTS_PER_ROW` | Adds a playfield height table |

`KERNEL_MULTISPRITE` only supports `A2600_MULTISPRITE_COUNT` and
`A2600_MULTISPRITE_COLORS` from this option group. DPC+, PXE, CDF,
SARA, and Superchip kernels are not part of this target pass.

Generated constants include `A2600_MISSILE0_AVAILABLE`,
`A2600_MISSILE1_AVAILABLE`, `A2600_BALL_AVAILABLE`,
`A2600_INPUT_MODE`, `A2600_PLAYFIELD_VISIBLE_SCANLINES`,
`A2600_PLAYFIELD_VBLANK_SCANLINES`, `A2600_PLAYFIELD_OVERSCAN_SCANLINES`,
and matching `A2600_TILE_*` names.

Small table APIs are available on kernels that use them:

| Call | Purpose |
| --- | --- |
| `A2600_SCORE_SET VALUE` | Set score/status value |
| `A2600_STATUS_SET LEFT, RIGHT` | Set two status values |
| `A2600_PLAYER0_ROW_COLOR ROW, C` | Set player 0 row color |
| `A2600_PLAYER1_ROW_COLOR ROW, C` | Set player 1 row color |
| `A2600_PLAYFIELD_ROW_COLOR ROW, C` | Set playfield row color |
| `A2600_BACKGROUND_ROW_COLOR ROW, C` | Set background row color |
| `A2600_PLAYFIELD_ROW_HEIGHT ROW, H` | Set playfield row height |
| `A2600_MULTISPRITE_X/Y/GRAPH ID, VALUE` | Set multisprite table entries |
| `A2600_MULTISPRITE_COLOR ID, C` | Set a per-sprite color (`A2600_MULTISPRITE_COLORS ON`) |

## Text

Atari 2600 text is target-specific. It is not a portable text screen.
Playfield text draws 3x5 glyphs into the TILE playfield. Sprite text
uses the two player objects and can show two 8x8 characters.

| Option | Purpose |
| --- | --- |
| `A2600_FONT "default"` | Load the default playfield font |
| `A2600_SPRITE_FONT "default"` | Load the default sprite font |
| `A2600_TEXT_RENDERER TEXT_PLAYFIELD` | Default `A2600_TEXT` renderer |
| `A2600_TEXT_RENDERER TEXT_SPRITE` | Use player sprite text by default |

| Call | Purpose |
| --- | --- |
| `A2600_TEXT X, Y, S$` | Draw text with the active renderer |
| `A2600_TEXT_CLEAR X, Y, COUNT` | Clear text with the active renderer |
| `A2600_TEXT_MODE MODE` | Switch between playfield and sprite renderers when both are loaded |
| `A2600_PLAYFIELD_TEXT X, Y, S$` | Draw playfield text |
| `A2600_PLAYFIELD_TEXT_CLEAR X, Y, COUNT` | Clear playfield text |
| `A2600_SPRITE_TEXT X, Y, S$` | Draw up to two sprite text characters |
| `A2600_SPRITE_TEXT_CLEAR X, Y, COUNT` | Hide sprite text players |
| `PRINT_AT X, Y, S$` | Portable spelling that maps to `A2600_TEXT` |

The current text routines use small fixed strings. Keep strings short;
the built-in Atari 2600 text PROCs accept `STRING * 5`, and sprite text
uses `STRING * 2`. Loading both playfield and sprite text in the same
program costs more RAM, so the examples keep those renderers separate.

## Input

| Call | Purpose |
| --- | --- |
| `PAD1`, `PAD2` | Active-high direction and fire mask |
| `JOY(PORT)` | Portable direction mask |
| `JOY_BUTTON(PORT, 0)` | Fire button |
| `PADDLE(AXIS)` | Raw INPT bit, or timed paddle value with analog mode |
| `CONSOLE` | Reset, Select, TV type, and difficulty switches |
| `A2600_CONSOLE_RESET` | Reset switch |
| `A2600_CONSOLE_SELECT` | Select switch |
| `A2600_CONSOLE_BW` | Color/BW switch bit |
| `A2600_CONSOLE_P0_DIFFICULTY` | Player 0 difficulty bit |
| `A2600_CONSOLE_P1_DIFFICULTY` | Player 1 difficulty bit |

Portable direction bits are up, down, left, right, and fire in bits
0 through 4.

Atari 2600 paddles are timing devices. `INPT0` through `INPT3` only
report whether each paddle has tripped. In the default
`INPUT_STANDARD` input mode, `PADDLE` returns that raw TIA bit.

For a position value, use `@OPTION A2600_INPUT_MODE INPUT_TIMED_PADDLE`.
The selected display kernel then discharges the paddles during blanking,
times them during the visible frame, and makes `PADDLE(AXIS)` return the
timed value. This is not a separate display kernel.

## Sound

| Call | Purpose |
| --- | --- |
| `SOUND CH, FREQ, CTRL, VOL` | Set one TIA voice |
| `SOUND_OFF CH` | Silence one voice |
| `SOUND_ALL_OFF` | Silence both voices |
| `BEEP DURATION` | Short tone on channel 0 |

`CH` is `0` or `1`. `CTRL` is the TIA waveform value `0..15`,
`FREQ` is `0..31`, and `VOL` is `0..15`.

## Low Level Hardware

Direct TIA and RIOT register names are available when you need them.
The target also exposes small helper PROCs for common direct use.

| Area | Calls |
| --- | --- |
| Direct playfield | `A2600_PLAYFIELD_DIRECT`, `A2600_PF0_DIRECT`, `A2600_PF1_DIRECT`, `A2600_PF2_DIRECT` |
| Player control | `A2600_PLAYER*_REFLECT`, `A2600_PLAYER*_CTRL`, `A2600_PLAYER*_VDELAY`, `A2600_PLAYER*_FINE_MOTION` |
| Player size/copies | `A2600_NUSIZ_ONE_COPY`, `A2600_NUSIZ_TWO_CLOSE`, `A2600_NUSIZ_TWO_MEDIUM`, `A2600_NUSIZ_THREE_CLOSE`, `A2600_NUSIZ_TWO_WIDE`, `A2600_NUSIZ_DOUBLE`, `A2600_NUSIZ_THREE_MEDIUM`, `A2600_NUSIZ_QUAD` |
| Missile control | `A2600_MISSILE0_ENABLE`, `A2600_MISSILE1_ENABLE`, `A2600_MISSILE0_RESET`, `A2600_MISSILE1_RESET`, `A2600_MISSILE0_LOCK`, `A2600_MISSILE1_LOCK`, `A2600_MISSILE0_FINE_MOTION`, `A2600_MISSILE1_FINE_MOTION` |
| Missile size | `A2600_NUSIZ_MISSILE_1`, `A2600_NUSIZ_MISSILE_2`, `A2600_NUSIZ_MISSILE_4`, `A2600_NUSIZ_MISSILE_8` |
| Ball control | `A2600_BALL_ENABLE`, `A2600_BALL_RESET`, `A2600_BALL_SIZE`, `A2600_BALL_VDELAY`, `A2600_BALL_FINE_MOTION` |
| Ball size | `A2600_BALL_SIZE_1`, `A2600_BALL_SIZE_2`, `A2600_BALL_SIZE_4`, `A2600_BALL_SIZE_8` |
| Motion | `A2600_HMOVE`, `A2600_HMCLR` |
| Timers | `A2600_INTIM`, `A2600_TIMINT`, `A2600_TIM1T`, `A2600_TIM8T`, `A2600_TIM64T`, `A2600_T1024T` |
| Collisions | `A2600_COLLISION_CLEAR` and `A2600_HIT_*` helpers |

Collision helpers read TIA latch bits. Read them after a frame has
been drawn, then call `A2600_COLLISION_CLEAR` before the next frame.

| Collision helper | Pair |
| --- | --- |
| `A2600_HIT_PLAYER0_PLAYER1` | Player 0 with player 1 |
| `A2600_HIT_PLAYER0_PLAYFIELD`, `A2600_HIT_PLAYER1_PLAYFIELD` | Player with playfield |
| `A2600_HIT_PLAYER0_BALL`, `A2600_HIT_PLAYER1_BALL` | Player with ball |
| `A2600_HIT_MISSILE0_PLAYER0`, `A2600_HIT_MISSILE0_PLAYER1` | Missile 0 with player |
| `A2600_HIT_MISSILE1_PLAYER0`, `A2600_HIT_MISSILE1_PLAYER1` | Missile 1 with player |
| `A2600_HIT_MISSILE0_PLAYFIELD`, `A2600_HIT_MISSILE1_PLAYFIELD` | Missile with playfield |
| `A2600_HIT_MISSILE0_BALL`, `A2600_HIT_MISSILE1_BALL` | Missile with ball |
| `A2600_HIT_MISSILE0_MISSILE1` | Missile 0 with missile 1 |
| `A2600_HIT_BALL_PLAYFIELD` | Ball with playfield |

Useful register names:

| Namespace | Useful names |
| --- | --- |
| `TIA` | `COLUBK`, `COLUPF`, `COLUP0`, `COLUP1`, `PF0`, `PF1`, `PF2`, `GRP0`, `GRP1`, `ENAM0`, `ENAM1`, `ENABL`, `WSYNC`, `HMOVE`, `HMCLR`, `CXCLR`, `INPT0`..`INPT5` |
| `RIOT` | `SWCHA`, `SWCHB`, `INTIM`, `TIMINT`, `TIM1T`, `TIM8T`, `TIM64T`, `T1024T` |

## Cartridge Sizes

Default builds try 2K, then 4K. Larger banked carts use `@OPTION
MAPPER`:

| Mapper | Size | Banks |
| --- | --- | --- |
| `f8` | 8K | 2 |
| `f6` | 16K | 4 |
| `f4` | 32K | 8 |

Use `@BANK PRG N` for code that belongs in a switched bank. See
[`../USAGE.md#banked-builds`](../USAGE.md#banked-builds).

## Not Supported

| Feature | Reason |
| --- | --- |
| Portable `PRINT`, `INPUT`, `GET` | No portable text screen or keyboard |
| `REAL` | No floating-point ROM |
| Large strings and arrays | RAM is only 128 bytes |
| Portable bitmap graphics | The 2600 draws one scanline at a time |
| `ON ERROR` | Not available on this target |

## Examples

See [`../../examples/atari2600/crustybasic/`](../../examples/atari2600/crustybasic/).

| Topic | Examples |
| --- | --- |
| Kernel families | `static_kernel.cbs`, `playfield_tile_kernel.cbs`, `standard_game_kernel.cbs`, `multisprite_kernel.cbs`, `rich_playfield_kernel.cbs` |
| Basics | `blank_color.cbs`, `beep.cbs`, `playfield.cbs`, `sprite.cbs` |
| Input | `joystick.cbs`, `paddle_raw.cbs`, `hardware_status.cbs` |
| TILE and text | `tile_border.cbs`, `text.cbs`, `playfield_text_20.cbs`, `playfield_text_40.cbs`, `sprite_text.cbs`, `text_modes.cbs`, `playfield_repeat_vs_reflect.cbs` |
| Images | `image.cbs` |
| Standard options | `standard_per_row_colors.cbs`, `standard_analog_paddle.cbs`, `standard_score_status.cbs` |
| Low level TIA | `player_controls.cbs`, `missile_ball.cbs`, `collision_status.cbs` |
| Banked carts | `f8_bank.cbs`, `f6_bank.cbs`, `f4_bank.cbs` |
