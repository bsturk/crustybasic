# crustyBASIC on TRS-80 Color Computer

Use `@OPTION TARGET coco` for the default Extended Color BASIC
profile, or pick an exact system with `@OPTION SYSTEM`.

Portable runtime calls are documented in [`../API.md`](../API.md).
This page covers CoCo systems, modes, formats, and hardware notes. For
command-line target selection and dialects, see
[`../USAGE.md`](../USAGE.md).

## Systems

| System | Output | Notes |
| --- | --- | --- |
| `coco.1` | `.bin` | Color BASIC only. |
| `coco.ecb` | `.bin` | Default. Extended Color BASIC. |
| `coco.3` | `.bin` | CoCo 3 with Super Extended Color BASIC. |

Program Pak `.ccc` output is selected with `OUTPUT_TYPE cart` on the
same systems.

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
| String encoding | Color BASIC ROM text |
| Input | Keyboard, 2 analog joystick ports |
| Sound | 6-bit DAC through Color BASIC routines |

Stock text is uppercase plus graphics characters. Lowercase depends on
the machine setup.

## Graphics

CoCo graphics use PMODE 4-style hi-res graphics. Color BASIC
semigraphics are also available. The CoCo has no hardware sprites;
sprite calls use a one-sprite 8x8 hi-res blitter.

The default graphics mode is `BITMAP_MULTICOLOR`.

| Mode | CoCo mode | Size | Colors |
| --- | --- | --- | --- |
| `CELL` | VDG alphanumeric text | 32x16 | text |
| `BITMAP_HIRES` | PMODE 4 bitmap | 256x192 | 2 |
| `BITMAP_LORES` | not supported | | |
| `BITMAP_MULTICOLOR` | semigraphics 4 | 64x32 | 8 |
| `CELL_MULTICOLOR` | not supported | | |

`GFX_PLOT_COLOR_AVAILABLE` is `TRUE` in semigraphics modes and `FALSE` in
PMODE 4. PMODE 4 uses one shared hardware colorset, so three-argument
`PLOT` falls back to the current foreground pixel.

The bitmap TILE backend defaults to `BITMAP_HIRES`, so
`TILE_COLOR_AVAILABLE` is `FALSE` after `DISPLAY TILE_DEFAULT_DISPLAY_MODE`.
It becomes `TRUE` after `DISPLAY BITMAP_MULTICOLOR`.

## Images

Native image display supports `IMAGE_FMT_COCO_PM4` for 6144-byte PMODE
4 screens. The converter accepts raw PMODE 4 dumps and DECB `.BIN` or
`.MAX` files. The descriptor aux byte's bit 0 selects the VDG colorset.
Disk BASIC-backed runtime image loading is available.

## CoCo 3 GIME

`coco.3` exposes the `GIME` namespace for direct register access.
CoCo 3-specific shared graphics modes are still pending, so use
`GIME.*` only when you are intentionally writing CoCo 3 hardware code.

Common names include `GIME.INIT0`, `GIME.INIT1`, `GIME.IRQENR`,
`GIME.VMODE`, `GIME.VRES`, `GIME.BORDER`, and
`GIME.PALETTE0`..`GIME.PALETTE15`.

## Input

| Capability | Value |
| --- | --- |
| Keyboard | Yes |
| Individual held keys | Yes |
| Joystick ports | 2 |
| Buttons | 1 per port |
| Stick type | Analog |
| Paddle axes | 4 |
| Keypad ports | 0 |
| Keypad `INPUT` | No |
| Mouse buttons | 0 |

Paddle reads return the raw 0..63 analog value. Joystick reads convert
the analog stick to a digital direction mask.

`KEY_HELD` polls the keyboard matrix without consuming typed input.

## Timing

The runtime tick source is Color BASIC `TIMER` at `$0112`. It is a
16-bit 60 Hz counter, so it wraps about every 18 minutes. Cassette and
sound mask the IRQ and pause it; cartridge builds may not have the
BASIC IRQ running.

## Sound

The bell toggles the DAC directly. Sound uses the same 6-bit DAC path.

## Files

The CoCo provider uses Disk BASIC ROM calls. It assumes a disk
controller and Disk BASIC are present at runtime.

Sequential read and write are supported. Append, update, and directory
modes report unsupported. CoCo-specific opens can pass the Disk BASIC
mode and drive number directly.

`AUX1` is the Disk BASIC mode byte:
`DISK_FILE_MODE_INPUT`, `DISK_FILE_MODE_OUTPUT`, or
`DISK_FILE_MODE_DIRECT`. `AUX2` is the default drive number. File names
use the Disk BASIC `NAME/EXT:DRIVE` shape; the explicit drive in the
path overrides `AUX2`.

Target-specific helpers are also callable: `DISK_OPEN`,
`DISK_CLOSE`, `DISK_READ_BYTE`, `DISK_WRITE_BYTE`,
`DISK_SET_NAME`, and `DISK_ENSURE`. Direct files still
require Disk BASIC random-record discipline.

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

Select Program Pak output with `OUTPUT_TYPE cart`:

```basic
@OPTION SYSTEM coco.ecb
@OPTION OUTPUT_TYPE cart
```

Default Program Pak output is 8K/16K. For larger cartridges:

```basic
@OPTION SYSTEM coco.ecb
@OPTION OUTPUT_TYPE cart
@OPTION MAPPER banked_16k
```

`banked_16k` provides a fixed 8K area plus switched 8K banks. See
[`../USAGE.md#banked-builds`](../USAGE.md#banked-builds) for `@BANK`.

## Assembler

CoCo builds use `vasm6809_oldstyle` through `vasm`. Release bundles do
not include the assembler binary. Place it at
`tools/assemblers/vasm/<platform>/vasm6809_oldstyle` beside the compiler,
using the `.exe` suffix on Windows, or pass its path with `--assembler`.

## Dialects

`color_basic` and `extended_color_basic` use CoCo suffix rules: bare
numeric names, `!`, and `#` are `REAL`, `%` is 16-bit integer, and `$`
is `STRING`.

In these dialects, positive `RND(N)` returns an integer from `1`
through `N`. Use `RAND(N)` for portable `0` through `N - 1` values.

## Examples

See [`../../examples/coco/`](../../examples/coco/).
