# crustyBASIC on Tandy 1000

Use `@OPTION TARGET tandy1000` for the default Tandy 1000 profile, or
pick an exact system with `@OPTION SYSTEM tandy1000.exe` or
`@OPTION SYSTEM tandy1000.II`.

Portable runtime calls are documented in [`../API.md`](../API.md).
This page covers Tandy 1000 systems, modes, formats, and hardware
notes. For command line target selection and dialects, see
[`../USAGE.md`](../USAGE.md).

Segment memory access uses `PEEK(SEGMENT, OFFSET)` and
`POKEB SEGMENT, OFFSET, VALUE`. `VIDSEG` is `$B800` for color text
and `BDASEG` is `$0040` for BIOS data. See
[direct memory access](DOS16.md#direct-memory-access) for details.

## Systems

| System | Output | Notes |
| --- | --- | --- |
| `tandy1000.exe` | `.exe`, optional `.img` | Tandy 1000 DOS MZ executable. |
| `tandy1000.II` | `.exe`, optional `.img` | Tandy Video II DOS MZ executable for SL, TL, and RL systems. |

NASM is the supported assembler. The optional `.img` artifact is a
mountable 360K FAT12 disk image when the DOS floppy helper is
configured. Files staged by a two-path `@INCLUDE_*` form go on that image,
uppercased and trimmed to the DOS 8 character name plus 3 character
extension.

For a 720K FAT12 `.img`, use `@OPTION OUTPUT_TYPE disk-720k` or:

```bash
crustybasic build hello.cbs --set target=tandy1000 --set output-type=disk-720k -o hello.img
```

Both systems support this output. The image contains the executable and
companion files. The `.exe` is also kept separately. Boot DOS before running
the program from the image.

## Basics

| Item | Value |
| --- | --- |
| CPU | 8086 real mode |
| Text | 80x25 |
| Code storage | RAM |
| String encoding | ASCII, with non ASCII bytes encoded as `?` |
| Integer math | Built in 16 bit integer support |
| `REAL` math | Built in fixed point, `Q24_8` by default |
| Sound | SN76496 compatible PSG, PIT channel 2, and Video II DAC |
| Host OS | DOS services are available |

Both systems produce DOS MZ executables. There is no Tandy 1000 `.com`
system.

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

The default graphics mode is `BITMAP_HIRES`. Its resolution depends on
the selected system.

### Tandy 1000

| Mode | BIOS mode | Size | Colors | Row bytes | Bitmap base |
| --- | --- | --- | --- | --- | --- |
| `CELL` | 3 | 80x25 | 16 | 0 | text memory |
| `PCJR_TANDY_GRAPHICS_4` | 4 | 320x200 | 4 | 80 | `$B800` |
| `PCJR_TANDY_GRAPHICS_5` | 5 | 320x200 | 4 | 80 | `$B800` |
| `PCJR_TANDY_GRAPHICS_6` | 6 | 640x200 | 2 | 80 | `$B800` |
| `BITMAP_LORES`, `PCJR_TANDY_GRAPHICS_8` | 8 | 160x200 | 16 | 80 | `$B800` |
| `BITMAP_HIRES`, `PCJR_TANDY_GRAPHICS_9` | 9 | 320x200 | 16 | 160 | `$B800` |
| `PCJR_TANDY_GRAPHICS_10` | `0Ah` | 640x200 | 4 | 160 | `$B800` |

This table applies to `tandy1000.exe`.

### Tandy Video II

`tandy1000.II` keeps all of the modes above, with these hires mappings:

| Mode | Hardware mode | Size | Colors | Row bytes | Bitmap base |
| --- | --- | --- | --- | --- | --- |
| `BITMAP_HIRES`, `TANDY_GRAPHICS_640_16` | direct Video II setup | 640x200 | 16 | 320 | `$A000` |
| `PCJR_TANDY_GRAPHICS_9` | BIOS mode 9 | 320x200 | 16 | 160 | `$B800` |

The 640x200x16 mode has no stock BIOS mode number. The
`TANDY_GRAPHICS_640_16` name is a descriptive crustyBASIC target API
name. The TANDY11 BIOS extension called the same hardware setup mode
11, but that number is not part of the Tandy BIOS.

The Video II mode is available on later SL, TL, and RL hardware. It is
selected by programming the video registers directly and uses one
linear 64000 byte bitmap at `$A000`.

The graphics runtime sets the BIOS video mode and initializes its
current width, height, row byte, and bitmap base metadata. The Video II
system restores the normal controller state before selecting a BIOS
mode.

`PCJR_TANDY_GRAPHICS_*` names are nonportable and only meaningful for the
PCjr/Tandy graphics provider. `PCJR_TANDY_GRAPHICS_8` and
`PCJR_TANDY_GRAPHICS_9` are alternate spellings of `BITMAP_LORES` and
`BITMAP_HIRES` on `tandy1000.exe`. On `tandy1000.II`, mode 9 remains the
explicit 320x200x16 mode. `BITMAP_HIRES` and
`TANDY_GRAPHICS_640_16` both select 640x200x16 and report
`BITMAP_HIRES` as `DISPLAY_MODE`.

If a program changes video modes directly with inline assembly, the
graphics API cannot infer the new dimensions or color count. Prefer
runtime mode selection; target or runtime code has an internal attach
path for cases where the hardware mode was already changed.

Drawing colors are masked to the active mode: 0 through 1 in 2 color
modes, 0 through 3 in 4 color modes, and 0 through 15 in 16 color
modes. Every mode except `PCJR_TANDY_GRAPHICS_10` writes and reads
pixels directly in its page; mode `0Ah` uses BIOS pixel calls. Building
with `--set optimize=size` uses the smaller BIOS calls where the BIOS
supports the mode. Video II 640x200x16 drawing always writes the bitmap
directly. In mode 9, sprite composition mirrors background writes into
its offscreen surfaces when the sprite engine is active.

`CLS` fills the page with the current background color, except in
mode `0Ah` where it resets through the BIOS and so always clears to
black. Video II 640x200x16 clears all 64000 bitmap bytes.

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

The Video II 640x200x16 mode uses one 64K bitmap, so `GFX_SWAP` is a
no-op in that mode. BIOS mode 9 keeps the normal Tandy page flipping
behavior.

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
mode. On `tandy1000.II`, `IMAGE_FMT_TANDY16` still selects BIOS mode 9
rather than the 640x200 mode. Images are linked into the program rather
than loaded from DOS files at runtime.

Both formats support wipes and slides in all four directions, four step
dither fades, and dissolves with 1, 2, 4, or 8 pixel square blocks.
Effects that read an image use unpacked image data in memory.

## Sprites

The Tandy 1000 has no hardware sprite provider. Sprites are backed by
8x8 opaque software blits for mode 9 graphics. This remains mode 9 on
`tandy1000.II`; sprites do not switch to the 640x200 mode.

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

The Tandy 1000 uses the generic tile runtime.

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

`KEY_HELD` tracks ordinary make and break codes through the classic
Tandy keyboard latch without consuming BIOS input. The tracker installs
at program start and restores the previous keyboard interrupt on normal exit.
This profile does not cover the later RSX AT keyboard controller.

Joystick and paddle input use BIOS joystick services. Digital joystick
reads threshold the analog axes with the default deadzone value of 48,
and paddle reads return the raw axis value.

## Sound

Portable sound source numbers are zero based:

| Source | `tandy1000.exe` | `tandy1000.II` |
| --- | --- | --- |
| 0 | PSG square wave 1 | PSG square wave 1 |
| 1 | PSG square wave 2 | PSG square wave 2 |
| 2 | PSG square wave 3 | PSG square wave 3 |
| 3 | PSG noise | PSG noise |
| 4 | PIT channel 2 speaker input | PIT channel 2 speaker input |
| 5 | Not available | Video II DAC sound channel |

The PSG uses port `0C0h`. Its three square wave sources and noise source
have independent volume from 0 through 15. Source 4 programs PIT channel
2 and has on or off volume only. The PIT signal passes through the Tandy
sound mixer, so it can play with the PSG sources.

`tandy1000.exe` reports `SOUND_VOICES = 5`.
`tandy1000.II` reports `SOUND_VOICES = 6`.
The note range is 45 through 95 on both systems.

On `tandy1000.II`, source 5 supports `SOUND_SHAPE_PULSE`,
`SOUND_SHAPE_SAW` for the hardware ramp, and
`SOUND_SHAPE_TRIANGLE`. Its 0 through 15 portable volume is mapped to
the DAC's eight hardware levels. The other sources keep their fixed
waveform when used through `PLAY_NOTE_SHAPE`.

Exact chip control uses the same `SN76496.SOUND` call and noise procs
as the PCjr. They are documented under [PCjr Sound](PCJR.md#sound).

## Audio

Prepared sample playback is available on `tandy1000.II`. It is not
available on `tandy1000.exe`.

- `AUDIO_PLAYBACK_MODEL = AUDIO_PLAYBACK_AUTOMATIC` - the BIOS uses DMA
  and playback continues after `AUDIO_PLAY` returns.
- `AUDIO_FILE_SUPPORTED = FALSE` - load embedded prepared assets.
- `AUDIO_TARGET_FILE_SUPPORTED = FALSE` - runtime WAV file loading is not
  provided.
- `AUDIO_SOUND_SHARED = FALSE` - sample playback owns the Video II DAC, so
  do not use generated sound calls until `AUDIO_STOP` or
  `AUDIO_UNLOAD`.

```basic
@INCLUDE_AUDIO VOICE "assets/voice.wav"

OK = AUDIO_LOAD(ADDR VOICE_AUDIO)
OK = AUDIO_PLAY
```

Accepted WAV sources are PCM with one or two channels and 8-bit or
16-bit samples. The converter prepares unsigned 8-bit mono samples:
stereo is mixed to mono, 16-bit samples are reduced to 8-bit, and the
nearest Video II BIOS rate divisor is stored in the CBA metadata. The
source sample rate is otherwise unchanged. Rates outside the DAC
divisor range are rejected during conversion.

Playback uses the Tandy BIOS `int 1Ah` sound service. `AUDIO_PLAY`
starts DMA output at volume 7, `AUDIO_STOP` stops it, and the BIOS
handles buffers that cross a 64K DMA boundary. Playback is automatic.

## Timing

Frame waits poll the vertical retrace status bit at `3DAh` with a
timeout. The runtime tick source is the BIOS tick count at `0040:006C`.
The true rate is about 18.2065 Hz, and the midnight reset is compensated
for one crossing.
