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
| RAM | `$0200-$07FF` for data, `$0100-$01FF` CPU stack |
| Text | 32x30 tile grid |
| `REAL` | Not supported |
| Keyboard | No |

The NES has no operating system. The cartridge provides all input,
graphics, and sound support.

## Text And Graphics

Text uses the PPU nametable as a 32x30 tile grid. `PRINT`, `CLS`, and
`POSITION` write through the PPU during VBlank-safe windows.

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

## Sound

`SOUND VOICE, FREQ, WAVEFORM, VOLUME` drives the four primary APU
voices: pulse 1, pulse 2, triangle, and noise. The DMC channel has
separate helpers.

| Call | Purpose |
| --- | --- |
| `SOUND VOICE, FREQ, WAVEFORM, VOLUME` | Start one voice. |
| `SOUND_OFF VOICE` | Silence one voice. |
| `SOUND_ALL_OFF` | Silence all voices. |
| `SOUND_FREQ NOTE` | Convert a note number to a timer value. |
| `DMC_LOAD ADDR_IDX, LEN_IDX, RATE` | Set up DMC sample playback. |
| `DMC_TRIGGER` | Start DMC playback. |

APU register namespaces are available as `APU.*` and `APU_IO.*`.

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

## Mappers

| Mapper | iNES | Layout |
| --- | ---: | --- |
| `nrom` | 0 | Default. 16K or 32K fixed PRG ROM. |
| `uxrom` | 2 | 16K fixed bank plus 16 switched 16K banks. |

Select UxROM like this:

```basic
@OPTION TARGET nes
@OPTION MAPPER uxrom
```

Both mappers use CHR RAM. Load tile graphics into CHR RAM at runtime.

Use `@BANK N` for switched-bank code and `DATA`. `MAIN` stays in the
fixed bank. With UxROM, switched-bank routines cannot call directly
into another switched bank yet, and `@CHR_BANK` is reserved.

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
