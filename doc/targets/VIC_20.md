# crustyBASIC on Commodore VIC-20

Use `@OPTION TARGET vic20` for the unexpanded VIC-20 profile, or
pick an exact memory expansion with `@OPTION SYSTEM`.

For portable calls such as `CLS`, `POSITION`, `BEEP`, `SOUND`, `JOY`,
`JOY_BUTTON`, `PADDLE`, `CELL_*`, and `SPRITE_*`, see
[`../API.md`](../API.md). For the `cbm_basic_vic20` dialect, see
[`../USAGE.md#dialects`](../USAGE.md#dialects).

## Systems

| System | Output | Notes |
| --- | --- | --- |
| `vic20.orig` | `.prg` | Default unexpanded VIC-20. |
| `vic20.3k` | `.prg` | 3K expansion. BASIC starts at `$0401`. |
| `vic20.8k` | `.prg` | 8K expansion in BLK1. BASIC starts at `$1201`. |
| `vic20.16k` | `.prg` | 16K expansion. |
| `vic20.24k` | `.prg` | 24K expansion. |

All systems use the resident KERNAL and BASIC V2 ROMs. `REAL` is
available through the BASIC ROM.

## Memory And Screen

| Item | `vic20.orig` / `vic20.3k` | `vic20.8k` / `vic20.16k` / `vic20.24k` |
| --- | --- | --- |
| Code start | `$1010` / `$0410` | `$1210` |
| Program RAM top | `$1E00` | `$4000` / `$6000` / `$8000` |
| Text screen | `$1E00` | `$1000` |
| Color RAM | `$9600` | `$9400` |
| Text | 22x23 | 22x23 |

Use `@OPTION PROGRAM_START <addr>` to move the BASIC loader stub.
If `CODE_START` is not set, the machine code starts 15 bytes after
`PROGRAM_START`, matching the target default. Use `@OPTION CODE_START
<addr>` when a custom layout needs a different machine code entry.

`VIC20_SCREEN_BASE` and `VIC20_COLOR_BASE` follow the active system, so
programs can use them instead of hard-coded screen addresses.

## VIC-I

The `VIC` namespace exposes the VIC-I video, sound, paddle, and light
pen registers at `$9000-$900F`.

| Name | Purpose |
| --- | --- |
| `VIC.SCREEN_X`, `VIC.SCREEN_Y` | Screen origin. |
| `VIC.COLUMNS`, `VIC.ROWS` | Text layout and cell height bits. |
| `VIC.RASTER` | Current raster line. |
| `VIC.CHAR_BASE` | Screen and character memory selection. |
| `VIC.POT_X`, `VIC.POT_Y` | Paddle values. |
| `VIC.SOUND1`, `VIC.SOUND2`, `VIC.SOUND3` | Bass, alto, soprano voices. |
| `VIC.NOISE` | Noise voice. |
| `VIC.VOLUME` | Master volume and auxiliary color. |
| `VIC.COLOR` | Background, inverse, and border color. |

```basic
VIC.COLOR = $1B
VIC.VOLUME = 15
VIC.SOUND3 = $E0
```

## Text And Cells

The VIC-20 target focuses on the 22x23 character grid. The portable
text and cell APIs work on every memory expansion.

Strings encode to PETSCII. Common escapes:

| Escape | Byte |
| --- | --- |
| `{RETURN}`, `{ENTER}` | `$0D` |
| `{TAB}` | Four spaces |
| `{CLEAR}`, `{CLR}` | `$93` |
| `{HOME}` | `$13` |
| `{LOWER}`, `{TEXT_LOWER}` | `$0E` |
| `{UPPER}`, `{TEXT_UPPER}`, `{GRAPHICS}` | `$8E` |
| `{LEFT}`, `{RIGHT}`, `{UP}`, `{DOWN}` | Cursor controls |
| `{RVS_ON}`, `{RVS_OFF}` | Reverse video controls |

`CHARSET_INSTALL`, `CHARSET_DEFINE`, and `CHARSET_RESET` are available
for custom 8-byte character glyphs.

## Files

The portable `FILE_OPEN` defaults to disk device 8 and uses the logical
channel as the secondary address. For example, channel 1 maps like
`OPEN 1,8,1,"name,S,R"` for read mode.

Use `FILE_OPEN_NATIVE` when a VIC-20-specific program needs another
device or secondary address:

```basic
@OPTION TARGET vic20

FILE_OPEN_NATIVE 1, "DATA,S,W", 9, 1
FILE_WRITE_STR 1, "HELLO"
FILE_CLOSE 1
```

The lower KERNAL channel helpers are also callable by target-specific
programs: `OPEN_CH`, `CLOSE_CH`, `GET_CH`, `PUT_CH`, `PRINT_CH_STR`,
`ST`, and `CMD_CH`. These are not portable API names.

## Sprites

The VIC-20 has no hardware sprites. The portable `SPRITE_*` API is a
four-slot character-cell sprite layer.

| Feature | Value |
| --- | --- |
| Sprite count | 4 |
| Size | 8x8 cell |
| Movement | Snaps to 8x8 cells |
| Data | One screen character, or one custom 8x8 glyph |
| Collision | Not hardware backed |

`SPRITE_DATA_8X8 ID, ADDR` installs an 8-byte custom glyph for one
sprite slot. Call `VIC20_SPRITE_CHARSET_INSTALL ADDR` first only if
you need to choose a specific 1K-aligned character RAM page.

## Input

| Capability | Value |
| --- | --- |
| Keyboard | Yes |
| Joystick ports | 1 |
| Buttons | 1 |
| Paddle axes | 2 |
| Mouse buttons | 0 |

`JOY(0)` reads the single joystick port. Other ports return
`JOY_NONE`. `JOY_BUTTON(0, 0)` reads the fire button.

`PADDLE(0)` reads `VIC.POT_X` and `PADDLE(1)` reads `VIC.POT_Y`.
`PTRIG` always returns released because there is no separate paddle
trigger line.

## Sound

`SOUND VOICE, FREQ, WAVEFORM, VOLUME` maps to VIC-I sound:

| Voice | Register |
| --- | --- |
| `0` | `VIC.SOUND3`, soprano |
| `1` | `VIC.SOUND2`, alto |
| `2` | `VIC.SOUND1`, bass |
| `3` | `VIC.NOISE` |

`FREQ` is the 7-bit VIC-I pitch value. `WAVEFORM` is ignored because
the voices have fixed shapes. `BEEP(DURATION)` uses the soprano voice.

## Dialect

`cbm_basic_vic20` uses Commodore BASIC V2 suffix rules: bare numeric
names, `!`, and `#` are `REAL`, `%` is 16-bit integer, and `$` is
`STRING`.

Use `RAND(N)` for portable bounded integers. Compatibility `RND`
follows the dialect rules.

## Examples

See [`../../examples/vic20/`](../../examples/vic20/).
