# Usage

For a 30-second intro see the [Quick Start](../README.md#quick-start).

## Examples

There are a lot of reference example listings under the examples/ dir.  
There are some that are for individual systems (crustybasic/) and some that will build on
multiple platforms (multi/).

## Compile Or Build

`compile` generates assembly text. Use it when you want to inspect
the assembly instead of making a program.

```bash
crustybasic compile examples/c64/crustybasic/color_banner.cbs -o /tmp/color_banner.s
```

`build` runs the target assembler too and creates a program/cart.

```bash
crustybasic build examples/c64/crustybasic/color_banner.cbs -o /tmp/color_banner.prg
```

`build-examples` walks `examples/`. The portable listings under
`examples/multi/` build for all targets unless you select a target.

```bash
crustybasic build-examples --set target=c64
crustybasic build-examples:crustybasic --set target=c64
```

The optional `:path-filter` builds only example files whose full path 
contains that text.

Listings can have requirements:

```basic
@REQUIRES SPRITE_KIND = SPRITE_KIND_HW @ELSE "hardware sprites are not supported"
```

`build-examples` skips an example with a warning when its requirement
is not supported for the selected target. Use `--keep-going-requires` to attempt
to build it anyway.

## Commands

```text
crustybasic compile <input.cbs> [-o output.s] [--include path] [--emit-ir] [--debug] [--show-warnings] [--crustybasic-toml path] [--set key=value]
crustybasic <input.cbs> [-o output.s] [--include path] [--emit-ir] [--debug] [--show-warnings] [--crustybasic-toml path] [--set key=value]
crustybasic build <input.cbs> [-o output.bin] [--include path] [--emit-asm output.s] [--assembler path] [--cart-size N] [--debug] [--show-warnings] [--crustybasic-toml path] [--set key=value]
crustybasic build-examples[:path-filter] [--examples-dir dir] [--output-dir dir] [--examples-asm-dir dir] [--assembler path] [--cart-size N] [--debug] [--crustybasic-toml path] [--cart-include-variants] [--keep-going-requires] [--keep-going|-k] [--jobs N|-j N] [--set key=value]
crustybasic detokenize <input> [-o output] [--from-tokenized=NAME] [--from-hex] [--repair-vnt] [--set dialect=NAME]
crustybasic test-regression [--corpus dir] [--out-dir dir] [--bless] [--no-assemble]
crustybasic targets
crustybasic target-info [system|all]
crustybasic options
crustybasic <no arg>|help|-h|--help
crustybasic version|-v|--version
```

With no command, `crustybasic <input.cbs>` is the same as
`crustybasic compile <input.cbs>`.

## Command Line Flags

| Flag | What it does |
| --- | --- |
| `-o`, `--output` | Output path. Assembly for `compile`, runnable program for `build`, text for `detokenize`. |
| `--jobs N`, `-j N` | Number of parallel example builds. Defaults to available CPU parallelism. |
| `--keep-going`, `-k` | Continue `build-examples` after failures and report them at the end. |
| `--keep-going-requires` | Build examples even when their `@REQUIRES` check does not match. |
| `--set key=value` | Override a compile option. |
| `--emit-asm path` | Keep the assembly file alongside a `build`. |
| `--emit-ir` | Print an internal debug listing to stderr. |
| `--debug` | Set `DEBUG` to 1 and enable `@ASSERT`. Without this flag, `DEBUG` is 0 and `@ASSERT` is ignored. |
| `--show-warnings` | Print compiler warnings to stderr. Warnings are suppressed by default. |
| `--assembler path` | Use a specific assembler executable. |
| `--include path` | Include a `.cbi` file before the source. Repeat for multiple files. They are included in the order specified from left to right. |
| `--crustybasic-toml path` | Use this standalone config instead of the `crustybasic.toml`. |
| `--from-tokenized=NAME` | Force a BASIC decoder for `detokenize`. |
| `--from-hex` | Read `detokenize` input as an ASCII hex dump before decoding. |
| `--repair-vnt` | For Atari BASIC SAVE files, synthesize variable names from the VVT when the VNT is corrupt. |
| `--output-dir dir` | Output directory for `build-examples` program files. Defaults to `build/`. |
| `--examples-dir dir` | Example tree for `build-examples`. Defaults to `examples/`. |
| `--examples-asm-dir dir` | Directory for assembly files from `build-examples`. |
| `--cart-size N` | Choose an exact cartridge size (`16k`, `32k`, `0x8000`, `$8000`, or decimal bytes). |
| `--cart-include-variants` | Include cartridge systems when running `build-examples`. |

Run `crustybasic options` for the `--set` keys, or
`crustybasic target-info <system>` for details about one system.

## `--set` Options

These are the current command-line option keys:

| Key | Values | What it does |
| --- | --- | --- |
| `target` | `apple2`, `atari800`, `atari5200`, `atari2600`, `c64`, `coco`, `nes`, `plus4`, `vic20` | Select a target family. |
| `system` | Any exact system name | Select an exact system. |
| `dialect` | Any known dialect | Select source compatibility rules. |
| `rom` | Target-specific ROM profile | Select which ROM services the program may use. |
| `mapper` | Target-specific mapper name | Select a cartridge mapper. |
| `optimize` | `default`, `speed`, `size` | Choose a general speed or size preference. |
| `region` | `ntsc`, `pal` | Choose timing data for region-sensitive targets. |
| `throttle` | `$XXXX`, `0xXXXX`, or decimal | Add delay to approximate interpreted BASIC timing. |
| `ram-top` | `$XXXX`, `0xXXXX`, or decimal | Set the highest address available for program data. |
| `start-program` | `$XXXX`, `0xXXXX`, or decimal | Set the outer loaded program wrapper start when the startup format has one. |
| `start-code` | `$XXXX`, `0xXXXX`, or decimal | Set the generated machine code start address. |
| `start-data` | `$XXXX`, `0xXXXX`, or decimal | Set the mutable data start address for split code/data layouts. |
| `math-real` | `auto`, `target`, `builtin` | Choose where `REAL` math comes from. |
| `math-integer` | `auto`, `target`, `builtin` | Choose where integer math comes from. |
| `numeric-mode` | `integer`, `real`, `real_narrow`, `integer_only_warn`, `integer_only_error` | Choose the default numeric policy. |
| `disk-image` | `on`, `off`, `true`, `false`, `1`, `0` | Create or skip a disk image when supported. |
| `asm-verbose` | `0`, `1`, `2` | Control how much detail appears in generated assembly files. |
| `array-base` | `0`, `1` | Choose the first array index. |

Run `crustybasic options` for the target-specific `rom`, `system`, and
`mapper` lists.

## Line Numbers

Numbered listings are detected automatically. Mixed numbered and
unnumbered program lines aren't allowed.

```basic
10 PRINT "HELLO"
20 GOTO 10
```

## Targets And Systems

Two names matter:

- **Target** - a machine family: `apple2`, `atari800`, `atari5200`,
  `atari2600`, `c64`, `coco`, `nes`, `plus4`, or `vic20`.
- **System** - an exact build choice within a target, such as
  `apple2.plus`, `c64.orig.cart`, or `coco.3`.

Default system for each target:

| Target | Default system |
| --- | --- |
| `apple2` | `apple2.plus` |
| `atari800` | `atari800.xl` |
| `atari5200` | `atari5200.orig` |
| `c64` | `c64.orig` |
| `coco` | `coco.ecb` |
| `nes` | `nes.orig` |
| `plus4` | `plus4.orig` |
| `vic20` | `vic20.orig` |

Current systems:

| System | CPU | Output | Text grid | `REAL` | Assembler | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| `apple2.plus` | 6502 | `.bin`, optional `.dsk` | 40x24 | yes | `vasm` | Apple II Plus with Applesoft ROM. |
| `apple2.int` | 6502 | `.bin`, optional `.dsk` | 40x24 | no | `vasm` | Original Apple II with Integer BASIC ROM. |
| `apple2.e` | 6502 | `.bin`, optional `.dsk` | 40x24 | yes | `vasm` | Apple IIe with 80-column switches exposed. |
| `apple2.c` | 6502 | `.bin`, optional `.dsk` | 40x24 | yes | `vasm` | Apple IIc, treated like `apple2.e` today. |
| `atari800.xl` | 6502 | `.xex`, optional `.atr` | 40x24 | yes | `mads` | Atari XL/XE with OS ROM. |
| `atari800.orig` | 6502 | `.xex`, optional `.atr` | 40x24 | yes | `mads` | Original Atari 400/800 OS profile. |
| `atari800.xl.cart` | 6502 | `.car` | 40x24 | yes | `mads` | Atari XL/XE cartridge image. |
| `atari800.orig.cart` | 6502 | `.car` | 40x24 | yes | `mads` | Atari 400/800 cartridge image. |
| `atari5200.orig` | 6502 | `.a52` | 40x24 | no | `mads` | Atari 5200 console cartridge. |
| `c64.orig` | 6502 | `.prg` | 40x25 | yes | `vasm` | Original NMOS C64. Exposes `VIC` and `SID`. |
| `c64.orig.cart` | 6502 | `.crt` | 40x25 | yes | `vasm` | Original C64 cartridge image. |
| `c64.c64c` | 6502 | `.prg` | 40x25 | yes | `vasm` | C64C profile. |
| `c64.c64c.cart` | 6502 | `.crt` | 40x25 | yes | `vasm` | C64C cartridge image. |
| `plus4.orig` | 6502 | `.prg` | 40x25 | yes | `vasm` | Commodore Plus/4. Exposes `TED`. |
| `coco.1` | 6809 | `.bin` | 32x16 | yes | `lwasm` | CoCo Color BASIC. Some math functions need ECB. |
| `coco.ecb` | 6809 | `.bin` | 32x16 | yes | `lwasm` | CoCo Extended Color BASIC. |
| `coco.ecb.cart` | 6809 | `.ccc` | 32x16 | yes | `lwasm` | CoCo Extended Color BASIC cartridge. |
| `coco.1.cart` | 6809 | `.ccc` | 32x16 | yes | `lwasm` | CoCo Color BASIC cartridge. |
| `coco.3` | 6809 | `.bin` | 32x16 | yes | `lwasm` | CoCo 3 with `GIME` registers exposed. |
| `coco.3.cart` | 6809 | `.ccc` | 32x16 | yes | `lwasm` | CoCo 3 cartridge. |
| `dos16.exe` | 8086 | `.exe`, optional `.img` | 80x25 | yes | `nasm` | Generic 16 bit DOS executable. The `.img` is a mountable FAT12 data disk. |
| `pcjr.exe` | 8086 | `.exe`, optional `.img` | 40x25 | yes | `nasm` | IBM PCjr executable. The `.img` is a mountable FAT12 data disk. |
| `tandy1000.exe` | 8086 | `.exe`, optional `.img` | 40x25 | yes | `nasm` | Tandy 1000 executable. The `.img` is a mountable FAT12 data disk. |
| `nes.orig` | 6502 | `.nes` | 32x30 | no | `vasm` | iNES cartridge. NROM by default. |
| `vic20.orig` | 6502 | `.prg` | 22x23 | yes | `vasm` | Unexpanded VIC-20. |
| `vic20.3k` | 6502 | `.prg` | 22x23 | yes | `vasm` | VIC-20 with 3K expansion. |
| `vic20.8k` | 6502 | `.prg` | 22x23 | yes | `vasm` | VIC-20 with 8K expansion. |
| `vic20.16k` | 6502 | `.prg` | 22x23 | yes | `vasm` | VIC-20 with 16K expansion. |
| `vic20.24k` | 6502 | `.prg` | 22x23 | yes | `vasm` | VIC-20 with 24K expansion. |

Names are case-insensitive. Each target has its own text encoding
such as PETSCII or ATASCII, so stick to plain ASCII for portable
programs. See the target docs under [`targets/`](targets/) for more.

## Picking A Target Or System

Use source options or CLI flags:

```basic
@OPTION TARGET c64
@OPTION SYSTEM coco.ecb
```

```bash
crustybasic compile examples/multi/strings.cbs --set target=coco -o /tmp/strings.s
crustybasic compile examples/multi/strings.cbs --set system=coco.ecb -o /tmp/strings.s
crustybasic build-examples --set target=c64
```

Config profiles can set `target` or `system` too; see
[Folder-Wide Config](#folder-wide-config). CLI `--set` always wins. A
dialect default only applies when nothing more specific selected a
target or system.

## Cartridge Mappers

NES defaults to NROM. UxROM, MMC1, and MMC3 enable banked PRG ROM:

```basic
@OPTION TARGET nes
@OPTION MAPPER uxrom
```

Equivalent on the CLI:

```bash
crustybasic build program.cbs --set target=nes --set mapper=uxrom -o /tmp/program.nes
```

NES builds can package external CHR ROM bytes:

```bash
crustybasic build program.cbs --set target=nes --set chr-rom=tiles.chr -o /tmp/program.nes
```

When `chr-rom` is set, runtime CHR RAM upload helpers are no-ops; put
the font and tile data the program uses in the supplied CHR file.
For MMC3, source can place 8K CHR ROM banks directly:

```basic
@OPTION TARGET nes
@OPTION MAPPER mmc3
@BANK CHR 0 "tiles.chr"
```

Other banked options:

| System | Standard | Banked alternative |
| --- | --- | --- |
| `atari800.xl.cart`, `atari800.orig.cart` | 8K/16K cart | `xegs32` (XEGS 32K) |
| `c64.orig.cart`, `c64.c64c.cart` | 8K/16K cart | `supergames` (Super Games 64K) |
| `coco.ecb.cart`, `coco.1.cart`, `coco.3.cart` | Program Pak | `banked_16k` (128K banked Pak) |
| `nes.orig` | `nrom` | `uxrom`, `mmc1`, `mmc3` |

## Banked Builds

Use `@BANK PRG N ... @ENDBANK` for code and DATA banks. See
[`@BANK` / `@ENDBANK`](LANGUAGE.md#bank--endbank).

Pass `--emit-asm` to keep the assembly. Switched banks get sibling
files with a `.bankN.s` suffix:

```text
/tmp/program_fixed.s
/tmp/program_fixed.bank0.s
/tmp/program_fixed.bank1.s
```

The build also prints per-bank size estimates:

```text
cart format: vasm_ines_uxrom (49152 bytes)
fixed bank: 1804 / 16384 bytes estimated
switched bank 0: 0 / 16384 bytes estimated
switched bank 1: 1697 / 16384 bytes estimated
```

Current banked mappers reject calls from one switched bank directly to
another switched bank. This applies to `uxrom`, `mmc1`, `mmc3`,
`xegs32`, `supergames`, and `banked_16k`. Put shared helper PROCs in
the main fixed area so switched-bank code can call them.

## Dialects

Select a dialect with `--set dialect=...`, `@OPTION DIALECT`, or a
config profile. The available dialect names are listed in the language
reference: [Dialects](LANGUAGE.md#dialects). Config profile discovery
and layering are covered in [Folder-Wide Config](#folder-wide-config).

### Tokenized BASIC Files

Use `detokenize` to inspect tokenized BASIC files:

```bash
crustybasic detokenize PROGRAM.BAS --set dialect=atari_basic -o /tmp/program.bas
```

Without `-o`, the decoded text goes to stdout. If no dialect is set,
`detokenize` tries known decoders and otherwise treats the input as
plain text. Use `--from-tokenized=NAME` to force a decoder. Use
`--from-hex` for ASCII hex dumps and `--repair-vnt` to recover Atari
BASIC SAVE files with damaged variable names.

## Folder-Wide Config

The nearest `crustybasic.config.toml` applies to sources in that folder
and its children. A per-file `<name>.config.toml` beside the source is
layered on top of the folder file for that source, overriding only the
values it sets. Config discovery starts from the entry source file.
Sibling config files beside `@INCLUDE`d files are ignored.

Config files use the same names and values as the public `--set`
options, except that `asm-verbose` is not a config key. They also accept
`build-examples = false` to skip a source during `build-examples`, and
`startup` to select a named startup template from the target manifests
(also available in source as `@OPTION STARTUP`).

Config values use TOML syntax, so string choices are quoted and booleans
are `true` or `false`. Config files also accept `include`,
equivalent to repeated `--include` flags. Includes from layered config
files are kept in base-first order. Paths in config files are relative
to the config file.

Any config setting can be limited to one target or system with
`[target.NAME]` or `[system.NAME]`. Matching system sections override
matching target sections.

```toml
# crustybasic.config.toml
target = "c64"
array-base = 1
include = "my_style.cbi"

[target.vic20]
start-program = "$2001"
start-data = "$6000"
ram-top = "$1C00"
optimize = "size"

[target.atari2600]
optimize = "size"

[system.vic20.orig]
build-examples = false
```

```toml
# game.config.toml for game.cbs
target = "nes"
mapper = "nrom"
```

For `game.cbs`, `target` is `nes`, `mapper` is `nrom`, and
`array-base` remains `1`. `my_style.cbi` is still included before the
source and can hold folder-wide spelling preferences:

```basic
' my_style.cbi
@REWRITE ENDW = ENDWHILE
@REWRITE INKEY$ = INKEY
```

## Assemblers

`build` needs the assembler expected by the selected system: `vasm`
for Apple II, Commodore 64/Plus/4/VIC-20, and NES; `mads` for Atari;
`lwasm` for CoCo; `nasm` for DOS 16/PCjr/Tandy 1000. The
[systems table](#targets-and-systems) lists the assembler per system.

By default no configuration is needed: the bundled assembler is
located automatically next to the `crustybasic` executable. For each
build the compiler probes these locations in order and uses the first
one that exists:

```text
tools/assemblers/<assembler>/<platform>/<executable>
tools/assemblers/<assembler>/<platform>/bin/<executable>
```

`<platform>` matches the machine running the compiler:
`macos-aarch64` (Apple Silicon), `macos-x86_64` (Intel Mac),
`linux-x86_64`, or `windows-x86_64`. `<executable>` is the assembler
binary (`vasm6502_oldstyle`, `mads`, `lwasm`, `nasm`), with `.exe`
appended on Windows.

So a Linux bundle finds the C64 assembler at
`tools/assemblers/vasm/linux-x86_64/vasm6502_oldstyle`. Release
bundles create these platform directories for you - drop an assembler
binary in and it's picked up with no configuration. The bundled
helper tools (`cb-cart-wrap` and friends) are found the same
way, under `tools/bin/` next to the compiler.

To use a different executable, pass one directly:

```bash
crustybasic build hello.cbs --assembler /path/to/vasm6502_oldstyle -o hello.prg
```

Or create a `crustybasic.toml` next to the compiler and pin it per
target:

```toml
[targets.c64]
assembler = "/path/to/vasm6502_oldstyle"
```

(When building from the source tree, the same override goes in
`Cargo.toml` as `[package.metadata.crustybasic.targets.c64]`.)

An explicit `--assembler` flag beats the config entry, which beats the
automatic bundled lookup. Relative override paths resolve against the
directory you run `crustybasic` from, so prefer absolute paths there;
the automatic lookup always resolves against the install directory and
works from anywhere. Use `--crustybasic-toml path` to test a config
without moving it beside the compiler.

## Extra Output Files

Some builds run helper tools after the assembler:

- Apple II `.bin` builds can also produce a bootable `.dsk` when
  `--set disk-image=true`, AppleCommander, and a JRE are available.
  Set `APPLECOMMANDER_JAR` to point at AppleCommander if needed.
- Atari 800 `.xex` builds can also produce a bootable `.atr` when
  `--set disk-image=true` and the vendored `mkatr` tools are built.
- DOS 16, PCjr, and Tandy 1000 `.exe` builds try to produce a
  mountable 360K FAT12 `.img` by default. Use
  `--set disk-image=false` to skip it. The `.img` is not bootable;
  boot DOS separately and mount it as a data floppy in the emulator.
- Cartridge systems use the bundled `cb-cart-wrap` helper to
  write `.crt`, `.car`, `.a52`, `.ccc`, or `.nes` images.

Optional disk-image steps warn and continue if their external tool is
missing. The main `.bin`, `.xex`, or `.exe` still gets built.

## Compile Options In Source

Common source options:

```basic
@OPTION TARGET c64
@OPTION DIALECT crustybasic
@OPTION ARRAY_BASE 1
@OPTION MATH_REAL BUILTIN
@OPTION MATH_INTEGER BUILTIN
@OPTION NUMERIC_MODE INTEGER
@OPTION REGION_NTSC
@OPTION THROTTLE 10
```

`@OPTION TARGET` or `@OPTION SYSTEM` must appear before target-specific
statements. `@OPTION DIALECT` applies that dialect's defaults only to
options you have not already set.


`crustybasic options` prints the authoritative list of CLI options.

## Throttling

Native code is much faster than interpreted BASIC. Throttling adds a
small delay to each statement so timing-sensitive games feel closer to
the original interpreter.

```basic
@OPTION THROTTLE 8
```

```bash
crustybasic compile game.bas --set dialect=applesoft_basic --set throttle=8 -o /tmp/game.s
```

`0` disables throttling. Routines marked with `@ASYNC`, and routines
only called from them, are left unthrottled. Start low and tune by
feel.

## Examples

| Path | What's in it |
| --- | --- |
| `examples/multi/` | Portable crustyBASIC programs. |
| `examples/<target>/crustybasic/` | Target-specific crustyBASIC programs. |
| `examples/<target>/dialects/<dialect>/` | Compatibility dialect listings. |

Compile an example using its directory [config profile](#folder-wide-config):

```bash
crustybasic compile examples/apple2/dialects/applesoft_basic/calculator.bas -o /tmp/calculator.s
```

Or with everything explicitly specified:

```bash
crustybasic compile examples/apple2/dialects/applesoft_basic/calculator.bas --set dialect=applesoft_basic --set system=apple2.plus -o /tmp/calculator.s
```
The core language reference is in [LANGUAGE.md](LANGUAGE.md), and the
portable API reference is in [API.md](API.md).
