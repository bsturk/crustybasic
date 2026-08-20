# crustyBASIC on Tandy Color Computer

Use `@OPTION TARGET coco` for the default Extended Color BASIC
profile, or select an exact machine with `@OPTION SYSTEM`.

Portable runtime calls are documented in [`../API.md`](../API.md).
This page covers CoCo systems, current library support, known limits,
and the remaining work found in the July 2026 library audit. For
command-line target selection and dialects, see
[`../USAGE.md`](../USAGE.md).

## Systems

| System | Default output | ROM expected |
| --- | --- | --- |
| `coco.1` | `.bin` | Color BASIC |
| `coco.ecb` | `.bin` | Extended Color BASIC |
| `coco.3` | `.bin` | Super Extended Color BASIC |

`coco.ecb` is the default. Program Pak `.ccc` output is selected with
`OUTPUT_TYPE cart` on the same systems.

## Basics

| Item | Value |
| --- | --- |
| CPU | Motorola 6809E |
| Stock video | MC6847 VDG through the MC6883 SAM |
| CoCo 3 video | GIME |
| Text | 32x16 stock VDG text |
| Code start | `$0E00` for `.bin`, `$C000` for standard cart |
| Program RAM top | `$6000` |
| Text screen | `$0400` |
| PMODE 4 screen | `$6000` to `$77FF` |
| Page-flip back screen | `$3800` to `$4FFF` |
| String encoding | Color BASIC ROM text |
| Input | 8x7 keyboard matrix and two analog joystick ports |
| Sound | Direct 6-bit DAC output through `$FF20` |

Stock text is uppercase plus graphics characters. Lowercase depends on
the machine setup.

## Current support at a glance

The status words in this table are deliberate:

- **Runtime verified** means the result was seen running in MAME.
- **Implemented** means the library and focused compiler tests cover it,
  but this audit did not exercise the hardware behavior.
- **Partial** means useful behavior exists but the public surface
  promises more.
- **Runtime requirement** means the library assumes optional ROM or
  hardware support that is not present on every CoCo.

| Area | Status | What works now |
| --- | --- | --- |
| `.bin` and `.ccc` output | Runtime verified | Standard Program Pak output starts and runs |
| Stock text | Implemented | 32x16 output and direct cell access |
| Semigraphics | Runtime verified | Semigraphics 4, 64x32, 8 colors |
| Hi-res graphics | Runtime verified | PMODE 4, 256x192, 2 colors |
| Graphics primitives | Runtime verified | Plot, point, horizontal line, line, box, filled box, and circle |
| Bitmap tiles | Implemented | 8x8 PMODE 4 tile drawing |
| Images | Implemented | Native 6144-byte PMODE 4 display and Disk BASIC loading |
| Software sprites | Runtime verified | Two moving 8x8 sprites with page flipping; API maximum is six |
| Keyboard and joystick | Implemented | Held keys, raw analog axes, digital directions, and one button per port |
| Frame wait | Runtime verified | Direct 60 Hz VSYNC polling |
| Frame callback | Partial | Callback runs from `FRAME_WAIT`, not from a hardware interrupt |
| Sound | Partial | One blocking square-wave voice plus direct DAC writes |
| Disk files | Runtime requirement | Sequential Disk BASIC files |
| FujiNet and network | Runtime requirement | DriveWire transport through a compatible ROM |
| `REAL` math | Implemented | Color BASIC and Extended Color BASIC ROM math |
| CoCo 3 | Placeholder | Named GIME registers only; stock VDG behavior is inherited |
| `ON_ERROR` | Stub | Syntax compiles, but no CoCo error trap is installed |

## Stock VDG graphics

crustyBASIC currently exposes three of the stock VDG presentations:

| crustyBASIC mode | Hardware presentation | Size | Colors |
| --- | --- | --- | --- |
| `CELL`, `VDG_TEXT` | Alphanumeric | 32x16 cells | text |
| `BITMAP_MULTICOLOR`, `VDG_SEMIGRAPHICS_4` | Semigraphics 4 | 64x32 pixels | 8 |
| `BITMAP_HIRES`, `VDG_PMODE_4` | 1-bit graphics | 256x192 pixels | 2 |

The default graphics mode is `BITMAP_MULTICOLOR`.

The MC6847 and SAM can also produce the following eleven stock modes,
but the library does not expose them yet:

| Missing presentation | Size | Colors |
| --- | --- | --- |
| Semigraphics 6 | 64x48 | 4 |
| Semigraphics 8 | 64x64 | 8 |
| Semigraphics 12 | 64x96 | 8 |
| Semigraphics 24 | 64x192 | 8 |
| Color graphics | 64x64 | 4 |
| Resolution graphics | 128x64 | 2 |
| Color graphics | 128x64 | 4 |
| Resolution graphics | 128x96 | 2 |
| Color graphics | 128x96 | 4 |
| Resolution graphics | 128x192 | 2 |
| Color graphics | 128x192 | 4 |

This is the largest stock graphics gap. The portable `PMODE` helper
does not select those modes: mode 0 selects text and every nonzero mode
selects PMODE 4.

### Color behavior

Three-argument `PLOT` uses its color in semigraphics but not in PMODE 4.
A semigraphics cell carries one of eight colors. PMODE 4 has one bit per
pixel and one hardware colorset for the whole screen, so a plot can only
set or clear a pixel.

`GFX_COLOR` changes the semigraphics drawing color and the PMODE 4
hardware colorset. Its background argument cannot select a separate
PMODE 4 pixel color.

The bitmap tile backend defaults to `BITMAP_HIRES`, so
`TILE_COLOR_AVAILABLE` is `FALSE` after
`DISPLAY TILE_DEFAULT_DISPLAY_MODE`. It becomes `TRUE` after
`DISPLAY BITMAP_MULTICOLOR`.

### Extended Color BASIC compatibility

The `extended_color_basic` dialect accepts `PMODE`, `SCREEN`, `PCLS`,
`COLOR`, `PSET`, `PRESET`, `PPOINT`, `LINE`, `CIRCLE`, `PAINT`,
`PCOPY`, and `DRAW`, but it is not yet a complete Extended Color BASIC
graphics implementation:

- `PMODE` 1 through 4 all use the same PMODE 4 screen.
- The `PMODE` page argument is accepted but ignored.
- `SCREEN` switches only between text and PMODE 4.
- `PCLS` always clears to zero; its color argument is ignored.
- `COLOR` uses the foreground to select the hardware colorset; its
  background argument is ignored.
- Graphics `GET` and `PUT` are not implemented.
- `PCOPY` is currently a no-op.
- `CIRCLE` ignores height, start angle, and end angle arguments.
- `PAINT` ignores paint and border colors and uses a fixed 256-entry
  point stack. A large fill can stop before the area is complete.
- `DRAW` implements `U`, `D`, `L`, `R`, `E`, `F`, `G`, `H`, `M`, `B`,
  `N`, and `C`. It parses but ignores angle and scale, and skips the
  `X` substring command.
- Mixed text and graphics mode is not implemented.

## Images

Native image display supports `IMAGE_FMT_COCO_PM4` for a 6144-byte
PMODE 4 screen. The converter accepts raw PMODE 4 dumps and DECB
`.BIN` or `.MAX` files. Bit 0 of the descriptor aux byte selects the
VDG colorset.

Disk BASIC-backed runtime image loading is present. It has the same
Disk BASIC runtime requirements and file limitations described below.

## Software sprites

The CoCo has no hardware sprites. The library provides 8x8,
one-bit software sprites on the PMODE 4 bitmap. Set
`SOFT_SPRITE_ACTIVE_COUNT` to the number used by the program, up to
`SPRITE_MAX_COUNT`, which is six.

Page-flip mode draws against `$3800` and `$6000` and changes the SAM
display page at frame boundaries. A two-sprite Program Pak was visibly
stable in MAME. Mutable graphics and sprite bookkeeping now lives in
RAM in cartridge builds; it is not emitted into Program Pak ROM.

The remaining sprite gaps are:

- `SPRITE_COLOR` does not change the one-bit blit result.
- `SPRITE_DATA_TILES` ignores its width and height.
- `SPRITE_EXPAND`, `SPRITE_FLIP_X`, and `SPRITE_FLIP_Y` are no-ops.
- `SPRITE_PALETTE`, `SPRITE_PALETTE_SET`, and `SPRITE_PRIORITY` are
  no-ops.
- `SPRITE_HIT` and `SPRITE_HIT_BG` always return zero.
- Non-page-flip drawing uses XOR and does not save the background.

## Text and cells

The cell surface writes directly to the 32x16 screen at `$0400`.
`CELL_ATTRIB_SUPPORTED` is true because inverse text is supported. The color
attribute setter is empty, so per-cell foreground and background color
are not available. There is no custom character API.

## Input

| Capability | Value |
| --- | --- |
| Keyboard | Yes |
| Individual held keys | Yes |
| Joystick ports | 2 |
| Buttons | 1 per port |
| Stick type | Analog |
| Paddle axes | 4 |
| Raw analog range | 0 to 63 |
| Keypad ports | 0 |
| Mouse buttons | 0 |

`PADDLE` reads one of the four raw analog axes. `JOY` converts a pair
of axes to the portable digital direction mask. `KEY_HELD` polls the
keyboard matrix without consuming typed input.

The joystick button lines share the keyboard matrix. `JOY_BUTTON`
currently writes `$FF` to PIA port `$FF02` and does not restore the
previous keyboard column selection. That can disturb code which
interleaves its own matrix scan and should be fixed.

The cassette interface and bit-banged RS-232 interface are not exposed
through crustyBASIC APIs.

## Timing and frame callbacks

`FRAME_WAIT` polls the PIA VSYNC flag directly, increments
`FRAME_COUNTER`, and dispatches an installed callback. It assumes
60 Hz NTSC timing. There is no PAL timing selection.

`FRAME_INSTALL` only stores the callback address. The enable helper is
a no-op and the callback runs only when the program calls
`FRAME_WAIT`. `FRAME_INSTALL_SUPPORTED` therefore overstates the
current behavior if a program expects an interrupt-driven callback.

The general tick source is Color BASIC `TIMER` at `$0112`. It is a
16-bit 60 Hz counter and wraps in about 18 minutes. Cassette and sound
ROM activity can mask the IRQ and pause it. A cartridge relies on the
ROM's existing IRQ setup; the crustyBASIC runtime does not install its
own timer IRQ.

## Sound

Sound writes the top six DAC bits through `$FF20` and enables the PIA
sound gate. Available behavior is:

- one voice
- blocking square-wave note and tone calls
- note range 65 through 95
- volume 0 through 15
- `DAC_OUT` for a single raw 6-bit sample
- `SOUND_OFF` and `SOUND_ALL_OFF`

The `U16` frequency or period surfaces use only the low byte in the
delay loop. Waveform selection is ignored. Noise, multiple voices,
background playback, timed PCM streaming, and audio source selection
are not implemented.

The July 2026 MAME run advanced through the showcase `BEEP`, but audio
pitch and duration were not measured.

## Files

The file provider calls Disk BASIC ROM routines. It assumes a disk
controller and a recognized Disk BASIC ROM are present at runtime.
The implementation recognizes the Disk BASIC 1.0 and 1.1 signatures
currently in the library.

Sequential read and write are implemented. Append, update, and
directory modes report unsupported. `FILE_OPEN_NATIVE` also accepts
the Disk BASIC direct mode byte, but random-record field and record
operations are not implemented.

`AUX1` is `DISK_FILE_MODE_INPUT`, `DISK_FILE_MODE_OUTPUT`, or
`DISK_FILE_MODE_DIRECT`. `AUX2` is the default drive. File names use
the Disk BASIC `NAME/EXT:DRIVE` form; an explicit drive in the path
overrides `AUX2`.

Target-specific helpers are also callable: `DISK_OPEN`, `DISK_CLOSE`,
`DISK_READ_BYTE`, `DISK_WRITE_BYTE`, `DISK_SET_NAME`, and
`DISK_ENSURE`.

Known correctness and safety gaps:

- `FILE_SUPPORTED` and `IMAGE_FILE_SUPPORTED` are true for every CoCo
  system even when Disk BASIC hardware and ROM are absent.
- `FILE_READ_LINE` does not stop on `FILE_EOF`. A file without a
  carriage return can loop after EOF.
- `DISK_SET_NAME` does not bound the path while copying it into the
  eleven-byte Disk BASIC name and extension work area. A long path can
  overwrite adjacent ROM workspace.
- Disk file behavior was compile-tested but not run against a disk
  image during this audit.

## FujiNet and network

The transport uses the DriveWire entry vectors at `$D941` for write
and `$D93F` for read. Those addresses and the command framing match
the official `fujinet-lib` CoCo implementation.

This support requires a DriveWire-aware ROM such as HDB-DOS. A stock
Color BASIC or Extended Color BASIC ROM does not provide the required
vectors. The current availability function always returns true and
does not probe the ROM or device. On an unsupported ROM, the first
request can jump through invalid vectors or wait forever.

`FUJINET_SUPPORTED`, `NET_CLIENT_SUPPORTED`, and `NET_AVAILABLE`
therefore mean that the library code was compiled, not that working
FujiNet hardware was detected. The transport was source-audited but
not run against a FujiNet device during this audit.

## Error handling

`ON_ERROR`, `ERR`, and the shared error surface compile because the
6809 stub reserves the expected state. The CoCo ROM wrappers do not
install or dispatch a real error trap. `ON_ERROR_SUPPORTED` is
currently set to true and should be false until the trap is
implemented, or the library should gain the real ROM error-vector
hook.

The private Disk BASIC file provider has its own local error hook. That
does not make the public `ON_ERROR` surface functional.

## `REAL`

CoCo `REAL` uses the Color BASIC floating-point format.

| Works on | Details |
| --- | --- |
| `coco.1` | Basic arithmetic, integer powers, and Color BASIC ROM math |
| `coco.ecb`, `coco.3` | Adds Extended Color BASIC math such as `LOG`, `COS`, `ATN`, and fractional powers |

`POW(X, N)` with a non-negative integer constant `N` does not need
Extended Color BASIC. Fractional powers do.

## CoCo 3 audit

The `coco.3` system currently inherits the stock 32x16 text, PMODE 4,
semigraphics, memory ceiling, and runtime behavior from `coco.ecb`.
`include/targets/coco/helpers/three_gime.cbi` is an explicit
placeholder.

The manifest exposes these named GIME registers:

- `$FF90` `INIT0`
- `$FF91` `INIT1`
- `$FF92` `IRQENR`
- `$FF93` `FIRQENR`
- `$FF94` and `$FF95` timer
- `$FF98` `VMODE`
- `$FF99` `VRES`
- `$FF9A` `BORDER`
- `$FF9C` `VSCROLL`
- `$FF9D` and `$FF9E` video offset
- `$FF9F` `HOFF`
- `$FFB0` through `$FFBF` palette entries

The MMU registers at `$FFA0` through `$FFAF` are not in the manifest.
There are also no library calls for MMU mapping, 1.78 MHz mode, GIME
interrupts, timer setup, palette management, border color, scrolling,
or video offset.

The CoCo 3 hardware adds 32, 40, and 80-column text with attributes,
plus graphics widths of 160, 256, 320, 512, and 640 pixels. Depending
on width, it supports 2, 4, or 16 colors and 192, 200, or 225 lines.
None of those native text or graphics modes is currently exposed.

The missing native graphics combinations are:

- 640 pixels with 2 or 4 colors
- 512 pixels with 2 or 4 colors
- 320 pixels with 4 or 16 colors
- 256 pixels with 2, 4, or 16 colors
- 160 pixels with 16 colors

Real CoCo 3 support is the largest machine-specific improvement
opportunity after correcting capability claims that are currently
false.

## Improvement order

The audit suggests this order:

1. Correct capability claims and unsafe runtime assumptions:
   `ON_ERROR_SUPPORTED`, Disk BASIC availability, FujiNet detection,
   `FILE_READ_LINE` EOF handling, bounded Disk BASIC names,
   `JOY_BUTTON` PIA restoration, and the meaning of
   `FRAME_INSTALL_SUPPORTED`.
2. Add real CoCo 3 support: MMU registers and mapping, fast CPU mode,
   40/80-column text, native bitmap modes, palette, scrolling, timer,
   and interrupts.
3. Add the eleven missing stock VDG modes and make Extended Color BASIC
   `PMODE`, pages, `PCOPY`, graphics `GET`/`PUT`, and mixed mode real.
4. Complete software sprite collision, flip, expansion, color,
   palette, priority, and background preservation.
5. Improve sound with nonblocking playback, full period handling,
   noise, and timed sample output.
6. Add cassette and serial interfaces and runtime tests for disk,
   keyboard, joystick, sound, and FujiNet.

## Verification performed

The July 2026 audit used MAME 0.276 with a CoCo ROM and standard
Program Pak output.

A small primitive probe visibly confirmed a box, diagonal line,
horizontal line, circle, and point at their intended coordinates.
The larger [`star_rider.cbs`](../../examples/coco/crustybasic/star_rider.cbs)
showcase visibly confirmed:

- an eight-color semigraphics intro
- concentric circles and color bars
- a PMODE 4 logo, star field, moon, craters, and terrain
- two independently moving 8x8 sprites
- repeated `FRAME_WAIT` and page-flipped `SPRITES_FLUSH`

The runtime check caught a real Program Pak bug: mutable graphics,
sprite, and sound bytes had been assembled into cartridge ROM. They
are now typed private variables placed in RAM. Focused compiler tests
check that cartridge assembly aliases this state to `__cb_vars_start`
and does not emit writable `FCB` or `FDB` labels in ROM.

Focused tests also compile the Extended Color BASIC colorset path,
native CoCo image display, bitmap tile drawing, PMODE 4 spans and
colors, one- and two-sprite page flipping, and low-byte sound periods.

Not runtime-tested in this audit:

- typed text input and held-key behavior
- physical joystick and button input
- sound pitch, duration, and sample quality
- Disk BASIC files and image loading
- FujiNet or DriveWire networking
- public `ON_ERROR`
- CoCo 3 native hardware

## Hardware names

CoCo 1 and 2 do not expose chip namespaces. Use `POKE` and `PEEK` for
PIA, SAM, and VDG addresses when direct hardware access is needed.

CoCo 3 exposes `GIME.*` register names as listed above.

## Text escapes

| Escape | Byte |
| --- | --- |
| `{RETURN}`, `{ENTER}` | `$0D` |
| `{TAB}` | Four spaces |
| `{CLEAR}`, `{CLR}`, `{HOME}` | `$0C` |
| `{LEFT}`, `{BACKSPACE}` | `$08` |
| `{RIGHT}` | `$09` |
| `{UP}`, `{DOWN}` | `$03`, `$0A` |

## Cartridges

Select standard Program Pak output with:

```basic
@OPTION SYSTEM coco.ecb
@OPTION OUTPUT_TYPE cart
```

Default Program Pak output is 8K or 16K. For larger cartridges:

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
`tools/assemblers/vasm/<platform>/vasm6809_oldstyle` beside the
compiler, using the `.exe` suffix on Windows, or pass its path with
`--assembler`.

## Dialects

`color_basic` and `extended_color_basic` use CoCo suffix rules: bare
numeric names, `!`, and `#` are `REAL`, `%` is 16-bit integer, and `$`
is `STRING`.

In these dialects, positive `RND(N)` returns an integer from `1`
through `N`. Use `RAND(N)` for portable `0` through `N - 1` values.

## Sources used for the audit

- [Color Computer Technical Reference Manual, Tandy](https://colorcomputerarchive.com/repo/Documents/Manuals/Hardware/Color%20Computer%20Technical%20Reference%20Manual%20%28Tandy%29.pdf)
- [Color Computer 3 Service Manual, Tandy](https://colorcomputerarchive.com/repo/Documents/Manuals/Hardware/Color%20Computer%203%20Service%20Manual%20%28Tandy%29.pdf)
- [Color Computer 3 Extended Basic manual, Tandy](https://colorcomputerarchive.com/repo/Documents/Manuals/Hardware/Color%20Computer%203%20Extended%20Basic%20%28Tandy%29.pdf)
- [Color Computer Disk System Programming Manual, Tandy](https://colorcomputerarchive.com/repo/Documents/Manuals/Hardware/Color%20Computer%20Disk%20System%20Programming%20Manual%20%28Tandy%29.pdf)
- [FujiNet for Tandy Color Computer](https://fujinet.online/tandy-color-computer/)
- [Official FujiNet `fujinet-lib`](https://github.com/FujiNetWIFI/fujinet-lib)
- Local Color BASIC ROM disassembly:
  [`bas.asm`](../../ref/tandy/color_basic/bas.asm),
  [`extbas.asm`](../../ref/tandy/color_basic/extbas.asm), and
  [`supbas.asm`](../../ref/tandy/color_basic/supbas.asm)

## Examples

See [`../../examples/coco/`](../../examples/coco/) and the
[`star_rider.cbs`](../../examples/coco/crustybasic/star_rider.cbs)
showcase.
