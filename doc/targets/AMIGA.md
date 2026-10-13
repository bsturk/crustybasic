# crustyBASIC on the Commodore Amiga

Use `@OPTION TARGET amiga` for the default Amiga 500 profile. The exact system
names are `amiga.500`, `amiga.600` and `amiga.1200`.

Programs normally run as AmigaDOS executables from the Shell or a
Workbench icon. You can also build a disk that boots straight into your
program, using Kickstart for screens and input.

Shared BASIC calls are documented in the [API reference](../API.md). This page
covers Amiga systems and behavior specific to them.

## Basics

| Item | Value |
| --- | --- |
| CPU | Motorola 68000 on A500/A600, 68EC020 on A1200 |
| Systems | Amiga 500, Amiga 600, Amiga 1200 |
| Operating system | AmigaOS, Kickstart 1.3 and later |
| Output | HUNK executable `.exe` and a bootable `.adf` floppy image |
| Display | Shell window, 640 pixel screens in 16 colors, 320 pixel screens in 32 colors or 256 on A1200 |
| Text | Shell window size varies; program screens have 80 or 40 columns |
| Frame rate | About 60 Hz on NTSC systems, 50 Hz on PAL systems |
| Ticks | 50 Hz system clock |
| String encoding | Latin-1 |
| `REAL` | Q16.16 fixed point by default |

The default screen height is 200 pixels, or 25 text rows. Use
`@OPTION REGION REGION_PAL` on a PAL system for 256 pixels and 32 rows.

## Systems

| System | Chipset | Kickstart | Chip RAM | Notes |
| --- | --- | --- | --- | --- |
| `amiga.500` | OCS | 1.3 | 512 KB | Default. For Kickstart 1.3 and later. |
| `amiga.600` | ECS | 2.05 | 1 MB | Adds ECS registers and Kickstart 2.0 Shell features. |
| `amiga.1200` | AGA | 3.0 | 2 MB | Adds 256 color screens, 24 bit color, and AGA registers. |

Select a system with `--set system=amiga.1200` or
`@OPTION SYSTEM amiga.1200`. These constants describe the selected system:

| Constant | Meaning |
| --- | --- |
| `AMIGA_CHIPSET` | `AMIGA_OCS`, `AMIGA_ECS` or `AMIGA_AGA` |
| `AMIGA_KICKSTART` | 34, 37 or 39 |
| `AMIGA_HIRES_DEPTH` / `AMIGA_LORES_DEPTH` | Color depth of the two bitmap modes |
| `PALETTE_SIZE` | 32, or 256 on the A1200 |

## Building and loading

Use vasm's `vasmm68k_mot` assembler. Place it in
`tools/assemblers/vasm/<platform>/` or pass its path with `--assembler`.

```bash
crustybasic build hello.cbs
```

This writes `hello.exe` and an 880 KB `hello.adf` floppy image beside
`hello.cbs`. Boot the disk to run the program, or open it in Workbench
and double click the program's icon. When copying the executable to
another disk, copy its matching `.info` file too.

Choose AmigaDOS output for programs that use files, Shell commands, or
Workbench windows.

For direct disk boot, select `@OPTION OUTPUT_TYPE nodos` in the source or
pass `--set output-type=nodos`:

```bash
crustybasic build hello.cbs --set output-type=nodos -o hello.adf
```

The disk boots straight into the program using Kickstart's graphics and
input services. It starts on an 80 column text screen. `END` stops sound
and leaves the final screen visible until you reset the machine.

## Shell and Workbench

From the Shell, `PRINT` and `INPUT` use the Shell window. From Workbench,
a text program opens its own console window and waits for a key before
closing it at the end. Programs return 0 to the Shell when they finish.

## Desktop windows

`WINDOW_OPEN` opens windows on Workbench, with up to eight open at once,
and calls `APP_INIT` automatically. `WINDOW_TEXT` uses the system font.

Use `WINDOW_*` drawing calls for Workbench windows. `DISPLAY` and `PRINT`
use your program's screen or console.

The A600 and A1200 also report window movement with `APP_EVENT_MOVE`.

## Text

In `CONSOLE`, `COLOR` selects Workbench palette entries 0 to 7.

Your program's screen uses an 8x8 font with 64 characters, covering codes
32 through 95. Lowercase letters display as uppercase.
`CHARSET_INSTALL` expects a 512 byte font, and `CHARSET_DEFINE` uses
character codes 32 to 95. Charset calls open `CELL` mode when used from
the console.

## Keyboard, joystick and mouse

`RAWKEY` and `RAWKEY_CODE` use uppercase letter codes regardless of Shift.
After reading a typed character, release and press the key again to make
it available to `RAWKEY`. `KEY_HELD` keeps reporting the key while it is
held. These special keys are available:

| Constant | Key |
| --- | --- |
| `KEY_ENTER` | Return, code 13 |
| `KEY_BACKSPACE`, `KEY_DELETE`, `KEY_TAB`, `KEY_ESCAPE` | Backspace, Del, Tab, Esc |
| `KEY_UP`, `KEY_DOWN`, `KEY_RIGHT`, `KEY_LEFT` | Cursor keys |
| `KEY_F1` to `KEY_F10`, `KEY_HELP` | Function keys and Help |

`JOY(PORT)` reads the two controller ports: port 0 is the mouse port and
port 1 the usual joystick port. `JOY_DEFAULT_PORT` selects port 1.
`JOY_BUTTON(PORT, 0)` is the fire button.
`INPUT_ANY` checks held keys, joystick directions on port 1, and the
fire buttons on both ports.

The mouse has left and right buttons. `MOUSE_X`, `MOUSE_Y` and
`MOUSE_POSITION` use pixels on the program's screen, or the active screen
when using the console. `MOUSE_HIDE` and `MOUSE_SHOW` control the pointer
over the program's cell or bitmap screen.

## Graphics

AmigaDOS programs start in `CONSOLE`. `DISPLAY CONSOLE` closes the program
screen and returns text output to the console. Direct boot (`nodos`)
programs start in `CELL`.

Select a screen with `DISPLAY`:

| Mode | Size with NTSC settings | Colors |
| --- | --- | --- |
| `CELL` | 80x25 characters, 8x8 pixels each | 16, with foreground and background per cell |
| `BITMAP_HIRES` | 640x200 pixels | 16 |
| `BITMAP_LORES` | 320x200 pixels | 32 on A500/A600, 256 on A1200 |

PAL settings give each screen 256 pixel rows, or 32 character rows.
Calling `DISPLAY` without a mode selects `BITMAP_HIRES`.

Bitmap tiles default to the 320 pixel screen and use 8x8 cells.
`BACKGROUND_COLOR` changes pen 0. Palette entries use 12 bit color on
A500/A600 and 24 bit color on A1200.

Screen modes start with these colors in pens 0 to 15:

| Pen | Color | Pen | Color |
| --- | --- | --- | --- |
| 0 | `BLACK` | 8 | `ORANGE` |
| 1 | `WHITE` | 9 | `BROWN` |
| 2 | `RED` | 10 | `LIGHT_RED` |
| 3 | `CYAN` | 11 | `DARK_GRAY` |
| 4 | `PURPLE` | 12 | `GRAY` |
| 5 | `GREEN` | 13 | `LIGHT_GREEN` |
| 6 | `BLUE` | 14 | `LIGHT_BLUE` |
| 7 | `YELLOW` | 15 | `LIGHT_GRAY` |

Pens 16 to 31 repeat these colors and are shared with sprites.
`PALETTE_RESET` restores pens 0 to 31.

| Call | Result |
| --- | --- |
| `AMIGA_RGB12(COLOR)` | A default color in Amiga's 12 bit `$0RGB` format |
| `AMIGA_GFX_PLANE(N)` | Address of bitplane `N` of the open screen |
| `AMIGA_GFX_ROW_BYTES` | Bytes per bitplane row |
| `AMIGA_SCREEN_ADDR` / `AMIGA_WINDOW_ADDR` | Addresses of the program's screen and window |

`CELL_PUTC COL, ROW, CODE, GLYPH_ADDR` draws an 8x8 glyph using the
cell's colors and stores its character code for `CELL_GETC`.
`CELL_COLOR COL, ROW, COLOR, GLYPH_ADDR` changes the foreground color
and redraws that glyph. Pass zero as the glyph address to use the text font.
Custom glyphs contain eight bytes, one per row, with bit 7 on the left.

## Sprites

Seven hardware sprites are available as IDs 0 to 6. Use
`@OPTION SPRITE_COUNT 8` for IDs 0 to 7; this hides the mouse pointer
while sprites are active. A normal sprite is 16x16 pixels, with three
colors and a transparent background. Positions use 320 pixel coordinates
in both bitmap modes.

`SPRITE_DATA` takes 64 bytes: two 16 bit color masks for each of the
16 rows. `SPRITE_DATA_8X8` draws a single color 8x8 shape in the left
half. `SPRITE_DATA_TILES` builds single color sprites up to two tiles
wide and four tiles tall.

`SPRITES_ON` opens a 320 pixel screen when `DISPLAY_MODE` is `CONSOLE`.
`SPRITE_SHAPE_INSTALL` provides 32 monochrome animation slots.
`AMIGA_SPRITE_SHAPE_INSTALL SLOT, SOURCE, HEIGHT` accepts word pairs
for shapes up to 32 pixels tall.

Sprites share colors in pairs:

| Pair | With seven sprites | With eight sprites | Palette pens |
| --- | --- | --- | --- |
| 0 | Mouse pointer and ID 0 | IDs 0 and 1 | 17 to 19 |
| 1 | IDs 1 and 2 | IDs 2 and 3 | 21 to 23 |
| 2 | IDs 3 and 4 | IDs 4 and 5 | 25 to 27 |
| 3 | IDs 5 and 6 | IDs 6 and 7 | 29 to 31 |

Single color shapes use separate colors within each pair. `SPRITE_COLOR`
sets that shape's color, or the first color for a `SPRITE_DATA` shape.
`SPRITE_PALETTE_SET` uses pair numbers 0 to 3.

`SPRITE_HIT` checks for overlapping sprite boxes. `SPRITE_HIT_BG`
reports a pair's collision with odd numbered background pens. Call
`SPRITES_FLUSH` before reading background collisions.

## Scrolling

The portable `SCROLL_*` calls move the `CELL` screen horizontally or
vertically in eight pixel steps, keeping cell colors and attributes.

`AMIGA_SCROLL_OPEN(COLUMNS, TOP, ROWS, BUFFER_COLUMNS)` opens a scrolling
band between fixed rows on a 16 color, 320 pixel screen. It returns a
bitmap handle, or zero if it fails. The arguments use character cells.
A width of 38 cells leaves room for all eight sprites. Set the horizontal
position in pixels with `AMIGA_SCROLL_POSITION X`, then call
`AMIGA_SCROLL_PRESENT` to show it at the next frame.

`AMIGA_SCROLL_GLYPH COL, ROW, GLYPH_ADDR, FG, BG` draws an 8x8 glyph in
the scrolling buffer. Columns and rows start at zero in the full buffer.
`AMIGA_SCROLL_CLEAR` clears that buffer to color zero.

## Timing and sound

`TICKS` and the elapsed timer use 50 ticks per second on both PAL and
NTSC systems.
AmigaDOS rounds `DELAY` up to 20 ms steps.

Paula plays four voices with volumes 0 to 64. The note and tone calls
use square waves. `TONE` and `PAULA.SOUND VOICE, PERIOD, VOLUME` use
Paula periods: smaller values give higher notes. Set `REGION` for your
machine to keep notes in tune.

`PAULA.NOTE_PERIOD(NOTE)` returns the period for a two sample waveform
cycle. Divide it by four for an eight sample cycle.

`BEEP` lets the program continue while the sound plays.
`TONE PERIOD, DURATION, TRUE` plays in the background and stops
automatically, including while the program is in `DELAY`. `BEEP` and
`TONE` durations use milliseconds.

Audio uses 8 bit mono WAV. `AUDIO_PLAY` plays it once on voice 3 in the
background.

`AMIGA_SAMPLE_BUFFER(BYTES)` gives you a cleared buffer for signed 8 bit
samples and returns its address, or zero if it fails. It stops the sound
voices and replaces the previous buffer. The buffer is released when
the program ends.

`AMIGA_SAMPLE_PLAY VOICE, ADDRESS, BYTES, PERIOD, VOLUME` loops a sample
from that buffer on voice 0 to 3. The address must be even and the byte
count an even number from 2 to 65534. Period is the raw Paula period and
volume is 0 to 64.

## Files and Shell commands

With AmigaDOS output, the file API uses paths such as `DF0:`, `RAM:`, `T:`
and `SYS:`. Text files use line feed line endings. An empty path with
`FILE_MODE_DIR` names the current directory. `FILE_OPEN_NATIVE` uses 4
for read/write, 5 for an existing file, or 6 to create a file as its
first extra argument.

`FILE_COMMAND 1` deletes a file and `FILE_COMMAND 3` creates a directory.
File channels run from 0 to 15, and all 16 can be open at once.

`CMD` runs Shell commands in the console window. On `amiga.500`, `CMD`
and `CMD_CLOSE` return 0 when the command ran, or -1 if it failed to
start. The other profiles return the command's own return code.

## Images

`@INCLUDE_IMAGE` converts pictures to 320x200 using the 16 default
colors. Smaller pictures are centered.

With AmigaDOS output, `IMAGE_LOAD(PATH)` also accepts indexed color IFF
ILBM pictures and applies their palette. ILBM files can be plain or
ByteRun1 compressed.

## Hardware registers

Use `CUSTOM.*` for the display, blitter, copper, sprite and audio
registers, and `CIAA.*` and `CIAB.*` for the two CIA chips. The A600 and
A1200 add `ECS.*`; the A1200 also adds `AGA.*`.

## References

- [Amiga Hardware Reference Manual](http://amigadev.elowar.com/read/ADCD_2.1/Hardware_Manual_guide/node0000.html)
- [ROM Kernel Reference Manual: Libraries](http://amigadev.elowar.com/read/ADCD_2.1/Libraries_Manual_guide/node0000.html)
- [AmigaDOS Manual](http://amigadev.elowar.com/read/ADCD_2.1/AmigaDOS_Manual_guide/node0000.html)
