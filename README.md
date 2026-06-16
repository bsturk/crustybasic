```text
         .---------------------------.      ____                 _        ____    _    ____ ___ ____
        /---------------------------/|     / ___|_ __ _   _  ___| |_ _   | __ )  / \  / ___|_ _/ ___|
       /| _________________________ ||    | |   | '__| | | |/ __| __| | ||  _ \ / _ \ \___ \| | |
      | | |                       | ||    | |___| |  | |_| |\__ \ |_| |_|| |_) / ___ \ ___) | | |___
      | | | 10 PRINT "CRUSTYBASIC"| ||     \____|_|   \__,_||___/\__|\__,|____/_/   \_\____/___|____|
      | | | 20 GOTO 10            | ||                                |___/
      | | | RUN                   | ||
      | | | _                     | ||
      | | |_______________________| ||
      | |  _           _        _   ||    .----------.
      | | |_|_________|_|______|_|__|/    | [======] |
      /_____________________________/     |    __    |
      |_____________________________|     |   |  |   |
        [_][_][_][_][_][_][_][_][_]       |___|__|___|
        [_][_][_][=======][_][_][_]

            \ )             ( /
             \ \__~^~^~^~__/ /
      _______-<   \ o   o /  >-_______
     ----------|    \___/   |----------
               \___________/
```

# crustyBASIC

CrustyBASIC is a Rust-based BASIC cross-compiler for 80s era
computers and consoles. It compiles BASIC source into native machine code.

**Currently supported targets:**

- Apple ][
- Atari
  - 800
  - 5200
  - 2600
- Commodore
  - 64
  - Plus/4
  - VIC-20
- Tandy/RS
  - Color Computers 1/2/3
- MS-DOS 16-bit
- NES

## Downloading the Compiler

To download the latest compiled binaries for Windows, macOS, or Linux, please visit the **[Releases Section](https://github.com/bsturk/crustybasic/releases)**.

## Quick Start

Save the below text as `hello.cbs`:

```basic
@OPTION TARGET c64

CLS
POSITION 0, 0
PRINT "HELLO, WORLD!"
```

Compile it to assembly:

```bash
crustybasic compile hello.cbs -o hello.s
```

Build a runnable C64 `.prg` in one shot. This needs a compatible assembler
(like `vasm`); see [USAGE.md#assemblers](doc/USAGE.md#assemblers).

```bash
crustybasic build hello.cbs -o hello.prg
```

## Screenshots

Screenshots and emulator captures live in [screenshots](./screenshots).

## Submitting Bugs

Please submit bug reports through the GitHub Issues tab.
Include the CrustyBASIC version, your operating system, the target or system you
are building for, the command you ran, and any compiler or assembler output.

Small self-contained `.cbs` files are the easiest reports to reproduce.

## Learn more

- [**Usage guide**](doc/USAGE.md) - commands, targets, dialects, assemblers, and examples.
- [**Language reference**](doc/LANGUAGE.md) - core language syntax.
- [**API reference**](doc/API.md) - portable runtime calls, graphics, sound, input, and target capabilities.
- [**Target documentation**](doc/targets/) - machine-specific notes for supported targets.
