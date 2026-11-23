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
| Random | Seeded from the frame count at first use |
| Recursive PROCs | Yes |

The NES has no operating system. The cartridge provides all input,
graphics, and sound support.

## Text And Graphics

`DISPLAY_HIDE` hides the display without clearing it. `DISPLAY_SHOW`
reveals the current contents. `DISPLAY_VISIBILITY_SUPPORTED` is `TRUE`.
Background and sprite visibility settings are preserved.

Text uses the PPU nametable as a 32x30 tile grid. `PRINT` and cell
writes queue PPU updates that are drained by NMI. The built-in font and
initial nametable clear are loaded lazily the first time text or cell
output is used, so sound and controller-only programs do not ship the
font table. Built-in text maps lowercase to uppercase glyphs. CHR ROM
builds use the glyphs and tiles in the supplied CHR file. New lines at
the bottom scroll text upward.

Cursor positioning clamps out-of-bounds coordinates to the last valid
column or row.

`CELL_GETC` first checks a small cache of recent text and cell writes.
On a miss, it waits for vblank and reads the nametable through the PPU.
The fetched value is cached for later reads.

`CLS`, `CELL_CLS`, `CELL_MEMMOVE`, and `CELL_SCROLL` work on the nametable
with rendering off, so the screen blanks until the next frame.
`CELL_COLOR` and `CELL_COLORS` use a background palette whose first
foreground color already matches the request, so palettes set with
`TILE_PALETTE` stay intact. Otherwise they install the requested
foreground. The four-color form of `CELL_COLORS` also sets the backdrop
and the other two foreground colors.

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

## Tile assets

Native `.cbt`, PNG, and CharPad tiles convert to `nes` output by
default. Each 8x8 piece stores 16 bytes, low plane followed by high plane.
The importer uses four background palettes with a shared backdrop and up to
three foreground colors per palette. Palette areas cover 16x16 pixels; align
logical tiles to that grid when adjacent tiles use different palettes.
`FORMAT mono` produces single color shapes. Imports use patterns `96`..`127`
and `192`..`255`, leaving `128`..`191` for hardware sprite shapes. With
software sprite slots enabled, imports use `96`..`127` and software shapes
use `192`..`255`.

## Images

Native image display supports `IMAGE_FMT_NES_SCREEN`: 16 palette bytes,
1024 nametable and attribute bytes, and 4096 pattern table bytes.
The converter accepts a `--chr`, `--nam`, and `--pal` file triplet, and
PNG or PCX sources. Images are linked into the program rather
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

`PPU_SCROLL X, Y` sets the pixel offsets used by subsequent frame refreshes.
Use `PPU_SCROLL 0, 0` to restore the normal text position.

## Scrolling

`SCROLL_SETUP(SCROLL_TYPE_HW)` selects hardware scrolling for `CELL`
and `PPU_TEXT`. `SCROLL_BEGIN AXES` starts at the top left of the current
screen. It briefly blanks the display to prepare screen memory.
Both scroll granularities are one pixel.

`SCROLL_MOVE DX, DY` accepts signed pixel steps. Move one axis per call
and cross at most one 8x8 cell boundary. Fill the returned edge with
`SCROLL_CELL_PUTC` or `SCROLL_CELL_COLUMN`, then call `SCROLL_PRESENT`.
Presentation waits for queued writes and applies the position at vblank.

The backing grid is 32x30 cells. An enabled horizontal axis reports a
31-column view; an enabled vertical axis reports a 29-row view. The
remaining column or row is the incoming edge and can appear at the
screen border. Read the `SCROLL_BUFFER_*` and `SCROLL_VIEW_*` values
for the active dimensions.

Tile patterns and palettes move together. The foreground argument to
`SCROLL_CELL_PUTC` selects the same palette as `CELL_COLOR`. Each palette
covers a 2x2 group of tiles, so changing an incoming tile's palette also
changes its neighbors in that group. Hardware sprite positions stay
fixed. Scrolling works with CHR RAM or CHR ROM.

`SCROLL_CLOSE` retains the displayed position and releases the scrolling
state. `PPU_SCROLL 0, 0` returns to the physical tile map's top left.

## Sprites

The NES has 64 hardware sprites in OAM, using IDs `0`..`63`.

| Feature | Value |
| --- | --- |
| Sprite count | 64 |
| Size | 8x8, or global 8x16 mode |
| Data | Tile index, not a memory pointer |
| Flip | Hardware X and Y flip |
| Palette | 4 sprite palettes |
| Collision | Bounding boxes and sprite 0 background hit |

Sprite data selects a tile index. It does not copy bytes from that value
as an address.

`SPRITE_DATA_8X8` uploads an eight byte shape. In native 8x16 mode, it
uses the upper tile and clears the lower tile. `SPRITE_EXPAND ID, 0, 1`
doubles all installed 8x8 shapes vertically in CHR RAM; zero restores
8x8 mode. Expansion keeps copies in table 0 patterns `128`..`191` and
uses table 1 patterns `128`..`255`, with `0`..`63` as temporary storage.
`SPRITES_8X16` selects the native format separately.

`SPRITE_DATA_TILES` uploads a grid using consecutive hardware IDs starting
at `ID`; reserve `W * H` slots. Move, show, hide, flip, palette, and
priority calls on the first ID apply to the whole shape. The grid uses
one hardware slot per 8x8 tile and counts toward the scanline sprite limit.

`SPRITE_COLOR` shares four palettes between sprites. If a new color
cannot fit, the sprite keeps its current palette; a sprite without a
palette uses palette 0.

Sprite writes update a RAM OAM shadow at `$0700-$07FF`. The NMI handler
copies completed sprite updates each frame. Call `SPRITES_FLUSH` after
sprite changes to display them together at the next frame.

## Software Sprites

The NES supports hardware and software sprites together through the
[shared SPRITE API](../API.md#hardware-and-software-sprites).

`@OPTION SPRITE_COUNT 66` adds software IDs `64` and `65` after the
hardware slots. `SPRITE_HW_COUNT` is `64`; `SPRITE_MAX_COUNT` reports the
total. `SPRITE_KIND` is `SPRITE_KIND_MIXED` when software IDs are allocated.
Software sprites use CHR RAM and background tiles in `CELL` or `PPU_TEXT`.

The same `SPRITE_*` calls handle both kinds. `SPRITE_DATA ID, TILE` copies
an 8x8 tile from the hardware sprite pattern table. For software IDs,
`SPRITE_DATA ID, WIDTH, HEIGHT, ADDR SHAPE` accepts pixel dimensions and
one bit per pixel, highest bit first, with rows padded to whole bytes.
`SPRITE_DATA_8X8` takes eight bytes. `SPRITE_DATA_TILES` takes consecutive
8x8 one bit tiles in row order. `SPRITE_WIDTH` and `SPRITE_HEIGHT` describe
hardware sprites.

Software sprites support horizontal and vertical flips.
Software positions refer to the first nametable and move by pixels.
Use positive dimensions and keep the whole shape inside the 256x240 area.
Visible software shapes share 64 generated tiles, using one per covered
cell. They occupy tiles `192`..`255` in the background pattern table;
keep other artwork below `192` when it uses the same table.

`SPRITE_COLOR` selects a background palette and sets its first foreground
color. `SPRITE_PALETTE` selects background palette `0`..`3` for software IDs.
With software slots enabled, `SPRITE_PALETTE_SET` updates the matching
background and hardware sprite palettes. Background palettes are shared
across 16x16 pixel regions.

`SPRITES_FLUSH` restores old backgrounds and draws visible software shapes
in ID order. Software refresh blanks rendering while background tiles
change. `SPRITES_RESTORE` restores backgrounds before other drawing.
`SPRITE_HIT` compares visible sprite rectangles across both kinds.

Variable IDs select hardware or software at each call. Speed builds can
resolve constant IDs directly. A known `BIND PROC` selects a direct handler
for variable IDs too, using the same global IDs:

```basic
BIND PROC SPRITE_MOVE_TO TO __cb_nes_sprite_hw_move
SPRITE_MOVE_TO ID, X, Y
BIND PROC SPRITE_MOVE_TO TO __cb_nes_sprite_sw_move
SPRITE_MOVE_TO ID, X, Y
BIND PROC SPRITE_MOVE_TO TO SPRITE_MOVE_TO
```

## Sound

Sound drives the four primary APU voices: pulse 1, pulse 2, triangle,
and noise. Note helpers use NTSC or PAL timer tables selected by
`REGION`. The DMC channel has separate target-specific helpers:

| Call | Purpose |
| --- | --- |
| `DMC_LOAD ADDR_IDX, LEN_IDX, RATE` | Set up DMC sample playback. |
| `DMC_TRIGGER` | Start DMC playback. |

`BEEP` plays a C5 tone on pulse 1.

APU register namespaces are available as `APU.*` and `APU_IO.*`.

Timed notes on four voices advance after `FRAME_WAIT`. They continue
while the main program updates its game state. Attack, decay, sustain
and release envelopes also advance after `FRAME_WAIT`. Triangle levels
gate its fixed amplitude on and off.

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
controllers. `JOY` and `JOY_BUTTON` share one sample per port each frame.

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
runtime with tile, sprite, or `CHR_UPLOAD`. `TILE_DEFINE` and
`SPRITE_DATA_8X8` update their tile during the next frame without
blanking the screen. Rendering is off while `CHR_UPLOAD` copies, so a
large upload blanks the rest of that frame. `CHR_UPLOAD` accepts pattern
indices `0..511` across both pattern tables.

`CHARSET_DEFINE` and `CHARSET_INSTALL_RANGE` install 8 byte monochrome
glyphs in CHR RAM. Each glyph uses the first foreground color.

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
`NES_CHR_ROM` supplies the complete CHR image. NROM holds one 8K CHR
unit. Banked CHR mappers may hold multiple units; the cartridge wrapper
pads the final unit when needed.

Use `@BANK N` for switched-bank code and `DATA`. `MAIN` stays in the
fixed bank. UxROM, MMC1, and MMC3 share the same `@BANK` behavior. Calls
between switched banks route through the fixed bank automatically.
With MMC3, `@BANK` uses the `$8000-$9FFF` 8K PRG window; the fixed
reset/vector bank lives at `$E000-$FFFF`. Banked cartridges pad the PRG
ROM to a power of two bank count and keep the fixed bank last.

Constant `POKE` statements into a banked mapper's cartridge PRG window
are rejected because that address range is mapper space, not writable
RAM.

See [`../USAGE.md#banked-builds`](../USAGE.md#banked-builds) for the
full `@BANK` workflow.

## Raster interrupts

MMC3 cartridges provide the shared raster interface. `RASTER_MARK_ROW`
marks an 8 pixel row from `0`..`29`; the two argument
`RASTER_LINE_INSTALL` selects a pixel line. Row zero runs during vertical blank, ready for the top of the
picture. `RASTER_MARK_ROWS` marks several rows in display order. The
line callback runs at each marked row and the frame callback runs during
vertical blank. `RASTER_CLEAR` removes all line marks,
`RASTER_STOP` pauses callbacks, and `RASTER_UNINSTALL` removes both hooks.

Use background pattern table 0 and sprite pattern table 1. In 8x16 mode,
all sprite tile indices must select table 1. Keep callback work short;
PPU updates that blank rendering also pause line callbacks.

## Prepared samples

`@INCLUDE_AUDIO` converts WAV files to aligned DMC assets. Load the
generated `NAME_AUDIO` with `AUDIO_LOAD ADDR NAME_AUDIO`, then use
`AUDIO_PLAY`, `AUDIO_STOP`, and `AUDIO_UNLOAD`. Playback runs independently
and leaves the four note voices available. Stop retains the sample for
replay; unload releases it.

DMC payloads occupy fixed ROM at `$C000` or above. For NROM, select
`@OPTION START_CODE $C000`; banked cartridges use their fixed window.
Keep prepared assets outside switched bank blocks. Samples contain at
most 4081 bytes and use the selected NTSC or PAL playback rates.
