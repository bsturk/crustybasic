# crustyBASIC on ZX Spectrum

Use `@OPTION TARGET zxspectrum` for the ZX Spectrum 48K target.
Use `@OPTION SYSTEM zxspectrum.128k` for the Spectrum 128.

Use `@OPTION DIALECT sinclair_basic` for line-numbered Sinclair BASIC
source. The dialect selects the Spectrum target automatically.

Single program `.tap` and `.tzx` files can be compiled directly. The
Sinclair decoder reads the BASIC program and preserves its saved starting
line. `detokenize` also keeps this setting in the extracted source.

```bash
crustybasic compile hello.tzx -o hello.s
crustybasic detokenize hello.tap -o hello.bas
```

This page covers Spectrum features and Sinclair BASIC compatibility.
Shared runtime calls are documented in the [API reference](../API.md).

## System

| Item | Value |
| --- | --- |
| System | `zxspectrum.48k` or `zxspectrum.128k` |
| CPU | Zilog Z80 |
| Output | `.tap` tape image, or `.tzx` and raw `.bin` |
| Code start | `$8000` |
| Program RAM top | `$FF58` on 48K, `$BFE0` on 128K |
| Text area | 32 columns by 22 rows |
| Bitmap | 256 by 192 at `$4000` |
| Attributes | 32 by 24 at `$5800` |
| Frame rate | 50 Hz, or 60 Hz with `region = "REGION_NTSC"` |
| `REAL` | Sinclair 5-byte format using the 48K ROM |

The default `.tap` image holds a BASIC loader followed by the machine
code, so `LOAD ""` starts the program. The loader lowers RAMTOP to protect
the code while loading, then sets it to one below the program RAM top
(`$FF57` on 48K, `$BFDF` on 128K) and starts the code at `$8000`.
Select `.tzx` with `@OPTION OUTPUT_TYPE tzx` or `--set output-type=tzx`.

On the 128, load through the Tape Loader menu or 128 BASIC to keep RAM
paging available. Code, ordinary variables, and the program stack stay
in fixed RAM at `$8000-$BFDF`. The manifest reserves the screen, system
workspace, and `$C000-$FFFF` paging window separately.

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

`@OPTION NOBASIC TRUE` or `--set nobasic=true` reclaims the Sinclair BASIC
workspace at `$6000-$7FFF` for writable data on both 48K and 128K. The program
initializes this area at startup and keeps the heap above the loaded code.
The ROM workspace below `$6000` stays reserved.
When using a custom loader, keep the BASIC caller below `$6000`.

### 128K memory

The 128K has eight physical 16 KB RAM banks.
The EXTMEM API provides five 16 KB data banks, for 80 KB total.
EXTMEM bank numbers 0 through 4 map to physical banks 0, 1, 3, 4, and 6.
Transfers cross bank boundaries and restore the previous paging value.
Bank 2 holds the program; bank 5 holds the primary screen and system
workspace; bank 7 holds the shadow screen.

`@BANK n` selects physical bank `n`. Place PROCs, CONST arrays, and DATA in
banks 0, 1, 3, 4, or 6; banks 2, 5, and 7 are reserved as described above.
Bank requests enable RAM banking automatically; `@OPTION MAPPER ram`
selects it explicitly.
TAP and TZX output load the banks before starting MAIN. Calls select the
callee's bank and restore the caller's bank on return. MAIN, shared
runtime routines, ordinary variables, and the stack stay in fixed RAM.
The mapper owns all five data banks; `EXTMEM_AVAILABLE()` returns `0`.

`RAM_PAGE bank` maps a physical bank from 0 through 7 at
`$C000`. `RAM_PAGE()` returns the mapped bank.
`ZXSPECTRUM_RAM_WINDOW` and `ZXSPECTRUM_RAM_BANK_BYTES` give the window
address and bank size. Restore the shadow bank before drawing to page 1
after changing the mapping directly. Run manual page switching and shadow
screen drawing from fixed code, restoring the caller's page before returning
to banked code.

## Text and input

ROM screen output provides `PRINT`, `CLS`, and `POSITION`, using columns
0 through 31 and rows 0 through 21. The lower two screen rows are reserved
by the ROM. Printing past the last row scrolls without the `scroll?` prompt.
`CURSOR_COL` and `CURSOR_ROW` follow the cursor after printing, control
codes, wrapping, scrolling, and Sinclair `AT` or `TAB`.

`CELL_PUTC`, `CELL_PRINT`, and `CELL_DRAW` draw character codes 32 through
127 from the current font, mosaic characters 128 through 143, and custom
characters 144 through 164 without
moving the text cursor. `CELL_COLOR` changes one cell's ink;
`CELL_COLORS` sets ink and paper across the text area. Cell writes preserve
existing cell colors and appear immediately.
`CELL_GETC` recognizes these characters, returning 0 for an unrecognized
bitmap.

The scroll API moves a cell screen or rectangular cell region in software.
Movement has one pixel granularity on both axes. Use `SCROLL_MOVE(1, 0)`
for a smooth horizontal step or `SCROLL_MOVE(8, 0)` for a character step.
Cells use the current font and ink and paper colors. Color attributes move
at character boundaries. `SCROLL_PRESENT` paces movement to the display.
`--set optimize=speed` uses larger, faster bitmap copy and shift routines.

`@OPTION ZXSPECTRUM_SCROLL_COLUMNS` and `@OPTION ZXSPECTRUM_SCROLL_ROWS`
set the largest scroll region in cells, defaulting to 32 by 24. Keep each
region within those limits. Smaller limits reduce the backing storage to
two bytes per cell, including one extra row and column.

`CHARSET_COPY_DEFAULT` copies the ROM font into a 768 byte RAM font.
`CHARSET_DEFINE` edits font glyphs 32 through 127 or custom glyphs 144
through 164. Custom glyphs use the ROM's user defined character area.
Cell output draws all 21 custom glyphs. ROM `PRINT` draws codes 144 through
164 in 48K mode and 144 through 162 in 128K BASIC mode.
`CHARSET_INSTALL` copies a complete font with eight bytes per glyph,
starting with the space character.
`CHARSET_RESET` restores the ROM font.
Both `PRINT` and cell output use the selected font. Changes affect text
drawn afterwards; existing screen pixels stay unchanged.

In text mode `COLOR ink, paper` sets the ROM's permanent ink and paper,
so later `PRINT` and `CLS` calls use them. Bright and flash keep their
current values.

`BACKGROUND_COLOR color` changes paper across the drawing page, preserving
each cell's ink, bright, and flash. It also sets paper for later text,
graphics, and clear calls.

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
- Symbol Shift punctuation combinations
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
`JOY_DEFAULT_PORT` is 0, selecting Kempston.

`JOY_SELECTION_REQUIRED` is `TRUE`, so joystick ports start disabled.
After the player selects an interface, use `INPUT_JOY_ENABLE PORT, TRUE`
to enable its reads. Pass `FALSE` to disable it again. Disabled ports return
zero from `JOY` and `JOY_BUTTON`, including reads through `INPUT_ANY`,
`INPUT_CLEAR`, and `INPUT_JOY_KEYMAP`. Mapped keyboard keys remain active.
Enabling a port does not detect or attach hardware.

## Graphics

`BITMAP_BASE` is `$4000`; `ATTR_BASE` is `$5800`. Each attribute byte
sets ink and paper for one 8 by 8 cell, at `ATTR_BASE + row * 32 + col`.

`DISPLAY BITMAP_HIRES` selects bitmap mode with a top-left origin.

The 128 has two screen pages. `GFX_PAGE_DRAW 0` draws to the primary
screen; `GFX_PAGE_DRAW 1` draws to the shadow screen. `GFX_PAGE_SELECT`
chooses which screen is visible independently of drawing. Use
`GFX_PAGE_BASE` and `GFX_PAGE_ATTR_BASE` for their mapped addresses.
Bitmap, cell, tile, image, sprite, and scroll calls use the selected
drawing page. ROM `PRINT` uses the primary screen. Selecting text mode
restores primary drawing and display.
Hide sprites and flush them before changing drawing pages.

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
or `cb-image --target zxspectrum` to convert PNG, PCX, or a 6912 byte
`.scr` screen dump. Pixel conversion chooses two colors and a shared
brightness for each 8 by 8 cell. Images use `IMAGE_FMT_ZXSPECTRUM_SCREEN`
with bitmap and attribute data. Screen images use RLE when it reduces
size. Wipes and slides move in 8 by 8 cells, including their attributes.
Fades use normal brightness between the original colors and black. Dissolves support 1, 2, 4, and 8 pixel blocks and erase to each
cell's paper color.

## Sprites

Software sprites draw on the bitmap and save the pixels and attributes
underneath. There are eight slots by default; use `@OPTION SPRITE_COUNT`
to change the count. Select `DISPLAY BITMAP_HIRES` before `SPRITES_ON`, which
resets the slots.

On the 128K, `@OPTION SPRITE_LIBRARY_BANK n` places sprite control and
composition code in physical RAM bank 0, 1, 3, 4, or 6 and enables RAM banking
automatically. Omitting the option keeps the library fixed.
Screen access routines stay fixed so both screen pages remain usable.
Sprite state and buffers stay in fixed RAM; `NOBASIC` provides extra room
for state. Keep shape data in fixed RAM.

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
Buffered drawing is enabled by default. Use `@OPTION SPRITE_DOUBLE_BUFFER FALSE`
to save memory and draw changes directly on the bitmap without waiting.
With buffering enabled, the call prepares sprites in a RAM buffer, waits
for the next frame, then copies the changed areas to the bitmap.
Call `FRAME_WAIT` after each update; it counts that wait without waiting again.
The buffer grows to cover the rectangle
containing old and new sprite positions.
An unchanged buffered flush keeps frame pacing and reuses the current screen.

`SPRITES_FLUSH frames` waits until at least that many frames have elapsed
since the previous update before displaying changes.
Use `SPRITES_FLUSH 2` for updates two frames apart when drawing keeps up.
This form calls `FRAME_WAIT` as needed; no extra wait is needed afterward.
`SPRITES_FLUSH 0` displays changes without waiting.

For manual drawing order, `SPRITE_RESTORE id` restores one sprite's
background while keeping its settings. The form
`SPRITE_RESTORE id, x, y, w, h` restores it only when its saved area
overlaps that pixel rectangle. Saved areas span whole 8 pixel columns.
Restore overlapping sprites in reverse drawing order before changing
the background, then use `SPRITE_DRAW id` to save and draw each visible
sprite in the desired order. These calls draw directly without waiting
for a frame.

`SPRITES_OFF` hides sprites and restores their backgrounds immediately;
`SPRITES_RESET` also frees the buffer and saved backgrounds and resets the slots.
Hide sprites before changing the background beneath them. `CLS` discards
saved backgrounds so later moves cannot restore an old screen.

## Timing

`FRAME_WAIT` waits for the next frame interrupt. `FRAME_WAIT count` and
`FRAME_DELAY count` wait for that many frames.

`FRAME_INSTALL ADDR handler` calls a procedure on each frame interrupt while
the main program runs. The ROM keyboard scan and clock continue to run.
`FRAME_UNINSTALL` removes the callback. Program exit restores the ROM
interrupt handler.

`TICKS` reads the ROM frame counter atomically. It runs at `TICKS_HZ`, one
tick per frame, and wraps after 24 bits.
`TICKS_RESET` sets a new zero point without changing the ROM counter.

Sinclair `PAUSE frames` uses the same frame source. A new key press ends
the pause early, and `PAUSE 0` waits for one new key press. A key held when
the pause starts must be released first.

## Sound

The 48K Spectrum has one blocking 1-bit beeper voice.

| Call | Meaning |
| --- | --- |
| `BEEP duration` | Play a fixed C4 tone for `duration` milliseconds |
| `BEEP period, cycles` | Call the ROM beeper with native values |
| `PLAY_NOTE` | Play a short MIDI note from 36 through 95 |
| `PLAY_NOTE_FOR` | Play a MIDI note for `duration` ticks (1/50 s) |

`SOUND_OFF` has no effect because a tone has already finished when a
blocking call returns.

Sinclair `BEEP duration,pitch` takes seconds and semitones relative to
middle C, including fractional and negative values.

The 128 uses its AY sound chip for three simultaneous voices, numbered
0 through 2, with volume 0 through 15. `PLAY_NOTE` starts a tone and
returns immediately; `SOUND_OFF voice` silences it. `SOUND_ALL_OFF`
silences all three voices. `PLAY_NOTE_FOR` holds a note for the given
number of ticks, then silences it. `PLAY_NOTE_SHAPE` also accepts
`SOUND_SHAPE_NOISE`; the noise period is shared by all voices.
`BEEP` continues to use the beeper.

On the 128K system, Sinclair `PLAY` accepts one to eight music strings.
The first three parts use the AY voices; `Y` assigns parts to MIDI
channels. Strings support notes, octaves, lengths, rests, ties, triplets,
repeats, tempo, volume, and envelope effects. `PLAY` waits for the music
to finish. CAPS SHIFT with SPACE interrupts it.

`SOUND_TIMED_NOTE` and `SOUND_TIMED_NOTE_SHAPE` start a note and return
immediately. Each voice has its own duration in 1/50 second ticks.
`SOUND_TIMED_REST` schedules silence; `SOUND_TIMED_BUSY` and
`SOUND_TIMED_REMAINING` report progress. `SOUND_TIMED_WAIT_ALL` waits for
all three voices. `SOUND_TIMED_UNINSTALL` stops the timers and silences them.
Timed sound and frame callbacks can run together.

`SOUND_ENVELOPE voice, attack, decay, sustain, release` gives each voice
its own volume envelope. Times use 1/50 second ticks; sustain ranges
from 0 through 15 and scales the note's volume. `SOUND_RELEASE voice`
starts the release fade. `SOUND_OFF voice` stops immediately.
The stage table form accepts duration and level pairs and stops the
voice after the last stage. Envelopes advance while the main program runs.

`AY.WRITE register, value` and `AY.READ(register)` access AY registers,
including the envelope period and shape. Set bit 4 of a voice's volume
register to use the envelope.

## Files

The FILE API uses the 48K ROM cassette routines. Files are standard tape
CODE blocks with names of up to ten characters; longer names are truncated.
An empty read name accepts the next CODE file. Names are case sensitive.

Sinclair `LOAD`, `SAVE`, and `VERIFY` handle `CODE` blocks. An omitted
load address uses the saved address; an optional length sets the maximum
block size. `SCREEN$` transfers the primary bitmap and attributes.
`VERIFY` compares the tape bytes with memory. Sinclair `SAVE` accepts
names of one to ten characters.

`@INCLUDE_FILE` adds files after the program in `.tap` and `.tzx` images.

One file can be open at a time. All channel numbers use the same stream.
`FILE_MODE_READ` loads the file when opened. `FILE_MODE_WRITE` buffers the
data and records it when `FILE_CLOSE` is called. The stream buffer holds
`ZXSPECTRUM_FILE_CAPACITY` bytes (4096). `FILE_LOAD` and `FILE_SAVE_BYTES`
handle larger CODE blocks in the supplied memory region. Text lines end
with carriage return.

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

Scalar strings grow in the available heap. Code starts at `$8000` on both
48K and 128K; `START_CODE` can select another available program address.

Additional dialect forms include:

- `LET`, `GO TO`, `GO SUB`, `RUN`, `STOP`, and computed branches
- `DEF FN` and numeric or string `FN` calls
- `FOR`, `NEXT`, `IF`, `READ`, `DATA`, and `RESTORE`
- `RND` and `RANDOMIZE`
- `IN port` and `OUT port,value`
- `INT`, `ABS`, `SGN`, `SQR`, `SIN`, `COS`, `TAN`, `ASN`, `ACS`, `ATN`,
  `EXP`, `LN`, `PI`, and `^`
- `CHR$`, `STR$`, `CODE`, `LEN`, string slicing, `VAL`, and `VAL$`
- numeric `USR address`, returning BC, and upper- or lowercase
  `USR "a"` style UDG addresses

`RUN` clears variables and the screen, then starts at the first line or
the requested line.

`STR$` uses the ROM's normal number format. `VAL` evaluates a numeric
expression; `VAL$` evaluates a string expression. Both use the current
variables, array bounds, and `DEF FN` definitions.

## References

- [ZX Spectrum BASIC Programming manual](https://worldofspectrum.org/ZXBasicManual/)
- [Complete Spectrum ROM disassembly](https://skoolkid.github.io/rom/)
