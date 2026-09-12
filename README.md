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

CrustyBASIC is a Rust-based BASIC cross-compiler for vintage
computers and video game consoles as well as modern systems. It compiles BASIC 
source into native machine code to run faster.  It is available 
for Windows, MacOS, and Linux.

## Portable and Native

CrustyBASIC supports both portable programs and direct hardware access. Shared
graphics, sound, input, file, tile, and sprite APIs work across targets, while
target APIs expose native display modes, memory, hardware sprites, and chips
such as SID, POKEY, and PSG.

Image and audio support includes both portable sources and native machine
formats. Assets can be embedded, converted, or loaded from storage.
See the [API reference](doc/API.md) and [target documentation](doc/targets/).

**Currently supported targets:**

- Apple
    - ][+, //c, //e
- Atari
  - 400/800/XL/XE
  - 5200
  - 2600
  - 7800
- Commodore
  - 64, 64 Ultimate
  - 128
  - Plus/4
  - VIC-20
- Tandy/Radio Shack
  - Coco 1, 2, 3
- PC
    - MS-DOS 16-bit
    - Windows 64 bit GDI and console (`win64`, Win XP...Win 11)
    - Linux x64 console (`linux64`)
- Nintendo
  - NES
- Sinclair
  - ZX Spectrum
  - ZX81
  - Timex Sinclair 1000
- Many more to come!!

## Downloading the Compiler

To download the latest compiled binaries, visit the **[Releases Section](https://github.com/bsturk/crustybasic/releases)**.

## Downloading the Assembler

Building programs requires the target's assembler; see [USAGE.md#assemblers](doc/USAGE.md#assemblers).

## Quick Start

### Build for one target

Save this as `hello.cbs`. `@OPTION TARGET c64` selects the C64:

```basic
@OPTION TARGET c64

CLS
POSITION 0, 0
PRINT "HELLO, C64!"
```

Build the runnable program:

```bash
crustybasic.exe hello.cbs
```

For the C64, this writes `hello.prg` by default. Use `-o` only to
choose a different name. You can also write the command explicitly:

```bash
crustybasic.exe build hello.cbs
```

To stop after generating assembly instead, use `compile`:

```bash
crustybasic.exe compile hello.cbs -o hello.s
```

### Build for several targets

To build the same program for several targets when the source is portable,
leave out the `@OPTION TARGET c64` line:

```basic
CLS
POSITION 0, 0
PRINT "HELLO, {target}!"
```

```bash
crustybasic.exe build hello.cbs --set target=c64
crustybasic.exe build hello.cbs --set target=atari800
crustybasic.exe build hello.cbs --set target=zx81
```

Each command creates the usual runnable file for that target, and
`{target}` is replaced with its target name. 

### Use target native features

This uses the C64's included Koala image support and talks directly to VIC-II and SID:

```basic
@OPTION TARGET c64
@INCLUDE_IMAGE TITLE "examples/c64/crustybasic/CRUSTY.KLA"

VIC.BORDER = BLACK
OK         = IMAGE_LOAD(ADDR TITLE_IMAGE)

SID.MASTER_VOLUME VOLUME_MAX
SID.ADSR 0, 0, 8, 12, 6
SID.FREQ 0, $116E
SID.CTRL 0, TRIANGLE + GATE
FRAME_DELAY 60
SID.GATE_OFF 0
```

## Screenshots

Screenshots and emulator pics are in [screenshots](./screenshots).

## Community

- [Discord](https://discord.gg/hCJf2Pcph) - join the discussions
- [itch.io](https://telengard.itch.io/crustybasic) - visit the CrustyBASIC page

## Submitting Bugs

Please submit bug reports through the GitHub Issues tab above or on Discord.
Include the CrustyBASIC version, your operating system, the target or system you
are building for, the command you ran, and any compiler or assembler output.

Small self-contained `.cbs` files are the easiest reports for me to reproduce.

## Learn more

- [**Usage guide**](doc/USAGE.md) - commands, targets, dialects, assemblers, and examples.
- [**Language reference**](doc/LANGUAGE.md) - core language syntax.
- [**API reference**](doc/API.md) - shared runtime calls, graphics, sound, input, and target capabilities.
- [**Target documentation**](doc/targets/) - native formats, display modes, chip APIs, and machine-specific limits.
