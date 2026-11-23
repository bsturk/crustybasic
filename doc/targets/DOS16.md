# crustyBASIC on 16 bit DOS

Use `@OPTION TARGET dos16` for the canonical DOS executable profile.
Exact systems include `dos16.exe` and `dos16.com`.

Portable runtime calls are documented in [`../API.md`](../API.md).
This page covers DOS systems, modes, formats, and hardware notes. For
command line target selection and dialects, see
[`../USAGE.md`](../USAGE.md).

## Systems

| System | Output | Notes |
| --- | --- | --- |
| `dos16.exe` | `.exe`, optional `.img` | Generic 16 bit DOS MZ executable. |
| `dos16.com` | `.com`, optional `.img` | Generic 16 bit DOS COM program, loaded at `$0100`. |

NASM is the supported assembler. The optional `.img` artifact is a
mountable 360K FAT12 disk image when the DOS floppy helper is
configured. Files staged by a two-path `@INCLUDE_*` form go on that image,
uppercased and trimmed to the DOS 8 character name plus 3 character
extension.

## Basics

| Item | Value |
| --- | --- |
| CPU | 8086 real mode |
| Text | 80x25 |
| Code storage | RAM |
| String encoding | ASCII, with non ASCII bytes encoded as `?` |
| Newline byte | `$0D` |
| Integer math | 8, 16, and 32 bit integers; division requires a 16 bit divisor |
| `REAL` math | Built in fixed point, `Q24_8` by default |
| Elapsed timer | BIOS tick count, about 18.2 Hz |
| Host OS | DOS services are available |

All current DOS systems use a 256 byte string buffer limit.

`READ` supports the full `I32` and `U32` ranges.

`ON_ERROR`, `ON_ERROR OFF`, `ERR`, `RESUME`, and `RESUME_NEXT` support
file error handling. Failed file operations report their file status
through `ERR`; reaching the end of a file does not trigger the handler.

File calls report `FILE_NOT_FOUND`, `FILE_TOO_MANY_OPEN`,
`FILE_ACCESS_DENIED`, `FILE_DISK_FULL` for a short write, and
`FILE_ERROR` for other DOS errors. A disk error such as a drive that is
not ready fails the call with `FILE_ERROR` instead of showing the DOS
Abort, Retry, Fail prompt.

## Direct Memory Access

DOS adds byte access overloads with an explicit segment and offset:

```basic
VALUE = PEEK(SEGMENT, OFFSET)
POKEB SEGMENT, OFFSET, VALUE
```

`SEGMENT` and `OFFSET` are `U16`; `VALUE` is `U8`. These access real mode
memory at `SEGMENT * 16 + OFFSET`. They do not change the program's data
segment or subsequent memory accesses. Ordinary `PEEK(OFFSET)` and
`POKEB OFFSET, VALUE` still access the program's data segment.

`MEMMOVE(SRC_SEGMENT, SRC, DST_SEGMENT, DST, COUNT)` copies `COUNT` bytes
forward between segment and offset pairs. All five arguments are `U16`.
The program's data segment is restored after the copy.

| Constant | Type | Value | Memory |
| --- | --- | --- | --- |
| `VIDSEG` | `U16` | `$B800` | Color text video RAM |
| `BDASEG` | `U16` | `$0040` | BIOS data area |

For example, `POKEB VIDSEG, 0, 65` writes an `A` to the first text cell,
and `PEEK(BDASEG, $49)` reads the BIOS video mode byte.

In 80x25 color text mode, segment `$B800` holds character and attribute
byte pairs. Cell `(ROW, COL)` starts at offset `(ROW * 80 + COL) * 2`.
The attribute is `BACKGROUND * 16 + FOREGROUND`, with foreground colors
0 through 15, background colors 0 through 7, and bit 7 controlling blink.
The BIOS data area is at segment `$0040`; offsets `$006C` through `$006F`
hold the live tick count, lowest byte first. Separate byte reads are not
an atomic snapshot of that counter.

## Text And Cells

Programs start in `CONSOLE`, using normal DOS text output.
`DISPLAY CONSOLE` returns to 80x25 text output after graphics.

Clearing, positioning, and cursor visibility use BIOS text services.
Cell display selects BIOS mode 3, 80x25 text with 16 colors.

In bitmap modes, `PRINT`, `PRINT_CHAR`, and text clearing use the active
drawing surface and the `COLOR` pair. Buffered text appears at `GFX_SWAP`
and stays in the background when sprites move. Text uses the BIOS font
and character height for the selected mode.

| Target | Cell color | Cell attributes |
| --- | --- | --- |
| `dos16` | Yes | Yes |

`CELL_COLOR` sets a cell's foreground and gives it the `COLOR`
background. `INVERSE` shows a cell with its colors swapped and `NORMAL`
puts them back. `CELL_COLORS FG, BG` fills the screen and becomes the
`COLOR` pair that `PRINT` uses.

Cell storage is allocated in DOS memory for the active screen and
released when the display mode changes. Allocation failure exits with
code 8.

The shared color constants use the DOS/BIOS 0 through 15 palette.
`ORANGE` maps to `BROWN`, and `GRAY` maps to `LIGHT_GRAY`.

The charset API manages 256 character shapes, with characters 128 through
255 available for custom artwork. Read `CHARSET_BYTES_PER_ENTRY` after
selecting a display. Bitmap glyphs use eight bytes; text glyphs use the
BIOS character height.

Text font changes update existing characters. Redraw bitmap text after
changing its glyphs. Graphics `PRINT` scales custom glyphs to its character
height. Selecting a display restores its normal charset.

## Graphics

The default graphics mode is `BITMAP_HIRES`.

| Target | Mode | Size | Colors | Row bytes | Bitmap base |
| --- | --- | --- | --- | --- | --- |
| `dos16` | VGA mode 13h | 320x200 | 256 | 320 | `$A000` |
| `dos16` | VGA mode 12h | 640x480 | 16 | 80 per plane | `$A000` |
| `dos16` | EGA mode 10h | 640x350 | 16 | 80 per plane | `$A000` |

| Mode | DOS mode | Size | Colors |
| --- | --- | --- | --- |
| `TEXT_40X25` | BIOS mode 1 text | 40x25 | 16 |
| `CONSOLE` | DOS text output | 80x25 | 16 |
| `CELL`, `TEXT_80X25` | BIOS mode 3 text | 80x25 | 16 |
| `TEXT_80X50` | VGA mode 3 text, 8x8 font | 80x50 | 16 |
| `BITMAP_HIRES` | VGA mode 13h | 320x200 | 256 |
| `VGA_640X480_16` | VGA mode 12h | 640x480 | 16 |
| `EGA_640X350_16` | EGA mode 10h | 640x350 | 16 |
| `BITMAP_LORES` | not supported | | |
| `BITMAP_MULTICOLOR` | not supported | | |
| `CELL_MULTICOLOR` | not supported | | |

Use `DISPLAY VGA_640X480_16` for 640x480 graphics with independent
pixel colors. It supports drawing, fills, `POINT`, and `GFX_CELL_BLIT`.

QBasic `SCREEN 9` selects `EGA_640X350_16`. Its `PALETTE` statement
maps the 16 pixel colors to the 64 EGA colors.

`PALETTE_SET index, RGB(r, g, b)` changes any of the 256 palette entries
in mode 13h or the 16 pixel colors in modes 10h and 12h.
Channels range from 0 through 255 and are reduced
to six bits for the VGA DAC. Existing pixels using that index change
color too. `PALETTE_SET index, RED` uses a standard color.
`PALETTE_RESET` restores the display mode's palette without changing
pixels, drawing colors, or active buffers.

`GFX_BUFFERING` is `GFX_BUFFERING_COPY`. In `BITMAP_HIRES`,
`VGA_640X480_16`, and `EGA_640X350_16`, the first `GFX_SWAP` after
`DISPLAY` preserves the picture and starts buffered drawing. Drawing
calls and `POINT` then use that buffer. Later swaps copy the changed
rectangle to the screen and retain the buffer contents.

Call `FRAME_WAIT` before `GFX_SWAP` to pace presentation. Sprite updates
also go into the drawing buffer while buffering is active. Selecting a
display mode releases the buffer and resumes direct drawing.

The buffer uses DOS memory: 64000 bytes for 320x200, 112000 bytes for
640x350, or 153600 bytes for 640x480. If DOS cannot allocate it, the
program exits with code 8.

`GFX_BUFFER_INIT` requests the buffer explicitly and returns `TRUE` on
success or `FALSE` when allocation fails. A failed request leaves drawing
direct, so the program can continue without a buffer or select another mode.

## Scrolling

`SCROLL_SETUP SCROLL_TYPE_SW` selects scrolling in every display mode.
After `SCROLL_BEGIN`, both scroll granularities are 1: `SCROLL_MOVE`
accepts signed pixel amounts on either enabled axis. Positive X moves
content left; positive Y moves it up. Move one axis and cross at most
one cell boundary per call.

Text scrolling uses VGA panning and preserves character colors and
attributes. Cell dimensions follow the active font, including 80x50
text. Graphics scrolling moves every color plane, preserving all 16 or
256 pixel colors, and uses 8x8 cells.

Each enabled axis reserves one backing cell for incoming content.
Use `SCROLL_BUFFER_COLUMNS`, `SCROLL_BUFFER_ROWS`,
`SCROLL_VIEW_COLUMNS`, and `SCROLL_VIEW_ROWS` for the active dimensions.
Fill the edge returned by `SCROLL_MOVE` with `SCROLL_CELL_PUTC` or
`SCROLL_CELL_COLUMN`, then call `SCROLL_PRESENT`. The four argument
`SCROLL_CELL_PUTC` selects the foreground color; the background comes
from `COLOR`.

`SCROLL_BEGIN` retains the current picture. Graphics drawing continues
in the retained buffer; use `SCROLL_PRESENT` to show the scrolled view.
Text writes to the scrolling surface use `SCROLL_CELL_PUTC` and
`SCROLL_CELL_COLUMN`. `SCROLL_CLOSE` restores the full view and text
cursor settings. Graphics buffering remains active until the next
`DISPLAY` call.

Text scrolling requires VGA and uses about 3, 6, or 12 KB of DOS
memory. Graphics uses the drawing buffer sizes listed above. Selecting
another display mode releases the buffer. Allocation failure exits with
code 8.

## Images

Native image display supports:

| Target | Format | Accepted files |
| --- | --- | --- |
| `dos16` | `IMAGE_FMT_CGA4` | Indexed PNG/PCX, CGA mode 4 BSAVE, or raw 16384 byte dumps |
| `dos16` | `IMAGE_FMT_VGA256` | Indexed PNG/PCX, 320x200 with 256 colors |
| `dos16` | `IMAGE_FMT_EGA16` | Indexed PNG/PCX, 640x350 with 16 colors |
| `dos16` | `IMAGE_FMT_VGA16` | Indexed PNG/PCX, 640x480 with 16 colors |

Select VGA images with `@INCLUDE_IMAGE PICTURE "picture.pcx" AS vga256`.
Use `AS cga4` for CGA; it is also the default when `AS` is omitted.
Select planar images with `AS ega16` or `AS vga16`. Smaller sources are
centered in the selected display. PNG input uses 8 bit indexed color.
VGA images preserve the source palette, with RGB channels reduced to six
bits. EGA colors are rounded to its palette of 64 colors. Each format selects
its display mode when loaded.

`IMAGE_FMT_CGA4` uses B800 dump order. On generic DOS the aux byte is
the raw CGA color select value for port `3D9h`.

`IMAGE_LOAD(path$)` reads converted native image files from DOS storage.
Raw and RLE8 packed images stream directly to video memory. Converted
images use RLE8 when it makes them smaller. VGA256 data contains 64000
pixel bytes and 768 palette bytes before compression. Planar images store
four planes followed by 16 EGA palette bytes or 48 VGA palette bytes.

Embedded images share the DOS executable's 64 KB address space.

CGA images, packed or unpacked, support wipes and slides in all four
directions, splits, fade in and out, and dissolves with 1, 2, 4, or 8
pixel square blocks. Packed effects use a temporary 16 KB DOS memory
block and return 0 when DOS cannot supply it. Wipes and
slides move in steps of four pixels horizontally or one pixel vertically.
CGA fades use four levels of pixel density. Fade out and dissolve out work
on the current CGA screen, including images loaded with RLE8 packing.

VGA images, packed or unpacked, support the same wipes, slides, splits,
and dissolves. These effects decode the image into a 64 KB block of DOS
memory and return 0 when DOS cannot supply it. VGA fades step the palette
through four levels from or to black. In programs with VGA images or no
images, fade out and dissolve out also work on any `BITMAP_HIRES` screen.

## Sprites

The DOS16 target does not have hardware sprites. Sprites are backed by
8x8 opaque software blits in the 320x200 `BITMAP_HIRES` mode.

| Feature | Value |
| --- | --- |
| Sprite count | Selected by `SPRITE_COUNT`, default 8 |
| Size | Runtime `w * 8` by `h * 8` pixels |
| Data bytes | 8 per tile |
| Surface | Pixel |
| Collision | Sprite masks and background scenery |
| Flip | Horizontal and vertical, per sprite |
| Expansion | Double width and height independently |
| Priority | In front of or behind background pixels |

On `dos16`, sprites compose into off screen buffers when page flip
support is enabled, then copy touched rectangles after the frame wait.
`SPRITE_DATA_TILES` stores tiles left to right and then top to bottom.
`SPRITE_FLIP_X` and `SPRITE_FLIP_Y` mirror the whole shape without changing
its source data. `SPRITE_EXPAND` doubles either or both dimensions.
Sprites are clipped at the right and bottom edges.
`SPRITE_PRIORITY` with a true flag leaves foreground scenery visible. A
background pixel matching the current `COLOR` background lets the sprite
show through.

`SPRITE_HIT` reports overlapping sprite mask pixels. `SPRITE_HIT_BG`
checks scenery against the current background color. Both queries use the
current recorded sprite settings and respect masks, flips, expansion,
and screen edges.

## Tile

The DOS16 target uses the generic tile runtime.

| Target | Default backend | Cell tiles | Bitmap tiles | Tile colors |
| --- | --- | --- | --- | --- |
| `dos16` | `TILE_BITMAP` | Yes | Yes | Yes |

Tile definitions, blocks, and colors are available through the generic
runtime. Tile kernels are not available on these targets.

## Input

| Capability | Value |
| --- | --- |
| Keyboard | Yes |
| Joystick ports | 2 |
| Buttons | 2 per port |
| Stick type | Analog, exposed through digital direction masks |
| Paddle axes | 4 |
| Paddle range | 0 through 255 |
| Keypad ports | 0 |
| Keypad `INPUT` | No |
| Mouse buttons | Up to 3 through a DOS mouse driver |

Keyboard input uses BIOS `int 16h` and is nonblocking. Lowercase
letters are normalized to uppercase. Extended arrow keys return the
portable cursor bytes.

Mouse input uses a Microsoft compatible DOS driver. `MOUSE_X`, `MOUSE_Y`,
and `MOUSE_POSITION` use graphics pixels, including 320x200 modes.
`MOUSE_STATE` uses bits 0, 1, and 2 for left, right, and middle buttons.
`MOUSE_SHOW` and `MOUSE_HIDE` control the pointer.

Joystick and paddle input use BIOS joystick services. Digital joystick
reads threshold the analog axes with the default deadzone value of 48,
and paddle reads return the raw axis value.

## Sound

| Target | Hardware | Voices | Volume max | Note range |
| --- | --- | --- | --- | --- |
| `dos16` | PC speaker | 1 | 1 | 36 through 95 |

Sound supports one PC speaker voice, simple tones, and note frequency
lookup. Envelope, filter, and pulse width controls are not supported on
this target.

`TONE` and 2 argument `BEEP` durations count BIOS ticks, about 55 ms
each. `BEEP` rounds its milliseconds to whole ticks. The period is the speaker timer divisor, 1193180 divided by
the frequency in Hz.

QBasic `PLAY` supports notes, rests, octaves, tempo, note lengths, dotted
notes, and articulation. `MB` queues `PLAY` and QBasic `SOUND` in the
background; `MF` waits for playback to finish. Note timing follows the BIOS timer, about 55 ms per tick.

## Timing

Frame waits poll the vertical retrace status bit with a timeout. The
runtime tick source is the BIOS tick count at `0040:006C`. The true
rate is about 18.2065 Hz, and the midnight reset is compensated for one
crossing.

`DELAY` counts milliseconds on the PC timer, so it runs the same on any
CPU speed.

## Files

The DOS file provider uses DOS `int 21h` file handles. Read, write,
append, update, and directory modes are supported. Directory paths use
DOS wildcards; read names with `FILE_READ_LINE` until `FILE_EOF`. Each
channel maintains its own listing. Native opens use `AUX1` as the mode
value and ignore `AUX2`. File channels run from 0 to 15, and up to 15 files
can be open at once when `FILES=` in `CONFIG.SYS` allows it.

`FILE_SEEK` sets an absolute byte offset. `FILE_POSITION` returns the
zero based offset of the next byte the program will read as `U32`.

`FILE_COMMAND` reports its result through `FILE_STATUS(channel)` and
ignores `AUX1` and `AUX2`:

| Command | Path |
| --- | --- |
| `FILE_COMMAND_DELETE` | File to delete |
| `FILE_COMMAND_RENAME` | Old and new paths separated by a pipe character |
| `FILE_COMMAND_MKDIR` | Directory to create |
| `FILE_COMMAND_RMDIR` | Empty directory to remove |
