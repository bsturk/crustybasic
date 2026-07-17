# CrustyBASIC Language Reference

The basics:

- In native CrustyBASIC mode, unsuffixed numeric names use the
  platform's unsigned default: `U8`, `U16`, or `U32`.
  Use suffixes or `AS` types when you need a specific type.
- Procedures and flow: `PROC`, `FOR`/`NEXT`, `WHILE`,
  `DO`. `GOTO` and `GOSUB` are also available.
- Runtime and portable API calls are documented in [API.md](API.md).
  Core builtins are documented below.
- Dialects help with compiling "old" BASIC listings.
  Compatible defaults are chosen and the source is translated to crustyBASIC.
  Dialects can also be used for new development, if desired.

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
code does not need line numbers, but can use them if desired. Use
`@OPTION USE_LINE_NUMBERS TRUE` to force line-number mode.

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

### Typed casts

`U8(expr)`, `U16(expr)`, `U32(expr)`, `I8(expr)`, `I16(expr)`, and
`I32(expr)` convert a numeric expression to exactly that type. A cast
never warns: it is the explicit "I know this fits" (or "I want the
wrap") form of the conversions above.

- The value is truncated to the target width. Out-of-range values wrap
  modulo that width instead of being a literal-range error:
  `U8(300)` is `44`, `U8(-1)` is `255`, `I8(255)` is `-1`.
- Constant operands fold at compile time, and casts are valid in
  `CONST` initializers: `CONST LOW = U8($1234)` is `$34`.
- Casting a `STRING` is an error. `REAL` operands follow the same
  reach as implicit conversion (`REAL` does not cast to `U32` or
  `I32`), just without the warning.

```basic
HI#1 = U8(PTR#2 >> 8)        ' high byte of an address
LO#1 = U8(PTR#2)             ' low byte, wraps modulo 256
IDX  = U8(IDX + 1)           ' deliberate mod-256 counter
PLOT U8(BASE_X + DX + 1), U8(BASE_Y + DY + 1)
```

Use a cast where the in-range guarantee is yours rather than the
compiler's - it documents the claim at the exact spot the narrowing
happens and silences the "possible integer overflow" warning for that
expression only.

Signed integers are two's complement; division truncates toward zero,
`MOD` keeps the sign of the left operand.

`REAL` depends on the target - see the target docs under [`targets/`](targets/).

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
DIM VALUE AS U8 = 0
DIM NUMS[4] AS U8 = 1, 2, 3, 4, 5
PRINT LEN(NUMS)              ' element count

SCORE#2 AS U16               ' DIM-less scalar declaration
TITLE$ AS STRING * 80
LEFT_EDGE, RIGHT_EDGE AS U16
SCORES(10) AS U16            ' DIM-less array declaration
GRID[7, 7] AS U8
FLAGS(3)
BUF[3] AS U8 = 1, 2, 3, 4
SCORES#2[10]                 ' U16 from suffix
NAMES[3] AS STRING
```

Array bounds are inclusive. `A(10)` has indexes `0..10` (or `1..10`
with `ARRAY_BASE 1`). Either `()` or `[]` works for array declarations,
initializer lists, and references. `LEN(A)` returns the element count.
`ADDR(A(I))` returns the address of one element; for string arrays this
is the element's string descriptor.

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

For `A(10)` statements, a matching `PROC A` is treated as a call. Without
a matching routine, it is a DIM-less array declaration. A name cannot be
both callable and storage.

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
CONST SHAPE[7] AS U8 = $3C, $42, $A5, $81, $A5, $99, $42, $3C
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
[@WARNING "message"]
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
- Use `AS type ADDR` for a storage address. Pass `ADDR ARRAY`,
  `ADDR ARRAY(I)`, or a raw numeric address. When passing addressable
  storage, the storage element type must match the parameter element type.
- Inside the PROC, a `type ADDR` parameter can be indexed with `[]` or
  `()` to read and write memory at the passed address. The element type
  controls the stride. Indexed `ADDR` parameters currently support `U8`,
  `I8`, `U16`, and `I16` elements. The parameter is also a normal
  address value, so memory operations like `PEEK` and `POKE` can use it
  directly.
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
- `@WARNING "message"` emits the message when user code calls that
  PROC overload. Calls introduced only by library code stay quiet.

```basic
BUF[4] AS U8
WORDS[2] AS U16

PROC TOUCH_BYTES(P AS U8 ADDR)
    P[0] = 1
    P(1) = 2
ENDPROC

PROC TOUCH_WORDS(P AS U16 ADDR, I AS U8)
    P[I] = 4660
ENDPROC

TOUCH_BYTES ADDR BUF
TOUCH_WORDS ADDR WORDS, 1
```

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

### Core builtins

These functions and operators are part of the language.

#### Random numbers

| Function          | Returns            | Description                                      |
| ----------------- | ------------------ | ------------------------------------------------ |
| `RAND()`          | `REAL`             | Fraction from `0` up to, but not including, `1`. |
| `RAND(max)`       | same type as `max` | Scaled random value. Integer `RAND(0)` returns `0`. |
| `RAND_SEED(seed)` | none               | Seed `RAND`. Seed `0` is treated as `1`.         |

`RAND()` returns a `REAL` fraction from `0` up to, but not including,
`1` on targets with `REAL`.

`RAND(max)` scales the result. With a positive `U8`, `U16`, or `REAL`
maximum, it returns a value from `0` up to, but not including, `max`.
Integer `RAND(0)` returns `0`.

`RAND_SEED(seed)` seeds the target random source when it has software
state. Targets whose random source is hardware or ROM backed keep their
native source until `RAND_SEED` is called, then use a seeded software
stream for later `RAND` calls.

```basic
RAND_SEED 42
X#2 = RAND(320)
JITTER#1 = RAND(5)
COLOR#1 = RAND(U8(8))
R! = RAND()
```

Legacy dialects may also provide their original `RND(x)` behavior. Use
`RAND(max)` when you want the portable bounded form.

#### Numeric helpers

| Function    | Returns             | Description                                      |
| ----------- | ------------------- | ------------------------------------------------ |
| `MIN(a, b)` | common numeric type | Smaller value.                                   |
| `MAX(a, b)` | common numeric type | Larger value.                                    |
| `INC(x)`    | same type as `x`    | `x + 1`.                                         |
| `DEC(x)`    | same type as `x`    | `x - 1`.                                         |
| `DIG(n)`    | `U8`                | `1` for character codes `0` through `9`.         |
| `DIG(s$)`   | `U8`                | `1` for one-character strings `"0"` through `"9"`. |

`INC` and `DEC` accept `U8`, `U16`, `U32`, `I8`, `I16`, and `I32`,
including suffix-selected variables such as `COUNT#1` or `OFFSET%2`.

```basic
COUNT#1 = INC(COUNT#1)
OFFSET%2 = DEC(OFFSET%2)
```

#### Bitwise

| Function     | Operator | Returns | Description                       |
| ------------ | -------- | ------- | --------------------------------- |
| `BAND(a, b)` | `A & B`  | `U16`   | Bitwise AND.                      |
| `BOR(a, b)`  | `A | B`  | `U16`   | Bitwise OR.                       |
| `BXOR(a, b)` | `A ^ B`  | `U16`   | Bitwise XOR.                      |
| `BNOT(x)`    | -        | `U16`   | Bitwise NOT, one's complement.    |
| `SHL(x, n)`  | `X << N` | `U16`   | Shift left. Operator form requires a constant integer literal shift count. |
| `SHR(x, n)`  | `X >> N` | `U16`   | Shift right. Operator form requires a constant integer literal shift count. |

#### Logical

For boolean conditions (`IF`, `WHILE`, `UNTIL`) and truth-valued
expressions. These are operators, not PROCs, and return `0` or `1`.
Both sides of `AND`, `OR`, and `XOR` are always evaluated. The
operators do not short-circuit.

`TRUE` and `FALSE` are reserved constants and can be used anywhere a
numeric expression is accepted. CrustyBASIC uses the `U8` values `1`
and `0`. Compatibility dialects that use `-1` for true translate
user-written `TRUE` to that native value.

| Operator                | Description                                |
| ----------------------- | ------------------------------------------ |
| `A AND B` or `A && B`   | `1` only if both sides are non-zero.       |
| `A OR B` or `A \|\| B`  | `1` if either side is non-zero.            |
| `A XOR B`               | `1` if exactly one side is non-zero.       |
| `NOT A` or `!A`         | `1` if the operand is zero, `0` otherwise. |

#### REAL functions

The target has to support `REAL` to use these.

| Function    | Description                                                                |
| ----------- | -------------------------------------------------------------------------- |
| `POW(a, b)` | Exponentiation. Integer operands stay integer. For `REAL`, small non-negative integer constant exponents are optimized automatically; other exponents use the target's FP support. |
| `INT(x)`    | `REAL` to integer, truncate.                                               |
| `LOG(x)`    | Natural log.                                                               |
| `SGN(x)`    | Sign as a `REAL`.                                                          |
| `SIN(x)`    | Sine. Radians by default, degrees after `DEG()`.                           |
| `COS(x)`    | Cosine. Same convention.                                                   |
| `ATN(x)`    | Arctangent. Same convention.                                               |

`DEG()` and `RAD()` flip the process-wide trig angle mode. `RAD()` is
the default.

#### String functions

| Function                  | Returns  | Description                                       |
| ------------------------- | -------- | ------------------------------------------------- |
| `LEN(s$)`                 | `U8`     | Character count.                                  |
| `ASC(s$)`                 | `U8`     | First character code.                             |
| `CHR(n)`                  | `STRING` | One-character string.                             |
| `STR(n)`                  | `STRING` | Decimal representation.                           |
| `VAL(s$)`                 | `U16`    | Parse unsigned integer.                           |
| `LEFT(s, n)`              | `STRING` | Leading characters.                               |
| `RIGHT(s, n)`             | `STRING` | Trailing characters.                              |
| `MID(s, pos)`             | `STRING` | Substring from a 1-based position.                |
| `MID(s, pos, len)`        | `STRING` | Substring of the requested length.                |
| `INSTR(haystack, needle)` | `U16`    | 1-based position, 0 if absent. |
| `UPPER(s)`                | `STRING` | ASCII uppercase copy.                             |
| `LOWER(s)`                | `STRING` | ASCII lowercase copy.                             |
| `LTRIM(s)`                | `STRING` | Strip leading ASCII spaces.                       |
| `RTRIM(s)`                | `STRING` | Strip trailing ASCII spaces.                      |
| `TRIM(s)`                 | `STRING` | Both ends.                                        |
| `SPACE(n)`                | `STRING` | ASCII spaces.                                     |
| `REPLICATE(n, c$)`        | `STRING` | Copies of the first character.                    |
| `HEX(n)`                  | `STRING` | Uppercase hex, variable width.                    |
| `OCT(n)`                  | `STRING` | Octal, variable width.                            |
| `BIN(n)`                  | `STRING` | Binary, variable width.                           |
| `NIBBLE(n)`               | `STRING` | Single hex digit for `0..15`.                     |

String declaration syntax, capacities, and `MID(...) = ...` slice
assignment are documented in [Strings](#strings). `BIND` can point a
string at caller-managed memory.

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
| 1     | unary `-`, prefix `&`, operand `NOT` | highest where an operand is required |
| 2     | `*` `/` `\` `MOD`          | left-associative     |
| 3     | `+` `-`                    | left-associative     |
| 4     | `<<` `>>`                  | left-associative     |
| 5     | `&`                        | bitwise AND          |
| 6     | `^`                        | bitwise XOR          |
| 7     | `|`                        | bitwise OR           |
| 8     | `=` `<>` `<` `<=` `>` `>=` | left-associative     |
| 9     | leading `NOT` (or `!`)     | applies to compares  |
| 10    | `AND` (or `&&`)            | left-associative     |
| 11    | `OR` (or `\|\|`)           | left-associative     |
| 12    | `XOR`                      | lowest               |

`NOT X > 5` means `NOT (X > 5)`. Where an arithmetic operand is
required, `NOT` applies to that operand: `A * NOT B` means
`A * (NOT B)`. `A + B << 2` means `(A + B) << 2`.

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

For single key reads, use the input API:

```basic
K = KEY()
K$ = INKEY()
K = INKEY_CODE()
K = RAWKEY_CODE()
```

`KEY()`, `INKEY()`, and `INKEY_CODE()` read typed character events from
the target's normal key path. `KEY()` blocks; `INKEY()` and
`INKEY_CODE()` return immediately. `RAWKEY_CODE()` reads current held key
state when the target has a direct key path, so it is better for games
and controls.

### GET

```basic
GET CH
GET CH$
```

Blocks until a key is available. Numeric variables get the encoded
character code; string variables get a one-character string. No echo.
This is a typed character read like `KEY()`, not a current held key
poll.

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
without FP is a compile error. See [REAL functions](#real-functions)
for helper functions and the target docs under [`targets/`](targets/).

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

`{NAME}` may also name an integer `CONST`; in that case it expands to
the evaluated literal value. `STRING` and `REAL` constants are rejected.

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

### @STARTUP

Replace the target's startup template with a user-authored one. The
annotation marks a top-level `ASM` block as the program's startup:
the loader stub bytes, hardware init, the call into the program, and
the exit path all become yours.

```basic
@OPTION TARGET vic20
@OPTION START_PROGRAM $2001
@OPTION START_CODE $2010

@STARTUP
ASM
        ORIGIN ${start_program}
        WORD __bas_eop
        WORD 10
        BYTE $9e
        BYTE {start_code_decimal_bytes}
        BYTE 0
__bas_eop:
        WORD 0

        ORIGIN ${start_code}
__start:
        sei
        jsr __cb_target_init
        jsr {entry_label}
__halt:
        jmp ($C002)
ENDASM
```

Inside a `@STARTUP` block, `{...}` interpolation first matches the
startup placeholders below, and otherwise falls back to `DIM` global
names or integer `CONST` values like a regular `ASM` block. The block
must reference `{entry_label}` (the compiled program's entry point) or
the compile fails.

| Placeholder | Expands to |
| --- | --- |
| `{start_code}` | Machine code start as 4-digit hex (`START_CODE` or the target default). |
| `{start_code_decimal}` | The same address in decimal, e.g. for a `SYS` operand. |
| `{start_code_decimal_bytes}` | The decimal digits as comma-separated PETSCII byte values, for hand-assembled `SYS` operands. |
| `{start_program}` | Loaded program wrapper start as 4-digit hex (`START_PROGRAM`). |
| `{start_data}` | Mutable data start as 4-digit hex (`START_DATA`). |
| `{ram_top}` | Mutable-data ceiling as 4-digit hex. |
| `{ram_top_high}` | High byte of `{ram_top}` as 2-digit hex. |
| `{ram_size}` | RAM size as 4-digit hex. |
| `{ram_size_high}` | High byte of `{ram_size}` as 2-digit hex. |
| `{entry_label}` | Assembler symbol of the compiled program's entry point. |

Addresses expand without a `$` prefix, so templates write
`ORIGIN ${start_code}` to produce `ORIGIN $1010`.

Rules:

- At most one `@STARTUP` block applies per program; `ASM CPU`
  filtering works, so a listing can carry one block per CPU.
- A `@STARTUP` block conflicts with `@OPTION STARTUP`; pick one.
- A user startup carries no layout metadata, so `START_PROGRAM`
  requires an explicit `START_CODE`.
- Banked cartridge mappers build separate fixed/switched images with
  their own startups and do not support `@STARTUP`.

### @BANK / @ENDBANK

`@BANK PRG N` starts a switched-bank block; `@ENDBANK` closes it. Every
`PROC` and top-level `DATA` statement inside the block is placed in
switched bank `N`. Blank lines, comments, and `DATA` labels (`FOO:`)
are allowed inside the block.

```basic
@OPTION TARGET nes
@OPTION MAPPER uxrom

@BANK PRG 1
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

- Use `@BANK` only with a banked mapper: e.g. `uxrom`, `mmc1`,
  `mmc3`, `xegs32`, `supergames`, `banked_16k`.
- Bank numbers start at `0`. The always-available main area is not
  counted as a bank.
- `PROC MAIN` stays in the main area. Put shared helper PROCs there too.
- Code in the main area can call any bank.
- Inside `@BANK PRG N ... @ENDBANK`, the top level can contain only
  `PROC`, `DATA`, and `DATA` labels. Put `DIM`, `CONST`, inline `ASM`,
  and ordinary statements outside the bank block.
- Do not put `@BANK` blocks inside other `@BANK` blocks. Every
  `@ENDBANK` must match an earlier `@BANK`.
- Each call to a banked PROC starts `READ` at that bank's first `DATA`
  item. Use `RESTORE LABEL` when you need a specific item.
- String and `REAL` literals live with the bank that uses them.

Calling and DATA rules depend on the mapper's shape.

On **direct-fixed mappers** — the fixed window stays mapped while any
switched bank is active, and every banked PROC has a fixed-bank
trampoline (`xegs32`, NES `uxrom`/`mmc1`/`mmc3`):

- Code in any bank can call any other bank; cross-bank calls route
  through the callee's fixed-bank trampoline automatically.
- Unannotated PROCs are placed automatically; oversized call-graph
  groups are split across banks, largest PROC first, preferring the
  bank that already holds a PROC's callers and callees.
- `ADDR(PROC)` pins an auto-placed PROC to the fixed bank so raw
  addresses (interrupt handlers, call pointers) stay valid with any
  switched bank mapped.
- Main-area (non-`@BANK`) `DATA` is readable from **any** bank: the
  DATA cursor tracks the owning bank and the read helpers map it in
  around each fetch. When the fixed bank overflows, the compiler
  spills trailing `RESTORE`-target-delimited `DATA` runs into
  switched-bank free space; a `RESTORE`-then-`READ` sequence stays
  inside one relocated run, but sequential `READ`s that cross from
  one relocated run into the next are not supported.
- Explicit `@BANK PRG N` `DATA` keeps the same-bank rule below.

On **other banked mappers** (whole-window switching like EasyFlash, or
replicated fixed content like CoCo `banked_16k`):

- Code inside a bank can call the main area, but not another bank.
- A banked PROC can `READ` and `RESTORE` only that bank's own `DATA`.
  If code in the main area needs banked `DATA`, put the read in a PROC
  in that bank and call it.
- `RESTORE LABEL` can jump only to `DATA` in the same bank. Code in
  the main area can restore only main-area `DATA`.

Some mappers expose source-level asset banks too. On NES MMC3,
`@BANK CHR N "path"` places one file into an 8K CHR ROM bank.

### @DEFINE and @UNDEF

```basic
@DEFINE DEBUG
@UNDEF DEBUG
```

Set or clear a symbol that `@IF DEFINED(...)` can test while the
program is being compiled.

### @IF, @ELIF, @ELSE, @ENDIF

```basic
@IF TARGET = c64 THEN
    SID.MODEVOL = 15
@ELIF TARGET = atari800 THEN
    POKEY.AUDCTL = 0
@ELSE
    @ERROR "target does not support this module"
@ENDIF

@IF SYSTEM = apple2.plus THEN
    @INCLUDE "targets/apple2/applesoft_rom.cbi"
@ENDIF

@IF CODE_STORAGE = ram THEN
    PRINT "loaded from writable program memory"
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
| `TARGET = name`                                     | Active target name, such as `apple2`, `atari800`, `atari5200`, `atari2600`, `c64`, `c128`, `plus4`, `vic20`, `coco`, `nes`, `dos16`, or `winx64`. |
| `SYSTEM = name`                                     | Active system name, such as `apple2.plus`, `atari800.xl`, `c64.orig`, `coco.ecb`, or `nes.orig`. Using a target name here is a compile error. |
| `CPU = 6502`                                        | Current CPU string, such as `6502`, `6809`, or `z80`.                                                  |
| `TARGET_C64`, `TARGET_APPLE2`, ...                  | True when the active target matches.                                                                   |
| `SYSTEM_APPLE2_PLUS`, `SYSTEM_C64_ORIG`, ...        | True when the active system matches.                                                                   |
| `CPU_6502`, `CPU_6809`, `CPU_Z80`                   | CPU flag.                                                                                              |
| `CB_VERSION = "1.1.0"`                              | Compiler version string (matches `crustybasic --version`).                                             |
| `CB_VERSION_MAJOR`, `CB_VERSION_MINOR`, `CB_VERSION_PATCH` | Integer parts of the compiler version; use with `>=`, `<`, etc. for range checks.               |
| `DEFINED("symbol")`                                 | Symbol set with `@DEFINE`.                                                                             |
| `HAS_CHIP("name")`                                  | Target exposes the chip.                                                                               |
| `JOY_PORTS`, `GRAPHICS_AVAILABLE`, `BITMAP_AVAILABLE`, `PLOT_AVAILABLE`, `SPRITE_KIND`, etc. | Target constants such as controller counts and supported APIs. |

Conditions support `AND`, `OR`, `NOT`, parentheses, booleans,
strings, decimal integers, `$` hex integers, and the usual
comparisons.

String comparisons accept bare values when the left side is a string,
so `TARGET = c64`, `SYSTEM = vic20.8k`, `CPU = 6502`, and
`SYSTEM = atari800.xe`, and `CODE_STORAGE = ram` are valid. Quoted strings are still valid and are
required for the empty string or values that are not simple names, such
as `TARGET <> ""`.

Common coarse checks are `TEXT_OUTPUT_AVAILABLE`, `TEXT_CHARMAP_AVAILABLE`,
`GRAPHICS_AVAILABLE`, `BITMAP_AVAILABLE`, `PLOT_AVAILABLE`,
`FRAME_AVAILABLE`, and `SPRITE_AVAILABLE`. `GRAPHICS_AVAILABLE` means the
target provides portable graphics drawing hooks such as `PLOT`; fixed
cell targets may still accept `DISPLAY(CELL)`. Use `BITMAP_AVAILABLE`
when a program needs a real `BITMAP_*` display surface. `CODE_STORAGE` is `ram` for
writable program images and `rom` for cartridge/ROM images. `TARGET`
is the active target name
(e.g. `c64`, `apple2`, `coco`), which is useful when the system name
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
| `{target}` | Target name, such as `apple2`, `atari800`, `c64`, or `nes`. |
| `{system}` | Full system name, such as `apple2.plus`, `atari800.xl`, or `c64.orig`. |
| `{program_name}` | Program filename without extension. |
| `{media}` | Media variant, such as `cart`, when the selected system has one. |
| `{cpu}` | CPU name, such as `6502` or `6809`. |
| `{mapper}` | Selected mapper, such as `nrom` or `uxrom`, when one is active. |
| `{version}` | CrustyBASIC version string, such as `1.1.0`; always available. |

## Compiler options

Set with `@OPTION` or the CLI.

```basic
@OPTION TARGET c64
@OPTION DIALECT atari_basic
@OPTION OUTPUT_TYPE cart
@OPTION MAPPER supergames
@OPTION ARRAY_BASE 0
@OPTION NUMERIC_MODE INTEGER
@OPTION MATH_REAL AUTO
@OPTION BUILTIN_REAL AUTO
@OPTION START_PROGRAM $2001
@OPTION START_CODE $2100
@OPTION START_DATA $6000
@OPTION REGION_NTSC
@OPTION THROTTLE 10
```

| Option                           | Values                                  | What it does                                                                  |
| -------------------------------- | --------------------------------------- | ----------------------------------------------------------------------------- |
| `TARGET`                         | target name                             | Select the output target, such as `apple2`, `atari800`, `atari5200`, `atari2600`, `c64`, `c128`, `plus4`, `vic20`, `coco`, `nes`, `dos16`, or `winx64`. Must come before target dependent source. Passing a system name here is a compile error. |
| `SYSTEM`                         | system name                             | Select a specific system, such as `apple2.plus`, `atari800.xl`, `c64.orig`, `coco.ecb`, or `nes.orig`. Implies its parent target. Passing a target name here is a compile error. |
| `DIALECT`                        | dialect name                            | Apply that dialect's defaults to options you haven't set.                     |
| `ROM`                            | ROM profile name                        | Select a ROM profile for targets that expose one.                             |
| `OUTPUT_TYPE`                    | output name                             | Select a manifest output such as `cart`, `disk`, or `cart-easy-flash`.        |
| `THROTTLE`                       | `0..65535`                              | MOSLO-style delay. `0` disables throttling.                                   |
| `TILE_BACKEND`                   | `TILE_CELL`, `TILE_NATIVE`, `TILE_BITMAP`, `TILE_KERNEL` | Select the tile drawing backend at compile time.                |
| `MAPPER`                         | mapper name                             | Pick a cartridge mapper.                                                      |
| `NUMERIC_MODE`                   | see below                               | Default numeric policy.                                                       |
| `MATH_REAL`                      | `AUTO`, `TARGET`, `BUILTIN`             | Select the REAL math implementation.                                          |
| `BUILTIN_REAL`                   | `AUTO`, `Q16_16`, `Q24_8`               | Select the compiler-provided REAL format.                                     |
| `MATH_INTEGER`                   | `AUTO`, `TARGET`, `BUILTIN`             | Select the integer math implementation.                                       |
| `START_PROGRAM`                  | address                                 | Set the outer loaded program wrapper start when the startup format has one.   |
| `START_CODE`                     | address                                 | Set the generated machine code start address.                                 |
| `START_DATA`                     | address                                 | Set the mutable data start address for split code/data layouts.               |
| `STRING_BUFFER_LENGTH`           | `2..256`                                | Set the default and runtime string buffer length.                             |
| `CHR_ROM`                        | path string                             | Include a CHR ROM file in NES cartridge output.                               |
| `STARTUP`                        | startup name                            | Select a named startup template from the target manifests (e.g. `vic20_basic_8k`). A startup with program-wrapper metadata also moves `START_PROGRAM`/`START_CODE` to match. |
| `MEMORY_REGION`                  | target region name                      | Add a named target memory region. Repeating the option adds more regions.     |
| `MEMORY_ACTION`                  | target action name                      | Add a named target startup action.                                            |
| `REGION_NTSC`, `REGION_PAL`      | flag                                    | Pick timing/frequency tables. Default is NTSC.                                |
| `INLINE_ASM`                     | `TRUE`, `FALSE`                         | Allow `ASM ... ENDASM`.                                                       |
| `ARRAY_BASE`                     | `0`, `1`                                | First array index.                                                            |
| `FOLD_LOWERCASE_TO_UPPERCASE`    | `TRUE`, `FALSE`                         | Uppercase lowercase ASCII before target encoding.                             |
| `USE_LINE_NUMBERS`               | `TRUE`, `FALSE`                         | Force line-number mode. Complete listings are detected automatically.         |

Targets may also define their own `@OPTION`s. See the individual target
docs for those.

CLI `--set throttle=N` overrides `@OPTION THROTTLE N`.

### Numeric and math options

`NUMERIC_MODE` is normally chosen by the selected dialect. Override it
only when you want to change the dialect or native numeric policy:

- `INTEGER` - unsuffixed numeric names are integer.
- `REAL` - unsuffixed numeric names are `REAL`.
- `REAL_NARROW` - unsuffixed numeric names default to `REAL`, but
  proven integer-only implicit variables may use integer storage.
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

`BUILTIN_REAL` chooses the builtin `REAL` format when `MATH_REAL` is
`BUILTIN`, or when `MATH_REAL AUTO` resolves to builtin math. `AUTO`
uses the target's preferred builtin format. `Q16_16` has more
fractional precision; `Q24_8` has more whole-number range.

`TARGET` and `BUILTIN` are fixed choices. `--set optimize=speed` and
`--set optimize=size` do not change them.

`AUTO` can change with `--set optimize=speed` or
`--set optimize=size`, but only on targets that define a faster or
smaller choice. Otherwise it uses the target's normal default.

Integer math uses `MATH_INTEGER`. `REAL` math uses `MATH_REAL`. A mixed
program can use both.

### Memory regions

Memory regions are target data. They describe their address ranges,
supported uses, loading method, and limitations. A region may also
require a startup action, such as banking out ROM before the program
touches that RAM. For flat memory targets, direct regions only raise the
usable top when they connect to normal RAM without crossing a reserved
range.

Run `crustybasic target-info <system>` for the authoritative list of
memory regions available to that system. The command reads the target
manifest directly and shows each region's ranges, uses, loading method,
whether it is enabled automatically, required actions, and limitations.

The loading methods are `direct` when output bytes load at their final
address, `relocate` when startup copies initialized bytes into the
region, and `runtime` for uninitialized storage. Listed actions run
automatically. Listed disabled facilities cannot be used with that
region.

Enabling a region adds its address ranges to the memory the compiler may
use. The compiler decides what code or data to place there according to
the region's supported uses and loading method. Enabling a region does
not assign a particular variable or PROC to it.

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
crustybasic compile <program.bas> --set dialect=cbm_basic_v2 -o /tmp/program.s
```

```toml
dialect = "cbm_basic_v2"
```

Available dialects:

| Dialect | Use for |
| --- | --- |
| `crustybasic` | Native CrustyBASIC source. (the default so not needed) |
| `applesoft_basic` | Applesoft BASIC listings for Apple II. |
| `integer_basic` | Apple Integer BASIC listings for Apple II. |
| `atari_basic` | Atari BASIC listings for Atari 8 bit systems. |
| `basic_xl` | BASIC XL listings for Atari 8 bit systems. |
| `cbm_basic_vic20` | Commodore BASIC V2 listings for VIC-20. |
| `cbm_basic_v2` | Commodore BASIC V2 listings for C64. |
| `cbm_basic_v3_5` | Commodore BASIC 3.5 listings for Plus/4. |
| `cbm_basic_v7_0` | Commodore BASIC 7.0 listings for C128. |
| `color_basic` | TRS-80 Color BASIC listings for CoCo. |
| `extended_color_basic` | Extended Color BASIC listings for CoCo. |
| `qbasic` | QBASIC style listings. Select a target or system explicitly. |
| `gwbasic` | GW-BASIC listings for 16 bit DOS. |

Many compatibility dialects are partial and target scoped. See the
matching target page for current limits.

## Source Rewrites

`@REWRITE` lets you map one spelling to another before the program is checked.

`@REWRITE_EXPR OLD = NEW` rewrites expression fragments inside statements, such as translating `{l:atom}^{r:atom}` to `POW({l}, {r})`.

Call shaped rewrite rules without parentheses also accept source calls with
parentheses. For example, `@REWRITE_EXPR DISPLAY CELL = FAST_DISPLAY`
matches both `DISPLAY CELL` and `DISPLAY(CELL)`.

Add your own preferred syntax by putting `@REWRITE OLD = NEW`
lines in a `.cbi` file and listing that file in `crustybasic.config.toml`
with `include` to pick it up automatically or @INCLUDE'ing it in your source.

Targets may also publish source rewrite files through their manifest.
These target vocabulary aliases are available in native CrustyBASIC
source for that target only. e.g. `FAST` and `SLOW` for the C128.

## Mappers

Cart outputs and default mappers:

Only cart outputs can use `@OPTION MAPPER`. `none` means the output is a
fixed or linear cart layout with no mapper selection.

| System(s) | Cart output(s) | Default mapper | Other mapper options |
| --- | --- | --- | --- |
| `atari2600.orig` | `cart` `.a26` | `standard` | `f8` / `f6` / `f4` |
| `atari5200.orig` | `cart-32k` `.a52` | none | none |
| `atari800.orig` / `atari800.xl` / `atari800.xe` / `atari800.xegs` | `cart` `.car` | `standard` | `xegs32` |
| `c64.orig` / `c64.c64c` / `c64.ultimate` | `cart` `.crt`, `cart-easy-flash` `.crt` | `standard`; `easyflash` for `cart-easy-flash` | `supergames` / `easyflash` |
| `c128.orig` | `cart` `.bin` | none | none |
| `coco.1` / `coco.ecb` / `coco.3` | `cart` `.ccc` | `standard` | `banked_16k` |
| `nes.orig` | `cart` `.nes` | `nrom` | `uxrom` / `mmc1` / `mmc3` |
| `vic20.orig` / `vic20.3k` / `vic20.8k` / `vic20.16k` / `vic20.24k` | `cart` `.a0` | none | none |
