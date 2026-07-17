# Usage

For a quick intro see the [Quick Start](../README.md#quick-start).

## Compile Or Build

`compile` generates assembly text. Use it when you want to inspect
the assembly instead of making a program.

```bash
crustybasic compile examples/c64/crustybasic/color_banner.cbs -o /tmp/color_banner.s
```

`build` runs the selected system's assembler and creates its runnable
output.

```bash
crustybasic build examples/c64/crustybasic/color_banner.cbs -o /tmp/color_banner.prg
```

Without `-o`, both commands choose a path from the project output
configuration and print each path they write. Sources under `examples/`
use the artifact layout under `build/` by default.

## Building Examples

`build-examples` recursively finds `.bas` and `.cbs` files under
`examples/`. A listing with no target selection is tried on one default
system per target. Listings that select a target or system build only for
compatible systems. Config and `@REQUIRES` can skip individual builds.

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
is false for the selected target. Use `--keep-going-requires` to ignore
`@REQUIRES` and attempt the build. Add `--keep-going` if a failed build
should not stop the run.

## Commands

```text
crustybasic compile <input.cbs> [-o output.s] [--include path] [--emit-ir] [--debug] [--timings] [--show-warnings] [--crustybasic-toml path] [--set key=value]
crustybasic <input.cbs> [-o output.s] [--include path] [--emit-ir] [--debug] [--timings] [--show-warnings] [--crustybasic-toml path] [--set key=value]
crustybasic build <input.cbs> [-o output.bin] [--include path] [--emit-asm output.s] [--assembler path] [--cart-size N] [--size-report] [--debug] [--timings] [--show-warnings] [--crustybasic-toml path] [--set key=value]
crustybasic build-examples[:path-filter] [--examples-dir dir] [--output-dir dir] [--examples-asm-dir dir] [--assembler path] [--cart-size N] [--debug] [--timings] [--show-warnings] [--crustybasic-toml path] [--cart-include-variants] [--keep-going-requires] [--keep-going|-k] [--jobs N|-j N] [--set key=value]
crustybasic detokenize <input> [-o output] [--from-tokenized=NAME] [--from-hex] [--set dialect=NAME]
crustybasic targets
crustybasic target-info [system|all]
crustybasic options
crustybasic <no arg>|help|-h|--help
crustybasic version|-v|--version
```

With no command, `crustybasic <input.cbs>` is the same as
`crustybasic compile <input.cbs>`.

## File Extensions

| Extension | Meaning |
| --- | --- |
| `.cbs` | CrustyBASIC source |
| `.bas` | BASIC source, commonly used for compatibility listings |
| `.cbi` | CrustyBASIC include |
| `.cbf` | CrustyBASIC font |

## Command Line Flags

| Flag | What it does |
| --- | --- |
| `-o`, `--output` | Output path. Assembly for `compile`, runnable program for `build`, text for `detokenize`. |
| `--jobs N`, `-j N` | Number of parallel example builds. Defaults to available CPU parallelism. |
| `--keep-going`, `-k` | Continue `build-examples` after failures and report them at the end. |
| `--keep-going-requires` | Build examples even when their `@REQUIRES` check does not match. |
| `--set key=value` | Override a compile option. |
| `--emit-asm path` | Keep the assembly file alongside a `build`. |
| `--size-report` | Show artifact sizes, memory region use, cart banks, and available section estimates after a successful `build`. Exact, estimated, and partial values are labeled. |
| `--emit-ir` | Print an internal debug listing to stderr. |
| `--debug` | Set `DEBUG` to 1 and enable `@ASSERT`. Without this flag, `DEBUG` is 0 and `@ASSERT` is ignored. |
| `--show-warnings` | Print compiler warnings to stderr. Warnings are suppressed by default. |
| `--assembler path` | Use a compatible executable for the selected system's assembler. |
| `--include path` | Include a `.cbi` file before the source. Repeat for multiple files. They are included in the order specified from left to right. |
| `--crustybasic-toml path` | Use this standalone project/tool config instead of the `crustybasic.toml` beside the compiler. |
| `--from-tokenized=NAME` | Force a BASIC decoder for `detokenize`. |
| `--from-hex` | Read `detokenize` input as an ASCII hex dump before decoding. |
| `--output-dir dir` | Put `build-examples` program files directly in this directory instead of the configured artifact layout. |
| `--examples-dir dir` | Example tree for `build-examples`. Defaults to `examples/`. |
| `--examples-asm-dir dir` | Directory for assembly files from `build-examples`. |
| `--cart-size N` | Choose an exact size for a cartridge without bank switching (`16k`, `32k`, `0x8000`, `$8000`, or decimal bytes). |
| `--cart-include-variants` | For listings without a target selection, build every exact system variant instead of one representative per target. |

Run `crustybasic options` for the `--set` keys, or
`crustybasic target-info <system>` for details about one system,
including its usable memory regions.

## `--set` Options

These are the current command-line option keys:

| Key | Values | What it does |
| --- | --- | --- |
| `target` | Any target shown by `crustybasic options` | Select a target family. |
| `system` | Any exact system shown by `crustybasic options` | Select an exact system. |
| `dialect` | Any dialect shown by `crustybasic options` | Select source compatibility rules. |
| `rom` | Target specific ROM profile | Select which ROM services the program may use. |
| `mapper` | Target specific mapper name | Choose how cartridge storage is arranged. |
| `output-type` | Target specific output name | Select the runnable format to build. |
| `tile-backend` | `cell`, `native`, `bitmap`, `kernel` | Choose the TILE drawing backend. |
| `optimize` | `default`, `speed`, `size` | Choose a general speed or size preference. |
| `math-real` | `auto`, `target`, `builtin` | Choose where `REAL` math comes from. |
| `builtin-real` | `auto`, `q16_16`, `q24_8` | Choose the compiler supplied `REAL` format. |
| `math-integer` | `auto`, `target`, `builtin` | Choose where integer math comes from. |
| `numeric-mode` | `integer`, `real`, `real_narrow`, `integer_only_warn`, `integer_only_error` | Choose the default numeric policy. |
| `type-policy` | Any policy shown by `crustybasic options` | Choose identifier suffixes and implicit numeric types. |
| `array-base` | `0`, `1` | Choose the first array index. |
| `region` | `REGION_NTSC`, `REGION_PAL` | Choose timing data for targets that differ by region. |
| `throttle` | `0` through `65535` | Add delay to approximate interpreted BASIC timing. |
| `ram-top` | Decimal or hex with `$` or `0x` | Set the highest address available for program data. |
| `start-program` | Decimal or hex with `$` or `0x` | Set the outer loaded program wrapper start when the startup format has one. |
| `start-code` | Decimal or hex with `$` or `0x` | Set the generated machine code start address. |
| `start-data` | Decimal or hex with `$` or `0x` | Set the mutable data start address for split code/data layouts. |
| `chr-rom` | File path | Include a CHR ROM file in a cartridge. |
| `disk-image` | `on`, `off`, `true`, `false`, `1`, `0` | Create or skip a disk image when supported. |
| `asm-verbose` | `0`, `1`, `2` | Control how much detail appears in generated assembly files. |

Run `crustybasic options` for the target specific `rom`, `system`, and
`mapper` lists.

`numeric-mode=real_narrow` still gives unsuffixed numeric names a
`REAL` default, but the compiler may store proven integer only implicit
variables as integers. `numeric-mode=integer` changes the default type
and is not the same as a `REAL` dialect with narrowing.

`type-policy` controls suffixes and the implicit types assigned to names.
It normally comes from the selected target or dialect. See
[Types](LANGUAGE.md#types) before overriding it directly.

`math-real=builtin` selects CrustyBASIC's `REAL` implementation.
`builtin-real` chooses which builtin format to use. It does not affect
`math-real=target`.

| `builtin-real` | Meaning |
| --- | --- |
| `auto` | Use the target's preferred builtin format. |
| `q16_16` | Fixed point with more fractional precision. |
| `q24_8` | Fixed point with more integer range. |

Some compile options are config and source only, so they are not
accepted by `--set`:

| Key | Values | What it does |
| --- | --- | --- |
| `string-buffer-length` | `2` through `256` | Set the default and runtime string buffer length. |
| `startup` | Target specific startup name | Select a startup template supplied by the target. |
| `memory-regions` | array of target region names | Add named target memory regions to the program's usable RAM. |
| `memory-actions` | array of target action names | Run named target startup actions. |

Named regions include their required actions automatically.

## Memory Regions

Run `crustybasic target-info <system>` to inspect regions available for
normal compiler placement on that system. It shows each listed region's
ranges, code or data uses, loading method, whether it is enabled
automatically, required actions, and disabled facilities.

Enabling a region adds its address ranges to the memory the compiler may
use. The compiler decides what code or data to place there according to
the region's supported uses and loading method. It does not assign a
particular variable or PROC to the region.

Enable a region in source with:

```basic
@OPTION MEMORY_REGION name
```

Or enable it in the program's config file:

```toml
memory-regions = ["name"]
```

Repeat `@OPTION MEMORY_REGION` or add more names to the config array to
enable multiple regions. Regions shown as enabled automatically need no
option or config entry.

Some targets also expose startup actions that can be requested without
adding a memory region:

```basic
@OPTION MEMORY_ACTION name
```

```toml
memory-actions = ["name"]
```

`target-info` shows actions required by its listed regions. Other action
names come from target data and are not currently listed by this
command. Required actions are added automatically when you enable a
region.

## Line Numbers

Numbered listings are detected automatically. Mixed numbered and
unnumbered program lines aren't allowed.

```basic
10 PRINT "HELLO"
20 GOTO 10
```

## Targets And Systems

Two names matter:

- **Target** - a machine family, such as `apple2`, `c64`, or `coco`.
  Selecting a target uses that family's default system.
- **System** - an exact machine profile within a family, such as
  `apple2.plus`, `c64.orig`, or `coco.3`.

The compiler is the authoritative source for current names:

```bash
crustybasic options
crustybasic targets
crustybasic target-info c64.orig
crustybasic target-info all
```

`options` lists valid target, system, ROM, dialect, mapper, and type
policy names. `targets` summarizes every exact system's CPU, assemblers,
runnable extensions, and exposed chips. `target-info` shows one system's
default output, memory layout and regions, chips, and cartridge details.
It expects an exact system name. Use `all`, or omit the name, to show
every system.

Use `OUTPUT_TYPE` for cartridge, disk, and other output choices within a
system. Names ignore letter case. Each target has its own text
encoding, such as PETSCII or ATASCII, so stick to plain ASCII for
portable programs. When available, target specific setup, APIs, and
limitations are in the [`targets/`](targets/) docs.

## Picking A Target Or System

Use source options or CLI flags:

```basic
@OPTION TARGET c64
```

Or select a specific system:

```basic
@OPTION SYSTEM coco.ecb
```

```bash
crustybasic compile examples/__portable__/strings.cbs --set target=coco -o /tmp/strings.s
crustybasic compile examples/__portable__/strings.cbs --set system=coco.ecb -o /tmp/strings.s
crustybasic build-examples --set target=c64
```

Config profiles can set `target` or `system` too; see
[Folder Config](#folder-config). CLI `--set` always wins. A
dialect default only applies when nothing more specific selected a
target or system.

## Cartridge Mappers

A mapper describes how a cartridge presents ROM to the program. A simple
layout exposes one fixed ROM area. A banked layout divides a larger ROM
into sections and swaps selected sections into a smaller address range
when needed.

`MAPPER` applies to cartridge output. Select it in source:

```basic
@OPTION OUTPUT_TYPE cart
@OPTION MAPPER name
```

Or on the command line:

```bash
crustybasic build program.cbs --set output-type=cart --set mapper=name
```

You can omit `OUTPUT_TYPE` when cartridge is already the system's
default output. The accepted output and mapper names are target specific.

Mapper names, default layouts, bank sizes, cartridge limits, and any
extra cartridge inputs depend on the target. Run `crustybasic options`
for the accepted names, and see the individual target's cart info for specifics


## Banked Builds

A banked build divides cartridge content into numbered sections that may
not all be available at the same time. The mapper decides which areas
remain available, which areas are switched, and what code and DATA
can be used across banks.

With a banked mapper, use `@BANK PRG N ... @ENDBANK` to place enclosed
PROCs and DATA in program bank `N`. Bank numbers start at `0`.

```basic
@BANK PRG 0
DATA 1, 2, 3
@ENDBANK
```

Where supported, the related controls have the same general purpose:

| Control | What it does |
| --- | --- |
| `@OPTION OUTPUT_TYPE name` or `--set output-type=name` | Select a cartridge output. |
| `@OPTION MAPPER name` or `--set mapper=name` | Select the cartridge layout. |
| `@BANK PRG N ... @ENDBANK` | Place enclosed PROCs and DATA in numbered program bank `N`. |
| `--emit-asm path` | Keep the generated main and bank assembly files. |
| `--size-report` | Show the space used and available in the cartridge banks. |

See [`@BANK` / `@ENDBANK`](LANGUAGE.md#bank--endbank) for the source
rules. See the target pages listed under [Cartridge Mappers](#cartridge-mappers)
for the available layouts and target specific details.

## Dialects

Select a dialect with `--set dialect=...`, `@OPTION DIALECT`, or a
config profile. Run `crustybasic options` for the names supported by the
installed compiler. The language reference explains what dialects
change: [Dialects](LANGUAGE.md#dialects). Config profile discovery and
layering are covered in [Folder Config](#folder-config).

### Tokenized BASIC Files

Use `detokenize` to inspect tokenized BASIC files:

```bash
crustybasic detokenize PROGRAM.BAS --set dialect=atari_basic -o /tmp/program.bas
```

Without `-o`, the decoded text goes to stdout. If no dialect is set,
`detokenize` tries known decoders and otherwise treats the input as
plain text. Use `--from-tokenized=NAME` to force a decoder and
`--from-hex` for ASCII hex dumps.

## Folder Config

The nearest `crustybasic.config.toml` applies to sources in that folder
and its children. A file specific `<name>.config.toml` beside the source
is layered on top of the folder file for that source, overriding only
the values it sets. Config discovery starts from the entry source file.
Sibling config files beside `@INCLUDE`d files are ignored.

Compile option keys normally use the same spelling as `--set`, except
that `asm-verbose` is CLI only. Config values use TOML syntax, so string
choices are quoted and booleans are `true` or `false`. Named enum values
use their source spelling, for example:

```toml
region = "REGION_PAL"
tile-backend = "TILE_BITMAP"
```

Config also has these project keys:

| Key | What it does |
| --- | --- |
| `include` | Include one path or an array of paths before the source. Relative paths start from the config file's directory. |
| `build-examples` | Set to `false` to skip the source during `build-examples`. |
| `bit-on-char`, `bit-off-char` | Change the two characters used by visual binary literals. |

The config and source only compile options are listed above under
`--set` Options. Config can also define a custom `[type-policy.NAME]`;
see [Types](LANGUAGE.md#types).

When profiles are layered, scalar values in the more specific file
replace earlier values. Lists such as `include`, `memory-regions`, and
`memory-actions` append with the base file first.

Any config setting other than `target` or `system` can be limited to one
target or system with `[target.NAME]` or `[system.NAME]`. A matching
system section replaces scalar values from the matching target section;
list values append. The active scope comes from the CLI, the unscoped
config, or a dialect default. A target selected only inside the BASIC
source is too late to activate a scoped config section.

```toml
# crustybasic.config.toml
target = "c64"
array-base = 1
include = "my_style.cbi"

[target.vic20]
optimize = "size"
string-buffer-length = 32

[target.atari2600]
optimize = "size"

[system.vic20.orig]
build-examples = false
```

```toml
# game.config.toml for game.cbs
target = "nes"
optimize = "speed"
```

For `game.cbs`, `target` is `nes`, `optimize` is `speed`, and
`array-base` remains `1`. `my_style.cbi` is still included before the
source and can hold folder spelling preferences:

```basic
' my_style.cbi
@REWRITE ENDW = ENDWHILE
@REWRITE INKEY$ = INKEY
```


## Assemblers

`build` needs the assembler and syntax expected by the selected system.
Run `crustybasic targets` to see each system's current default assembler
and executable name.

The compiler looks for a compatible executable using the paths declared
by the target manifests. Release bundles create the matching
`tools/assemblers/<assembler>/<platform>/` directory, but do not include
assembler binaries. Place the expected executable there or pass it with
`--assembler`. Build helpers are discovered from the same manifest data.
If a required executable is unavailable, the build reports what must be
provided.

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

## Outputs And Side Artifacts

A system may support several runnable formats. `target-info` shows its
default and any cartridge output names. When a page is available under
[`targets/`](targets/), it covers that target's other formats. Select an
output by its manifest name:

```bash
crustybasic target-info c64.orig
crustybasic build program.cbs --set output-type=cart
```

An explicit `-o` extension also selects the matching output when that
choice is unambiguous. Use `output-type` when you want the choice to be
independent of the filename.

Some output recipes also create side artifacts. The build prints the
primary output and every side artifact it writes. `disk-image=true` is a
convenience for selecting a system's sole disk output or enabling its
disk side artifact:

```bash
crustybasic build program.cbs --set disk-image=true
```

If a system offers more than one disk format, select the exact one with
`output-type`. Use `disk-image=false` to suppress side artifacts that
would otherwise be created by default.

A missing required assembler or helper fails the build. A recipe step
marked optional warns and leaves the main output intact. See the
available [`targets/`](targets/) docs for format contents and runtime
requirements.

## Compile Options In Source

Common source options:

```basic
@OPTION TARGET c64
@OPTION DIALECT crustybasic
@OPTION ARRAY_BASE 1
@OPTION MATH_REAL BUILTIN
@OPTION BUILTIN_REAL Q16_16
@OPTION MATH_INTEGER BUILTIN
@OPTION NUMERIC_MODE INTEGER
@OPTION REGION REGION_NTSC
@OPTION STRING_BUFFER_LENGTH 64
@OPTION THROTTLE 10
```

`@OPTION TARGET` or `@OPTION SYSTEM` must appear before target specific
statements. `@OPTION DIALECT` applies that dialect's defaults only to
options you have not already set.

`crustybasic options` prints the authoritative list of CLI options. See
[Compiler options](LANGUAGE.md#compiler-options) for source options and
the available [`targets/`](targets/) docs for target specific options.

## Throttling

Native code is much faster than interpreted BASIC. Throttling adds a
small delay to each statement so timing dependent games feel closer to
the original interpreter.

```basic
@OPTION THROTTLE 8
```

```bash
crustybasic compile game.bas --set dialect=applesoft_basic --set throttle=8 -o /tmp/game.s
```

`0` disables throttling. Routines marked with `@ASYNC`, and every PROC
they can call, are left unthrottled. Start low and tune by feel.

## Example Layout

| Path | What's in it |
| --- | --- |
| `examples/__portable__/` | Portable crustyBASIC programs. |
| `examples/<target>/crustybasic/` | Target specific crustyBASIC programs. |
| `examples/<target>/dialects/<dialect>/` | Compatibility dialect listings. |

Compile an example using its directory [config profile](#folder-config):

```bash
crustybasic compile examples/apple2/dialects/applesoft_basic/calculator.bas -o /tmp/calculator.s
```

Or with everything explicitly specified:

```bash
crustybasic compile examples/apple2/dialects/applesoft_basic/calculator.bas --set dialect=applesoft_basic --set system=apple2.plus -o /tmp/calculator.s
```

The core language reference is in [LANGUAGE.md](LANGUAGE.md), and the
portable API reference is in [API.md](API.md).
