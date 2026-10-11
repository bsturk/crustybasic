```text
         .---------------------------.                           _        ____    _    ____ ___ ____
        /---------------------------/|       ___ _ __ _   _  ___| |_ _   | __ )  / \  / ___|_ _/ ___|
       /| _________________________ ||      / __| '__| | | |/ __| __| | ||  _ \ / _ \ \___ \| | |
      | | |                       | ||     | (__| |  | |_| |\__ \ |_| |_|| |_) / ___ \ ___) | | |___
      | | | 10 PRINT "CRUSTYBASIC"| ||      \___|_|   \__,_||___/\__|\__,|____/_/   \_\____/___|____|
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

crustyBASIC is a Rust based BASIC cross compiler for vintage
computers and video game consoles as well as modern systems. It compiles BASIC
source into native machine code that runs much faster than interpreted BASIC.
The compiler runs on Windows, macOS, and Linux.

## Your BASIC, your target

**Write in a familiar BASIC dialect or use crustyBASIC's own syntax.**

Supported [dialects](doc/LANGUAGE.md#dialects) include Sinclair BASIC,
Commodore BASIC, Applesoft BASIC, Atari BASIC, QBasic, and more.
Select one with `@OPTION DIALECT`, such as `@OPTION DIALECT sinclair_basic`.
Dialect programs can use crustyBASIC's APIs and target other systems too,
such as QBasic on the C128, provided the program's features are available
on the target.

Write portable programs with shared graphics, sound, input, file, tile,
and sprite APIs, or build for a specific machine using its native display
modes, memory, hardware sprites, and chips such as SID, POKEY, and PSG.

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
  - Amiga 500, 600, 1200
- Tandy/Radio Shack
  - CoCo 1, 2, 3
  - 1000 HX/SX/etc.
- PC
  - MS-DOS 16 bit
  - IBM PCjr
  - Windows 64 bit graphics and console (`win64`, Win Vista...Win 11)
  - Linux x64 console (`linux64`)
- Nintendo
  - NES
- Sinclair
  - ZX Spectrum
  - ZX81
  - Timex Sinclair 1000
- Many more to come!!

## Downloading the compiler

To download the latest compiled binaries, visit the **[Releases page](https://github.com/bsturk/crustybasic/releases)**.

## Downloading the assembler

Building programs requires the target's assembler; see [Assemblers](doc/USAGE.md#assemblers) in the usage guide.

## Quick start

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
crustybasic hello.cbs
```

For the C64, this writes `hello.prg` under `build/` and prints its
path. Use `-o` to choose a different path. You can also write the
command explicitly:

```bash
crustybasic build hello.cbs
```

To stop after generating assembly instead, use `compile`:

```bash
crustybasic compile hello.cbs -o hello.s
```

### Build for several targets

To build the same program for several targets when the source is portable,
leave out the `@OPTION TARGET c64` line:

```basic
CLS
POSITION 0, 0
PRINT "HELLO, {target}"
```

```bash
crustybasic build hello.cbs --set target=c64
crustybasic build hello.cbs --set target=atari800
crustybasic build hello.cbs --set target=zx81
```

Each command creates the usual runnable file for that target, and
`{target}` is replaced with its target name.

### Use target native features

This shows a Koala image and talks directly to the VIC-II and SID:

```basic
@OPTION TARGET c64
@INCLUDE_IMAGE TITLE "CRUSTY.KLA"

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

Screenshots are in the [screenshots](https://github.com/bsturk/crustybasic/tree/main/screenshots) folder.

## Community

- [Discord](https://discord.gg/JbYXZZur) - join the discussions, get access to betas
- [itch.io](https://telengard.itch.io/crustybasic) - the main crustyBASIC page

## Submitting bugs

Please submit bug reports through [GitHub Issues](https://github.com/bsturk/crustybasic/issues) or on Discord.
Include the crustyBASIC version, your operating system, the target or system you
are building for, the command you ran, and any compiler or assembler output.

A small, self contained `.cbs` file that shows the problem is the easiest for me to reproduce.

## Learn more

- [**Usage guide**](doc/USAGE.md) - commands, targets, dialects, assemblers, and examples.
- [**Language reference**](doc/LANGUAGE.md) - core language syntax.
- [**API reference**](doc/API.md) - shared runtime calls, graphics, sound, input, and target capabilities.
- [**Target documentation**](doc/targets/) - native formats, display modes, chip APIs, and machine specific limits.
