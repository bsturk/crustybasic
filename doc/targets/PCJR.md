# crustyBASIC on IBM PCjr

Use `@OPTION TARGET pcjr` for the default PCjr profile, or pick the
exact system with `@OPTION SYSTEM pcjr.exe`.

Portable runtime calls are documented in [`../API.md`](../API.md).
This page covers PCjr systems, modes, formats, and hardware notes. For
command line target selection and dialects, see
[`../USAGE.md`](../USAGE.md).

Segment memory access uses `PEEK(SEGMENT, OFFSET)` and
`POKEB SEGMENT, OFFSET, VALUE`. `VIDSEG` is `$B800` for color text
and `BDASEG` is `$0040` for BIOS data. See
[direct memory access](DOS16.md#direct-memory-access) for details.

## Systems

| System | Output | Notes |
| --- | --- | --- |
| `pcjr.exe` | `.exe`, optional `.img` | IBM PCjr DOS MZ executable. |

NASM is the supported assembler. The optional `.img` artifact is a
mountable 360K FAT12 disk image when the DOS floppy helper is
configured. Files staged by a two-path `@INCLUDE_*` form go on that image,
uppercased and trimmed to the DOS 8 character name plus 3 character
extension.

For a 720K FAT12 `.img`, use `@OPTION OUTPUT_TYPE disk-720k` or:

```bash
crustybasic build hello.cbs --set target=pcjr --set output-type=disk-720k -o hello.img
```

The image contains the executable and companion files. The `.exe` is also
kept separately. Boot DOS before running the program from the image.

## Basics

| Item | Value |
| --- | --- |
| CPU | 8086 real mode |
| Text | 80x25 |
| Code storage | RAM |
| String encoding | ASCII, with non ASCII bytes encoded as `?` |
| Integer math | Built in 16 bit integer support |
| `REAL` math | Built in fixed point, `Q24_8` by default |
| Sound | SN76496 compatible, port `0C0h` |
| Host OS | DOS services are available |

The current system is `pcjr.exe`; there is no PCjr `.com` system.

## Text And Cells

Cell display selects BIOS mode 3, 80x25 text with 16 colors. Clearing,
positioning, and cursor visibility use BIOS text services.

Cell drawing uses B800 text memory. Cell color and attributes are
available; attributes include inverse and blink. `COLOR` sets the
foreground and background that `PRINT` uses, and `CURSOR_HIDE` and
`CURSOR_SHOW` control the BIOS cursor.

`CELL_COLOR` sets a cell's foreground and gives it the `COLOR`
background. `INVERSE` shows a cell with its colors swapped and `NORMAL`
puts them back. `CELL_COLORS FG, BG` fills the screen and becomes the
`COLOR` pair that `PRINT` uses.

## Graphics

The default graphics mode is `BITMAP_HIRES`.

| Mode | BIOS mode | Size | Colors | Row bytes | Bitmap base |
| --- | --- | --- | --- | --- | --- |
| `CELL` | 3 | 80x25 | 16 | 0 | text memory |
| `PCJR_TANDY_GRAPHICS_4` | 4 | 320x200 | 4 | 80 | `$B800` |
| `PCJR_TANDY_GRAPHICS_5` | 5 | 320x200 | 4 | 80 | `$B800` |
| `PCJR_TANDY_GRAPHICS_6` | 6 | 640x200 | 2 | 80 | `$B800` |
| `BITMAP_LORES`, `PCJR_TANDY_GRAPHICS_8` | 8 | 160x200 | 16 | 80 | `$B800` |
| `BITMAP_HIRES`, `PCJR_TANDY_GRAPHICS_9` | 9 | 320x200 | 16 | 160 | `$1800` |
| `PCJR_TANDY_GRAPHICS_10` | `0Ah` | 640x200 | 4 | 160 | `$1800` |

The graphics runtime sets the BIOS video mode and initializes its
current width, height, row byte, and bitmap base metadata.

`PCJR_TANDY_GRAPHICS_*` names are nonportable and only meaningful for the
PCjr/Tandy graphics provider. `PCJR_TANDY_GRAPHICS_8` and
`PCJR_TANDY_GRAPHICS_9` are alternate spellings of `BITMAP_LORES` and
`BITMAP_HIRES`; both select the same mode and report the same
`DISPLAY_MODE`.

If a program changes video modes directly with inline assembly, the
graphics API cannot infer the new dimensions or color count. Prefer
runtime mode selection; target or runtime code has an internal attach
path for cases where the hardware mode was already changed.

Drawing colors are masked to the active mode: 0 through 1 in 2 color
modes, 0 through 3 in 4 color modes, and 0 through 15 in 16 color
modes. Every mode except `PCJR_TANDY_GRAPHICS_10` writes and reads
pixels directly in its page; mode `0Ah` uses BIOS pixel calls. Building
with `--set optimize=size` uses the smaller BIOS calls everywhere. In
high resolution mode, sprite composition mirrors background writes into
its offscreen surfaces when the sprite engine is active.

`CLS` fills the page with the current background color, except in
mode `0Ah` where it resets through the BIOS and so always clears to
black.

Mode `0Ah` is the odd one out: it stores two interleaved bit planes
rather than packed pixels, so a byte pair covers 8 pixels with the even
byte holding bit 0 of each and the odd byte bit 1. The reported byte
shape describes one plane.

## Buffering

`GFX_BUFFERING` is `GFX_BUFFERING_FLIP`. The gate array scans one page of
a pair, so `GFX_SWAP` shows the page just drawn and moves drawing to the
other one. Because the surfaces swap rather than being copied, after a
swap you are drawing into the frame from two swaps ago and have to redraw
what you need.

The second page is claimed from the largest free DOS block on first use.
When it does not fit, `GFX_SWAP` is a no-op and drawing stays on the
visible page, so output is still correct but unbuffered.

`GFX_PAGE_COUNT` stays 1. The pages live in main memory rather than
dedicated video RAM, so reporting a page the program does not own would
invite it to write over DOS.

## Palette

| Capability | Value |
| --- | --- |
| `PALETTE_SUPPORTED` | Yes |
| `PALETTE_SIZE` | 16 |
| `PALETTE_RGB_SUPPORTED` | No |

`PALETTE_SET` remaps one of the 16 displayed colors to any of the
16 hardware colors by programming the video gate array. `PALETTE_RESET`
restores the identity mapping a graphics mode starts with. The palette
selects from a fixed color set, so `PALETTE_SET` with an `RGB` value does
nothing.

## Images

Native image display supports:

| Format | Accepted files |
| --- | --- |
| `IMAGE_FMT_CGA4` | CGA mode 4 BSAVE or raw 16384 byte dumps |
| `IMAGE_FMT_TANDY16` | Mode 9 320x200x16 BSAVE or raw 32768 byte dumps |

`IMAGE_FMT_CGA4` uses B800 dump order. The aux byte uses the CGA
`3D9h` layout, but palette and border changes go through BIOS calls
and the intensity bit is ignored.

Native image display syncs graphics metadata after selecting its BIOS
mode. Images are linked into the program rather than loaded from DOS
files at runtime.

Both formats support wipes and slides in all four directions, four step
dither fades, and dissolves with 1, 2, 4, or 8 pixel square blocks.
Effects that read an image use unpacked image data in memory.

## Sprites

The PCjr has no hardware sprite provider. Sprites are backed by 8x8
opaque software blits for mode 9 graphics.

| Feature | Value |
| --- | --- |
| Sprite count | Selected by `SPRITE_COUNT`, default 6 |
| Size | Runtime `w * 8` by `h * 8` pixels |
| Data bytes | 8 per tile |
| X range | 0 through 312 |
| Y range | 0 through 192 |
| Collision | Not supported |
| Flip, stretch, priority, palette | Not supported |

With software sprite page flipping enabled, startup probes RAM and uses
a CRT page flip when two 32K video pages fit. Otherwise it uses a
software compose buffer and dirty rectangles, or a degraded visible
screen path when only a background buffer fits. Draw bitmap backgrounds
after starting or resetting sprites so the background buffer stays valid.
`SPRITE_DATA_TILES` stores tiles left to right and then top to bottom.

The flip needs a real segment for each page, so startup locates the
displayed page by trying both gate array layouts and confirming the
guess through the `B800` window. When neither matches, the runtime
falls back to the compose path.

## Tile

The PCjr uses the generic tile runtime.

| Capability | Value |
| --- | --- |
| Default backend | `TILE_BITMAP` |
| Native cells | No |
| Bitmap tiles | Yes |
| Tile colors | Yes |
| Tile kernels | No |

Tile definitions, blocks, and colors are available through the generic
runtime.

## Input

| Capability | Value |
| --- | --- |
| Keyboard | Yes |
| Individual held keys | Yes |
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

`KEY_HELD` tracks keyboard make and break events without consuming the
normal BIOS input queue. The tracker installs at program start and
restores the previous keyboard interrupt on normal exit.

Joystick and paddle input use BIOS joystick services. Digital joystick
reads threshold the analog axes with the default deadzone value of 48,
and paddle reads return the raw axis value.

## Sound

The PCjr sound provider uses the SN76496 compatible sound chip at port
`0C0h`.

| Capability | Value |
| --- | --- |
| Voices | 3 |
| Volume max | 15 |
| Note range | 45 through 95 |
| Envelope | No |
| Filter | No |
| Pulse width | No |

Sound supports three voices, simple tones, and note frequency lookup.
The sound chip is heard through the monitor or TV audio output.

Exact chip control uses the chip interface:

```basic
SN76496.SOUND VOICE, DIVISOR, SHAPE, VOLUME
```

| Argument | PCjr meaning |
| --- | --- |
| `VOICE` | Square wave voice `0` through `2` |
| `DIVISOR` | Native pitch divisor `1` through `1023`; `0` silences the voice |
| `SHAPE` | `SOUND_SHAPE_TONE` for a square wave, `SOUND_SHAPE_NOISE` for the noise source |
| `VOLUME` | `0` through `15`; `0` silences the voice |

The chip plays `111861 / DIVISOR` Hz, so a divisor of `254` is close to
440 Hz. The call writes the chip registers and returns. The voice keeps
sounding until another call changes it, `SOUND_OFF VOICE` stops it, or
`SOUND_ALL_OFF` stops every voice and the noise source.

With `SOUND_SHAPE_NOISE` the sound comes from the chip's noise source,
`VOICE` is ignored, and the low three bits of `DIVISOR` pick the noise:

| `DIVISOR` | Noise |
| --- | --- |
| `0`, `1`, `2` | Periodic noise, high to low pitch |
| `3` | Periodic noise at the pitch of voice 2 |
| `4`, `5`, `6` | White noise, high to low pitch |
| `7` | White noise at the pitch of voice 2 |

`PCJR_TANDY_NOISE SOURCE, VOLUME` sets the noise source directly with
the same `SOURCE` values, and `PCJR_TANDY_NOISE_OFF` silences it.

## Timing

Frame waits poll the vertical retrace status bit at `3DAh` with a
timeout. The runtime tick source is the BIOS tick count at `0040:006C`.
The true rate is about 18.2065 Hz, and the midnight reset is compensated
for one crossing. PCjr diskette I/O can lose ticks.
