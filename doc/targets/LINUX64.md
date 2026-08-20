# crustyBASIC on Linux x64

Use `@OPTION TARGET linux64` for the native Linux x64 executable target.
The default output extension is `.elf`.

Portable runtime calls are documented in [`../API.md`](../API.md).
For command line target selection, see [`../USAGE.md`](../USAGE.md).

## Build Requirements

`crustybasic build` uses NASM:

```bash
crustybasic build examples/linux64/crustybasic/terminal_palette.cbs
```

No Linux compiler or linker is needed. A suitable NASM build can produce
the executable from Linux, Windows, or macOS.

The executable dynamically loads glibc, libm, and libdl when it runs.
Builds made on x64 Linux use that system's glibc loader path. Builds made
on other hosts use the standard `/lib64/ld-linux-x86-64.so.2` path. A
build does not inherit the build host's glibc version requirement. A file
moved from Windows may need its executable permission set on Linux.

Sound and prepared WAV playback load SDL2 at runtime. Programs can still
build and run without SDL2 when they do not use sound or audio.

## Basics

| Item | Value |
| --- | --- |
| CPU | x64 |
| Text | Current terminal size, default 80x25 |
| Code storage | RAM |
| Address size | 64 bit |
| String encoding | ASCII |
| Newline byte | `$0A` |
| Integer math | Built in 16 bit integer support |
| `REAL` math | Native 64 bit IEEE 754 |
| Host OS | Linux services are available |

## Terminal

`PRINT`, `CLS`, `POSITION`, `TEXT_COLOR`, `CURSOR_HIDE`, and
`CURSOR_SHOW` use the terminal. Keyboard input uses raw terminal input.
The original input mode and visible cursor are restored when the program
exits normally or receives Ctrl C.

`TEXT_COLOR_RGB` accepts red, green, and blue values from 0 through 255
and selects the nearest fixed xterm-256 color cube or grayscale entry.

`TEXT_WIDTH` and `TEXT_HEIGHT` use the current terminal size when stdout
is a terminal. They default to 80x25 otherwise.

## Files

The portable file API supports read, write, append, update, and directory
modes. Directory mode returns one entry per line.

## Networking

The portable net API supports TCP and UDP client connections. Socket
handles use channels 1 through 15, and the current read buffer is 512
bytes. Server calls are not available.

## System Commands

Linux supports all portable system command calls:

| Call | Purpose |
| --- | --- |
| `CMD` | Run a command and return its exit status |
| `CMD_OPEN` | Run a command and capture its output |
| `CMD_READ_LINE` | Read one captured line |
| `CMD_CLOSE` | Finish the command and return its exit status |

Only one captured command can be active at a time.

## Timing

`DELAY` uses milliseconds. `FRAME_WAIT` paces at about 60 Hz.
`TICKS` uses a 1000 Hz monotonic counter. `TIMER_START` and `TIMER_STOP`
store elapsed microseconds in `TIMER_HI:TIMER_LO`.

## Sound And Audio

The portable sound API provides four square wave voices through SDL2.
Prepared audio accepts PCM WAV data from an embedded CBA asset or a file.
Playback advances automatically. `SOUND_PRESENT` tries to open the SDL2
output used by portable notes.

## Visual APIs

Bitmap graphics, images, sprites, and tiles are not provided. Terminal
text and terminal colors are available.

## Examples

Linux examples live under:

- [`../../examples/linux64/`](../../examples/linux64/)
