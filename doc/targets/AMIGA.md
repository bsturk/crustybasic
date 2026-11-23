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
| Integers | `U8` through `U64`, `I8` through `I64` |

The default screen height is 200 pixels, or 25 text rows. Use
`@OPTION REGION REGION_PAL` on a PAL system for 256 pixels and 32 rows.

## Heap

The heap arena defaults to 65,534 bytes. Set its size with
`@OPTION HEAP_BYTES 131072` or `--set heap-bytes=131072`.
`HEAP_AVAIL` returns the total free bytes across the arena.

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
fire buttons on both ports. In a `mask`, `INPUT_JOY` checks port 1 and
`INPUT_MOUSE` checks the left and right mouse buttons.

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
| `CELL_MULTICOLOR` | 80x25 characters, 8x8 pixels each | 16, with four colours per cell |
| `BITMAP_HIRES` | 640x200 pixels | 16 |
| `BITMAP_LORES` | 320x200 pixels | 32 on A500/A600, 256 on A1200 |

PAL settings give each screen 256 pixel rows, or 32 character rows.
Calling `DISPLAY` without a mode selects `BITMAP_HIRES`.

Native modes select a resolution and color depth directly:

| Mode family | Width | A500 | A600 | A1200 |
| --- | --- | --- | --- | --- |
| `AMIGA_LORES_2` through `AMIGA_LORES_256` | 320 | 2 to 32 colors | 2 to 32 colors | 2 to 256 colors |
| `AMIGA_HIRES_2` through `AMIGA_HIRES_256` | 640 | 2 to 16 colors | 2 to 16 colors | 2 to 256 colors |
| `AMIGA_SUPERHIRES_2` through `AMIGA_SUPERHIRES_256` | 1280 | - | 2 or 4 colors | 2 to 256 colors |
| `AMIGA_LORES_EHB` | 320 | 64 colors | 64 colors | 64 colors |
| `AMIGA_LORES_HAM6` | 320 | HAM6 | HAM6 | HAM6 |
| `AMIGA_LORES_HAM8` | 320 | - | - | HAM8 |
| `AMIGA_HIRES_EHB`, `AMIGA_HIRES_HAM6`, `AMIGA_HIRES_HAM8` | 640 | - | - | EHB, HAM6 or HAM8 |
| `AMIGA_SUPERHIRES_EHB`, `AMIGA_SUPERHIRES_HAM6`, `AMIGA_SUPERHIRES_HAM8` | 1280 | - | - | EHB, HAM6 or HAM8 |

Color counts are powers of two. Append `_LACED` to any native mode name
for interlacing: 400 rows with NTSC settings or 512 with PAL settings.
Use `DISPLAY_HAS_MODE` to check the selected Amiga model. `DISPLAY_MODE`
retains the native mode name's value after selection.

EHB uses 32 palette entries plus 32 colors at half their brightness.
HAM pixels select a palette entry or change one color component of the
preceding pixel. `PLOT`, `POINT`, drawing colors and color count queries
use the raw pixel codes: 64 codes for HAM6 and 256 for HAM8. HAM6 has
16 base palette entries; HAM8 has 64.

Bitmap tiles default to the 320 pixel screen. PNG sheets use `TILE_SIZE`
with dimensions such as `8 8`, `16 16`, or `32 32`. The default `mono` format
packs each logical tile into 8x8 single color pieces. Use the imported span
constants for `TILE_BEGIN`, then call the tileset's `INSTALL` routine.
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

Character imports use the writable 8x8 glyphs at codes 32 through 95.
`CHARSET_CUSTOM_FIRST`, `CHARSET_CUSTOM_LAST`, and `CHARSET_CUSTOM_COUNT`
identify that range.

`DISPLAY CELL_MULTICOLOR` uses the same cell layout and 16 colour screen
as `DISPLAY CELL`, with four colours per character. `CELL_COLORS FG, BG,
COLOR2, COLOR3` selects those colours. `CHARSET_DEFINE` reads 16 bytes:
eight rows of low bits followed by eight rows of high bits, with bit 7
on the left. Pixel values 0, 1, 2 and 3 select BG, COLOR2, COLOR3 and FG.
The normal font uses BG and FG.

`CHARSET_BYTES_PER_ENTRY` is 16 in multicolour cell mode and 8 in ordinary
cell mode. Bitmap modes and scrolling retain the selected character format.
`CELL_PUTC` and `SCROLL_CELL_PUTC` use the same character definitions.

## Sprites

PNG sprite sheets specify `FRAME_SIZE width height`. Imported logical frames
share up to three visible colors plus transparency and are split into 16x16
pieces. A 32x32 frame uses four sprite IDs. The generated frame and slot
constants describe its geometry;
`APPLY_FRAME`, `MOVE`, `SHOW`, and `HIDE` keep the pieces together.
`APPLY_FRAME` loads the imported colors into each hardware pair's shared
palette. `FORMAT mono` selects white single color artwork.

The mouse pointer uses one of the eight hardware sprites, leaving seven as
IDs 0 to 6. `@OPTION MOUSE_POINTER FALSE` hides the pointer while
sprites are active and gives its sprite to ID 0, so `@OPTION SPRITE_COUNT 8`
provides hardware IDs 0 to 7. A normal sprite is 16x16 pixels, with three
colors and a transparent background. Positions use 320 pixel coordinates
in both bitmap modes.

For hardware IDs, `SPRITE_DATA` takes 64 bytes: two 16 bit color masks for each of the
16 rows. `SPRITE_DATA ID,16,HEIGHT,ADDRESS` accepts 1 to 255 rows in
the same format. `SPRITE_DATA_8X8` draws a single color 8x8 shape in the left
half. `SPRITE_DATA_TILES` builds single color sprites up to two tiles
wide and four tiles tall.

Sprites need a screen selected with `DISPLAY` before `SPRITES_ON`; they stay
off while `DISPLAY_MODE` is `CONSOLE`.
`SPRITE_SHAPE_INSTALL` provides 32 monochrome animation slots.
`AMIGA_SPRITE_SHAPE_INSTALL SLOT, SOURCE, HEIGHT` accepts word pairs
for shapes up to 32 pixels tall.

`AMIGA_SPRITE_LIST CHANNEL,ADDRESS` displays a Chip RAM sprite DMA list
on physical channel 0 to 7. Each image has two control words, two pixel
words per row, and the next image or two zero words. Control words use
absolute hardware coordinates; bit 7 of the second word attaches an odd
channel to its even partner for 15 visible colors.
`AMIGA_SPRITE_LIST_RELEASE CHANNEL` returns the channel to the system.

Hardware sprites share colors in pairs:

| Pair | With the mouse pointer | Without the mouse pointer | Palette pens |
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

### Software sprites

Hardware and software sprites share the
[portable SPRITE API](../API.md#hardware-and-software-sprites).
`@OPTION SPRITE_COUNT 10` allocates seven hardware sprites and software
IDs 7 to 9, or eight hardware sprites and software IDs 8 and 9 without the
mouse pointer. `SPRITE_HW_COUNT` is 7, or 8 without the mouse pointer;
`SPRITE_MAX_COUNT` reports the total.
`SPRITE_KIND` is `SPRITE_KIND_MIXED` when software slots are allocated.

Software sprites use the blitter to draw into the main screen bitmap
in `CELL`, `BITMAP_HIRES`, and `BITMAP_LORES`. Positions start at the
display's top left and move by pixels, including all 640 horizontal
pixels in hires. Use positive
dimensions and keep the whole shape inside the display. In mixed displays,
keep software sprites inside the bitmap area.

`SPRITE_DATA id, width, height, address` takes pixel dimensions and a
one bit bitmap, highest bit first, with each row padded to whole bytes.
The two argument form uses a 16x16 one bit shape, taking 32 bytes.
`SPRITE_DATA_8X8` takes eight bytes. `SPRITE_DATA_TILES` takes dimensions
in 8x8 tiles, stored left to right, then top to bottom. Keep the source
data available while the sprite is visible.

For one bit shapes, `SPRITE_COLOR` selects an existing screen palette pen.
Imported color sprites use three screen pens: the `APPLY_FRAME` palette
argument, from 0 to 3, selects pens `4 * palette + 1` through
`4 * palette + 3` and loads the imported colors into them.

`@OPTION SOFT_SPRITE_OPAQUE TRUE` fills
zero bits with the current graphics background pen. `SPRITE_SHAPE_SELECT`
can use the shared monochrome animation slots, including their installed
height. Shape and background buffers use Chip RAM sized to the image
and screen depth; software sprites share no fixed cell or glyph budget.

Hardware changes appear immediately. `SPRITES_FLUSH` applies software
changes, restoring old backgrounds and drawing visible software sprites
in ID order. `SPRITES_RESTORE` restores their backgrounds before other
drawing or scrolling; the next flush draws the sprites again. Hiding
applies at the next flush. Disabling, resetting, changing displays, and
closing the program restore backgrounds; display changes and program
exit also release the software buffers.

`SPRITE_HIT` checks rectangles among visible software sprites.
`SPRITE_HIT_BG` returns 0 for software IDs.

Speed builds can select direct calls for constant IDs. Variable IDs
select hardware or software at each call. `BIND PROC` selects a direct
handler for variable IDs too, using the same global sprite numbers:

```basic
BIND PROC SPRITE_MOVE_TO TO __cb_amiga_sprite_hw_move
SPRITE_MOVE_TO ID, X, Y
BIND PROC SPRITE_MOVE_TO TO __cb_amiga_sprite_sw_move
SPRITE_MOVE_TO ID, X, Y
BIND PROC SPRITE_MOVE_TO TO SPRITE_MOVE_TO
```

## Scrolling

`SCROLL_SETUP SCROLL_TYPE_HW` selects a hardware scrolling band that moves
on either axis. Use `SCROLL_BEGIN_REGION 0, TOP, COLUMNS, ROWS, AXES` to
initialize it with up to one screen width of columns: 40 in low resolution,
80 in hires and 160 in superhires. Fill all `SCROLL_BUFFER_COLUMNS` columns,
including the two hidden columns, and with `SCROLL_AXIS_Y` all
`SCROLL_BUFFER_ROWS` rows, including the hidden row. Vertical scrolling
doubles the band's memory.

Hardware scrolling keeps the screen selected by `DISPLAY`, including
interlaced, EHB and HAM modes. Low resolution bands move in one pixel steps,
hires bands in two and superhires bands in four; use `SCROLL_STEP_X` to round
a speed. Hires bands show up to 16 colors and superhires bands up to 4, so a
deeper screen reopens with that many colors. `CELL` screens continue as
`BITMAP_HIRES`. Starting from `CONSOLE` opens a low resolution 16 color
screen.

A low resolution band scrolls from the left edge of the screen. While it is
open, hardware sprite 7 is not displayed. That is ID 6, or ID 7 without the
mouse pointer.
`@OPTION AMIGA_SCROLL_INSET TRUE` keeps all eight sprites. The screen then
starts 15 pixels further right, and bands hold up to 38 columns. Hires and
superhires bands keep all eight sprites without the inset.

The default scrolling method, also selected by `SCROLL_SETUP SCROLL_TYPE_SW`,
moves in one pixel increments on either axis in every bitmap display mode,
including `CELL`. Starting from `CONSOLE` opens a `CELL` screen.
Region coordinates are screen cells; scrolling cell writes are relative
to the region.

The full region stays visible, with one hidden backing column or row for
each enabled axis. Fill these extra cells using `SCROLL_BUFFER_COLUMNS`
and `SCROLL_BUFFER_ROWS`. Draw the initial picture before beginning, then
use the scrolling cell calls to update its retained buffer. `SCROLL_PRESENT`
shows the staged movement. It draws the next picture on a hidden copy of the
screen and shows it at the next frame, which uses one more screen of Chip RAM.
`@OPTION SCROLL_DOUBLE_BUFFER FALSE` draws on the visible screen instead, as
does a software region opened while a hardware band is open.
`SCROLL_CLOSE` leaves the last picture visible and releases the buffer unless
it was reserved.

Both scrolling methods support `SCROLL_KEEP_BUFFER` on their begin calls
and `SCROLL_RESERVE` before beginning. Reservations use the selected display
depth and include hidden cells, character attributes and the hidden screen
where needed. They
survive closing scrolling and changing screens. `SCROLL_RELEASE` frees the
selected method's buffer; program exit frees both methods' reservations.

Scrolling preserves all bitplanes and the palette, including EHB colors.
HAM modes retain their pixel codes; horizontal movement can cause color
fringes because HAM colors depend on preceding pixels.
`CELL` and `CELL_MULTICOLOR` also retain character colors and inverse attributes.

For a hardware playfield with a software panel, initialize the hardware band
first, then initialize the software region in the fixed rows. Select either
method before moving, filling, presenting or closing it. Each selection
restores that method's buffer sizes and granularity without restarting it.

`AMIGA_SCROLL_OPEN(COLUMNS, TOP, ROWS, BUFFER_COLUMNS)` opens a scrolling
band between fixed rows, using the same screen and color depth rules. It
returns a bitmap handle, or zero if it fails.
The arguments use character cells,
with the same column limit and sprite rules. Set the horizontal
position in pixels with `AMIGA_SCROLL_POSITION X`, then call
`AMIGA_SCROLL_PRESENT` to show it at the next frame.

`AMIGA_SCROLL_GLYPH COL, ROW, GLYPH_ADDR, FG, BG` draws an 8x8 glyph in
the scrolling buffer. Columns and rows start at zero in the full buffer.
`AMIGA_SCROLL_CLEAR` clears that buffer to color zero.
`AMIGA_SCROLL_CLOSE` closes the band, restores the full width bitmap screen
and releases its buffer unless it was reserved. Normal program exit
also closes the band and releases its memory automatically.

## Timing and sound

`TICKS` and the elapsed timer use 50 ticks per second on both PAL and
NTSC systems.
AmigaDOS rounds `DELAY` up to 20 ms steps.

Paula plays four voices with volumes 0 to 64. The note and tone calls
use square waves. `TONE` and `PAULA.SOUND VOICE, PERIOD, VOLUME` use
Paula periods: smaller values give higher notes. Set `REGION` for your
machine to keep notes in tune.

`SOUND_ENVELOPE` supports ADSR settings and tables of duration/level pairs
on all four voices. Envelopes advance in the background at `TICKS_HZ`,
including during waits and input. `SOUND_RELEASE` fades from the current
volume. Envelopes also apply to `AMIGA_SAMPLE_PLAY`.

`PAULA.NOTE_PERIOD(NOTE)` returns the period for a two sample waveform
cycle. Divide it by four for an eight sample cycle.

`BEEP` lets the program continue while the sound plays.
`TONE PERIOD, DURATION, TRUE` plays in the background and stops
automatically, including while the program is in `DELAY`. `BEEP` and
`TONE` durations use milliseconds.

Audio plays WAV clips and four channel, 31 sample ProTracker MOD files
with `M.K.`, `M!K!`, or `4CHN` signatures. WAV clips play once on voice 3;
MOD music loops using all four Paula voices. Both play in the background.
MOD playback follows the song's speed, tempo, effects and sample loops.

`AUDIO_LOAD("music.mod")` loads a MOD from AmigaDOS storage. To package
music with the program, use:

```basic
@INCLUDE_AUDIO MUSIC "MUSIC.CBA" "music.mod"

OK AS U8
OK = AUDIO_LOAD(MUSIC_AUDIO_FILE)
OK = AUDIO_PLAY
```

The one path form, `@INCLUDE_AUDIO MUSIC "music.mod"`, embeds the music;
load it with `AUDIO_LOAD(ADDR MUSIC_AUDIO)`. Embedded audio holds up to
65535 bytes including its CBA header. The file form loads larger songs
into available chip RAM.

`AUDIO_STOP` stops music and keeps it loaded for replay. `AUDIO_PLAY`
restarts from the beginning. `AUDIO_UNLOAD` releases the loaded music.
Use sound and note calls after stopping AUDIO playback.

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

`ON_ERROR` handles file operation errors. `ERR` returns the last error
status. `RESUME` retries the statement; `RESUME_NEXT` continues after it.
End of file does not trigger the handler.

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

## AMOS BASIC

Use `@OPTION DIALECT amos` for AMOS text listings. Tokenized `.AMOS`
files select the dialect automatically. Unsuffixed numbers are signed
32 bit integers; `#` selects real numbers and `$` selects strings.
Procedures use `PROCEDURE` and `END PROC`, with arguments in square
brackets. `PARAM`, `PARAM#`, and `PARAM$` read procedure results.
Each call starts private scalar numbers at zero and strings empty.
`GLOBAL` and `SHARED` accept scalar name patterns with `*` and `?`.
Shared arrays use explicit names followed by `()`.

`SPACE$` and `STRING$` build repeated text; `FLIP$` reverses a string.
`INSTR` accepts an optional starting position. `HEX$` and `BIN$` include
their `$` and `%` prefixes and accept a digit count. `STR$` puts a space
before positive numbers and zero.

`GR LOCATE` sets the graphics cursor. `DRAW TO` and `POLYLINE TO`
continue from it; `XGR` and `YGR` read its coordinates. `PLOT`,
`CIRCLE`, and `ELLIPSE` accept omitted cursor coordinates.
`HIDE` and `SHOW` count nested pointer visibility changes. `HIDE ON`
and `SHOW ON` reset that count and force the pointer hidden or visible.

`SCREEN OPEN` creates numbered screens from 0 to 7. `SCREEN N` selects
the drawing screen. Each screen keeps its pixels, palette, pens, cursor
and detection zones. `SCREEN CLOSE N` releases a screen. `SCREEN` reads
the current number; `SCREEN WIDTH` and `SCREEN HEIGHT` read its size,
or accept a screen number in parentheses.

`SPRITE N,X,Y,IMAGE` displays a bank image at hardware coordinates.
Omitted coordinates and image numbers keep their previous values.
`X SPRITE`, `Y SPRITE` and `I SPRITE` read those settings; `SPRITE OFF`
hides all sprites, or accepts one sprite number.

Numbers 0 to 7 select hardware channels; numbers 8 to 63 share channels
at different vertical positions. Wider images use adjacent channels;
15 color images use attached pairs. Images can span 128 pixels with three
visible colors, or 64 pixels with 15 colors. These forms work on OCS, ECS
and AGA.
`HIDE ON` makes channel zero available. Overlapping images share the eight
channel limit, with attached images consuming twice as many channels.

Updates run at vertical blank. `SPRITE UPDATE OFF` holds the displayed
images until `SPRITE UPDATE`; `SPRITE UPDATE ON` restores automatic updates.
`SET SPRITE BUFFER ROWS` sets each channel list's capacity, including one
control row per image. Sprite banks retain their palettes and hotspots.

`EVERY interval PROC name` schedules a procedure call in 50ths of a
second. Include `EVERY ON` before `END PROC` to repeat the call.
`EVERY OFF` disables it. Timed calls continue during loops, `WAIT`
and `WAIT KEY`.

`SET ENVEL 1,PHASE TO FRAMES,LEVEL` defines phases 0 to 6 of the
default square wave's volume envelope. Levels range from 0 to 63.
Define phases in order: setting a phase removes the following phases.
A zero frame count ends the envelope. Envelope times follow video
frames, rounded to the nearest sound tick.

`PLAY PITCH,FRAMES` starts the note on all four voices. Use
`PLAY MASK,PITCH,FRAMES` to select voices with bits 1, 2, 4 and 8.
Pitches range from 1 to 96; pitch 0 silences the selected voices.
The frame count delays the next statement while the envelope runs
independently. A count of zero returns immediately. `PLAY OFF` stops
all voices; `PLAY OFF MASK` stops only the selected voices.

`PEN` and `PAPER` set text colors. `INK` sets drawing colors, with optional
background and border colors. Omitted `INK` values keep their current
settings. `SET PAINT 1` adds a border to bars; `SET PAINT 0` draws plain bars.

`PRINT` uses the 8 pixel Topaz font. `INVERSE ON` swaps text pen and paper;
`INVERSE OFF`, `PEN`, or `PAPER` restores normal mode.
`TEXT` draws at pixel coordinates and advances the graphics cursor.
`TEXT BASE` and `TEXT LENGTH` give its baseline and width. `SET TEXT`
selects underline, bold and italic styles for `TEXT`; `TEXT STYLES`
reads the selected style.

`COLOUR` reads or sets a palette entry. `FLASH` cycles a color through
RGB values and frame delays while the program runs. An empty sequence
stops one color; `FLASH OFF` stops all sequences on the current screen.
Screens start with the AMOS palette and color 3 flashing when the screen
has at least four colors.

`RESERVE ZONE` allocates rectangular detection zones. `SET ZONE` defines
their inclusive corners; `ZONE` and `MOUSE ZONE` return the first matching
zone number. `RESET ZONE` clears one or all definitions. Omitting the count
from `RESERVE ZONE` releases their memory.

`RESERVE AS WORK` and `RESERVE AS DATA` create numbered memory banks;
their `CHIP` forms allocate Chip memory. `START` returns a bank address
and `LENGTH` returns its size. `BANK SWAP` exchanges bank assignments;
`BANK SHRINK` releases the unused tail while keeping the address.
`ERASE`, `ERASE TEMP`, and `ERASE ALL` release banks. Attached Work and
Data banks retain their names and contents when the program starts.
Attached sprite and icon banks retain their image numbers, empty slots,
sizes, palettes and hotspots. `LENGTH` returns their image count.
`LOAD` and `SAVE` read and write raw bank files, including groups of banks.
`BLOAD` and `BSAVE` transfer binary memory blocks without bank headers.

`PASTE ICON X,Y,N` draws an image from bank 2, including colour zero.
`PASTE BOB X,Y,N` draws from bank 1 with colour zero transparent.
Both use the image's top left corner and clip at the screen edges.
`HREV`, `VREV` and `REV` select horizontal, vertical or combined flips.
Pastes keep the current screen palette and drawing pens.

`UNPACK BANK` draws a packed picture at its stored position.
`UNPACK BANK,X,Y` draws at a new position, rounding X down to an eight
pixel boundary. The picture must fit the current screen and use the same
number of bitplanes. Both forms keep the screen palette and drawing state.
`UNPACK BANK TO N` creates or replaces screen N with the stored dimensions,
mode and palette, then draws the picture at its top left corner.
A memory address can replace the bank number.

`OPEN IN`, `OPEN OUT`, `APPEND`, and `CLOSE` manage sequential files.
Use `PRINT #`, `INPUT #`, `LINE INPUT #`, and `INPUT$` for file data.
`POF` reads or changes the file position; `LOF` gives its length and
`EOF` tests whether the current position is at the end.

## References

- [Amiga Hardware Reference Manual](http://amigadev.elowar.com/read/ADCD_2.1/Hardware_Manual_guide/node0000.html)
- [ROM Kernel Reference Manual: Libraries](http://amigadev.elowar.com/read/ADCD_2.1/Libraries_Manual_guide/node0000.html)
- [AmigaDOS Manual](http://amigadev.elowar.com/read/ADCD_2.1/AmigaDOS_Manual_guide/node0000.html)
