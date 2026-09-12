# crustyBASIC Language Reference

This page covers crustyBASIC syntax and built in language features.
The basics:

- In crustyBASIC source, a numeric name without a suffix or `AS` uses
  the target's default unsigned type: `U8`, `U16`, or `U32`. Add a
  suffix or `AS` to choose a specific type.
- Procedures and flow: `PROC`, `FOR`/`NEXT`, `WHILE`,
  `DO`. `GOTO` and `GOSUB` are also available.
- Built in language functions are documented below.
- Dialects translate older BASIC listings and choose compatible
  defaults. You can also use them for new programs.

[API.md](API.md) covers portable APIs, target support, and hardware access.
[USAGE.md](USAGE.md) covers the command line. See [Dialects](#dialects)
when compiling a program written for another BASIC.

## Code

A source file can contain directives, declarations, `PROC` definitions,
and statements. Put the main program in statements outside a `PROC`, or
in a `PROC MAIN` with no arguments. Do not use both styles. A bare `ASM`
block outside a `PROC` is also a statement, so put it inside `MAIN` when
using `PROC MAIN`.

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

Assigning to a suffixed name such as `NAME$`, `LIVES#1`, or `SCORE#2`
creates that variable. You can instead declare it with
`DIM <VAR_NAME> AS <TYPE>` and optionally give it a starting value.

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

Both forms continue to the end of the line. They can start a line or
follow a statement.

crustyBASIC ships with three block comment pairs:

| Start | End   |
| ----- | ----- |
| `/*`  | `*/`  |
| `/'`  | `'/`  |
| `'''` | `'''` |

```basic
/' this text is ignored,
   however many lines it runs to '/

X = 3 /' and inline, mid statement '/ + 4

/* this form works too */

'''
this whole region is ignored
'''
```

Block comments with different start and end delimiters nest, including
across delimiter pairs. A pair that uses the same delimiter for both
ends, such as `'''`, closes at the next delimiter and does not nest
inside itself.

```basic
/* outer
   /' inner '/
   ''' another inner block '''
   still commented out
*/
```

Nothing inside a block comment is read, including `@INCLUDE` and `@IF`.
A block that is never closed is an error.

Two things to know:

- A block opened part way through a statement has to close on the same
  line. `X = 3 /' note` followed by `'/ + 4` on the next line is two
  broken lines, not one expression.
- Closing delimiters are special only while they match an open block.
  `X = '/'` is still the one character string.
- An exact `'''` opens or closes a block before the single apostrophe
  comment rule is considered. A single `'` still starts a line comment.

To replace the shipped pairs, set matching lists in
`crustybasic.config.toml`:

```toml
block-comment-start = ["{*", "'''"]
block-comment-end = ["*}", "'''"]
```

Each start is paired with the end in the same list position. Both lists
must be present, nonempty, and the same length. Delimiters must contain
at least two ASCII punctuation characters and cannot overlap each other,
except when one pair uses the same start and end. List every pair you want
to use because configured lists replace all three shipped pairs.

Block comments are crustyBASIC syntax only.

### Multiple statements

Use `:` to chain statements on one line.

```basic
X = 1 : Y = 2 : PRINT X + Y
```

### Line numbers

crustyBASIC recognizes a complete listing with line numbers
automatically. A program cannot mix numbered and unnumbered program
lines. New code does not need line numbers. To require them, use
`@OPTION USE_LINE_NUMBERS TRUE`.

```basic
10 PRINT "HELLO"
20 GOTO 10
```

## Types

| Type     | Size            | Range                     |
| -------- | --------------- | ------------------------- |
| `U1`     | 1 bit, packed   | `0..1`                    |
| `U8`     | 1 byte          | `0..255`                  |
| `U16`    | 2 bytes         | `0..65535`                |
| `U32`    | 4 bytes         | `0..4294967295`           |
| `U64`    | 8 bytes         | `0..18446744073709551615` |
| `I8`     | 1 byte          | `-128..127`               |
| `I16`    | 2 bytes         | `-32768..32767`           |
| `I32`    | 4 bytes         | `-2147483648..2147483647` |
| `I64`    | 8 bytes         | `-9223372036854775808..9223372036854775807` |
| `REAL`   | varies by target | varies by target          |
| `STRING` | varies          | `0` through its capacity  |
| `ADDR`   | varies by target | target address range      |

crustyBASIC type suffixes:

| Suffix    | Type                                               |
| --------- | -------------------------------------------------- |
| `NAME`    | target's unsigned default (`U8`, `U16`, or `U32`)  |
| `NAME#1`  | `U8`                                               |
| `NAME#2`  | `U16`                                              |
| `NAME#4`  | `U32`                                              |
| `NAME#8`  | `U64`                                              |
| `NAME%1`  | `I8`                                               |
| `NAME%2`  | `I16`                                              |
| `NAME%4`  | `I32`                                              |
| `NAME%8`  | `I64`                                              |
| `NAME!`   | `REAL`                                             |
| `NAME$`   | `STRING`                                           |
| `NAME?`   | `DYNAMIC STRING`                                   |

Some targets do not support `U64` or `I64`. See the selected target's
page under [`targets/`](targets/).

An explicit `AS` sets the type. You can also use a matching suffix, as
in `DIM TOTAL#4 AS U32`. The suffix and `AS` type must agree.

`BOOL` is accepted as an alias for `U1`. `U1` is the normal spelling
used by crustyBASIC and there is no separate boolean type.

`?` is the one suffix that needs no `AS`, because a `DYNAMIC STRING`
sizes itself. `TEXT?` on its own line is a complete declaration.

`AS ADDR` declares a memory address with the size used by the target.
You cannot assign it to a smaller integer type.

`MAX_DEFAULT_UINT` and `MAX_DEFAULT_INT` give the largest unsigned and
signed values for the target's default integer width. They follow that
width, not the CPU or address width. Dialect and type policy overrides
do not change these target constants.

| Default integer width | `MAX_DEFAULT_UINT` | `MAX_DEFAULT_INT` |
| --------------------- | ------------------ | ----------------- |
| 8 bits                | 255                | 127               |
| 16 bits               | 65535              | 32767             |
| 32 bits               | 4294967295         | 2147483647        |

Use them in expressions or compile-time conditions to choose settings:

```basic
@IF MAX_DEFAULT_UINT = 255 THEN
    CONST STAR_COUNT = 30
@ELSE
    CONST STAR_COUNT = 200
@ENDIF
```

To use a different suffix style, define it in
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

If the config file defines more than one style, the source can choose
one:

```basic
@OPTION TYPE_POLICY my_suffixes
```

Suffixes must start with punctuation. Any remaining characters can be
punctuation or digits. The other settings choose the default numeric,
address, and index types. Keep the shown values unless you want to
change those types too.

Compatibility dialects use the suffixes from the BASIC they are
matching. See [Dialects](#dialects) for the short version.

In crustyBASIC source, the first assignment can declare a variable that
holds one value. A variable first assigned inside a `PROC` belongs to
that `PROC`; one first assigned outside a `PROC` is global. A variable
must be declared or assigned before it is read. Compatibility dialects
may follow different rules.

Integer literals:

- When assigned to an integer variable, a literal uses that variable's
  type if it fits. The destination can also provide the larger type
  needed by an expression. For example, `M#2 = K#1 * 8` can produce a
  `U16` result when `K#1` is `U8`.
- By itself, a positive literal uses the smallest unsigned type that
  fits. A negative literal needs a signed destination. Assigning a
  negative value to an unsigned destination is an error.

Signed and unsigned values can be mixed when one integer type can hold
both without losing information.

### Type conversion

crustyBASIC converts a number when it is used somewhere that requires a
specific type. This includes assignments, `READ`, `RETURN`, value
parameters, array and string indexes, addresses, and byte values.

- A smaller integer can go into a larger one without a warning, such as
  `U8` to `U16`.
- A literal that fits the destination is accepted without a warning.
- Integer math uses the declared type and wraps naturally. A `U8`
  counter wraps from 255 to 0, so `COUNT = COUNT + 1` needs no cast and
  does not warn.
- Putting a larger integer into a smaller type warns only when the value
  might not fit.
- An integer literal that cannot fit an assignment target is an error.
- `REAL` converts to 8 or 16 bit integer destinations, with a warning
  unless the expression is already explicitly integer, such as `INT(X)`.
  `REAL` does not convert directly to `U32`, `I32`, `U64`, or `I64`.
- Strings do not convert to numbers.

Some expressions clearly fit in a byte and can be assigned to `U8`
without a cast. For a `U16` value, these include `WORD & 255`,
`WORD \ 256`, `WORD MOD 256`, `WORD MOD BYTE_LIMIT` when `BYTE_LIMIT`
is `U8`, and `WORD >> 8`.

### Typed casts

`U1(expr)`, `U8(expr)`, `U16(expr)`, `U32(expr)`, `I8(expr)`, `I16(expr)`,
and `I32(expr)` convert a numeric expression to that type. This says that
the conversion is intentional, so it does not warn even when the value
wraps. There are no `U64(expr)` or `I64(expr)` cast forms.

- Values outside the chosen type's range wrap instead of causing a
  literal range error: `U1(3)` is `1`, `U8(300)` is `44`, `U8(-1)` is
  `255`, and `I8(255)` is `-1`.
- Casts are valid in `CONST` initializers: `CONST LOW = U8($1234)` is
  `$34`.
- Casting a `STRING` is an error. `REAL` has the same limits as a normal
  conversion: it cannot be cast to a 32 or 64 bit integer. The cast only
  removes the warning.

```basic
HI#1   = PTR#2 >> 8           ' high byte
LO#1   = PTR#2 & $FF          ' low byte
IDX    = IDX + 1              ' U8 arithmetic wraps naturally
BYTE#1 = U8(VALUE#2)          ' keep the low byte on purpose
```

Use a cast when deliberately putting a larger value into a smaller type.
It silences the "possible lossy conversion" warning for that expression.

Signed integers use two's complement. Division drops the fractional
part toward zero. `MOD` keeps the sign of the left operand.

`REAL` depends on the target. See the target pages under
[`targets/`](targets/).

## Declaration examples

```basic
DIM X AS U16
DIM B AS U8
DIM VALUE AS REAL
DIM NAME AS STRING
DIM TITLE AS STRING * 80

DIM X_POS, Y_POS, VX, VY AS U8
DIM LEFT_EDGE, RIGHT_EDGE AS U16
DIM FIRST_NAME, LAST_NAME AS STRING

DIM COUNT AS U8 = 1, TOTAL AS U16 = 300

DIM SCORES(10) AS U16        ' 11 elements with base 0, 0...10
DIM GRID[7, 7] AS U8
DIM MAP(15, 11), SHADOW(15, 11) AS U8
DIM START_VALUE AS U8 = 0
DIM NUMS[4] AS U8 = 1, 2, 3, 4, 5
PRINT LEN(NUMS)              ' element count

SCORE#2 AS U16               ' declaration without DIM
TITLE$ AS STRING * 80
LEFT_EDGE, RIGHT_EDGE AS U16
SCORES(10) AS U16            ' array declaration without DIM
GRID[7, 7] AS U8
FLAGS(3)
BUF[3] AS U8 = 1, 2, 3, 4
SCORES#2[10]                 ' U16 from suffix
NAMES[3] AS STRING
```

`DIM`, a declaration without `DIM`, and `CONST` all take a comma
separated list of names. Each name can carry its own `AS <type>` and
`= value`. A name with no type of its own takes the type of the last
name on the line, so `DIM X_POS, Y_POS, VX, VY AS U8` types all four.
An array with an initializer ends the list, since the values after its
`=` are the array contents. Put any further names on their own line.

The last array index is included. `A(10)` has indexes `0..10`, or
`1..10` with `ARRAY_BASE 1`. You can use either `()` or `[]` for array
declarations, initializer lists, and references. `LEN(A)` returns the
number of elements. `ADDR(A(I))` returns the address of one element. For
a string array, use `ADDR(A(I))` for the element you need rather than
calculating its address from another element.

`U1` scalars and arrays are packed from bit 0 upward. Up to eight scalar
declarations with the same owner share one byte, so `A, B, C AS U1`
takes one byte. A `U1` array takes one byte for every eight elements:

```basic
A, B, C AS U1             ' 1 byte
FLAGS(38) AS U1           ' 39 elements, 5 bytes
```

`U1` parameters, return values, and expression results use a byte while
they are being passed or evaluated. Reading or writing a packed value
needs extra shift and mask work, so use `U8` when access speed matters
more than stored size.

`ADDR` of a `U1` scalar or array element returns the address of its
containing byte. The bit number is not part of the address.

Use `ALIGN <bytes>` when a global array must start at an address that is
evenly divisible by a particular number of bytes:

```basic
SCREEN[999] AS U8 ALIGN 1024
DIM WORK(127) AS U8 ALIGN 256
```

The byte count must be a positive integer literal. `ALIGN` applies to
one global array declaration, including a `CONST` array. It is not
available on arrays declared inside a `PROC` or arrays whose size is
chosen while the program runs. Some assemblers accept only powers of
two.

An array can take its upper bound from a value calculated while the
program runs. This makes the array itself dynamic; there is no separate
`DYNAMIC ARRAY` declaration:

```basic
PROC MAIN
    N AS U16
    OK AS U8 = 0

    WHILE OK = 0
        PRINT "SIZE: ";
        OK = INPUT(N)
    ENDWHILE

    A(N) AS U8
    A(0) = 7
ENDPROC
```

These arrays support one numeric dimension and cannot have an
initializer. Elements can be any type, including `STRING * N`. The
array lives on the heap, so the target must have one; see
[`HEAP_SUPPORTED`](API.md#heap).

A dynamic `U1` array also packs eight elements per byte. `LEN` and
`RESIZE` still count elements, not bytes.

Each time the declaration runs it makes the array exactly that size. An
unchanged size costs one comparison and nothing else. A changed size
keeps the elements that still fit. New numeric elements start at zero and
new string elements start empty. The program ends if there is not enough
memory.

`RESIZE A(NEW_UPPER)` changes the upper bound without running the
declaration again. It keeps the elements that still fit, removes entries
after the new bound, and starts new entries empty when the array grows.

`ERASE <name>` hands a runtime sized array's memory back. `LEN` reads
`0` afterwards, and running the declaration again allocates it fresh.

An `A(10)` statement calls `PROC A` when that PROC exists. Otherwise it
declares an array without `DIM`. The same name cannot be both a `PROC`
and a variable or array.

Declarations inside a `PROC` belong to that `PROC`, and their values are
kept between calls. A local name cannot hide a global, `CONST`, `PROC`,
or parameter. Different PROCs can use the same local name; each gets
its own variable.

### Strings

Each string variable has a fixed maximum size. `S$ AS STRING` gets the
target's default capacity. To pick a different size, use `STRING * N`
where `N` is the exact usable capacity:

```basic
S$ AS STRING
NAME$ AS STRING * 16
LINE$ AS STRING * 80
HEX$ AS STRING * (SIZEOF(U16) * 2)

S$ = "HELLO"
LINE$ = S$ + " WORLD"
S$ += "!"

MID(S$, 0, 5) = "HOWDY"
```

`+=` appends to a string variable without replacing its existing contents.

The capacity can use constant integer arithmetic.
`SIZEOF(T)` works in numeric expressions for the fixed numeric types
`U1`, `U8`, `U16`, `U32`, `U64`, `I8`, `I16`, `I32`, and `I64`.
`SIZEOF(U1)` is `1`, the smallest addressable storage unit. Templates
can therefore use expressions such as `SIZEOF(@T) * 8`.

`STRING_DEFAULT_CAPACITY` changes the usable capacity of declarations
written as plain `STRING`. For example, a value of 256 allows 256
characters. An explicit `STRING * N` always has capacity `N`.

A `DYNAMIC STRING` takes its characters from the heap instead of a fixed
buffer, so it only ever occupies what it is currently holding:

```basic
NAME$ AS STRING             ' fixed, the target's default capacity
DESC$ AS STRING * 120       ' fixed 120
TEXT$ AS DYNAMIC STRING     ' takes only the room it needs
TEXT?                       ' the same thing, written short
```

It still holds at most the target's default capacity, the same as a plain
`STRING`. What changes is cost: a table of a hundred short descriptions
costs what those descriptions actually take rather than a hundred full
buffers.

Assigning a shorter value hands any usable tail back to the heap.
The memory comes back when you assign the empty string:

```basic
TEXT$ = "YOU ARE IN A MAZE OF TWISTY PASSAGES"
TEXT$ += ", ALL ALIKE"
TEXT$ = ""                  ' hands the block back
```

An array works the same way, one block per element:

```basic
ROOM$(199) AS DYNAMIC STRING
ROOM?[199]                  ' the same thing, written short

READ ROOM$(0)
ROOM$(1) = "A LONG HALL RUNS NORTH."
ROOM$(1) += " A DOOR STANDS OPEN."
ROOM$(1) = ""               ' hands that element's block back
```

A `?` array declares with brackets or with `DIM ROOM?(199)`. Written as
`ROOM?(199)` on its own line it reads as a call instead. Elements are
always indexed with parentheses, as in `ROOM?(1)`.

The array itself is a table of pointers, two bytes per element. An element
you never write reads as the empty string. This is where the saving shows
up: two hundred descriptions the compiler cannot predict cost what they
actually hold instead of two hundred full buffers.

The size can come from a value calculated while the program runs.
`RESIZE` trims or grows the table, and `ERASE` hands back the elements as
well as the table:

```basic
ROOM$(COUNT) AS DYNAMIC STRING
...
RESIZE ROOM$(NEW_LAST)
ERASE ROOM$
```

Shrinking keeps entries through `NEW_LAST` and frees every string after
it. Assigning `""` frees only that element's contents; the element and
the array's size remain unchanged. `RESIZE` does not search for empty
elements.

Putting `MID` on the left of `=` does not work on an element of one of
these; assign the whole element instead.

You mark declarations. `PROC` parameters and return values written as a
plain `STRING` follow whatever reaches them, so `DYNAMIC` never has to
appear in a signature:

```basic
PROC SHOW(S$ AS STRING)      ' takes a DYNAMIC STRING without a copy
    PRINT S$
ENDPROC

PROC JOIN(A$ AS STRING, B$ AS STRING) AS STRING
    R$ AS DYNAMIC STRING     ' so the return value is dynamic too
    R$ = A$ + " " + B$
    RETURN R$
ENDPROC

PROC CLIP(S$ AS STRING * 8)  ' you asked for a fixed 8, you get it
    PRINT S$
ENDPROC
```

Anything you gave a size to keeps it, including a plain `STRING` you
declared yourself. Only storage the compiler had to size for you follows.

The target needs a heap; see [`HEAP_SUPPORTED`](API.md#heap).

crustyBASIC rejects an assignment when the value is known to be too
large. When the size is known only while the program runs,
`--show-warnings` can display a warning instead. This includes an append
when the current string length is unknown.

String bounds checks are off by default. Enable them with:

```basic
@OPTION STRING_BOUNDS_CHECKS TRUE
```

With checks enabled, a whole string assignment or append first checks
the new length. If it is too large, the program prints
`STRING BOUNDS ERROR`, adds a newline, and ends without changing the
string.

Putting `MID` on the left of `=` overwrites those characters without
changing the string's length.

After [`BIND`](API.md#bind) points a string at memory you provide, reads
and writes use that memory.

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

A numeric or string `CONST` gives a name to a value that cannot change.

One `CONST` can declare several names, separated by commas, using the
same list rules as `DIM`. Each gets its own value, and a later one can
use an earlier one:

```basic
CONST WIDTH = 40, HEIGHT = 25, CELLS = WIDTH * HEIGHT
```

A numeric or string `CONST` can also be declared inside a `PROC`. It is
visible everywhere in that `PROC`, including lines above the
declaration, and it hides a global or `CONST` of the same name for the
whole `PROC`:

```basic
CONST LIMIT = 3

PROC SHOW
    CONST LIMIT = 5
    PRINT LIMIT
ENDPROC
```

`SHOW` prints 5, while code outside it still sees 3. Two PROCs can use
the same local `CONST` name with different values. A local `CONST` name
cannot match a parameter, a local variable, or a `PROC`. `CONST` arrays
must be declared outside a `PROC`.

Typed `CONST` arrays hold values that cannot be changed. Numeric
`CONST` arrays can be read like normal arrays, and you can take their
address:

```basic
PTR#2 = ADDR(SHAPE)
PRINT HEX(SHAPE(2))
```

### PROC

```basic
[@ASYNC]
[@ASYNC_INSTALL param]
[@ASYNC_UNINSTALL name[, name]]
[@WARNING "message"]
PROC name [([IN|OUT|INOUT] param [AS type][, ...])] [AS type]
    [RETURN [expr]]
ENDPROC
```

Empty parentheses are optional. A parameter list in a `PROC` definition
still needs them. Calls can use parentheses or leave them out when the
meaning is clear:

```basic
KEY()
POSITION(0, 0)
POSITION 0, 0
DELAY 60
```

If a call starts its arguments with `(`, it must end them with `)`.

Parameters can use `AS` types or type suffixes. Add `OUT` or `INOUT`
when the `PROC` should copy its final value back to the caller's
variable. A `PROC` with a return type is used in an expression. One
without a return type is called as a statement.

```basic
PROC DOUBLE(X AS U8) AS U8
    RETURN X + X
ENDPROC

PRINT DOUBLE(4)
PRINT DOUBLE 4
```

Both calls print the same value.

Rules:

- Parameters are passed by value.
- Value parameters use the normal numeric conversions. Moving to a
  larger compatible type is silent. A conversion that might lose data
  or change between signed and unsigned types produces a warning.
- Use `AS type ADDR` for the address of numeric data. Pass `ADDR ARRAY`,
  `ADDR ARRAY(I)`, or a raw numeric address. When passing addressable
  data, its element type must match the parameter's element type.
- Inside the PROC, a `type ADDR` parameter can be indexed with `[]` or
  `()` to read and write memory at that address. Each index moves by the
  size of the element type; `U1 ADDR` indexes bits packed from bit 0.
  Indexed `ADDR` parameters currently support `U1`, `U8`, `I8`, `U16`,
  and `I16` elements. The parameter is also a normal
  address, so `PEEK`, `POKE`, and other memory calls can use it directly.
- Assigning to a normal parameter does not affect the caller.
- `OUT` parameters require an assignable variable argument with the
  same type. The final value is copied to that variable when the `PROC`
  returns.
- `INOUT` parameters also require an assignable variable argument with
  the same type. The caller's value is copied in before the call and
  copied back when the `PROC` returns.
- A `PROC` can call itself, directly or through another `PROC`, on 6502
  and 68000 targets. Each nested call saves the PROC's parameters, locals,
  and working values on the hardware stack, so keep recursion shallow on
  8 bit machines. Other targets report an error. A `PROC` still cannot be
  started by a handler while an earlier call is running.
- A typed `PROC` can be used anywhere its return type is valid.
- Interrupt callbacks take no arguments. Install them using `ADDR(name)`
  and keep their work short and simple.
- `@ASYNC_INSTALL param` declares an interrupt callback parameter.
  Pass a no argument PROC using `ADDR(name)`.
- `@ASYNC` declares an interrupt handler whose address is passed through
  a variable or written directly to an interrupt vector.
- `@ASYNC_UNINSTALL <name>[, <name>]` marks a `PROC` that switches
  handlers off. Each name is the `PROC` that installed one of those
  handlers. When a program uses one of the named installers, the compiler
  calls the marked `PROC` before `END` and when the program runs off its
  end, so handlers never outlive the program.
- `@WARNING "message"` shows the message when your source calls that
  overload.

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

crustyBASIC chooses an overload from the argument types. An exact match
is preferred, followed by a move to a larger type without data loss,
then a conversion that might lose data. No match or a tie is an error.

Use `@TYPE_TEMPLATE` to create one version for each listed type:

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

`SWAP` exchanges two variables of the same type that each hold one
value. It does not work with strings, arrays, `MID` slices, target
register names, or constants.

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

Hexadecimal, binary, and octal forms work only for integers. The
lowercase `h` suffix avoids a name conflict: `CH` stays an identifier,
while `FFh` is hexadecimal.

crustyBASIC converts string literals to the target's character set.
Inside a literal, `{NAME}` inserts a named target control code. Each
target lists its names, which usually include `{CLR}`, `{HOME}`,
`{RETURN}`, and `{TAB}`:

```basic
PRINT "{CLR}READY."
```

String literals also accept the placeholders listed under
[`@INCLUDE`](#include):

```basic
PRINT "{CLR}BUILT FOR {system} ON {cpu}"
```

A known placeholder without a value, such as `{mapper}` when no mapper
is selected, is a compile error. An unknown `{name}` stays as literal
text. Write `{{target}}` to produce the text `{target}`. `\t` produces
the same bytes as `{TAB}`.

`ADDR "..."` uses the same encoding. The bytes at that address match
what `PRINT` would write for the same literal, including escapes.

Visual binary is handy for sprites, tiles, and character shapes:

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

Use `TILE_DEFINE` to give one of these shapes a tile ID:

```basic
TILE_DEFINE TILE_SHIP, ADDR SHIP
```

See [`TILE_DEFINE`](API.md#tile) in the TILE API for supported targets
and shape requirements. A target may convert constant shape data while
building the program or convert it while the program runs.

The default visual bit characters are `X` for 1 and `.` for 0.
Change them in `crustybasic.config.toml`:

```toml
bit-on-char = "#"
bit-off-char = "-"
```

`PACK N` combines each group of `N` rows into one integer. It aligns the
bits to the right, with the first written bit at the high end:

```basic
CONST FONT_DATA(0) AS U16 = % PACK 5
    .X.
    X.X
    XXX
    X.X
    X.X
END%
```

`EDGE(direction, first, step)` turns a rectangular visual block into
one coordinate for each column or row of a `CONST` array:

```basic
CONST HILL[7] AS U8 EDGE(TOP, 100, 2) = %
....X...
...XXX..
..XXXXX.
XXXXXXXX
END%
```

This produces `106, 106, 104, 102, 100, 102, 104, 106`. The first
source row has value 100, and each row below adds 2.

| Direction | Generated values |
| --------- | ---------------- |
| `TOP` | Topmost set bit in each column |
| `BOTTOM` | Bottommost set bit in each column |
| `LEFT` | Leftmost set bit in each row |
| `RIGHT` | Rightmost set bit in each row |

For `TOP` and `BOTTOM`, `first` is the value of the first source row
and `step` is added for each row below it. For `LEFT` and `RIGHT`,
`first` is the value of the first source column and `step` is added
for each column to its right. Both arguments must be constant integer
expressions.

Every row must have the same width. `TOP` and `BOTTOM` require a set
bit in every column. `LEFT` and `RIGHT` require a set bit in every row.
`EDGE` blocks may be wider than 32 cells and cannot use `PACK`. The
array contains the resulting coordinates.

### Built in functions

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

`RAND_SEED(seed)` starts a repeatable sequence. Without an explicit
seed, a target may supply a changing initial value.

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
| `DIG(s$)`   | `U8`                | `1` for strings containing one digit, `"0"` through `"9"`. |

`INC` and `DEC` accept `U8`, `U16`, `U32`, `I8`, `I16`, and `I32`,
including variables typed by a suffix, such as `COUNT#1` or `OFFSET%2`.

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
| `SHL(x, n)`  | `X << N` | `U16`   | Shift left.                       |
| `SHR(x, n)`  | `X >> N` | `U16`   | Shift right, zero fill.           |

#### Logical

Use these operators for conditions in `IF`, `WHILE`, and `UNTIL`, or
any other expression that needs true or false. They return `U1` values,
either `0` or `1`.
`AND`, `OR`, and `XOR` always evaluate both sides.

`TRUE` and `FALSE` are reserved constants and can be used anywhere a
numeric expression is accepted. crustyBASIC uses the `U1` values `1`
and `0`. A compatibility dialect may use its original BASIC's value for
`TRUE`, such as `-1`.

| Operator                | Description                                |
| ----------------------- | ------------------------------------------ |
| `A AND B` or `A && B`   | `1` only if both sides are non-zero.       |
| `A OR B` or `A \|\| B`  | `1` if either side is non-zero.            |
| `A XOR B`               | `1` if exactly one side is non-zero.       |
| `NOT A` or `!A`         | `1` if the value is zero, `0` otherwise.   |

#### REAL functions

The target has to support `REAL` to use these.

| Function    | Description                                                                |
| ----------- | -------------------------------------------------------------------------- |
| `POW(a, b)` | Exponentiation. Integer values stay integer; `REAL` values use floating point. |
| `INT(x)`    | Round down to the greatest whole number less than or equal to `x`. Returns `REAL`. |
| `EXP(x)`    | Natural exponential.                                                       |
| `LOG(x)`    | Natural log.                                                               |
| `SGN(x)`    | Sign as a `REAL`.                                                          |
| `SIN(x)`    | Sine. Radians by default, degrees after `DEG()`.                           |
| `COS(x)`    | Cosine. Same convention.                                                   |
| `ATN(x)`    | Arctangent. Same convention.                                               |

`DEG()` and `RAD()` change the angle mode for all trigonometry calls.
`RAD()` is the default.

`INT(2.5)` is `2`, `INT(-2.5)` is `-3`, and `INT(-2.0)` is `-2`.

#### String functions

| Function                  | Returns  | Description                                       |
| ------------------------- | -------- | ------------------------------------------------- |
| `LEN(s$)`                 | `U8` or `U16` | Character count. Uses `U16` when the string can exceed 255 characters. |
| `ASC(s$)`                 | `U8`     | First character code.                             |
| `CHR(n)`                  | `STRING` | String containing one character.                  |
| `STR(n)`                  | `STRING` | Decimal representation.                           |
| `VAL(s$)`                 | `U16`    | Unsigned integer read from text.                  |
| `LEFT(s, n)`              | `STRING` | Leading characters.                               |
| `RIGHT(s, n)`             | `STRING` | Trailing characters.                              |
| `MID(s, pos)`             | `STRING` | Substring from a position, counted from `ARRAY_BASE`. |
| `MID(s, pos, len)`        | `STRING` | Substring of the requested length.                |
| `INSTR(haystack, needle)` | `U16`    | Position numbered from 1, or 0 if absent.         |
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
| `NIBBLE(n)`               | `STRING` | Single hex digit for `0` through `15`.            |

String declarations, capacities, and `MID(...) = ...` assignment are
covered in [Strings](#strings). `BIND` can point a string at memory you
provide.

### Operators

| Operators                  | Meaning                                                    |
| -------------------------- | ---------------------------------------------------------- |
| `+` `-` `*` `/`            | Arithmetic                                                 |
| `\`                        | Integer divide. Rejects `REAL` - use `INT(A / B)` instead. |
| `MOD`                      | Integer remainder                                          |
| `POW(a, b)`                | Exponentiation                                             |
| `&`, `|`, `^`              | Bitwise AND, OR, XOR                                       |
| `<<` `>>`                  | Logical shift left / right                                 |
| prefix `&`                 | Address of (same as [`ADDR(...)`](API.md#addr))             |
| `=` `<>` `<` `<=` `>` `>=` | Comparison                                                 |
| `NOT` `AND` `OR` `XOR`     | Logical, returns `0` or `1`                                |
| `!` `&&` `\|\|`            | Other spellings for `NOT`, `AND`, and `OR`                 |
| `+` on strings             | Join strings                                               |
| `+=` on string variables   | Append to the existing string                              |

The meaning of `&` depends on its position. Some compatibility dialects
use `& BYTE, COUNT` at the start of a statement to print a byte several
times. In an expression, `&FOO` means the address of `FOO`, while
`A & B` is bitwise AND.

A constant shift count must be smaller than the number of bits in the
value. When the count comes from a variable, a count that large gives
`0`.

`>>` always zero fills, so `-1024 >> 3` is not a divide by 8. Use `/`.

### Precedence

Lower numbered levels are evaluated first.

| Level | Operators                  | Notes                |
| ----- | -------------------------- | -------------------- |
| 1     | unary `-`, prefix `&`, `NOT` before a value | before other operators |
| 2     | `*` `/` `\` `MOD`          | evaluated left to right |
| 3     | `+` `-`                    | evaluated left to right |
| 4     | `<<` `>>`                  | evaluated left to right |
| 5     | `&`                        | bitwise AND          |
| 6     | `^`                        | bitwise XOR          |
| 7     | `|`                        | bitwise OR           |
| 8     | `=` `<>` `<` `<=` `>` `>=` | evaluated left to right |
| 9     | `NOT` (or `!`) before a comparison | applies to the comparison |
| 10    | `AND` (or `&&`)            | evaluated left to right |
| 11    | `OR` (or `\|\|`)           | evaluated left to right |
| 12    | `XOR`                      | lowest               |

`NOT X > 5` means `NOT (X > 5)`. Where an arithmetic value is
required, `NOT` applies to that value: `A * NOT B` means
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

`ELSE IF` can also be used instead of `ELIF`.

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

`SELECT CASE` compares one value with each `CASE` in order and runs the
first match. A `CASE` can list several values. `CASE ELSE` is optional
and runs only when no earlier case matches.

### FOR

```basic
FOR I = 1 TO 10
    PRINT I
NEXT I

FOR I = 10 TO 1 STEP -1
    PRINT I
NEXT I
```

`STEP` defaults to 1. The loop stops when its variable passes the end
value in the direction of the step. With `STEP 0`, the variable never
changes. The loop runs forever when its starting value is no greater
than its end value, and is skipped otherwise.

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

The loop body always runs at least once.

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

A `DO` loop can test `WHILE` or `UNTIL` before the body, after it, or
not at all.

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

Without a loop name, `EXIT` and `CONTINUE` use the loop containing them.
A form such as `EXIT FOR` or `CONTINUE WHILE` uses the nearest containing
loop of that kind. There must be a matching loop.

### Labels, GOTO, GOSUB, POP

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

`RETURN` resumes after the most recent pending `GOSUB`. `POP` discards
that return address and continues with the next statement. A later
`RETURN` resumes after the next outer `GOSUB`.

```basic
GOSUB WORK
END

WORK:
    POP
    GOTO FINISHED

FINISHED:
    PRINT "done"
    END
```

Use `POP` only with a pending `GOSUB` in the current PROC. It takes no
arguments and does not exit a loop; use `EXIT` for that. Too many pending
`GOSUB` calls, or `RETURN` or `POP` without a matching call, can crash the
program.

When using line numbers, `GOTO <EXPR>` and `GOSUB <EXPR>` also work. The
result is matched against the program's line numbers while it runs.

### ON GOTO / ON GOSUB

```basic
ON MODE GOTO TEXT_MODE, GRAPH_MODE, SPRITE_MODE
ON CHOICE GOSUB OPT1, OPT2, OPT3
```

A value of `1` jumps to the first label, `2` to the second, and so on.
If the value has no matching entry, execution continues with the next
statement. In line number mode, the list can contain line numbers
instead of labels.

### END and RUN

`END` ends the program. `RUN`, or `RUN()` in crustyBASIC source,
restarts it from the top. A program does not need `END` at the bottom.

### ON_ERROR

```basic
ON_ERROR A_LABEL
ON_ERROR OFF
ON_ERROR 300
ON_ERROR HANDLER_PROC
ON_ERROR X
ON_ERROR X + 5
```

`ON_ERROR` sets a handler for target I/O errors. `OFF` disables it.
The handler is paused while it is running so another error does not
enter it again.

The handler can be a label in the same `PROC`, a line label when using
line numbers, a `PROC` with no arguments or return type, or a numeric
expression. A numeric result is matched to a line number. If there is no
match, the handler is disabled.

Inside a handler, `ERR()` returns the saved `U8` error code until
the next error occurs.

```basic
RESUME
RESUME_NEXT
```

`RESUME` retries the statement that raised the error. `RESUME_NEXT`
continues with the following statement. Both leave the handler and
reactivate it.

Only target ROM or operating system calls that report an error can
trigger `ON_ERROR`. See the selected target's page under
[`targets/`](targets/) for support details.

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

`PRINT` writes text. A semicolon puts items next to each other with no
added space. A comma adds one space. `PRINT` adds a newline unless the
statement ends with `;` or `,`.

`?` at the start of a statement is short for `PRINT`. Attached to the end
of a name it is the `DYNAMIC STRING` suffix instead, so `? TEXT?` prints
a dynamic string.

`PRINT USING` formats numbers and strings into fixed fields:

```basic
FOR I = 1 TO 15
    PRINT USING "##"; I
NEXT I

FORMAT$ = "TOTAL: $$###,###.##"
PRINT USING FORMAT$; TOTAL
```

The format must be a string followed by `;` and at least one value.
Commas and semicolons both separate values without adding spaces. A
final `;` or `,` prevents the newline. Literal characters are copied to
the output. If values remain after the end of the format, formatting
starts again from the beginning. A format can contain up to 255
characters.

| Field | Meaning |
|---|---|
| `!` | first character of a string |
| `&` | whole string |
| `\   \` | fixed width string, including both backslashes |
| `#` | numeric digit position |
| `.` | decimal point |
| `,` | thousands separators |
| `+` before the field | leading sign |
| `+` after the field | trailing sign |
| `-` after the field | trailing minus or a space |
| `**` | fill unused positions with `*` |
| `$$` | leading dollar sign |
| `**$` | star fill and a dollar sign |
| `^^^^` | scientific notation |
| `_` | print the next format character literally |

If a number is too wide for its field, `%` is printed before it. A
format can use at most 24 numeric digit positions. If a value's type
does not match the next field, the program reports `TYPE MISMATCH`. An
empty format or one with no field reports `ILLEGAL FUNCTION CALL`.

Some targets cannot use `PRINT USING` because they have very little
writable RAM or do not provide general text output. See the selected
target's page under [`targets/`](targets/).

### INPUT

```basic
X AS U16
NAME AS STRING * 40
OK AS U8

OK = INPUT(X)

OK = 0
WHILE OK = 0
    PRINT "Enter name: ";
    OK = INPUT(NAME)
ENDWHILE
```

`INPUT(VALUE)` reads and echoes one line. The type of `VALUE` tells it
whether to read a number or a string. It returns `1` after assigning the
value.

Invalid numeric text, a number that is too large, or text that does not
fit returns `0` and leaves `VALUE` unchanged. Each call tries once. It does
not print an error or try again, so your code controls the prompt and
loop. The call waits for Enter or the target's equivalent key.

A compatibility dialect may also accept the traditional `INPUT X`
statement. That form ignores the returned status. It still does not
retry or print an error automatically.

For individual key reads, use the input API:

```basic
K = KEY()
K$ = INKEY()
K = INKEY_CODE()
K = RAWKEY_CODE()
```

`KEY()`, `INKEY()`, and `INKEY_CODE()` read typed characters. `KEY()`
waits for a key; `INKEY()` and `INKEY_CODE()` return immediately.
`RAWKEY_CODE()` reports a key that is currently held when the target
supports it, which is usually better for games and controls.

### GET

```basic
GET CH
GET CH$
```

`GET` waits for a key. A numeric variable receives the encoded character
code, while a string variable receives a one character string. The key
is not echoed. Like `KEY()`, this reads a typed character rather than
checking which key is currently held.

### DATA, READ, RESTORE

```basic
ITEMS:
DATA 10, 20, 30, "hello", 42

READ A, B, C
READ S$
RESTORE
RESTORE ITEMS
```

`DATA` stores a list of values. `READ` takes them one at a time and
converts each one to the receiving variable's type. `RESTORE` goes back
to the first value, while `RESTORE LABEL` moves to a named `DATA` block.
When using line numbers, `RESTORE <EXPR>` moves to the `DATA` at the
matching line number. Do not read past the last `DATA` value; the result
is not defined.

## REAL floating point

`REAL` is optional. Declaring it on a target without floating point
support is a compile error. See [REAL functions](#real-functions) and
the target pages under [`targets/`](targets/).

## Assembly

### ASM

```basic
VALUE AS U8

PRINT "BEFORE"
ASM CPU 6502
    lda #$01
    sta {VALUE}
ENDASM
PRINT "AFTER"
```

In the assembly output, crustyBASIC expands `{...}` names and applies
any CPU filter.

A bare `ASM` block runs when the program reaches it, whether it is
inside a `PROC` or among statements outside a `PROC`.

A bare `ASM` block outside a `PROC` cannot be used with `PROC MAIN`. Put
it inside `MAIN`, or use `ASM MODULE` when the block should be present
but not run as a statement.

Use `ASM MODULE` for assembly routines, data, or storage that should be
present without running as a statement:

```basic
ASM MODULE
asm_helper:
    lda #$01
    rts
ENDASM

ASM MODULE
asm_table:
    BYTE $40, $80, $10
ENDASM

ASM MODULE
asm_buffer:
    .ds 16
ENDASM
```

A module can hold routines, tables, reserved memory, assembler
directives, or any mixture of them. It does not run by itself or change
where the program starts.

In your own include files, use `ASM MODULE` for blocks that should not
run as statements.

Filter by CPU (useful for more portable programs):

```basic
ASM CPU 6502
    lda #$01
ENDASM

ASM CPU 6809
    ldb #$01
ENDASM
```

A block for another CPU is left out. The target page lists the CPU name
to use.

The CPU filter follows the mode:

```basic
ASM MODULE CPU 6502
asm_table:
    BYTE $01
ENDASM
```

The header order is `ASM`, an optional mode (`MODULE` or `STARTUP`), an
optional `WHEN` name, and then an optional `CPU` filter. A form that uses
any of those additions must put its assembly and `ENDASM` on later
lines. Only bare assembly can use one line, such as
`ASM pla : pla ENDASM`.

`MODULE` and `STARTUP` can be used only outside a `PROC`. Use a bare
`ASM` block for assembly that runs inside a `PROC`.

### ASM MODULE WHEN

`ASM MODULE WHEN` makes an assembly block available when the program
uses it:

```basic
ASM MODULE WHEN asm_helper CPU 6502
asm_helper:
    lda #$01
    rts
ENDASM
```

The name after `WHEN` identifies the block. The block is added when the
program needs one of its labels.

`WHEN` is decided before the program runs. It does not test BASIC
variables. A label reference, including `ADDR label` or a reference from
another assembly block, makes the block needed.

The same `WHEN` name can be used on executable assembly inside a `PROC`:

```basic
PROC DRAW
    ASM WHEN asm_draw CPU 6502
        jsr asm_draw
    ENDASM
ENDPROC
```

Inside `ASM`, `{NAME}` becomes the assembler symbol for a variable.
Globals, `PROC` parameters, and local variables all work:

```basic
DIM SCORE AS U8
DIM BUF(16) AS U8

ASM
    lda {SCORE}
    sta {BUF}+1
    lda {BUF},y
ENDASM
```

`{NAME}` can also name an integer `CONST`. In that case, it becomes the
calculated integer value. `STRING` and `REAL` constants are not allowed.

Inside a typed `PROC`, `{RETURN}` names its return value. A final
unconditional `ASM` block that writes `{RETURN}` can provide the PROC's
return value.

### ASM STARTUP

`ASM STARTUP` replaces the target's normal startup with your own. You
must provide the load header if one is needed, set up the hardware, call
the crustyBASIC program, and decide what happens when it returns.

Inside `ASM STARTUP`, the placeholders below provide the selected
addresses and the name used to enter the crustyBASIC program. Other
`{...}` names can refer to a global `DIM` variable or integer `CONST`,
just as in a regular `ASM` block. Every custom startup must use
`{entry_label}`. See the selected target's page for a complete startup
example.

| Placeholder | Expands to |
| --- | --- |
| `{start_code}` | Code start as 4 digit hex (`START_CODE` or the target default). |
| `{start_code_decimal}` | The same address in decimal, e.g. for a `SYS` operand. |
| `{start_code_decimal_bytes}` | Decimal address digits as character byte values separated by commas, for start formats that use a BASIC `SYS` command. |
| `{start_program}` | Start of the loaded program as 4 digit hex (`START_PROGRAM`). |
| `{start_data}` | Start of writable data as 4 digit hex (`START_DATA`). |
| `{ram_top}` | Highest usable writable address as 4 digit hex. |
| `{ram_top_high}` | High byte of `{ram_top}` as 2 digit hex. |
| `{ram_size}` | RAM size as 4 digit hex. |
| `{ram_size_high}` | High byte of `{ram_size}` as 2 digit hex. |
| `{entry_label}` | Assembler name used to enter the crustyBASIC program. |

Addresses do not include a `$` prefix. Write `ORIGIN ${start_code}` to
produce `ORIGIN $1010`.

Rules:

- A program can use at most one `ASM STARTUP` block. Use
  `ASM STARTUP CPU <cpu>` when it is for only one CPU.
- `ASM STARTUP` cannot use `WHEN`.
- An `ASM STARTUP` block conflicts with `@OPTION STARTUP`; pick one.
- When setting `START_PROGRAM`, also set `START_CODE` so the custom
  startup has both addresses.
- Banked mappers use their own startups and do not support
  `ASM STARTUP`.

## Directives

Directives start with `@`. They set compile options, include files, or
choose which source is used.

### @OPTION

See [Compiler options](#compiler-options).

### @BANK / @ENDBANK

`@BANK N` starts a switched bank block, and `@ENDBANK` closes it.
Every `PROC`, `CONST` array, and top level `DATA` statement in the block
goes into bank `N`. Scalar `CONST` values are not placed in a bank.
Blank lines, comments, and `DATA` labels such as `FOO:` are allowed.

```basic
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

- Use `@BANK` only after selecting a banked mapper.
- Bank numbers start at `0`. The main area, which stays available, is
  not counted as a bank.
- `PROC MAIN` stays in the main area. Put shared helper PROCs there too.
- Code in the main area can call any bank.
- At the top level of `@BANK N ... @ENDBANK`, use only
  `CONST`, `PROC`, `DATA`, and `DATA` labels. Put `DIM`, inline `ASM`,
  and ordinary statements outside the bank block. `@INCLUDE`,
  `@INCLUDE_FILE`, `@INCLUDE_FONT`, `@INCLUDE_CHARSET`, `@INCLUDE_CHARMAP`,
  `@INCLUDE_TILESET`, `@INCLUDE_TILEMAP`, `@INCLUDE_SPRITE`, `@INCLUDE_TEXT`,
  and `@INCLUDE_IMAGE` work when their generated declarations follow the same
  rule. Their embedded declarations go in bank `N`.
- Do not nest `@BANK` blocks. Every `@ENDBANK` must match an earlier
  `@BANK`.
- Each call to a banked PROC starts `READ` at that bank's first `DATA`
  item. Use `RESTORE LABEL` when you need a specific item.
- A banked `CONST` array can be read only by code in the same bank.

The mapper decides which banks stay visible and what can cross a bank
boundary. This affects calls between banked PROCs, `ADDR(PROC)`, and how
`READ` and `RESTORE` work with banked `DATA`. See the matching
[target documentation](targets/) for those rules, valid bank numbers,
and bank sizes.

### @DEFINE and @UNDEF

```basic
@DEFINE DEBUG
@UNDEF DEBUG
```

`@DEFINE` sets a symbol and `@UNDEF` clears it. Test the symbol with
`@IF DEFINED(...)`.

### @IF, @ELIF, @ELSE, @ENDIF

```basic
@IF GRAPHICS_SUPPORTED THEN
    PRINT "graphics available"
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

`@IF` can check:

| Form                                                | Meaning                                                                                                |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `TARGET = name`                                     | Check the current target name.                                                                         |
| `SYSTEM = name`                                     | Check the current system name. Using a target name here is an error.                                   |
| `CPU = name`                                        | Check the current CPU name.                                                                            |
| target flag                                         | `TARGET_` followed by the target name; true when it matches.                                           |
| system flag                                         | `SYSTEM_` followed by the system name; true when it matches.                                           |
| CPU flag                                            | `CPU_` followed by the CPU name; true when it matches.                                                 |
| `CB_VERSION = "1.1.0"`                              | Compiler version string (matches `crustybasic --version`).                                             |
| `CB_VERSION_MAJOR`, `CB_VERSION_MINOR`, `CB_VERSION_PATCH` | Integer parts of the compiler version; use with `>=`, `<`, etc. for range checks.               |
| `DEFINED("symbol")`                                 | Symbol set with `@DEFINE`.                                                                             |
| `HAS_CHIP("name")`                                  | Target has the named chip.                                                                             |
| `OPTION("name")`                                    | Current value of any scalar source option.                                                             |
| `OPTION_SET("name")`                                | True when a source option was set. Use this for repeatable options.                                    |
| `JOY_PORTS`, `GRAPHICS_SUPPORTED`, `BITMAP_SUPPORTED`, `PLOT_SUPPORTED`, `SPRITE_KIND`, etc. | Target constants such as controller counts and supported APIs. |

Conditions can use `AND`, `OR`, `NOT`, parentheses, booleans, strings,
decimal integers, `$` hexadecimal integers, and the usual comparisons.
Repeatable options such as `MEMORY_REGION` do not have one value for
`OPTION()` to return. Test them with `OPTION_SET()`.

When the left side is a string, a simple name on the right does not need
quotes. This applies to target, system, CPU, and storage names. Quotes
are still allowed and are required for an empty string or a value that
is not a simple name, such as `TARGET <> ""`.

Common support checks include `TEXT_OUTPUT_SUPPORTED`,
`GRAPHICS_SUPPORTED`, `BITMAP_SUPPORTED`, `PLOT_SUPPORTED`,
`FRAME_SUPPORTED`, and `SPRITE_SUPPORTED`. `GRAPHICS_SUPPORTED` means
some graphics API is present. Check `PLOT_SUPPORTED` for `PLOT` and
`LINE`, and `BITMAP_SUPPORTED` when the program needs a `BITMAP_*` mode.

`CODE_STORAGE` is `ram` for writable program images and `rom` for
cartridge or ROM images. `TARGET` is the current target name. Use it
when several systems share the same target.

`START_CODE` and `RAM_TOP` bracket the memory a program has, so a
module that needs a certain amount of room can say so:

```basic
@REQUIRES RAM_TOP > $1F00 @ELSE "this needs more RAM than the machine has"
```

A quoted path requires and includes a file:

```basic
@REQUIRES "{program_name}/{target}.cbi"
```

Paths use the same lookup rules as `@INCLUDE`. If the file is missing,
`build-examples` skips that target; compiling directly reports an unmet
requirement. Errors in an existing file remain errors. An optional
`@ELSE "message"` supplies the missing file message.

### @WARN

```basic
@WARN "this feature will be removed"
```

Shows your warning and continues compiling.

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

`@INCLUDE` inserts a file at that point in the source. crustyBASIC first
looks beside the file containing the directive, then under `include/`
relative to the current working directory.

Add `ELSE` and a second path to make the first one optional:

```basic
@INCLUDE "graphics/{target}.cbi" ELSE "graphics/shared.cbi"
```

The fallback loads only when nothing the first path names is on disk, so
a target gets its own version of a routine just by adding a file, and
every other target keeps the shared one. If neither exists the error
names both.

All `@INCLUDE*` directive names and paths ignore letter case, so
`@include`, `@INCLUDE_Bin`, and `"Common/Utils.cbi"` all work. Two files
whose names differ only by case are an error, since there is no way to
tell which one you meant.

Every `@INCLUDE` path, including paths used by `@INCLUDE_*` directives,
and BASIC string literals can use the same placeholders. Their names
ignore letter case.

| Placeholder | Expands to |
| ----------- | ---------- |
| `{target}` | Target name. |
| `{family}` | Hardware family from the target manifest, or the target name when none is set |
| `{system}` | Full system name. |
| `{program_name}` | Program filename without extension. |
| `{media}` | Media variant, such as `cart`, when the selected system has one. |
| `{cpu}` | CPU name. |
| `{mapper}` | Selected mapper name when one is active. |
| `{version}` | crustyBASIC version string; always available. |

Use a family include as a fallback:

```basic
@INCLUDE "sprite/{target}.cbi" ELSE "sprite/{family}.cbi"
```

## Compiler options

Set options with `@OPTION` or `--set` on the command line. Some are
available in only one of those places. Run `crustybasic options` to see
the accepted forms and values.

```basic
@OPTION OUTPUT_TYPE cart
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
| `TARGET`                         | target name                             | Select the output target. It must come before source that depends on the target. A system name is not accepted here. |
| `SYSTEM`                         | system name                             | Select a specific system and its parent target. A target name is not accepted here. |
| `DIALECT`                        | dialect name                            | Apply that dialect's defaults to options you haven't set.                     |
| `ROM`                            | ROM profile name                        | Select a ROM profile for targets that have one.                               |
| `OUTPUT_TYPE`                    | output name                             | Select a build output such as `cart` or `disk`.                               |
| `THROTTLE`                       | `0..65535`; some targets allow more     | Slow the program by adding delays. `0` disables it.                           |
| `TILE_BACKEND`                   | `TILE_CELL`, `TILE_HARDWARE`, `TILE_BITMAP`, `TILE_KERNEL` | Choose how `TILE` draws.                                        |
| `MAPPER`                         | mapper name or `name(PROVIDER, bank)`   | Choose a mapper using defaults or an explicit provider and physical bank.     |
| `NUMERIC_MODE`                   | see below                               | Choose the default type for numeric names.                                    |
| `TYPE_POLICY`                    | policy name                             | Choose the meanings of type suffixes and names without an explicit type.      |
| `MATH_REAL`                      | `AUTO`, `TARGET`, `BUILTIN`             | Choose where REAL math comes from.                                            |
| `BUILTIN_REAL`                   | `AUTO`, `Q16_16`, `Q24_8`, `BINARY64`   | Choose crustyBASIC's REAL format.                                             |
| `MATH_INTEGER`                   | `AUTO`, `TARGET`, `BUILTIN`             | Choose where integer math comes from.                                         |
| `START_PROGRAM`                  | address                                 | Set the address where the loadable program begins.                            |
| `START_CODE`                     | address                                 | Set the address where program code begins.                                    |
| `START_DATA`                     | address                                 | Set the address where writable program data begins.                           |
| `STRING_DEFAULT_CAPACITY`        | `1..256`                                | Set the usable capacity for plain `STRING` declarations.                      |
| `STRING_BOUNDS_CHECKS`           | `TRUE`, `FALSE`                         | Check whole-string assignments and appends while the program runs. Default is `FALSE`. |
| `HEAP_CHECKS`                    | `TRUE`, `FALSE`                         | Report heap exhaustion and bad releases while the program runs. Default is `FALSE`. |
| `STARTUP`                        | startup name                            | Select a startup provided by the target. It may also set `START_PROGRAM` and `START_CODE`. |
| `MEMORY_REGION`                  | target region name                      | Add a named target memory region. Repeating the option adds more regions.     |
| `MEMORY_ACTION`                  | target action name                      | Enable a setup action provided by the target.                                 |
| `REGION_NTSC`, `REGION_PAL`      | flag                                    | Pick timing/frequency tables. Default is NTSC.                                |
| `INLINE_ASM`                     | `TRUE`, `FALSE`                         | Allow `ASM ... ENDASM`.                                                       |
| `ARRAY_BASE`                     | `0`, `1`                                | First array index.                                                            |
| `FOLD_LOWERCASE_TO_UPPERCASE`    | `TRUE`, `FALSE`                         | Use uppercase target characters for lowercase ASCII text.                     |
| `USE_LINE_NUMBERS`               | `TRUE`, `FALSE`                         | Require line numbers. Complete listings are recognized automatically.         |

Some targets define more `@OPTION` names. Their target pages list them.

A command line `--set` value takes priority over the same `@OPTION` in
the source. For example, `--set throttle=0` overrides
`@OPTION THROTTLE 10`.

### Numeric and math options

The selected dialect normally chooses `NUMERIC_MODE`. Set it yourself
only when you want different numeric behavior:

- `INTEGER` - unsuffixed numeric names are integer.
- `REAL` - unsuffixed numeric names are `REAL`.
- `REAL_NARROW` - unsuffixed numeric names behave as `REAL`, but names
  used only with whole numbers can use faster, smaller integer math.
- `INTEGER_ONLY_WARN` - integer names, with warnings for `REAL` syntax.
- `INTEGER_ONLY_ERROR` - integer names, with errors for `REAL` syntax.

`MATH_REAL` and `MATH_INTEGER` choose how operations are handled. They
do not change a value's type. `NUMERIC_MODE`, suffixes, and `AS`
declarations do that.

| Value     | Meaning |
| --------- | ------- |
| `AUTO`    | Let the target choose. This is the default when the option is not set. |
| `TARGET`  | Use the target's own math. Error if unavailable. |
| `BUILTIN` | Use crustyBASIC's own math. Error if unavailable. |

`BUILTIN_REAL` chooses the `REAL` format supplied by crustyBASIC.
`AUTO` uses the format preferred by the target. `Q16_16` keeps more
digits after the decimal point, while `Q24_8` allows larger whole
numbers. Both use four bytes.

`BINARY64` uses eight bytes. It keeps about 15 significant digits and
has a range of roughly 1e-308 to 1e308. It also makes `EXP` available.
It is much slower and twice the size of the four byte formats, and some
targets do not support it. Very small results become zero, and results
outside its range stop at the largest value it can hold. It does not
produce infinity or NaN values.

`TARGET` and `BUILTIN` are explicit choices. Neither
`--set optimize=speed` nor `--set optimize=size` changes them.

`AUTO` may choose differently for `--set optimize=speed` and
`--set optimize=size` when the target offers faster and smaller
versions. Otherwise it uses the target's normal choice.

Integer math uses `MATH_INTEGER`. `REAL` math uses `MATH_REAL`. A mixed
program can use both.

### Memory regions

Memory regions describe extra areas of memory that a target can use.
Each one lists its addresses, what it can hold, how it is prepared, and
any limits. A region may require an action during startup before the
program can use it.

Run `crustybasic target-info <system>` to see the regions for one
system. It shows each region's addresses, allowed uses, loading method,
whether it is enabled automatically, required actions, and limits.

The loading method is `direct` when bytes are loaded where the program
will use them. It is `relocate` when the startup copies initial values
into the region, and `runtime` when the region starts empty. Required
actions run automatically. A feature listed as disabled cannot use that
region.

Enabling a region makes it available for the uses listed by
`target-info`. It does not let you place a particular variable or
`PROC` there yourself.

Enable a region in source with:

```basic
@OPTION MEMORY_REGION name
```

Or enable it in the program's config file:

```toml
memory-regions = ["name"]
```

Repeat `@OPTION MEMORY_REGION`, or add more names to the config array,
to enable several regions. A region marked as automatic needs no option
or config entry.

## Dialects

Dialects help compile programs written for another BASIC. Choose one
with `@OPTION DIALECT`, `--set dialect=...`, or
`crustybasic.config.toml`. A dialect supplies suitable defaults. Any
target, system, ROM, or other option you choose takes priority.

The dialect controls how the source is read, while the target controls
where the finished program runs. You can select them separately. If you
do not select a target, the dialect uses its usual one.

A dialect can be paired with another target when that target provides
everything the program needs. Hardware addresses used by `PEEK`,
`POKE`, and `CALL` are not translated, so check them before building for
a different machine.

Dialects follow the naming habits of the source BASIC. For example,
Applesoft, Atari BASIC, Commodore BASIC, and Color Basic treat bare
numeric names as `REAL`, `%` names as 16 bit integers, and `$` names as
strings. Apple Integer BASIC treats bare numeric names as 16 bit
integers and `$` names as strings.

Dialects whose normal type is `REAL` use `REAL_NARROW` automatically.
Bare numeric names still behave as `REAL`, while names and arrays used
only with whole numbers can use faster, smaller integer math. Printed
values, fractions, explicit `REAL` declarations, and `REAL` math remain
`REAL`.

Available dialects:

| Dialect | Use for |
| --- | --- |
| `crustybasic` | crustyBASIC source. This is the default. |
| `applesoft_basic` | Applesoft BASIC listings for Apple II. |
| `integer_basic` | Apple Integer BASIC listings for Apple II. |
| `atari_basic` | Atari BASIC listings for Atari 8 bit systems. |
| `basic_xl` | BASIC XL listings for Atari 8 bit systems. |
| `basic65` | BASIC 65 listings for Mega65. |
| `cbm_basic_vic20` | Commodore BASIC V2 listings for VIC-20. |
| `cbm_basic_v2` | Commodore BASIC V2 listings for C64. |
| `cbm_basic_v3_5` | Commodore BASIC 3.5 listings for Plus/4. |
| `cbm_basic_v7_0` | Commodore BASIC 7.0 listings for C128. |
| `pet_basic_v2` | Commodore BASIC V2 listings for PET. |
| `pet_basic_v4` | Commodore BASIC V4 listings for PET. |
| `x16_basic` | Commander X16 BASIC listings. |
| `color_basic` | TRS-80 Color BASIC listings for CoCo. |
| `extended_color_basic` | Extended Color BASIC listings for CoCo. |
| `locomotive_basic` | Locomotive BASIC listings for Amstrad CPC. |
| `sinclair_basic` | Sinclair BASIC listings for ZX Spectrum. |
| `zx81_basic` | Sinclair BASIC listings for ZX81. |
| `ti_basic` | TI BASIC listings for TI-99/4A. |
| `ti_extended_basic` | TI Extended BASIC listings for TI-99/4A. |
| `trs80_level2_basic` | Level II BASIC listings for TRS-80 Model I. |
| `trs80_model2_basic` | Model II BASIC listings for TRS-80 Model II. |
| `trs80_model3_basic` | Model III BASIC listings for TRS-80 Model III. |
| `trs80_model4_disk_basic` | Model 4 Disk BASIC listings. |
| `qbasic` | QBASIC style listings. Select a target or system explicitly. |
| `gwbasic` | GW-BASIC listings for 16 bit DOS. |
| `pcjr_basic` | GW-BASIC listings for IBM PCjr. |
| `tandy1000_basic` | GW-BASIC listings for Tandy 1000. |
| `gfa_basic` | GFA BASIC 3.x listings for the Atari ST. |

Many dialects support only part of the original BASIC. See the target
page for current limits.

## Source rewrites

`@REWRITE` replaces one spelling with another before the program is
checked.

`@REWRITE_EXPR OLD = NEW` replaces part of an expression inside a
statement. Names in braces capture values from the old spelling and
insert them into the new one:

```basic
@REWRITE_EXPR DOUBLE {VALUE} = ({VALUE}) * 2
```

A rewrite for a call written without parentheses also matches the same
call with parentheses. The example above matches both `DOUBLE X` and
`DOUBLE(X)`.

When the replacement needs more than one line, use `@REWRITE_BLOCK`.
Everything up to `@END_REWRITE` becomes the replacement, so it can hold
`IF`, `WHILE`, and `DO` blocks, which need a line of their own:

```basic
@REWRITE_BLOCK SHOW_FRAME
	SPRITES_FLUSH
	GFX_SWAP
@END_REWRITE
```

An empty replacement removes the statement, which costs nothing:

```basic
@REWRITE_BLOCK SHOW_FRAME
@END_REWRITE
```

A variable the block declares belongs to that use of the block, the
same way a variable declared inside a `PROC` belongs to that `PROC`:

```basic
@REWRITE_BLOCK FADE_BORDER {n}
	I AS U8

	FOR I = 0 TO {n}
		POKEB $D020, I
	NEXT
@END_REWRITE
```

`I` becomes `FADE_BORDER_I_1` where the block is first used and
`FADE_BORDER_I_2` where it is used next, so it never clashes with an `I`
the caller already has, or with the other use. Labels work the same way.

To declare something the caller can use afterwards, take the name as a
value instead:

```basic
@REWRITE_BLOCK START_SCORE {name}
	{name} AS U16
	{name} = 0
@END_REWRITE

START_SCORE PLAYER_SCORE
```

The first matching rule wins, so a file loaded earlier can replace a
rule that a later file supplies as the default.

To add your own syntax, put `@REWRITE OLD = NEW` lines in a `.cbi` file.
List that file under `include` in `crustybasic.config.toml`, or load it
from the source with `@INCLUDE`.

Some targets add their own words. Those words are listed on the target's
page and work only when that target is selected.

## Mappers

A mapper controls how program code and data are arranged. Cartridge
mappers can divide ROM into switched banks. Other mappers can load
overlays or switch RAM while using a normal disk or other output.

The output type chooses the container. The mapper chooses the layout and
bank switching method. Select a mapper and use its target defaults with:

```basic
@OPTION MAPPER name
```

Some mappers accept an explicit storage provider and physical bank:

```basic
@OPTION MAPPER name(PROVIDER, 0)
```

The plain form uses that mapper's default provider. The explicit form is
accepted only when the selected mapper lists that provider. It does not
change any `EXTMEM` handle's bank numbering.

Mapper names, defaults, bank sizes, valid bank numbers, providers, and
supported output types depend on the target. So do the rules for calls and
`DATA` that cross banks. Run `crustybasic options` for the available names
and `crustybasic target-info <system>` for the selected system's mapper
details. The matching
[target documentation](targets/) explains how to use each mapper.
