# Usage

Start with the [Quick Start](../README.md#quick-start) if you just want
to compile and run a small program.

## Compile or build

By default, crustyBASIC builds a program you can run:

```bash
crustybasic examples/c64/crustybasic/color_banner.cbs -o color_banner.prg
```

You can write `build` explicitly to do the same thing:

```bash
crustybasic build examples/c64/crustybasic/color_banner.cbs -o color_banner.prg
```

`compile` writes an assembly file and stops there. Use it when you want
to inspect the assembly written by crustyBASIC or run the assembler
yourself.

```bash
crustybasic compile examples/c64/crustybasic/color_banner.cbs -o color_banner.s
```

If you leave out `-o`, crustyBASIC chooses the output path and prints
every path it writes. Examples go under `build/` by default.

## Building examples

`build-examples` searches `examples/` and its subfolders for `.bas` and
`.cbs` files. When an example does not choose a target, the command tries
it on one default system for each target. When an example does choose a
target or system, only compatible systems are used. Config settings and
`@REQUIRES` can skip individual builds.

```bash
crustybasic build-examples --set target=c64
crustybasic build-examples:crustybasic --set target=c64
```

Add `:path-filter` to build only examples whose full path contains that
text.

An example can say what it needs:

```basic
@REQUIRES SPRITE_KIND = SPRITE_KIND_HW @ELSE "the target's own sprites are required"
```

If a requirement is false for the selected target, `build-examples`
skips that example and prints a warning. Use `--keep-going-requires` to
ignore `@REQUIRES` and try the build anyway. Add `--keep-going` if one
failed build should not stop the rest.

## Commands

```text
crustybasic <input.cbs> [-o output.bin] [--include path] [--emit-asm output.s] [--assembler path] [--cart-size N] [--size-report] [--debug] [--timings] [--show-warnings] [--crustybasic-toml path] [--set key=value]
crustybasic build <input.cbs> [-o output.bin] [--include path] [--emit-asm output.s] [--assembler path] [--cart-size N] [--size-report] [--debug] [--timings] [--show-warnings] [--crustybasic-toml path] [--set key=value]
crustybasic compile <input.cbs> [-o output.s] [--include path] [--emit-ir] [--debug] [--timings] [--show-warnings] [--crustybasic-toml path] [--set key=value]
crustybasic build-examples[:path-filter] [--examples-dir dir] [--output-dir dir] [--examples-asm-dir dir] [--assembler path] [--cart-size N] [--debug] [--timings] [--show-warnings] [--crustybasic-toml path] [--cart-include-variants] [--keep-going-requires] [--keep-going|-k] [--jobs N|-j N] [--set key=value]
crustybasic detokenize <input> [-o output] [--from-tokenized=NAME] [--from-hex] [--set dialect=NAME]
crustybasic targets
crustybasic target-info [system|all]
crustybasic options
crustybasic <no arg>|help|-h|--help
crustybasic version|-v|--version
```

You can leave out the `build` command. `crustybasic <input.cbs>` means
the same thing as `crustybasic build <input.cbs>`.

## File extensions

| Extension | Meaning |
| --- | --- |
| `.cbs` | crustyBASIC source |
| `.bas` | BASIC source, commonly used for compatibility listings |
| `.cbi` | crustyBASIC include |
| `.cbfont` | crustyBASIC font |

## Command line flags

| Flag | What it does |
| --- | --- |
| `-o`, `--output` | Choose the output path: assembly for `compile`, a runnable program for `build`, or text for `detokenize`. |
| `--jobs N`, `-j N` | Run this many example builds at once. The default uses the available processor cores. |
| `--keep-going`, `-k` | Keep building examples after a failure, then report all failures at the end. |
| `--keep-going-requires` | Build examples even when their `@REQUIRES` check does not match. |
| `--set key=value` | Set or replace a compile option. |
| `--emit-asm path` | Keep the assembly file produced during a `build`. |
| `--size-report` | After a successful `build`, show output sizes, memory use, cartridge banks, and estimated free space. Exact, estimated, and partial values are labeled. |
| `--emit-ir` | Print a detailed listing for diagnosing compilation problems. |
| `--timings` | Show how long compilation takes, split into its main steps. |
| `--debug` | Set `DEBUG` to 1 and enable `@ASSERT`. Without it, `DEBUG` is 0 and `@ASSERT` is ignored. |
| `--show-warnings` | Print crustyBASIC warnings. They are hidden by default. |
| `--assembler path` | Use this assembler executable for the selected system. It must be compatible with that system. |
| `--include path` | Include a `.cbi` file before the source. Repeat the flag for more files. Files are included from left to right. |
| `--crustybasic-toml path` | Use a different `crustybasic.toml` tool config. |
| `--from-tokenized=NAME` | Tell `detokenize` which tokenized BASIC format to read. |
| `--from-hex` | Make `detokenize` read its input as an ASCII hex dump before decoding it. |
| `--output-dir dir` | Put programs from `build-examples` directly in this folder instead of the usual output folders. |
| `--examples-dir dir` | Search this example tree with `build-examples`. The default is `examples/`. |
| `--examples-asm-dir dir` | Put assembly files from `build-examples` in this folder. |
| `--cart-size N` | Set the exact size of a cartridge stored as one fixed area (`16k`, `32k`, `0x8000`, `$8000`, or decimal bytes). |
| `--cart-include-variants` | For examples that do not choose a target, build every exact system variant instead of one system per target. |
| `--out-dir dir` | Put regression output in this folder. |
| `--bless` | Replace the expected messages for failing regression cases with their current messages. |
| `--no-assemble` | Skip regression cases that need an assembler. |

Run `crustybasic options` to see the `--set` keys. Run
`crustybasic target-info <system>` to inspect one system, including its
usable memory regions.

## Compile options

These options can be set from the command line with `--set key=value`:

| Key | Values | What it does |
| --- | --- | --- |
| `target` | Any target shown by `crustybasic options` | Choose a target family. |
| `system` | Any exact system shown by `crustybasic options` | Choose an exact system. |
| `dialect` | Any dialect shown by `crustybasic options` | Choose a set of source compatibility rules. |
| `rom` | Target specific ROM name | Choose which ROM services the program can use. |
| `mapper` | Target specific mapper name | Choose how cartridge storage is arranged. |
| `output-type` | Target specific output name | Choose the runnable format to build. |
| `tile-backend` | `cell`, `hardware`, `bitmap`, `kernel` | Choose which kind of screen the TILE API uses. |
| `optimize` | `default`, `speed`, `size` | Favor speed, size, or the default balance. |
| `math-real` | `auto`, `target`, `builtin` | Choose automatically, use the target's `REAL` math, or use crustyBASIC's. |
| `builtin-real` | `auto`, `q16_16`, `q24_8`, `binary64` | Choose how crustyBASIC stores `REAL` values. |
| `math-integer` | `auto`, `target`, `builtin` | Choose automatically, use the target's integer math, or use crustyBASIC's. |
| `numeric-mode` | `integer`, `real`, `real_narrow`, `integer_only_warn`, `integer_only_error` | Choose default numeric types and whether `REAL` is allowed. |
| `type-policy` | Any policy shown by `crustybasic options` | Choose suffix meanings and types for names without an explicit type. |
| `array-base` | `0`, `1` | Choose the first array index. |
| `region` | `REGION_NTSC`, `REGION_PAL` | Choose timing data for systems that differ by region. |
| `throttle` | `0` through `65535` | Add a delay that approximates interpreted BASIC timing. |
| `ram-top` | Decimal or hex with `$` or `0x` | Set the highest address crustyBASIC may use for variables and other changeable data. |
| `start-program` | Decimal or hex with `$` or `0x` | Set the program's load address when the selected format supports it. |
| `start-code` | Decimal or hex with `$` or `0x` | Set the address where program code starts. |
| `start-data` | Decimal or hex with `$` or `0x` | Set the address where changeable data starts when code and data are kept separate. |
| `nes-chr-rom` | File path | Include NES graphics data from a CHR ROM file. |
| `disk-image` | `on`, `off`, `true`, `false`, `1`, `0` | Create or skip a disk image when the system supports one. |
| `asm-verbose` | `0`, `1`, `2` | Choose how much detail appears in assembly files written by crustyBASIC. |

The available `rom`, `system`, and `mapper` names depend on the target.
Run `crustybasic options` to see them.

For `tile-backend`, `cell` uses the character screen, `hardware` uses
the target's built in tile display, `bitmap` draws tiles in a bitmap
graphics mode, and `kernel` uses a target specific tile mode.

`numeric-mode` values have these meanings:

| Value | What it does |
| --- | --- |
| `integer` | Names without a type default to integers. `REAL` can still be used explicitly. |
| `real` | Names without a type default to `REAL`. |
| `real_narrow` | Names without a type default to `REAL`, but variables used only for whole numbers may be stored as integers. |
| `integer_only_warn` | Names without a type default to integers and `REAL` use produces a warning shown by `--show-warnings`. |
| `integer_only_error` | Names without a type default to integers and `REAL` use is an error. |

`type-policy` controls what suffixes such as `%`, `&`, and `!` mean, and
the types assigned to names without a suffix. The target or dialect
normally chooses it for you. Read [Types](LANGUAGE.md#types) before
changing it.

Set `math-real=builtin` to use crustyBASIC's own `REAL` math.
`builtin-real` then chooses its format. It has no effect when
`math-real=target`.

| `builtin-real` | Meaning |
| --- | --- |
| `auto` | Use the target's preferred crustyBASIC format. |
| `q16_16` | Fixed point with more fractional precision. |
| `q24_8` | Fixed point with more integer range. |
| `binary64` | Standard eight byte floating point, where the target supports it. |

The following options work in config files and source code, but not
with `--set`:

| Key | Values | What it does |
| --- | --- | --- |
| `string-default-capacity` | `1` through `256` | Set the usable capacity of a plain `STRING` declaration. |
| `string-bounds-checks` | `true` or `false` | Check whole string assignments and appends while the program runs. The default is `false`. |
| `startup` | Name listed on the target page | Choose how the program starts on the target. |
| `memory-regions` | One or more target region names | Let crustyBASIC use these named memory regions. |
| `memory-actions` | One or more target action names | Request extra setup named by the target page. |

Any setup required by a memory region is enabled with it.

## Memory regions

Run `crustybasic target-info <system>` to see the extra memory regions
available on that system. For each region, it shows:

- its address ranges
- whether it can hold code, data, or both
- how it becomes available to the program
- whether it is enabled automatically
- any setup it needs before use
- any features you give up by using it

Enabling a region lets crustyBASIC use that extra memory. crustyBASIC
chooses what to place there; enabling a region does not place a
particular variable or PROC there.

In source code, enable a region with:

```basic
@OPTION MEMORY_REGION name
```

In the program's config file, use:

```toml
memory-regions = ["name"]
```

Repeat `@OPTION MEMORY_REGION` to enable more than one region in source,
or add more names to the config array. You do not need to list regions
that `target-info` marks as enabled automatically.

Some targets offer extra setup that can be requested without enabling a
memory region:

```basic
@OPTION MEMORY_ACTION name
```

```toml
memory-actions = ["name"]
```

Setup required by a memory region is added automatically. Use
`MEMORY_ACTION` only when the target page names separate setup that your
program needs.

## Line numbers

crustyBASIC detects numbered listings automatically. A program cannot
mix numbered and unnumbered lines.

```basic
10 PRINT "HELLO"
20 GOTO 10
```

## Targets and systems

You can choose a target family or one exact system:

- **Target** means a machine family, such as `apple2`, `c64`, or `coco`.
  When you choose a target, crustyBASIC uses that family's default
  system.
- **System** means one exact machine setup in a family, such as
  `apple2.plus`, `c64.orig`, or `coco.3`.

Run these commands to see the names currently supported:

```bash
crustybasic options
crustybasic targets
crustybasic target-info c64.orig
crustybasic target-info all
```

`options` lists the valid target, system, ROM, dialect, mapper, and type
policy names. `targets` gives a short summary of each exact system,
including its processor, supported assemblers, runnable file extensions,
and named chips that programs can use.

`target-info` gives the details for one exact system: its default output,
available memory, character codes, numbers available for custom
characters, chips, and cartridge support. Pass `all`, or leave out the
name, to show every system.

Use `output-type` on the command line or `OUTPUT_TYPE` in source to
choose between cartridge, disk, and other formats offered by a system.
Names are not case sensitive. Each target has its own text encoding,
such as PETSCII or ATASCII, so use plain ASCII in portable programs. The
[`targets/`](targets/) pages cover setup, APIs, and limitations for
individual targets.

## Picking a target or system

Choose a target in the source:

```basic
@OPTION TARGET c64
```

Or choose an exact system:

```basic
@OPTION SYSTEM coco.ecb
```

```bash
crustybasic compile examples/__portable__/strings.cbs --set target=coco -o strings.s
crustybasic compile examples/__portable__/strings.cbs --set system=coco.ecb -o strings.s
crustybasic build-examples --set target=c64
```

You can also set `target` or `system` in a config file; see
[Folder config](#folder-config). A command line `--set` takes priority
over config and source settings. A target or system supplied by the
dialect only applies when nothing else has chosen one.

## Cartridge mappers

A cartridge mapper controls how a cartridge is divided. A simple mapper
uses one fixed area. A banked mapper divides a larger cartridge into
numbered banks that the program uses as needed.

`MAPPER` only applies to cartridge output. Choose one in source:

```basic
@OPTION OUTPUT_TYPE cart
@OPTION MAPPER name
```

Or choose it on the command line:

```bash
crustybasic build program.cbs --set output-type=cart --set mapper=name
```

You can leave out `OUTPUT_TYPE` when cartridge is already the system's
default output. Output and mapper names depend on the target.

Run `crustybasic options` to see the available mapper names. The
[`targets/`](targets/) pages explain each target's default layout, bank
size, cartridge limits, and any extra files it needs.

## Banked builds

A banked cartridge is divided into numbered sections that are not all
available at once. Some banks stay available, while others are selected
as needed. The matching [target page](targets/) explains when PROCs and
DATA can be used across banks.

With a banked mapper, put `@BANK N` and `@ENDBANK` around the PROCs and
DATA that belong in program bank `N`. Bank numbers start at `0`.

```basic
@BANK 0
DATA 1, 2, 3
@ENDBANK
```

When `@INCLUDE_BIN` or `@INCLUDE_IMAGE` appears inside the block, its
`CONST` array goes into the same bank.

These are the controls you will normally use:

| Control | What it does |
| --- | --- |
| `@OPTION OUTPUT_TYPE name` or `--set output-type=name` | Choose a cartridge output. |
| `@OPTION MAPPER name` or `--set mapper=name` | Choose the cartridge layout. |
| `@BANK N ... @ENDBANK` | Put the enclosed PROCs, CONST arrays, and DATA in program bank `N`. |
| `--emit-asm path` | Keep the assembly files created for the main program and its banks. |
| `--size-report` | Show used and available space in the cartridge banks. |

See [`@BANK` / `@ENDBANK`](LANGUAGE.md#bank--endbank) for the exact
source rules. The [`targets/`](targets/) pages explain the layouts
available on each target.

## Dialects

Choose a dialect with `--set dialect=...`, `@OPTION DIALECT`, or a
config file. Run `crustybasic options` to see the dialects supported by
your copy of crustyBASIC. See [Dialects](LANGUAGE.md#dialects) for the
source rules a dialect can change.

### Tokenized BASIC files

Use `detokenize` to turn a tokenized BASIC file back into readable
source:

```bash
crustybasic detokenize PROGRAM.BAS --set dialect=atari_basic -o decoded_program.bas
```

Without `-o`, the decoded text is printed in the terminal. If you do not
choose a dialect, `detokenize` tries the supported tokenized formats and
then treats the input as plain text when none match. Use
`--from-tokenized=NAME` to choose the format yourself. Add `--from-hex`
when the input is an ASCII hex dump.

## Folder config

For a source file, crustyBASIC looks in its folder and then each parent
folder for the nearest `crustybasic.config.toml`. That config applies to
source files in its folder and subfolders.

You can also put `<name>.config.toml` beside one source file. For
`game.cbs`, that would be `game.config.toml`. Its settings are applied
after the folder config, and it only replaces values that it sets.
Config files beside files brought in with `@INCLUDE` are ignored.

Most compile option keys use the same spelling as `--set`.
`asm-verbose` is command line only and cannot be used here. Config files
use TOML syntax, so strings need quotes and booleans are `true` or
`false`. Named values use their source spelling:

```toml
region = "REGION_PAL"
tile-backend = "TILE_BITMAP"
```

Config files also support these project settings:

| Key | What it does |
| --- | --- |
| `include` | Include one path, or an array of paths, before the source. Relative paths start from the config file's folder. |
| `build-examples` | Set this to `false` to skip the source during `build-examples`. |
| `build-example-targets` | In the top example folder, limit example builds to these target families. |
| `bit-on-char`, `bit-off-char` | Choose the two characters used in visual binary literals. |

The options that only work in config and source are listed under
[Compile options](#compile-options). A config file can also define a custom
`[type-policy.NAME]`; see [Types](LANGUAGE.md#types).

When both folder and file config apply, a single value in the file config
replaces the folder value. Lists such as `include`, `memory-regions`, and
`memory-actions` are combined instead, with the folder config's items
first.

You can limit any setting except `target` or `system` to one target or
system by putting it under `[target.NAME]` or `[system.NAME]`. When both
sections match, a value in the system section replaces the same value
from the target section. Lists are combined.

Target and system sections are selected by the command line, settings at
the top of the config file, or the dialect default. An `@OPTION TARGET`
or `@OPTION SYSTEM` inside the BASIC source does not select config
sections.

```toml
# crustybasic.config.toml
target = "c64"
array-base = 1
include = "my_style.cbi"

[target.vic20]
optimize = "size"
string-default-capacity = 32

[target.atari2600]
optimize = "size"

[system.vic20.orig]
build-examples = false
```

To limit a full example run, put `build-example-targets` in the
`crustybasic.config.toml` at the top of that example folder:

```toml
build-example-targets = ["apple2", "c64", "coco"]
```

```toml
# game.config.toml for game.cbs
target = "nes"
optimize = "speed"
```

For `game.cbs`, the file config changes `target` to `nes` and
`optimize` to `speed`. It does not mention `array-base`, so that remains
`1`. `my_style.cbi` is still included before the source and can define
spelling preferences for the folder:

```basic
' my_style.cbi
@REWRITE ENDW = ENDWHILE
@REWRITE INKEY$ = INKEY
```

## Assemblers

`build` needs an assembler that understands the selected system's
assembly syntax. Run `crustybasic targets` to see the assembler and
executable name expected for each system.

crustyBASIC looks for that executable in the paths set for the target. A
release bundle contains the matching
`tools/assemblers/<assembler>/<platform>/` folder, but not the assembler
itself. Put a compatible executable there, or give its path with
`--assembler`. If the system needs another build tool and it is missing,
`build` tells you what it needs.

To choose an assembler for one build, pass its path directly:

```bash
crustybasic build hello.cbs --assembler /path/to/vasm6502_oldstyle -o hello.prg
```

To keep the choice, create `crustybasic.toml` beside the crustyBASIC
executable and set the assembler for that target. This is the tool
config, not the `crustybasic.config.toml` used by individual programs:

```toml
[targets.c64]
assembler = "/path/to/vasm6502_oldstyle"
```

`--assembler` takes priority over the tool config. A relative path starts
from the folder where you run `crustybasic`, so an absolute path is
usually safer. When neither is set, crustyBASIC checks its install
folder.

Use `--crustybasic-toml path` to try a tool config without putting it
beside the crustyBASIC executable.

## Outputs and extra files

A system can support more than one runnable format. `target-info` shows
the default format and any cartridge formats. The matching page under
[`targets/`](targets/) describes other formats when they are available.
Choose a format by name:

```bash
crustybasic target-info c64.orig
crustybasic build program.cbs --set output-type=cart
```

The extension on an explicit `-o` path can also choose the format when
only one format uses that extension. Use `output-type` when the filename
should not decide.

Some formats create extra files along with the main program. `build`
prints every path it writes. `disk-image=true` asks for a disk image. It
uses the system's disk format when only one is available:

```bash
crustybasic build program.cbs --set disk-image=true
```

If a system has more than one disk format, choose the exact one with
`output-type`. Use `disk-image=false` to skip disk images that would
otherwise be created automatically.

The build fails when a required assembler or other tool is missing. If an
optional step cannot run, the build warns you but keeps the main output.
The [`targets/`](targets/) pages explain what each format contains and
what it needs when the program runs.

## Compile options in source

These are some commonly used source options:

```basic
@OPTION TARGET c64
@OPTION DIALECT crustybasic
@OPTION ARRAY_BASE 1
@OPTION MATH_REAL BUILTIN
@OPTION BUILTIN_REAL Q16_16
@OPTION MATH_INTEGER BUILTIN
@OPTION NUMERIC_MODE INTEGER
@OPTION REGION REGION_NTSC
@OPTION STRING_DEFAULT_CAPACITY 64
@OPTION STRING_BOUNDS_CHECKS TRUE
@OPTION THROTTLE 10
```

Put `@OPTION TARGET` or `@OPTION SYSTEM` before any target specific
statements. `@OPTION DIALECT` fills in the dialect's defaults without
replacing options you have already set.

Run `crustybasic options` for the available command line options. See
[source options](LANGUAGE.md#compiler-options) and
the [`targets/`](targets/) pages for target specific choices.

## Throttling

Compiled code runs much faster than interpreted BASIC. Throttling adds a
small delay to each statement so games that depend on interpreter speed
feel closer to the original.

```basic
@OPTION THROTTLE 8
```

```bash
crustybasic compile game.bas --set dialect=applesoft_basic --set throttle=8 -o game.s
```

Set throttling to `0` to turn it off. Code that can run from an `@ASYNC`
PROC is not throttled. Start with a small value and adjust it by feel.

## Example layout

| Path | What's in it |
| --- | --- |
| `examples/__portable__/` | Portable crustyBASIC programs. |
| `examples/<target>/crustybasic/` | Target specific crustyBASIC programs. |
| `examples/<target>/dialects/<dialect>/` | Compatibility dialect listings. |

The config file in an example's folder normally chooses the right
target and dialect:

```bash
crustybasic compile examples/apple2/dialects/applesoft_basic/calculator.bas -o calculator.s
```

You can also choose everything on the command line:

```bash
crustybasic compile examples/apple2/dialects/applesoft_basic/calculator.bas --set dialect=applesoft_basic --set system=apple2.plus -o calculator.s
```

For the language itself, see [LANGUAGE.md](LANGUAGE.md). For portable
library routines and constants, see [API.md](API.md).
