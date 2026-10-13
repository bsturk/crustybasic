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

| Target | Cell color | Cell attributes |
| --- | --- | --- |
| `dos16` | Yes | Yes |

`CELL_COLOR` sets a cell's foreground and gives it the `COLOR`
background. `INVERSE` shows a cell with its colors swapped and `NORMAL`
puts them back. `CELL_COLORS FG, BG` fills the screen and becomes the
`COLOR` pair that `PRINT` uses.

The shared color constants use the DOS/BIOS 0 through 15 palette.
`ORANGE` maps to `BROWN`, and `GRAY` maps to `LIGHT_GRAY`.

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
color too. The standard color overload and `PALETTE_RESET` are not
supported; selecting the display mode again restores its palette.

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

## Images

Native image display supports:

| Target | Format | Accepted files |
| --- | --- | --- |
| `dos16` | `IMAGE_FMT_CGA4` | Indexed PNG/PCX, CGA mode 4 BSAVE, or raw 16384 byte dumps |
| `dos16` | `IMAGE_FMT_VGA256` | Indexed PNG/PCX, 320x200 with 256 colors |

Select VGA images with `@INCLUDE_IMAGE PICTURE "picture.pcx" AS vga256`.
Use `AS cga4` for CGA; it is also the default when `AS` is omitted.
Sources up to 320x200 are centered in the screen. PNG input uses 8 bit
indexed color. VGA images preserve the source palette, with RGB channels
reduced to six bits, and select mode 13h when displayed.

`IMAGE_FMT_CGA4` uses B800 dump order. On generic DOS the aux byte is
the raw CGA color select value for port `3D9h`.

Images are linked into the program rather than loaded from DOS files at
runtime. Converted images use RLE8 when it makes them smaller. VGA data
contains 64000 pixel bytes and 768 palette bytes before compression.
The image and program share the DOS executable's 64 KB address space.

Unpacked CGA images support wipes and slides in all four directions, fade in
and out, and dissolves with 1, 2, 4, or 8 pixel square blocks. Wipes and
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
| Collision | Not supported |
| Flip, stretch, priority, palette | Not supported |

On `dos16`, sprites compose into off screen buffers when page flip
support is enabled, then copy touched rectangles after the frame wait.
`SPRITE_DATA_TILES` stores tiles left to right and then top to bottom.

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
| Mouse buttons | 0 |

Keyboard input uses BIOS `int 16h` and is nonblocking. Lowercase
letters are normalized to uppercase. Extended arrow keys return the
portable cursor bytes.

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

`BEEP`, `TONE`, and 2 argument `BEEP` durations count BIOS ticks, about
55 ms each. The period is the speaker timer divisor, 1193180 divided by
the frequency in Hz.

QBasic `PLAY` supports notes, rests, octaves, tempo, note lengths, dotted
notes, and articulation. `MB` plays in the background; `MF` waits for the
music to finish. Note timing follows the BIOS timer, about 55 ms per tick.

## Timing

Frame waits poll the vertical retrace status bit with a timeout. The
runtime tick source is the BIOS tick count at `0040:006C`. The true
rate is about 18.2065 Hz, and the midnight reset is compensated for one
crossing.

`DELAY` counts milliseconds on the PC timer, so it runs the same on any
CPU speed.

## Files

The DOS file provider uses DOS `int 21h` file handles. Read, write,
append, and update modes are supported. Directory mode and command
channels are not supported. Native opens use `AUX1` as the mode value
and ignore `AUX2`. File channels run from 0 to 15, and up to 15 files
can be open at once when `FILES=` in `CONFIG.SYS` allows it.
