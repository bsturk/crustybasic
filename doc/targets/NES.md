# crustyBASIC on NES / Famicom

Use `@OPTION TARGET nes` for Nintendo Entertainment System or
Famicom cartridges. The system name is `nes.orig`. Output is an iNES
`.nes` file.

For portable calls such as `CLS`, `POSITION`, `WAIT_FRAME`, `SOUND`,
`JOY`, `JOY_BUTTON`, `ANY_INPUT`, and `SPRITE_*`, see
[`../API.md`](../API.md).

## Basics

| Item | Value |
| --- | --- |
| CPU | 2A03, 6502 family |
| Output | `.nes` |
| Code start | `$8000` |
| RAM | `$0200-$05FF` compiler data, `$0600-$07FF` target runtime |
| Text | 32x30 tile grid |
| `REAL` | Not supported; use Q8.8 helpers for small fractional math |
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

`POSITION` clamps out-of-bounds coordinates to the last valid column or
row.

`CELL_GETC` first checks a small cache of recent text and cell writes.
On a miss, it waits for vblank and reads the nametable through the PPU.
There is no full RAM nametable shadow.

The portable graphics calls cover cell/text work. Bitmap-style `PLOT`,
`POINT`, and `LINE` are not meaningful on the NES target today.
Use tiles, charmap calls, and sprites for game graphics.

Common text escapes:

| Escape | Byte |
| --- | --- |
| `{RETURN}`, `{ENTER}`, `{LF}` | `$0A` |
| `{HOME}` | `$0B` |
| `{CLEAR}`, `{CLR}` | `$0C` |
| `{TAB}` | Four spaces |

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

The NES has 64 hardware sprites in OAM. The portable `SPRITE_*` API
uses those sprites.

| Feature | Value |
| --- | --- |
| Sprite count | 64 |
| Size | 8x8, or global 8x16 mode |
| Data | Tile index, not a memory pointer |
| Flip | Hardware X and Y flip |
| Palette | 4 sprite palettes |
| Collision | Sprite 0 hit only |

`SPRITE_DATA ID, TILE` selects a tile index. It does not copy bytes
from `TILE` as an address.

Sprite writes update a RAM OAM shadow at `$0700-$07FF`. `SPRITES_FLUSH`
waits for the next frame; the NMI handler performs OAM DMA from that
shadow page.

## Sound

`SOUND VOICE, FREQ, WAVEFORM, VOLUME` drives the four primary APU
voices: pulse 1, pulse 2, triangle, and noise. The DMC channel has
separate helpers. `SOUND_FREQ` and the portable note helpers use
NTSC or PAL timer tables selected by `REGION`.

| Call | Purpose |
| --- | --- |
| `SOUND VOICE, FREQ, WAVEFORM, VOLUME` | Start one voice. |
| `SOUND_OFF VOICE` | Silence one voice. |
| `SOUND_ALL_OFF` | Silence all voices. |
| `SOUND_FREQ NOTE` | Convert a note number to a timer value. |
| `DMC_LOAD ADDR_IDX, LEN_IDX, RATE` | Set up DMC sample playback. |
| `DMC_TRIGGER` | Start DMC playback. |

APU register namespaces are available as `APU.*` and `APU_IO.*`.

## Fixed Point

NES keeps `REAL` unsupported. Use the core unsigned Q8.8 helpers for
small fractional math: `FX_FROM_INT`, `FX_FROM_PARTS`, `FX_INT`,
`FX_FRAC`, `FXMUL`, and `FXDIV`.

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
| Keyboard | No |
| Paddle axes | 0 |

`JOY_BUTTON(PORT, N)` uses this order: A, B, Select, Start. A full
controller read also includes Up, Down, Left, and Right. `ANY_INPUT`
polls both controllers.

Typed `INPUT` is rejected at compile time on NES because the target has
no keyboard text input or line editor.

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
runtime with helpers such as `TILE_DEFINE`, `SPRITE_DATA_8X8`, or
`NES_CHR_UPLOAD`.

To build a cart with CHR ROM, pass a CHR file:

```bash
crustybasic build program.cbs --set target=nes --set chr-rom=tiles.chr -o /tmp/program.nes
```

Source can also declare the same build input:

```basic
@OPTION NES_CHR_ROM "tiles.chr"
```

CHR ROM builds package those bytes into the iNES file and skip runtime
pattern-table uploads. Text, tiles, sprites, and native images must use
tile data already present in the supplied CHR file.

Use `@BANK PRG N` for switched-bank code and `DATA`. `MAIN` stays in the
fixed bank. With banked NES mappers, switched-bank routines cannot call
directly into another switched bank yet.
With MMC3, `@BANK PRG` uses the `$8000-$9FFF` 8K PRG window; the fixed
reset/vector bank lives at `$E000-$FFFF`. MMC3 IRQ and CHR banking
register selection helpers are not exposed yet.

MMC3 also supports source-declared CHR ROM bank placement:

```basic
@OPTION TARGET nes
@OPTION MAPPER mmc3
@BANK CHR 0 "tiles.chr"
```

Each `@BANK CHR N "path"` file is placed into the 8K CHR ROM bank `N`.
Relative paths are resolved from the source file. Undeclared CHR banks
are filled with `$FF`.

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
