# crustyBASIC on NES / Famicom

Use `@OPTION TARGET nes` for Nintendo Entertainment System or
Famicom cartridges. The system name is `nes.orig`. Output is an iNES
`.nes` file.

Portable runtime calls are documented in [`../API.md`](../API.md).
This page covers NES modes, cartridge formats, mapper behavior, and
hardware notes.

## Basics

| Item | Value |
| --- | --- |
| CPU | 2A03, 6502 family |
| Output | `.nes` |
| Code start | `$8000` |
| RAM | `$0200-$05FF` compiler data, `$0600-$07FF` target runtime |
| Text | 32x30 tile grid |
| `REAL` | Not supported; use Q8.8 helpers for small fractional math |
| String encoding | ASCII |
| Keyboard | No |

The NES has no operating system. The cartridge provides all input,
graphics, and sound support.

## Text And Graphics

Text uses the PPU nametable as a 32x30 tile grid. `PRINT` and cell
writes queue PPU updates that are drained by NMI. The built-in font and
initial nametable clear are loaded lazily the first time text or cell
output is used, so sound and controller-only programs do not ship the
font table. CHR ROM builds skip the built-in font upload; the supplied
CHR file must contain the glyphs and tiles the program expects.

Cursor positioning clamps out-of-bounds coordinates to the last valid
column or row.

`CELL_GETC` first checks a small cache of recent text and cell writes.
On a miss, it waits for vblank and reads the nametable through the PPU.
There is no full RAM nametable shadow.

Graphics support covers cell/text work. Bitmap-style plotting is not
meaningful on the NES target today. Use tiles, charmap calls, and
sprites for game graphics. The default graphics mode is `CELL`.

| Mode | NES mode | Size | Colors |
| --- | --- | --- | --- |
| `CELL` | PPU nametable cell mode | 32x30 | text |
| `PPU_TEXT` | PPU nametable cell mode | 32x30 | text |
| `BITMAP_HIRES` | not supported | | |
| `BITMAP_LORES` | not supported | | |
| `BITMAP_MULTICOLOR` | not supported | | |
| `CELL_MULTICOLOR` | not supported | | |

Common text escapes:

| Escape | Byte |
| --- | --- |
| `{RETURN}`, `{ENTER}`, `{LF}` | `$0A` |
| `{HOME}` | `$0B` |
| `{CLEAR}`, `{CLR}` | `$0C` |
| `{TAB}` | Four spaces |

## Images

Native image display supports `IMAGE_FMT_NES_SCREEN`: 16 palette bytes,
1024 nametable and attribute bytes, and 4096 pattern table bytes.
The converter accepts a `--chr`, `--nam`, and `--pal` file triplet, and
indexed PNG or PCX sources. Images are linked into the program rather
than loaded from files at runtime. Converted images use RLE8 when it
makes them smaller and otherwise keep the original 5136 byte payload.

CHR ROM builds skip the pattern-table upload; the image must use tile
data already present in the supplied CHR file.

Wipes and slides use 16 by 16 pixel cells and move their palettes with
them. They start with a blank screen and require a blank tile in the
image's pattern data. Fades darken the background palette in four steps.
Wipes, slides, fade in, and dissolve in require unpacked images.

Dissolves support 1, 2, 4, and 8 pixel blocks on CHR RAM builds. They
change tile patterns, so repeated copies of a tile dissolve together.
CHR ROM builds support wipes, slides, and fades, but cannot dissolve
read only tile patterns. Updates wait for video memory access, even
with a delay of zero.

## PPU

The `PPU` namespace exposes the eight CPU-visible PPU registers:

| Name | Address | Purpose |
| --- | --- | --- |
| `PPU.CTRL` | `$2000` | Nametable, increment, sprite mode, NMI enable. |
| `PPU.MASK` | `$2001` | Background/sprite visibility and color bits. |
| `PPU.STATUS` | `$2002` | VBlank, sprite 0 hit, overflow. |
| `PPU.OAMADDR` | `$2003` | OAM byte index. |
| `PPU.OAMDATA` | `$2004` | OAM data. |
| `PPU.SCROLL` | `$2005` | Scroll X, then Y. |
| `PPU.ADDR` | `$2006` | VRAM address high, then low. |
| `PPU.DATA` | `$2007` | VRAM data. |

`PPU.CTRL` and `PPU.MASK` are write-only on hardware. Prefer the
provided sprite, text, and graphics helpers unless you need exact PPU
timing.

## Sprites

The NES has 64 hardware sprites in OAM. Sprite calls use those sprites.

| Feature | Value |
| --- | --- |
| Sprite count | 64 |
| Size | 8x8, or global 8x16 mode |
| Data | Tile index, not a memory pointer |
| Flip | Hardware X and Y flip |
| Palette | 4 sprite palettes |
| Collision | Sprite 0 hit only |

Sprite data selects a tile index. It does not copy bytes from that value
as an address.

`SPRITE_COLOR` shares four palettes between sprites. If a new color
cannot fit, the sprite keeps its current palette; a sprite without a
palette uses palette 0.

Sprite writes update a RAM OAM shadow at `$0700-$07FF`. The NMI handler
copies that shadow page each frame. Call `FRAME_WAIT` after sprite updates
to wait until they are displayed.

## Sound

Sound drives the four primary APU voices: pulse 1, pulse 2, triangle,
and noise. Note helpers use NTSC or PAL timer tables selected by
`REGION`. The DMC channel has separate target-specific helpers:

| Call | Purpose |
| --- | --- |
| `DMC_LOAD ADDR_IDX, LEN_IDX, RATE` | Set up DMC sample playback. |
| `DMC_TRIGGER` | Start DMC playback. |

APU register namespaces are available as `APU.*` and `APU_IO.*`.

## Fixed Point

NES keeps `REAL` unsupported. Use the unsigned Q8.8 helpers for small
fractional math.

Common names:

| Namespace | Useful names |
| --- | --- |
| `APU` | `PULSE1_*`, `PULSE2_*`, `TRI_*`, `NOISE_*`, `DMC_*`, `STATUS`, `FRAME_COUNTER` |
| `APU_IO` | `JOY1`, `JOY2`, `OAM_DMA` |

## Input

| Capability | Value |
| --- | --- |
| Joystick ports | 2 |
| Buttons | A, B, Select, Start |
| Stick type | Digital |
| Keyboard | No |
| Paddle axes | 0 |
| Keypad ports | 0 |
| Keypad `INPUT` | No |
| Mouse buttons | 0 |

Button order is A, B, Select, Start. A full controller read also
includes Up, Down, Left, and Right. Any-input polling checks both
controllers.

Typed `INPUT` is rejected at compile time on NES because the target has
no keyboard text input or line editor.

## Timing

The runtime tick source is the NMI frame counter, one tick per frame.
The true rate is about 60.0988 Hz on NTSC and 50.0070 Hz on PAL.
Timer metadata reports 60 Hz for NTSC builds and 50 Hz for PAL builds.

## Mappers

| Mapper | iNES | Layout |
| --- | ---: | --- |
| `nrom` | 0 | Default. 16K or 32K fixed PRG ROM. |
| `mmc1` | 1 | 16K fixed bank plus up to 15 switched 16K banks. |
| `uxrom` | 2 | 16K fixed bank plus up to 15 switched 16K banks. |
| `mmc3` | 4 | 8K switched PRG banks plus an 8K fixed reset bank. |

Select a banked mapper like this:

```basic
@OPTION TARGET nes
@OPTION MAPPER uxrom
```

By default NES mappers use CHR RAM. Load tile graphics into CHR RAM at
runtime with tile, sprite, or `CHR_UPLOAD`.

To build a cart with CHR ROM, pass a CHR file:

```bash
crustybasic build <input.cbs> --set target=nes --set nes-chr-rom=tiles.chr -o /tmp/program.nes
```

Source can also declare the same build input:

```basic
@OPTION NES_CHR_ROM "tiles.chr"
```

CHR ROM builds package those bytes into the iNES file and skip runtime
pattern-table uploads. Text, tiles, sprites, and native images must use
tile data already present in the supplied CHR file.
`NES_CHR_ROM` supplies the complete CHR image. It may contain multiple
8K CHR units; the cartridge wrapper pads the final unit when needed.

Use `@BANK N` for switched-bank code and `DATA`. `MAIN` stays in the
fixed bank. UxROM, MMC1, and MMC3 share the same `@BANK` behavior. Calls
between switched banks route through the fixed bank automatically.
With MMC3, `@BANK` uses the `$8000-$9FFF` 8K PRG window; the fixed
reset/vector bank lives at `$E000-$FFFF`. MMC3 IRQ and CHR banking
register selection helpers are not exposed yet.

Constant `POKE` statements into a banked mapper's cartridge PRG window
are rejected because that address range is mapper space, not writable
RAM.

See [`../USAGE.md#banked-builds`](../USAGE.md#banked-builds) for the
full `@BANK` workflow.

## Not Supported

| Feature | Reason |
| --- | --- |
| `REAL` | No floating-point ROM. |
| Keyboard input | The base target has only controllers. |
| Bitmap plotting | Graphics are tile and sprite based. |
| `ON ERROR` | Not available on this target. |

## Examples

See [`../../examples/nes/`](../../examples/nes/).
