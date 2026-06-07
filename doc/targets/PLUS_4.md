# Commodore Plus/4

Use `@OPTION TARGET plus4` for the default Commodore Plus/4 profile.
It builds a BASIC-loaded `.prg`.

| System | CPU | Output | Text | Host ROM | Default assembler |
| ------ | --- | ------ | ---- | -------- | ----------------- |
| `plus4.orig` | 6502 | `.prg` | 40x25 | yes | `vasm` |

Supported today:

- KERNAL text output, keyboard input, file channels, `CLS`, `POSITION`
- TED `BEEP`, `SOUND`, and portable `PLAY_NOTE`
- TED chip register namespace as `TED.*`
- two digital joystick ports through `JOY` and `JOY_BUTTON`
- BASIC 3.5 ROM floating point routines
- direct text charmap helpers for the 40x25 text screen
- portable `DISPLAY`, `GFX_COLOR`, `PLOT`, `LINE`, `BOX`, `CIRCLE`, and
  related bitmap graphics calls

## Files

The portable `FILE_OPEN` defaults to disk device 8 and uses the logical
channel as the secondary address. For example, channel 1 maps like
`OPEN 1,8,1,"name,S,R"` for read mode.

Use `FILE_OPEN_NATIVE` when a Plus/4-specific program needs another
device or secondary address:

```basic
@OPTION TARGET plus4

FILE_OPEN_NATIVE 1, "DATA,S,W", 9, 1
FILE_WRITE_STR 1, "HELLO"
FILE_CLOSE 1
```

The lower KERNAL channel helpers are also callable by target-specific
programs: `OPEN_CH`, `CLOSE_CH`, `GET_CH`, `PUT_CH`, `PRINT_CH_STR`,
`ST`, and `CMD_CH`. These are not portable API names.

`cbm_basic_v3_5` targets `plus4` by default.
