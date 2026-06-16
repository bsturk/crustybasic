# crustyBASIC on Apple II

Use `@OPTION TARGET apple2` for the default Apple II Plus profile, or
pick an exact system with `@OPTION SYSTEM`.

For portable calls such as `CLS`, `POSITION`, `BEEP`, `JOY`,
`PADDLE`, `PLOT`, `LINE`, and `SPRITE_*`, see
[`../API.md`](../API.md). For command-line target selection and
dialects, see [`../USAGE.md`](../USAGE.md).

## Systems

| System | Machine | Notes |
| --- | --- | --- |
| `apple2.plus` | Apple II Plus | Default. Applesoft ROM, `REAL` supported. |
| `apple2.int` | Original Apple II | Integer BASIC ROM. No `REAL`. |
| `apple2.e` | Apple IIe | Adds lowercase and 80-column softswitches. |
| `apple2.c` | Apple IIc | Same as IIe today, plus one mouse button capability. |

Output is a DOS 3.3 `BRUN` binary (`.bin`). With
`--set disk-image=true`, AppleCommander, and a Java runtime, `build`
also writes a bootable `.dsk`.

## Basics

| Item | Value |
| --- | --- |
| CPU | 6502 |
| Text | 40x24 |
| Code start | `$0800` |
| Program RAM top | `$9600` |
| Text screen | `$0400`, Apple interleaved rows |
| Sound | Speaker click, `BEEP`, `SOUND` |
| Input | Keyboard, 2 joystick ports, 4 paddle axes |

`POSITION COL, ROW` handles the Apple text screen layout. If you poke
screen memory yourself, use the usual Apple II row addressing rules.

`apple2.plus` and `apple2.int` fold text to uppercase. `apple2.e` and
`apple2.c` keep mixed case.

## Graphics

The portable graphics API supports text/cell mode, lo-res graphics,
and hi-res graphics. Apple II has no hardware sprites; `SPRITE_*` is a
software 7x8 hi-res blitter with `SPRITE_MAX_COUNT` slots. By default
`SPRITES_FLUSH` page flips between the two hi-res pages, so sprites
update without tearing; draw the bitmap background after
`SPRITES_BEGIN` or `SPRITES_RESET` so it lands on both pages.

Page-flip programs must fit below hi-res page 1 at $2000. Larger
programs can load above both pages instead with `@OPTION START_CODE
$6000`, trading the low-memory region for ~13.5 KB above $6000.

Useful portable entry points:

| Call | Purpose |
| --- | --- |
| `DISPLAY mode` | Select text, lo-res, or hi-res. |
| `GFX_COLOR fg, bg` | Pick the drawing color. |
| `PLOT x, y` / `LINE x0, y0, x1, y1` | Draw pixels and lines. |
| `SPRITE_DATA id, addr` | Set the 7x8 sprite bitmap. |

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
| `DISPLAY` | `TEXT_ON`, `TEXT_OFF`, `MIXED_ON`, `MIXED_OFF`, `PAGE1`, `PAGE2`, `HIRES_ON`, `HIRES_OFF` |
| `GAME` | `SW0`, `SW1`, `SW2`, `PADDLE0`..`PADDLE3`, `STROBE` |
| `IIE` | `SET40COL`, `SET80COL`, `PRIMARYCHAR`, `ALTCHAR`, `RD80COL`, `RDALTCHAR` |

`IIE` is available on `apple2.e` and `apple2.c`.

## Input

Use `JOY(PORT)` for a digital view of the game controls and
`PADDLE(AXIS)` for the raw 0..255 paddle timer value. `JOY_BUTTON(PORT, N)`
reads pushbuttons.

Capability summary:

| Capability | Value |
| --- | --- |
| Keyboard | Yes |
| Joystick ports | 2 |
| Buttons | 2 per port |
| Paddle axes | 4 |
| Mouse buttons | `apple2.c`: 1, others: 0 |

## Sound

`BEEP(DURATION)` uses the ROM bell. `SOUND(FREQ, DURATION)` toggles the
speaker directly for simple tones.

## Files

The Apple II provider uses DOS 3.3 text-file commands through the
monitor I/O vectors. The program must be running under DOS 3.3.

Portable `FILE_OPEN` supports `FILE_MODE_READ`, `FILE_MODE_WRITE`, and
`FILE_MODE_APPEND`. `FILE_MODE_UPDATE` and `FILE_MODE_DIR` are not
supported.

```basic
@OPTION TARGET apple2

FILE_OPEN 1, "DATA", FILE_MODE_WRITE
FILE_WRITE_STR 1, "HELLO"
FILE_WRITE_BYTE 1, 13
FILE_CLOSE 1
```

Use `FILE_OPEN_NATIVE` when Apple II-specific code wants to pass the DOS
mode directly:

```basic
FILE_OPEN_NATIVE 1, "DATA", FILE_MODE_APPEND, 0
```

Target-specific helpers are also callable: `APPLE_DOS_COMMAND`,
`APPLE_DOS_OPEN`, `APPLE_DOS_CLOSE`, `APPLE_DOS_READ_SELECT`,
`APPLE_DOS_WRITE_SELECT`, `APPLE_DOS_INPUT_RESTORE`,
`APPLE_DOS_OUTPUT_RESTORE`, `APPLE_DOS_READ_RAW_BYTE`, and
`APPLE_DOS_WRITE_RAW_BYTE`. These are DOS 3.3 text-file helpers, not
portable API names.

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

Apple II examples live under:

- [`../../examples/apple2/crustybasic/`](../../examples/apple2/crustybasic/)
- [`../../examples/apple2/dialects/`](../../examples/apple2/dialects/)
