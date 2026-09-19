# crustyBASIC on ZX Spectrum

Use `@OPTION TARGET zxspectrum` for the ZX Spectrum 48K target.

Use `@OPTION DIALECT sinclair_basic` for line-numbered Sinclair BASIC
source. The dialect selects the Spectrum target automatically.

This page covers Spectrum features and Sinclair BASIC compatibility.
Shared runtime calls are documented in the [API reference](../API.md).

## System

| Item | Value |
| --- | --- |
| System | `zxspectrum.48k` |
| CPU | Zilog Z80 |
| Output | `.tap` tape image, or `.tzx` and raw `.bin` |
| Code start | `$8000` |
| Program RAM top | `$FF58` |
| Text area | 32 columns by 22 rows |
| Bitmap | 256 by 192 at `$4000` |
| Attributes | 32 by 24 at `$5800` |
| Frame rate | 50 Hz |
| `REAL` | Sinclair 5-byte format using the 48K ROM |

The default `.tap` image holds a BASIC loader followed by the machine
code, so `LOAD ""` starts the program. The loader lowers RAMTOP to protect
the code while loading, then sets it to one below the program RAM top
(`$FF57` unless `ram-top` is changed) and starts the code at `$8000`.
Select `.tzx` with `@OPTION OUTPUT_TYPE tzx` or `--set output-type=tzx`.

`@OPTION OUTPUT_TYPE raw` or `--set output-type=raw` writes only the
machine code block. A normal tape or disk loader can protect the code
area, load the file, and start it like this:

```basic
CLEAR 32767
LOAD "" CODE 32768
RANDOMIZE USR 32768
```

The program initializes the ROM screen channel, interrupt mode, cursor
state, and border state before entering `MAIN`. Returning from `MAIN`
or using `END` returns to the BASIC caller.

## Text and input

ROM screen output provides `PRINT`, `CLS`, and `POSITION`, using columns
0 through 31 and rows 0 through 21. The lower two screen rows are reserved
by the ROM. Printing past the last row scrolls without the `scroll?` prompt.
`CURSOR_COL` and `CURSOR_ROW` follow the cursor after printing, control
codes, wrapping, scrolling, and Sinclair `AT` or `TAB`.

`CELL_PUTC`, `CELL_PRINT`, and `CELL_DRAW` draw character codes 32 through
127 from the current font without moving the text cursor. `CELL_COLOR`
changes one cell's ink; `CELL_COLORS` sets ink and paper across the text
area. Cell writes preserve existing cell colors and appear immediately.
`CELL_GETC` recognizes these characters, returning 0 for an unrecognized
bitmap.

`CHARSET_COPY_DEFAULT` copies the ROM font into a 768 byte RAM font.
`CHARSET_DEFINE` edits a glyph, and `CHARSET_INSTALL` copies a complete
font with eight bytes per glyph, starting with the space character.
`CHARSET_RESET` restores the ROM font.
Both `PRINT` and cell output use the selected font. Changes affect text
drawn afterwards; existing screen pixels stay unchanged.

In text mode `COLOR ink, paper` sets the ROM's permanent ink and paper,
so later `PRINT` and `CLS` calls use them. Bright and flash keep their
current values.

String literals use Spectrum character codes and accept the `{RETURN}`
and `{HOME}` escapes. The up arrow, pound sign, and copyright sign map
to their Spectrum character positions; unavailable characters become `?`.

Sinclair `PRINT` commas use the ROM's 16-column print zones.
`PRINT #0;` and `PRINT #1;` select the lower screen. `PRINT #2;` selects
the upper screen, which is also the default for each `PRINT` statement.

Portable `INKEY_CODE`, `INKEY`, `GET`, and `INPUT` consume characters
from the ROM `LAST_K` value. As on the real machine, an unshifted letter
key arrives as a lowercase code unless CAPS LOCK is on. Sinclair `INKEY$`
scans the currently held key without consuming it. `KEY_HELD` also reads
the keyboard matrix directly and supports:

- letters, with upper- and lowercase character codes
- digits and space
- Enter
- Caps Shift plus 0 for Delete
- Caps Shift plus 5, 6, 7, or 8 for Left, Down, Up, or Right

Sinclair `INPUT`, including its `LINE` form, reads at the current print
position. Delete erases the previous character. Numeric input accepts
number text, and string input accepts literal text.

### Joysticks

`JOY_PORTS` is 3 and `JOY_BUTTONS` is 1. `JOY(port)` returns the portable
direction flags, including diagonals. `JOY_BUTTON(port, 0)` returns 1 while
fire is pressed and 0 otherwise.

| Port | Interface | Left, right, down, up, fire keys |
|---|---|---|
| 0 | Kempston | Separate joystick input |
| 1 | Sinclair joystick 1 | 6, 7, 8, 9, 0 |
| 2 | Sinclair joystick 2 | 1, 2, 3, 4, 5 |

Use the port matching your attached interface or emulator setting. Port 0
requires a Kempston interface; its readings are undefined without one.
The Sinclair ports also respond to the listed keyboard keys.

`JOY_SELECTION_REQUIRED` is `TRUE`, so `INPUT_ANY` starts with keyboard
input only. After the player selects an interface, use
`INPUT_JOY_ENABLE PORT, TRUE` to include it in `INPUT_ANY` and
`INPUT_CLEAR`. Pass `FALSE` to exclude it again. This does not detect
hardware or change direct `JOY` and `JOY_BUTTON` reads.

## Graphics

`BITMAP_BASE` is `$4000`; `ATTR_BASE` is `$5800`. Each attribute byte
sets ink and paper for one 8 by 8 cell, at `ATTR_BASE + row * 32 + col`.

`DISPLAY BITMAP_HIRES` selects bitmap mode with a top-left origin.

Text and graphics share the bitmap. `CELL_PRINT` can draw text directly
in bitmap mode. `DISPLAY_MIXED BITMAP_HIRES, first_row, rows` selects a
text region within rows 0 through 21. Mixed mode reports a 256 by 176
area so each text row matches eight pixel rows. Draw the graphics first,
then write the text. `CELL_CLS` and `CELL_COLORS` affect only the selected
text rows; `CLS` clears the bitmap. A later `DISPLAY` restores the full
cell area. Use cell calls for mixed text; ROM `PRINT` still uses its
normal cursor and scrolling.

Drawing calls include `PLOT`, `UNPLOT`, `POINT`, `LINE`, `BOX`, `FILLBOX`,
`CIRCLE`, and `CIRCLE_E`.

`TILE_BEGIN 1, 1` divides the bitmap into 32 columns by 24 rows of 8 by 8
pixels, with custom shapes, tile colors, blocks, and tile text.

The Sinclair dialect uses the original graphics coordinate system:

- `PLOT`, `POINT`, `DRAW`, and `CIRCLE` use a bottom-left origin
- X is 0 through 255 and Y is 0 through 175
- `DRAW dx,dy,angle` draws an arc using short line segments

Sinclair `INK`, `PAPER`, `FLASH`, `BRIGHT`, `INVERSE`, and `OVER`
statements change the permanent attributes. The same controls inside
`PRINT`, `PLOT`, `DRAW`, or `CIRCLE` are temporary. Transparent and
contrast color 8 and 9 behavior follows the ROM attribute rules.

`SCREEN$(row,col)` recognizes normal or inverted characters from the
current ROM font. `ATTR(row,col)` returns the attribute byte.

`IMAGE_LOAD(ADDR picture)` displays an embedded screen image, including
ink, paper, bright, and flash attributes. Use `@INCLUDE_IMAGE`
or `cb-image --target zxspectrum` to convert a 6912 byte `.scr` screen
dump. Images use `IMAGE_FMT_ZXSPECTRUM_SCREEN` with unpacked bitmap and
attribute data. Wipes and slides move in 8 by 8 cells, including their
attributes. Fades use normal brightness between the original colors and
black. Dissolves support 1, 2, 4, and 8 pixel blocks and erase to each
cell's paper color.

## Sprites

Software sprites draw on the bitmap and save the pixels and attributes
underneath. There are eight slots by default; use `@OPTION SOFT_SPRITE_COUNT`
to change the count. `SPRITES_ON` resets the slots and selects bitmap mode.

`SPRITE_DATA` and `SPRITE_DATA_8X8` take eight bytes, one per row,
with bit 7 at the left. Set bits draw in the sprite's ink; clear bits
leave the background visible. `SPRITE_DATA_TILES` accepts larger shapes
as rows of these 8 by 8 tiles.

Movement uses pixel coordinates from the top left. Keep the sprite origin
within the bitmap; shapes are clipped at the right and bottom edges.
`SPRITE_COLOR` changes ink while preserving paper, bright, and flash.
Sprites and background pixels in the same 8 by 8 cell share ink. Higher
numbered sprites choose the ink where sprites overlap.

Call `SPRITES_FLUSH` to apply data, position, color, show, and hide changes.
It prepares sprites in a RAM buffer, waits for the next frame, then copies
changed areas to the screen. Call `FRAME_WAIT` after each update; it counts
that wait without waiting again. The buffer uses about 7 KB of RAM in
addition to saved backgrounds.

`SPRITES_OFF` hides sprites and restores their backgrounds immediately;
`SPRITES_RESET` also frees the saved backgrounds and resets the slots.
Hide sprites before changing the background beneath them. `CLS` discards
saved backgrounds so later moves cannot restore an old screen.

## Timing

`FRAME_WAIT` waits for the next frame interrupt. `FRAME_WAIT count` and
`FRAME_DELAY count` wait for that many frames.

`TICKS` reads the ROM frame counter atomically. It runs at `TICKS_HZ = 50`
and wraps after 24 bits.
`TICKS_RESET` sets a new zero point without changing the ROM counter.

Sinclair `PAUSE frames` uses the same frame source. A new key press ends
the pause early, and `PAUSE 0` waits for one new key press. A key held when
the pause starts must be released first.

## Sound

The Spectrum has one blocking 1-bit beeper voice.

| Call | Meaning |
| --- | --- |
| `BEEP cycles` | Play a fixed C4 tone |
| `BEEP period, cycles` | Call the ROM beeper with native values |
| `PLAY_NOTE` | Play a short MIDI note from 36 through 95 |
| `PLAY_NOTE_FOR` | Play a MIDI note for `duration` ticks (1/50 s) |

`SOUND_OFF` has no effect because a tone has already finished when a
blocking call returns.

Sinclair `BEEP duration,pitch` takes seconds and semitones relative to
middle C, including fractional and negative values.

## Files

The FILE API uses the 48K ROM cassette routines. Files are standard tape
CODE blocks with names of up to ten characters; longer names are truncated.
An empty read name accepts the next CODE file. Names are case sensitive.

`@INCLUDE_FILE` adds files after the program in `.tap` and `.tzx` images.

One file can be open at a time. All channel numbers use the same stream.
`FILE_MODE_READ` loads the file when opened. `FILE_MODE_WRITE` buffers the
data and records it when `FILE_CLOSE` is called. Each file can hold up to
`ZXSPECTRUM_FILE_CAPACITY` bytes (4096). Text lines end with carriage return.

Start recording before closing an output file. Before opening an input
file, position the tape before it and press Play; for included files,
leave the program tape playing. Rewind before reading back a file you
just recorded. In an emulator, use its cassette controls. The calls wait
for the tape; hold Space to abort an active transfer.

Check `FILE_STATUS` after opening and closing. It returns `FILE_OK`,
`FILE_EOF`, or one of these errors:

| Status | Meaning |
| --- | --- |
| `ZXSPECTRUM_FILE_IO_ERROR` | Tape data could not be read or its checksum failed |
| `ZXSPECTRUM_FILE_BREAK` | Tape transfer was stopped with Space |
| `ZXSPECTRUM_FILE_TOO_LARGE` | File exceeds the buffer capacity |
| `FILE_UNSUPPORTED` | Mode or command is unavailable |

`FILE_OPEN_NATIVE` uses `AUX1` as the portable mode and ignores `AUX2`.

## Sinclair BASIC dialect

The dialect uses 1-based arrays and string positions, dynamic strings,
case-insensitive identifiers, preserved case in strings, and `REAL`
as the default numeric type.

Additional dialect forms include:

- `LET`, `GO TO`, `GO SUB`, `RUN`, `STOP`, and computed branches
- `DEF FN` and numeric or string `FN` calls
- `FOR`, `NEXT`, `IF`, `READ`, `DATA`, and `RESTORE`
- `RND` and `RANDOMIZE`
- `IN port` and `OUT port,value`
- `INT`, `ABS`, `SGN`, `SQR`, `SIN`, `COS`, `TAN`, `ASN`, `ACS`, `ATN`,
  `EXP`, `LN`, `PI`, and `^`
- `CHR$`, `STR$`, `CODE`, `LEN`, string slicing, and numeric `VAL`
- numeric `USR address`, returning BC, and upper- or lowercase
  `USR "a"` style UDG addresses

`RUN` branches to the first line or the requested line. Unlike ROM BASIC,
it does not clear variables or the screen.

`STR$` uses the ROM's normal number format. `VAL` accepts numeric text
with optional leading spaces, a sign, a decimal part, and an `E`
exponent.

## References

- [ZX Spectrum BASIC Programming manual](https://worldofspectrum.org/ZXBasicManual/)
- [Complete Spectrum ROM disassembly](https://skoolkid.github.io/rom/)
