# crustyBASIC on Commodore VIC-20

Use `@OPTION TARGET vic20` for the default unexpanded VIC-20 profile, or
select an exact RAM configuration with `@OPTION SYSTEM`, e.g. `vic20.3k`.

Portable runtime calls are documented in [`../API.md`](../API.md). This page
covers VIC-20 systems, memory layouts, display modes, formats, and hardware
notes. For command-line target selection and dialects, see
[`../USAGE.md`](../USAGE.md).

## Systems

| System | Default output | Notes |
| --- | --- | --- |
| `vic20.orig` | `.prg` | Unexpanded VIC-20, vic20 TARGET defaults to this |
| `vic20.3k` | `.prg` | 3K RAM expansion |
| `vic20.8k` | `.prg` | 8K RAM expansion in BLK1 |
| `vic20.16k` | `.prg` | 16K RAM expansion |
| `vic20.24k` | `.prg` | 24K RAM expansion |

All profiles use the VIC-20 KERNAL and BASIC V2 ROMs. `REAL` uses the BASIC
ROM, and text output supports Commodore control codes.

The unexpanded profile has limited RAM. It is suitable for small
text programs; custom images and software sprites generally require a RAM
expansion.

## Memory

These addresses are relevant when converting an existing BASIC listing,
installing a custom character set, or loading a native image format.

| Item | `vic20.orig` | `vic20.3k` | `vic20.8k` | `vic20.16k` | `vic20.24k` |
| --- | --- | --- | --- | --- | --- |
| Code starts at | `$1010` | `$0410` | `$1210` | `$1210` | `$1210` |
| Program RAM ends at | `$1DFF` | `$1DFF` | `$3FFF` | `$5FFF` | `$7FFF` |
| `SCREEN_BASE` | `$1E00` | `$1E00` | `$1000` | `$1000` | `$1000` |
| `COLOR_BASE` | `$9600` | `$9600` | `$9400` | `$9400` | `$9400` |

`SCREEN_BASE` and `COLOR_BASE` follow the selected system. Programs should
use these constants instead of fixed addresses.

If imported data requires a fixed memory range, relocate the program with
`@OPTION START_CODE` and `@OPTION START_DATA`. `@OPTION START_PROGRAM` is
only required when the BASIC launcher overlaps that range.

Use `CHARSET_COPY_DEFAULT`, `CHARSET_INSTALL`, and `CHARSET_DEFINE` for custom
characters. These helpers reserve `$1800` to `$1BFF` at compile time. Code,
data, and the heap stay outside that area; expansion RAM above it remains
available.

## Colors

Cells show eight colors: `BLACK`, `WHITE`, `RED`, `CYAN`, `PURPLE`,
`GREEN`, `BLUE` and `YELLOW`, and `STD_COLORS` holds exactly those.
`ORANGE` and the `LIGHT_` colors are background colors: `BACKGROUND_COLOR`
takes them, a `LIGHT_` color used for a cell drops to its base color, and
`ORANGE` or `LIGHT_ORANGE` used for a cell is a compile error. There is no
gray or brown, so those names do not exist on this target.

## Display modes

The VIC-I displays a 22 by 23 character grid. Each cell is 8 by 8 pixels,
giving a visible character area of 176 by 184 pixels.

Portable display modes map as follows:

| Portable mode | VIC-20 mode | Result |
| --- | --- | --- |
| `CELL` | `VIC_TEXT` | Normal 22x23 character screen |
| `CELL_MULTICOLOR` | `VIC_CELL_MULTICOLOR` | 22x23 multicolor character screen |
| `BITMAP_HIRES` | Not available | `DISPLAY` returns `0` |
| `BITMAP_LORES` | Not available | `DISPLAY` returns `0` |
| `BITMAP_MULTICOLOR` | Not available | `DISPLAY` returns `0` |

`CELL` is the default. Calling `DISPLAY` without an argument also selects it.
Portable programs should use `CELL` or `CELL_MULTICOLOR`.

The native VIC-20 mode names are:

| VIC-20 mode | Value | Size | Colors | Notes |
| --- | --- | --- | --- | --- |
| `VIC_TEXT` | `$80` | 22x23 cells | One of 8 foreground colors per cell, plus one shared background | Normal VIC-I character mode, equivalent to portable `CELL` |
| `VIC_CELL_MULTICOLOR` | `$81` | 22x23 cells | Four colors in each multicolor character | VIC-I multicolor character mode, equivalent to portable `CELL_MULTICOLOR` |

`CELL` and `VIC_TEXT` select the same hardware mode. `CELL_MULTICOLOR` and
`VIC_CELL_MULTICOLOR` also select the same hardware mode. `DISPLAY_MODE`
retains the mode name passed to `DISPLAY`, allowing portable and VIC-20 mode
names to be distinguished.

All modes report `DISPLAY_WIDTH = 22` and `DISPLAY_HEIGHT = 23`. In either
multicolor mode, each character row contains four pixels. Each pixel is
encoded with 2 bits and displayed at double width. The four colors are the
shared background, shared border, shared auxiliary color, and the cell
foreground.

`DISPLAY mode` returns `1` for all four supported names.
`DISPLAY_HAS_MODE(mode)` reports support without changing the current mode.
`DISPLAY_MODE_COLOR_COUNT` returns `4` for `CELL_MULTICOLOR` and
`VIC_CELL_MULTICOLOR`, and `0` for the two normal character names.

Changing modes does not clear the screen. Call `CLS` to clear it. Switching
to a multicolor mode marks the screen cells as multicolor. Switching to
either normal character name returns them to normal character color.

The VIC-20 has no bitmap display mode. `PLOT`, `POINT`, and mixed text and
bitmap displays are not available. Graphics use cells, tiles, custom
characters, or software sprites.

## Colors

The VIC-I has 16 colors. Background and auxiliary colors can use all
16. Border and character foreground colors are limited to values 0 through
7.

| Value | VIC-20 color | crustyBASIC constant |
| --- | --- | --- |
| `0` | Black | `BLACK` |
| `1` | White | `WHITE` |
| `2` | Red | `RED` |
| `3` | Cyan | `CYAN` |
| `4` | Purple | `PURPLE` |
| `5` | Green | `GREEN` |
| `6` | Blue | `BLUE` |
| `7` | Yellow | `YELLOW` |
| `8` | Orange | `ORANGE` |
| `9` | Light orange | `LIGHT_ORANGE` |
| `10` | Pink | `LIGHT_RED` |
| `11` | Light cyan | `LIGHT_CYAN` |
| `12` | Light purple | `LIGHT_PURPLE` |
| `13` | Light green | `LIGHT_GREEN` |
| `14` | Light blue | `LIGHT_BLUE` |
| `15` | Light yellow | `LIGHT_YELLOW` |

crustyBASIC uses portable color constants, so their names do not always
match the VIC-20 names. `LIGHT_RED` selects pink, and `LIGHT_GRAY` maps to
cyan.

In multicolor mode, every cell has its own foreground color. The background,
border, and auxiliary colors are shared by the whole screen.

```basic
DISPLAY VIC_CELL_MULTICOLOR
CELL_COLOR 10, 5, YELLOW
CELL_COLORS WHITE, BLACK, RED, CYAN
```

`CELL_COLOR` changes the foreground of the cell at column 10, row 5 to
yellow.
`CELL_COLORS` then sets every cell's foreground to white, the shared background to
black, the shared border to red, and the shared auxiliary color to cyan.

Direct VIC-I color control uses the following ranges:

| Use | Range | Where it lives |
| --- | --- | --- |
| Cell foreground | `0..7` | Color RAM |
| Cell multicolor selection | Bit 3 | Color RAM |
| Background | `0..15` | High nibble of `VIC.COLOR` |
| Border | `0..7` | Low bits of `VIC.COLOR` |
| Auxiliary color | `0..15` | High nibble of `VIC.VOLUME` |

```basic
@OPTION TARGET vic20

BG     = LIGHT_BLUE
BORDER = BLUE
AUX    = LIGHT_RED

VIC.COLOR  = (BG * 16) + BORDER
VIC.VOLUME = (AUX * 16) + (VIC.VOLUME & 15)

' set one cell to multicolor with a red foreground
POKEB COLOR_BASE + 5 * TEXT_WIDTH + 10, 8 + RED
```

## Text, cells, and custom characters

Text, cells, and tiles use the same 22 by 23 character screen.

Regular crustyBASIC source keeps the direct byte values for letters:
`A-Z` uses `$41-$5A`, `a-z` uses `$61-$7A`, and a newline uses `$0D`. The
glyph on screen still depends on whether the VIC-20 is showing its upper or
lower character set.

The `cbm_basic_vic20` dialect follows VICE petcat listing conversion instead.
In that dialect, `A-Z` uses `$C1-$DA`, `a-z` uses `$41-$5A`, and `~` uses
`$FF`.

Use `{$NN}` for an exact byte value. The following named escapes cover common
screen controls:

| Escape | Result |
| --- | --- |
| `{RETURN}`, `{ENTER}` | Return, `$0D` |
| `{TAB}` | Four spaces |
| `{CLEAR}`, `{CLR}` | Clear screen |
| `{HOME}` | Move to the home position |
| `{LOWER}`, `{TEXT_LOWER}` | Select the lower character set |
| `{UPPER}`, `{TEXT_UPPER}`, `{GRAPHICS}` | Select the upper and graphics character set |
| `{LEFT}`, `{RIGHT}`, `{UP}`, `{DOWN}` | Move the cursor |
| `{RVS_ON}`, `{RVS_OFF}` | Turn reverse video on or off |

## VIC-I registers

The `VIC` namespace provides direct access to VIC-I registers. Use the
portable APIs for common operations and `VIC.*` names for direct hardware
control.

| Name | What it controls |
| --- | --- |
| `VIC.SCREEN_X`, `VIC.SCREEN_Y` | Screen position |
| `VIC.COLUMNS`, `VIC.ROWS` | Screen size and character height bits |
| `VIC.RASTER` | Current raster line |
| `VIC.CHAR_BASE` | Screen and character memory selection |
| `VIC.LIGHT_PEN_X`, `VIC.LIGHT_PEN_Y` | Light pen position |
| `VIC.POT_X`, `VIC.POT_Y` | Paddle values |
| `VIC.SOUND1`, `VIC.SOUND2`, `VIC.SOUND3` | Bass, alto, and soprano voices |
| `VIC.NOISE` | Noise voice |
| `VIC.VOLUME` | Master volume and auxiliary color |
| `VIC.COLOR` | Background, reverse mode, and border color |

```basic
VIC.COLOR  = $1B
VIC.VOLUME = 15
VIC.SOUND3 = $E0
```

## Images

The VIC-20 image API supports three source formats:

| Input | Format | Display size | Notes |
| --- | --- | --- | --- |
| Indexed PNG or PCX | `IMAGE_FMT_VIC_I_CELLS` | 176x184 | Converted to VIC-I custom characters |
| FCBPaint | `IMAGE_FMT_VIC_I_FCBPAINT` | 168x192 | Native VIC-20 format, PAL only |
| MINIPAINT | `IMAGE_FMT_VIC_I_MINIPAINT` | 160x192 | Native VIC-20 format |

`@INCLUDE_IMAGE` prepares any of these formats during the build. Images can be
embedded in the program or stored in a prepared `.img` file for
`IMAGE_LOAD`. See the [image API](../API.md#image) for the shared include and
loading forms.

Image display requires expanded RAM or a cartridge build. It is not available
to an unexpanded `vic20.orig` PRG.

All three formats support wipes, slides, four step fades, and dissolves.
Both uncompressed and RLE8 images work. Dissolve block sizes are 1, 2, 4,
and 8 native pixels.

Cell images move in 8x8 character steps. Repeated copies of a character
dissolve together. MINIPAINT moves in 4x16 native pixel cells. FCBPaint
moves in 4x8 native pixel cells; its raster colours stay fixed during slides.
MINIPAINT and FCBPaint count each multicolour pixel pair as one native pixel.

FCBPaint keeps control until Space after loading or an incoming effect.
Outgoing effects blank the display and return when finished. Fades briefly blank it
between levels while updating the raster colours.

Cell effects keep a 1012 byte screen buffer; incoming dissolves also keep
1024 bytes of character data. MINIPAINT and FCBPaint share a 4096 byte
effect buffer. Allow room for the source image and the displayed image too.
For cell images on expansions of 8K or more, use `@OPTION START_CODE $2000`
to leave room for the character set. Native paint formats need their own
memory layout; see `examples/vic20/crustybasic/image_types.cbs`.

## Software sprites

The VIC-20 has no hardware sprites. Portable sprite calls draw movable shapes
with custom characters and require an 8K or larger expansion.

| VIC-20 limit | Value |
| --- | --- |
| Horizontal position | Every pixel |
| Vertical position | Every pixel |
| Colors | One foreground color for each sprite |
| Collision | Sprite to sprite and sprite to background checks |
| Updates | Applied by `SPRITES_FLUSH` |
| Sprite slots | Four by default, selected by `SOFT_SPRITE_COUNT` |

## Input

| Feature | VIC-20 support |
| --- | --- |
| Keyboard | Yes |
| Individual held keys | Yes, through `KEY_HELD` |
| Joystick ports | 1 |
| Buttons | 1 |
| Paddles | 2 axes |
| Keypad | No |
| Mouse | No |

The joystick is port 0. Other port numbers read as neutral. Paddle axes 0
and 1 read `VIC.POT_X` and `VIC.POT_Y`. There is no separate paddle trigger,
so trigger reads return released.

`KEY_HELD` checks the keyboard without consuming typed input.

## Timing

`FRAME_WAIT` follows the video refresh: 60 Hz on NTSC, 50 Hz on PAL. It
returns at the start of the bottom border, and `SPRITES_FLUSH` draws at
that same point, so a loop that calls both spends one frame per pass.
`TICKS` uses the KERNAL clock. The KERNAL programs the same timer value on
NTSC and PAL, so that clock runs at about 55 Hz on NTSC and 60 Hz on PAL.
`TICKS_HZ` reports 55 or 60 to match the current `REGION`.

## Sound

The four sound voices map directly to VIC-I:

| Voice | VIC-I register | Sound |
| --- | --- | --- |
| `0` | `VIC.SOUND3` | Soprano |
| `1` | `VIC.SOUND2` | Alto |
| `2` | `VIC.SOUND1` | Bass |
| `3` | `VIC.NOISE` | Noise |

## Files

`FILE_LOAD_PROGRAM(path$)` loads a PRG from disk device 8 using its
stored address. `FILE_LOAD_PROGRAM(path$, dst)` uses `dst` instead.
Both skip the two address bytes and load the remaining file through
KERNAL LOAD. They return zero on success or the KERNAL error code,
without running the loaded code.

File calls default to disk device 8. `OPEN_CH`, `CLOSE_CH`, `GET_CH`,
`PUT_CH`, `PRINT_CH_STR`, `ST`, and `CMD_CH` provide direct KERNAL channel
access and support other device numbers.

## Dialect

`cbm_basic_vic20` follows Commodore BASIC V2 type suffixes. A bare numeric
name, `!`, or `#` is `REAL`; `%` is a 16 bit integer; and `$` is a string.

Commodore BASIC listings often use POKEs to locations 51/52 and 55/56 to
reserve memory. The dialect recognizes fixed values and reserves the matching
top of RAM. Calculated or incomplete values produce a warning requesting an
explicit `@OPTION RAM_TOP $xxxx`.

Use `RAND(N)` for a portable bounded integer. Compatibility `RND` follows the
dialect's Commodore behavior.

## Examples

VIC-20 examples live in [`../../examples/vic20/`](../../examples/vic20/).
