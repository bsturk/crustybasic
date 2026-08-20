# crustyBASIC on Windows x64

Use `@OPTION TARGET win64` for the Windows x64 executable target.
Output is a native `.exe`.

Portable runtime calls are documented in [`../API.md`](../API.md).
This page covers Windows x64 modes, options, formats, and runtime
notes. For command line target selection and dialects, see
[`../USAGE.md`](../USAGE.md).

## Systems

| System | Output | Notes |
| --- | --- | --- |
| `win64` | `.exe` | Native Windows x64 PE executable. |

NASM is the supported assembler. The target emits a PE executable
directly from NASM output.

## Basics

| Item | Value |
| --- | --- |
| CPU | x64 |
| Text | 80x25 |
| Code storage | RAM |
| Address size | 64 bit |
| String encoding | ASCII, with non ASCII bytes encoded as `?` |
| Newline byte | `$0D` |
| Integer math | Built in 16 bit integer support |
| `REAL` math | Native 64 bit IEEE 754 |
| Host OS | Win32 APIs are available |

All current Windows x64 programs use a 256 byte string buffer limit.

## Program Window

By default, `win64` builds a console subsystem executable. Text output
uses the console, and input reads console events.

When a console build also opens a GDI window, keyboard input follows the
focused window.

When a console executable is started by double clicking it, Windows
closes the console window as soon as the program exits. Run it from
`cmd.exe` or PowerShell to see final output, or keep the program alive
with input, a wait, or a loop. GDI window builds behave the same way:
the window closes when the program returns.

Add this source option for a GUI subsystem executable with no console
window:

```basic
@OPTION WIN64_CONSOLE_WINDOW FALSE
```

| Option | Default | Effect |
| --- | --- | --- |
| `WIN64_CONSOLE_WINDOW` | `TRUE` | `FALSE` uses only the GDI window for text, graphics, and input. |

In UI window mode, `PRINT` text is drawn in the GDI window, `KEY` and
`INKEY_CODE` read window character input, and closing the window exits
the process on the next message pump or frame wait.

Console mode remains useful for text only programs and command line
tools. GDI examples that should not show a console window should set
`WIN64_CONSOLE_WINDOW FALSE`.

## Text And Cells

Console mode uses Win32 console APIs for output, clearing, positioning,
and keyboard events.

The console text grid follows the visible console window. `TEXT_WIDTH`,
`TEXT_HEIGHT`, `TEXT_COLUMNS`, `CLS`, and `POSITION` use the cached window
size and its position within the scrollback buffer. Resizing is applied when
the program next checks keyboard input. Ordinary printing does not query the
window size. Public dimensions cap at 255 cells because text coordinates are
U8; clearing and cursor tracking still use the full Windows dimensions.

UI window mode draws text into the GDI window using 8x16 cells. The
text grid is still 80x25. `TEXT_COLOR`, `CELL_COLORS`, and per-cell
`CELL_COLOR` are available;
cell attributes are not.

| Capability | Value |
| --- | --- |
| Text color | Yes |
| Cell color | Yes |
| Cell color size | 1x1 |
| Cell attributes | No |

## Graphics

Graphics are drawn into a buffered 32 bit GDI DIB. The portable color
model exposes 16 logical colors. `GFX_BUFFERING` is `GFX_BUFFERING_COPY`, so `GFX_SWAP` is
`TRUE`.

The default graphics mode is `BITMAP_HIRES`.

| Mode | Size | Colors | Notes |
| --- | --- | --- | --- |
| `CELL`, `TEXT_80X25` | 80x25 cells | 16 | Text display. |
| `BITMAP_HIRES`, `GDI_640X480` | 640x480 | 16 | Default bitmap mode. |
| `GDI_1024X768` | 1024x768 | 16 | Larger GDI bitmap mode. |
| `GDI_256X192_SCALE2` | 256x192 logical | 16 | Draws at 512x384 centered in a 640x480 window. |
| `BITMAP_LORES` | not supported | | |
| `BITMAP_MULTICOLOR` | not supported | | |
| `CELL_MULTICOLOR` | not supported | | |

`GDI_COLOR(r, g, b)` sets an arbitrary RGB foreground color for
GDI drawing. `GDI_FILL_RECT(x, y, w, h)` fills a rectangle with
the current GDI foreground color.

`GFX_CLS`, `GDI_FILL_RECT`, and other drawing calls update the buffered
bitmap. `GFX_SWAP` publishes it immediately. `FRAME_WAIT` and blocking
input calls also publish pending changes, so portable programs do not
need a Windows-specific swap.

A typical bitmap frame loop is:

```basic
DO
	GFX_CLS
	PLOT 10, 10
	GFX_SWAP
	FRAME_WAIT
LOOP
```

## Images

Native image display supports:

| Format | Accepted files |
| --- | --- |
| `IMAGE_FMT_WIN64_INDEXED` | 256x192, one palette index byte per pixel. |

Images are linked into the program rather than loaded from Windows
files at runtime. Runtime image file loading is not available.

## Sprites

The Windows x64 target uses 8x8 software sprites drawn into the GDI
bitmap buffer.

| Feature | Value |
| --- | --- |
| Sprite count | 16 |
| Size | 8x8 |
| Data bytes | 8 |
| X range | 0 through 632 |
| Y range | 0 through 472 |
| Collision | General sprite hit flag |
| Flip, stretch, priority, palette | Not supported |

Call `SPRITES_FLUSH` before `GFX_SWAP` to draw staged sprites into the
buffered bitmap. `FRAME_WAIT` does not flush or publish sprites.

## Tile

The Windows x64 target uses the generic tile runtime.

| Capability | Value |
| --- | --- |
| Default backend | `TILE_BITMAP` |
| Native cells | Yes |
| Bitmap tiles | Yes |
| Tile colors | Yes |
| Tile kernels | No |
| Default wait | 1 frame |

The bitmap tile default display mode is `GDI_256X192_SCALE2`.
This gives tile programs a 256x192 logical surface using 8x8 bitmap
tiles, scaled to 512x384 and centered in a 640x480 GDI window.

## Input

| Capability | Value |
| --- | --- |
| Keyboard | Yes |
| Individual held keys | Yes |
| Joystick ports | 1 |
| Buttons | 1 |
| Stick type | Keyboard mapped digital directions |
| Paddle axes | 0 |
| Mouse buttons | 1 |
| Keypad ports | 0 |
| Keypad `INPUT` | No |

In console mode, keyboard input reads Win32 console input events. In UI
window mode, keyboard input comes from the GDI window message loop.

`KEY_HELD` asks Windows whether a key is down without consuming input.

The joystick API maps arrow keys and `W`, `A`, `S`, `D` to directions.
The button maps to Space, Enter, or Ctrl.

## Sound

The portable sound calls mix four square wave voices through Windows
audio. `PLAY_NOTE` continues until `SOUND_OFF` or `SILENCE`.
`PLAY_NOTE_FOR`, `PLAY_NOTE_SHAPE_FOR`, and `BEEP` stop after their
duration. `PLAY_NOTE_SHAPE` is accepted, but this target has only
square wave output.

Notes map MIDI note numbers 36..95 through a Hz table. Volume is on or
off. `SOUND_PRESENT` reports whether Windows has a waveform output device.

## Audio

Win64 plays prepared PCM WAV assets through `PlaySoundA` with the
memory, asynchronous, and no-default flags:

- `AUDIO_PLAYBACK_MODEL = AUDIO_PLAYBACK_AUTOMATIC` - Windows advances
  playback; `AUDIO_SERVICE` is a no-op.
- `AUDIO_FILE_SUPPORTED = TRUE` - `AUDIO_LOAD(path$)` preloads a prepared
  `.CBA` file into allocated memory and closes the file before
  returning.
- `AUDIO_TARGET_FILE_SUPPORTED = TRUE` - the path overload also accepts a
  plain RIFF/WAVE PCM file.
- `AUDIO_SOUND_SHARED = FALSE` - `PlaySound` is process wide, so AUDIO and
  the generated sound voices replace each other; do not use portable
  note calls while AUDIO is active.

Accepted WAV sources are `WAVE_FORMAT_PCM` with one or two channels and
8- or 16-bit samples. RIFX, RF64, IEEE float, `WAVE_FORMAT_EXTENSIBLE`,
and compressed codecs are rejected at conversion time. The CBA payload
is a canonical RIFF/WAVE image, so Windows consumes it directly. If the
waveform device rejects the prepared format at runtime, `AUDIO_PLAY`
returns `0` and the asset stays loaded and stopped.

The build host cannot try the waveform device, so device format support
is a runtime question; playback has not yet been verified audibly on
Windows hardware.

## Timing

`FRAME_WAIT` publishes pending graphics, uses `GetTickCount64` and `Sleep`
to pace frames at about 60 Hz, and advances `FRAME_COUNTER`. `TICKS` uses
the frame counter fallback.

| Capability | Value |
| --- | --- |
| `FRAME_SUPPORTED` | Yes |
| `TIMER_SUPPORTED` | No |
| `TICKS_HZ` | 60 |
| `TICKS_FREE_RUNNING` | FALSE |

## System Commands

Windows x64 can run system commands and return their exit status.
`CMD_OPEN` captures standard output and standard error together. Use
`CMD_READ_LINE` to read the captured output, then `CMD_CLOSE` to finish
the command and get its exit status. Only one captured command can be
active at a time, and each line is limited by the destination string's
capacity.

| Capability | Value |
| --- | --- |
| `CMD_SUPPORTED` | Yes |
| `CMD_OUTPUT_SUPPORTED` | Yes |

## Files And Net

The file provider uses Win32 file handles. Read, write, append, update,
and directory modes are supported. Directory mode uses a `FindFirstFileA`
style listing. Paths are copied into a 260 byte native path buffer.

The net provider uses Winsock client sockets. TCP and UDP connects are
available through the portable net API. Socket handles use channels 1
through 15, and the current net buffer is 512 bytes.

## Examples

Windows x64 examples live under:

- [`../../examples/win64/`](../../examples/win64/)
