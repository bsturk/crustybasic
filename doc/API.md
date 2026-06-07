# CrustyBASIC API Reference

This page covers crustyBASIC runtime calls, portable helper libraries,
hardware access, memory operations, and per-target API capabilities.
Core language syntax is documented in [LANGUAGE.md](LANGUAGE.md), and
command-line use is documented in [USAGE.md](USAGE.md).

## Portable API

Core calls work on all targets unless noted. Optional helper libraries
list their own limits.

### Text

| Call                               | What it does                                                     |
| ---------------------------------- | ---------------------------------------------------------------- |
| `CELL_CLS()`                       | Clear text screen and home cursor. (CLS() is an alias of this)   |
| `CLEAR_LINE()` / `CLEAR_LINE(row)` | Blank current or zero-based text row and leave cursor at start.  |
| `HIDE_CURSOR()`                    | Hide the text cursor when the target has a visible one.          |
| `SHOW_CURSOR()`                    | Show the text cursor when the target has a visible one.          |
| `POSITION(col, row)`               | Move text cursor to zero-based col/row.                          |
| `CURSORCOL()`                      | Current text cursor column (returns `U8`).                       |
| `CURSORROW()`                      | Current text cursor row (returns `U8`).                          |

### Sound

| Call                 | What it does                                                     |
| -------------------- | ---------------------------------------------------------------- |
| `BEEP(duration)`     | Play a target specific tone. Duration units are target specific. |

### Timing

| Name              | What it does                                                    |
| ----------------- | --------------------------------------------------------------- |
| `TIMER_AVAILABLE` | `1` when the target has a usable timer, otherwise `0`.          |
| `TIMER_HIGH_RES`  | `1` for sub-frame raw ticks, `0` for coarse target ticks.       |
| `TIMER_START()`   | Start or reset the elapsed timer.                               |
| `TIMER_STOP()`    | Stop the timer and store elapsed ticks in `TIMER_HI:TIMER_LO`.  |
| `TIMER_HI`        | High 16 bits of the elapsed tick count.                         |
| `TIMER_LO`        | Low 16 bits of the elapsed tick count.                          |
| `DELAY(n)`        | Pause for roughly target specific units.                        |

Raw tick units are target specific. Compare `TIMER_HI` first, then
`TIMER_LO`; lower elapsed ticks means faster. Use
`@IF TIMER_AVAILABLE THEN` when a program needs a timer fallback.

### Files

The portable file API uses logical channels and byte streams. A target
provider maps those channels to its native file system, ROM calls, host
handles, or other I/O layer.

Current target providers include Atari 800 CIO, CBM KERNAL disk I/O,
Apple II DOS 3.3 text files, and CoCo Disk BASIC sequential files.
Targets without a provider keep the default compile time `@ERROR`.

| Name                    | Type | What it means                                  |
| ----------------------- | ---- | ---------------------------------------------- |
| `FILE_MODE_READ`        | `U8` | Open for reading.                              |
| `FILE_MODE_WRITE`       | `U8` | Open for writing.                              |
| `FILE_MODE_APPEND`      | `U8` | Open for appending.                            |
| `FILE_MODE_UPDATE`      | `U8` | Open for reading and writing.                  |
| `FILE_MODE_DIR`         | `U8` | Open a directory style listing when supported. |
| `FILE_OK`               | `U8` | No error.                                      |
| `FILE_EOF`              | `U8` | End of file.                                   |
| `FILE_UNSUPPORTED`      | `U8` | Operation is not supported.                    |

| Call                                        | What it does                                      |
| ------------------------------------------- | ------------------------------------------------- |
| `FILE_OPEN(channel, path$, mode)`           | Open a path on a logical channel.                 |
| `FILE_OPEN_NATIVE(channel, path$, a1, a2)`  | Open with target native aux/device values.        |
| `FILE_CLOSE(channel)`                       | Close a logical channel.                          |
| `FILE_READ_BYTE(channel)`                   | Read one byte and return `U8`.                    |
| `FILE_WRITE_BYTE(channel, byte)`            | Write one byte.                                   |
| `FILE_READ_LINE(channel, out line$)`        | Read one text record into a string.               |
| `FILE_WRITE_STR(channel, text$)`            | Write a string without adding a newline.          |
| `FILE_STATUS(channel)`                      | Return the target status byte for the channel.    |
| `FILE_COMMAND(cmd, channel, a1, a2, path$)` | Run a target native file command.                 |

Program file commands are used by dialects and can also be used directly.

| Call                         | What it is for                           |
| ---------------------------- | ---------------------------------------- |
| `FILE_SAVE_PROGRAM(path$)`   | Dialect `SAVE` style command.            |
| `FILE_LOAD_PROGRAM(path$)`   | Dialect `LOAD` style command.            |
| `FILE_LIST_PROGRAM(path$)`   | Dialect `LIST path` style command.       |
| `FILE_ENTER_PROGRAM(path$)`  | Dialect `ENTER` style command.           |
| `FILE_RUN_PROGRAM(path$)`    | Dialect `RUN "path"` style command.      |

`FILE_OPEN_NATIVE` arguments are target specific. See the target docs
when a program needs explicit device, secondary address, drive, or ROM
mode values.

### Raster interrupts

Some targets provide a portable raster-interrupt API.

- `RASTER_CLEAR()` - clear the list of marked display rows.
- `RASTER_MARK_ROW(row)` - mark one visible display row.
- `RASTER_MARK_ROWS(first, count, step)` - mark several visible rows.
- `RASTER_MARK_OFFSET(offset)` - mark a raw display-list byte offset.
- `RASTER_INSTALL(frame_addr, line_addr)` - install no-argument PROC
  addresses for per-frame and per-line work. Pass `0` for either hook
  when you do not need it.
- `RASTER_OFF()` - disable raster interrupts and clear row marks.
- `RASTER_WSYNC()` - wait for the current scanline boundary.
- `RASTER_INDEX` - `U8` line counter reset before the frame hook and
  incremented after each line hook.

```basic
@OPTION TARGET atari800

COLORS(3) AS U8

@ASYNC
PROC BARS_FRAME
    COLORS(0) = COLORS(0) + 1
ENDPROC

@ASYNC
PROC BARS_LINE
    RASTER_WSYNC
    GTIA.COLBK = COLORS(RASTER_INDEX)
ENDPROC

PROC MAIN
    RASTER_CLEAR
    RASTER_MARK_ROWS 32, 4, 8
    RASTER_INSTALL ADDR(BARS_FRAME), ADDR(BARS_LINE)
    RASTER_OFF
ENDPROC
```

The installed PROCs must be no-argument `@ASYNC` PROCs.

### Graphics

To use the portable graphics API, add `@INCLUDE "graphics.cbi"`.
Available graphics modes vary by target; the target docs under
[`targets/`](targets/) list the details.

| Call                                              | What it does                                  |
| ------------------------------------------------- | --------------------------------------------- |
| `DISPLAY()`                                       | Switch to the target's default drawable display mode. Returns `1` if accepted, `0` if unavailable. |
| `DISPLAY(mode)`                                   | Switch to a specific display mode. Returns `1` if accepted, `0` if unsupported. |
| `DISPLAY_COLOR_COUNT(mode)`                       | Portable color-value count for the mode, or `0` when not drawable. |
| `GFX_COLOR(fg, bg)`                               | Set foreground/background.                    |
| `GFX_HAS_MODE(mode)`                              | `1` if the active target supports the mode.   |
| `GFX_BITMAP_BASE()`                               | Bitmap memory base, or `0` when unavailable.  |
| `GFX_ROW_BYTES()`                                 | Bytes per bitmap row, or `0` when unavailable.|
| `GFX_CLS()`                                       | Clear the active graphics surface. (GCLS() is an alias of this) |
| `GFX_CAN_PLOT(mode)`                              | `1` if `PLOT`/`LINE`-style drawing is supported in the mode. |
| `GFX_CAN_POINT(mode)`                             | `1` if `POINT` readback is supported in the mode. |
| `PLOT(x, y)` / `UNPLOT(x, y)`                     | Set/clear a point.                            |
| `POINT(x, y)`                                     | Read a point. `0` means background/off; nonzero values are mode-specific. |
| `HLINE(x0, x1, y)` / `UNHLINE(x0, x1, y)`         | Horizontal span. Endpoints may be in either order. |
| `LINE(x0, y0, x1, y1)`                            | Arbitrary line.                               |
| `BOX(x0, y0, x1, y1)` / `FILLBOX(x0, y0, x1, y1)` | Outline / filled rectangle.                   |
| `CIRCLE(xc, yc, r)` / `CIRCLE_E(xc, yc, rx, ry)`  | Circle / ellipse.                             |
| `PLAYFIELD_SCROLL_Y(pixels)`                      | Set vertical fine scroll; no-op if unsupported. |

Coordinates are in the current graphic mode's space (pixels in bitmap modes, hardware cells in low-res).

Active mode globals:

| Name                                      | What it means                                                |
| ----------------------------------------- | ------------------------------------------------------------ |
| `GFX_MODE`                                | Last mode accepted by `DISPLAY`.                             |
| `GFX_WIDTH` / `GFX_HEIGHT`                | Size of the active drawing space.                            |
| `GFX_BYTE_W` / `GFX_BYTE_H`               | Pixels per bitmap byte, or `0` outside bitmap modes.         |
| `GFX_COLOR_ATTR_W` / `GFX_COLOR_ATTR_H`   | Color attribute block size in pixels. Usually `1`.           |

Modes: `CELL`, `CELL_EXTENDED`,
`BITMAP_HIRES`, `BITMAP_LORES`, and
`BITMAP_MULTICOLOR`. `COLOR_MODE` is the target's best
mode for drawings that keep earlier `GFX_COLOR` colors.

Portable color constants:

`BLACK` `WHITE` `RED` `CYAN`
`PURPLE` `GREEN` `BLUE`
`YELLOW` `ORANGE` `BROWN`
`LIGHT_RED` `DARK_GRAY` `GRAY`
`LIGHT_GREEN` `LIGHT_BLUE` `LIGHT_GRAY`

`GFX_NATIVE_COLOR(idx)` returns the target specific byte for an index
in the active mode.

### Cell

The cell API writes raw text/screen cells. It is useful for tile
games and custom character sets.

| Call                                           | What it does                                  |
| ---------------------------------------------- | --------------------------------------------- |
| `CELL_DRAW(x, y, w, h, addr)`                  | Copy a packed block of cells to the screen.   |
| `CELL_GETC(x, y)`                              | Read one cell when the target can.            |
| `CELL_PUTC(x, y, c)`                           | Write one cell.                               |
| `CELL_CODE(c)`                                 | Convert a printable byte to a raw cell code.  |
| `CELL_COLOR(x, y, color)`                      | Set cell color where the target supports it.  |
| `CELL_ATTRIB(x, y, attr)`                      | Set cell attributes where supported.          |
| `CELL_FLUSH()`                                 | Push staged writes; no-op on direct targets.  |

Cell values are target specific screen codes. Color granularity is also target specific.
Use `CELL_CODE(ASC("O"))` when a program wants the target's raw cell
code for a printable character, such as for `CELL_PUTC` or a native
`TILE_ALIAS`.
Current attributes are `NORMAL`, `INVERSE`, `ITALIC`, and `BLINKING`.

### Tile

To use the portable tile API, add `@INCLUDE "tile.cbi"`. TILE draws a
playfield on whatever the target supports: text cells, graphics, or a
tile display. Coordinates are tile positions, not pixels.

```basic
@INCLUDE "tile.cbi"

TILE_BEGIN TILE_AUTO, 2, 2
TILE_ALIAS 0, 32
TILE_ALIAS 1, 35
TILE_CLEAR_TILE 0
TILE_BOX 0, 0, TILE_COLUMNS - 1, TILE_ROWS - 1, 1
```

Most programs should start with `TILE_BEGIN TILE_AUTO, cell_w, cell_h`.
`cell_w` and `cell_h` are the size of one tile in normal text cells.
Use `1, 1` for single-character tiles, or larger values for block tiles.

`TILE_BEGIN(mode, cell_w, cell_h, base)` is for targets that need a
character or tile memory address. Omit `base` unless the target docs ask
for it. `TILE_MODE(mode)` changes the drawing method later when the
target supports it.

Tile modes:

| Name          | Meaning                                      |
| ------------- | -------------------------------------------- |
| `TILE_AUTO`   | Target chooses the drawing method. Start here. |
| `TILE_NATIVE` | Draw on the target text/tile screen.         |
| `TILE_BITMAP` | Draw tiles on a graphics bitmap.             |
| `TILE_KERNEL` | Use a target-specific tile display.          |

`@OPTION TILE_BACKEND native` sets what `TILE_AUTO` means for this
build. Values are `auto`, `native`, `bitmap`, and `kernel`. The
command-line form is `--set tile-backend=...`. Use these only to force
or compare drawing methods; unsupported choices are compile errors.

Tile drawing:

| Call                                  | What it does                                |
| ------------------------------------- | ------------------------------------------- |
| `TILE_PLOT(x, y, tile)`               | Draw one logical tile.                      |
| `TILE_CLEAR(x, y)`                    | Draw the selected clear tile.               |
| `TILE_HLINE(x0, x1, y, tile)`         | Draw a horizontal tile run.                 |
| `TILE_UNHLINE(x0, x1, y)`             | Clear a horizontal tile run.                |
| `TILE_VLINE(x, y0, y1, tile)`         | Draw a vertical tile run.                   |
| `TILE_UNVLINE(x, y0, y1)`             | Clear a vertical tile run.                  |
| `TILE_BOX(x0, y0, x1, y1, tile)`      | Draw a tile rectangle outline.              |
| `TILE_FILLBOX(x0, y0, x1, y1, tile)`  | Fill a tile rectangle.                      |
| `TILE_CLS()`                          | Clear the TILE surface.                     |

Tile timing:

| Call                                  | What it does                                |
| ------------------------------------- | ------------------------------------------- |
| `WAIT_TILE()`                         | Wait for the target default tile interval.  |
| `WAIT_TILE(count)`                    | Wait for a target-specific tile interval.   |

Tile assets:

| Call                                  | What it does                                |
| ------------------------------------- | ------------------------------------------- |
| `TILE_ALIAS(id, native)`              | Map an abstract ID to a native cell code.   |
| `TILE_DEFINE(id, addr)`               | Define tile glyph data for capable targets. |
| `TILE_BLOCK_DEFINE(id, w, h, addr)`   | Name a row-major native-cell block.         |
| `TILE_BLIT(x, y, block)`              | Draw a named block.                         |
| `TILE_BLIT(x, y, w, h, addr)`         | Draw an ad hoc block.                       |
| `TILE_CLEAR_TILE(id)`                 | Select the clear tile ID.                   |
| `TILE_COLOR(id, fg, bg)`              | Set bitmap TILE colors when available.      |

Capability and geometry names:

| Name / call                    | What it means                                   |
| ------------------------------ | ----------------------------------------------- |
| `TILE_AVAILABLE`               | Tile drawing is available.                      |
| `TILE_NATIVE_AVAILABLE`        | Text/tile screen drawing is available.          |
| `TILE_BITMAP_AVAILABLE`        | Bitmap drawing mode exists.                     |
| `TILE_KERNEL_AVAILABLE`        | Target-specific tile display is available.      |
| `TILE_DEFINE_AVAILABLE`        | `TILE_DEFINE` can install tile data.            |
| `TILE_BLOCK_AVAILABLE`         | `TILE_BLOCK_DEFINE` and `TILE_BLIT` are useful. |
| `TILE_COLOR_AVAILABLE`         | `TILE_COLOR` is available.                      |
| `TILE_BACKEND()`               | Active drawing mode.                            |
| `TILE_CELL_W()` / `TILE_CELL_H()` | Tile size in screen cells.                   |
| `TILE_PIXEL_W()` / `TILE_PIXEL_H()` | Tile size in pixels.                       |
| `TILE_COLUMNS()` / `TILE_ROWS()` | Logical TILE surface size.                    |

Bitmap TILE uses `DISPLAY` and the portable graphics API internally and can draw glyphs supplied by
`TILE_DEFINE`; bitmap `TILE_BLIT` treats block bytes as tile IDs, so
defined glyphs and `TILE_COLOR` settings are honored. Kernel TILE
targets may expose only a small subset. Atari 2600 tile support maps
empty/non-empty tile IDs onto a 20-column TIA playfield row kernel and reports
`TILE_BLOCK_AVAILABLE = 0`, `TILE_COLOR_AVAILABLE = 0`.

### Input

| Call                     | What it does                                        |
| ------------------------ | --------------------------------------------------- |
| `KEY()`                  | Blocking key-code read. Returns `U8`.               |
| `INKEY()`                | Non-blocking key as a string, or `""`.              |
| `INKEY_CODE()`           | Non-blocking key code, or `0`.                      |
| `RAWKEY()`               | Non-blocking raw key as a string, or `""`.          |
| `RAWKEY_CODE()`          | Non-blocking raw key code, or `0`.                  |
| `HAS_KEYBOARD()`         | `1` when keyboard input is available.               |
| `HAS_JOYSTICK()`         | `1` when joystick input is available.               |
| `HAS_JOYSTICK_BUTTONS()` | `1` when joystick buttons are available.            |
| `HAS_ANALOG_JOYSTICK()`  | `1` for analog joysticks.                           |
| `HAS_DIGITAL_JOYSTICK()` | `1` for digital joysticks.                          |
| `HAS_PADDLES()`          | `1` when `PADDLE(axis)` has supported axes.         |
| `HAS_MOUSE()`            | `1` when mouse input is available.                  |
| `ANY_INPUT()`            | `1` when a key, joystick, or button is active.      |
| `WAIT_ANY_INPUT()`       | Blocks until `ANY_INPUT()` is true.                 |
| `JOY(port)`              | Direction bits from a zero-based joystick port.     |
| `JOY_BUTTON(port, button)`   | `1` if pressed, `0` otherwise.                      |
| `JOY_SET_DEADZONE(value)` | set analog joystick dead zone                     |
| `PADDLE(axis)`           | Raw analog axis value.                              |

Input constants:

| Name                    | What it means                                       |
| ----------------------- | --------------------------------------------------- |
| `JOY_PORTS`             | Number of joystick ports for `JOY(port)`.           |
| `JOY_BUTTONS`           | Buttons per joystick port for `JOY_BUTTON(port, button)`. |
| `JOY_DEADZONE_DEFAULT`  | initial analog joystick dead zone                   |
| `PADDLE_AXES`           | Number of analog axes for `PADDLE(axis)`.           |
| `ANALOG_AXIS_PAIR_NAME` | Name for an X/Y analog axis pair.                   |
| `MOUSE_BUTTONS`         | Number of mouse buttons; `0` when none.             |

Test joysticks against `JOY_UP`, `JOY_DOWN`, `JOY_LEFT`, `JOY_RIGHT` with bitwise `&`:

```basic
IF JOY(0) & JOY_LEFT <> 0 THEN ...
```

Missing joystick ports and buttons return 0. On analog joystick targets,
`JOY` still returns the shared direction bits.

analog targets start with `JOY_DEADZONE_DEFAULT`; larger
`JOY_SET_DEADZONE(n)` values ignore more center drift, smaller values
make the stick more sensitive, and digital targets accept the call as a no op

`MOUSE_BUTTONS` reports the button count, not the current button state.
Mouse position and mouse-button state are target-specific until a
portable mouse input surface exists.

`RAWKEY()` and `RAWKEY_CODE()` are for games and control polling. They
bypass text editor input when a target has a separate editor or console
input path. Use `INKEY()` and `INKEY_CODE()` for normal text keyboard
input.

`ANY_INPUT()` is for "press anything" prompts. On keyboard targets it
may consume the pending key - call `RAWKEY_CODE()` or `RAWKEY()` if you
need the raw key.

[Per-target capabilities](#input-capabilities) has the per target inputs chart.

### Game helpers

These helpers are for portable game code.

| Call              | What it does                              |
| ----------------- | ----------------------------------------- |
| `LFSR_SEED(seed)` | Seed the deterministic byte generator.    |
| `LFSR_NEXT()`     | Next pseudo-random byte.                  |
| `LFSR_BYTE()`     | Same as `LFSR_NEXT()`.                    |
| `LFSR_RANGE(hi)`  | Pseudo-random value through the upper bound. |

| Call                                                   | What it does                         |
| ------------------------------------------------------ | ------------------------------------ |
| `AABB_HIT(x1, y1, w1, h1, x2, y2, w2, h2)`             | `1` if two rectangles overlap.       |

| Name / call                      | What it does                         |
| -------------------------------- | ------------------------------------ |
| `SPRITE_AVAILABLE`               | Defined as `1` when the target exposes `SPRITE_*`. |
| `SPRITE_KIND`                    | `SPRITE_KIND_HW` or `SPRITE_KIND_POLYFILL`. |
| `SPRITE_SURFACE_KIND`            | `SPRITE_SURFACE_KIND_PIXEL`, `_CELL`, or `_KERNEL`. |
| `SPRITE_MAX_COUNT`               | Number of portable sprite slots.     |
| `SPRITE_WIDTH` / `SPRITE_HEIGHT` | Native sprite dimensions.            |
| `SPRITE_DATA_BYTES`              | Native data size used by `SPRITE_DATA`. |
| `SPRITE_X_GRANULARITY` / `SPRITE_Y_GRANULARITY` | Coordinate snap size. |
| `SPRITES_BEGIN()` / `SPRITES_RESET()` | Initialize or reset sprite state. |
| `SPRITE_DATA(id, addr)`          | Install target-native sprite data.   |
| `SPRITE_DATA_8X8(id, addr)`      | Install an 8-byte MSB-left 1bpp shape. |
| `SPRITE_ALIGN_X(x)` / `SPRITE_ALIGN_Y(y)` | Snap a coordinate to the target's sprite grid. |
| `SPRITE_AT(id, x, y)`            | Move a sprite.                       |
| `SPRITE_COLOR(id, color)`        | Set sprite color where supported.    |
| `SPRITE_SHOW(id)` / `SPRITE_HIDE(id)` | Show / hide a sprite.          |
| `SPRITES_FLUSH()`                | Commit staged sprite changes where needed. |

| Name / call                      | What it does                         |
| -------------------------------- | ------------------------------------ |
| `MISSILE_COUNT`                  | Number of portable missiles.         |
| `MISSILE_INIT()`                 | Initialize missile state.            |
| `MISSILE_AT(id, x, y)`           | Move a missile.                      |
| `MISSILE_COLOR(id, color)`       | Set missile color.                   |
| `MISSILE_SHOW(id)` / `MISSILE_HIDE(id)` | Show / hide a missile.       |
| `MISSILE_HIT_BG(id)`             | Hardware/background hit if available. |
| `MISSILE_HIT_SPRITE(id)`         | Sprite/player hit if available.      |
| `MISSILE_HIT_HW`                 | `1` when hit reads use hardware.     |

### Numeric helpers

| Function      | Returns             | Description                                   |
| ------------- | ------------------- | --------------------------------------------- |
| `RND()`       | `U8`                | Target specific pseudo-random byte.           |
| `RAND()`      | `REAL`              | Fraction from `0` up to, but not including, `1`. |
| `RAND(max)`   | same type as `max`  | Scaled random value. Integer `RAND(0)` returns `0`. |
| `MIN(a, b)`   | common numeric type | Smaller value.                                |
| `MAX(a, b)`   | common numeric type | Larger value.                                 |
| `DIG(n)`      | `U8`                | `1` for character codes `0` through `9`.      |
| `DIG(s$)`     | `U8`                | `1` for one-character strings `"0"` through `"9"`. |

`RND()` is the target's native byte-sized random source. It is the
cheap primitive for animation jitter, timers, and masks such as
`RND() & 7`. `RAND()` and `RAND(max)` are the portable scaled forms.
`RAND(max)` accepts `U8`, `U16`, or `REAL` maxima; for positive `max`,
the result is at least `0` and less than `max`. The `REAL` forms require
target `REAL` support. `RAND` is heavier than `RND()`, so use `RND()`
in tight per-frame loops.

### Bitwise

| Function     | Operator | Returns | Description                       |
| ------------ | -------- | ------- | --------------------------------- |
| `BAND(a, b)` | `A & B`  | `U16`   | Bitwise AND.                      |
| `BOR(a, b)`  | `A | B`  | `U16`   | Bitwise OR.                       |
| `BXOR(a, b)` | `A ^ B`  | `U16`   | Bitwise XOR.                      |
| `BNOT(x)`    | -        | `U16`   | Bitwise NOT (one's complement).   |
| `SHL(x, n)`  | `X << N` | `U16`   | Shift left. Operator form requires a constant integer literal shift count. |
| `SHR(x, n)`  | `X >> N` | `U16`   | Shift right. Operator form requires a constant integer literal shift count. |

### Logical

For boolean conditions (`IF`, `WHILE`, `UNTIL`). These are
operators, not PROCs, and return `0` (false) or `1` (true). Both
sides of `AND` / `OR` are always evaluated.  The operators do not
short-circuit.

| Operator                | Description                                |
| ----------------------- | ------------------------------------------ |
| `A AND B` or `A && B`   | `1` only if both sides are non-zero.       |
| `A OR B` or `A \|\| B`  | `1` if either side is non-zero.            |
| `NOT A` or `!A`         | `1` if the operand is zero, `0` otherwise. |

### REAL

The target has to support `REAL` to use these.

| Function    | Description                                                                |
| ----------- | -------------------------------------------------------------------------- |
| `POW(a, b)` | Exponentiation. Integer operands stay integer. For `REAL`, small non-negative integer constant exponents are optimized automatically; other exponents use the target's FP support. |
| `INT(x)`    | REAL to integer (truncate).                                                |
| `LOG(x)`    | Natural log.                                                               |
| `SGN(x)`    | Sign as a REAL.                                                            |
| `SIN(x)`    | Sine. Radians by default, degrees after `DEG()`.                           |
| `COS(x)`    | Cosine. Same convention.                                                   |
| `ATN(x)`    | Arctangent. Same convention.                                               |

`DEG()` and `RAD()` flip the process-wide trig angle mode. `RAD()`
is the default.

### Strings

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
| `INSTR(haystack, needle)` | `U16`    | 1-based position, 0 if absent, 1 for empty needle. |
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
assignment are documented in [LANGUAGE.md](LANGUAGE.md#strings).
[`BIND`](#bind) can point a string at caller-managed memory.

## REAL floating point

`REAL` is optional and target dependent.

```basic
X! = 3.14159
Y! = LOG(X!)
PRINT STR(Y!)
```

Declaring `REAL` on a target without FP is a compile error.

## Memory operations

### ADDR

`ADDR(name)`, `ADDR name`, and `&name` evaluate to a `U16` address.
They accept addressable storage, numeric array elements, string
literals, and no-argument `PROC`s in fixed memory. Use them to pass an
address to `MEMMOVE`, `MEMFILL`, `DPOKE`, callback installers such as
`ON_FRAME_INSTALL`, or inline `ASM`.

```basic
BUF(255) AS U8
WORDS(15) AS U16

PTR## = ADDR BUF
PTR## = ADDR(BUF)
PTR## = &BUF
PTR## = ADDR BUF[10]
PTR## = &BUF[10]
PTR## = ADDR WORDS[3]
PTR## = ADDR(WORDS(3))
PTR## = ADDR "LITERAL"

PROC FOO
    PRINT "IN PROC"
ENDPROC

PROC_PTR## = ADDR FOO
```

`ADDR(ARR(I))` returns the address of the indexed element, not the start of
the array. String array elements are not supported.

For a string variable, `ADDR(S$)` points at the length byte (or two
bytes if the string was declared with capacity over 255); the
actual characters start right after.

`ADDR(FOO)` on a `PROC` returns its entry address.
The `PROC` must take no parameters and must not live in a
switched bank.

`ADDR` can take the address of `DIM` storage, typed `CONST` arrays, and
eligible PROCs. It cannot take the address of a scalar `CONST` or a
`PROC` parameter.

### PEEK, POKE, DPEEK, DPOKE

```basic
B = PEEK($D020)
POKE $D020, 6
W = DPEEK $0400
DPOKE($0410, $ABCD)
```

- `PEEK` reads one byte from an address.
- `POKE` writes one byte to an address.
- `DPEEK` reads two bytes from an address and returns a 16-bit value: low byte first, high byte second.
- `DPOKE` writes a 16-bit value to an address: low byte first, high byte second.

### MEMMOVE and MEMFILL

```basic
SRC(15) AS U8
DST(15) AS U8

MEMFILL(ADDR(SRC), 16, 0)
MEMMOVE(ADDR(SRC), ADDR(DST), 16)
MEMMOVE(ADDR(DST), ADDR(DST(1)), 15)
```

`MEMMOVE(src, dst, count)` copies bytes from source to destination.
The source and destination ranges may overlap.

`MEMFILL(dst, count, value)` fills a range of bytes with one byte value.

### BIND

```basic
BIND STRING$ TO ADDR, COUNT

RAW(79) AS U8
VIEW$ AS STRING * 80

BIND VIEW$ TO ADDR(RAW), 80
VIEW$ = "HELLO"
MID(VIEW$, 10, 1) = CHR(255)
```

`BIND` redirects a string's characters to your own writable memory.
The string still cannot grow past its declared capacity. No hidden length
byte is placed before the bound address.

- The target must be a string variable.
- `ADDR` and `COUNT` are treated as 16-bit values.
- If `COUNT` is a fixed number, it cannot be larger than the string's declared size.
- Must point at writable RAM.
- Strings declared up to 255 bytes (`STRING * 255` or smaller) are portable.

### CALL

```basic
CALL $FCA8
```

`CALL` runs the machine-code routine at a fixed address. How you pass
values in, what registers it changes, and how it returns values depend
on the machine and the routine you are calling.

### USR

`USR(...)` is dialect specific - the calling convention follows the
host BASIC. See [`targets/`](targets/) for the exact form.

For CrustyBASIC code, use `CALL` for a simple fixed-address machine-code routine,
or inline `ASM` when you need to pass values or get a result.

### Hardware registers

Targets expose chip registers as names:

```basic
VIC.BORDER = 6
VIC.BACKGROUND = 0
```

The target docs under [`targets/`](targets/) list chip namespaces and
register details.

## Per-target capabilities

The tables below cover each target as a whole. Specific systems
within a target (`apple2.e`, `atari800.xl.cart`, ...) may differ in
small ways - see the target docs under [`targets/`](targets/) for
those details.

### Screen, REAL, and text encoding

| Target      | Text screen | `REAL`       | String encoding  |
| ----------- | ----------- | ------------- | ---------------- |
| `apple2`    | 40x24       | yes           | Uppercased ASCII |
| `atari800`  | 40x24       | yes           | ATASCII          |
| `atari2600` | (none)      | no            | ASCII            |
| `c64`       | 40x25       | yes           | PETSCII          |
| `plus4`     | 40x25       | yes           | PETSCII          |
| `coco`      | 32x16       | partial       | Color BASIC ROM  |
| `nes`       | 32x30       | no            | ASCII            |
| `vic20`     | 22x23       | yes           | PETSCII          |

`TEXT_OUTPUT_AVAILABLE` is the check for character/text output.
`TEXT_CHARMAP_AVAILABLE` is the check for direct text-screen charmap
support. `GRAPHICS_AVAILABLE`, `FRAME_AVAILABLE`, and `SPRITE_AVAILABLE` are
the coarse checks for those portable surfaces. `POSITION`, `CURSORCOL`,
and `CURSORROW` use the text screen dimensions; portable code can read
`TEXT_WIDTH` and `TEXT_HEIGHT`.

Declaring `REAL` on a target without FP is a compile error. Target
docs list system-specific limits and notes.

### Input capabilities

Use `HAS_KEYBOARD()`, `HAS_JOYSTICK()`, and the related `HAS_*` calls
when a program needs to adjust prompts or controls.

| Target      | Keyboard | Joysticks | Buttons | Stick type | Paddle axes | Mouse buttons |
| ----------- | -------- | --------: | ------: | ---------- | ----------: | ------------: |
| `apple2`    | yes      | 2         | 2       | analog     | 4           | 0, `apple2.c`: 1 |
| `atari800`  | yes      | 4         | 1       | digital    | 8           | 0             |
| `atari5200` | no       | 2         | 1       | analog     | 4           | 0             |
| `atari2600` | no       | 2         | 1       | digital    | 4           | 0             |
| `c64`       | yes      | 2         | 1       | digital    | 4           | 0             |
| `plus4`     | yes      | 2         | 1       | digital    | 0           | 0             |
| `coco`      | yes      | 2         | 1       | analog     | 4           | 0             |
| `nes`       | no       | 2         | 4       | digital    | 0           | 0             |
| `vic20`     | yes      | 2         | 1       | digital    | 2           | 0             |

For simple "press anything to continue" prompts, `ANY_INPUT()` is
usually enough. Paddle axes are the numbered axes accepted by
`PADDLE(axis)`. Mouse button counts are available through
`MOUSE_BUTTONS`.
