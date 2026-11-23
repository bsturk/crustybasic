# crustyBASIC on Apple II

Use `@OPTION TARGET apple2` for the default Apple //e profile, or
pick an exact system with `@OPTION SYSTEM`.

Portable runtime calls are documented in [`../API.md`](../API.md).
This page covers Apple II systems, modes, formats, and hardware notes.
For command-line target selection and dialects, see
[`../USAGE.md`](../USAGE.md).

## RAM pages

On `apple2.e` and `apple2.c`, `RAM_PAGE 0` selects main RAM and
`RAM_PAGE 1` selects auxiliary RAM for reads and writes at `$0200-$BFFF`.
`RAM_PAGE()` returns the read bank. The paging routine and caller must
be present at the same addresses in both banks. Zero page, stack, ROM,
and display controls retain their settings; `80STORE` can override
the selected bank for screen memory.

## Systems

| System | Machine | Notes |
| --- | --- | --- |
| `apple2.plus` | Apple II Plus | Applesoft ROM, `REAL` supported. |
| `apple2.int` | Original Apple II | Integer BASIC ROM. No `REAL`. |
| `apple2.e` | Apple //e | Default. Adds lowercase and 80-column softswitches. |
| `apple2.c` | Apple //c | IIe features plus mouse input. |

`CAN_USE_UNDOCUMENTED_OPCODES` is `TRUE` for `apple2.plus` and
`apple2.int`. The IIe and IIc profiles use the documented instruction set
for compatibility with 65C02 processors.

## Outputs

The default `dos33` output is a DOS 3.3 `BRUN` binary (`.bin`). With
`--set disk-image=true`, `build` also writes a bootable `.dsk` using
the bundled DOS 3.3 image writer. Files staged by a two-path
`@INCLUDE_*` form go on that disk, uppercased and cut to the 30
characters a DOS 3.3 catalog holds.

Select `--set output-type=nodos` or `@OPTION OUTPUT_TYPE nodos` to
boot directly into the program without loading DOS. Code starts at
`$0900`, and program RAM extends through `$BFFE`. Use
`--set disk-image=true` to write the bootable `.dsk`.

Select `--set output-type=prodos` or `@OPTION OUTPUT_TYPE prodos` for
a ProDOS `.system` program. Copy it onto a ProDOS volume as type `SYS`
with auxiliary type `$2000`. ProDOS loads it at `$2000`; the program
then moves to `START_CODE`. This output requires 64 KB with a language
card. Copy generated companion files beside it as type `BIN`; their
names use up to 15 uppercase letters, digits, and dots.

Choose `dos33` for `APPLE_DOS_COMMAND`. The `nodos`
output keeps ROM services, including Applesoft math where available.

## Basics

| Item | Value |
| --- | --- |
| CPU | 6502 |
| Text | 40x24 |
| Code start | `$0C00` |
| Program RAM top | `$9AA6` for `dos33` (`$9600` with DOS file I/O), `$BFFF` for `nodos`, `$BF00` for `prodos` |
| Text screen | `$0400` or `$0800`, Apple interleaved rows |
| String encoding | Uppercase ASCII on II/II Plus, ASCII on //e,//c |
| Sound | Speaker click; optional six voice Mockingboard |
| Input | Keyboard, 2 joystick ports, 4 paddle axes |
| Recursive PROCs | Yes |

Cursor positioning handles the Apple text screen layout. If you poke
screen memory yourself, use the usual Apple II row addressing rules.

`apple2.plus` and `apple2.int` fold text to uppercase. `apple2.e` and
`apple2.c` keep mixed case.

`DISPLAY TEXT_40X24` selects 40 column text. On `apple2.e` and
`apple2.c`, `DISPLAY TEXT_80X24` selects 80 column text.

The runtime uses `$0300-$039F` for changeable data. The `nodos` output
also uses `$0200-$02FF`, which DOS 3.3 needs for its command buffer.
`$03A0-$03CF` holds saved Applesoft zero-page state. Programs using their
own assembly must not use these ranges or call the Monitor line editor,
which uses page 2 as its input buffer.

Stock Apple II text glyphs are in character ROM and are not
RAM-redefinable through runtime charset support.

## Graphics

Apple II graphics support text/cell mode, lo-res graphics, and hi-res
graphics. Sprite calls use an 8x8 software blitter. Sprites draw on HGR,
and on IIe and IIc also on `DLGR`, `DHGR_COLOR`, or `DHGR_MONO`. Select the
mode with `DISPLAY` before `SPRITES_ON`. Sprites use page flipping by default. With page flipping,
background drawing updates both pages while preserving the sprites.

With page flipping enabled, `SPRITES_FLUSH` waits for a safe page change
on `apple2.e` and `apple2.c`. Call `FRAME_WAIT` after each update; it counts
that wait without waiting again.
Sprites whose position, color, and shape bytes match the hidden page are
left alone. Changing raw shape bytes in place is supported.

HGR speed builds cache single tile sprites at their current horizontal alignment
and combine overlapping moves when their colors match. Each configured sprite
slot reserves a 24 byte cache and a 24 byte saved copy per screen page.
Changing shape bytes or horizontal alignment refreshes the cache.
Larger tiled sprites, default builds, and size builds use compact shift
routines. Page flipping remains a separate choice.

`SPRITE_DATA_8X8` accepts the portable MSB-left 8x8 format and converts
it automatically. Raw `SPRITE_DATA` uses Apple II native 8x8 rows: bit 0
is the leftmost pixel and bit 7 is the rightmost. Rows copied from hi-res
screen memory need their color bit cleared first, or it draws as a pixel.
In HGR, each row spans two 7 pixel screen bytes. `SPRITE_DATA_TILES` accepts any runtime tile
width and height. Tiles are converted and drawn left to right, then top
to bottom, with each tile covering 8x8 pixels.

`SPRITE_FLIP_X` and `SPRITE_FLIP_Y` reflect the whole shape.
`SPRITE_EXPAND` doubles its width, height, or both. These settings apply
to later shape changes and leave the source bytes unchanged.

Without page flipping, default and size builds update sprites immediately.
Speed builds wait for `SPRITES_FLUSH`. HGR can combine matching single tile
moves into one update.

Background drawing works with either buffering choice. `SPRITE_HIT` tests
overlapping opaque pixels at the recorded sprite positions. `SPRITE_HIT_BG`
tests those pixels against the background without counting other sprites.
Both queries return `1` for a collision and `0` otherwise.

In HGR, sprite colors can recolor neighboring pixels that share a byte.
Moving or hiding a sprite restores the background colors. Overlapping
sprites retain the color of the most recently drawn sprite in that byte.

In DLGR and DHGR color, sprites XOR their selected color with the background.
DHGR mono treats any nonzero sprite color as ink. Sprites start in white.

`SPRITE_PRIORITY ID, TRUE` keeps the sprite behind occupied background
pixels and preserves their colors. Use `FALSE` to return the
sprite to the front.

HGR and DHGR page flipping use both hi-res pages. Programs must fit below
`$2000` or load above both pages with `@OPTION START_CODE $6000`.
DLGR page flipping uses `$0400-$0BFF` in main and auxiliary memory.

The default graphics mode is `BITMAP_HIRES`.

`DLGR`, `DHGR_COLOR`, and `DHGR_MONO` are full screen modes available
only on `apple2.e` and `apple2.c`. On Apple //e, all three require revision B
video hardware. `DLGR` requires a suitable 80 column auxiliary memory card.
The DHGR modes require an Extended 80 Column Card, normally giving the
machine 128K total. Apple //c has the needed hardware built in. There is no
runtime hardware check, so selecting one of these modes assumes the required
memory and video hardware are present.

| Mode | Apple II mode | Size | Colors |
| --- | --- | --- | --- |
| `CELL` | 40 column text page 1 | 40x24 | text |
| `TEXT_40X24` | 40 column text page 1 | 40x24 | text |
| `TEXT_80X24` | 80 column text page 1 on //e and //c | 80x24 | text |
| `BITMAP_HIRES` | HGR page 2 | 280x192 | 2 |
| `BITMAP_LORES` | full screen lores | 40x48 | 16 |
| `HGR_PAGE2` | HGR page 2 | 280x192 | 2 |
| `HGR_PAGE2_MIXED` | HGR page 2 mixed text | 280x160 | 2 |
| `HGR_PAGE1` | HGR page 1 | 280x192 | 2 |
| `HGR_PAGE1_MIXED` | HGR page 1 mixed text | 280x160 | 2 |
| `LGR` | full screen lores | 40x48 | 16 |
| `LGR_MIXED` | lores mixed text | 40x40 | 16 |
| `DLGR` | double lores page 1, main and auxiliary memory | 80x48 | 16 |
| `DHGR_COLOR` | double hires page 2, main and auxiliary memory | 140x192 | 16 |
| `DHGR_MONO` | double hires page 2, main and auxiliary memory | 560x192 | 2 |
| `BITMAP_MULTICOLOR` | not supported | | |
| `CELL_MULTICOLOR` | not supported | | |

Portable mixed display is the fixed Apple II split:

```basic
STATUS = DISPLAY_MIXED(BITMAP_HIRES, 20, 4)
STATUS = DISPLAY_MIXED(BITMAP_LORES, 20, 4)
```

Both modes place graphics above text rows 20 through 23. Graphics keep their
normal coordinates, though the bottom 32 hi-res rows or bottom 8 lo-res rows
are hidden. `CLS` clears the bitmap without erasing the text rows. `CELL_CLS`
clears only the four visible text rows. Other modes and row layouts return `0`
without changing the display. Passing `0` for the text row count selects the
normal full screen mode.

`DISPLAY` sets `TILE_COLOR_ALTERS_SHAPE` to `TRUE` in HGR modes, where
colors use alternating pixels, and `FALSE` in other modes. White on black
preserves the tile's full shape.

Hi-res mixed mode uses page 1. Because that page occupies `$2000-$3FFF`,
hi-res mixed display programs must fit below `$2000` or use
`@OPTION START_CODE $6000` to load above both hi-res pages.

`DHGR_MONO` uses individual 560 dot bits and is intended for a monochrome
display. A composite color display interprets the same memory as double hires
color.

`GFX_PIXEL_BLIT` draws an opaque one-bit image in every graphics mode.
Source rows place the left pixel in bit 7 and pad to whole bytes. Set bits
use the foreground color; clear bits use the background color.

### Cell blits and XOR movement

`GFX_CELL_BLIT` and `GFX_CELL_XOR` draw native 8x8 cells in HGR, DLGR,
and both DHGR modes. Source cells are consecutive:

- HGR uses eight bytes per cell, one byte per row
- DHGR monochrome uses eight bytes per cell, with the left pixel in bit 0
- DHGR color uses 32 bytes per cell, four bytes per row, with the left
  pixel in the low nibble and native color bit order
- DLGR uses 32 bytes per cell, four rows of eight vertical pixel pairs
  Each byte holds the upper pixel in the low nibble and the lower pixel
  in the high nibble, alternating auxiliary and main bank colors from left
  to right

DHGR cells are repacked across the display's seven-bit bytes. Bits outside
the cells are preserved.

`GFX_CELL_XOR_MOVE OLD_SRC, OLD_CX, OLD_PY, NEW_SRC, NEW_CX, NEW_PY, W`
removes an XOR-drawn cell strip and draws its replacement in one call. The
source and position may both change. HGR uses specialized paths for
horizontal and vertical moves. Other moves use two XOR passes.

Code that already knows the movement axis can avoid the direction check with:

- `GFX_CELL_XOR_MOVE_HORIZONTAL OLD_SRC, OLD_CX, PY, NEW_SRC, NEW_CX, W`
- `GFX_CELL_XOR_MOVE_VERTICAL OLD_SRC, NEW_SRC, CX, OLD_PY, NEW_PY, W`

Each source contains `W` adjacent cells in the native format above.
`OLD_CX` and `NEW_CX` are 8-pixel cell columns. `OLD_PY` and `NEW_PY` must be
multiples of eight. The old strip must have been XOR-drawn with the same
source. HGR also uses the same foreground color. The call does not clip.

## Scrolling

`SCROLL_TYPE_SW` supports one native pixel steps in both directions in every
`DISPLAY` mode, including mixed graphics and 40 or 80 column text. Positive
movement moves the view right or down. Mixed mode keeps its text area fixed.

Use `@OPTION START_CODE $6000`. Scrolling uses both graphics pages and text
page 2 as working space. `SCROLL_BEGIN` keeps the existing picture and reserves
one hidden cell along each enabled axis for incoming content. Read
`SCROLL_BUFFER_COLUMNS`, `SCROLL_BUFFER_ROWS`, `SCROLL_VIEW_COLUMNS`, and
`SCROLL_VIEW_ROWS` after beginning.

Scroll cells are 7x8 native pixels in text and hires modes, and 8x8 in lores
modes. `SCROLL_CELL_PUTC` and `SCROLL_CELL_COLUMN` draw into the backing page;
`SCROLL_PRESENT` shows the movement and writes. The optional foreground color
uses the selected graphics mode's palette. Text scrolls with the built in
bitmap font; `SCROLL_CLOSE` restores native text with the updated characters.

Hires artifact colors depend on horizontal pixel position, so moving by an
odd number of pixels can change their appearance. Double hires color and
lores move whole color pixels. Software scrolling copies the picture and can
take more than one video frame, especially in double resolution modes.

## Images

Native image display supports:

| Format | Accepted files |
| --- | --- |
| `IMAGE_FMT_HGR` | HGR screen dumps, 8184/8188/8192 bytes, native row order, with or without a DOS binary header |
| `IMAGE_FMT_LGR` | 1024-byte lores text page dumps |
| `IMAGE_FMT_DLGR` | 2048 byte double lores dumps, auxiliary bank followed by main bank |
| `IMAGE_FMT_DHGR` | 16384 byte double hires dumps, auxiliary bank followed by main bank |

DLGR and DHGR require `apple2.e` or `apple2.c` with auxiliary video memory.
Both banks use native screen row order. DOS binary headers are accepted.
Lores loading preserves the unused memory slots in each screen bank.

For PNG or PCX conversion, select a format with `@INCLUDE_IMAGE ... AS`:

| Format | Size | Colors |
| --- | --- | --- |
| `a2-dlgr` | 80x48 | 16 |
| `a2-dhgr` | 140x192 | 16 |
| `a2-dhgr-mono` | 560x192 | 2 |

Raw DHGR dumps default to color. Use `AS a2-dhgr-mono` for monochrome.
Both use `IMAGE_FMT_DHGR`; the image auxiliary value is 0 for color and
1 for monochrome. `IMAGE_LOAD` selects the matching display mode.

Runtime image loading uses the selected file output.
Converted images use RLE8 automatically when it makes the image
smaller. This works for both embedded images and image files.

`HGR_IMAGE_BYTES` is 8204: an unpacked 8192 byte HGR screen plus the
12 byte crustyBASIC image header. Declare a buffer as
`PICTURE[HGR_IMAGE_BYTES - 1] AS U8`; array bounds are inclusive.
`DLGR_IMAGE_BYTES` is 2060 and `DHGR_IMAGE_BYTES` is 16396, also including
the 12 byte image header. Use `nodos` output for embedded unpacked DHGR
images to leave room for the 16 KB source buffer.

Image effects use `UNPACKED` images.
`IMAGE_WIPE` and `IMAGE_SLIDE` support unpacked HGR, DLGR, and DHGR images
in all four directions. Blocks preserve color alignment: 14x8 dots for HGR,
2x2 pixels for DLGR, 7x8 pixels for DHGR color, and 28x8 dots for DHGR monochrome.
`IMAGE_FADE_IN` and `IMAGE_FADE_OUT` use a seven-step patterned pixel fade
that preserves pixel colors. Fade out works on the active graphics screen.
DLGR and DHGR effects update both memory banks.

`IMAGE_DISSOLVE_IN` reveals scattered pixels from an unpacked HGR, DLGR,
or DHGR image. `IMAGE_DISSOLVE_OUT` erases the active graphics screen to
black. Both preserve pixel colors. Use `0` for the delay to run at full speed.
The optional final block size is 1, 2, 4, or 8 pixels per side, using
individual dots for HGR and DHGR monochrome.

## Hardware Names

Apple II softswitches trigger when accessed. Assigning any value is
enough for write-style switches:

```basic
SPEAKER.CLICK = 0
DISPLAY.TEXT_ON = 0
```

Common chip namespaces:

| Namespace | Useful names |
| --- | --- |
| `KBD` | `DATA`, `STROBE` |
| `SPEAKER` | `CLICK` |
| `MOCKINGBOARD` | `WRITE`, `RESET`, AY register constants |
| `DISPLAY` | `TEXT_ON`, `TEXT_OFF`, `MIXED_ON`, `MIXED_OFF`, `PAGE1`, `PAGE2`, `HIRES_ON`, `HIRES_OFF` |
| `GAME` | `SW0`, `SW1`, `SW2`, `PADDLE0`..`PADDLE3`, `STROBE` |
| `IIE` | `SET40COL`, `SET80COL`, `PRIMARYCHAR`, `ALTCHAR`, `RD80COL`, `RDALTCHAR` |

`IIE` is available on `apple2.e` and `apple2.c`.

## Input

Joystick reads expose a digital view of the game controls. Paddle reads
return the raw 0..255 timer value, and button reads use the Apple
pushbuttons.

`MOUSE_X` and `MOUSE_Y` read an AppleMouse card or the IIc mouse port.
`MOUSE_POSITION X, Y` sets its position; `MOUSE_BUTTON(0)` and bit 0 of
`MOUSE_STATE` read the button. The first mouse call searches slots 7 through 1
for mouse firmware. `APPLE_MOUSE_ENABLE SLOT` selects and initializes a
particular slot. Programs draw their own pointer.

On the original Apple II and II Plus, initialize the mouse before drawing:
the card's initialization clears hi-res page 1.

The IIc serial interface uses `APPLE_SERIAL_PRINTER` (port 1) and
`APPLE_SERIAL_MODEM` (port 2). `APPLE_SERIAL_OPEN PORT, BAUD` selects eight
data bits, one stop bit, and no parity. Baud selectors are
`APPLE_SERIAL_BAUD_300`, `APPLE_SERIAL_BAUD_1200`, `APPLE_SERIAL_BAUD_2400`,
`APPLE_SERIAL_BAUD_4800`, `APPLE_SERIAL_BAUD_9600`, and `APPLE_SERIAL_BAUD_19200`.

`APPLE_SERIAL_READY(PORT)` reports whether input is available.
`APPLE_SERIAL_READ_BYTE(PORT)` waits for input and returns one byte;
`APPLE_SERIAL_WRITE_BYTE PORT, BYTE` waits for the transmitter and sends it.
Both preserve all eight bits. `APPLE_SERIAL_CLOSE PORT` disables the port.

`APPLE_SERIAL_STATUS(PORT)` returns the hardware status. Test it with
`APPLE_SERIAL_PARITY_ERROR`, `APPLE_SERIAL_FRAMING_ERROR`,
`APPLE_SERIAL_OVERRUN`, `APPLE_SERIAL_RX_READY`, and `APPLE_SERIAL_TX_READY`.

Capability summary:

| Capability | Value |
| --- | --- |
| Keyboard | Yes |
| Joystick ports | 2 |
| Buttons | 2 per port with stacked wiring, otherwise 1 |
| Stick type | Analog |
| Paddle axes | 4 |
| Keypad ports | 0 |
| Keypad `INPUT` | No |
| Mouse buttons | `apple2.c`: 1, others: 0 |

`APPLE2_JOYSTICK_STACKED` defaults to `TRUE` for stacked adapter wiring:

| Port | Button 0 | Button 1 |
| --- | --- | --- |
| 0 | PB0 (`$C061`) | PB1 (`$C062`) |
| 1 | PB2 (`$C063`) | PB1 (`$C062`, shared) |

For wiring with one button per joystick, use:

```basic
@OPTION APPLE2_JOYSTICK_STACKED FALSE
```

This maps button 0 to PB0 on port 0 and PB1 on port 1, and sets
`JOY_BUTTONS` to 1. `INPUT_ANY` follows the selected wiring and checks
each switch at most once, respecting `INPUT_JOY_ENABLE`.

Select the wiring used by your adapter or emulator. The shared PB1 button
cannot be assigned to just one joystick. On the Apple IIe, the Apple keys
also activate PB0 and PB1. A Shift-key modification on an Apple II or II+
can occupy PB2 and interfere with the second joystick.

## Timing

By default, `TICKS` counts `FRAME_WAIT` calls at 60 Hz.

`@OPTION APPLE2_CLOCK_SLOT 4` selects a Mockingboard clock in slot 4.
Use the slot containing the card. The clock uses the first VIA's timer 1
and starts on the first `TICKS`, `TICKS_RESET`, or `TIMER_START` call.
It advances at 60 Hz without frame waits, and `TICKS_FREE_RUNNING` is
`TRUE`. `FRAME_COUNTER` continues to count frame waits separately.

With the clock selected, `TIMER_START` and `TIMER_STOP` measure elapsed
clock ticks in `TIMER_HI:TIMER_LO`. `TICKS_RESET` leaves that measurement
and periodic scheduling unchanged.

The clock enables interrupts and restores the previous timer and interrupt
settings when the program ends. Disabling interrupts delays clock updates;
several ticks can become one update during disk calls or other code that
keeps interrupts disabled.

On `apple2.e` and `apple2.c`, `FRAME_WAIT` synchronizes with vertical
blanking using the handler for that system. On `apple2.plus` and `apple2.int`,
`FRAME_WAIT` uses an approximate 60 Hz software delay.

## Sound

`SOUND_NOTE_BLOCKING` is `TRUE` for stock speaker notes on the II,
II Plus, and IIc. It is `FALSE` on the IIe, which queues notes for frame
waits, and after `MOCKINGBOARD_ENABLE` selects the sound card.

Portable sound uses the speaker until a sound card is enabled. A standard
Mockingboard provides two AY sound chips and six voices. Enable it with
the slot containing the card:

```basic
MOCKINGBOARD_ENABLE 4
```

Slots 1 through 7 are supported. The common slot is 4. The card is not
detected, so a program can ask the user before calling
`MOCKINGBOARD_ENABLE`. Calls made before it continue to use the speaker.

The stock speaker supports synchronous noise with
`PLAY_NOTE_SHAPE_FOR` and `SOUND_SHAPE_NOISE`. Other stock-speaker shapes
use a tone.

Apple II also provides the target-specific `TONE` call. `PERIOD` is the
speaker half period and `DURATION` is the number of full waves:

```basic
TONE PERIOD, DURATION
TONE PERIOD, DURATION, ASYNC
```

The two-argument form is synchronous. The three-argument form is also
synchronous when `ASYNC` is false. When `ASYNC` is true on the Apple //e,
the call queues the tone for available `FRAME_WAIT` idle time. The other
systems have no idle playback window, so the tone plays before the call
returns. Joystick sampling uses the same window, so queued tones should
not be used while polling the joystick.

After enabling the card, portable `SOUND_OFF`, `SOUND_ALL_OFF`,
`BEEP`, `PLAY_NOTE`, `PLAY_NOTE_SHAPE`, `PLAY_NOTE_FOR`, and
`PLAY_NOTE_SHAPE_FOR` use the card. Voices 0 through 2 use the first AY
and voices 3 through 5 use the second. Tone and noise shapes are
supported, with a per voice volume from 0 through 15. Exact AY periods
and other register effects use `MOCKINGBOARD.WRITE`.

On the card, `SOUND_ENVELOPE` supports ADSR and stage tables independently
on all six voices. `FRAME_WAIT` advances envelopes by one tick; duration
note calls wait in ticks too. `SOUND_RELEASE` starts the release stage.
`SOUND_OFF`, `SOUND_ALL_OFF`, and `MOCKINGBOARD.RESET` stop it immediately.

`SOUND_TIMED_NOTE` and `SOUND_TIMED_NOTE_SHAPE` start a note and return
immediately. `FRAME_WAIT` advances timed notes and rests by one tick,
before envelopes and the installed frame callback. Use voice 0 for the
speaker, or voices 0 through 5 after `MOCKINGBOARD_ENABLE`.
Speaker tones and noise play during the frame wait; the card keeps
playing between calls. `SOUND_TIMED_WAIT` and `SOUND_TIMED_WAIT_ALL` wait
while advancing frames. Remaining durations stay independent of
`TICKS_RESET`.

Starting an ordinary note on a voice cancels its timed note. `SOUND_OFF`,
`SOUND_ALL_OFF`, and `SOUND_TIMED_UNINSTALL` cancel playback; resetting or
selecting the card also clears timed notes. `TONE` and speaker `BEEP`
replace a timed speaker note.

`SOUND_VOICES` and `SOUND_TIMED_VOICES` describe the maximum runtime
interface, so both are 6 even before the card is enabled.
`SOUND_TIMED_FREE_RUNNING` is `FALSE`.

| Call | Purpose |
| --- | --- |
| `MOCKINGBOARD_ENABLE SLOT` | Select, initialize, and use the card for portable sound. |
| `MOCKINGBOARD.WRITE CHIP, REG, VALUE` | Write an AY register on chip 0 or 1. |
| `MOCKINGBOARD.RESET` | Silence and reset both AY chips. |

`MOCKINGBOARD.CHIPS`, `MOCKINGBOARD.VOICES`, and
`MOCKINGBOARD.VOLUME_MAX` describe the card. AY register constants are
`TONE_A_FINE`, `TONE_A_COARSE`, `TONE_B_FINE`, `TONE_B_COARSE`,
`TONE_C_FINE`, `TONE_C_COARSE`, `NOISE_PERIOD`, `MIXER`,
`AMPLITUDE_A`, `AMPLITUDE_B`, `AMPLITUDE_C`, `ENVELOPE_FINE`,
`ENVELOPE_COARSE`, and `ENVELOPE_SHAPE` under the same namespace.

Prepared audio uses the built-in speaker, including when a Mockingboard
is enabled. `@INCLUDE_AUDIO` converts mono or stereo PCM WAV files with
8 or 16 bit samples. Playback uses 16 volume levels through pulse widths;
8 kHz WAV files fit the speaker's playback timing.

`AUDIO_LOAD` accepts an embedded asset address or a disk filename. File
loads read the complete asset into RAM. `AUDIO_PLAY` returns when playback
finishes and takes over portable sound. `AUDIO_STOP` retains the asset for
replay, and `AUDIO_UNLOAD` releases its file buffer.

## Files

### DOS 3.3

The file provider calls the DOS 3.3 file manager. Run the program under
DOS 3.3 to use it.

Sequential read, write, append, update, and directory listing are supported.
Update mode reads and writes an existing file using one shared position.
A directory listing returns one catalog name per line.
File channels run from 0 to 15, and DOS 3.3 allows 3 files open at once.

`FILE_OPEN` uses DOS binary files and manages their address and length
header. Byte and string operations preserve all eight bits, including
zero. Files hold up to 65535 data bytes. Append starts at the stored
length, and write mode replaces the contents.

Use `FILE_OPEN_NATIVE` for DOS text files, with the mode in `AUX1` and
zero in `AUX2`. Text line reads remove bit 7 and stop at a zero byte;
text string writes set bit 7.

Generated companion files use binary headers. Text files placed in the
program's `.disk_files` folder keep DOS text format when their names end
in `.seq` or `.txt`.

`FILE_LOAD_PROGRAM` reads DOS binary files, removes the four-byte address
and length header, and loads the data without running it. The one argument
form uses the stored address; the destination form uses the supplied address.
Keep the destination clear of the running program, its variables, graphics,
and DOS memory.

`FILE_COMMAND` accepts the following constants. Pass zero for both auxiliary
arguments. Rename uses `"OLD,NEW"` as its path; the other commands take one
filename. The channel reports the result through `FILE_STATUS`.

| Constant | DOS command number |
|---|---|
| `FILE_COMMAND_DELETE` | 5 |
| `FILE_COMMAND_LOCK` | 7 |
| `FILE_COMMAND_UNLOCK` | 8 |
| `FILE_COMMAND_RENAME` | 9 |
| `FILE_COMMAND_VERIFY` | 12 |

Target-specific helpers are also callable: `APPLE_DOS_COMMAND`,
`APPLE_DOS_OPEN`, `APPLE_DOS_CLOSE`, `APPLE_DOS_READ_SELECT`,
`APPLE_DOS_WRITE_SELECT`, `APPLE_DOS_INPUT_RESTORE`,
`APPLE_DOS_OUTPUT_RESTORE`, `APPLE_DOS_READ_RAW_BYTE`, and
`APPLE_DOS_WRITE_RAW_BYTE`. These are DOS 3.3 text-file helpers.

End text records with byte `13` before closing or switching output.

### ProDOS

The `prodos` output supports read, write, append, update, and directory
channels through ProDOS. Channels run from 0 to 15, with eight files
open at once. Paths can be absolute or relative to the launched program's
directory. An empty directory path lists that directory, one name per line.

Byte and string operations preserve all eight bits, including zero.
Text lines end with byte `13`. Write mode replaces the file contents;
append starts at the file's exact length. Seek positions use 24 bits.

`FILE_OPEN_NATIVE` takes the mode in `AUX1` and the type for newly created
files in `AUX2`. `FILE_OPEN` creates type `BIN` (`$06`).
`FILE_LOAD_PROGRAM` loads a `BIN` file at its auxiliary address or a
`SYS` file at `$2000`; its destination form uses the supplied address.
Keep loaded data clear of the program, variables, graphics, and ProDOS.

`FILE_COMMAND` accepts the delete, lock, unlock, rename, and verify
constants listed above. Pass zero for both auxiliary arguments. Rename
takes `"OLD,NEW"`, with both paths in the same directory.

## REAL And Dialects

`REAL` uses Applesoft ROM math on `apple2.plus`, `apple2.e`, and
`apple2.c`. It is an error on `apple2.int`.

`applesoft_basic` follows Applesoft suffix rules: bare numeric names,
`!`, and `#` are `REAL`, `%` is 16-bit integer, and `$` is `STRING`.

`integer_basic` uses 16-bit integers for bare numeric names and `$` for
strings. Its `RND(N)` returns an integer from `0` through `N - 1`.

Both dialects leave page 3 (`$0300-$039F`) free for the program's own
machine code.

## Text Escapes

| Escape | Byte |
| --- | --- |
| `{CR}`, `{RETURN}`, `{ENTER}` | `$0D` |
| `{TAB}` | Four spaces |
