# crustyBASIC on Apple II

Use `@OPTION TARGET apple2` for the default Apple //e profile, or
pick an exact system with `@OPTION SYSTEM`.

Portable runtime calls are documented in [`../API.md`](../API.md).
This page covers Apple II systems, modes, formats, and hardware notes.
For command-line target selection and dialects, see
[`../USAGE.md`](../USAGE.md).

## Systems

| System | Machine | Notes |
| --- | --- | --- |
| `apple2.plus` | Apple II Plus | Applesoft ROM, `REAL` supported. |
| `apple2.int` | Original Apple II | Integer BASIC ROM. No `REAL`. |
| `apple2.e` | Apple //e | Default. Adds lowercase and 80-column softswitches. |
| `apple2.c` | Apple //c | Same as IIe today, plus one mouse button capability. |

Output is a DOS 3.3 `BRUN` binary (`.bin`). With
`--set disk-image=true`, `build` also writes a bootable `.dsk` using
the bundled DOS 3.3 image writer. Files staged by a two-path
`@INCLUDE_*` form go on that disk, uppercased and cut to the 30
characters a DOS 3.3 catalog holds.

## Basics

| Item | Value |
| --- | --- |
| CPU | 6502 |
| Text | 40x24 |
| Code start | `$0800` |
| Program RAM top | `$9600` |
| Text screen | `$0400`, Apple interleaved rows |
| String encoding | Uppercase ASCII on II/II Plus, ASCII on //e,//c |
| Sound | Speaker click; optional six voice Mockingboard |
| Input | Keyboard, 2 joystick ports, 4 paddle axes |

Cursor positioning handles the Apple text screen layout. If you poke
screen memory yourself, use the usual Apple II row addressing rules.

`apple2.plus` and `apple2.int` fold text to uppercase. `apple2.e` and
`apple2.c` keep mixed case.

The runtime automatically uses `$0200-$02FF` and `$0300-$039F` for
changeable data. `$03A0-$03CF` holds saved Applesoft zero-page state.
Programs using their own assembly must not use these ranges or call the
Monitor line editor, which uses page 2 as its input buffer.

Stock Apple II text glyphs are in character ROM and are not
RAM-redefinable through runtime charset support.

## Graphics

Apple II graphics support text/cell mode, lo-res graphics, and hi-res
graphics. Apple II has no hardware sprites; sprite calls use a software
7x8 hi-res blitter. The sprite flush path page flips between the two
hi-res pages by default, so sprites update without tearing; draw the
bitmap background after starting or resetting sprites so it lands on
both pages.

With page flipping enabled, `SPRITES_FLUSH` waits before showing the
finished page. Use it once per game update without a separate `FRAME_WAIT`.
Sprites whose position, color, and shape bytes match the hidden page are
left alone. Changing raw shape bytes in place is supported.

Speed builds use about 3.5 KB of lookup tables to draw sprite rows faster.
With page flipping, speed builds also combine overlapping single tile moves
when their colors match. Default and size builds use the smaller routines.

`SPRITE_DATA_8X8` accepts the portable MSB-left 8x8 format and converts
it automatically. Raw `SPRITE_DATA` uses Apple II-native 7x8 rows: bit 0
is the leftmost pixel, bit 6 is the rightmost, and bit 7 is ignored.
`SPRITE_DATA_TILES` accepts any runtime tile width and height. Tiles are
converted and drawn left to right, then top to bottom, with each tile
covering 7x8 pixels.

Without page flipping, default and size builds update sprites immediately.
Speed builds wait for `SPRITES_FLUSH` so matching single tile moves can erase
the old position and draw the new one together.

Page-flip programs must fit below hi-res page 1 at $2000. Larger
programs can load above both pages instead with `@OPTION START_CODE
$6000`, trading the low-memory region for ~13.5 KB above $6000.

The default graphics mode is `BITMAP_HIRES`.

`DLGR`, `DHGR_COLOR`, and `DHGR_MONO` are full screen modes available
only on `apple2.e` and `apple2.c`. On Apple //e, all three require revision B
video hardware. `DLGR` requires a suitable 80 column auxiliary memory card.
The DHGR modes require an Extended 80 Column Card, normally giving the
machine 128K total. Apple //c has the needed hardware built in. There is no
runtime hardware check, so selecting one of these modes assumes the required
memory and video hardware are present.

| Mode | Apple II mode | Size | Colors |
| --- | --- | --- | --- |
| `CELL` | 40 column text page 1 | 40x24 | text |
| `BITMAP_HIRES` | HGR page 2 | 280x192 | 2 |
| `BITMAP_LORES` | full screen lores | 40x48 | 16 |
| `HGR_PAGE2` | HGR page 2 | 280x192 | 2 |
| `HGR_PAGE2_MIXED` | HGR page 2 mixed text | 280x160 | 2 |
| `HGR_PAGE1` | HGR page 1 | 280x192 | 2 |
| `HGR_PAGE1_MIXED` | HGR page 1 mixed text | 280x160 | 2 |
| `LGR` | full screen lores | 40x48 | 16 |
| `LGR_MIXED` | lores mixed text | 40x40 | 16 |
| `DLGR` | double lores page 1, main and auxiliary memory | 80x48 | 16 |
| `DHGR_COLOR` | double hires page 2, main and auxiliary memory | 140x192 | 16 |
| `DHGR_MONO` | double hires page 2, main and auxiliary memory | 560x192 | 2 |
| `BITMAP_MULTICOLOR` | not supported | | |
| `CELL_MULTICOLOR` | not supported | | |

Portable mixed display is the fixed Apple II split:

```basic
STATUS = DISPLAY_MIXED(BITMAP_HIRES, 20, 4)
STATUS = DISPLAY_MIXED(BITMAP_LORES, 20, 4)
```

Both modes place graphics above text rows 20 through 23. Graphics keep their
normal coordinates, though the bottom 32 hi-res rows or bottom 8 lo-res rows
are hidden. `CLS` clears the bitmap without erasing the text rows. `CELL_CLS`
clears only the four visible text rows. Other modes and row layouts return `0`
without changing the display. Passing `0` for the text row count selects the
normal full screen mode.

Hi-res mixed mode uses page 1. Because that page occupies `$2000-$3FFF`,
hi-res mixed display programs must fit below `$2000` or use
`@OPTION START_CODE $6000` to load above both hi-res pages.

`DHGR_MONO` uses individual 560 dot bits and is intended for a monochrome
display. A composite color display interprets the same memory as double hires
color.

The double graphics modes do not yet support mixed text, native image
display, software sprites, bitmap tile stamping, or page flipping.

### Cell XOR Movement

`GFX_CELL_XOR_MOVE OLD_SRC, OLD_CX, OLD_PY, NEW_SRC, NEW_CX, NEW_PY, W`
removes an XOR-drawn cell strip and draws its replacement in one call. The
source and position may both change. Horizontal and vertical moves use
specialized paths, while diagonal moves use two XOR passes.

Code that already knows the movement axis can avoid the direction check with:

- `GFX_CELL_XOR_MOVE_HORIZONTAL OLD_SRC, OLD_CX, PY, NEW_SRC, NEW_CX, W`
- `GFX_CELL_XOR_MOVE_VERTICAL OLD_SRC, NEW_SRC, CX, OLD_PY, NEW_PY, W`

Each source contains `W` adjacent 8x8 cells stored as eight bytes per cell.
`OLD_CX` and `NEW_CX` are 8-pixel cell columns. `OLD_PY` and `NEW_PY` must be
multiples of eight. The old strip must have been XOR-drawn with the same
source and foreground color. The call does not clip.

## Images

Native image display supports:

| Format | Accepted files |
| --- | --- |
| `IMAGE_FMT_HGR` | HGR screen dumps, 8184/8188/8192 bytes, native row order, with or without a DOS binary header |
| `IMAGE_FMT_LGR` | 1024-byte lores text page dumps |

DOS 3.3-backed runtime image loading is available.
Converted HGR and LGR images use RLE8 automatically when it makes the image
smaller. This works for both embedded images and DOS 3.3 image files.

`IMAGE_WIPE` and `IMAGE_SLIDE` support unpacked HGR images in all four
directions, using 14-dot by 8-line blocks to preserve color alignment.
`IMAGE_FADE_IN` and `IMAGE_FADE_OUT` use a seven-step patterned pixel fade
that preserves the color phase bits. Fade out works on the active HGR page.
The `image_bars` example cycles through fades, wipes, and slides.

`IMAGE_DISSOLVE_IN` reveals scattered pixels from an unpacked HGR image.
`IMAGE_DISSOLVE_OUT` erases the active hires page to black. Both preserve
the hires color bits. Use `0` for the delay to run at full speed.
The optional final block size is 1, 2, 4, or 8 hires dots per side.

The `hgr_slideshow` example uses an embedded image and two DOS 3.3 files,
dissolving each image in and out. Press Space for the next effect; press
1, 2, 4, or 8 to choose the next transition's block size.

NOTE: native DLGR and DHGR image display is not available (yet).

## Hardware Names

Apple II softswitches trigger when accessed. Assigning any value is
enough for write-style switches:

```basic
SPEAKER.CLICK = 0
DISPLAY.TEXT_ON = 0
```

Common chip namespaces:

| Namespace | Useful names |
| --- | --- |
| `KBD` | `DATA`, `STROBE` |
| `SPEAKER` | `CLICK` |
| `MOCKINGBOARD` | `WRITE`, `RESET`, AY register constants |
| `DISPLAY` | `TEXT_ON`, `TEXT_OFF`, `MIXED_ON`, `MIXED_OFF`, `PAGE1`, `PAGE2`, `HIRES_ON`, `HIRES_OFF` |
| `GAME` | `SW0`, `SW1`, `SW2`, `PADDLE0`..`PADDLE3`, `STROBE` |
| `IIE` | `SET40COL`, `SET80COL`, `PRIMARYCHAR`, `ALTCHAR`, `RD80COL`, `RDALTCHAR` |

`IIE` is available on `apple2.e` and `apple2.c`.

## Input

Joystick reads expose a digital view of the game controls. Paddle reads
return the raw 0..255 timer value, and button reads use the Apple
pushbuttons.

Capability summary:

| Capability | Value |
| --- | --- |
| Keyboard | Yes |
| Joystick ports | 2 |
| Buttons | 2 per port |
| Stick type | Analog |
| Paddle axes | 4 |
| Keypad ports | 0 |
| Keypad `INPUT` | No |
| Mouse buttons | `apple2.c`: 1, others: 0 |

## Timing

The runtime tick source advances during frame waits, so it is not free
running. It reports 60 Hz.

On `apple2.e` and `apple2.c`, `FRAME_WAIT` synchronizes with vertical
blanking using the handler for that system. On `apple2.plus` and `apple2.int`,
bare `FRAME_WAIT` advances the logical frame without adding a delay. Counted
frame waits use an approximate 60 Hz software delay on those systems.

## Sound

Portable sound uses the speaker until a sound card is enabled. A standard
Mockingboard provides two AY sound chips and six voices. Enable it with
the slot containing the card:

```basic
MOCKINGBOARD_ENABLE 4
```

Slots 1 through 7 are supported. The common slot is 4. The card is not
detected, so a program can ask the user before calling
`MOCKINGBOARD_ENABLE`. Calls made before it continue to use the speaker.

The stock speaker supports synchronous noise with
`PLAY_NOTE_SHAPE_FOR` and `SOUND_SHAPE_NOISE`. Other stock-speaker shapes
use a tone.

Apple II also provides the target-specific `TONE` call. `PERIOD` is the
speaker half period and `DURATION` is the number of full waves:

```basic
TONE PERIOD, DURATION
TONE PERIOD, DURATION, ASYNC
```

The two-argument form is synchronous. The three-argument form is also
synchronous when `ASYNC` is false. When `ASYNC` is true, the call plays a
short initial part and queues the rest for available `FRAME_WAIT` idle time.
Only Apple //e currently provides that idle playback window. Joystick
sampling uses the same window, so queued tones should not be used while
polling the joystick.

After enabling the card, portable `SOUND_OFF`, `SOUND_ALL_OFF`,
`BEEP`, `PLAY_NOTE`, `PLAY_NOTE_SHAPE`, `PLAY_NOTE_FOR`, and
`PLAY_NOTE_SHAPE_FOR` use the card. Voices 0 through 2 use the first AY
and voices 3 through 5 use the second. Tone and noise shapes are
supported, with a per voice volume from 0 through 15. Exact AY periods
and other register effects use `MOCKINGBOARD.WRITE`.
Timed sound is not available.

`SOUND_VOICES` and the other portable sound constants describe the
maximum runtime interface, so `SOUND_VOICES` is 6 even before the card
is enabled.

| Call | Purpose |
| --- | --- |
| `MOCKINGBOARD_ENABLE SLOT` | Select, initialize, and use the card for portable sound. |
| `MOCKINGBOARD.WRITE CHIP, REG, VALUE` | Write an AY register on chip 0 or 1. |
| `MOCKINGBOARD.RESET` | Silence and reset both AY chips. |

`MOCKINGBOARD.CHIPS`, `MOCKINGBOARD.VOICES`, and
`MOCKINGBOARD.VOLUME_MAX` describe the card. AY register constants are
`TONE_A_FINE`, `TONE_A_COARSE`, `TONE_B_FINE`, `TONE_B_COARSE`,
`TONE_C_FINE`, `TONE_C_COARSE`, `NOISE_PERIOD`, `MIXER`,
`AMPLITUDE_A`, `AMPLITUDE_B`, `AMPLITUDE_C`, `ENVELOPE_FINE`,
`ENVELOPE_COARSE`, and `ENVELOPE_SHAPE` under the same namespace.

## Files

The Apple II provider uses DOS 3.3 text-file commands through the
monitor I/O vectors. The program must be running under DOS 3.3.

Sequential read, write, and append are supported. Update and directory
modes are not supported. Apple II-specific opens can pass the DOS mode
directly.

Target-specific helpers are also callable: `APPLE_DOS_COMMAND`,
`APPLE_DOS_OPEN`, `APPLE_DOS_CLOSE`, `APPLE_DOS_READ_SELECT`,
`APPLE_DOS_WRITE_SELECT`, `APPLE_DOS_INPUT_RESTORE`,
`APPLE_DOS_OUTPUT_RESTORE`, `APPLE_DOS_READ_RAW_BYTE`, and
`APPLE_DOS_WRITE_RAW_BYTE`. These are DOS 3.3 text-file helpers.

DOS 3.3 text files are line oriented. For clean command handling, end
written records with byte `13` before closing or switching output.

## REAL And Dialects

`REAL` uses Applesoft ROM math on `apple2.plus`, `apple2.e`, and
`apple2.c`. It is an error on `apple2.int`.

`applesoft_basic` follows Applesoft suffix rules: bare numeric names,
`!`, and `#` are `REAL`, `%` is 16-bit integer, and `$` is `STRING`.

`integer_basic` uses 16-bit integers for bare numeric names and `$` for
strings. Its `RND(N)` returns an integer from `0` through `N - 1`.

## Text Escapes

| Escape | Byte |
| --- | --- |
| `{CR}`, `{RETURN}`, `{ENTER}` | `$0D` |
| `{TAB}` | Four spaces |

## Examples

Apple II specific examples live under:

- [`../../examples/apple2/`](../../examples/apple2/)
