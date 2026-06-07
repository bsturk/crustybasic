# crustyBASIC on TRS-80 Color Computer

Use `@OPTION TARGET coco` for the default Extended Color BASIC
profile, or pick an exact system with `@OPTION SYSTEM`.

For portable calls such as `CLS`, `POSITION`, `BEEP`, `SOUND`, `JOY`,
`JOY_BUTTON`, `PADDLE`, `PLOT`, `LINE`, and `SPRITE_*`, see
[`../API.md`](../API.md). For command-line target selection and
dialects, see [`../USAGE.md`](../USAGE.md).

## Systems

| System | Output | Notes |
| --- | --- | --- |
| `coco.1` | `.bin` | Color BASIC only. |
| `coco.ecb` | `.bin` | Default. Extended Color BASIC. |
| `coco.3` | `.bin` | CoCo 3 with Super Extended Color BASIC. |
| `coco.1.cart` | `.ccc` | Color BASIC Program Pak. |
| `coco.ecb.cart` | `.ccc` | Extended Color BASIC Program Pak. |
| `coco.3.cart` | `.ccc` | CoCo 3 Program Pak. |

The CoCo target uses the Motorola 6809. It is the only non-6502 target
currently shipped with crustyBASIC.

## Basics

| Item | Value |
| --- | --- |
| CPU | 6809 |
| Text | 32x16 |
| Code start | `$0E00` for `.bin`, `$C000` for cart |
| Program RAM top | `$6000` |
| Text screen | `$0400` |
| Input | Keyboard, 2 analog joystick ports |
| Sound | 6-bit DAC through Color BASIC routines |

Stock text is uppercase plus graphics characters. Lowercase depends on
the machine setup.

## Graphics

The portable graphics API uses PMODE 4-style hi-res graphics. Color
BASIC semigraphics are also available.

Useful calls:

| Call | Purpose |
| --- | --- |
| `DISPLAY mode` | Select text or bitmap graphics. |
| `GFX_COLOR fg, bg` | Set drawing colors. |
| `PLOT x, y`, `UNPLOT x, y` | Draw or clear pixels. |
| `LINE x0, y0, x1, y1` | Draw a line. |
| `SET x, y, color` | Color BASIC semigraphics dot. |
| `RESET x, y` | Clear a semigraphics dot. |

The CoCo has no hardware sprites. `SPRITE_*` is a one-sprite 8x8
hi-res blitter.

## CoCo 3 GIME

`coco.3` exposes the `GIME` namespace for direct register access.
Portable CoCo 3-specific graphics modes are still pending, so use
`GIME.*` only when you are intentionally writing CoCo 3 hardware code.

Common names include `GIME.INIT0`, `GIME.INIT1`, `GIME.IRQENR`,
`GIME.VMODE`, `GIME.VRES`, `GIME.BORDER`, and
`GIME.PALETTE0`..`GIME.PALETTE15`.

## Input

| Capability | Value |
| --- | --- |
| Keyboard | Yes |
| Joystick ports | 2 |
| Buttons | 1 |
| Analog joystick | Yes |
| Paddle axes | 4 |

`PADDLE(AXIS)` reads the raw 0..63 analog value. `JOY(PORT)` converts
the analog stick to a digital direction mask. `JOY_BUTTON(PORT, 0)` reads
the joystick button.

## Sound

`BEEP(DURATION)` toggles the DAC directly. The portable
`SOUND VOICE, FREQ, WAVEFORM, VOLUME` shape is also available through
the sound API.

## Files

The CoCo provider uses Disk BASIC ROM calls. It assumes a disk
controller and Disk BASIC are present at runtime.

Portable `FILE_OPEN` supports sequential `FILE_MODE_READ` and
`FILE_MODE_WRITE`. `FILE_MODE_APPEND`, `FILE_MODE_UPDATE`, and
`FILE_MODE_DIR` are not portable on CoCo and set `FILE_UNSUPPORTED`.

```basic
@OPTION TARGET coco

FILE_OPEN 1, "DATA/TXT", FILE_MODE_WRITE
FILE_WRITE_STR 1, "HELLO"
FILE_WRITE_BYTE 1, 13
FILE_CLOSE 1
```

Use `FILE_OPEN_NATIVE` for CoCo-specific mode and drive control:

```basic
FILE_OPEN_NATIVE 1, "DATA/TXT:1", COCO_FILE_MODE_OUTPUT, 1
```

`AUX1` is the Disk BASIC mode byte:
`COCO_FILE_MODE_INPUT`, `COCO_FILE_MODE_OUTPUT`, or
`COCO_FILE_MODE_DIRECT`. `AUX2` is the default drive number. File names
use the Disk BASIC `NAME/EXT:DRIVE` shape; the explicit drive in the
path overrides `AUX2`.

Target-specific helpers are also callable: `COCO_DISK_OPEN`,
`COCO_DISK_CLOSE`, `COCO_DISK_READ_BYTE`, `COCO_DISK_WRITE_BYTE`,
`COCO_DISK_SET_NAME`, and `COCO_DISK_ENSURE`. These are not portable API
names. Direct files still require Disk BASIC random-record discipline.

## REAL

CoCo `REAL` uses the Color BASIC floating-point format.

| Works on | Details |
| --- | --- |
| `coco.1` | Basic arithmetic, integer powers, and Color BASIC ROM math. |
| `coco.ecb`, `coco.3` | Adds Extended Color BASIC math such as `LOG`, `COS`, `ATN`, and fractional powers. |

`POW(X, N)` with a non-negative integer constant `N` does not need
Extended Color BASIC. Fractional powers do.

## Hardware Names

CoCo 1 and 2 do not expose chip namespaces. Use `POKE` and `PEEK` for
PIA, SAM, and VDG addresses when you need direct hardware access.

CoCo 3 exposes `GIME.*` names as described above.

## Text

Strings use Color BASIC text conventions.

| Escape | Byte |
| --- | --- |
| `{RETURN}`, `{ENTER}` | `$0D` |
| `{TAB}` | Four spaces |
| `{CLEAR}`, `{CLR}`, `{HOME}` | `$0C` |
| `{LEFT}`, `{BACKSPACE}` | `$08` |
| `{RIGHT}` | `$09` |
| `{UP}`, `{DOWN}` | `$03`, `$0A` |

## Cartridges

Use an exact cartridge system:

```basic
@OPTION SYSTEM coco.ecb.cart
```

Default Program Pak output is 8K/16K. For larger cartridges:

```basic
@OPTION SYSTEM coco.ecb.cart
@OPTION MAPPER banked_16k
```

`banked_16k` provides a fixed 8K area plus switched 8K banks. See
[`../USAGE.md#banked-builds`](../USAGE.md#banked-builds) for `@BANK`.

## Dialects

`color_basic` and `extended_color_basic` use CoCo suffix rules: bare
numeric names, `!`, and `#` are `REAL`, `%` is 16-bit integer, and `$`
is `STRING`.

In these dialects, positive `RND(N)` returns an integer from `1`
through `N`. Use `RAND(N)` for portable `0` through `N - 1` values.

## Examples

See [`../../examples/coco/`](../../examples/coco/).
