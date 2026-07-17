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
the bundled DOS 3.3 image writer.

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

Stock Apple II text glyphs are in character ROM and are not
RAM-redefinable through runtime charset support.

## Graphics

Apple II graphics support text/cell mode, lo-res graphics, and hi-res
graphics. Apple II has no hardware sprites; sprite calls use a software
7x8 hi-res blitter. The sprite flush path page flips between the two
hi-res pages by default, so sprites update without tearing; draw the
bitmap background after starting or resetting sprites so it lands on
both pages.

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

`DHGR_MONO` uses individual 560 dot bits and is intended for a monochrome
display. A composite color display interprets the same memory as double hires
color.

The double graphics modes do not yet support mixed text, native image
display, software sprites, bitmap tile stamping, or page flipping.

## Images

Native image display supports:

| Format | Accepted files |
| --- | --- |
| `IMAGE_FMT_HGR` | HGR screen dumps, 8184/8188/8192 bytes, native row order |
| `IMAGE_FMT_LGR` | 1024-byte lores text page dumps |

DOS 3.3-backed runtime image loading is available.

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
| `MOCKINGBOARD` | `INIT`, `WRITE`, `RESET`, AY register constants |
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

## Sound

The default `BUILTIN` sound backend uses the ROM bell and toggles the
speaker for simple tones.

A standard Mockingboard provides two AY sound chips and six voices. Set
the card's slot to expose its direct interface:

```basic
@OPTION APPLE_MOCKINGBOARD_SLOT 4
```

Slots 1 through 7 are accepted. There is no default slot and the card is
not detected at runtime. The common slot is 4. Use
`OPTION_SET("APPLE_MOCKINGBOARD_SLOT")` to test whether a slot was set;
`HAS_CHIP("MOCKINGBOARD")` only reports that the target has the interface.

To use the card for portable sound calls, also select it as the backend:

```basic
@OPTION APPLE_MOCKINGBOARD_SLOT 4
@OPTION APPLE_SOUND_BACKEND MOCKINGBOARD
```

`APPLE_SOUND_BACKEND` accepts `BUILTIN` or `MOCKINGBOARD` and defaults
to `BUILTIN`. Selecting `MOCKINGBOARD` without setting a slot is an
error. Setting a slot alone leaves portable sound on the speaker while
making the `MOCKINGBOARD.*` interface available.

With the Mockingboard backend, `SOUND`, `SOUND_RAW`, `SOUND_OFF`,
`SOUND_ALL_OFF`, `BEEP`, `SOUND_FREQ`, `PLAY_NOTE`, `PLAY_NOTE_SHAPE`,
and blocking `PLAY_NOTE_FOR` use the card. Voices 0 through 2 use the
first AY and voices 3 through 5 use the second. Native `SOUND` frequency
values are 12 bit AY periods. Tone and noise shapes are supported, with
a per voice volume from 0 through 15.
`SOUND_AVAILABLE` is `1` for this backend. Timed sound is not
available.

The direct interface initializes itself when portable sound first uses
it. Call `MOCKINGBOARD.INIT` before using the direct interface:

| Call | Purpose |
| --- | --- |
| `MOCKINGBOARD.INIT` | Initialize both AY chips. |
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
