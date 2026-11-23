# crustyBASIC on the Atari ST

Use `--set target=atarist` to build a TOS program (`.prg`) and a floppy
image containing it (`.st`). The Atari ST requires `vasmm68k_mot`. Install
it in `tools/assemblers/vasm/<platform>/` or on `PATH`. See the
[assembler setup](../USAGE.md#assemblers).

Choose `--set system=atarist.gem` for a GEM application (`.app`),
`atarist.console` for a console program (`.tos`), or `atarist.ttp` for a
program that takes command line arguments (`.ttp`). Use
`--set output-type=disk` for a floppy image (`.st`) or
`--set target=atarist --set output-type=cart` for a cartridge image (`.img`).

Select ST BASIC with `--set dialect=st_basic` or GFA BASIC 3.x with
`--set dialect=gfa_basic`.

## Paddles

The `atarist` target reads four analog axes from the enhanced controller
ports on the STE and Falcon030.

| Axis | Input |
| --- | --- |
| 0 | Port A, X |
| 1 | Port A, Y |
| 2 | Port B, X |
| 3 | Port B, Y |

`PADDLE(AXIS)` returns the raw position as `U8`. `PADDLE_AXES` is 4 and
`PADDLE_MAX` is 255. Each port accepts a pair of paddles.

`PADDLE_BUTTON(AXIS)` returns 1 while the paddle's button is pressed and
0 when released.

Both calls detect the machine before accessing the controller ports.
On other ST models, position reads return 0 and button reads return 0.

## Keyboard

`KEY_HELD` finds letter and digit keys by their position on a US keyboard.
Typed input keeps working while it is in use.

## Mouse

Programs start with the mouse pointer hidden. `MOUSE_SHOW` and
`MOUSE_HIDE` control it. `atarist.gem` programs and GFA BASIC listings
show the arrow pointer.

## Heap

The heap arena defaults to 65,534 bytes. Set its size with
`@OPTION HEAP_BYTES 131072` or `--set heap-bytes=131072`.
`HEAP_AVAIL` returns the total free bytes across the arena.

## Errors

`ON_ERROR` handles file operation errors. `ERR` returns the last error
status. `RESUME` retries the statement; `RESUME_NEXT` continues after it.
End of file does not trigger the handler.

## Integers

`U64` and `I64` provide exact 64 bit integer calculations.

## Files

`FILE_SEEK` sets the byte position from the start of an open file.
`FILE_POSITION` returns the position of the next byte to read or write.

Open a directory with `FILE_MODE_DIR`. Read its filenames with
`FILE_READ_LINE`, one filename per line. Paths may contain `*` and `?`.
`FILE_STATUS` returns `FILE_EOF` at the end of the directory.

`FILE_LOAD_PROGRAM(path$, dst)` loads a TOS program without running it.
It places the code and data at `dst`, adjusts the program's addresses for
that location, and clears its BSS. `dst` needs room for all three.

For the `FILES_*` interface, `FILES_STATUS` returns `FILES_OK` after a
successful operation and `FILES_ERR` after an error.

## Bitmap display

| Resolution | Size | Colors | `GFX_BPP` |
| --- | --- | --- | --- |
| Low | 320 by 200 | 16 | 4 |
| Medium | 640 by 200 | 4 | 2 |
| High | 640 by 400 | 2 | 1 |

`BITMAP_HIRES` draws in the current resolution. `DISPLAY ST_LOW`,
`DISPLAY ST_MED`, and `DISPLAY ST_HIGH` switch resolution and then select
`BITMAP_HIRES`. A color monitor shows low and medium resolution and a mono
monitor shows high resolution. `DISPLAY` returns `0` for a resolution the
monitor cannot show.

## Colors

Programs start with this palette, and console text is `WHITE` on `BLACK`:

| Pixel | Color | Pixel | Color |
| --- | --- | --- | --- |
| 0 | `BLACK` | 8 | `ORANGE` |
| 1 | `WHITE` | 9 | `BROWN` |
| 2 | `RED` | 10 | `LIGHT_RED` |
| 3 | `CYAN` | 11 | `DARK_GRAY` |
| 4 | `PURPLE` | 12 | `GRAY` |
| 5 | `GREEN` | 13 | `LIGHT_GREEN` |
| 6 | `BLUE` | 14 | `LIGHT_BLUE` |
| 7 | `YELLOW` | 15 | `LIGHT_GRAY` |

Medium resolution shows pixels 0 to 3 and high resolution pixels 0 and 1.
Pixel 0 is the background. `CLS` clears to it and `UNPLOT` draws it.
`BACKGROUND_COLOR` changes its color. `PALETTE_RESET` loads this palette
again.

`atarist.gem` programs, ST BASIC listings, and GFA BASIC listings keep the
desktop palette. An `atarist.gem` program switches to this palette when it
selects a `DISPLAY` mode. The desktop palette returns when the program ends.

## Cells and character sets

Cell calls draw 8 by 8 pixel characters. The grid is 40 by 25 cells in low
resolution, 80 by 25 in medium, and 80 by 50 in high. `DISPLAY(CELL)`
reports these dimensions. TOS console output keeps its native text height.

Each cell has its own foreground color and shares the background selected
by `CELL_COLORS`. Attributes include `NORMAL`, `INVERSE`, and `ITALIC`.
`CELL_GETC` returns character codes written through cell calls.

`CHARSET_COPY_DEFAULT` copies the system 8 by 8 font. `CHARSET_INSTALL`
loads 256 glyphs, eight consecutive bytes per glyph. `CHARSET_DEFINE` and
`CHARSET_INSTALL_RANGE` replace individual glyphs. `CHARSET_RESET` selects
the system font again. Character set changes apply to subsequent cell and
scroll drawing calls.

## Scrolling

Software scrolling moves bitmap graphics one pixel at a time on either
axis. `SCROLL_BEGIN` uses the full screen; `SCROLL_BEGIN_REGION` keeps
pixels outside a cell rectangle fixed.

The backing buffer adds one cell along each enabled axis. Use
`SCROLL_CELL_PUTC` to draw into it, `SCROLL_MOVE` to advance the view, and
`SCROLL_PRESENT` to show it after a frame wait. The fill result identifies
the newly exposed buffer edge. `SCROLL_CLOSE` releases the buffer.

## Sprites

Sprites are drawn in software on the bitmap display. `GFX_SWAP` builds
sprite changes in a 32000 byte off screen buffer, waits for the vertical
blank, then copies the changed areas to the screen. The following
`FRAME_WAIT` counts that wait. `@OPTION SPRITE_DOUBLE_BUFFER FALSE` saves
the buffer and redraws sprites on the visible screen instead.

## Timing and sound

`DELAY`, `BEEP`, and `TONE` durations use milliseconds in 5 ms steps of the
200 Hz system clock. `TONE` takes a YM2149 tone period: smaller values give
higher notes. `BEEP` and `TONE` wait until the sound ends.

## Sound envelopes

`SOUND_ENVELOPE` sets a separate volume envelope for each of the three
voices. It accepts attack, decay, sustain, and release values, or a table
of duration and level pairs. Levels range from 0 to 15. Durations use the
200 Hz tick clock.

`SOUND_RELEASE` starts the release phase. `SOUND_OFF` stops a voice
immediately. Envelopes and timed notes advance during `FRAME_WAIT`, `DELAY`,
and timed sound status calls. A timed note stops when its duration expires.

## GFA BASIC program control

`RUN` restarts the current program, clears variables and arrays, resets
DATA reads, and closes open files. It also works from a procedure.

`CHAIN PATH$` and `RUN PATH$` hand control to a compiled program through
GEM, passing the current command line. The filename is used as written.
`LOAD` is ignored as an editor command.
