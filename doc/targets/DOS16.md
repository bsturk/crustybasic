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
| `dos16.com` | `.com` | Generic 16 bit DOS COM program, loaded at `$0100`. |

NASM is the supported assembler. The optional `.img` artifact is a
mountable 360K FAT12 disk image when the DOS floppy helper is
configured.

## Basics

| Item | Value |
| --- | --- |
| CPU | 8086 real mode |
| Text | 80x25 |
| Code storage | RAM |
| String encoding | ASCII, with non ASCII bytes encoded as `?` |
| Newline byte | `$0D` |
| Integer math | Built in 16 bit integer support |
| `REAL` math | Not supported currently |
| Elapsed timer | Not available |
| Host OS | DOS services are available |

All current DOS systems use a 256 byte string buffer limit.

## Text And Cells

Clearing, positioning, and cursor visibility use BIOS text services.
Cell display selects BIOS mode 3, 80x25 text with 16 colors.

| Target | Cell color | Cell attributes |
| --- | --- | --- |
| `dos16` | Yes | No |

The shared color constants use the DOS/BIOS 0 through 15 palette.
`ORANGE` maps to `BROWN`, and `GRAY` maps to `LIGHT_GRAY`.

## Graphics

The default graphics mode is `BITMAP_HIRES`.

| Target | Mode | Size | Colors | Row bytes | Bitmap base |
| --- | --- | --- | --- | --- | --- |
| `dos16` | VGA mode 13h | 320x200 | 256 | 320 | `$A000` |

| Mode | DOS mode | Size | Colors |
| --- | --- | --- | --- |
| `TEXT_40X25` | BIOS mode 1 text | 40x25 | 16 |
| `CELL`, `TEXT_80X25` | BIOS mode 3 text | 80x25 | 16 |
| `TEXT_80X50` | VGA mode 3 text, 8x8 font | 80x50 | 16 |
| `BITMAP_HIRES` | VGA mode 13h | 320x200 | 256 |
| `BITMAP_LORES` | not supported | | |
| `BITMAP_MULTICOLOR` | not supported | | |
| `CELL_MULTICOLOR` | not supported | | |

## Images

Native image display supports:

| Target | Format | Accepted files |
| --- | --- | --- |
| `dos16` | `IMAGE_FMT_CGA4` | CGA mode 4 BSAVE or raw 16384 byte dumps |

`IMAGE_FMT_CGA4` uses B800 dump order. On generic DOS the aux byte is
the raw CGA color select value for port `3D9h`.

Images are linked into the program rather than loaded from DOS files at
runtime.

## Sprites

The DOS16 target does not have hardware sprites. Sprites are backed by
8x8 opaque software blits.

| Feature | Value |
| --- | --- |
| Sprite count | 6 |
| Size | 8x8 |
| Data bytes | 8 |
| Surface | Pixel |
| Collision | Not supported |
| Flip, stretch, priority, palette | Not supported |

On `dos16`, sprites compose into off screen buffers when page flip
support is enabled, then copy touched 8x8 rectangles after the frame
wait.

## Tile

The DOS16 target uses the generic tile runtime.

| Target | Default backend | Cell tiles | Bitmap tiles | Tile colors |
| --- | --- | --- | --- | --- |
| `dos16` | `TILE_CELL` | Yes | Yes | Yes |

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

## Timing

Frame waits poll the vertical retrace status bit with a timeout. The
runtime tick source is the BIOS tick count at `0040:006C`. The true
rate is about 18.2065 Hz, and the midnight reset is compensated for one
crossing.

## Files

The DOS file provider uses DOS `int 21h` file handles. Read, write,
append, and update modes are supported. Directory mode and command
channels are not supported. Native opens use `AUX1` as the mode value
and ignore `AUX2`.

## Examples

MS-DOS specific examples live under:

- [`../../examples/dos16/`](../../examples/dos16/)
