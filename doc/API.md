# CrustyBASIC API Reference

This page covers crustyBASIC runtime calls, portable helper libraries,
hardware access, and memory operations.
Core language syntax and builtins are documented in
[LANGUAGE.md](LANGUAGE.md), and command-line use is documented in
[USAGE.md](USAGE.md).

Portable runtime calls work on all targets unless noted. Optional
helper libraries list their own limits.

## Text

| Call                               | What it does                                                     |
| ---------------------------------- | ---------------------------------------------------------------- |
| `CLS()`                            | Alias of `CELL_CLS()`.                                           |
| `CLEAR_LINE()` / `CLEAR_LINE(row)` | Blank current or zero-based text row and leave cursor at start.  |
| `POSITION(col, row)`               | Move text cursor to zero-based col/row.                          |
| `PRINT_CENTER(row, s$[, width])`   | Print s$ centered on a text row; width defaults to `TEXT_WIDTH`. |
| `CURSORCOL()`                      | Current text cursor column (returns `U8`).                       |
| `CURSORROW()`                      | Current text cursor row (returns `U8`).                          |
| `CURSOR_HIDE()`                    | Hide the text cursor when the target has a visible one.          |
| `CURSOR_SHOW()`                    | Show the text cursor when the target has a visible one.          |
| `TEXT_COLOR(color)`                | Set the current `PRINT`/`INPUT` text color when supported.       |

`TEXT_COLOR` is always callable. It does nothing on targets without a
current text color, and `TEXT_COLOR_AVAILABLE` is `1` when the active
target implements it. Direct text cell access is separate:
`CELL_AVAILABLE` is `1` when raw cell APIs are meaningful.
`CELL_HAS_COLOR` reports whether `CELL_COLOR` has target support, and
`CELL_COLOR_MODEL` describes how those colors are stored. `CELL_COLOR_W`
and `CELL_COLOR_H` report color attribute granularity in cells, not
glyph pixel size. `CELL_MULTICOLOR_AVAILABLE` is `1` when
`DISPLAY(CELL_MULTICOLOR)` has target support. Cell attributes use
`CELL_HAS_ATTRIB`; `CELL_ATTRIB_W` and `CELL_ATTRIB_H` report their
granularity.

## Display

To use the portable display mode API, add `@INCLUDE "api/display.cbi"`.
Available display modes vary by target; the target docs under
[`targets/`](targets/) list the details.

`DISPLAY(mode)` accepts the portable mode constants listed below. Some
targets also accept target specific mode constants. When a target accepts
one, `DISPLAY` switches the hardware mode and updates the active mode
state used by `GFX_WIDTH`, `GFX_HEIGHT`, `GFX_BPP`,
`GFX_COLOR_COUNT`, `GFX_PLOT_COLOR_AVAILABLE`, `GFX_ROW_BYTES`, and
related calls.
Target specific mode constants are not portable; see the target docs
before using them.
Targets without display mode hardware accept `DISPLAY(CELL)` as a
no-op. Bitmap metadata queries return `0` unless a target display
provider sets them.

Prefer `DISPLAY(target_mode)` over setting a video mode with inline
assembly. Raw hardware mode changes bypass the active mode state that
APIs use for clipping, dimensions, color count, and bitmap metadata.

| Call                                              | What it does                                  |
| ------------------------------------------------- | --------------------------------------------- |
| `DISPLAY()`                                       | Switch to the target's default drawable display mode. Returns `1` if accepted, `0` if unavailable. |
| `DISPLAY(mode)`                                   | Switch to a specific display mode. Returns `1` if accepted, `0` if unsupported. |
| `DISPLAY_MIXED(mode, text_rows)`                  | Switch to graphics with a bottom text window. `text_rows = 0` requests full screen graphics. Returns `1` if accepted. |
| `GFX_MODE_COLOR_COUNT(mode)`                      | Portable color-value count for the mode, or `0` when not drawable. |
| `GFX_HAS_MODE(mode)`                              | `1` if the active target supports the mode.   |
| `GFX_ROW_BYTES()`                                 | Bytes per bitmap row, or `0` when unavailable.|
| `GFX_BITMAP_BASE()`                               | Bitmap memory base, or `0` when unavailable.  |
| `GFX_PAGE_COUNT()`                                | Number of selectable display pages. Usually `1`. |
| `GFX_PAGE_BASE(page)`                             | Primary memory base for a display page, or `0` when unavailable. |
| `GFX_PAGE_SELECT(page)`                           | Select a display page. Returns `1` if accepted. |
| `GFX_PAGE_ATTR_BASE(page)`                        | Attribute memory base for a display page, or `0` when unavailable. |

Some targets expose drawable video RAM that is not directly mapped into
CPU memory. For those modes, `GFX_BITMAP_BASE`, `GFX_PAGE_BASE`, and
`GFX_PAGE_ATTR_BASE` report the target's native video memory offset, not
a CPU address. Use `PLOT`, `POINT`, `GFX_CLS`, and target helpers to
access the surface.

`DISPLAY_MIXED` uses the same mode constants as `DISPLAY`. When a text
window is accepted, `GFX_HEIGHT` is the drawable graphics height above
the bottom text window. `DISPLAY_MIXED_AVAILABLE` is `1` when the target
supports this form. `DISPLAY_MIXED_VARIABLE_ROWS` is `1` when row counts
other than `0` and `DISPLAY_MIXED_DEFAULT_ROWS` can be used.
`DISPLAY_MIXED_MAX_ROWS` is the largest accepted bottom text window.

Active mode globals:

| Name                                      | What it means                                                |
| ----------------------------------------- | ------------------------------------------------------------ |
| `GFX_MODE`                                | Last mode accepted by `DISPLAY`.                             |
| `GFX_WIDTH` / `GFX_HEIGHT`                | Size of the active drawing space.                            |
| `GFX_BYTE_W` / `GFX_BYTE_H`               | Pixels per bitmap byte, or `0` outside bitmap modes.         |
| `GFX_BPP`                                 | Bits per stored pixel, or `0` outside packed bitmap modes.   |
| `GFX_COLOR_COUNT`                         | Color-value count for the active mode.                       |
| `GFX_COLOR_ATTR_W` / `GFX_COLOR_ATTR_H`   | Color attribute block size in pixels. Usually `1`.           |
| `GFX_PLOT_COLOR_AVAILABLE`               | `1` when three-argument `PLOT` can apply its color in the active mode. |

Modes: `CELL`, `CELL_MULTICOLOR`,
`BITMAP_HIRES`, `BITMAP_LORES`, and
`BITMAP_MULTICOLOR`.

## Graphics

To use the portable graphics drawing API, add `@INCLUDE "api/graphics.cbi"`.
It includes the display API automatically. Available drawing surfaces
vary by target.

| Call                                              | What it does                                  |
| ------------------------------------------------- | --------------------------------------------- |
| `PLOT(x, y)` / `PLOT(x, y, color)` / `UNPLOT(x, y)` | Set/clear a point. Color plot uses black background and falls back to `PLOT(x, y)` when color is unavailable. |
| `POINT(x, y)`                                     | Read a point. `0` means background/off; nonzero values are mode-specific. |
| `LINE(x0, y0, x1, y1)`                            | Arbitrary line.                               |
| `HLINE(x0, x1, y)` / `UNHLINE(x0, x1, y)`         | Horizontal span. Endpoints may be in either order. |
| `VLINE(x, y0, y1)` / `UNVLINE(x, y0, y1)`         | Vertical span. Endpoints may be in either order. |
| `BOX(x0, y0, x1, y1)` / `FILLBOX(x0, y0, x1, y1)` | Outline / filled rectangle.                   |
| `CIRCLE(xc, yc, r)` / `CIRCLE_E(xc, yc, rx, ry)`  | Circle / ellipse.                             |
| `GFX_CLS()`                                       | Clear the active graphics surface.            |
| `GFX_COLOR(fg)` / `GFX_COLOR(fg, bg)`             | Set colors. Single arg call uses black background. |
| `GFX_CAN_PLOT(mode)`                              | `1` if `PLOT`/`LINE`-style drawing is supported in the mode. |
| `GFX_CAN_POINT(mode)`                             | `1` if `POINT` readback is supported in the mode. |
| `GFX_BLIT(addr, cx, py, w)`                       | Copy pre-rendered bitmap bytes into `w` adjacent 8x8 cells. |
| `GFX_CELL_COLOR_ATTR(cx, cy, fg, bg)`             | Set one graphics color cell's attributes.     |
| `GFX_CELL_COLOR_ATTR_SPAN(cx, cy, w, fg, bg)`     | Set attributes for `w` color cells in a row.  |
| `GFX_CELL_COLOR_ATTR_ROW(cy, fg, bg)`             | Set attributes for a whole color cell row.    |

Coordinates are in the current graphic mode's space (pixels in bitmap modes, hardware cells in low-res).

`GFX_PLOT_COLOR_AVAILABLE` is active display state, not a compile-time
constant. Check it after `DISPLAY` before using three-argument `PLOT`.
`GFX_COLOR_ATTR_W` and `GFX_COLOR_ATTR_H` report whether that color is
per pixel or shared by a block. Targets with no color-capable plotting
mode also warn when three-argument `PLOT` is used.

`GFX_BLIT` draws pre-rendered bitmap data at cell granularity: `cx` is a
cell column, `py` a pixel row that must be a multiple of 8, and the
source holds `w` cells of 8 bytes each in cell-column major order (all
8 rows of the leftmost cell, then the next cell to its right). Bytes
are in the active bitmap mode's native pixel format and nothing is
clipped. Smooth sub-cell motion is an application technique: keep
pre-shifted copies of the pattern and blit the aligned window that
contains it. `GFX_BLIT_AVAILABLE` is `1` on targets that implement it;
elsewhere the call is a no-op.

`GFX_CELL_COLOR_ATTR` and its span/row variants set color attributes on
the active bitmap surface. A graphics color cell is one block sized by
`GFX_COLOR_ATTR_W` and `GFX_COLOR_ATTR_H`. These calls are distinct from
the text API's `CELL_COLOR`, which colors the text screen. On targets
without writable graphics color cells (`GFX_CELL_COLOR_ATTR_AVAILABLE`
is `0`) the single cell and row calls do nothing.

Portable color constants:

`BLACK` `WHITE` `RED` `CYAN`
`PURPLE` `GREEN` `BLUE`
`YELLOW` `ORANGE` `BROWN`
`LIGHT_RED` `DARK_GRAY` `GRAY`
`LIGHT_GREEN` `LIGHT_BLUE` `LIGHT_GRAY`

`GFX_STD_COLORS` is the same 16 colors as a `U8` const array. Use
`LEN(GFX_STD_COLORS)` for its element count.

`GFX_NATIVE_COLOR(idx)` returns the target specific byte for an index
in the active mode.

## Misc

| Name             | What it does                                                     |
| ---------------- | ---------------------------------------------------------------- |
| `BEEP(duration)` | Play a target specific tone. Duration units are target specific. |
| `DELAY(n)`       | Pause for roughly target specific units.                         |

## Sound

The sound interface has a portable note layer and a native target layer.
`PLAY_NOTE` and the other note calls are portable: `VOICE` is zero based,
`NOTE` uses the `NOTE_*` constants from `NOTE_C2` through `NOTE_B6`, and
volume uses `0` through `SOUND_VOLUME_MAX`. `NOTE_REST` silences or
reserves the voice.

`SOUND` and `SOUND_RAW` are native target calls. Their four argument
form is shared, but the meaning and valid range of every argument are
target specific. Depending on the target, the second argument may be a
frequency, period, divider, or pitch value, while the third may select a
waveform, control value, or distortion. These calls start or update
sound immediately and do not include a duration. See the active target's
documentation for its exact values.

`SOUND_FREQ` also returns a target specific representation. Some targets
pack additional sound control into its `U16` result, so treat it as an
opaque value for that target rather than a frequency in common units.

| Name | What it does |
| --- | --- |
| `SOUND_AVAILABLE` | `1` when the target provides sound. |
| `SOUND_VOICES` | Number of target sound voices. |
| `SOUND_VOLUME_MAX` | Highest volume accepted by the portable note calls. |
| `SOUND_HAS_ENVELOPE` | `1` when target sound exposes envelopes. |
| `SOUND_HAS_FILTER` | `1` when target sound exposes a filter. |
| `SOUND_HAS_PULSEWIDTH` | `1` when target sound exposes pulse width. |
| `SOUND_HAS_SHAPE` | `1` when target sound exposes waveform or noise shape selection. |
| `SOUND_HAS_NOISE` | `1` when target sound exposes a noise shape or noise channel. |
| `SOUND_HAS_NATIVE` | `1` when target sound uses native frequency values. |
| `SOUND_VOLUME_MODEL` | Volume behavior, one of the `SOUND_VOLUME_MODEL_*` values. |
| `SOUND_NOTE_MIN` | Lowest MIDI note number supported by target sound. |
| `SOUND_NOTE_MAX` | Highest MIDI note number supported by target sound. |
| `SOUND_FREQ(note)` | Convert a note constant to the target's native sound value. |
| `SOUND(voice, value, control, volume)` | Start or update sound using target specific arguments. |
| `SOUND_RAW(voice, value, control, volume)` | The same native operation as `SOUND`, with an explicit raw name. |
| `SOUND_OFF(voice)` | Stop one sound voice. |
| `SOUND_ALL_OFF()` | Stop all target sound. Pure CrustyBasic also accepts `SILENCE`. |
| `PLAY_NOTE(voice, note, volume)` | Start a note immediately. |
| `PLAY_NOTE_SHAPE(voice, note, shape, volume)` | Start a note immediately with a portable shape. |
| `PLAY_NOTE_FOR(voice, note, volume, duration)` | Play a blocking note for target delay units. |

Timed sound schedules a voice and returns immediately where the target
supports it. Durations use the same units as `TICKS`: `TICKS_HZ` ticks
per second. The scheduler uses its own counters, so `TICKS_RESET` does
not affect active timed sounds.

| Name | What it does |
| --- | --- |
| `SOUND_TIMED_AVAILABLE` | `1` when timed sound scheduling is available. |
| `SOUND_TIMED_FREE_RUNNING` | `1` when timed sound advances from an interrupt or OS clock. |
| `SOUND_TIMED_VOICES` | Number of schedulable timed voices. |
| `SOUND_TIMED_INIT()` | Enable the timed sound scheduler. Timed play calls do this lazily. |
| `SOUND_TIMED_UNINSTALL()` | Disable the scheduler and stop timed voices. |
| `SOUND_TIMED_SERVICE()` | Advance non-free-running schedulers. No-op when `SOUND_TIMED_FREE_RUNNING = 1`. |
| `SOUND_TIMED_NOTE(voice, note, volume, ticks)` | Schedule a note for `ticks`. |
| `SOUND_TIMED_NOTE_SHAPE(voice, note, shape, volume, ticks)` | Schedule a shaped note for `ticks`. |
| `SOUND_TIMED_RAW(voice, freq, control, volume, ticks)` | Schedule target specific raw sound for `ticks`. |
| `SOUND_TIMED_REST(voice, ticks)` | Reserve a voice silently for `ticks`. |
| `SOUND_TIMED_BUSY(voice)` | `1` while a voice has a note or rest scheduled. |
| `SOUND_TIMED_REMAINING(voice)` | Remaining ticks for the voice. |
| `SOUND_TIMED_WAIT(voice)` | Wait until the voice is no longer busy. |
| `SOUND_TIMED_WAIT_ALL()` | Wait until all timed voices are no longer busy. |
| `SOUND_TIMED_OFF(voice)` | Cancel and silence one timed voice. |
| `SOUND_TIMED_OFF_ALL()` | Cancel and silence all timed voices. |

Use `@IF SOUND_TIMED_AVAILABLE THEN` or
`@REQUIRES SOUND_TIMED_AVAILABLE` when a program depends on scheduled
sound. On targets without a timed provider these calls compile as no-ops.

## Timing

| Name                    | What it does                                                    |
| ----------------------- | --------------------------------------------------------------- |
| `TIMER_TYPE`            | Timer source kind, one of the `TIMER_*` values below.           |
| `TIMER_AVAILABLE`       | `1` when `TIMER_TYPE <> TIMER_NONE`, otherwise `0`.             |
| `TIMER_HIGH_RES`        | `1` for sub-frame raw ticks, `0` for coarse target ticks.       |
| `TIMER_START()`         | Start or reset the elapsed timer.                               |
| `TIMER_STOP()`          | Stop the timer and store elapsed ticks in `TIMER_HI:TIMER_LO`.  |
| `TIMER_HI`              | High 16 bits of the elapsed tick count.                         |
| `TIMER_LO`              | Low 16 bits of the elapsed tick count.                          |
| `TIMER_NONE`            | No elapsed timer provider.                                      |
| `TIMER_FREE_RUNNING_HW` | `TIMER_START` samples a free running hardware counter.          |
| `TIMER_FREE_RUNNING_SW` | `TIMER_START` samples a free running software or ROM counter.   |
| `TIMER_START_STOP_HW`   | `TIMER_START` starts a hardware counter for measurement.        |
| `TIMER_START_STOP_SW`   | `TIMER_START` starts a software counter for measurement.        |

Raw tick units are target specific. Compare `TIMER_HI` first, then
`TIMER_LO`; lower elapsed ticks means faster. Use
`@IF TIMER_TYPE <> TIMER_NONE THEN` when a program needs a timer
fallback.

## Frames

Frame pacing is a runtime API: each target provides its vsync wait,
and the constants below gate support.

| Name                        | What it does                                                  |
| --------------------------- | ------------------------------------------------------------- |
| `FRAME_WAIT()`              | Block until the next frame and advance `FRAME_COUNTER`.       |
| `FRAME_WAIT(count)`         | Wait `count` frames.                                          |
| `FRAME_COUNTER`             | `U16` frame count advanced by `FRAME_WAIT` or an installed frame provider. |
| `FRAME_AVAILABLE`           | `1` when the target has a frame provider.                     |
| `FRAME_VBI_AVAILABLE`       | `1` when the target has a vector hookable VBI.                |
| `FRAME_INSTALL_AVAILABLE`   | `1` when `FRAME_INSTALL` can run a per-frame hook.            |
| `FRAME_INSTALL_INTERRUPT`   | `1` when the installed hook runs from an interrupt.           |
| `FRAME_INSTALL_SYNTHESIZED` | `1` when the installed hook runs during `FRAME_WAIT`.         |
| `FRAME_INSTALL(addr)`       | Install a PROC address to run once per frame.                 |
| `FRAME_UNINSTALL()`         | Remove the installed per-frame hook.                          |

Targets without a frame provider get defaults: `FRAME_WAIT` just
increments `FRAME_COUNTER` without waiting, and the hook calls are
inert. Guard frame-paced code with `@IF FRAME_AVAILABLE THEN` (or
`@REQUIRES FRAME_AVAILABLE`).

## Ticks

A free-running elapsed-tick counter, available on every target.
`TICKS_RESET` latches the system counter into a hidden base;
`TICKS` returns the ticks elapsed since the last reset. The system
clock itself is never written.

| Name                 | What it does                                                    |
| -------------------- | --------------------------------------------------------------- |
| `TICKS`              | `U32` ticks elapsed since the last `TICKS_RESET`.               |
| `TICKS_RESET()`      | Restart the elapsed count from zero.                            |
| `TICKS_HZ`           | Nominal ticks per second for the target.                        |
| `TICKS_FREE_RUNNING` | `1` when an interrupt or OS clock drives the count, `0` when it only advances during explicit frame wait or draw calls. |

Tick sources, rates, and caveats are target-specific. See the target
docs under [`targets/`](targets/) when a program needs exact timing.

## Raster interrupts

Some targets provide a portable raster-interrupt API.

- `RASTER_CLEAR()` - clear the list of marked display rows.
- `RASTER_MARK_ROW(row)` - mark one visible display row.
- `RASTER_MARK_ROWS(first, count, step)` - mark several visible rows.
- `RASTER_MARK_OFFSET(offset)` - mark a raw display-list byte offset.
- `RASTER_LINE_INSTALL(line_addr)` - install a no-argument PROC address
  for marked display rows.
- `RASTER_FRAME_INSTALL(frame_addr)` - install a no-argument PROC address
  for per-frame work.
- `RASTER_OFF()` - disable raster interrupts and clear row marks.
- `RASTER_WSYNC()` - wait for the current scanline boundary.
- `RASTER_INDEX` - `U8` line counter reset before the frame hook and
  incremented after each line hook.

The installed PROCs must be no-argument `@ASYNC` PROCs.
Target docs contain hardware-specific raster examples and timing notes.

## Files

The portable file API uses logical channels and byte streams. A target
provider maps those channels to its native file system, ROM calls, host
handles, or other I/O layer.

Targets without a provider keep the default compile time `@ERROR`.
Target docs list available providers and native argument meanings.

| Name                    | Type | What it means                                  |
| ----------------------- | ---- | ---------------------------------------------- |
| `FILE_MODE_READ`        | `U8` | Open for reading.                              |
| `FILE_MODE_WRITE`       | `U8` | Open for writing.                              |
| `FILE_MODE_APPEND`      | `U8` | Open for appending.                            |
| `FILE_MODE_UPDATE`      | `U8` | Open for reading and writing.                  |
| `FILE_MODE_DIR`         | `U8` | Open a directory style listing when supported. |
| `FILE_OK`               | `U8` | No error.                                      |
| `FILE_EOF`              | `U8` | End of file.                                   |
| `FILE_UNSUPPORTED`      | `U8` | Operation is not supported.                    |

| Call                                        | What it does                                      |
| ------------------------------------------- | ------------------------------------------------- |
| `FILE_OPEN(channel, path$, mode)`           | Open a path on a logical channel.                 |
| `FILE_OPEN_NATIVE(channel, path$, a1, a2)`  | Open with target native aux/device values.        |
| `FILE_CLOSE(channel)`                       | Close a logical channel.                          |
| `FILE_READ_BYTE(channel)`                   | Read one byte and return `U8`.                    |
| `FILE_WRITE_BYTE(channel, byte)`            | Write one byte.                                   |
| `FILE_READ_LINE(channel, out line$)`        | Read one text record into a string.               |
| `FILE_WRITE_STR(channel, text$)`            | Write a string without adding a newline.          |
| `FILE_WRITE_DATA(channel, value)`           | Write one `RESTORE FILE` item to an open file.    |
| `FILE_LOAD(channel, path$, dst, count)`     | Load `count` bytes from a path into memory.       |
| `FILE_SAVE_BYTES(channel, path$, src, count)` | Save `count` bytes from memory to a path.       |
| `FILE_STATUS(channel)`                      | Return the target status byte for the channel.    |
| `FILE_COMMAND(cmd, channel, a1, a2, path$)` | Run a target native file command.                 |

`RESTORE FILE channel, path$, ADDR buffer, count` loads `count` bytes
from `path$` into `buffer`, then makes later `READ` statements consume
that buffer instead of compiled-in `DATA`. The file must already be in
the raw DATA stream format: each item is one length byte followed by the
item text bytes. Numeric items are stored as their decimal text, not as
native binary integers.

Use `FILE_WRITE_DATA` on a file opened for writing to save values in
that format:

```basic
BUF(8) AS U8
A AS U8
S$ AS STRING * 8

FILE_OPEN 1, "DATA.BIN", FILE_MODE_WRITE
FILE_WRITE_DATA 1, U8(55)
FILE_WRITE_DATA 1, "HELLO"
FILE_CLOSE 1

RESTORE FILE 1, "DATA.BIN", ADDR BUF, 9
READ A
READ S$
```

After that example, `A` is `55` and `S$` is `"HELLO"`. A later bare
`RESTORE` switches `READ` back to the program's compiled `DATA` pool.
Use `FILE_SAVE_BYTES` only when you have already built the raw byte
stream yourself.

Program file commands are used by dialects and can also be used directly.

| Call                         | What it is for                           |
| ---------------------------- | ---------------------------------------- |
| `FILE_SAVE_PROGRAM(path$)`   | Dialect `SAVE` style command.            |
| `FILE_LOAD_PROGRAM(path$)`   | Dialect `LOAD` style command.            |
| `FILE_LIST_PROGRAM(path$)`   | Dialect `LIST path` style command.       |
| `FILE_ENTER_PROGRAM(path$)`  | Dialect `ENTER` style command.           |
| `FILE_RUN_PROGRAM(path$)`    | Dialect `RUN "path"` style command.      |

`FILE_OPEN_NATIVE` arguments are target specific. See the target docs
when a program needs explicit device, secondary address, drive, or ROM
mode values.

## External Storage

To use the portable external storage API, add
`@INCLUDE "api/external_storage.cbi"`.

This API is for fast target-specific storage providers that are separate
from the normal logical channel file API. It is currently backed by the
Ultimate Command Interface where available. One file is active at a
time.

| Name / call | What it does |
| --- | --- |
| `FILES_AVAILABLE()` | `1` when external storage is available. |
| `FILES_MODE_READ` / `FILES_MODE_WRITE` / `FILES_MODE_CREATE` | Open modes. |
| `FILES_OK` / `FILES_ERR` | Result constants. |
| `FILES_OPEN(path$, mode)` | Open one path. Returns `FILES_OK` on success. |
| `FILES_CLOSE()` | Close the current file. |
| `FILES_WRITE(data$)` | Write a string to the current file. |
| `FILES_READ_REQUEST(maxlen)` | Request a read chunk. |
| `FILES_HAS_DATA()` | `1` while the read stream has bytes. |
| `FILES_READ_BYTE()` | Read one byte from the current read stream. |
| `FILES_READ_END()` | Finish the current read chunk. |
| `FILES_DELETE(path$)` | Delete a path. |
| `FILES_CHDIR(path$)` | Change directory where supported. |
| `FILES_MKDIR(path$)` | Create a directory where supported. |
| `FILES_MOUNT(path$)` | Mount media where supported. |

## Networking

To use the portable networking API, add `@INCLUDE "api/net.cbi"`.

Network support is optional. Targets without a provider return `0` from
`NET_AVAILABLE()` and fail connection/listen calls.

| Name / call | What it does |
| --- | --- |
| `NET_TCP` / `NET_UDP` | Protocol constants. |
| `NET_AVAILABLE()` | `1` when networking is available. |
| `NET_GET_IP(out ip$)` | Store the active interface IP address, or an empty string. |
| `NET_CONNECT(proto, host$, port)` | Open a TCP or UDP socket. Returns a socket handle, or `0`. |
| `NET_CLOSE(sock)` | Close a socket. |
| `NET_WRITE(sock, data$)` | Write a string to a socket. |
| `NET_READ_REQUEST(sock, maxlen)` | Request a read chunk from a socket. |
| `NET_HAS_DATA()` | `1` while the read stream has bytes. |
| `NET_READ_BYTE()` | Read one byte from the current read stream. |
| `NET_READ_END()` | Finish the current read chunk. |
| `NET_LISTEN(port)` | Start a TCP listener. Returns a listener handle, or `0`. |
| `NET_ACCEPT(listener)` | Accept a pending TCP client. Returns a socket handle, or `0`. |
| `NET_CLOSE_CLIENT(sock)` | Close an accepted client while keeping the listener where supported. |

## UCI

To check for a C64 Ultimate or 1541 Ultimate-II+ provider, add
`@INCLUDE "api/uci.cbi"`.

| Call | What it does |
| --- | --- |
| `UCI_AVAILABLE()` | `1` when the Ultimate Command Interface is reachable. |

## FujiNet

FujiNet support is target provided. Targets with a provider expose
`FUJINET_AVAILABLE`, a `FUJINET_TRANSPORT` value, and the raw command
and shared buffer API.

| Name / call | What it does |
| --- | --- |
| `FUJINET_TRANSPORT_NONE` / `FUJINET_TRANSPORT_SIO` / `FUJINET_TRANSPORT_DRIVEWIRE` | Transport constants. |
| `FUJINET_STATUS_OK` / `FUJINET_STATUS_IO_ERROR` / `FUJINET_STATUS_UNSUPPORTED` | Status constants. |
| `FUJINET_DIR_NONE` / `FUJINET_DIR_READ` / `FUJINET_DIR_WRITE` | Command direction constants. |
| `FUJINET_BUFFER_MAX` | Maximum shared buffer size. |
| `FUJINET_STATUS()` | Return the current adapter status. |
| `FUJINET_COMMAND(device, command, aux1, aux2, direction, count)` | Send one transport native command. |
| `FUJINET_EXCHANGE(device, command, aux1, aux2, direction, write_count, read_count)` | Send request bytes and optionally read reply bytes. |
| `FUJINET_BUFFER_CLEAR()` | Clear the shared command buffer. |
| `FUJINET_BUFFER_SET_LEN(count)` / `FUJINET_BUFFER_LEN()` | Set or read the active buffer length. |
| `FUJINET_BUFFER_POKE(offset, value)` / `FUJINET_BUFFER_PEEK(offset)` | Write or read one shared buffer byte. |
| `FUJINET_BUFFER_WRITE_STRING(data$)` / `FUJINET_BUFFER_READ_STRING(out data$)` | Write or read the buffer as a string. |

The include also exports FujiNet device and command constants for raw
protocol use.

## Image

The portable image API displays each platform's native image formats at
native resolution: the build tool wraps a native screen dump in a small
descriptor, and `IMAGE_DISPLAY` sets the video mode and copies the
payload straight into video memory. It is intended for splash/loading
screens, game backgrounds, and static art; hardware sprite state is left
untouched so sprites can move over a displayed background.

See the `## Images` section in each image capable target page under
[`targets/`](targets/) for the image formats supported by that target.

Generated includes define one label per image:

```basic
CONST TITLE_IMAGE(...) AS U8 = ...

OK = IMAGE_DISPLAY(ADDR TITLE_IMAGE)
```

The compiler can generate the descriptor at compile time:

```basic
@INCLUDE_IMAGE TITLE "src/title.png"
@INCLUDE_IMAGE TITLE "TITLE.IMG" "src/title.png"
```

The first form emits `CONST TITLE_IMAGE(...) AS U8 = ...`. The second
form also emits `CONST TITLE_IMAGE_FILE = "TITLE.IMG"` and writes
`TITLE.IMG` beside the build output so `IMAGE_LOAD_DISPLAY` can load
it at runtime. If the source is already a `.img` descriptor, the
compiler validates it for the active target and reuses the bytes.

On targets with target file loading, `IMAGE_LOAD_DISPLAY` also accepts
target image files. C64 accepts Koala Painter files.

Format constants such as `IMAGE_FMT_*` identify native descriptor
payloads. The converter strips file containers, expands compression
when needed, and stores payloads in the target's native video-memory
dump order, so every runtime blit is a straight copy. The target image
sections list accepted file formats, native image sizes, and format IDs.

Calls:

| Call | What it does |
| --- | --- |
| `IMAGE_DISPLAY(src_addr)` | Set the format's video mode and blit the payload. Returns `1` if accepted. |
| `IMAGE_CLEAR` | Clear available graphics, tile, and cell surfaces. |
| `IMAGE_WIDTH(src_addr)` / `IMAGE_HEIGHT(src_addr)` | Native pixel size. |
| `IMAGE_FORMAT(src_addr)` | The `IMAGE_FMT_*` id. |
| `IMAGE_AUX(src_addr)` | Format-specific extra byte. |
| `IMAGE_LOAD_DISPLAY(path$)` | Open an image file and stream it straight into video memory when `IMAGE_FILE_AVAILABLE` is `1`. Returns `1` on success. |

Capability names:

| Name | What it means |
| --- | --- |
| `IMAGE_AVAILABLE` | Native image display is supported on this target. |
| `IMAGE_FILE_AVAILABLE` | `IMAGE_LOAD_DISPLAY` can stream image files from disk. |
| `IMAGE_TARGET_FILE_AVAILABLE` | `IMAGE_LOAD_DISPLAY` can load target image file formats directly. |

Converter usage:

```sh
tools/bin/cb-image --target TARGET --name TITLE input.png -o title.cbi
tools/bin/cb-image --target TARGET input.png --img -o title.img
```

Most image-capable targets also accept indexed PNG and PCX sources up
to the native resolution. Source colors map to the nearest native
palette entry and per-cell hardware constraints are satisfied by
majority reduction. Sources larger than the native mode are an error -
scale them down first.

`--as` forces a format when the input is ambiguous. `--aux` overrides
the descriptor aux byte. `--img` writes the raw descriptor bytes as a
binary instead of a `.cbi` include.

The descriptor layout (version 2) is a 12-byte header: magic `73`,
version, format id, aux, width/height as `u16le`, reserved, payload
length as `u16le`, followed by the payload.

## Cell

The cell API writes raw text/screen cells. It is useful for tile
games and custom character sets.

| Call                                           | What it does                                  |
| ---------------------------------------------- | --------------------------------------------- |
| `CELL_DRAW(x, y, w, h, addr)`                  | Copy a packed block of cells to the screen.   |
| `CELL_MEMMOVE(src_x, src_y, dst_x, dst_y, count)` | Move a contiguous raw cell range within the screen. |
| `CELL_SCROLL_ROW(row, x, w, count, fill, direction)` | Scroll one row segment left or right. |
| `CELL_SCROLL(x, y, w, h, count, fill, direction)` | Scroll a rectangle left, right, up, or down. |
| `CELL_CLS()`                                   | Clear the text screen and home cursor.       |
| `CELL_GETC(x, y)`                              | Read one cell when the target can.            |
| `CELL_PUTC(x, y, c)`                           | Write one raw cell.                           |
| `CELL_PUTC(x, y, c, color, attr)`              | Write one raw cell, color, and attributes.    |
| `CELL_PRINT(x, y, s$)`                         | Write a string starting at one cell.          |
| `CELL_PRINT(x, y, s$, color, attr)`            | Write a string, color, and attributes.        |
| `CELL_CODE(c)` / `CELL_CODE(s$)`               | Convert a printable byte or string to a raw cell code. |
| `CELL_COLOR(fg, bg)`                           | Set default/shared cell foreground and background colors. |
| `CELL_COLOR(fg, bg, color2, color3)`           | Set four cell color slots where supported.    |
| `CELL_COLOR(x, y, color)`                      | Set cell color where the target supports it.  |
| `CELL_ATTRIB(x, y, attr)`                      | Set cell attributes where supported.          |
| `CELL_FLUSH()`                                 | Push staged writes; no-op on direct targets.  |

Cell values are target specific screen codes. Color and attribute
granularity are also target specific.
Use `CELL_PUTC x, y, CELL_CODE("O")` for a printable character, or
`CELL_PRINT x, y, "HELLO"` for a string. The five argument
`CELL_PUTC` overload sets color, writes the cell, then applies
attributes. `CELL_PRINT` uses the same order for each cell, which is
the portable order when attributes share storage with the cell code.
In `CELL_MULTICOLOR` modes such as Plus/4 multicolor text,
`CELL_COLOR fg, bg, color2, color3` sets the default per-cell
foreground, shared background, and two shared multicolor slots.
`CELL_COLOR x, y, color` sets that cell's foreground color; the other
three colors are shared.

Cell capability constants:

| Constant | Meaning |
| --- | --- |
| `CELL_AVAILABLE` | Raw cell/charmap APIs are meaningful. |
| `CELL_HAS_COLOR` | `CELL_COLOR` has target support. |
| `CELL_COLOR_MODEL` | How the target stores cell color. |
| `CELL_COLOR_W` / `CELL_COLOR_H` | Color attribute granularity in cells. |
| `CELL_MULTICOLOR_AVAILABLE` | `DISPLAY(CELL_MULTICOLOR)` is supported. |
| `CELL_HAS_ATTRIB` | `CELL_ATTRIB` has target support. |
| `CELL_ATTRIB_W` / `CELL_ATTRIB_H` | Attribute granularity in cells. |

Cell color model constants:

| Constant | Meaning |
| --- | --- |
| `CELL_COLOR_MODEL_NONE` | No portable cell color. |
| `CELL_COLOR_MODEL_SHARED` | Shared/global cell colors only. |
| `CELL_COLOR_MODEL_PER_CELL_FG` | Per-cell foreground with shared background. |
| `CELL_COLOR_MODEL_PER_CELL_FG_BG` | Per-cell foreground/background pair. |
| `CELL_COLOR_MODEL_PER_CELL_PALETTE` | Per-cell palette or color-set selection. |
| `CELL_COLOR_MODEL_BLOCK_PALETTE` | Palette or color-set selection shared by a cell block. |

Current attributes are `NORMAL`, `INVERSE`, `ITALIC`, and `BLINKING`.
Scroll directions are `CELL_SCROLL_LEFT`, `CELL_SCROLL_RIGHT`,
`CELL_SCROLL_UP`, and `CELL_SCROLL_DOWN`. `CELL_MEMMOVE` works on a
physically contiguous range from the source cell, so use it for a
single row or for target specific layouts where the range is known to
be contiguous.

## Charset

Targets with writable text character sets expose `CHARSET_AVAILABLE =
1`. Portable programs that require this feature should start with:

```basic
@REQUIRES CHARSET_AVAILABLE @ELSE "programmable charset support is required"
```

| Call | What it does |
| --- | --- |
| `CHARSET_COPY_DEFAULT(addr)` | Copy the target's default font into RAM. |
| `CHARSET_INSTALL(addr)` | Select the RAM charset surface. Does not copy font bytes. |
| `CHARSET_DEFINE(code, addr)` | Copy one glyph into a raw charset slot. |
| `CHARSET_DEFINE(s$, addr)` | Copy one glyph into the slot for the first printable character in `s$`. |
| `CHARSET_RESET()` | Restore the target's default charset. |

Call `CHARSET_COPY_DEFAULT(addr)` before `CHARSET_INSTALL(addr)` when
you want to keep the normal glyphs and override only a few entries.
`CHARSET_DEFINE("!", ADDR GLYPH)` is equivalent to
`CHARSET_DEFINE(CELL_CODE("!"), ADDR GLYPH)` and returns the same value
as `CELL_CODE("!")`. Use the numeric form for raw slots that do not
have a printable character.

Charset shape constants describe the selected surface:

| Constant | Meaning |
| --- | --- |
| `CHARSET_NUM_ENTRIES` | Number of entries in the charset surface. |
| `CHARSET_BYTES_PER_ENTRY` | Bytes per glyph entry. |
| `CHAR_PIXEL_WIDTH` | Glyph width in pixels. |
| `CHAR_PIXEL_HEIGHT` | Glyph height in pixels. |

## Tile

To use the portable tile API, add `@INCLUDE "tile.cbi"`. TILE draws a
playfield on whatever the target supports: text cells, graphics, or a
tile display. Coordinates are tile positions, not pixels.

```basic
@INCLUDE "tile.cbi"

DISPLAY TILE_DEFAULT_DISPLAY_MODE
TILE_BEGIN 2, 2
TILE_ALIAS 0, " "
TILE_ALIAS 1, "#"
TILE_CLEAR_TILE 0
TILE_BOX 0, 0, TILE_COLUMNS - 1, TILE_ROWS - 1, 1
```

Most programs should select a display mode, then call
`TILE_BEGIN cell_w, cell_h`.
`cell_w` and `cell_h` are the size of one tile in normal text cells.
Use `1, 1` for single-character tiles, or larger values for block tiles.

`TILE_BEGIN(cell_w, cell_h, base)` is for targets that need a
character or tile memory address. Omit `base` unless the target docs ask
for it.

The drawing method (the backend) is fixed when the program compiles:

| Name          | Meaning                                      |
| ------------- | -------------------------------------------- |
| `TILE_CELL` | Draw through the target text/cell screen. |
| `TILE_NATIVE` | Draw through target tile hardware such as a nametable or display list. |
| `TILE_BITMAP` | Draw tiles on a graphics bitmap. |
| `TILE_KERNEL` | Use a target-specific tile display. |

`@OPTION TILE_BACKEND TILE_BITMAP` picks the backend from source. On the
command line use `--set tile-backend=cell`, `--set tile-backend=native`,
`--set tile-backend=bitmap`, or `--set tile-backend=kernel`. If it is
not set, the target's preferred backend is used. Only the selected
backend's drawing code links into the program, and the read-only
`TILE_BACKEND` constant reports the choice. Unsupported choices are
compile errors.
`TILE_BEGIN` does not change the display mode. Use
`DISPLAY TILE_DEFAULT_DISPLAY_MODE` for the selected tile backend's
portable default, or call `DISPLAY` with a target-specific mode before
`TILE_BEGIN`.

Fixed tile sizes can be declared as source optimization hints:

```basic
@OPTION TILE_FIXED_CELL_W 1
@OPTION TILE_FIXED_CELL_H 1
```

These hints are optional. Targets may use them to select smaller or
faster TILE helpers, and targets may ignore them. Use them only when
every `TILE_BEGIN` call uses that cell size. If the hint and the runtime
tile size disagree, target-specific optimized TILE paths may draw the
wrong cells.

Tile drawing:

| Call                                  | What it does                                |
| ------------------------------------- | ------------------------------------------- |
| `TILE_PLOT(x, y, tile)`               | Draw one logical tile.                      |
| `TILE_CLEAR(x, y)`                    | Draw the selected clear tile.               |
| `TILE_PRINT(x, y, text$)`             | Draw text starting at a tile position.      |
| `TILE_HLINE(x0, x1, y, tile)`         | Draw a horizontal tile run.                 |
| `TILE_UNHLINE(x0, x1, y)`             | Clear a horizontal tile run.                |
| `TILE_VLINE(x, y0, y1, tile)`         | Draw a vertical tile run.                   |
| `TILE_UNVLINE(x, y0, y1)`             | Clear a vertical tile run.                  |
| `TILE_BOX(x0, y0, x1, y1, tile)`      | Draw a tile rectangle outline.              |
| `TILE_FILLBOX(x0, y0, x1, y1, tile)`  | Fill a tile rectangle.                      |
| `TILE_CLS()`                          | Clear the TILE surface.                     |

`TILE_PRINT` uses tile coordinates and does not clip text. The caller
must keep the string inside the TILE surface. Cell TILE backends write
normal screen cells. Native TILE backends write target tile-map entries.
Bitmap TILE backends use the built-in 8x8 font unless the target
provides its own text renderer.

Additional Font Glyphs:

| Name | Code | Use |
| --- | --- | --- |
| `TILE_FONT_SPADE` | `91` | `CHR(TILE_FONT_SPADE)` |
| `TILE_FONT_HEART` | `92` | `CHR(TILE_FONT_HEART)` |
| `TILE_FONT_DIAMOND` | `93` | `CHR(TILE_FONT_DIAMOND)` |
| `TILE_FONT_CLUB` | `94` | `CHR(TILE_FONT_CLUB)` |

Tile timing:

| Call                                  | What it does                                |
| ------------------------------------- | ------------------------------------------- |
| `WAIT_TILE()`                         | Wait for the target default tile interval.  |
| `WAIT_TILE(count)`                    | Wait for a target-specific tile interval.   |

Tile assets:

| Call                                  | What it does                                |
| ------------------------------------- | ------------------------------------------- |
| `TILE_ALIAS(id, code)` / `TILE_ALIAS(id, s$)` | Map an abstract ID to a backend tile or cell code. |
| `TILE_DEFINE(id, addr)`               | Define tile glyph data for capable targets. |
| `TILE_BLOCK_DEFINE(id, w, h, addr)`   | Name a row-major tile or cell block.        |
| `TILE_BLIT(block, x, y)`              | Draw a named block.                         |
| `TILE_BLIT(addr, x, y, w, h)`         | Draw an ad hoc block.                       |
| `TILE_CLEAR_TILE(id)`                 | Select the clear tile ID.                   |
| `TILE_COLOR(id, fg, bg)`              | Set TILE colors when available.             |
| `TILE_DEFAULT_DISPLAY_MODE()`         | Return a default `DISPLAY` mode for the selected tile backend. |

Capability and geometry names:

| Name / call                    | What it means                                   |
| ------------------------------ | ----------------------------------------------- |
| `TILE_AVAILABLE`               | Tile drawing is available.                      |
| `TILE_CELL_AVAILABLE`          | Text/cell screen drawing is available.          |
| `TILE_NATIVE_AVAILABLE`        | Target tile hardware drawing is available.      |
| `TILE_BITMAP_AVAILABLE`        | Bitmap drawing mode exists.                     |
| `TILE_KERNEL_AVAILABLE`        | Target-specific tile display is available.      |
| `TILE_DEFINE_AVAILABLE`        | `TILE_DEFINE` can install tile data.            |
| `TILE_BLOCK_AVAILABLE`         | `TILE_BLOCK_DEFINE` and `TILE_BLIT` are useful. |
| `TILE_COLOR_AVAILABLE`         | Current tile surface honors `TILE_COLOR`.       |
| `TILE_BACKEND`                 | Active drawing mode.                            |
| `TILE_CELL_W` / `TILE_CELL_H`  | Tile size in screen cells.                      |
| `TILE_PIXEL_W` / `TILE_PIXEL_H` | Tile size in pixels.                           |
| `TILE_COLUMNS` / `TILE_ROWS`   | Logical TILE surface size.                      |

Bitmap TILE uses `DISPLAY` and the portable graphics API internally and can draw glyphs supplied by
`TILE_DEFINE`; bitmap `TILE_BLIT` treats block bytes as tile IDs, so
defined glyphs and `TILE_COLOR` settings are honored. Kernel TILE
targets may expose only a small subset. Target docs list backend
limits.

`TILE_COLOR_AVAILABLE` is read-only runtime state that reports whether
the selected TILE backend honors `TILE_COLOR`. For bitmap TILE, it follows
the current `DISPLAY` mode and should be read after that mode is selected.
It cannot be used with `@IF` or `@REQUIRES`.

## Input

| Call                     | What it does                                        |
| ------------------------ | --------------------------------------------------- |
| `KEY()`                  | Blocking typed key code. Returns `U8`.              |
| `INKEY()`                | Nonblocking typed key as a string, or `""`.         |
| `INKEY_CODE()`           | Nonblocking typed key code, or `0`.                 |
| `RAWKEY()`               | Nonblocking held key as a string, or `""`.          |
| `RAWKEY_CODE()`          | Nonblocking held key code, or `0`.                  |
| `KEY_HELD(code)`         | `1` while the requested key is down, or `0`.        |
| `KEYBOARD_AVAILABLE`           | `1` when keyboard input is available.               |
| `KEY_HELD_AVAILABLE`           | `1` when individual held key polling is available.  |
| `KEYPAD_AVAILABLE`             | `1` when keypad input is available.                 |
| `JOYSTICK_AVAILABLE`           | `1` when joystick input is available.               |
| `JOYSTICK_BUTTONS_AVAILABLE`   | `1` when joystick buttons are available.            |
| `ANALOG_JOYSTICK_AVAILABLE`    | `1` for analog joysticks.                           |
| `DIGITAL_JOYSTICK_AVAILABLE`   | `1` for digital joysticks.                          |
| `PADDLES_AVAILABLE`            | `1` when `PADDLE(axis)` has supported axes.         |
| `MOUSE_AVAILABLE`              | `1` when mouse input is available.                  |
| `INPUT_ANY()`            | `1` when a key, keypad, joystick, or button is active. |
| `INPUT_ANY(wait)`        | `INPUT_ANY()`, but blocks first when `wait` is `1`. |
| `INPUT_CLEAR()`          | `1` when no key, keypad, joystick, or button is active. |
| `INPUT_CLEAR(wait)`      | `INPUT_CLEAR()`, but blocks first when `wait` is `1`. |
| `JOY(port)`              | Direction bits from a zero-based joystick port.     |
| `JOY_BUTTON(port, button)`   | `1` if pressed, `0` otherwise.                      |
| `JOY_SET_DEADZONE(value)` | set analog joystick dead zone                     |
| `PADDLE(axis)`           | Raw analog axis value.                              |
| `KEYPAD_CODE(port)`      | Raw keypad code, or `KEYPAD_NONE`.                  |

Input constants:

| Name                    | What it means                                       |
| ----------------------- | --------------------------------------------------- |
| `JOY_PORTS`             | Number of joystick ports for `JOY(port)`.           |
| `JOY_DEFAULT_PORT`      | Default joystick port for gameplay input.           |
| `JOY_BUTTONS`           | Buttons per joystick port for `JOY_BUTTON(port, button)`. |
| `JOY_DEADZONE_DEFAULT`  | initial analog joystick dead zone                   |
| `PADDLE_AXES`           | Number of analog axes for `PADDLE(axis)`.           |
| `ANALOG_AXIS_PAIR_NAME` | Name for an X/Y analog axis pair.                   |
| `MOUSE_BUTTONS`         | Number of mouse buttons; `0` when none.             |
| `KEYPAD_PORTS`          | Number of keypad ports for `KEYPAD_CODE(port)`.     |
| `KEYPAD_KEYS`           | Number of keypad key codes, excluding `KEYPAD_NONE`. |
| `KEYPAD_FUNCTION_KEYS`  | Number of nonnumeric keypad key codes.              |
| `KEYPAD_TEXT_INPUT_AVAILABLE` | `true` when keypad numeric `INPUT` is available. |
| `KEY_SPACE`               | ASCII keyboard space code.                         |
| `KEY_0` through `KEY_9` | ASCII keyboard digit codes.                         |
| `KEY_A` through `KEY_Z` | Uppercase ASCII keyboard letter codes.              |
| `KEY_LOWER_A`, `KEY_LOWER_Z`, `KEY_LOWER_TO_UPPER_DELTA` | Helpers for folding lowercase ASCII letters. |

`KEYPAD_CODE(port)` returns `KEYPAD_NONE`, `KEYPAD_0` through
`KEYPAD_9`, `KEYPAD_ASTERISK`, `KEYPAD_POUND`, `KEYPAD_START`,
`KEYPAD_PAUSE`, or `KEYPAD_RESET`. Missing keypad ports return
`KEYPAD_NONE`.

Keyboard constants `KEY_SPACE`, `KEY_0` through `KEY_9`, and `KEY_A`
through `KEY_Z` are useful with `KEY()`, `INKEY_CODE()`, and
`RAWKEY_CODE()` on targets that report ASCII-compatible characters.
Cursor and function key codes are target specific.

Test joysticks against `JOY_UP`, `JOY_DOWN`, `JOY_LEFT`, `JOY_RIGHT` with bitwise `&`:

```basic
IF JOY(0) & JOY_LEFT <> 0 THEN ...
```

Missing joystick ports and buttons return 0. On analog joystick targets,
`JOY` still returns the shared direction bits.

Analog targets start with `JOY_DEADZONE_DEFAULT`; larger
`JOY_SET_DEADZONE(n)` values ignore more center drift, smaller values
make the stick more sensitive, and digital targets accept the call
without changing state.

`MOUSE_BUTTONS` reports the button count, not the current button state.
Mouse position and mouse-button state are target-specific until a
portable mouse input surface exists.

`KEY()`, `INKEY()`, and `INKEY_CODE()` are typed keyboard input. They
read the target's normal key queue or equivalent translated key path, so
they report character events rather than the current physical key state.
`KEY()` blocks until a character is available. `INKEY()` and
`INKEY_CODE()` return immediately. Use them for prompts, menus, and text
input.

`RAWKEY()` and `RAWKEY_CODE()` are for games and control polling. They
use a direct key state path when a target has one, bypassing the text
editor or key queue. A held key may be reported on every poll. On
targets without a separate direct key path, raw key input may match
`KEY()`, `INKEY()`, and `INKEY_CODE()`.

`KEY_HELD(code)` checks one key without blocking or consuming typed
input. It accepts `KEY_SPACE`, `KEY_0` through `KEY_9`, and `KEY_A`
through `KEY_Z`. Letter codes identify the key regardless of Shift or
character case. Gate its use with `KEY_HELD_AVAILABLE`. Multiple key
detection follows the target keyboard's rollover and ghosting limits.

`INPUT_ANY()` is for "press anything" prompts. On keyboard targets it
checks raw key state and does not consume the text key queue. On keypad
targets it also reports active keypad keys. Pass `0` to poll and `1`
to wait until the condition is true. To prevent held input from
carrying into a prompt, wait for clear input before waiting for new
input:

```basic
INPUT_CLEAR 1
INPUT_ANY 1
```

Keypad backed `INPUT` is limited to unsigned integer variables. Press
the target's keypad commit key to finish entry. Target docs list keypad
details and per-target input counts.

## Game helpers

These helpers are for portable game code.

### LFSR

| Call              | Returns       | What it does                                  |
| ----------------- | ------------- | --------------------------------------------- |
| `LFSR_SEED(seed)` | none          | Seed the byte or word generator.              |
| `LFSR_NEXT()`     | `U8` or `U16` | Next pseudo-random byte or word.              |
| `LFSR_RANGE(hi)`  | `U8`          | Pseudo-random value from 0 through `hi`.      |

The seed type selects the byte or word generator. The destination type selects
which `LFSR_NEXT` to use. Use `U8(...)` or `U16(...)` when a literal would be
unclear. The byte and word generators have independent state. `LFSR_RANGE`
uses the byte generator.

### Collision

| Call                                                   | Returns | What it does                         |
| ------------------------------------------------------ | ------- | ------------------------------------ |
| `POINT_IN_RECT(px, py, x, y, w, h)`                    | `U8`    | `1` if the point is inside the rectangle. |
| `RECT_OVERLAP(x1, y1, w1, h1, x2, y2, w2, h2)`         | `U8`    | `1` if two rectangles overlap.       |
| `AABB_HIT(x1, y1, w1, h1, x2, y2, w2, h2)`             | `U8`    | `1` if two rectangles overlap.       |

Rectangles include their top-left point and exclude `x + w`, `y + h`.

### Sprites

| Name / call                      | What it does                         |
| -------------------------------- | ------------------------------------ |
| `SPRITE_AVAILABLE`               | Defined as `1` when the target exposes `SPRITE_*`. |
| `SPRITE_KIND`                    | `SPRITE_KIND_HW` or `SPRITE_KIND_POLYFILL`. |
| `SPRITE_SURFACE_KIND`            | `SPRITE_SURFACE_KIND_PIXEL`, `_CELL`, or `_KERNEL`. |
| `SPRITE_COLOR_MODEL`             | How the target colors a sprite: `SPRITE_COLOR_MODEL_PER_PIXEL`, `_PER_CELL`, or `_POSITIONAL`. |
| `SPRITE_MAX_COUNT`               | Number of portable sprite slots.     |
| `SPRITE_WIDTH` / `SPRITE_HEIGHT` | Sprite footprint in the target coordinate system. |
| `SPRITE_DATA_BYTES`              | Target native byte count copied by `SPRITE_DATA`. |
| `SPRITE_COLORS_PER_SPRITE`       | Visible nontransparent color codes in the active sprite mode. |
| `SPRITE_X_MIN` / `SPRITE_X_MAX`  | Fully visible horizontal position range. |
| `SPRITE_Y_MIN` / `SPRITE_Y_MAX`  | Fully visible vertical position range. |
| `SPRITE_X_GRANULARITY` / `SPRITE_Y_GRANULARITY` | Coordinate snap size. |
| `SPRITES_BEGIN()` / `SPRITES_RESET()` | Initialize or reset sprite state. |
| `SPRITE_HAS_DATA(id)`            | `1` when the slot accepts `SPRITE_DATA`. |
| `SPRITE_COLOR_SHARED(id)`        | `1` when changing the slot color can affect another sprite. |
| `SPRITE_NAME(id)`                | Target identifier for the slot, or a generic name. |
| `SPRITE_DATA(id, addr)`          | Install target-native sprite data.   |
| `SPRITE_DATA_8X8(id, addr)`      | Install an 8-byte MSB-left 1bpp shape. |
| `SPRITE_DATA_TILES(id, w, h, addr)` | Install tile-shaped sprite data where supported. |
| `SPRITE_HAS_INVERT`              | `1` when `SPRITE_INVERT` is supported. |
| `SPRITE_INVERT(id)`              | Invert the target-native sprite data loaded in the slot. |
| `SPRITE_ALIGN_X(x)` / `SPRITE_ALIGN_Y(y)` | Snap a coordinate to the target's sprite grid. |
| `SPRITE_MOVE(id, x, y)`          | Move a sprite.                       |
| `SPRITE_COLOR(id, color)`        | Set sprite color where supported.    |
| `SPRITE_BG(color)`               | Set the backdrop color the sprite color blends against on per-cell-color targets. No-op elsewhere. |
| `SPRITE_COLOR2(color)` / `SPRITE_COLOR3(color)` | Extra shared sprite colors on multicolor-capable targets. No-op elsewhere. |
| `SPRITE_FLIP_X(id, flag)` / `SPRITE_FLIP_Y(id, flag)` | Set hardware X/Y flip where supported. |
| `SPRITE_EXPAND(id, x_double, y_double)` | Set hardware expansion where supported. |
| `SPRITE_PRIORITY(id, behind_bg)` | Set hardware foreground/background priority where supported. |
| `SPRITE_PALETTE(id, palette)` | Select a hardware sprite palette where supported. |
| `SPRITE_PALETTE_SET(palette, c0, c1, c2)` | Set hardware sprite palette colors where supported. |
| `SPRITE_SHOW(id)` / `SPRITE_HIDE(id)` | Show / hide a sprite.          |
| `SPRITE_HIT(id)` | Sprite/sprite collision flag where supported, otherwise `0`. |
| `SPRITE_HIT_BG(id)` | Sprite/background collision flag where supported, otherwise `0`. |
| `SPRITES_OFF()`                  | Hide or disable all sprites.          |
| `SPRITES_FLUSH()`                | Commit staged sprite changes where needed. |

`SPRITE_MAX_COUNT` includes every movable sprite object exposed by the
target. Some targets have limited slots that cannot load custom data or
that share color hardware. Use `SPRITE_HAS_DATA` and
`SPRITE_COLOR_SHARED` when selecting slots dynamically. `SPRITE_NAME`
returns the target's identifier when one exists and otherwise returns a
name such as `SPRITE0`.

The size and bounds describe the footprint in the target coordinate
system, not necessarily the number of encoded color cells. A VIC-II
multicolor sprite still reports a 24x21 footprint while each row holds 12
double wide color cells. Plus/4 multicolor sprite coordinates use its
160 pixel bitmap and report a 4x8 footprint.

`SPRITE_DATA` copies target native bytes without converting them.
Changing modes changes how the target reads those bytes, so use a shape
made for that mode. `SPRITE_DATA_8X8` is the portable one bit 8x8 form
where the target supports it.

`SPRITE_COLOR_MODEL` tells a portable program how the target carries
sprite color so it can adapt to "attribute clash":

- `SPRITE_COLOR_MODEL_PER_PIXEL` - each pixel keeps its own color (no
  clash): hardware sprites and per-pixel bitmaps.
- `SPRITE_COLOR_MODEL_PER_CELL` - color is shared per attribute cell, so
  a sprite recolors the background pixels in any cell it overlaps. The
  runtime corrects this so a cell takes the sprite color only while a
  sprite currently overlaps it and reverts afterward; `SPRITE_BG`
  selects the backdrop color the sprite blends against.
- `SPRITE_COLOR_MODEL_POSITIONAL` - color is chosen by pixel position,
  so `SPRITE_COLOR` is best effort.

`SPRITE_BG`, `SPRITE_COLOR2`, and `SPRITE_COLOR3` are always callable and
are no-ops on targets whose color model does not use them. On supported
targets, `@OPTION MULTICOLOR_SPRITE TRUE` selects multicolor for every
sprite managed through the portable API. `FALSE` selects standard mode.
Target controls may still mix modes per sprite.

Multicolor shapes use two bit color codes with `00` transparent. The
other codes use `SPRITE_COLOR`, `SPRITE_COLOR2`, and `SPRITE_COLOR3` as
documented by each target. `SPRITE_BG` selects the backdrop on software
targets that blend sprite and bitmap colors.

Software sprite targets also accept an active slot count hint:

```basic
@OPTION SOFT_SPRITE_ACTIVE_COUNT 1
```

This is optional. It limits the software sprite reset, flush, and off
walks to the first N slots while keeping `SPRITE_MAX_COUNT` as the target
capacity. Use it only when the program never uses sprite ids greater than
or equal to N.

## Core builtins

Numeric helpers, bitwise and logical operators, `REAL` helpers, and
string functions are part of the core language. They do not require
an API include. See [LANGUAGE.md](LANGUAGE.md#core-builtins).

## REAL floating point

`REAL` is optional and target dependent.

```basic
X! = 3.14159
Y! = LOG(X!)
PRINT STR(Y!)
```

Declaring `REAL` on a target without FP is a compile error.
Core `REAL` functions are documented in
[LANGUAGE.md](LANGUAGE.md#real-functions).

## Fixed Point

Unsigned fixed-point helpers use Q8.8 values stored in `U16`: the high
byte is the integer part and the low byte is the fractional part.

| Name | What it does |
| ---- | ------------ |
| `FX_ONE` | The value `1.0` (`256`). |
| `FX_HALF` | The value `0.5` (`128`). |
| `FX_MAX` | Largest Q8.8 value (`65535`). |
| `FX_FROM_INT(n)` | Convert a `U8` integer to Q8.8. |
| `FX_FROM_PARTS(i, f)` | Build Q8.8 from integer and fractional bytes. |
| `FX_INT(x)` | Return the integer byte. |
| `FX_FRAC(x)` | Return the fractional byte. |
| `FX_MUL(a, b)` | Multiply two Q8.8 values. |
| `FX_DIV(a, b)` | Divide two Q8.8 values. |

## Memory operations

### ADDR

`ADDR(name)`, `ADDR name`, and `&name` evaluate to a `U16` address.
They accept addressable storage, array elements, string literals, and
no-argument `PROC`s in fixed memory. Use them to pass an
address to `MEMMOVE`, `MEMFILL`, `DPOKE`, callback installers such as
`FRAME_INSTALL`, or inline `ASM`.

```basic
BUF(255) AS U8
WORDS(15) AS U16
NAMES$(3) AS STRING * 16

PTR## = ADDR BUF
PTR## = ADDR(BUF)
PTR## = &BUF
PTR## = ADDR BUF[10]
PTR## = &BUF[10]
PTR## = ADDR WORDS[3]
PTR## = ADDR(WORDS(3))
PTR## = ADDR(NAMES$(2))
PTR## = ADDR "LITERAL"

PROC FOO
    PRINT "IN PROC"
ENDPROC

PROC_PTR## = ADDR FOO
```

`ADDR(ARR(I))` returns the address of the indexed element, not the start of
the array. For a string array element, the address points at that element's
string descriptor. Characters begin after the descriptor: +1 for narrow
strings and +2 for strings with capacity over 255. The element offset uses
the same target and capacity stride as string array loads and stores.

For a string variable, `ADDR(S$)` points at the length byte (or two
bytes if the string was declared with capacity over 255); the
actual characters start right after.

`ADDR(FOO)` on a `PROC` returns its entry address.
The `PROC` must take no parameters and must not live in a
switched bank.

`ADDR` can take the address of `DIM` storage, typed `CONST` arrays, and
eligible PROCs. It cannot take the address of a scalar `CONST` or a
`PROC` parameter.

### PEEK, POKE, DPEEK, DPOKE

```basic
B = PEEK($2000)
POKE $2000, 6
W = DPEEK $2010
DPOKE($2020, $ABCD)
```

- `PEEK` reads one byte from an address.
- `POKE` writes one byte to an address.
- `DPEEK` reads two bytes from an address and returns a 16-bit value: low byte first, high byte second.
- `DPOKE` writes a 16-bit value to an address: low byte first, high byte second.

### MEMMOVE, MEMFILL, MEMCOMPARE

```basic
SRC(15) AS U8
DST(15) AS U8

MEMFILL(ADDR(SRC), 16, 0)
MEMMOVE(ADDR(SRC), ADDR(DST), 16)
IF MEMCOMPARE(ADDR(SRC), ADDR(DST), 16) THEN PRINT "SAME"
```

`MEMMOVE(src, dst, count)` copies bytes forward from source to destination.
It does not preserve overlapping source and destination ranges.

`MEMFILL(dst, count, value)` fills a range of bytes with one byte value.

`MEMCOMPARE(a, b, count)` returns `1` when two byte ranges match,
otherwise `0`.

`MEMSIZE_TEXT(size_kb)` returns a compact `K` or `MB` string for a KB
count, such as `64K` or `1 MB`.

### INCLUDE_BIN

`@INCLUDE_BIN` embeds a binary file as a typed byte array.

```basic
@INCLUDE_BIN CHARMAP, "assets/charmap.bin"
@INCLUDE_BIN SPRITES, "assets/sprites.bin", OFFSET $80, COUNT 64

MEMMOVE ADDR CHARMAP, $3000, CHARMAP_LEN
PRINT HEX(CHARMAP_END)
```

For `@INCLUDE_BIN LABEL, "path"`, the compiler creates:

| Name | What it is |
| ---- | ---------- |
| `LABEL` | `CONST` `U8` array containing the bytes. |
| `LABEL_LEN` | Byte count as `U16`. |
| `LABEL_LAST` | Last valid array index as `U16`. |
| `LABEL_END` | No-argument `PROC` returning `ADDR LABEL + LABEL_LEN`. |

Paths are resolved relative to the source file first, then relative to
the project root. `OFFSET` and `COUNT` accept decimal, `$` hex, and
`0x` hex literals.

`@INCLUDE_BIN` currently emits normal const data. `AT` placement and
`@BANK PRG` placement are reserved for a later compiler placement pass.

### EXTMEM

EXTMEM moves blocks between local RAM and target extended memory.
Targets may back it with a DMA peripheral such as the C64 REU or with a
banked RAM window such as the Commander X16.

The EXTMEM address is `BANK:XADDR`, where `BANK` is `U8` and `XADDR`
is `U16`. `COUNT = 0` means 65536 bytes.

| Name | What it does |
| ---- | ------------ |
| `EXTMEM_AVAILABLE()` | `1` when extended memory responds to a runtime probe, otherwise `0`. |
| `EXTMEM_STASH(count, localaddr, xaddr, bank)` | Copy local RAM to extended memory. |
| `EXTMEM_FETCH(count, localaddr, xaddr, bank)` | Copy extended memory to local RAM. |
| `EXTMEM_SWAP(count, localaddr, xaddr, bank)` | Exchange local and extended memory. |
| `EXTMEM_VERIFY(count, localaddr, xaddr, bank)` | Return `1` when the two ranges match. |
| `EXTMEM_POKE(bank, xaddr, value)` | Write one byte to extended memory. |
| `EXTMEM_PEEK(bank, xaddr)` | Read one byte from extended memory. |
| `EXTMEM_DETECT_KB()` | Probe the visible extended memory size in KB. |
| `EXTMEM_HAS_KB(size_kb)` | Return `1` when at least that size is visible. |
| `EXTMEM_PROBE_BANK(id)` | Target probe bank for a size step. |
| `EXTMEM_PROBE_KB(id)` | Size represented by a probe step. |
| `EXTMEM_BANK_OK(id)` | Return `1` when a probe bank does not alias earlier banks. |
| `EXTMEM_BANK_KB` | KB represented by one external bank unit. |

Low level code can call `EXTMEM_OP(op, count, localaddr, xaddr, bank)`
with `EXTMEM_OP_STASH`, `EXTMEM_OP_FETCH`, `EXTMEM_OP_SWAP`, or
`EXTMEM_OP_VERIFY`. Most programs should use the named calls instead.

Guard portable code with `IF EXTMEM_AVAILABLE() THEN` before
transferring. Targets without an EXTMEM provider return `0` from
`EXTMEM_AVAILABLE()`; transfer calls require a target provider.

### BIND

```basic
BIND STRING$ TO ADDR, COUNT

RAW(79) AS U8
VIEW$ AS STRING * 80

BIND VIEW$ TO ADDR(RAW), 80
VIEW$ = "HELLO"
MID(VIEW$, 10, 1) = CHR(255)
```

`BIND` redirects a string's characters to your own writable memory.
The string still cannot grow past its declared capacity. No hidden length
byte is placed before the bound address.

- The target must be a string variable.
- `ADDR` and `COUNT` are treated as 16-bit values.
- If `COUNT` is a fixed number, it cannot be larger than the string's declared size.
- Must point at writable RAM.
- Strings declared up to 255 bytes (`STRING * 255` or smaller) are portable.

## Events

The portable event API exposes asynchronous target events. On targets
with `EVENT_INTERRUPT_CALLBACKS = 1`, subscribed handlers run from an
interrupt or OS callback and must be no argument `@ASYNC` PROCs.

| Name | What it does |
| --- | --- |
| `EVENT_AVAILABLE` | `1` when the active target has an event provider. |
| `EVENT_INTERRUPT_CALLBACKS` | `1` when handlers are called from interrupt context. |
| `EVENT_SPRITE_SPRITE` | Sprite to sprite collision event mask. |
| `EVENT_SPRITE_BG` | Sprite to background collision event mask. |
| `EVENT_LIGHTPEN` | Light pen event mask. |
| `EVENT_SUBSCRIBE(mask, addr)` | Subscribe an `@ASYNC` handler address for one or more event bits. |
| `EVENT_UNSUBSCRIBE(mask)` | Remove handlers and disable those event bits. |
| `EVENT_ENABLE(mask)` | Enable one or more event bits without changing handlers. |
| `EVENT_DISABLE(mask)` | Disable one or more event bits. |
| `EVENT_PENDING()` | Return latched pending event bits. |
| `EVENT_ACK(mask)` | Clear pending event bits. |

On targets without an event provider these calls compile as no-ops, and
`EVENT_PENDING()` returns `0`.

## CALL

```basic
CALL $2000
```

`CALL` runs the machine-code routine at a fixed address. How you pass
values in, what registers it changes, and how it returns values depend
on the machine and the routine you are calling.

## USR

`USR(...)` is dialect specific - the calling convention follows the
host BASIC. See [`targets/`](targets/) for the exact form.

For CrustyBASIC code, use `CALL` for a simple fixed-address machine-code routine,
or inline `ASM` when you need to pass values or get a result.

## Hardware registers

Targets expose chip registers as names. The target docs under
[`targets/`](targets/) list chip namespaces and register details.

## Target docs

Per-target screen sizes, input counts, image formats, tick sources,
file providers, chip namespaces, and system caveats live in the target
docs under [`targets/`](targets/). Portable programs should prefer the
capability constants and runtime availability probes documented above.
