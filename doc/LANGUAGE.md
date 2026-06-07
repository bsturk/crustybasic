# CrustyBASIC Language Reference

The basics:

- In native CrustyBASIC mode, unsuffixed numeric names use the
  platform's unsigned default: `U8`, `U16`, or `U32`.
  Use suffixes or `AS` types when you need a specific type.
- Procedures and flow: `PROC`, `FOR`/`NEXT`, `WHILE`,
  `DO`. `GOTO` and `GOSUB` are also available.
- Runtime and portable API calls are documented in [API.md](API.md).
- Dialects help with compiling "old" BASIC listings.
  They choose compatible defaults and translate the source.

Companion pages: [API.md](API.md) covers portable/runtime APIs,
target capabilities, and hardware/memory calls; [USAGE.md](USAGE.md)
covers command-line use. See [Dialects](#dialects) for selecting
compatibility modes.

## Code

A source file can contain directives, declarations, `PROC`
definitions, and executable statements. The first executable statement
at the top level is where the program starts.  If there is a `PROC` named
MAIN with no arguments, that is called to start the program.

```basic
PROC SHOW_SCORE(SCORE#2)
    PRINT "SCORE "; SCORE#2
ENDPROC

NAME$   = "PLAYER"
LIVES#1 = 3
SCORE#2 = 0

PRINT "READY, "; NAME$

SCORE#2 = SCORE#2 + 10
SHOW_SCORE SCORE#2

PRINT "LIVES "; LIVES#1

END
```

You can declare variables by first assignment with a suffix
(`NAME$`, `LIVES#1`, `SCORE#2`) or by writing an explicit declaration
such as `DIM <VAR_NAME> AS <TYPE>`. You can also initialize when declaring.

```basic
DIM BOO$ = "BOO"
DIM STR$ AS STRING
DIM FOO AS STRING
DIM BAR AS STRING = "BAR"
TIC = "TIC"
TOC AS STRING = "TOC"
```

## Program syntax

### Comments

```basic
' comment
REM comment
```

Both comment types run to the end of the line and can appear at line start or after a
statement.

### Multiple statements

Use `:` to chain statements on one line.

```basic
X = 1 : Y = 2 : PRINT X + Y
```

### Line numbers

Complete line-numbered listings are detected automatically. Mixed
numbered and unnumbered program lines are rejected. New CrustyBASIC
code does not need line numbers, but can use them if desired.

```basic
10 PRINT "HELLO"
20 GOTO 10
```

## Types

| Type     | Size            | Range                     |
| -------- | --------------- | ------------------------- |
| `U8`     | 1 byte          | `0..255`                  |
| `U16`    | 2 bytes         | `0..65535`                |
| `U32`    | 4 bytes         | `0..4294967295`           |
| `I8`     | 1 byte          | `-128..127`               |
| `I16`    | 2 bytes         | `-32768..32767`           |
| `I32`    | 4 bytes         | `-2147483648..2147483647` |
| `REAL`   | target specific | target floating point     |
| `STRING` | `1+N` or `2+N`  | up to declared capacity   |

Native CrustyBASIC type suffixes:

| Suffix    | Type                                               |
| --------- | -------------------------------------------------- |
| `NAME`    | platform unsigned default (`U8`, `U16`, or `U32`)  |
| `NAME#1`  | `U8`                                               |
| `NAME#2`  | `U16`                                              |
| `NAME#4`  | `U32`                                              |
| `NAME%1`  | `I8`                                               |
| `NAME%2`  | `I16`                                              |
| `NAME%4`  | `I32`                                              |
| `NAME!`   | `REAL`                                             |
| `NAME$`   | `STRING`                                           |

An explicit `AS` sets the type. You can combine it with a suffix as
long as both say the same thing, such as `DIM TOTAL#4 AS U32`.
A mismatch is an error.

If you prefer a different suffix style, define it in
`crustybasic.config.toml` and select it with `type-policy`:

```toml
type-policy = "my_suffixes"

[type-policy.my_suffixes]
numeric-mode = "integer"
default-numeric = "U8"
address-type = "U16"
string-index-type = "U8"
array-index-type = "U16"
suffixes = [
  { suffix = "#", type = "U8" },
  { suffix = "##", type = "U16" },
  { suffix = "###", type = "U32" },
  { suffix = "%", type = "I16" },
  { suffix = "!", type = "REAL" },
  { suffix = "$", type = "STRING" },
]
```

With that config, `BYTE#` is `U8`, `WORD##` is `U16`,
`LONG###` is `U32`, `SIGNED%` is `I16`, `RATE!` is `REAL`, and
`NAME$` is `STRING`.

If you define more than one style in the config file, source can pick
one explicitly:

```basic
@OPTION TYPE_POLICY my_suffixes
```

Suffixes must start with punctuation and can use punctuation or digits
after that. The other lines in the example set the default numeric type
and the integer types used for addresses and indexes. Copy those values
unless you want to change those defaults too.

Compatibility dialects use the suffixes from the BASIC they are
matching. See [Dialects](#dialects) for the short version.

In strict-variable mode, the first assignment creates a scalar variable.
Inside a `PROC`, that variable belongs to the `PROC`. At the top level,
it is global. Reading a variable before it exists is an error.

Integer literals:

- Assigned into an integer variable: the literal takes that variable's
  type if it fits. Arithmetic widens to the destination when needed
  (`M#2 = K#1 * 8` with `K#1` as `U8` multiplies at `U16`).
- Without a destination: positive literals use the smallest unsigned
  type that fits. Negative literals need a signed destination.
  Assigning a negative value to an unsigned destination is an error.

Non-literal signed and unsigned values can mix when there is a
lossless common integer type.

### Type conversion

Numbers are converted when they cross a typed place: assignment, `READ`,
`RETURN`, value parameters, array or string indexes, addresses, and byte
values.

- Widening is silent: `U8` to `U16`, or any integer to `REAL`.
- A literal that fits the destination is accepted without a warning.
- A conversion that might lose information still compiles with a
  warning, then uses the destination type.
- An integer literal that cannot fit an assignment target is an error.
- `REAL` converts to 8- or 16-bit integer destinations, with a warning
  unless the expression is already explicitly integer, such as `INT(X)`.
  `REAL` does not convert directly to `U32` or `I32`.
- Strings do not convert to numbers.

Signed integers are two's complement; division truncates toward zero,
`MOD` keeps the sign of the left operand.

`REAL` depends on the target - see [per-target support](API.md#screen-real-and-text-encoding).

## Declaration examples

```basic
DIM X AS U16
DIM B AS U8
DIM VALUE AS REAL
DIM NAME AS STRING
DIM TITLE AS STRING * 80

DIM X, Y, VX, VY AS U8
DIM LEFT_EDGE, RIGHT_EDGE AS U16
DIM FIRST_NAME, LAST_NAME AS STRING

DIM SCORES(10) AS U16        ' 11 elements with base 0, 0...10
DIM GRID[7, 7] AS U8
DIM MAP(15, 11), SHADOW(15, 11) AS U8
DIM NUMS(4) AS U8 = 1, 2, 3, 4, 5

SCORE#2 AS U16               ' DIM-less scalar declaration
TITLE$ AS STRING * 80
LEFT_EDGE, RIGHT_EDGE AS U16
SCORES(10) AS U16            ' DIM-less array declaration
GRID[7, 7] AS U8
SCORES#2[10]                 ' U16 from suffix
NAMES[3] AS STRING
```

Array bounds are inclusive. `A(10)` has indexes `0..10` (or `1..10`
with `ARRAY_BASE 1`). Either `()` or `[]` works for references.

If every array bound is a compile time constant, the array uses static
storage. If an executable declaration uses a runtime expression for the
upper bound, the array is dynamic and the declaration allocates it from
the target's mutable RAM when the program runs:

```basic
PROC MAIN
N AS U16
INPUT N
A(N) AS U8
A(0) = 7
ENDPROC
```

Dynamic arrays currently support one numeric dimension, no initializer,
and no `STRING` elements. Repeating the executable declaration leaves
the existing allocation in place. The allocated bytes are zeroed.
Allocation failure ends the program.

Parenthesized DIM-less array declarations need `AS`. Without it,
`A(10)` is parsed as a call or array reference.

Declarations inside a `PROC` belong to that `PROC`, and the value is
kept between calls. A local name cannot hide a global, `CONST`,
`PROC`, or parameter. Two different PROCs may use the same local name;
those variables are separate.

### Strings

Each string variable has a fixed maximum size. `S$ AS STRING` gets the
default capacity of 255 bytes. To pick a different size, use
`STRING * N` where `N` is the byte capacity:

```basic
S$ AS STRING
NAME$ AS STRING * 16
LINE$ AS STRING * 80
HEX$ AS STRING * (SIZEOF(U16) * 2)

S$ = "HELLO"
LINE$ = S$ + " WORLD"
S$ =+ "!"

MID(S$, 1, 5) = "HOWDY"
```

The capacity expression is compile-time integer arithmetic. `SIZEOF(T)`
is available in numeric expressions for fixed scalar types (`U8`, `U16`,
`U32`, `I8`, `I16`, `I32`), so templated code can use expressions
like `SIZEOF(@T) * 8`.

Putting `MID` on the left of `=` overwrites that slice in place -
the string's overall length does not change, just the bytes you target.

After [`BIND`](API.md#bind) points a string at your own memory, reads
and writes on that string use that memory instead of its built-in
buffer.

### CONST

```basic
CONST SCREEN_W = 40
CONST SCREEN_H = 25
CONST PI       = 3.14159
CONST SHAPE(7) AS U8 = $3C, $42, $A5, $81, $A5, $99, $42, $3C
CONST FOO(2) AS U8 = %
    ..XX.X..
    ..X..XX.
    ..X..X..
END%
```

Scalar `CONST` names are compile-time values.

Typed `CONST` arrays are read-only initialized storage. Numeric `CONST`
arrays can be read like normal arrays and are addressable:

```basic
PTR#2 = ADDR(SHAPE)
PRINT HEX(SHAPE(2))
```

### PROC

```basic
[@ASYNC]
PROC name [([IN|OUT|INOUT] param [AS type][, ...])] [AS type]
    [RETURN [expr]]
ENDPROC
```

Empty parens are optional. A real parameter list still needs parens.
Paren and parenless calls both work when not ambiguous:

```basic
KEY()
POSITION(0, 0)
POSITION 0, 0
DELAY 60
```

If a call opens with `(`, it has to close with `)`.

Parameters can use `AS` types or strict suffixes. Prefix a parameter
with `OUT` or `INOUT` when the PROC should write the final value back
to the caller's variable. With a return type, call the PROC in an
expression; without a return type, call it as a statement.

```basic
PROC DOUBLE(X AS U8) AS U8
    RETURN X + X
ENDPROC

PRINT DOUBLE(4)
-or-
PRINT DOUBLE 4
```

A few things:

- Parameters are passed by value.
- Value parameters accept numeric boundary conversions. Lossless
  widening is silent; narrowing or signedness-changing conversions that
  can lose information compile with a warning.
- Assigning to a normal parameter does not affect the caller.
- `OUT` parameters require an assignable variable argument with the
  exact same type. The value is copied out when the PROC returns.
- `INOUT` parameters also require an assignable variable argument with
  the exact same type. The caller's value is copied in before the call
  and copied out when the PROC returns.
- PROCs are not reentrant. Do not use recursion.
- A typed PROC can be used anywhere an expression of its return type is valid.
- `@ASYNC` marks a PROC as safe to call from outside the normal main
  program flow, such as from an VBI, etc.

### PROC overloads

Two PROCs can share a name when their parameter types differ.

```basic
PROC CLAMP(N AS U8, HI AS U8) AS U8
    RETURN MIN N, HI
ENDPROC

PROC CLAMP(N AS U16, HI AS U16) AS U16
    RETURN MIN(N, HI)
ENDPROC
```

CrustyBASIC picks the best overload by argument types. An exact match
beats lossless widening, and lossless widening beats a warning-producing
lossy conversion. No match, or an ambiguous match, is a compile error.

Generate typed overloads from a template with `@TYPE_TEMPLATE`:

```basic
@TYPE_TEMPLATE T IN U8, U16, U32
PROC MIN(A AS @T, B AS @T) AS @T
    IF A < B THEN
        RETURN A
    ENDIF
    RETURN B
ENDPROC
@END_TYPE_TEMPLATE
```

### SWAP

```basic
SWAP A, B
```

Swaps two scalar variables. Both sides must be the same type.
Strings, arrays, `MID` slices, qualified registers, and constants aren't allowed.

## Expressions

### Literals

```basic
255        ' decimal
$FF        ' hex
&HFF       ' hex
0xFF       ' hex
FFh        ' hex, suffix form
%1010      ' binary
%..XX.X..  ' visual binary; . is 0, X is 1
0b1010     ' binary
0101b      ' binary, suffix
0o7332     ' octal
7332o      ' octal, suffix
3.14       ' REAL, needs REAL context
"hello"    ' string
```

Radix forms are integer-only. The lowercase `h` suffix avoids
clashing with identifiers - `CH` stays an identifier; `FFh` is hex.

Visual binary is handy for sprites, tiles, and character glyphs:

```basic
CONST SHIP(7) AS U8 = %
    ...XX...
    ..XXXX..
    .XXXXXX.
    XXXXXXXX
    .XXXXXX.
    ..XXXX..
    ...XX...
    ........
END%
```

The default visual bit characters are `X` for 1 and `.` for 0.
Change them in `crustybasic.config.toml`:

```toml
bit-on-char = "#"
bit-off-char = "-"
```

### Random numbers

`RND()` returns the target's raw random byte (`U8`). It is cheap and
handy for masks and small choices:

```basic
COLOR = RND() & 7
```

`RAND()` returns a `REAL` fraction from `0` up to, but not including,
`1` on targets with `REAL`.

`RAND(max)` scales the result. With a positive `U8`, `U16`, or `REAL`
maximum, it returns a value from `0` up to, but not including, `max`.
Integer `RAND(0)` returns `0`.

```basic
X#2 = RAND(320)
JITTER#1 = RAND(5)
R! = RAND()
```

Legacy dialects may also provide their original `RND(x)` behavior. Use
`RAND(max)` when you want the portable bounded form.

### Operators

| Operators                  | Meaning                                                    |
| -------------------------- | ---------------------------------------------------------- |
| `+` `-` `*` `/`            | Arithmetic                                                 |
| `\`                        | Integer divide. Rejects `REAL` - use `INT(A / B)` instead. |
| `MOD`                      | Integer remainder                                          |
| `POW(a, b)`                | Exponentiation                                             |
| `&`, `|`, `^`              | Bitwise AND, OR, XOR                                       |
| `<<` `>>`                  | Logical shift left / right. RHS must be a constant integer literal. |
| prefix `&`                 | Address of (same as [`ADDR(...)`](API.md#addr))             |
| `=` `<>` `<` `<=` `>` `>=` | Comparison                                                 |
| `NOT` `AND` `OR` `XOR`     | Logical, returns `0` or `1`                                |
| `!` `&&` `\|\|`            | Symbol aliases for `NOT`, `AND`, `OR` (same semantics)     |
| `+` on strings             | Concatenation                                              |

`&` is overloaded by position. At statement start, `& BYTE, COUNT` is
legacy repeat-print. In expression position, prefix `&FOO` is
address-of and infix `A & B` is bitwise AND.

The shift operators currently require a constant integer on
the right-hand side; variable shift counts will be a compile
error.

### Precedence

| Level | Operators                  | Notes                |
| ----- | -------------------------- | -------------------- |
| 1     | unary `-`, prefix `&`      | highest              |
| 2     | `*` `/` `\` `MOD`          | left-associative     |
| 3     | `+` `-`                    | left-associative     |
| 4     | `<<` `>>`                  | left-associative     |
| 5     | `&`                        | bitwise AND          |
| 6     | `^`                        | bitwise XOR          |
| 7     | `|`                        | bitwise OR           |
| 8     | `=` `<>` `<` `<=` `>` `>=` | left-associative     |
| 9     | `NOT` (or `!`)             | applies to compares  |
| 10    | `AND` (or `&&`)            | left-associative     |
| 11    | `OR` (or `\|\|`)           | left-associative     |
| 12    | `XOR`                      | lowest               |

`NOT X > 5` means `NOT (X > 5)`. `A + B << 2` means `(A + B) << 2`.

## Control flow

### IF

```basic
IF X > 10 THEN
    PRINT "big"
ELIF X > 5 THEN
    PRINT "medium"
ELSE
    PRINT "small"
ENDIF

IF X > 0 THEN PRINT "yes"
IF X > 0 THEN PRINT "yes" ELSE PRINT "no"
```

### SELECT

```basic
SELECT CASE SCORE
    CASE 0
        PRINT "ZERO"
    CASE 1, 2, 3
        PRINT "LOW"
    CASE ELSE
        PRINT "HIGH"
ENDSELECT
```

### FOR

```basic
FOR I = 1 TO 10
    PRINT I
NEXT I

FOR I = 10 TO 1 STEP -1
    PRINT I
NEXT I
```

`STEP` defaults to 1. The body runs until the iterator crosses the
end value in the step direction. `STEP 0` never advances the
iterator: spins forever when `START <= END`, skips otherwise.

### WHILE

```basic
WHILE X > 0
    X = X - 1
ENDWHILE
```

### REPEAT

```basic
REPEAT
    X = X + 1
UNTIL X >= 10
```

Body runs at least once.

### DO

```basic
DO
    X = X + 1
LOOP UNTIL X >= 10

DO WHILE X < 10
    X = X + 1
LOOP

DO
    IF DONE THEN EXIT DO
LOOP
```

Pre- and post-conditions are both optional and can use `WHILE` or `UNTIL`.

### EXIT and CONTINUE

```basic
EXIT
EXIT FOR
EXIT WHILE
EXIT DO

CONTINUE
CONTINUE FOR
CONTINUE WHILE
CONTINUE DO
```

Without a suffix, `EXIT` / `CONTINUE` act on whichever loop you're
currently inside. With a suffix (`EXIT FOR`, `CONTINUE WHILE`, ...)
they walk outward past any loops of other kinds and act on the
nearest one of the named kind. No matching loop is a compile error.

### Labels, GOTO, GOSUB

```basic
TOP:
    GOTO TOP

MAIN:
    GOSUB SETUP
    GOTO MAIN

SETUP:
    PRINT "init"
    RETURN
```

Always pair `GOSUB` with `RETURN`, and keep nesting shallow - too
many nested calls (or a missing `RETURN`) can crash the program.

In line-number mode, `GOTO <EXPR>` and `GOSUB <EXPR>` also work; the
expression is matched against known line numbers while the program
runs.

### ON GOTO / ON GOSUB

```basic
ON MODE GOTO TEXT_MODE, GRAPH_MODE, SPRITE_MODE
ON CHOICE GOSUB OPT1, OPT2, OPT3
```

A value of `1` jumps to the first label in the list, `2` to the
second, and so on. If the value doesn't line up with any entry,
the statement is skipped. In line-number mode, the list can be
line numbers instead of labels.

### END and RUN

`END` ends the program. `RUN` (or `RUN()` in core syntax) is a
built-in call that restarts the program from the top - handy for
"play again?" loops in ported listings.  An `END` is not required
at the bottom of a listing.

### ON_ERROR

```basic
ON_ERROR A_LABEL
ON_ERROR OFF
ON_ERROR 300
ON_ERROR HANDLER_PROC
ON_ERROR X
ON_ERROR X + 5
```

Sets a one-shot handler for the next platform I/O error. `OFF`
disables the handler. After an error is caught, the handler must call
`ON_ERROR` again if it wants to catch another error.

Valid targets: a label in the same `PROC`, a line label (line-number
mode), a no-argument untyped `PROC`, or a numeric expression matched
against line labels (no match disarms).

Inside a handler, `ERR()` returns the captured `U8` error code until
the next error occurs.

Only platform ROM/OS calls that report an error status can trigger
`ON_ERROR`. Per-target support varies; see the target docs under
[`targets/`](targets/) for the current support list.

## I/O and DATA

### PRINT

```basic
PRINT "hello"
PRINT X; " "; Y
PRINT A, B, C
? "shorthand"
PRINT TAB(10); "col"
PRINT SPC(5); "gap"
PRINT
```

`PRINT` writes to text output. Semicolons place items next to each
other without spacing. Commas print one space. A trailing newline is added
unless the line ends in `;` or `,`.

### INPUT

```basic
INPUT X
INPUT "Enter name: "; NAME$
```

Reads one echoed line from the text input device. Blocks until the Enter key
is pressed.

### GET

```basic
GET CH
GET CH$
```

Blocks until a key is available. Numeric variables get the encoded
character code; string variables get a one-character string. No echo.

### DATA, READ, RESTORE

```basic
ITEMS:
DATA 10, 20, 30, "hello", 42

READ A, B, C
READ S$
RESTORE
RESTORE ITEMS
```

`DATA` declares a list of values. `READ` pulls them out one at a
time, converting each to the destination variable's type. `RESTORE`
rewinds to the first value; `RESTORE LABEL` jumps to a labelled
`DATA` block. In line-number mode, `RESTORE <EXPR>` jumps to the
`DATA` at the matching line number. Reading past the end is
undefined.

## REAL floating point

`REAL` is optional and target dependent. Declaring `REAL` on a target
without FP is a compile error. See [API.md](API.md#real) for helper
functions and [per-target support](API.md#screen-real-and-text-encoding).

## Inline assembly

### ASM

```basic
ASM
    lda #$01
    sta $d020
ENDASM
```

CrustyBASIC copies the assembly into the generated output after
replacing `{...}` variable names and applying any CPU filter.

Filter by CPU (useful for more portable programs):

```basic
ASM CPU 6502
    lda #$01
ENDASM

ASM CPU 6809
    ldb #$01
ENDASM
```

Current CPU values are `6502` and `6809`.
A block for a different CPU is skipped.

Inside `ASM`, `{NAME}` expands to the assembler symbol for a
variable - globals, PROC parameters, and PROC-local variables all
work:

```basic
DIM SCORE AS U8
DIM BUF(16) AS U8

ASM
    lda {SCORE}
    sta {BUF}+1
    lda {BUF},y
ENDASM
```

`CONST` names are (currently) not valid in `{...}` assembly interpolation.

Inside a typed `PROC`, `{RETURN}` expands to that PROC's return-value
slot. If an unconditional tail `ASM` block references `{RETURN}`, it
satisfies the typed-PROC return check and falls through to the
compiler-generated PROC return.

## Directives

Directives start with `@`. They configure the compile, include files,
or choose source for the active target before the program is
compiled.

### @OPTION

See [Compiler options](#compiler-options).

### @BANK / @ENDBANK

`@BANK N` starts a switched-bank block; `@ENDBANK` closes it. Every
`PROC` and top-level `DATA` statement inside the block is placed in
switched bank `N`. Blank lines, comments, and `DATA` labels (`FOO:`)
are allowed inside the block.

```basic
@OPTION TARGET nes
@OPTION MAPPER uxrom

@BANK 1
DATA 10, 20, 30
PROC BANK_ONE_READ
    X AS U8
    READ X
ENDPROC
@ENDBANK

PROC MAIN
    BANK_ONE_READ
ENDPROC
```

- Use `@BANK` only with a banked mapper: e.g. `uxrom`, `xegs32`,
  `supergames`, `banked_16k`.
- Bank numbers start at `0`. The always-available main area is not
  counted as a bank.
- `PROC MAIN` stays in the main area. Put shared helper PROCs there too.
- Code in the main area can call any bank. Code inside a bank can call
  the main area, but not another bank.
- Inside `@BANK N ... @ENDBANK`, the top level can contain only
  `PROC`, `DATA`, and `DATA` labels. Put `DIM`, `CONST`, inline `ASM`,
  and ordinary statements outside the bank block.
- Do not put `@BANK` blocks inside other `@BANK` blocks. Every
  `@ENDBANK` must match an earlier `@BANK`.
- A banked PROC can `READ` and `RESTORE` that bank's own `DATA`. If
  code in the main area needs banked `DATA`, put the read in a PROC in
  that bank and call it.
- `RESTORE LABEL` can jump only to `DATA` in the same bank. Code in the
  main area can restore only main-area `DATA`.
- Each call to a banked PROC starts `READ` at that bank's first `DATA`
  item. Use `RESTORE LABEL` when you need a specific item.
- String and `REAL` literals live with the bank that uses them.

### @DEFINE and @UNDEF

```basic
@DEFINE DEBUG
@UNDEF DEBUG
```

Set or clear a symbol that `@IF DEFINED(...)` can test while the
program is being compiled.

### @IF, @ELIF, @ELSE, @ENDIF

```basic
@IF TARGET = "c64" THEN
    SID.MODEVOL = 15
@ELIF TARGET = "atari800" THEN
    POKEY.AUDCTL = 0
@ELSE
    @ERROR "target does not support this module"
@ENDIF

@IF SYSTEM = "apple2.plus" THEN
    @INCLUDE "targets/apple2/applesoft_rom.cbi"
@ENDIF

@IF DEFINED("DEBUG") THEN
    PRINT "debug"
@ENDIF

@IF JOY_PORTS >= 2 THEN
    PRINT "two controller ports"
@ENDIF

@IF CB_VERSION_MAJOR >= 1 THEN
    @INCLUDE "modern_only.cbi"
@ENDIF
```

Things `@IF` can check:

| Form                                                | Meaning                                                                                                |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `TARGET = "name"`                                   | Active target name (must be a target: `apple2`, `atari800`, `atari5200`, `atari2600`, `c64`, `coco`, `nes`, `plus4`, `vic20`). |
| `SYSTEM = "name"`                                   | Active system name (e.g. `apple2.plus`, `c64.cart`). Using a target name here is a compile error.      |
| `CPU = "6502"`                                      | Current CPU string.                                                                                    |
| `TARGET_C64`, `TARGET_APPLE2`, ...                  | True when the active target matches.                                                                   |
| `SYSTEM_APPLE2_PLUS`, `SYSTEM_C64_CART`, ...        | True when the active system matches.                                                                   |
| `CPU_6502`, `CPU_6809`                              | CPU flag.                                                                                              |
| `CB_VERSION = "1.1.0"`                              | Compiler version string (matches `crustybasic --version`).                                             |
| `CB_VERSION_MAJOR`, `CB_VERSION_MINOR`, `CB_VERSION_PATCH` | Integer parts of the compiler version; use with `>=`, `<`, etc. for range checks.               |
| `DEFINED("symbol")`                                 | Symbol set with `@DEFINE`.                                                                             |
| `HAS_CHIP("name")`                                  | Target exposes the chip.                                                                               |
| `JOY_PORTS`, `GRAPHICS_AVAILABLE`, `SPRITE_KIND`, etc. | Target constants such as controller counts and supported APIs. |

Conditions support `AND`, `OR`, `NOT`, parentheses, booleans,
strings, decimal integers, `$` hex integers, and the usual
comparisons.

Common coarse checks are `TEXT_OUTPUT_AVAILABLE`, `TEXT_CHARMAP_AVAILABLE`,
`GRAPHICS_AVAILABLE`, `FRAME_AVAILABLE`, and
`SPRITE_AVAILABLE`. `CODE_STORAGE` is `"ram"` for
writable program images and `"rom"` for cartridge/ROM images. `TARGET`
is the active target name
(e.g. `"c64"`, `"apple2"`), which is useful when the system name
varies but you want to branch on the target.

### @WARN

```basic
@WARN "POKE 53280 is deprecated; use VIC.BORDER"
```

Prints a non fatal compile-time warning with your message and continues.

### @ERROR

```basic
@ERROR "unsupported target for this module"
```

Fails the compile with your message.

### @INCLUDE

```basic
@INCLUDE "common/utils.cbi"
@INCLUDE "sprite_data/{system}.cbi"
@INCLUDE "{program_name}/sprites.cbi"
```

Includes a file at that point in the source. Paths are tried relative
to the including file first, then under `include/` directory relative
to the current working directory.

Include paths can use placeholders. Placeholder names are not
case-sensitive.

| Placeholder | Expands to |
| ----------- | ---------- |
| `{target}` | Target name, such as `apple2` or `c64`. |
| `{system}` | Full system name, such as `apple2.plus` or `c64.cart`. |
| `{program_name}` | Program filename without extension, such as `breakout` for `breakout.cbs`. |
| `{media}` | Media variant, such as `cart`, when the selected system has one. |
| `{cpu}` | CPU name, such as `6502` or `6809`. |
| `{mapper}` | Selected mapper, such as `nrom` or `uxrom`, when one is active. |
| `{version}` | CrustyBASIC version string, such as `1.1.0`; always available. |

## Compiler options

Set with `@OPTION` or the CLI.

```basic
@OPTION TARGET c64
@OPTION DIALECT atari_basic
@OPTION MAPPER uxrom
@OPTION ARRAY_BASE 0
@OPTION NUMERIC_MODE INTEGER
@OPTION MATH_REAL AUTO
@OPTION PROGRAM_START $2001
@OPTION CODE_START $2100
@OPTION REGION_NTSC
@OPTION THROTTLE 10
```

| Option                           | Values                                  | What it does                                                                  |
| -------------------------------- | --------------------------------------- | ----------------------------------------------------------------------------- |
| `TARGET`                         | target name                             | Select the output target (one of `apple2`, `atari800`, `atari5200`, `atari2600`, `c64`, `coco`, `nes`, `plus4`, `vic20`). Must come before target dependent source. Passing a system name here is a compile error. |
| `SYSTEM`                         | system name                             | Select a specific system (e.g. `apple2.plus`, `c64.cart`). Implies its parent target. Passing a target name here is a compile error. |
| `DIALECT`                        | dialect name                            | Apply that dialect's defaults to options you haven't set.                     |
| `THROTTLE`                       | `0..65535`                              | MOSLO-style delay. `0` disables throttling.                                   |
| `BITMAP_BANK`                    | `0..3`                                  | Target defined bitmap/screen memory bank.                                     |
| `MAPPER`                         | mapper name                             | Pick a cartridge mapper.                                                      |
| `NUMERIC_MODE`                   | see below                               | Default numeric policy.                                                       |
| `MATH_REAL`                      | `AUTO`, `TARGET`, `BUILTIN`             | Select the REAL math implementation.                                          |
| `MATH_INTEGER`                   | `AUTO`, `TARGET`, `BUILTIN`             | Select the integer math implementation.                                       |
| `PROGRAM_START`                  | address                                 | Set the outer loaded program wrapper start when the startup format has one.   |
| `CODE_START`                     | address                                 | Set the generated machine code start address.                                 |
| `REGION_NTSC`, `REGION_PAL`      | flag                                    | Pick timing/frequency tables. Default is NTSC.                                |
| `INLINE_ASM`                     | `ON`, `OFF`                             | Allow `ASM ... ENDASM`.                                                       |
| `ARRAY_BASE`                     | `0`, `1`                                | First array index.                                                            |
| `FOLD_LOWERCASE_TO_UPPERCASE`    | `ON`, `OFF`                             | Uppercase lowercase ASCII before target encoding.                             |

Some targets define additional source-only `@OPTION`s for their own
hardware profiles. See the individual target docs for those.

`NUMERIC_MODE` is normally chosen by the selected dialect. Override it
only when you want to change the dialect or native numeric policy:

- `INTEGER` - unsuffixed numeric names are integer.
- `REAL` - unsuffixed numeric names are `REAL`.
- `REAL_NARROW` - REAL-default dialect mode; see [Dialects](#dialects).
- `INTEGER_ONLY_WARN` - integer names, warn on REAL constructs.
- `INTEGER_ONLY_ERROR` - integer names, error on REAL constructs.

`MATH_REAL` and `MATH_INTEGER` choose which math code to use after the
types are known. They do not make a value `REAL` or integer:
`NUMERIC_MODE`, suffixes, and `AS` declarations do that.

| Value     | Meaning |
| --------- | ------- |
| `AUTO`    | Let the target choose. This is the default when the option is not set. |
| `TARGET`  | Use the machine's ROM/runtime math. Error if unavailable. |
| `BUILTIN` | Use CrustyBASIC's own math. Error if unavailable. |

`TARGET` and `BUILTIN` are fixed choices. `--set optimize=speed` and
`--set optimize=size` do not change them.

`AUTO` can change with `--set optimize=speed` or
`--set optimize=size`, but only on targets that define a faster or
smaller choice. Otherwise it uses the target's normal default.

Integer math uses `MATH_INTEGER`. `REAL` math uses `MATH_REAL`. A mixed
program can use both.

CLI `--set throttle=N` overrides `@OPTION THROTTLE N`.

## Dialects

Dialects are compatibility modes for existing BASIC listings. Pick one
with source `@OPTION DIALECT`, CLI `--set dialect=...`, or
`crustybasic.config.toml`. Dialect defaults only fill in options you
have not already set, so explicit target, system, ROM, and option
settings still win.

Dialects follow the naming habits of the source BASIC. For example,
Applesoft, Atari BASIC, Commodore BASIC, and Color BASIC treat bare
numeric names as `REAL`, `%` names as 16-bit integers, and `$` names as
strings. Apple Integer BASIC treats bare numeric names as 16-bit
integers and `$` names as strings.

REAL-default dialects use `REAL_NARROW` automatically. Bare numeric
names start as `REAL` for compatibility. If a dialect-default variable
or array is only used with whole numbers, CrustyBASIC may store it as
`U8`, `U16`, or `U32` for speed and size. Printed values, fractional
values, explicit `REAL` declarations, and REAL math stay `REAL`.

```basic
@OPTION DIALECT cbm_basic_v2
```

```bash
crustybasic compile program.bas --set dialect=cbm_basic_v2 -o /tmp/program.s
```

```toml
dialect = "cbm_basic_v2"
```

Available dialects:

| Dialect | Use for |
| --- | --- |
| `crustybasic` | Native CrustyBASIC source. (the default so not needed) |
| `applesoft_basic` | Applesoft BASIC listings. |
| `integer_basic` | Apple Integer BASIC listings. |
| `atari_basic` | Atari BASIC listings. |
| `basic_xl` | BASIC XL listings. |
| `cbm_basic_v2` | Commodore BASIC V2 listings. |
| `cbm_basic_vic20` | Commodore BASIC V2 listings for VIC-20. |
| `cbm_basic_v3_5` | Commodore BASIC 3.5 listings for Plus/4. |
| `cbm_basic_v7_0` | Commodore BASIC 7.0 listings. |
| `color_basic` | TRS-80 Color BASIC listings. |
| `extended_color_basic` | Extended Color BASIC listings. |

## Source Rewrites

`@REWRITE` lets you map one spelling to another before the program is
checked.

`@REWRITE_EXPR OLD = NEW` rewrites expression fragments inside statements, such as translating `{l:atom}^{r:atom}` to `POW({l}, {r})`.

Add your own preferred syntax by putting `@REWRITE OLD = NEW`
lines in a `.cbi` file and listing that file in `crustybasic.config.toml`
with `include` to pick it up automatically.

## Mappers

Default mappers:

| Target          | Default        | Banked option      |
| --------------- | ----------     | -------------      |
| `nes`           | `nrom`         | `uxrom`            |
| `atari800` cart | `standard`     | `xegs32`           |
| `atari2600`     | (default 4 KB) | `f8` / `f6` / `f4` |
| `c64` cart      | `standard`     | `supergames`       |
| `coco` cart     | `standard`     | `banked_16k`       |
