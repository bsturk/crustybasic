# crustyBASIC on Tandy Color Computer

Use `@OPTION TARGET coco` for the default Extended Color BASIC
profile, or select an exact machine with `@OPTION SYSTEM`.

Portable runtime calls are documented in [`../API.md`](../API.md).
This page covers CoCo systems, current library support, and known limits.
For command-line target selection and dialects, see
[`../USAGE.md`](../USAGE.md).

## Systems

| System | Default output | ROM expected |
| --- | --- | --- |
| `coco.1` | `.bin` | Color BASIC |
| `coco.ecb` | `.bin` | Extended Color BASIC |
| `coco.3` | `.bin` | Super Extended Color BASIC |

For CoCo 2, select `coco.1` if the machine has Color BASIC, or
`coco.ecb` if it has Extended Color BASIC or Disk Extended Color BASIC.

`coco.ecb` is the default. Program Pak `.ccc` output is selected with
`OUTPUT_TYPE cart` on the same systems.

## Optional upper RAM

A 64K CoCo has RAM behind the BASIC and cartridge ROM area. It is not
normally visible to reads. Enable it explicitly with:

```basic
@OPTION MEMORY_REGION upper_ram
```

This selects SAM all RAM mode, masks IRQ and FIRQ, and gives mutable
program data `$7800` through `$FEFF` on CoCo 1/2, or `$8000` through
`$FEFF` on CoCo 3 to keep data outside the bank window. Code stays below
the `$6000` PMODE 4 screen, so graphics cannot overwrite it. `END` loops instead
of returning to BASIC.

The option removes all Color BASIC, Extended Color BASIC, Disk BASIC,
and cartridge ROM services. Do not use `KEY`, `INPUT`, file or network
calls, ROM `REAL` math, `TIMER`, or `TICKS`. Direct hardware APIs such
as `KEY_HELD`, `FRAME_WAIT`, joystick input, graphics, and sound remain
available. The compiler reports the region as disabling `rom_services`,
`basic_return`, and `program_rom`

## VDG graphics

crustyBASIC exposes every stock VDG presentation by name:

| crustyBASIC mode | Hardware presentation | Size | Colors |
| --- | --- | --- | --- |
| `CELL`, `VDG_TEXT` | Alphanumeric | 32x16 cells | text |
| `BITMAP_LORES`, `VDG_SG4` | Semigraphics 4 | 64x32 | 9 including black |
| `VDG_SG6` | Semigraphics 6 | 64x48 | 5 including black |
| `VDG_SG8` | Semigraphics 8 | 64x64 | 9 including black |
| `VDG_SG12` | Semigraphics 12 | 64x96 | 9 including black |
| `VDG_SG24` | Semigraphics 24 | 64x192 | 9 including black |
| `VDG_CG1` | Color graphics 1 | 64x64 | 4 |
| `VDG_RG1` | Resolution graphics 1 | 128x64 | 2 |
| `VDG_CG2` | Color graphics 2 | 128x64 | 4 |
| `VDG_RG2` | Resolution graphics 2 | 128x96 | 2 |
| `VDG_CG3` | Color graphics 3 | 128x96 | 4 |
| `VDG_RG3` | Resolution graphics 3 | 128x192 | 2 |
| `BITMAP_MULTICOLOR`, `VDG_CG6` | Color graphics 6 | 128x192 | 4 |
| `BITMAP_HIRES`, `VDG_RG6` | Resolution graphics 6 | 256x192 | 2 |

The default graphics mode is `BITMAP_MULTICOLOR`.

The CoCo 3 supports the VDG text, SG4, CG, and RG modes. Its GIME does
not implement SG6, SG8, SG12, or SG24, so those names are available only
on CoCo 1 and 2 systems.

### Color behavior

On CoCo 1 and 2, `BLACK` is 0 and `GREEN` is 8. Semigraphics drawing,
clearing, and `POINT` use these distinct color values.

The three argument `PLOT` uses its color in semigraphics, VDG color graphics,
and multicolor GIME modes. VDG resolution graphics and one bit GIME
modes can only set or clear a pixel.

`COLOR` changes the semigraphics drawing color and the PMODE 4
hardware colorset. Its background argument cannot select a separate
PMODE 4 pixel color.

The bitmap tile backend follows the active mode's plot color support.
`TILE_PRINT` draws 8x8 glyphs in VDG and GIME bitmap modes, using the
current drawing color in color bitmap modes.

### Extended Color BASIC compatibility

The `extended_color_basic` dialect provides `PMODE`, `SCREEN`, `PCLS`,
`COLOR`, `PSET`, `PRESET`, `PPOINT`, `LINE`, `CIRCLE`, `PAINT`,
`PCOPY`, `DRAW`, graphics `GET`/`PUT`, `PLAY`, and `TIMER`.

Graphics reserve eight 1536-byte pages at `$4800-$77FF`. `PMODE` selects
where to draw; `SCREEN` selects what to display. Coordinates use the
native 256x192 grid in every PMODE. Colors use the native 0..8 numbering.
`COLOR` sets foreground and background independently.

`CIRCLE` accepts height/width ratio and start/end arc positions. `PAINT`
accepts paint and border colors. `DRAW` supports angle, scale, signed
relative moves, and `X` string references. `PLAY` accepts native music
strings. `TIMER` reads or sets the 60 Hz clock.

Both Color BASIC dialects accept `SOUND tone, duration` and `AUDIO ON` or
`AUDIO OFF`. Pitch follows the ROM tone scale, with tone 89 near middle C,
and one duration unit lasts about a sixteenth of a second. Both values are
approximate until checked against a real machine.

## Scrolling

`SCROLL_SETUP(SCROLL_TYPE_SW)` selects software scrolling in every
available `DISPLAY` mode. `SCROLL_MOVE DX, DY` accepts native pixel
steps in either direction; both granularity values are `1`. Move one
axis per call and cross at most one cell boundary before filling the
returned edge and calling `SCROLL_PRESENT`.

Bitmap cells are 8x8 pixels. VDG text uses 8x12 cells and GIME text
uses 8x8 cells. Each enabled axis keeps one cell hidden for incoming
content. Use `SCROLL_BUFFER_COLUMNS`, `SCROLL_BUFFER_ROWS`,
`SCROLL_VIEW_COLUMNS`, and `SCROLL_VIEW_ROWS` for the active sizes.
`SCROLL_CELL_PUTC` and `SCROLL_CELL_COLUMN` write the backing buffer.
The foreground argument follows the display's color choices.

Text scrolls through a bitmap font. CoCo 3 color text keeps foreground
and background colors; 64/80-column color text uses a 32/40-column
bitmap while retaining the full text width in the backing buffer.
`SCROLL_CLOSE` restores the native text mode with the updated characters
and colors. Bitmap modes retain the updated picture at a cell boundary.

On CoCo 3, `SCROLL_SETUP(SCROLL_TYPE_HW)` before `SCROLL_BEGIN` scrolls
GIME text displays with the video registers instead of redrawing them;
other displays keep software scrolling. Vertical moves use one pixel
steps. Horizontal moves are whole characters, so `SCROLL_X_GRANULARITY`
is `8`; monochrome GIME text scrolls vertically only and reports `0`.
The full text screen stays visible, all 64 or 80 columns included, with
the hidden column to its right and the hidden row below. Part of the
hidden row shows while the view sits between rows. `SCROLL_CLOSE`
restores the native text mode with the top 24 rows. Hardware text
scrolling keeps its cells in RAM blocks `$31` to `$33`.

Semigraphics moves shapes by one native pixel. Its shared hardware
colors follow the source cell at the top left of each destination
color group, so fine movement can change colors at shape boundaries.

CoCo 1/2 scrolling uses a 6K backing buffer. CoCo 3 uses spare RAM banks
and a backing area of 3232 bytes in program RAM, including on 128K
machines. Keep the scrolling display's memory in use until
`SCROLL_CLOSE`. Software redraw speed depends on the display size
and color depth.

## Images

Native image display supports `IMAGE_FMT_COCO_PM4` for a 6144-byte
PMODE 4 screen. The converter accepts raw PMODE 4 dumps and DECB
`.BIN` or `.MAX` files. Bit 0 of the descriptor aux byte selects the
VDG colorset.

CoCo 3 also supports `IMAGE_FMT_COCO_GIME`, a 320x192, 16 color
bitmap with its palette. Select `coco-gime` when converting a color image.
`IMAGE_LOAD` selects the matching display mode for embedded images
and converted `.img` files.

Converted descriptors use RLE8 when it makes the stored image smaller.
Images that do not shrink stay uncompressed.

`IMAGE_WIPE` and `IMAGE_SLIDE` move the bitmap in all four directions.
`IMAGE_FADE_IN` and `IMAGE_FADE_OUT` use four dither steps. Dissolves
use 1, 2, 4, or 8 pixel blocks. PMODE 4 effects accept unpacked and RLE8
images. A compressed effect temporarily uses 6144 bytes of heap space.

## Software sprites

CoCo sprites use the 256x192 PMODE 4 bitmap or the selected GIME bitmap,
with eight software sprite slots by default. Flipping, expansion,
priority, and pixel collisions are supported. Keep the transformed
shape within the screen.

In PMODE 4, `SPRITE_COLOR` draws black for zero and the active display
foreground for any nonzero value. Use `WHITE` for the foreground;
`COLOR` selects the display colorset. GIME sprites use the selected
bitmap palette and color depth.

Double buffering is enabled by default. PMODE 4 uses `$4600` and `$6000`
and swaps pages during vertical blank. GIME keeps its hidden buffer in
banks `$25`, `$26`, `$27`, and `$2B`, with 3232 additional bytes at
`$4600`. While GIME sprites use double buffering, plotting, bitmap text,
and other bitmap drawing use the hidden buffer. `GFX_SWAP` draws the
sprites, waits for vertical blank, and copies the completed picture to
the screen. The drawing buffer keeps its contents.
The following `FRAME_WAIT` counts that wait and runs frame services.
With `SPRITE_DOUBLE_BUFFER FALSE`, sprite updates draw on the visible
page. `GFX_SWAP` applies position, shape, and visibility changes.

Call `SPRITES_RESTORE` before drawing into the background, then
`GFX_SWAP` to redraw the sprites. Restoration preserves sprite
positions and visibility.

## Text and cells

The cell surface writes directly to the 32x16 screen at `$0400`.
`CELL_ATTRIB_SUPPORTED` is true because inverse text is supported. On CoCo 1
and 2, per cell foreground and background color are not available. CoCo 3
GIME color text provides separate foreground and background colors plus
inverse and blinking attributes for each cell. There is no custom character
API on the Coco.

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
keyboard matrix without consuming typed input. It knows letters in
either case, digits, `@ , - . / : ;`, space, the arrow keys, `ENTER`,
`CLEAR`, and `BREAK` as code 27.

The joystick button lines share the keyboard matrix. `JOY_BUTTON`
restores the previous keyboard column selection at `$FF02` after
reading the buttons.

## Timing

`FRAME_WAIT` uses the PIA VSYNC flag. CoCo timing assumes 60 Hz NTSC;
there is no PAL timing selection.

`TICKS` returns a 32-bit elapsed count at 60 Hz. `TICKS_RESET` starts a
new interval. Extended Color BASIC systems extend the ROM timer through
its interrupt; cassette and sound ROM activity can pause that clock.
On `coco.1`, ticks count `FRAME_WAIT` frames.

## Sound

Sound goes through the six bit DAC at `$FF20`. Every sound call selects
the DAC as the sound source and opens the PIA sound gate, so sound works
after a cassette `AUDIO ON`. Available behavior is:

- one voice
- blocking note and tone calls
- note range 36 through 95
- volume 0 through 15
- shapes: `SOUND_SHAPE_TONE`, `SOUND_SHAPE_PULSE`, `SOUND_SHAPE_TRIANGLE`,
  `SOUND_SHAPE_SAW`, and `SOUND_SHAPE_NOISE`
- `TONE PERIOD, DURATION` plays a square wave for `DURATION` ticks.
  `PERIOD` is about 55930 divided by the frequency in Hz
- `TONE PERIOD, DURATION, TRUE` schedules the tone using timed sound
  so gameplay can continue. `FALSE` plays the tone before returning.
- `DAC_OUT SAMPLE` sets the sound output level to a value from 0 to 63
- `SOUND_SOURCE SOURCE` picks what the speaker plays: `SOUND_SOURCE_DAC`,
  `SOUND_SOURCE_CASSETTE`, `SOUND_SOURCE_CARTRIDGE`, or `SOUND_SOURCE_OFF`
- `SOUND_OFF` and `SOUND_ALL_OFF`

`PLAY_NOTE` plays a short blip of about 18 ms and returns. `PLAY_NOTE_FOR`
holds the note for `DURATION` ticks. Both block until done, and
`SOUND_NOTE_BLOCKING` is `TRUE`.
Pulse, triangle, and saw step through an eight level table, so they lose
some pitch accuracy in the top octave. Noise picks a new random level
every half period, so low notes rumble and high notes hiss.

For example, `DAC_OUT 32` sets the output near the middle of its range.
The level stays until changed or silenced with `SOUND_ALL_OFF`.
A single call does not play a sustained tone.

`SOUND_SOURCE SOUND_SOURCE_CASSETTE` does what Color BASIC `AUDIO ON` does,
and `SOUND_SOURCE_CARTRIDGE` plays a sound cartridge such as the
Orchestra-90. Every note, tone, and audio call switches back to the DAC,
and a joystick read leaves the speaker muted, so select the source again
afterwards. `SOUND_SOURCE_OFF` closes the sound gate like `AUDIO OFF`.

### Timed sound

Timed sound has one voice and counts durations in frames.

On the CoCo 1 and 2 it is cooperative. `FRAME_WAIT` plays the current
note while waiting for the next video frame. The tone carries a frame
rate buzz, and the game loop must call `FRAME_WAIT` every frame.
`SOUND_TIMED_WAIT` calls `FRAME_WAIT` while it waits.

On the CoCo 3 timed sound runs from the GIME timer and vertical border
interrupts on FIRQ, so notes play by themselves while the program runs.
Tone and noise are supported. Pulse, triangle, and saw play as tone. The
handler is installed at the Color BASIC FIRQ vector `$010F` and removed
again when the program ends. The program should start from the 32 column
text screen, which is the GIME state the library assumes.

### Audio

`AUDIO_PLAYBACK_MODEL` is `AUDIO_PLAYBACK_BLOCKING`. `AUDIO_PLAY` streams
an unsigned 8 bit mono PCM asset through the DAC and returns when the last
sample has played. Interrupts and the frame clock pause during playback,
so keep clips short.
Memory is the other limit: one second at 8000 Hz takes 8000 bytes.

Sample rates up to about 23 kHz are accepted, and 8000 Hz is a good fit.
`AUDIO_FILE_SUPPORTED` is `FALSE`, so load embedded assets:

```basic
@INCLUDE_AUDIO SHOT "assets/shot.wav"

OK = AUDIO_LOAD(ADDR SHOT_AUDIO)
OK = AUDIO_PLAY
```

`AUDIO_SOUND_SHARED` is `FALSE`. Silence timed sound before `AUDIO_PLAY`
on the CoCo 3, since both drive the same DAC.

## Files

### Cassette

Use `FILE_CASSETTE_CHANNEL` (255) for one sequential cassette file:

```basic
FILE_OPEN FILE_CASSETTE_CHANNEL, "SCORES", FILE_MODE_WRITE
FILE_WRITE_LINE FILE_CASSETTE_CHANNEL, "PLAYER 1"
FILE_CLOSE FILE_CASSETTE_CHANNEL
```

Open with `FILE_MODE_READ` to read it back. Byte, string, and line calls
work with tape. Names use the first eight characters; an empty read name
accepts the next file. `FILE_STATUS` returns `FILE_OK`, `FILE_EOF`, or a
Color BASIC error code such as 40 for a tape I/O error.

Tape calls use Color BASIC ROM and need no disk controller. Transfers
pause the program while the tape runs. Set the recorder to record or
play before opening the file, and rewind before reading a saved file.
Append, update, directory, and file commands return `FILE_UNSUPPORTED`.
`FILE_OPEN_NATIVE` accepts the input/output mode bytes described below;
its drive argument is ignored for tape.

### Disk

Build a Disk BASIC `.bin` and a 35 track `.dsk` together with:

```text
crustybasic build hello.cbs --set disk-image=true -o hello.bin
```

The same `disk-image = true` setting can go in the program's config file.
Mount the image in drive 0, then enter `LOADM "HELLO":EXEC`.
Program names are uppercase, shortened to eight characters, with a `BIN` extension.

The bundled `cb-coco-dsk-wrap` tool creates the image without external disk tools:

```text
cb-coco-dsk-wrap decb disk.bin disk.dsk
```

Files in `<source name>.disk_files/` are included as sequential data files.
Names use eight characters plus a three character extension; unsupported
characters become underscores. Duplicate names after shortening report an error.
File bytes are copied unchanged; use carriage returns for text line endings.

Disk file calls use Disk BASIC ROM routines. They assume a disk
controller and a Disk BASIC 1.0 or 1.1 ROM are present at runtime.

Disk file channels run from 1 to 15, and all 15 can be open at once.
Read, write, append, update, and directory modes are available.
Append starts at the end and creates the file if needed. Update starts at
byte zero, preserves unread bytes, and extends the file when writing past
its end. Each extra channel reserves one buffer while open.
`FILE_OPEN_NATIVE` also accepts the Disk BASIC direct mode byte.

`AUX1` is `DISK_FILE_MODE_INPUT`, `DISK_FILE_MODE_OUTPUT`, or
`DISK_FILE_MODE_DIRECT`. `AUX2` is the default drive. File names use
the Disk BASIC `NAME/EXT:DRIVE` form; an explicit drive in the path
overrides `AUX2`. `NAME.EXT` works too. Names are uppercased and cut to
8 characters with a 3 character extension.

`FILE_COMMAND DISK_COMMAND_DELETE, CHANNEL, 0, DRIVE, PATH$` deletes a file.
Use `DISK_COMMAND_RENAME` with `"OLD/EXT,NEW/EXT"` to rename a file on the
same drive. `FILE_STATUS(CHANNEL)` returns the result. Open files cannot
be deleted or renamed.

`ON_ERROR` catches disk, directory, and tape errors, and `ERR()`
returns the error status. Reaching the end of a file is not an error.

`FILE_MODE_DIR` lists the whole disk. Use `""` for drive 0 or `":1"`
for drive 1. Read with `FILE_READ_BYTE` or `FILE_READ_LINE`; each line
contains a `NAME/EXT` filename followed by a carriage return. Names
without an extension omit the slash. Deleted entries are skipped and
the listing ends with `FILE_EOF`. Directory channels keep separate
read positions and can be used alongside ordinary file channels.

Target-specific helpers are also callable: `DISK_OPEN`, `DISK_CLOSE`,
`DISK_READ_BYTE`, `DISK_WRITE_BYTE`, `DISK_SET_NAME`, and
`DISK_ENSURE`.

Known correctness and safety gaps:

- `IMAGE_FILE_SUPPORTED` is true for every CoCo
  system even when Disk BASIC hardware and ROM are absent.

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
FujiNet hardware was detected.

## `REAL`

CoCo `REAL` uses the Color BASIC floating-point format.

| Works on | Details |
| --- | --- |
| `coco.1` | Basic arithmetic, integer powers, and Color BASIC ROM math |
| `coco.ecb`, `coco.3` | Adds Extended Color BASIC math such as `LOG`, `COS`, `ATN`, and fractional powers |

`POW(X, N)` with a non-negative integer constant `N` does not need
Extended Color BASIC. Fractional powers do.

## CoCo 3

`RAM_PAGE bank` selects a physical 8K RAM bank at `$6000-$7FFF` through
GIME's task 0 MMU entry. `RAM_PAGE()` returns that bank number.
Use task 0 with the MMU enabled. Selectors are `0` through `63`; a 128K
machine has 16 distinct banks, and a 512K machine has 64. Restore the
mapping before screen operations, which also use this window.

The `coco.3` system provides the GIME graphics, text, palette, video
offset, and MMU register names. The manifest exposes:

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
- `$FFA0` through `$FFAF` MMU task entries
- `$FFB0` through `$FFBF` palette entries

CoCo 3 color constants name 16 distinct palette slots. `BLACK` is 0,
`WHITE` is 1, and `GREEN` is 5. Palette colors use RGB monitor encoding.
`BORDER_COLOR` writes the named color's RGB value, or accepts
`RGB(r, g, b)` with channels reduced to two bits each.

`PALETTE_SET index, color` changes a slot using a named color or
`RGB(r, g, b)`. RGB channels are reduced to two bits each. GIME bitmap
modes use slots 0 through 15, with 2 or 4 color modes displaying only
the first 2 or 4 slots. GIME text exposes all 16 palette slots, allowing
foreground and background colors to be changed separately.

Selecting a display mode installs its starting palette. `PALETTE_RESET`
restores that palette. VDG modes use their compatible colors, and
returning to BASIC restores the VDG palette.

Native bitmap names use `GIME_<width>X<height>_<colors>`. The guaranteed
width/color pairs are 128, 160, 256, and 320 pixels with 2, 4, or 16
colors, plus 512 and 640 pixels with 2 or 4 colors. Every pair has 192,
200, and 225-line variants.

Native text names are `GIME_TEXT_32`, `GIME_TEXT_40`, `GIME_TEXT_64`,
and `GIME_TEXT_80`. The matching `_COLOR` names enable one attribute
byte per cell with 8 foreground and 8 background choices. Portable
`CELL` selects `GIME_TEXT_40_COLOR`. Each cell displays two colors,
so `CELL_COLORS_PER_CELL` is `2`.

Portable `BITMAP_LORES` keeps the compatible SG4 display. On CoCo 3,
`BITMAP_HIRES` selects `GIME_640X192_2` and `BITMAP_MULTICOLOR` selects
`GIME_320X192_16`.

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

## References

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
