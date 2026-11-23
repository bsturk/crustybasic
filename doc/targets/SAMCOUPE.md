# SAM Coupe

Use `--set target=samcoupe` for the 256K model, or
`--set system=samcoupe.512k` for the 512K model. Programs use the upper
32K of RAM and return to BASIC when they end.

Build with `vasmz80_oldstyle`, the same Z80 assembler used for the ZX Spectrum.
Install it in `tools/assemblers/vasm/<platform>/` or on `PATH`.
See the [assembler setup](../USAGE.md#assemblers).

The default output is a SAMDOS disk image
(`.mgt`) containing the program and its companion files. In SimCoupe, enable
"Auto-load media inserted at startup screen" under F10 > Helpers. At the
striped startup screen, press F1 and open the disk. From the BASIC prompt,
press Numpad 9 with Num Lock on to boot the disk in drive 1.
On a SAM with SAMDOS already loaded, insert the disk in drive 1 and type `BOOT`.
For a tape image, select `--set output-type=tap` and load it with `LOAD "" CODE`.
Use `--set output-type=raw` for a binary file (`.bin`).

Select SAM BASIC with `@OPTION DIALECT sam_basic` or `--set dialect=sam_basic`.
Spaces before numeric arguments are optional, as in `MODE4` or `PEEK32512`.
It accepts numbered listings, `DEF FN`, `DEF PROC` with `REF` parameters and `LOCAL`
variables, multiline `IF`, `DO`/`LOOP`, `EXIT IF`, `LOOP IF`, and `ON`
statement selection. Numeric calculations use the ROM calculator and its five byte number format.
`MOD` accepts fractions and is evaluated before multiplication and division.
`RANDOMIZE N` rounds a seed to an integer from 0 through 65535. A seed that
rounds to zero, or bare `RANDOMIZE`, uses the frame counter. Reusing a nonzero
seed repeats its sequence. `RND` returns a value at least 0 and less than 1.
`RND(N)` multiplies the next value by `N+1` and discards its fractional part.
Called procedures see the caller's parameters and LOCAL variables.
String value parameters have their own dimensions, visible through `LENGTH` and `VAL`.
LOCAL takes effect when executed; outer values return when the procedure ends.
`LOCAL A()` gives a numeric array its own dimensions and elements until the
procedure returns. Called procedures see the active local array.
`LOCAL A$` does the same for strings and string arrays, including their widths.
`DEF FN` parameters belong to that function. Other names use the current
program variables, including a calling procedure's LOCAL variables.
`VAL` and `VAL$` inside a function can read its parameters.
Numeric array dimensions can be calculated at runtime. `DIM` clears their elements.
`DIM` rounds dimensions and string widths to integers from 0 through 65535,
treating zero as one. It evaluates them from left to right.
Numeric and string arrays are removed before their new sizes are evaluated.
Array indices round to integers and must be within each dimension.
Each index is checked before the next index or the assigned value is evaluated.
The number of indices must match the current array dimensions.
`LENGTH(1,A())` and `LENGTH(2,A())` return array dimensions; use `A$` for
string arrays. `LENGTH(0,V)` returns the address of a variable or array's data.
It also returns that address when evaluated through `VAL`.
The axis rounds to an integer from 0 through 2. Numeric scalars accept only 0.
`INPUT` accepts prompts and reads variables, array elements, and string slices.
Use `AT` and `TAB` to position prompts and fields. Their operands round to
the nearest integer. Commas advance to the next tab stop, apostrophes start
a new line, and semicolons keep items together.
Prompt attributes last until editing starts; entered text uses the stream's
normal colors. `#N` selects a stream, including between fields, rounding
its number to the nearest integer from 0 through 16.
Parenthesized items display expressions and can contain layout controls;
`(A)` displays A instead of reading it.
`INPUT LINE` reads text without evaluating it.
Left and Right move within the text. Delete removes the character to the
left; Shift+Delete removes the character to the right.
Up and Down move between rows in the lower input area, or to the end if no
position matches. They move to the end in upper screen fields.
EDIT clears the field. Use the keypad toggle to enter numbers with F0 through F9.
`DEF KEYCODE N,TEXT` assigns text to a key code. `DEF KEYCODE N:...`
stores the rest of the line, including BASIC keyword codes.
INPUT inserts the definition
at the editing position and accepts the field automatically; a trailing
colon keeps editing open. An empty string removes the definition, and
`DEF KEYCODE ERASE` removes all definitions. Keypad number mode takes
precedence over definitions for F0 through F9.
After a syntax error, the text and editing position are kept for correction.
`VAL` evaluates a numeric expression stored in a string; `VAL$` evaluates
a string expression. These and `INPUT` expressions can read numeric and
string variables, including procedure parameters and LOCAL variables,
numeric and string array elements, and string slices.
Trailing procedure arguments may be omitted. `DEFAULT` supplies a value
when a variable has not been assigned. Zeroes, empty strings, and dimensioned
array elements keep their values. The proposed value is evaluated in either case.
`REF` assignments keep their values after returning, including zero and empty strings.
When the same numeric variable is passed more than once, the first parameter's
value is kept after returning.
Dimensioned strings keep their width through `REF`; `DIM` changes that width for the caller.
Numeric array parameters use `REF A()`; string arrays use `REF A$`.
Changes to their elements and dimensions
reach the caller. An omitted array parameter exists only for that call.
Supplying an unassigned numeric variable by `REF` gives it zero.
An undefined string passed by `REF` must receive a value before returning.
`DEF PROC NAME DATA` takes a list of values consumed by `READ`.
`ITEM` reports the next value's kind. DATA expressions are evaluated when read.
`RESTORE` selects a DATA line. `READ LINE` returns DATA text with its
quotes and keyword codes.
`LABEL` creates a numeric variable containing its line number.
`GO TO` and `GO SUB` accept expressions, round the destination, and use the
next numbered line when that line is absent.

`PEEK`, `DPEEK`, `POKE`, and `DPOKE` use SAM BASIC's linear memory addresses,
including RAM beyond 64K. Addresses and numeric values round to the nearest
integer. Words store the low byte at the lower address.
Starting addresses must be nonnegative and below 540672, before rounding.
At 32767.5 and 65535.5, byte and word forms wrap to 16384 and 49152;
string forms round into the next page.
After rounding, `POKE` accepts -255 through 255; negative values wrap into
a byte. `DPOKE` accepts 0 through 65535.
`POKE ADDRESS,A,B,C` writes consecutive bytes after evaluating all values.
Numeric lists contain up to 32 values and write from last to first.
An invalid value leaves any later bytes already written.
`MEM$(FIRST TO LAST)` reads an inclusive memory range into a string.
The result length rounds to 0 through 65535 bytes. Empty results still
require a valid starting address. The address limit applies to the start
of a word or string transfer.
`POKE ADDRESS,TEXT$` writes the string's bytes, including zero bytes.
Below 16384, these string operations access the RAM beneath the ROM.

`IN PORT` reads a byte from an I/O port; `OUT PORT,VALUE` writes one.
Ports and values round to the nearest integer. Ports range from 0 through
65535 and output values from 0 through 255.

`USR ADDRESS` calls machine code and returns the value in BC.
`USR$ ADDRESS` returns the string described by A (RAM page), DE (address),
and BC (length). Addresses can select RAM beyond 64K.
They follow the byte address rules above. BC contains the mapped address
on entry to the called routine.
`CALL ADDRESS` ignores the return value. Additional numeric and string
arguments are passed on the SAM calculator stack, with their count in A.
CALL evaluates its arguments before checking the destination address.

`DEVICE D1` or `DEVICE D2` selects a disk drive. A `D1:` or `D2:` filename
prefix selects the drive for that operation. Filenames ignore letter case.
`SAVE "name" CODE START,LENGTH` saves bytes using SAM BASIC's linear addresses.
Add a third address to run machine code when the file is loaded.
An existing filename gets a Yes/No overwrite prompt. Use
`SAVE OVER "name" CODE START,LENGTH` to replace it without a prompt.
`LOAD "name" CODE` restores the saved address; adding an address loads there.
An optional length sets the space available, followed by an optional execution
address. `MERGE "name" CODE` accepts the same arguments and skips the saved
execution address. `VERIFY "name" CODE` compares the file with memory; an
explicit length must match the file length.
Prefixing the saved filename with `CHR$ 2` protects its execution address
from overrides and makes `MERGE` run it too.
`ERASE "name"` deletes the named disk file.
With MasterDOS, `ERASE PATTERN$` deletes matching files. `ERASE OVER`
also deletes protected files. Append `?` to ask before each deletion.
`RENAME OLD$ TO NEW$` renames files. The new name can contain `?` to keep
the corresponding character or `*` to keep the rest of the name.
Append `?` to ask before each rename.
`COPY SOURCE$ TO DEST$` copies files using the same patterns and name
templates. `COPY OVER` replaces existing files without an overwrite prompt.
Append `?` to choose which source files to copy. Copies on one drive ask
for confirmation when switching between source and destination disks.
`PROTECT PATTERN$` protects matching disk files; `PROTECT OFF PATTERN$`
removes protection. `HIDE PATTERN$` hides matching files from directory
listings; `HIDE OFF PATTERN$` shows them again. Append `?` to either
command to ask before each change.

`SAVE "name" SCREEN$` saves the selected screen and its colours.
`LOAD "name" SCREEN$` and `MERGE "name" SCREEN$` restore its mode and image
to the selected screen. `VERIFY "name" SCREEN$` compares it with the file.
Loading a hidden screen keeps the display's current line colour changes.
`SAVE OVER` also applies to screens. `CODE` and `SCREEN$` files can be loaded
using either form; loading a `CODE` file as `SCREEN$` keeps the current mode.

`SAVE "name" DATA A()` saves a numeric array. Use `A$()` for a string array
or `A$` for a string. The file keeps the current dimensions and string widths.
`SAVE OVER` replaces an existing DATA file without a prompt.
`LOAD "name" DATA A()` restores a numeric array. Use `A$()` for a string array
or `A$` for a string. Loading creates the named variable or replaces its
previous contents, restoring the saved dimensions and string widths.
`VERIFY "name" DATA A()` compares a saved numeric array with memory.
Use `A$()` for a string array or `A$` for a string. Names, dimensions, lengths,
and values must match.

String slices use one based positions: `A$(2 TO 4)`, `A$(TO 4)` and
`A$(2 TO)`. Positions round to integers from 0 through 65535.
Character positions must be within the string. Nonempty ranges must fit
the string. A reversed range or an explicit end of 0 returns an empty string.
Slice assignments check the destination before evaluating the new text,
then pad with spaces or truncate to fit. Assigning an empty or reversed
range leaves the string unchanged.
The last dimension in `DIM A$(10,20)` sets each string's width.
String arrays accept a final character position or slice after their element indices.
`DIM` takes effect when executed and replaces any previous string or array
with blank strings. It can change the number of dimensions.
Function results can be sliced, as in
`(STR$(123))(TO 2)` or `FN TEXT$(A$)(2 TO 4)`.
`STRING$` repeats strings with a result of up to 511 characters. For a
nonempty result, the rounded repeat count and source length are limited
to 255. `TRUNC$` removes trailing spaces from strings of 1 through 255
characters. `INSTR` searches with the wildcard character stored at `SVAR(5)`
(normally `#`).
INSTR's optional first argument rounds to a position from 1 through 65535.
Search patterns contain up to 255 characters.
`CHR$` returns one character from a value that rounds to 0 through 255.
`BIN` introduces a binary number from 0 through 65535; spaces may separate
its digits. `BIN$` returns 8 or 16 binary digits; `HEX$` returns 2, 4 or 6
hexadecimal digits. `SVAR(3)` and `SVAR(4)` select the characters for binary
one and zero.
`SVAR(N)` returns `23040+N`, including fractional offsets.
`SCREEN$(ROW,COLUMN)` reads a character from the screen, including user
defined characters. Row and column round to integers and start at the top
left of the screen. They must fit the current window's right edge and its
bottom plus the two editing rows. It returns an empty string when no
character matches.

SAM BASIC graphics use `MODE`, `PEN`, `PAPER`, `BORDER`, `PALETTE`, `PLOT`, `DRAW`,
`DRAW TO`, `CIRCLE`, and `FILL`. Coordinates start above the editing area
and increase upwards. `MODE` rounds its number to the nearest integer
and accepts modes 1 through 4. It preserves drawing colors. Leaving mode 3
restores the colors used before entering it.
`BORDER` rounds its palette index to an integer from 0 through 15 and
updates the editing area colors.
`RECORD TO TEXT$` starts a fresh graphics string while commands continue
to draw. `RECORD STOP` ends recording. In mode 3, use `FATPIX 1` to record
PLOT, DRAW, DRAW TO, and CIRCLE commands.
`BLITZ TEXT$` draws a string of encoded graphics commands. It uses `XOS`,
`YOS`, `XRG`, and `YRG` for positioning and scaling. Color changes in the
string apply to that BLITZ call.
In `MODE 3`, `FATPIX 1` doubles pixel width and
`FATPIX 0` restores thin pixels. The setting carries across `MODE` changes.
The value rounds to the nearest integer and must be 0 or 1.
`XOS` and `YOS` move the graphics origin. `XRG` and `YRG` set the coordinate
ranges; smaller ranges enlarge drawings. Relative `DRAW` scales and rounds
the movement without adding the origin offsets. Movement is limited to
255 pixels vertically and 255 horizontally, or 511 horizontally with thin
mode 3 pixels. Zero movement draws nothing. An off-screen error keeps the
pixels already drawn.
`PLOT` and `DRAW TO` round the scaled
position to a pixel and require it to be on screen.
An optional third argument to `DRAW` or `DRAW TO` gives a curve angle in radians.
`CIRCLE` scales its center and keeps its radius
in pixels. The radius rounds to an integer from 0 through 255; zero plots
one pixel. `CIRCLE` leaves the drawing position unchanged.
These variables also work in `VAL` expressions.
`FILL X,Y` uses the current pen; `FILL PEN C,X,Y` supplies a temporary color.
It works in modes 3 and 4. The starting point uses the graphics origin and
ranges and must lie on screen.
`FILL USING A$,X,Y` uses a 128-byte pattern with 16 rows. Each byte holds
two mode 4 pixels or four mode 3 thin pixels. The optional final flag accepts
values that round to 0 or 1. `1` reuses the screen copy from the preceding
fill; omit the flag or use `0` to refresh that copy.
Temporary attributes can follow the pattern, as in
`FILL USING A$;PEN C;X,Y`. Commas and semicolons separate these items.
`GRAB A$,X,Y,W,H` captures a block in modes 3 and 4. `PUT X,Y,A$` draws it;
an additional string supplies a mask. Set mask bits copy the block and
clear bits keep the screen. Positions use the graphics origin and ranges
and align down to whole bytes. Coordinates and dimensions round to integers.
W accepts 1 through 256 mode 4 pixels, rounded up to an even width; mode 3
captures twice as many thin pixels. H accepts 1 through 192 pixel rows.
Block sizes do not scale. A capture clips at the bottom of the screen,
while its header keeps the requested height. Captured pixel data must fit
within 8172 bytes, plus the three byte header.
PUT accepts blocks up to 8191 bytes and masks up to 4095 bytes.
Mask dimensions must match the block dimensions.
For `PUT`, `OVER 1` uses XOR, `OVER 2` uses OR, and `OVER 3` uses AND.
`INVERSE 1` inverts the block colors. A mask takes precedence over `OVER`.
In modes 3 and 4, `ROLL D,P` wraps screen pixels; `SCROLL D,P` fills the
cleared area with the paper color. Directions are 1 left, 2 up, 3 right,
and 4 down. Omit P to move one pixel. Add `X,Y,W,H` to move a rectangle.
Direction and distance round to integers; distance must be from 1 through 255.
Positions use the graphics origin and ranges; dimensions and distances
do not scale. Horizontal units use mode 4 pixel widths, including in
mode 3. Rectangle positions and widths align down to whole bytes.
Horizontal distances greater than one round down to an even value.
After rounding, rectangle widths align down to an even value from 2 through
256. Heights round to integers from 1 through 255, and the rectangle must
fit on screen. Horizontal movement must be smaller than the width;
vertical movement may cover the whole height.
`WINDOW L,R,T,B` sets the text bounds in character columns and rows.
Coordinates round to the nearest integer and must fit the current mode
and character size. A window needs at least two columns and one row.
`WINDOW` restores the full text area. `PRINT AT` takes row then column,
relative to the window. In `PRINT`, commas advance to the next tab stop,
apostrophes start a new line, and semicolons keep items together.
`PRINT #N` selects an output stream, including between items. Stream 16
appends to the string selected by `RECORD TO`, even after `RECORD STOP`.
Stream numbers round to the nearest integer and must be from 0 through 16.
`AT` coordinates must round to 0 through 255; `TAB` accepts rounded values
from 0 through 65535.
`PEN`, `PAPER`, and other color items apply temporarily to the output.
`OPEN #N;CHANNEL$` assigns a stream to `"S"` (upper screen), `"K"` (lower
screen), `"P"` (formatted printer), `"B"` (raw printer), or `"$"` (recorded
string). Channel letters accept either case. `LPRINT` uses stream 3;
ordinary `PRINT` uses stream 2. `CLOSE #N` restores streams 0 through 3
to their defaults and closes other streams.
`OPEN #N;FILENAME$` opens a file through the installed DOS. With MasterDOS,
append `IN` or `OUT` to select reading or writing. `RND` opens a file for
both reading and writing, creating it if needed. `PRINT #N` writes to an
output file, and `CLOSE #N` saves its pending output.
`INPUT #N` reads strings and numeric expressions from an input file.
`INPUT #N;LINE TEXT$` reads text through the next carriage return.
`INKEY$ #N` reads one byte as a string. Keyboard streams return an empty
string when no key is waiting. File reads report an error at the end of the file.
`INP$(#N,COUNT)` reads a fixed number of bytes, including carriage returns.
The count is rounded to an integer from 1 through 16384.
`PTR N` gives the next byte position, starting at zero. `EOF N` returns 1
at the end of the file. `LENGTH #N` gives the file length in bytes; enclose
it in parentheses when combining its result with another expression.
`POINT #N,OFFSET` selects a byte position, starting at zero. Seeking to the
file length lets `PRINT #N` append data. `POINT #N,OVER COUNT` skips records
from the current position.
`DIR` lists files on the current drive. `DIR DRIVE,PATTERN$` selects a
drive and filename pattern; either argument can be used alone. A supplied
drive or pattern gives a detailed listing; append `!` for names only.
`DIR #N;` sends the listing to a stream. With MasterDOS, `DIR DATE`
includes file dates and filename patterns accept wildcards.
A quoted pattern can follow the stream or drive expression directly,
without a comma or semicolon.
With MasterDOS, `OPEN DIR NAME$` creates a folder and `DIR=PATH$` selects
one. `DIR=/` or `DIR=\` selects the root; `DIR=^` selects the parent.
`RENAME DIR OLD$ TO NEW$` renames a folder. `ERASE DIR NAME$` deletes
an empty folder. Both commands also accept patterns.
CODE and DATA file operations use the selected folder, as does `ERASE`.
Filenames can include folders separated by `/` or `\`. A leading separator
starts at the root; `^NAME` selects NAME in the parent folder. These paths
leave the selected folder unchanged. Paths can contain up to 79 characters,
with up to 10 characters in each folder or file name.
`DIR ?` lists files in every folder. The `?` can also follow a pattern
or its trailing `!`. Use `*` in place of a drive number for the current drive.
After sending an all-folder listing to a file, select that file's folder
with `DIR=` before closing it.
`CSIZE W,H` sets character dimensions. Mode 3 uses widths of 6 or 8 pixels;
the other modes use 8. Heights range from 6 to 32 pixels, with double height
characters at 16 or more. Dimensions round to the nearest integer.
`CSIZE` updates the window limits and graphics origin.
`CLS` or `CLS 0` clears the screen; `CLS 1` clears
only the window. The argument rounds to an integer from 0 through 255.
Values above 1 also clear the lower text area.
`CLS#` also resets the colors, border, and palette.
`OPEN SCREEN N,M` reserves a screen in mode M; omitting M uses the current
mode. `SCREEN N` selects it for drawing and printing, restoring its mode,
colors, window, and cursor positions. `DISPLAY N` shows a screen while
drawing can continue on another. `DISPLAY` or `DISPLAY 0` follows the
selected screen. `CLOSE SCREEN N` releases its memory. Screen numbers and
mode numbers round to the nearest integer. Screen numbers run from 1
through 16.
`PALETTE I,C` sets color C at palette position I. Positions run from 0
through 15 and colors from 0 through 127; both round to the nearest integer.
`PALETTE I,A,B` alternates two colors at palette position I.
Add `LINE Y` to change colors from that graphics line downwards;
`PALETTE I LINE Y` removes the change. Line positions use `YOS` and `YRG`.
The scaled position rounds to the nearest pixel. Changes start below the
first scan line, with up to 127 changes per screen.
`PEN`, `PAPER`, `FLASH`, `BRIGHT`, `INVERSE`, and `OVER` set attributes.
Their values round to the nearest integer. Attributes supplied before a
semicolon within a print or drawing command apply to that command.
Drawing attributes can also be separated by commas. The next command uses
the permanent settings. `BEEP` takes seconds and a semitone offset from
middle C. Fractional pitches give smaller intervals. The duration must be
below 16 seconds.
`SOUND` takes a register and value. Both round to the nearest integer:
registers run from 0 through 31,
and values from 0 through 255. Separate additional pairs with semicolons,
up to 127 pairs per statement.
`PAUSE` waits for a key, or for a given number of frames.
The count rounds to an integer from 0 through 65535. Zero waits for a key;
a key also ends a timed pause early.
`SCROLL CLEAR` makes text scroll automatically; `SCROLL RESTORE` brings
back the scroll prompt.

In crustyBASIC syntax, `DISPLAY` accepts `MODE_1` through `MODE_4`:

| Mode | Pixels | Colors |
| --- | --- | --- |
| 1 | 256 x 192 | two per 8 x 8 cell |
| 2 | 256 x 192 | two per 8 x 1 cell |
| 3 | 512 x 192 | four per pixel |
| 4 | 256 x 192 | sixteen per pixel |

Colors come from a palette of 128. Use `PALETTE_SET` with a native color
or RGB values. `ASIC.PALETTE_LINE` changes a palette entry at a scan
line. `RASTER_STOP` pauses these changes and `RASTER_START` resumes them.
`GFX_PAGE_DRAW` and `GFX_PAGE_SELECT` select two screen pages.
Modes 1 and 2 share brightness between foreground and background in each
cell. `COLOR` takes brightness from the foreground color.

`DISPLAY TEXT_32X24`, `DISPLAY TEXT_64X24`, and `DISPLAY TEXT_80X24`
select text grids with 24 rows. Graphics include drawing, pixel and cell
blits, bitmap tiles, software sprites, scrolling, and image transitions.
`SPRITE_EXPAND` doubles either sprite dimension. `SPRITE_PRIORITY` places
sprites behind pixels that differ from the current background color.
Page flipping keeps separate sprite backgrounds for each drawing page.
In mode 4, `SPRITES_FLUSH` builds sprite changes off screen and shows them
at the start of the next frame; the following `FRAME_WAIT` counts that
wait. `@OPTION SPRITE_DOUBLE_BUFFER FALSE` redraws sprites on the visible
screen instead. A program that shows sprites only in mode 4 can add
`@OPTION SAMCOUPE_SPRITE_LOW_MODES FALSE` to leave out the mode 1 to 3
sprite code and gain program space.
`CHARSET_DEFINE` changes eight byte character patterns.
In mode 4, `CELL_COLOR` recolors what a cell already shows. Use
`CELL_PUTC x, y, c, color, attr` to draw a cell in a color.
`IMAGE_LOAD` reads prepared images and native SAMDOS `SCREEN$` files.

Sound has six voices with tone and noise. `SAA1099.STEREO` sets separate
left and right volumes. Timed notes and frame callbacks run at 50 Hz.
`SOUND_ENVELOPE` sets an independent volume envelope or stage table for
each voice. Envelope durations use 50 Hz ticks. `SOUND_RELEASE` starts
the release phase; `SOUND_OFF` stops the voice immediately.
Timed ADSR notes enter release when their duration expires. The release tail
can continue after `SOUND_TIMED_BUSY` and `SOUND_TIMED_REMAINING` reach zero.
`AUDIO_LOAD` accepts prepared WAV audio from memory or disk. Playback uses
four bit mono samples through the sound chip and blocks until finished.
`SAA1099.WRITE` writes a SAA1099 register directly.

Input includes the keyboard, two joystick selections, and a mouse.
`KEY_ESCAPE` is 27 and `KEY_TAB` is 252. `RAWKEY_CODE` also reports the
cursor keys and Delete.
`MIDI_WRITE`, `MIDI_READY`, and `MIDI_READ` provide MIDI access.
`PORT_IN` and `PORT_OUT` access 16 bit I/O addresses.

Disk files use ten character names and an optional `1:` or `2:` drive
prefix, or the native `D1:` and `D2:` forms. One file can be open at a time.
Read, write, append, update, and directory modes are available.
`FILE_OPEN_NATIVE` accepts the mode in AUX1 and `DOS_FILE_REPLACE`
in AUX2 to let write mode replace a protected file.
`FILE_SEEK` selects a byte position from the start through the end of a file.
`FILE_POSITION` returns the next byte position as `U32`.
`FILE_LOAD_PROGRAM` uses the file's stored RAM page and address, or an
explicit destination. `FILE_COMMAND $A6` deletes a file.
`SECTOR_READ` and `SECTOR_WRITE` transfer a 512 byte sector.

`EXTMEM` uses free RAM in logical 64K banks. `EXTMEM_PEEK(bank, offset)`
and `EXTMEM_POKE bank, offset, value` access individual bytes.
`EXTMEM_DETECT_KB()` returns the available extra RAM. Transfers restore
the previous mapping and reserve their banks through BASIC's page table.

`RAM_PAGE bank` selects the physical 16K page at `$0000` through LMPR;
the following page appears at `$4000`. `RAM_PAGE()` returns the selected
page. The 256K model has pages `0` through `15`; the 512K model has pages
`0` through `31`. ROM and write protection settings are preserved.
Programs use a stack in upper RAM. ROM interrupts temporarily restore the
system page and can remain enabled. Keep working storage outside the mapped
range, and restore the page before ROM calls or EXTMEM operations. This
direct mapping does not reserve pages.
