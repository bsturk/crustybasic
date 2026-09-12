# crustyBASIC API Reference

This doc is a reference for the calls and values available to a
crustyBASIC program. It covers text, graphics, sound, input, files,
hardware access, memory operations, and more.

Core language syntax and builtins are documented in
[LANGUAGE.md](LANGUAGE.md), and command line use is documented in
[USAGE.md](USAGE.md).

Most names stay the same from one target to another, but the available
features vary. Each section tells you how to check optional features.

Names ending in `_SUPPORTED` are compile-time constants. Use them with
`@IF` or `@REQUIRES` when a program needs a target feature. Names ending
in `_AVAILABLE()` are runtime probes for hardware or services that may not
be present or ready.

## Text

| Call                               | What it does                                                     |
| ---------------------------------- | ---------------------------------------------------------------- |
| `CLS()`                            | Clear the current display. In a text or cell mode this clears text output and homes the cursor; after `DISPLAY` selects a bitmap mode it clears that bitmap and its color data. |
| `CLEAR_LINE()` / `CLEAR_LINE(row)` | Blank the current row or a row numbered from zero, and leave the cursor at its start. |
| `CLEAR_CHARS(count)`               | Blank count cells starting at the cursor. The cursor does not move. |
| `CLEAR_AT(col, row, count)`        | Blank count cells starting at a column and row numbered from zero. The cursor does not move. |
| `CLEAR_EOL()`                      | Blank from the cursor to the end of its row. The cursor does not move. |
| `CLEAR_EOS()`                      | Blank from the cursor to the end of the screen. The cursor does not move. |
| `PRINT_AT(col, row, s$)`           | Move the cursor and print a string without adding a newline.     |
| `PRINT_CENTER_X(row, s$[, width])` | Print s$ centered on a text row; width defaults to `TEXT_COLUMNS`. |
| `POSITION(col, row)`               | Move the text cursor to a column and row numbered from zero.     |
| `CURSOR_HIDE()`                    | Hide the text cursor when the target has a visible one.          |
| `CURSOR_SHOW()`                    | Show the text cursor when the target has a visible one.          |
| `CURSOR_COL()`                     | Current text cursor column (returns `U8`).                       |
| `CURSOR_ROW()`                     | Current text cursor row (returns `U8`).                          |
| `TEXT_MODE(columns)`               | Select a text width. Returns `1` on success or `0` when unavailable. |

Everything here is text output, so it goes through the cursor and
follows the target's own text behavior, including scrolling. To place
characters on a screen you are drawing yourself, or in a graphics or
mixed mode, use [`CELL_PRINT`](#cell) instead, which writes the cell
grid directly.

`TEXT_OUTPUT_SUPPORTED` is `TRUE` when ordinary text output is available.
`TEXT_WIDTH` and `TEXT_HEIGHT` give the text grid the machine starts in.
Use them where the value has to be known while the program is built:
array sizes, `CONST` values, and `@IF` checks. They also stay the right
size for cell calls, which always work in whole characters.

`TEXT_MODE(columns)` requests a text width, such as 40 or 80 columns.
After a successful change, `TEXT_COLUMNS()` gives the active width.
`TEXT_COLUMNS_80` is `TRUE` when 80 column text is available.

For the size the display is using right now, read `DISPLAY_WIDTH` and
`DISPLAY_HEIGHT`, which `DISPLAY` updates on every mode change. These are
pixels in a bitmap mode and characters in a cell mode, so use them for
drawing and `TEXT_WIDTH` / `TEXT_HEIGHT` for the cell grid.

## Display

Each target supports its own set of display modes. See the target pages
under [`targets/`](targets/) for the full list.

Along with target specific modes, `DISPLAY(mode)` accepts `CELL`,
`CELL_MULTICOLOR`, `BITMAP_HIRES`, `BITMAP_LORES`, and
`BITMAP_MULTICOLOR`. After a successful change, `DISPLAY_WIDTH`,
`DISPLAY_HEIGHT`, `GFX_BPP`, `GFX_COLOR_COUNT`, `PLOT_COLOR_COUNT`, and
`GFX_ROW_BYTES` describe the new mode.

Use `DISPLAY(mode)` before drawing. Later graphics calls then use the
right size, coordinates, and colors for that mode.

`BITMAP_HIRES_PIXEL_COLORS` gives the guaranteed number of `COLOR` values
per independent pixel in `BITMAP_HIRES`, or `0` when no guarantee is available.
Use `@REQUIRES BITMAP_HIRES_PIXEL_COLORS >= 6` to skip builds that cannot
guarantee six independent pixel colors.

| Call                                             | What it does                                  |
| ------------------------------------------------ | --------------------------------------------- |
| `DISPLAY()`                                      | Use the target's default drawing mode. Returns `1` on success or `0` when unavailable. |
| `DISPLAY(mode)`                                  | Use a specific display mode. Returns `1` on success or `0` when unsupported. |
| `DISPLAY_MIXED(mode, text_first_row, text_rows)` | Use graphics with one full width text region. Returns `1` on success or `0` when unsupported. |
| `DISPLAY_HAS_MODE(mode)`                         | `1` if the target supports the mode.          |
| `DISPLAY_MODE_COLOR_COUNT(mode)`                 | Number of color values the mode can draw, or `0` when it cannot draw graphics. |
| `GFX_CAN_PLOT(mode)`                             | `1` if calls such as `PLOT` and `LINE` work in the mode. |
| `GFX_CAN_POINT(mode)`                            | `1` if `POINT` readback is supported in the mode. |
| `GFX_PAGE_COUNT()`                               | Number of selectable display pages. Usually `1`. |
| `GFX_PAGE_SELECT(page)`                          | Select a display page. Returns `1` on success. |
| `GFX_PAGE_BASE(page)`                            | Primary memory base for a display page, or `0` when unavailable. |
| `GFX_PAGE_ATTR_BASE(page)`                       | Attribute memory base for a display page, or `0` when unavailable. |

### Mixed mode

`DISPLAY_MIXED` uses the same modes as `DISPLAY`. `text_first_row` and
cell calls use absolute rows in the full text grid, so row `0` is the top
and `TEXT_HEIGHT - 1` is the bottom. The bitmap keeps its normal graphics
coordinates, but pixels covered by the text rows are not visible.
`DISPLAY_WIDTH` and `DISPLAY_HEIGHT` are that whole bitmap, text rows
included, so pixels per text row is `DISPLAY_HEIGHT / TEXT_HEIGHT`. The
text rows themselves are `TEXT_WIDTH` wide. Passing `0` for `text_rows`
is the same as `DISPLAY(mode)`. A later `DISPLAY` call leaves mixed mode.
`DISPLAY_MIXED_SUPPORTED` is `TRUE` when `DISPLAY_MIXED` is supported.

`DISPLAY_MIXED_TEXT_ROWS` is a text row count the target guarantees.
`DISPLAY_MIXED_FIRST_ROW_MIN` and `DISPLAY_MIXED_FIRST_ROW_MAX` give the
supported placement range for that row count. When the minimum and maximum
are equal, the target has one fixed mixed layout. Other row counts and
positions may work; check the return value from `DISPLAY_MIXED`.

The text rows cover the same part of the bitmap. Clear and draw the bitmap
before writing those text rows. Use row based cell calls only within the
active text rows. `CELL_CLS` clears only the selected text rows and leaves
the rest of the bitmap unchanged. Mixed mode is graphics first, so `COLOR`
and `CLS` still affect the bitmap. Use `CELL_COLORS`, `CELL_COLOR`,
`CELL_PRINT`, and `CELL_CLS` for the text region.

### Display pages

Do not pass `GFX_PAGE_BASE` or `GFX_PAGE_ATTR_BASE`
to `PEEK` or `POKE` unless the target page says that is supported. Use
`PLOT`, `POINT`, `CLS`, and the target's graphics calls for portable
code.

### Current mode values

| Name                                      | What it means                                                |
| ----------------------------------------- | ------------------------------------------------------------ |
| `DISPLAY_MODE`                            | Last mode set by `DISPLAY`.                                  |
| `DISPLAY_WIDTH` / `DISPLAY_HEIGHT`        | Size of the current display. Pixels in a bitmap mode, columns and rows in a cell mode. |
| `GFX_ROW_BYTES()`                         | Bytes used by one row of bitmap data, or `0` when unavailable. |
| `GFX_BYTE_W` / `GFX_BYTE_H`               | Pixels per bitmap byte, or `0` outside bitmap modes.         |
| `GFX_BPP`                                 | Bits used for each pixel in bitmap data, or `0` outside bitmap modes. |
| `GFX_COLOR_COUNT`                         | Number of color values in the current mode.                  |
| `PLOT_COLOR_COUNT`                        | Number of colors selectable by `PLOT(x, y, color)`, or `0` when its color is ignored. |
| `GFX_COLOR_ATTR_W` / `GFX_COLOR_ATTR_H`   | Width and height of an area that shares one color setting. Usually `1`. |

`GFX_COLOR_ATTR_W` and `GFX_COLOR_ATTR_H` tell you whether a color
belongs to one pixel or a larger block.

Two prefixes run through this API. `DISPLAY_` is which mode you are in
and how big it is, and answers in a cell mode as well a bitmap one.
`GFX_` is drawing pixels, so those values read `0` or mean nothing outside
a bitmap mode.

## Color

| Call                          | What it does                                  |
| ----------------------------- | --------------------------------------------- |
| `COLOR(fg)` / `COLOR(fg, bg)` | Set the active text or graphics colors for the current display mode. |
| `COLOR_RGB(r, g, b)`          | Set the active text or graphics color from `0` through `255` red, green, and blue values when supported. |
| `RGB(r, g, b)`                | Make a portable `$00RRGGBB` color value using channels from 0 to 255. |
| `BACKGROUND_COLOR(color)`     | Set the target's main background color when supported. |
| `BORDER_COLOR(color)`         | Set the display border color when supported.  |
| `PALETTE_SET(index, color)`   | Change how pixels of value `index` look using a standard color or `RGB` value. |
| `PALETTE_RESET()`             | Restore the palette a graphics mode starts with. |

In a cell mode, `COLOR` sets the colors used by later `PRINT` and `INPUT`
calls. The one argument form changes the foreground and keeps the target's
current or default background. The two argument form also changes the
background when the target supports it.

In a bitmap mode, `COLOR` sets the colors used by later drawing and
`GFX_PRINT` calls. Some modes use pixel indices for the one argument form
and color codes for the two argument form.

When `BITMAP_MULTICOLOR_HUE_ONLY` is `TRUE`, `BITMAP_MULTICOLOR` shares
one brightness across the screen. The one argument `COLOR` form selects
the hue from the color value and uses a medium brightness.

`COLOR_RGB` sets the foreground for subsequent text or graphics drawing
where supported. Channels range from 0 through 255; the background is
unchanged. Unsupported targets do nothing.

`RGB` returns a `U32` in `$00RRGGBB` format.

`BACKGROUND_COLOR` and `BORDER_COLOR` accept target color values,
including the portable color constants such as `BLACK`, `WHITE`, and
`BLUE`. Both calls are available on every target. An unsupported call
does nothing and produces a compile warning.

| Support value | Meaning |
| --- | --- |
| `BACKGROUND_COLOR_SUPPORTED` | `TRUE` when `BACKGROUND_COLOR` changes the display. |
| `BORDER_COLOR_SUPPORTED` | `TRUE` when `BORDER_COLOR` changes the display border. |

Use these support values with `@IF` or `@REQUIRES` when the color change
is required.

### Portable color constants

`BLACK` `WHITE` `RED` `CYAN`
`PURPLE` `GREEN` `BLUE`
`YELLOW` `ORANGE` `BROWN`
`LIGHT_RED` `DARK_GRAY` `GRAY`
`LIGHT_GREEN` `LIGHT_BLUE` `LIGHT_GRAY`

`STD_COLORS` is a `U8` `CONST` array of the target's named color values.
Most targets list all 16; a target with fewer colors lists only those.
These are color codes, not necessarily pixel indices for the current mode.
Modes that share hue or brightness cannot show all these shades at once.
`STD_COLOR_LAST` is its last valid index and follows
`ARRAY_BASE`. Use `LEN(STD_COLORS)` for the element count.

A target defines only the color names it has, so using a missing name is
a compile error. A call that names a color its display cannot show in
that place fails the same way; a color computed at run time is not
checked.

`GFX_NATIVE_COLOR(idx)` returns the target color value for an index in
the current mode.

### Palettes

```basic
CONST SKY_BLUE AS U32 = RGB(48, 160, 255)

PALETTE_SET 1, SKY_BLUE
PALETTE_SET 2, RGB(255, 96, 32)
```

`PALETTE_SUPPORTED` is `TRUE` when palette entries can be changed, and
`PALETTE_SIZE` tells you how many entries there are.
`index` is the color number used by `PLOT` and `COLOR`.

`PALETTE_RGB_SUPPORTED` is `TRUE` when `PALETTE_SET index, RGB(r, g, b)`
can use any RGB color. If the target only offers a fixed set of colors,
an RGB palette write does nothing instead of guessing the nearest one.
Standard color values still use `PALETTE_SET`.

## Graphics

| Call                                              | What it does                                  |
| ------------------------------------------------- | --------------------------------------------- |
| `PLOT(x, y)` / `PLOT(x, y, color)` / `UNPLOT(x, y)` | Set/clear a point. Color plot uses black background and falls back to `PLOT(x, y)` when color is unavailable. |
| `POINT(x, y)`                                     | Read a point. `0` means background/off; other values depend on the mode. |
| `LINE(x0, y0, x1, y1)`                            | Draw a line between any two points.           |
| `HLINE(x0, x1, y)` / `UNHLINE(x0, x1, y)`         | Horizontal span. Endpoints may be in either order. |
| `VLINE(x, y0, y1)` / `UNVLINE(x, y0, y1)`         | Vertical span. Endpoints may be in either order. |
| `BOX(x0, y0, x1, y1)` / `FILLBOX(x0, y0, x1, y1)` | Outline / filled rectangle.                   |
| `CIRCLE(xc, yc, r)` / `CIRCLE_E(xc, yc, rx, ry)`  | Circle / ellipse.                             |
| `GFX_PRINT(x, y, text$, transparent)`             | Draw 8x8 text on the bitmap screen.            |
| `GFX_PRINT_CENTER_X(y, text$, transparent)`       | Draw text centered across the bitmap screen. |
| `GFX_PIXEL_BLIT(addr, x, y, w, h)`                | Draw a `w` by `h` 1 bit image at any pixel position. |
| `GFX_CELL_BLIT(addr, cx, py, w)`                  | Copy prepared bitmap bytes into `w` adjacent 8x8 cells. |
| `GFX_CELL_COLOR_ATTR(cx, cy, fg, bg)`             | Set one graphics color cell's attributes.     |
| `GFX_CELL_COLOR_ATTR_SPAN(cx, cy, w, fg, bg)`     | Set attributes for `w` color cells in a row.  |
| `GFX_CELL_COLOR_ATTR_ROW(cy, fg, bg)`             | Set attributes for a whole color cell row.    |
| `GFX_SWAP()`                                      | Make the finished graphics buffer visible.    |

Coordinates use the current graphics mode: pixels in bitmap modes and
cells in modes that use cell coordinates.

The three argument `PLOT` falls back to `PLOT(x, y)` when
`PLOT_COLOR_COUNT` is `0`. `DISPLAY` updates this value for the selected
mode. It can differ from `GFX_COLOR_COUNT` when the display supports
multiple colors but `PLOT` cannot select one for each call.

### Text on bitmap displays

`GFX_PRINT` and `GFX_PRINT_CENTER_X` use the included 8x8 font and the
current `COLOR`. The character shape uses the foreground color.
`FALSE` fills the rest of its 8x8 area with the background color, while
`TRUE` leaves those pixels unchanged. Lowercase letters use their uppercase
shapes, and unsupported characters draw as spaces. Use these calls in a mode
where `GFX_CAN_PLOT` returns `TRUE`.

`GFX_PRINT_CENTER_X` centers within `DISPLAY_WIDTH`.

### Blitting

`GFX_PIXEL_BLIT` draws a `w` by `h` image at any pixel position, so it is
not tied to the 8x8 cell grid. The source is portable 1 bit data in row
order: each row takes `(w + 7) / 8` bytes, and bit 7 of a byte is its
leftmost pixel. Set bits use the current `COLOR` foreground and clear
bits use the background, so the rectangle is opaque. Pixels outside the
rectangle keep their contents, and there is no clipping, so keep the whole
rectangle on screen.

Some bitmap modes share one color between neighboring pixels. There, pixels
just outside the rectangle can change color even though they keep their on
or off state.

`GFX_PIXEL_BLIT` repacks and shifts pixels while it draws, so it is much
slower than `GFX_CELL_BLIT`. Use it for title art, HUD panels, and other
drawing that does not repeat every frame.

`GFX_PIXEL_BLIT_SUPPORTED` is `TRUE` when the target draws it. Otherwise the
call does nothing.

`GFX_CELL_BLIT` copies prepared bitmap data in 8x8 cells. `cx` is the cell
column, and `py` is a pixel row that must be a multiple of 8. The source
contains `w` cells with 8 bytes per cell: all 8 rows of the leftmost
cell, followed by all 8 rows of the next cell. The bytes must already
use the current bitmap mode's format. `GFX_CELL_BLIT` does not clip.

For movement smaller than one cell, keep shifted copies of the image and
blit the aligned cell window that contains it. `GFX_CELL_BLIT_SUPPORTED` is
`1` when the target supports `GFX_CELL_BLIT`. Otherwise it does nothing.

### Color cells

`GFX_CELL_COLOR_ATTR` and its span and row forms set color attributes on
the current bitmap. One graphics color cell is
`GFX_COLOR_ATTR_W` by `GFX_COLOR_ATTR_H` pixels. These calls are
separate from `CELL_COLORS` and `CELL_COLOR`, which change text screen colors. When
`GFX_CELL_COLOR_ATTR_SUPPORTED` is `FALSE`, these calls do nothing.

### Buffering

`GFX_BUFFERING` tells you what `GFX_SWAP` does:

- `GFX_BUFFERING_NONE`: graphics are not buffered, and calling
  `GFX_SWAP` is a compile error.
- `GFX_BUFFERING_COPY`: `GFX_SWAP` copies your drawing to the screen.
  Your drawing buffer keeps its contents.
- `GFX_BUFFERING_FLIP`: `GFX_SWAP` exchanges the visible and drawing
  buffers. You then draw into an older frame and must redraw anything
  you still need.

Check `GFX_BUFFERING` before calling `GFX_SWAP`.

`GFX_SWAP` and the `GFX_PAGE_` calls are separate things, and a target
can have either one without the other. `GFX_SWAP` handles its own second
buffer, so you never see or name it. The `GFX_PAGE_` calls hand you pages
to select and write yourself. A target that flips inside `GFX_SWAP` still
reports one page, because there is only one you can reach.

## Layer

The layer API selects screen layers used for text, cells, and tiles. Use
it when a target has more than one layer.

```basic
@REQUIRES LAYER_SUPPORTED @ELSE "layer support is required"

LAYER_SELECT LAYER_0
CLS
PRINT "MAIN LAYER"

IF LAYER_COUNT > 1 THEN
	LAYER_SELECT LAYER_1
	LAYER_WRITE_PRIORITY LAYER_1, LAYER_PRIORITY_HIGH
	PRINT "FRONT LAYER"
ENDIF
```

| Call | What it does |
| --- | --- |
| `LAYER_SELECT(layer)` | Direct later text, cell, and tile writes to a layer. |
| `LAYER_SHOW(layer)` | Make a layer visible without changing its contents. |
| `LAYER_HIDE(layer)` | Hide a layer without changing its contents. |
| `LAYER_WRITE_PRIORITY(layer, priority)` | Set the priority used by later writes to a layer. |

`LAYER_SELECT` affects later `PRINT`, `CLS`, cell drawing, and tile
map writes. It does not clear, show, hide, or move the selected layer.
Selecting a layer does not select a palette. Use the SCROLL API to move
screen content.

Write priority starts at `LAYER_PRIORITY_LOW`. Use
`LAYER_PRIORITY_HIGH` for later cells or tiles that should use the
target's high priority setting. Changing write priority does not change
cells or tiles already on the layer.

| Constant | Meaning |
| --- | --- |
| `LAYER_SUPPORTED` | `TRUE` when the target has portable layer support. |
| `LAYER_COUNT` | Number of available screen layers. |
| `LAYER_VISIBILITY_SUPPORTED` | `LAYER_SHOW` and `LAYER_HIDE` are supported. |
| `LAYER_WRITE_PRIORITY_SUPPORTED` | `LAYER_WRITE_PRIORITY` is supported. |
| `LAYER_0` / `LAYER_1` | Layer numbers. Only use layers below `LAYER_COUNT`. |
| `LAYER_PRIORITY_LOW` / `LAYER_PRIORITY_HIGH` | Portable write priority values. |

When `LAYER_VISIBILITY_SUPPORTED` or `LAYER_WRITE_PRIORITY_SUPPORTED` is `FALSE`, its
calls do nothing. Check these values when a program depends on layer
visibility or write priority.

## Scrolling

The SCROLL API moves a area in small steps. When a new row or
column comes into view, the program fills it before showing the new
position.

On a target with screen layers, call `LAYER_SELECT` before `SCROLL_BEGIN` or
`SCROLL_BEGIN_REGION`. Scrolling stays on that layer until the next begin call,
so later writes can use a separate fixed layer without moving it.

Targets with `SCROLL_REGION_SUPPORTED` can leave cells outside the scrolling
area fixed. Region coordinates use absolute cell positions. Coordinates passed
to `SCROLL_CELL_PUTC` start at the top left of the scrolling area. Use normal
cell calls for fixed headers and status areas.

If only row regions are available, use `0` for `x` and `TEXT_WIDTH` for
`columns`. If only column regions are available, use `0` for `y` and
`TEXT_HEIGHT` for `rows`.

```basic
@REQUIRES SCROLL_REGION_ROWS_SUPPORTED @ELSE "fixed rows are required"

CELL_PRINT 0, 0, "FIXED TITLE"
SCROLL_BEGIN_REGION 0, 1, TEXT_WIDTH, TEXT_HEIGHT - 1, 0, 8
```

```basic
@REQUIRES SCROLL_CELL_SUPPORTED @ELSE "cell scrolling is required"

SCROLL_BEGIN U8(0), U8(8)

FILL = SCROLL_MOVE(I8(0), I8(1))

IF (FILL & SCROLL_FILL_BOTTOM) <> U8(0) THEN
	DRAW_NEW_ROW SCROLL_BUFFER_ROWS - U8(1)
ENDIF

EDGES = SCROLL_PRESENT
```

Positive X moves the view right and exposes the right edge. Positive Y moves
the view down and exposes the bottom edge. Negative movement exposes the
opposite edge. Move one axis and cross at most one cell boundary per call.

At a cell boundary, `SCROLL_MOVE` returns a `SCROLL_FILL_*` flag telling you
which outer edge to fill. Write that edge with `SCROLL_CELL_PUTC` before
calling `SCROLL_PRESENT`. The outer coordinates are column `0`, column
`SCROLL_BUFFER_COLUMNS - 1`, row `0`, and row `SCROLL_BUFFER_ROWS - 1`.

`SCROLL_PRESENT` shows the prepared position and returns the matching
`SCROLL_EDGE_*` flag when a cell boundary is now visible. Update the
program's scroll position from that result rather than the earlier fill
request.

| Name | What it does |
| --- | --- |
| `SCROLL_BEGIN(step_x, step_y)` | Start cell scrolling. Use `8, 8` for normal 8 pixel cells. A zero step disables that axis. |
| `SCROLL_BEGIN_REGION(x, y, columns, rows, step_x, step_y)` | Start scrolling inside a cell region while leaving cells outside it fixed. |
| `SCROLL_CELL_PUTC(col, row, code)` | Write one character code in the scrolling area. |
| `SCROLL_MOVE(dx, dy)` | Prepare relative movement and return a `SCROLL_FILL_*` edge when new cells are needed. |
| `SCROLL_PRESENT()` | Show the prepared movement and return a `SCROLL_EDGE_*` boundary that is now visible. |
| `SCROLL_BUFFER_COLUMNS` / `SCROLL_BUFFER_ROWS` | Size of the cell area available for scrolling content. |
| `SCROLL_VIEW_COLUMNS` / `SCROLL_VIEW_ROWS` | Visible cell area while scrolling is active. |
| `SCROLL_X_GRANULARITY` / `SCROLL_Y_GRANULARITY` | Smallest movement in target pixels for the current display mode. `0` means the axis is unavailable. |
| `SCROLL_FILL_LEFT` / `SCROLL_FILL_RIGHT` | The matching outer column must be filled. |
| `SCROLL_FILL_TOP` / `SCROLL_FILL_BOTTOM` | The matching outer row must be filled. |
| `SCROLL_EDGE_LEFT` / `SCROLL_EDGE_RIGHT` | The matching column boundary was shown. |
| `SCROLL_EDGE_TOP` / `SCROLL_EDGE_BOTTOM` | The matching row boundary was shown. |

| Support value | Meaning |
| --- | --- |
| `SCROLL_SUPPORTED` | `TRUE` when the target has portable scrolling. |
| `SCROLL_CELL_SUPPORTED` | Cell screens can scroll. |
| `SCROLL_CELL_COLOR_SUPPORTED` | Each scrolling cell can keep its own color. When `FALSE`, use shared colors such as `CELL_COLORS`. |
| `SCROLL_BITMAP_SUPPORTED` | Bitmap screens can scroll. |
| `SCROLL_X_SUPPORTED` / `SCROLL_Y_SUPPORTED` | The corresponding axis is supported. |
| `SCROLL_REGION_SUPPORTED` | The target supports a scrolling cell region. |
| `SCROLL_REGION_ROWS_SUPPORTED` | The region may leave fixed rows above or below it. |
| `SCROLL_REGION_COLUMNS_SUPPORTED` | The region may leave fixed columns beside it. |

Movement size may change after `DISPLAY`, so read the granularity values
after choosing a mode. The target pages list the available screen types,
axes, colors, and regions. The
[portable scroll example](../examples/__portable__/scroll.cbs) checks
the available axes.

## Misc

| Name             | What it does                                                     |
| ---------------- | ---------------------------------------------------------------- |
| `BEEP(duration)` | Play a target specific tone. Duration units are target specific. |
| `BEEP(period, duration)` | Play a tone with a chosen period on targets that provide this variant. Period and duration units are target specific. |
| `DELAY(n)`       | Pause for roughly target specific units.                         |
| `TONE(period, duration)` | Play a raw tone on targets that provide it. Period and duration units are target specific. |
| `TONE(period, duration, async)` | On supporting targets, `TRUE` lets gameplay continue while the tone plays. Some targets service playback during `FRAME_WAIT`. |

See the target page for the units used by `BEEP`, `DELAY`, and `TONE`.

## Sound

The sound API plays notes in a form that works across targets. `VOICE`
starts at zero. `NOTE` uses the `NOTE_*` constants from `NOTE_C2`
through `NOTE_B6`, and volume runs from `0` through `SOUND_VOLUME_MAX`.
`SHAPE` chooses a tone, pulse, noise, or another supported sound.
`NOTE_REST` either silences a voice or reserves it for a timed rest.

For sounds unique to one machine, see the sound and chip sections on its
target page.

| Name | What it does |
| --- | --- |
| `SOUND_SUPPORTED` | `TRUE` when the target supports sound. |
| `SOUND_PRESENT` | `TRUE` when sound output is currently available. |
| `SOUND_VOICES` | Number of target sound voices. |
| `SOUND_VOLUME_MAX` | Highest volume accepted by the portable note calls. |
| `SOUND_VOLUME_LOW` | Portable low volume scaled for the target. |
| `SOUND_VOLUME_MEDIUM` | Portable medium volume scaled for the target. |
| `SOUND_VOLUME_HIGH` | Portable high volume scaled for the target. |
| `SOUND_SHAPE_SUPPORTED` | `TRUE` when the target can select a waveform or noise shape. |
| `SOUND_NOISE_SUPPORTED` | `TRUE` when the target has a noise shape or noise channel. |
| `SOUND_NOTE_BLOCKING` | `TRUE` when notes are made in software, so `PLAY_NOTE` plays a short note and returns when it ends instead of leaving the voice sounding. |
| `SOUND_VOLUME_MODEL` | Volume behavior, one of the `SOUND_VOLUME_MODEL_*` values. |
| `SOUND_NOTE_MIN` | Lowest MIDI note number supported by target sound. |
| `SOUND_NOTE_MAX` | Highest MIDI note number supported by target sound. |
| `SOUND_OFF(voice)` | Stop one sound voice. |
| `SOUND_ALL_OFF()` | Stop all target sound. Pure crustyBASIC also accepts `SILENCE`. |
| `PLAY_NOTE(voice, note, volume)` | Start a note immediately. |
| `PLAY_NOTE_SHAPE(voice, note, shape, volume)` | Play a note with the chosen kind of sound. |
| `PLAY_NOTE_FOR(voice, note, volume, duration)` | Play a note for `duration` ticks, then stop it. |
| `PLAY_NOTE_SHAPE_FOR(voice, note, shape, volume, duration)` | Play a note with the chosen kind of sound for `duration` ticks, then stop it. |

Durations use the same units as `TICKS`, with `TICKS_HZ` ticks per second,
so a value of `TICKS_HZ` holds a note for one second on every target.
The `_FOR` calls block for the whole duration. `PLAY_NOTE` starts a note
and returns on targets with a sound chip. When `SOUND_NOTE_BLOCKING` is
`TRUE` the target makes sound in software, `PLAY_NOTE` plays a short note
and returns when it ends, and nothing keeps sounding afterwards.

`SOUND_VOLUME_MODEL` describes how volume is shared:

- `SOUND_VOLUME_MODEL_NONE` means volume cannot be changed.
- `SOUND_VOLUME_MODEL_MASTER` means all voices share one volume.
- `SOUND_VOLUME_MODEL_GROUP` means groups of voices share a volume.
- `SOUND_VOLUME_MODEL_VOICE` means each voice has its own volume.

Timed sound starts a voice and returns immediately. Durations use the
same units as `TICKS`, with `TICKS_HZ` ticks per second. Some targets
advance timed sound automatically; cooperative providers advance after
`FRAME_WAIT`. `TICKS_RESET` does not affect timed notes.

| Name | What it does |
| --- | --- |
| `SOUND_TIMED_SUPPORTED` | `TRUE` when timed notes are supported. |
| `SOUND_TIMED_FREE_RUNNING` | `TRUE` when timed notes advance automatically. |
| `SOUND_TIMED_VOICES` | Number of voices that can play timed notes. |
| `SOUND_TIMED_INIT()` | Start timed sound. Timed play calls do this when needed. |
| `SOUND_TIMED_UNINSTALL()` | Stop timed sound and silence its voices. |
| `SOUND_TIMED_NOTE(voice, note, volume, ticks)` | Play a note for `ticks`. |
| `SOUND_TIMED_NOTE_SHAPE(voice, note, shape, volume, ticks)` | Play a shaped note for `ticks`. |
| `SOUND_TIMED_REST(voice, ticks)` | Keep a voice silent for `ticks`. |
| `SOUND_TIMED_BUSY(voice)` | `1` while a timed note or rest is active on the voice. |
| `SOUND_TIMED_REMAINING(voice)` | Remaining ticks for the voice. |
| `SOUND_TIMED_WAIT(voice)` | Wait until the voice is no longer busy. |
| `SOUND_TIMED_WAIT_ALL()` | Wait until all timed voices are no longer busy. |
| `SOUND_TIMED_OFF(voice)` | Cancel and silence one timed voice. |
| `SOUND_TIMED_OFF_ALL()` | Cancel and silence all timed voices. |

Use `@IF SOUND_TIMED_SUPPORTED THEN` or
`@REQUIRES SOUND_TIMED_SUPPORTED` when a program needs timed sound. On
targets without timed sound support, these calls do nothing.

## Audio

The AUDIO API loads and plays one prepared audio asset, such as a
converted WAV file. One audio asset can be loaded at a time. Loading
is separate from playback, so a program can load music during a loading
screen and start it later:

```basic
@INCLUDE_AUDIO TITLE "assets/title.wav"

OK = AUDIO_LOAD(ADDR TITLE_AUDIO)

' finish setup or wait for input

OK = AUDIO_PLAY
```

Calls:

| Call | What it does |
| --- | --- |
| `AUDIO_LOAD(src_addr)` | Load embedded CBA audio without playing it. Returns `1` on success. |
| `AUDIO_LOAD(path$)` | Load a CBA file or supported target audio file. Returns `1` on success. |
| `AUDIO_PLAY` | Start the loaded asset's default song from its beginning. Returns `1` on success. |
| `AUDIO_PLAY(song)` | Start a one based song number from its beginning. The song number is not range checked. |
| `AUDIO_STOP` | Stop playback and release the sound output, but keep the asset loaded for replay. |
| `AUDIO_UNLOAD` | Stop playback and forget the loaded asset. |

Calling `AUDIO_LOAD` again stops and unloads the current asset first. If
the new load fails, no asset remains loaded.

`AUDIO_PLAY` only starts the loaded audio, so it never loads a file from
storage.

Audio support values:

| Name | What it means |
| --- | --- |
| `AUDIO_SUPPORTED` | `TRUE` when AUDIO is available for the chosen target and system. |
| `AUDIO_PLAYBACK_MODEL` | `AUDIO_PLAYBACK_NONE`, `AUDIO_PLAYBACK_AUTOMATIC`, `AUDIO_PLAYBACK_FRAME_SERVICE`, or `AUDIO_PLAYBACK_BLOCKING`. Automatic playback continues by itself. Frame service playback advances after each completed `FRAME_WAIT`. Blocking playback returns from `AUDIO_PLAY` when the sound has finished. |
| `AUDIO_FILE_SUPPORTED` | `AUDIO_LOAD(path$)` accepts prepared CBA files. |
| `AUDIO_TARGET_FILE_SUPPORTED` | `AUDIO_LOAD(path$)` also accepts the target file types listed on its target page. |
| `AUDIO_SOUND_SHARED` | `TRUE` when Sound API calls can be used during AUDIO playback. |

Frame service playback needs a regular `FRAME_WAIT` in the game loop:

```basic
WHILE RUNNING
	FRAME_WAIT

	' game logic
WEND
```

When `AUDIO_SOUND_SHARED` is `FALSE`, AUDIO has exclusive use of sound output
from a successful `AUDIO_PLAY` until `AUDIO_STOP` or `AUDIO_UNLOAD`. Do
not use notes, timed sounds, or target specific sound calls during that
time. If timed sound is active, call `SOUND_TIMED_UNINSTALL` before
`AUDIO_PLAY` and `SOUND_TIMED_INIT` after AUDIO stops.

### `@INCLUDE_AUDIO` and CBA (crustyBASIC Audio) files

```basic
@INCLUDE_AUDIO TITLE "assets/title.wav"
@INCLUDE_AUDIO TUNE "assets/tune.sid"
@INCLUDE_AUDIO BOSS "BOSS.CBA" "assets/boss.wav"
```

The one-path form embeds prepared audio as `TITLE_AUDIO`. Load it with
`AUDIO_LOAD(ADDR TITLE_AUDIO)`. The two-path form does not embed the audio.
It creates `BOSS_AUDIO_FILE`, stages that file with the program, and creates
`BOSS_LOAD`. Call `BOSS_LOAD` to load it from storage. crustyBASIC adjusts
the staged filename to suit the selected medium.

The source can be an audio file such as WAV or SID. crustyBASIC
recognizes it by its contents, converts it to CBA, and makes sure the
target can play it. See the `## Audio` section on the target's page for
the formats it accepts.

Embedded audio is loaded as CBA data, not as the original WAV or SID. There
are two ways to prepare and embed it:

- `@INCLUDE_AUDIO TITLE "assets/title.wav"` converts the source to CBA,
  embeds it as `TITLE_AUDIO`, and checks it for the target. Load it with
  `AUDIO_LOAD(ADDR TITLE_AUDIO)`.
- `cb-audio --cba` converts the source to a CBA file beforehand. Embed
  that file unchanged with `@INCLUDE_FILE TITLE_AUDIO "TITLE.CBA"`, then
  load it with `AUDIO_LOAD(ADDR TITLE_AUDIO)`.

`@INCLUDE_FILE` does not convert files. If it is given a WAV or SID file,
the embedded bytes are still WAV or SID data and `AUDIO_LOAD(ADDR ...)`
rejects them.

`@INCLUDE_AUDIO` also accepts an existing CBA file. crustyBASIC checks
that it works with the current target and uses it unchanged.
`@INCLUDE_AUDIO` cannot be used inside `@BANK`. An embedded CBA file
larger than the 65535 byte `CONST` data limit is a compile error.

You do not need a directive when the program chooses the path. Put the
CBA or supported target file on its disk or other storage:

```basic
PATH$ = "LEVEL3.CBA"
OK = AUDIO_LOAD(PATH$)
```

A CBA file contains audio prepared for one target. Use `@INCLUDE_AUDIO`
or `cb-audio` to create one.

Converter usage:

```sh
tools/bin/cb-audio --target TARGET --region REGION_NTSC --name TITLE input.wav -o title.cbi
tools/bin/cb-audio --target TARGET --region REGION_NTSC --cba input.wav -o TITLE.CBA
```

`--region` defaults to `REGION_NTSC`. Sources such as WAV that do not
depend on a region use region `0`. The `--cba` form writes a CBA file.
The `--name` form writes a `.cbi` include containing `TITLE_AUDIO`.

`--target` takes a target or system name. A target name uses its default
system. Use the system name when audio works on only some systems for a
target.

See the `## Audio` section in each supported target page under
[`targets/`](targets/) for its source formats, playback model, and
storage options. On targets without AUDIO support,
`AUDIO_PLAYBACK_MODEL` is `AUDIO_PLAYBACK_NONE`, `AUDIO_SUPPORTED` is
`FALSE`, and `AUDIO_LOAD` and `AUDIO_PLAY` return `0`. The other calls do
nothing.

## Timing

| Name                    | What it does                                                    |
| ----------------------- | --------------------------------------------------------------- |
| `TIMER_TYPE`            | Timer kind, one of the `TIMER_*` values in this table.          |
| `TIMER_SUPPORTED`       | `TRUE` when `TIMER_TYPE <> TIMER_NONE`, otherwise `FALSE`.       |
| `TIMER_HIGH_RES`        | `TRUE` when the timer is more precise than one display frame.    |
| `TIMER_START()`         | Start or reset the elapsed timer.                               |
| `TIMER_STOP()`          | Stop the timer and store the elapsed count in `TIMER_HI:TIMER_LO`. |
| `TIMER_HI`              | High 16 bits of the elapsed tick count.                         |
| `TIMER_LO`              | Low 16 bits of the elapsed tick count.                          |
| `TIMER_NONE`            | No elapsed timer.                                               |
| `TIMER_FREE_RUNNING_HW` | The target provides a timer that is always running.              |
| `TIMER_FREE_RUNNING_SW` | The target's system time is always running.                       |
| `TIMER_START_STOP_HW`   | The target provides a timer used only while measuring.           |
| `TIMER_START_STOP_SW`   | The target measures system time between start and stop.          |

Timer units differ by target. Compare `TIMER_HI` first and then
`TIMER_LO`; fewer ticks means a faster result on the same target. Check
`TIMER_TYPE <> TIMER_NONE` before relying on the timer.

## Frames

Use the frame API to keep animation in step with the display.

| Name                        | What it does                                                  |
| --------------------------- | ------------------------------------------------------------- |
| `FRAME_WAIT()`              | Block until the next frame, advance `FRAME_COUNTER`, and run cooperative frame services. |
| `FRAME_WAIT(count)`         | Wait `count` frames.                                          |
| `FRAME_DELAY(count)`        | Wait `count` frames and keep displays that need regular drawing active. |
| `FRAME_COUNTER`             | `U16` frame count advanced by `FRAME_WAIT` or automatic frame timing. |
| `FRAME_SUPPORTED`           | `TRUE` when the target can wait for display frames.            |
| `FRAME_VBI_SUPPORTED`       | `TRUE` when frame callbacks can run between displayed frames.  |
| `FRAME_INSTALL_SUPPORTED`   | `TRUE` when `FRAME_INSTALL` can run a callback once per frame.  |
| `FRAME_INSTALL_INTERRUPT`   | `TRUE` when the callback may interrupt normal program flow.     |
| `FRAME_INSTALL_SYNTHESIZED` | `TRUE` when `FRAME_WAIT` itself runs the callback.              |
| `FRAME_INSTALL(addr)`       | Set a no argument PROC as the frame callback.                 |
| `FRAME_UNINSTALL()`         | Remove the frame callback.                                     |

Pass the callback as `ADDR(name)`.

On targets without frame timing, `FRAME_WAIT` only increments
`FRAME_COUNTER`, and `FRAME_INSTALL` and `FRAME_UNINSTALL` do nothing. Use
`@IF FRAME_SUPPORTED THEN` or `@REQUIRES FRAME_SUPPORTED` when the
program depends on real frame timing. `FRAME_WAIT` does not swap
graphics buffers.

Use `@PERIODIC` for cooperative animation and other short frame work:

```basic
@PERIODIC
PROC UPDATE_EVERY_FRAME
ENDPROC

@PERIODIC FRAMES 5
PROC UPDATE_EVERY_FIFTH_FRAME
ENDPROC

@PERIODIC MS 1000
PROC UPDATE_EVERY_SECOND
ENDPROC
```

A periodic PROC has no parameters or return value. Bare `@PERIODIC` runs
after every completed `FRAME_WAIT`; `FRAMES N` first runs on wait N and
then every N waits. Periodic PROCs run in declaration order and in normal
program context. `N` is an integer literal from
1 through 65535. Periodic PROCs are available wherever `FRAME_WAIT` is
available.

`MS N` keeps time with `TICKS` instead of counting waits. The interval is
converted to ticks when the program is compiled, using `TICKS_HZ` for the
target and region being built, so the same source keeps the same pace on
a PAL or NTSC build. The PROC runs on the first `FRAME_WAIT` after a full
interval has passed since its previous run, and the first run waits a
full interval from program start. An interval shorter than one tick is a
compile error. On targets where `TICKS` only advances during `FRAME_WAIT`,
`MS` measures waits at the nominal rate like `FRAMES` does. Nothing is
added to `FRAME_WAIT` unless a program declares an `MS` PROC.

`@PERIODIC` does not use or replace the single target callback managed by
`FRAME_INSTALL`. Raster line callbacks are separate because they
can run several times during one displayed frame.

`FRAME_COUNTER` can change while a program reads it on targets where it
advances without `FRAME_WAIT`. Use `TICKS` when the count must not change
during a read.

## Ticks

(Almost) every target provides an elapsed tick counter. `TICKS_RESET` remembers
its current value, and `TICKS` returns the number of ticks since that
reset. It does not change the system clock.

| Name                 | What it does                                                    |
| -------------------- | --------------------------------------------------------------- |
| `TICKS`              | `U32` ticks elapsed since the last `TICKS_RESET`.               |
| `TICKS_RESET()`      | Restart the elapsed count from zero.                            |
| `TICKS_HZ`           | Nominal ticks per second for the target.                        |
| `TICKS_FREE_RUNNING` | `TRUE` when ticks advance automatically, `FALSE` when they advance only during frame wait or drawing calls. |

Tick sources, rates, and timing details differ by target. See the target
pages under [`targets/`](targets/) when a program needs exact timing.

## Raster interrupts

Some targets can run callbacks at selected display rows.

- `RASTER_CLEAR()` - clear the list of marked display rows.
- `RASTER_MARK_ROW(row)` - mark one visible display row.
- `RASTER_MARK_ROWS(first, count, step)` - mark several visible rows.
- `RASTER_MARK_OFFSET(offset)` - mark a target specific display position
  given as a byte offset. See the target page before using this form.
- `RASTER_LINE_INSTALL(line_addr)` - set the callback for marked display
  rows.
- `RASTER_FRAME_INSTALL(frame_addr)` - set a callback that runs once per
  frame.
- `RASTER_START()` - resume paused raster callbacks.
- `RASTER_STOP()` - pause raster callbacks without removing them.
- `RASTER_UNINSTALL()` - stop and remove the raster callbacks.
- `RASTER_WSYNC()` - wait until the display starts its next row.
- `RASTER_INDEX` - number of the current marked row callback, starting at
  `0` each frame.

The callback PROCs must take no arguments. Pass them as `ADDR(name)`.
Supported rows and timing vary by target; its page contains the details
and examples.

## Files

The file API opens files on numbered channels and reads or writes one
byte, line, or value at a time. What a path means depends on the target
and its available storage.

Using these calls on a target without file support is a compile error.
The target pages list the supported file systems and explain target
specific arguments.

`FILE_SUPPORTED` is `TRUE` when the file API can be used.
`FILE_DIRECTORY_SUPPORTED` is `TRUE` when directory listings are supported.

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
| `FILE_OPEN(channel, path$, mode)`           | Open a path on a numbered channel.                |
| `FILE_OPEN_NATIVE(channel, path$, a1, a2)`  | Open with extra values defined by the target.     |
| `FILE_CLOSE(channel)`                       | Close a numbered channel.                         |
| `FILE_READ_BYTE(channel)`                   | Read one byte and return `U8`.                    |
| `FILE_WRITE_BYTE(channel, byte)`            | Write one byte.                                   |
| `FILE_READ_LINE(channel, out line$)`        | Read one line of text into a string.               |
| `FILE_WRITE_STR(channel, text$)`            | Write a string without adding a newline.          |
| `FILE_WRITE_LINE(channel, text$)`           | Write a string followed by the target line ending. |
| `FILE_WRITE_DATA(channel, value)`           | Write one `RESTORE FILE` item to an open file.    |
| `FILE_LOAD(path$, dst, count)`              | Load `count` bytes using the shared asset channel. |
| `FILE_LOAD(channel, path$, dst, count)`     | Load `count` bytes from a path into memory.       |
| `FILE_LOAD_PROGRAM(path$)`                  | Load a program file at its stored address.        |
| `FILE_LOAD_PROGRAM(path$, dst)`             | Load a program file at a supplied address.        |
| `FILE_SAVE_BYTES(channel, path$, src, count)` | Save `count` bytes from memory to a path.       |
| `TEXT_LOAD(path$, name$, out text$)`        | Load one named block from an external text asset. |
| `FILE_STATUS(channel)`                      | Return the target status byte for the channel.    |
| `FILE_COMMAND(cmd, channel, a1, a2, path$)` | Run a target specific file command.               |

`FILE_LOAD_PROGRAM` loads a target's native program file into memory at
runtime without running it. The one argument form uses load address
information stored in the file. The destination form loads the program
data at `dst` instead. It returns `U8`: `FILE_OK` on success or a target
error code.

Supported formats, file header handling, devices, and memory restrictions
are described in the target documentation. Calling an unsupported form
is a compile error. Use `FILE_LOAD` for raw bytes with an explicit byte
count. Keep the loaded data clear of your running program and its variables.

`FILE_WRITE_STR` writes only the given text. `FILE_WRITE_LINE` writes
the text and then the normal line ending for the selected target. Use
`FILE_WRITE_LINE channel, ""` to write a blank line.

`RESTORE FILE channel, path$, ADDR buffer, count` loads `count` bytes
from `path$` into `buffer`. Later `READ` statements use that buffer
instead of the program's `DATA` statements. The file must use the
`RESTORE FILE` format: one length byte followed by the text bytes for
each item. Numbers are stored as decimal text, not as binary integers.

Use `FILE_WRITE_DATA` on a file opened for writing to save values in
that format:

```basic
BUF(8) AS U8
A AS U8
S$ AS STRING * 8

FILE_OPEN 1, "DATA.BIN", FILE_MODE_WRITE
FILE_WRITE_DATA 1, U8(55)
FILE_WRITE_DATA 1, "HELLO"
FILE_CLOSE 1

RESTORE FILE 1, "DATA.BIN", ADDR BUF, 9
READ A
READ S$
```

After that example, `A` is `55` and `S$` is `"HELLO"`. A later bare
`RESTORE` switches `READ` back to the program's `DATA` statements. Use
`FILE_SAVE_BYTES` when you want to save bytes rather than values for
later `READ` statements.

`FILE_OPEN_NATIVE` arguments differ by target. See the target page when
you need to choose a device, secondary address, drive, or ROM mode.

## System

| Support value | What it means |
| --- | --- |
| `CMD_SUPPORTED` | System commands can be run. |
| `CMD_OUTPUT_SUPPORTED` | Command output can be captured. |

| Call | What it does |
| --- | --- |
| `CMD(COMMAND AS STRING) AS I32` | Run a command, wait for it to finish, and return its exit status. |
| `CMD_OPEN(COMMAND AS STRING) AS U8` | Start a command and capture its output. Returns `1` when started. |
| `CMD_READ_LINE(OUT LINE AS STRING, OUT HAS_LINE AS U8)` | Read the next captured line and set `HAS_LINE` when one is available. |
| `CMD_CLOSE() AS I32` | Finish the active captured command and return its exit status. |

Captured output includes everything printed by the command, including
errors. Only one captured command can be active at a time. Call `CMD_READ_LINE` until
`HAS_LINE` is `0`; a blank line has an empty `LINE` and `HAS_LINE` set to
`1`. Each line must fit in the `STRING` passed to `CMD_READ_LINE`.
Programs can store the returned lines in their own string array.

## External storage

Some targets provide this simple storage API in addition to the normal
`FILE_*` calls. It can keep one file open at a time. Check
`FILES_AVAILABLE()` before using it.

| Name / call | What it does |
| --- | --- |
| `FILES_AVAILABLE()` | `1` when external storage is available. |
| `FILES_MODE_READ` / `FILES_MODE_WRITE` / `FILES_MODE_CREATE` | Open modes. |
| `FILES_OK` / `FILES_ERR` | Result constants. |
| `FILES_OPEN(path$, mode)` | Open one path. Returns `FILES_OK` on success. |
| `FILES_CLOSE()` | Close the current file. |
| `FILES_WRITE(data$)` | Write a string to the current file. |
| `FILES_READ_REQUEST(maxlen)` | Ask to read up to `maxlen` bytes. |
| `FILES_HAS_DATA()` | `1` while the requested bytes remain. |
| `FILES_READ_BYTE()` | Read one requested byte. |
| `FILES_READ_END()` | Finish the current read request. |
| `FILES_DELETE(path$)` | Delete a path. |
| `FILES_CHDIR(path$)` | Change directory where supported. |
| `FILES_MKDIR(path$)` | Create a directory where supported. |
| `FILES_MOUNT(path$)` | Mount media where supported. |

## Networking

Network support is optional. Connection calls return a socket number,
which identifies the connection in later calls. Without network support,
`NET_AVAILABLE()` returns `0` and connection and listener calls fail.

| Name / call | What it does |
| --- | --- |
| `NET_TCP` / `NET_UDP` | Protocol constants. |
| `NET_AVAILABLE()` | `1` when networking is available. |
| `NET_GET_IP(out ip$)` | Store the current IP address, or an empty string. |
| `NET_CONNECT(proto, host$, port)` | Open a TCP or UDP connection. Returns its socket number, or `0`. |
| `NET_CLOSE(sock)` | Close a connection. |
| `NET_WRITE(sock, data$)` | Write a string to a connection. |
| `NET_READ_REQUEST(sock, maxlen)` | Ask to read up to `maxlen` bytes from a connection. |
| `NET_HAS_DATA()` | `1` while the requested bytes remain. |
| `NET_READ_BYTE()` | Read one requested byte. |
| `NET_READ_END()` | Finish the current read request. |
| `NET_LISTEN(port)` | Listen for TCP connections. Returns a listener number, or `0`. |
| `NET_ACCEPT(listener)` | Accept a waiting TCP connection. Returns its socket number, or `0`. |
| `NET_CLOSE_CLIENT(sock)` | Close an accepted client while keeping the listener where supported. |

## Image

The image API prepares pictures for a target and displays them at a size
the target supports. `IMAGE_LOAD` selects the right display mode and
shows the image.

Use it for splash screens, loading screens, game backgrounds, and other
static art. It leaves sprites alone, so they can move over the
image.

See the `## Images` section in each image capable target page under
[`targets/`](targets/) for the image formats supported by that target.

Prepared images may use byte RLE on targets that support it.
`@INCLUDE_IMAGE` and `cb-image` use it automatically only when it makes
the stored image smaller. `IMAGE_LOAD` handles packed and unpacked images
through the same calls.

The included image data has its own label:

```basic
CONST TITLE_IMAGE(...) AS U8 = ...

OK = IMAGE_LOAD(ADDR TITLE_IMAGE)
```

Use `@INCLUDE_IMAGE` to convert and embed an image:

```basic
@INCLUDE_IMAGE TITLE "src/title.png"
@INCLUDE_IMAGE TITLE "TITLE.IMG" "src/title.png"
```

The first form creates `CONST TITLE_IMAGE(...) AS U8 = ...` and also
`CONST TITLE_IMAGE_WIDTH AS U16` / `CONST TITLE_IMAGE_HEIGHT AS U16`.
The second form does not embed the image. It creates `TITLE_IMAGE_FILE`,
stages `TITLE.IMG` with the program, and creates `TITLE_LOAD`. Call
`TITLE_LOAD` to load and display it from storage. The width, height, and
format metadata remain available to the program. crustyBASIC adjusts the
staged filename to suit the selected medium. If the source is already an
IMG file, crustyBASIC checks it for the active target and uses it unchanged.

`IMAGE_LOAD(ADDR ...)` expects prepared IMG data rather than the
original PNG or PCX. `@INCLUDE_IMAGE` performs that conversion. You can
also create an IMG file beforehand with `cb-image --img` and embed it
unchanged with `@INCLUDE_FILE TITLE_IMAGE "TITLE.IMG"`. Load that array
with `IMAGE_LOAD(ADDR TITLE_IMAGE)`. Embedding a PNG or PCX directly
with `@INCLUDE_FILE` does not convert it.

Inside an `@BANK N` block, the `CONST` array is placed in bank `N`.

When `IMAGE_TARGET_FILE_SUPPORTED` is `TRUE`, `IMAGE_LOAD` also
accepts the machine's own image files. The target page lists them.

Constants such as `IMAGE_FMT_*` identify the prepared image format. Each
target's image section lists its file formats, image sizes, and format
IDs.

### Effects

`IMAGE_WIPE`, `IMAGE_SLIDE`, `IMAGE_FADE_OUT`, and `IMAGE_FADE_IN` bring an
image onto the screen with a transition instead of all at once. `direction`
is `IMAGE_DOWN`, `IMAGE_UP`, `IMAGE_LEFT`, or `IMAGE_RIGHT`. `delay` is the
number of frames to wait after each step; `0` runs as fast as the copy
allows.

Wipes reveal the image in 8 pixel bands or cell columns, with the edge
moving in the given direction. Slides move the image in from the opposite
edge until it covers the screen. Fades take `IMAGE_FADE_STEPS` steps; how a
target darkens the picture depends on its hardware, see the target page.

`IMAGE_DISSOLVE_IN` reveals scattered pixels in a shuffled order.
`IMAGE_DISSOLVE_OUT` erases them to the native bitmap background colors.
The delay is the number of frames between batches of pixels; `0` runs
as fast as possible. Every pixel is visited, so no stray dots remain.
An optional final `block_size` argument selects square blocks of 1, 2, 4,
or 8 native pixels per side; the default is 1. Each batch visits 256 pixel
positions at size 1, or 16 block positions at larger sizes.

Most targets require an unpacked image in memory for wipes, slides, fade in,
and dissolve in. Add `UNPACKED` to `@INCLUDE_IMAGE`, or `--unpacked` to
`cb-image`, to keep an image unpacked. Some targets also accept RLE8 images;
see the target page. These calls return `0` for unsupported formats or packing. Fade out needs no image data and works on whatever the
screen shows, including images loaded from a file. Dissolve out also needs
no source image.

The effects draw over the current screen contents. Clear the screen first
when the image should appear over a blank screen.

```basic
@INCLUDE_IMAGE TITLE "src/title.png" UNPACKED

OK = IMAGE_WIPE(ADDR TITLE_IMAGE, IMAGE_DOWN, 1)
OK = IMAGE_FADE_OUT(3)
IMAGE_CLEAR
OK = IMAGE_FADE_IN(ADDR TITLE_IMAGE, 3)
```

Calls:

| Call | What it does |
| --- | --- |
| `IMAGE_LOAD(src_addr)` | Set the image's display mode and copy its data to the screen. Returns `1` on success. |
| `IMAGE_CLEAR` | Clear the available graphics, tile, and cell screens. |
| `IMAGE_WIDTH(src_addr)` / `IMAGE_HEIGHT(src_addr)` | Image width and height in pixels. |
| `IMAGE_FORMAT(src_addr)` | The `IMAGE_FMT_*` id. |
| `IMAGE_AUX(src_addr)` | Extra byte whose meaning depends on the image format. |
| `IMAGE_LOAD(path$)` | Load an image file and display it directly when `IMAGE_FILE_SUPPORTED` is `TRUE`. Returns `1` on success. |
| `IMAGE_WIPE(src_addr, direction, delay)` | Reveal the image band by band. Returns `1` on success. |
| `IMAGE_SLIDE(src_addr, direction, delay)` | Move the image in from the opposite edge. Returns `1` on success. |
| `IMAGE_FADE_OUT(delay)` | Fade the screen to black. Returns `1` on success. |
| `IMAGE_FADE_IN(src_addr, delay)` | Show the image black and brighten it to its colours. Returns `1` on success. |
| `IMAGE_DISSOLVE_IN(src_addr, delay [, block_size])` | Reveal scattered pixels from an unpacked image. Returns `1` on success. |
| `IMAGE_DISSOLVE_OUT(delay [, block_size])` | Erase scattered pixels to the bitmap background colors. Returns `1` on success. |

Image support values:

| Name | What it means |
| --- | --- |
| `IMAGE_SUPPORTED` | Image display is supported on this target. |
| `IMAGE_FILE_SUPPORTED` | `IMAGE_LOAD` can load image files from storage. |
| `IMAGE_TARGET_FILE_SUPPORTED` | `IMAGE_LOAD` can load target image file formats directly. |
| `IMAGE_WIPE_SUPPORTED` | `IMAGE_WIPE` is available. |
| `IMAGE_SLIDE_SUPPORTED` | `IMAGE_SLIDE` is available. |
| `IMAGE_FADE_SUPPORTED` | `IMAGE_FADE_OUT` and `IMAGE_FADE_IN` are available. |
| `IMAGE_DISSOLVE_SUPPORTED` | `IMAGE_DISSOLVE_OUT` and `IMAGE_DISSOLVE_IN` are available. |
| `IMAGE_FADE_STEPS` | Number of steps in a fade. |

Converter usage:

```sh
tools/bin/cb-image --target TARGET --name TITLE input.png -o title.cbi
tools/bin/cb-image --target TARGET input.png --img -o title.img
```

Most targets with image support also accept indexed PNG and PCX files up
to their supported image size. Colors are adjusted to what the target can
show, so the result may not match the source exactly. Images larger than
the target mode are an error, so scale them down first.

`--as` chooses a format when more than one would fit. `--aux` sets an
extra value for formats that define one on the target page. `--img`
writes an `.img` file instead of a `.cbi` include. `--unpacked` keeps the
image data unpacked so the image effects can use it.

## Cell

The cell API reads and writes screen cells directly. A raw cell value is
the target's own code for the character shape shown in that cell. Use
this API for text based games and custom character sets.

`CELL_PRINT x, y, s$` and `PRINT_AT col, row, s$` take the same
coordinates but are not the same call. `PRINT_AT` is ordinary text
output moved to a position, so it follows whatever the target's text
output does, including scrolling and the cursor, and it needs
`TEXT_OUTPUT_SUPPORTED`. `CELL_PRINT` writes the cells straight into the
grid, moves no cursor, never scrolls, and needs
`CELL_SURFACE_SUPPORTED`. Prefer `CELL_PRINT` when you are placing text
on a screen you are drawing yourself, and in graphics or mixed modes,
where text output may not be available at all.

| Call                                           | What it does                                  |
| ---------------------------------------------- | --------------------------------------------- |
| `CELL_DRAW(x, y, w, h, addr)`                  | Copy a block of cell data to the screen.      |
| `CELL_MEMMOVE(src_x, src_y, dst_x, dst_y, count)` | Move `count` consecutive cells within the screen. |
| `CELL_SCROLL_ROW(row, x, w, count, fill, direction)` | Scroll one row segment left or right. |
| `CELL_SCROLL(x, y, w, h, count, fill, direction)` | Scroll a rectangle left, right, up, or down. |
| `CELL_CLS()`                                   | Clear the cell screen.                         |
| `CELL_GETC(x, y)`                              | Read one cell when the target can.            |
| `CELL_PUTC(x, y, c)`                           | Write one raw cell.                           |
| `CELL_PUTC(x, y, c, color, attr)`              | Write one raw cell, color, and attributes.    |
| `CELL_PRINT(x, y, s$)`                         | Write a string starting at one cell.          |
| `CELL_PRINT(x, y, s$, color, attr)`            | Write a string, color, and attributes.        |
| `CELL_CODE(c)` / `CELL_CODE(s$)`               | Convert a printable byte or string to a raw cell code. |
| `CELL_COLORS(fg)`                              | Set default/shared cell foreground, preserving the background. |
| `CELL_COLORS(fg, bg)`                          | Set default/shared cell foreground and background colors. |
| `CELL_COLORS(fg, bg, color2, color3)`          | Set four cell color slots where supported.    |
| `CELL_COLOR(x, y, color)`                      | Set cell color where the target supports it.  |
| `CELL_ATTRIB(x, y, attr)`                      | Set cell attributes where supported.          |
| `CELL_FLUSH()`                                 | Apply any cell changes still waiting to appear. |

Cell values are screen codes, and the codes differ by target. Some
targets let each cell have its own color and attributes, while others
share them across a larger area. `CELL_CLS` clears the cell screen
without moving the text cursor.

Use `CELL_PUTC x, y, CELL_CODE("O")` for a printable character, or
`CELL_PRINT x, y, "HELLO"` for a string.

The five argument `CELL_PUTC` sets the cell, color, and attributes.
`CELL_PRINT` does the same for every cell in the string.

`CELL_MULTICOLOR` uses bit pairs for four color choices per pixel.
In this mode, `CELL_COLORS fg, bg, color2, color3` sets
the default foreground, shared background, and two shared multicolor
values. `CELL_COLOR x, y, color` changes one cell's foreground. The
other three colors remain shared.

Cell support constants:

| Constant | Meaning |
| --- | --- |
| `CELL_SURFACE_SUPPORTED` | `TRUE` when the target has a cell screen addressed by column and row. |
| `CELL_PIXEL_W` / `CELL_PIXEL_H` | Cell size in pixels. |
| `CELL_COLOR_SUPPORTED` | `CELL_COLORS` and `CELL_COLOR` have target support. |
| `CELL_COLORS_PER_CELL` | Maximum simultaneous colors per cell supported by the cell interface, including background; `0` without a cell surface. Monochrome cells count as `2`. |
| `CELL_COLOR_MODEL` | How cell colors can be selected. |
| `CELL_COLOR_W` / `CELL_COLOR_H` | Width and height, in cells, that share one color setting. |
| `CELL_MULTICOLOR_SUPPORTED` | `DISPLAY(CELL_MULTICOLOR)` is supported. |
| `CELL_ATTRIB_SUPPORTED` | `CELL_ATTRIB` has target support. |
| `CELL_ATTRIB_W` / `CELL_ATTRIB_H` | Width and height, in cells, that share one attribute setting. |

Cell color model constants:

| Constant | Meaning |
| --- | --- |
| `CELL_COLOR_MODEL_NONE` | No portable cell color. |
| `CELL_COLOR_MODEL_SHARED` | Shared/global cell colors only. |
| `CELL_COLOR_MODEL_PER_CELL_FG` | Each cell has its own foreground and shares the background. |
| `CELL_COLOR_MODEL_PER_CELL_BG` | Each cell has its own background and shares the foreground. |
| `CELL_COLOR_MODEL_PER_CELL_FG_BG` | Each cell has its own foreground and background pair. |
| `CELL_COLOR_MODEL_PER_CELL_PALETTE` | Each cell selects a palette or color set. |
| `CELL_COLOR_MODEL_BLOCK_PALETTE` | A block of cells shares one palette or color set. |
| `CELL_COLOR_MODEL_PER_PIXEL` | Each pixel has its own color. |

The available attributes are `NORMAL`, `INVERSE`, `ITALIC`, and
`BLINKING`.
Scroll directions are `CELL_SCROLL_LEFT`, `CELL_SCROLL_RIGHT`,
`CELL_SCROLL_UP`, and `CELL_SCROLL_DOWN`. `CELL_MEMMOVE` follows the
target's cell layout. Use it for one row unless the target page says a
larger range is safe.

## Charset

### Terms

These terms have separate meanings throughout the documentation:

- A glyph is one character shape.
- A font is a collection of glyphs mapped to text characters. It may also
  include spacing information for software text drawing.
- A charset is the character table currently used by a target's cell
  display.
- A charmap is a rectangular layout of character numbers from a charset.
- A tileset is game artwork addressed by tile number. It may use the same
  kind of bitmap data as glyphs without being a text font.
- A tilemap is a rectangular layout of tile numbers from a tileset.

A font can supply glyphs for software drawing or for a hardware charset. The
charset API only controls the target's active character table. Use it to
replace the complete table or individual glyphs.

The `.cbfont` format and `@INCLUDE_FONT` describe portable font assets. They
do not select the active charset.

### Importing a font

Use `@INCLUDE_FONT` when the source describes glyphs mapped to text
characters:

```basic
@INCLUDE_FONT UI_FONT "assets/ui.cbfont"
@INCLUDE_FONT LARGE_FONT "LARGE.FNT" "assets/large.cbfont"
```

The one-path form embeds `UI_FONT_DATA`. The two-path form stages the packed
glyph data and creates `LARGE_FONT_LOAD`, `LARGE_FONT_LOAD(dst)`, and
`LARGE_FONT_ADDR`. Font dimensions, spacing, character mapping, and width
metadata remain in the program in both forms.

### Importing a charset

Use `@INCLUDE_CHARSET` to import character artwork and prepare it for the
selected target:

```basic
@INCLUDE_CHARSET GAME_CHARS "assets/game.characters"
@INCLUDE_CHARSET OTHER_CHARS "OTHER.CHR" "assets/other.characters"

PROC MAIN
	CHARSET_COPY_DEFAULT
	CHARSET_INSTALL_RANGE U8(32), GAME_CHARS_COUNT, ADDR GAME_CHARS_DATA

	IF OTHER_CHARS_LOAD THEN
		CHARSET_INSTALL_RANGE U8(64), OTHER_CHARS_COUNT, OTHER_CHARS_ADDR
	ENDIF
ENDPROC
```

The source artwork is converted into the selected target's character layout.
Supported source formats and target packing are documented on the target
pages.

For `@INCLUDE_CHARSET GAME_CHARS "assets/game.characters"`, crustyBASIC
creates:

| Name | Meaning |
| --- | --- |
| `GAME_CHARS_WIDTH` / `GAME_CHARS_HEIGHT` | Target cell size. |
| `GAME_CHARS_COUNT` | Number of imported glyphs. |
| `GAME_CHARS_BYTES_PER_ENTRY` | Packed bytes for each target glyph. |
| `GAME_CHARS_DATA_LAST` | Last index in `GAME_CHARS_DATA`. |
| `GAME_CHARS_DATA` | Target packed glyph bytes. |
| `GAME_CHARS_COLORS` | Foreground metadata prepared for each target glyph. |
| `GAME_CHARS_PROPERTIES` | Imported property value for each glyph. |
| `GAME_CHARS_PIXEL_WIDTHS` | Logical pixel width for each source glyph. |
| `GAME_CHARS_BACKGROUND` / `GAME_CHARS_SHARED_COLOR_1` / `GAME_CHARS_SHARED_COLOR_2` | Source project colors. |
| `GAME_CHARS_PALETTE` | Source palette as red, green, blue byte triples. |

This generated form is independent of the source tool. Future importers can
produce the same names and target packed data.

The two-path form stages the packed glyph bytes instead of embedding
`LABEL_DATA`. It creates `LABEL_LOAD`, `LABEL_LOAD(dst)`, and `LABEL_ADDR`.
The no-argument load uses an automatically allocated buffer. The address
form loads into memory chosen by the program. `LABEL_ADDR` returns the
address used by the most recent successful load. Glyph metadata remains in
the program.

### Importing a charmap

Use `@INCLUDE_CHARMAP` to import the layout from the same project:

```basic
@INCLUDE_CHARSET LEVEL_CHARS "assets/level.characters"
@INCLUDE_CHARMAP LEVEL_MAP "assets/level.characters"
@INCLUDE_CHARMAP OTHER_MAP "OTHER.MAP" "assets/other.characters"
```

A charset contains the character artwork. A charmap contains only the
character numbers placed in each row and column. Keeping them separate lets a
map use target packed charset data without making the map target specific.

`@INCLUDE_CHARMAP` imports a direct character layout. Use the tileset and
tilemap directives below when the map uses reusable tiles.

For `@INCLUDE_CHARMAP LEVEL_MAP "assets/level.characters"`, crustyBASIC
creates:

| Name | Meaning |
| --- | --- |
| `LEVEL_MAP_WIDTH` / `LEVEL_MAP_HEIGHT` | Map size in character cells. |
| `LEVEL_MAP_COUNT` | Number of cells in the map. |
| `LEVEL_MAP_DATA_LAST` | Last index in `LEVEL_MAP_DATA`. |
| `LEVEL_MAP_DATA` | Row major character numbers as `U16` values. |

The values are kept as `U16` so maps can refer to more than 256 characters. A
target with an 8 bit character table can convert a value after placing its
imported charset in that target's available range.

The two-path form stages the row major `U16` data and creates the same
`LABEL_LOAD`, `LABEL_LOAD(dst)`, and `LABEL_ADDR` interface used by other
memory backed imports.

### Importing a tileset and tilemap

`@INCLUDE_TILESET` imports reusable tile pictures. `@INCLUDE_TILEMAP`
imports the layout that says where those pictures appear. Import them
separately when the artwork and layout are separate files:

```basic
@INCLUDE_TILESET LEVEL_TILES "assets/level.png" AS png_a7800_8x8
@INCLUDE_TILEMAP LEVEL_MAP "assets/level.cbm" USING LEVEL_TILES
```

The PNG supplies the pictures, not the placement map. The `.cbm` file supplies
the layout and names its tileset with `USING`. A PNG tileset does not create
`*_MAP_*` values automatically.

The portable `.cbm` format is JSON:

```json
{
  "width": 4,
  "height": 2,
  "tiles": [
    0, 1, 0, 2,
    2, 0, 1, 0
  ]
}
```

`width` and `height` are required and must be from 1 through 255. `tiles` is
a flat row major list: left to right, then top to bottom. It must contain
exactly `width * height` indexes, and every index must be from 0 through 255.
Line breaks and indentation do not define the rows; they are only for
readability.

A `.cbm` file contains only those three fields. It does not contain BASIC
syntax such as `CONST`, `#`, or `AS U8`, and the directive does not use `AS`.
The `.cbm` extension selects the format.

The indexes in a `.cbm` file are local to the tileset named by `USING`.
Index 0 selects its first tile, index 1 selects its second tile, and so on.
The named tileset must be an embedded `@INCLUDE_TILESET`. crustyBASIC checks
that every index exists, then adds the tileset's assigned base to each value.
For example, if `LEVEL_TILES_BASE` is 16, local indexes `0, 1, 2` become tile
IDs `16, 17, 18` in `LEVEL_MAP_DATA`.

PNG tile formats scan the picture in tile sized cells from left to right and
top to bottom. Each new converted picture receives the next local index.
Matching pictures reuse the earlier index. A map editor can use those local
indexes when writing the `.cbm` file.

Some editor formats contain both pictures and a layout. Import each part from
the same source file in that case. `USING` is not needed because the shared
path identifies the matching tileset. For example, a C64 CharPad project can
be imported with:

```basic
@INCLUDE_TILESET LEVEL_TILES "assets/level.ctm"
@INCLUDE_TILEMAP LEVEL_MAP "assets/level.ctm"
```

The tilemap is still imported explicitly. The format importer reads the
editor's layout and converts it to the same row major `U8` interface used by
`.cbm` maps.

Embedded `@INCLUDE_TILESET` directives also generate the active tile count.
The count covers the complete assigned range across all imported tilesets.
An explicit `@OPTION TILE_ACTIVE_COUNT` wins and must be large enough for
that range. Runtime file imports do not change the active count.

Each target declares the tile project formats it accepts. `AS` may be omitted
when an accepted editor format identifies itself. Some PNG formats require an
explicit `AS`, as shown in the Atari 7800 example above.

For `@INCLUDE_TILESET LEVEL_TILES "assets/level.png" AS png_a7800_8x8`,
crustyBASIC creates:

| Name | Meaning |
| --- | --- |
| `LEVEL_TILES_WIDTH` / `LEVEL_TILES_HEIGHT` | Picture size of one tile in pixels. |
| `LEVEL_TILES_COUNT` | Number of tiles. |
| `LEVEL_TILES_BYTES_PER_TILE` | Packed bytes in one tile picture. |
| `LEVEL_TILES_PALETTE_BYTES_PER_TILE` | Color bytes kept for each tile, or zero. |
| `LEVEL_TILES_DATA_LAST` | Last index in `LEVEL_TILES_DATA`. |
| `LEVEL_TILES_DATA` | Target packed `U8` pictures in tile number order. |
| `LEVEL_TILES_PALETTE_DATA_LAST` | Last color data index when the format keeps tile colors. |
| `LEVEL_TILES_PALETTE_DATA` | Target color data in tile number order when present. |
| `LEVEL_TILES_BASE` | First tile ID assigned to this embedded tileset. |

For `@INCLUDE_TILEMAP LEVEL_MAP "assets/level.cbm" USING LEVEL_TILES`,
crustyBASIC creates:

| Name | Meaning |
| --- | --- |
| `LEVEL_MAP_WIDTH` / `LEVEL_MAP_HEIGHT` | Map size in base tile cells. |
| `LEVEL_MAP_COUNT` | Number of map cells. |
| `LEVEL_MAP_DATA_LAST` | Last index in `LEVEL_MAP_DATA`. |
| `LEVEL_MAP_DATA` | Row major assigned tile IDs as `U8` values. |

The generated map can be passed directly to the tile API:

```basic
PROC MAIN
	TILE_BEGIN 1, 1
	TILE_BLIT ADDR LEVEL_MAP_DATA, 0, 0, LEVEL_MAP_WIDTH, LEVEL_MAP_HEIGHT
ENDPROC
```

Embedded tilesets receive permanent, nonoverlapping tile IDs in import order.
Maps imported with `USING`, or from the same editor project, use those IDs.
`TILE_BEGIN` binds the embedded pictures, so no setup PROC is needed. Runtime
file imports stay manual because their picture data is not in memory until
the program loads it.

The two-path forms stage the packed `U8` data and create
`LABEL_LOAD`, `LABEL_LOAD(dst)`, and `LABEL_ADDR`. Dimensions and counts
remain in the program.

### Runtime support

`CHARSET_SUPPORTED` is `TRUE` when the complete charset API can be used.
`CHARSET_DEFINE_SUPPORTED` is `TRUE` when the target instead supports changing
individual characters. Only one of these values is `1`. A program that only
needs a custom character can start with:

```basic
@REQUIRES CHARSET_SUPPORTED OR CHARSET_DEFINE_SUPPORTED @ELSE "custom character support is required"
```

| Call | Required support | What it does |
| --- | --- | --- |
| `CHARSET_COPY_DEFAULT()` | `CHARSET_SUPPORTED` | Make an editable copy of the target's normal charset and activate it. |
| `CHARSET_INSTALL(src_addr)` | `CHARSET_SUPPORTED` | Copy and activate a complete charset. |
| `CHARSET_DEFINE(code, addr)` | Either support value | Replace one glyph in the active charset. |
| `CHARSET_DEFINE(s$, addr)` | Either support value | Replace the glyph for the first printable character in `s$`. |
| `CHARSET_INSTALL_RANGE(first, count, src_addr)` | Either support value | Replace consecutive glyphs without changing the rest of the charset. |
| `CHARSET_RESET()` | `CHARSET_SUPPORTED` | Switch back to the target's normal charset. |

`CHARSET_MAPPED` defaults to `TRUE`: display hardware maps screen character
codes through the charset, so font changes update existing text. When
`FALSE`, drawing text copies glyph pixels into the screen bitmap; redraw
the text after changing the font. This value does not imply charset API
support.

### Installing character data

Call `CHARSET_COPY_DEFAULT()` before changing only a few characters. It
copies the target's normal glyphs into an editable charset and switches the
display to that copy.

`CHARSET_INSTALL` always copies
`CHARSET_NUM_ENTRIES * CHARSET_BYTES_PER_ENTRY` bytes, so you do not
pass a byte count. You can install complete embedded charset data directly:

```basic
@INCLUDE_FILE CHARSET_DATA "{program_name}/charset.bin"
CHARSET_INSTALL ADDR CHARSET_DATA
```

Without `COUNT`, `@INCLUDE_FILE` embeds the whole file.
`CHARSET_DEFINE("!", ADDR GLYPH)` changes the same character as
`CHARSET_DEFINE(CELL_CODE("!"), ADDR GLYPH)`. Both forms return the cell
code they changed. Use a number when the character position has no
printable character.

`CHARSET_INSTALL_RANGE` reads `count * CHARSET_BYTES_PER_ENTRY` bytes and
places them at consecutive character numbers starting with `first`. Call
`CHARSET_COPY_DEFAULT` first on targets with `CHARSET_SUPPORTED` when the
other characters should keep their normal shapes. `first + count` must stay
within `CHARSET_NUM_ENTRIES`.

### Custom characters

Use `CHARSET_CUSTOM_FIRST` when a program needs one custom character.
This example replaces the first custom character with a diamond and
draws it:

```basic
CONST GLYPH[7] AS U8 = %
	...XX...
	..XXXX..
	.XXXXXX.
	XXXXXXXX
	XXXXXXXX
	.XXXXXX.
	..XXXX..
	...XX...
END%

@IF CHARSET_SUPPORTED THEN
	CHARSET_COPY_DEFAULT
@ENDIF
CHARSET_DEFINE CHARSET_CUSTOM_FIRST, ADDR GLYPH
CELL_PUTC 10, 10, CHARSET_CUSTOM_FIRST
```

For several custom characters, use consecutive character numbers from
`CHARSET_CUSTOM_FIRST` through `CHARSET_CUSTOM_LAST`.
`CHARSET_CUSTOM_COUNT` is the number of characters in that range.

Run `crustybasic target-info <system>` to see which character numbers
these names use and how the default characters are arranged. The target's
default character map identifies the character numbers used for printable
text. Its custom range identifies the entries that libraries and programs
may replace.

| Constant | Meaning |
| --- | --- |
| `CHARSET_NUM_ENTRIES` | Number of character shapes. |
| `CHARSET_BYTES_PER_ENTRY` | Number of bytes in each character shape. |
| `CHARSET_CUSTOM_FIRST` / `CHARSET_CUSTOM_LAST` | First and last character numbers to use for custom shapes. |
| `CHARSET_CUSTOM_COUNT` | Number of character numbers from `CHARSET_CUSTOM_FIRST` through `CHARSET_CUSTOM_LAST`. |

## Tile

The TILE API draws a playfield using a character screen, bitmap
graphics, or the target's own tile display. Its coordinates are tile
positions, not pixels.

```basic
DISPLAY TILE_DEFAULT_DISPLAY_MODE
TILE_BEGIN 2, 2
TILE_ALIAS 0, " "
TILE_ALIAS 1, "#"
TILE_CLEAR_TILE 0
TILE_BOX 0, 0, TILE_COLUMNS - 1, TILE_ROWS - 1, 1
```

Most programs select a display mode and then call
`TILE_BEGIN span_w, span_h`. `span_w` and `span_h` say how many 8x8 base
units make up one game tile. Use `1, 1` for one base unit or larger
values for block tiles.

Some targets need a character or tile memory address. On those targets,
use `TILE_BEGIN(span_w, span_h, base)`. Leave out `base` unless the
target page asks for it.

TILE can draw in four ways:

| `@OPTION` value | Command line value | Meaning |
| --- | --- | --- |
| `TILE_CELL` | `cell` | Draw on the target's character screen. |
| `TILE_HARDWARE` | `hardware` | Use the target's built in tile display. |
| `TILE_BITMAP` | `bitmap` | Draw tile pictures in a bitmap graphics mode. |
| `TILE_KERNEL` | `kernel` | Use a target specific tile mode with its own limits. |

To use bitmap tiles:

```basic
@OPTION TILE_BACKEND TILE_BITMAP
```

From the command line, use `--set tile-backend=bitmap`. If neither form
is used, the target chooses its preferred way. An unsupported choice is
a compile error.

`TILE_BEGIN` does not change the display mode. For portable code, call
`DISPLAY TILE_DEFAULT_DISPLAY_MODE` first. You can also select a target
specific mode before `TILE_BEGIN`.

The default TILE span is 1 by 1. Programs using that size need no span
options:

```basic
TILE_BEGIN 1, 1
```

When every `TILE_BEGIN` uses another size, declare that size with both
span options:

```basic
@OPTION TILE_FIXED_SPAN_W 2
@OPTION TILE_FIXED_SPAN_H 2
TILE_BEGIN 2, 2
```

When a program uses more than one span, set both options to zero:

```basic
@OPTION TILE_FIXED_SPAN_W 0
@OPTION TILE_FIXED_SPAN_H 0
TILE_BEGIN WIDTH, HEIGHT
```

Fixed span values must match every `TILE_BEGIN` call in the program. A
mismatch may draw the wrong tiles.

If a program uses only a small range of tile IDs, it can set the number
of active IDs:

```basic
@OPTION TILE_ACTIVE_COUNT 4
```

Without embedded tilesets, the default is `TILE_MAX_COUNT`, normally 64,
though some targets offer less. Embedded tilesets generate the exact count
needed for their assigned tile range. A value of `N` allows tile IDs `0`
through `N - 1` and may use less memory. An explicit value overrides the
generated count and must still hold every imported tile. Do not use tile ID
`N` or higher.

Tile drawing:

| Call                                  | What it does                                |
| ------------------------------------- | ------------------------------------------- |
| `TILE_PLOT(x, y, tile)`               | Draw one game tile.                         |
| `TILE_CLEAR(x, y)`                    | Draw the selected clear tile.               |
| `TILE_PRINT(x, y, text$)`             | Draw text starting at a tile position.      |
| `TILE_HLINE(x0, x1, y, tile)`         | Draw a horizontal tile run.                 |
| `TILE_UNHLINE(x0, x1, y)`             | Clear a horizontal tile run.                |
| `TILE_VLINE(x, y0, y1, tile)`         | Draw a vertical tile run.                   |
| `TILE_UNVLINE(x, y0, y1)`             | Clear a vertical tile run.                  |
| `TILE_BOX(x0, y0, x1, y1, tile)`      | Draw a tile rectangle outline.              |
| `TILE_FILLBOX(x0, y0, x1, y1, tile)`  | Fill a tile rectangle.                      |
| `TILE_CLS()`                          | Clear the tile area.                        |

`TILE_PRINT` uses tile coordinates and does not clip text, so keep the
string inside the tile area. It uses the target's TILE font, which is
normally 8 by 8 pixels.

Additional font characters:

| Name | Code | Use |
| --- | --- | --- |
| `TILE_FONT_SPADE` | `91` | `CHR(TILE_FONT_SPADE)` |
| `TILE_FONT_HEART` | `92` | `CHR(TILE_FONT_HEART)` |
| `TILE_FONT_DIAMOND` | `93` | `CHR(TILE_FONT_DIAMOND)` |
| `TILE_FONT_CLUB` | `94` | `CHR(TILE_FONT_CLUB)` |

Tile assets:

| Call                                  | What it does                                |
| ------------------------------------- | ------------------------------------------- |
| `TILE_ALIAS(id, code)` / `TILE_ALIAS(id, s$)` | Map a tile ID to a tile or cell code. |
| `TILE_DEFINE(id, addr)`               | Install a picture for one tile ID when supported. |
| `TILE_PALETTE(id, addr)`              | Assign target palette data to one tile ID. |
| `TILE_BLOCK_DEFINE(id, w, h, addr)`   | Name a rectangular block of tile IDs stored one row after another. |
| `TILE_BLIT(block, x, y)`              | Draw a named block.                         |
| `TILE_BLIT(addr, x, y, w, h)`         | Draw a block directly from memory.          |
| `TILE_CLEAR_TILE(id)`                 | Select the clear tile ID.                   |
| `TILE_COLOR(id, fg, bg)`              | Set TILE colors when available.             |
| `TILE_DEFAULT_DISPLAY_MODE()`         | Return the portable `DISPLAY` mode for the current TILE choice. |

`TILE_DEFINE` takes a game tile ID. On cell displays, valid IDs run
from `0` through `CHARSET_CUSTOM_COUNT - 1`. `TILE_ACTIVE_COUNT` must
also include every ID used. Ordinary text characters remain unchanged.

Constant tile pictures use the same portable 8 byte shape as visual binary
data:

```basic
CONST ROCK_GLYPH[7] AS U8 = %
...XX...
..XXXX..
.XXXXXX.
XXXXXXXX
XXXXXXXX
.XXXXXX.
..XXXX..
...XX...
END%

PROC SETUP_TILES
	TILE_DEFINE TILE_ROCK, ADDR ROCK_GLYPH
ENDPROC
```

The ID remains the number used by `TILE_PLOT` and the other TILE calls. Set
`TILE_ACTIVE_COUNT` high enough to include every tile ID used. The target may
use the shape directly, convert it at runtime, or pack constant data into its
native layout while building the program.

`TILE_PALETTE` associates a tile ID with a target palette record. The target's
tile format defines that record, so it can hold the number and kind of colors
the hardware needs. Imported color tilesets can supply these calls
automatically; hand written tiles can use `TILE_PALETTE id, ADDR palette`.

`TILE_CUSTOM_SHAPE_SUPPORTED` covers constant `U8` shapes. A mutable array requires
`TILE_CUSTOM_SHAPE_DYNAMIC_SUPPORTED`. A target that packs constant shapes still
runs the call to bind the tile ID, so put it in the normal setup code. Both
capabilities describe the selected `TILE_BACKEND`, not every backend offered by
the target.

Tile support and size values:

| Name / call                    | What it means                                   |
| ------------------------------ | ----------------------------------------------- |
| `TILE_SUPPORTED`               | Tile drawing is available.                      |
| `TILE_CELL_SUPPORTED`          | Text/cell screen drawing is available.          |
| `TILE_HARDWARE_SUPPORTED`      | The target's built in tile display is available. |
| `TILE_BITMAP_SUPPORTED`        | Bitmap drawing mode exists.                     |
| `TILE_KERNEL_SUPPORTED`        | A target specific tile mode is available.       |
| `TILE_CUSTOM_SHAPE_SUPPORTED` | `TILE_DEFINE` accepts constant tile data.       |
| `TILE_CUSTOM_SHAPE_DYNAMIC_SUPPORTED` | `TILE_DEFINE` accepts mutable tile data. |
| `TILE_BLOCK_SUPPORTED`         | `TRUE` when tile blocks can be defined and drawn.  |
| `TILE_COLOR_AVAILABLE`         | `TRUE` when `TILE_COLOR` works in the current tile mode. |
| `TILE_BACKEND`                 | Current `TILE_CELL`, `TILE_HARDWARE`, `TILE_BITMAP`, or `TILE_KERNEL` choice. |
| `TILE_MAX_COUNT`               | Default number of available tile IDs.           |
| `TILE_BASE_PIXEL_W` / `TILE_BASE_PIXEL_H` | Base tile unit size in pixels.                |
| `TILE_SPAN_W` / `TILE_SPAN_H`  | Base units occupied by one game tile.           |
| `TILE_SPAN_PIXEL_W` / `TILE_SPAN_PIXEL_H` | Complete game tile size in pixels.            |
| `TILE_COLUMNS` / `TILE_ROWS`   | Width and height of the tile area.              |

With `TILE_BITMAP`, `TILE_BLIT` block data contains tile IDs and uses
the matching pictures and colors. `TILE_KERNEL` may support only part of
the TILE API. The target pages list those limits.

`TILE_COLOR_AVAILABLE` tells you whether `TILE_COLOR` works. With
`TILE_BITMAP`, its value depends on the current `DISPLAY` mode, so check
it afterward with a normal `IF`. It cannot be used with `@IF` or
`@REQUIRES`.

## Input

| Name / call              | What it does                                        |
| ------------------------ | --------------------------------------------------- |
| `INPUT(INOUT value)`     | Read one line and return `1` if it was assigned, or `0`. |
| `KEY()`                  | Wait for a typed key and return its `U8` code.        |
| `INKEY()`                | Return a typed key as a string immediately, or `""`. |
| `INKEY_CODE()`           | Return a typed key code immediately, or `0`.         |
| `RAWKEY()`               | Return a held key as a string immediately, or `""`.  |
| `RAWKEY_CODE()`          | Return a held key code immediately, or `0`.          |
| `KEY_HELD(code)`         | `1` while the requested key is down, or `0`.        |
| `KEYBOARD_SUPPORTED`           | `TRUE` when keyboard input is available.             |
| `KEY_HELD_SUPPORTED`           | `TRUE` when individual held keys can be checked.      |
| `KEYPAD_SUPPORTED`             | `TRUE` when keypad input is available.                |
| `JOYSTICK_SUPPORTED`           | `TRUE` when joystick input is available.              |
| `JOYSTICK_BUTTONS_SUPPORTED`   | `TRUE` when joystick buttons are available.           |
| `ANALOG_JOYSTICK_SUPPORTED`    | `TRUE` for analog joysticks.                          |
| `DIGITAL_JOYSTICK_SUPPORTED`   | `TRUE` for digital joysticks.                         |
| `PADDLES_SUPPORTED`            | `TRUE` when `PADDLE(axis)` has supported axes.        |
| `MOUSE_SUPPORTED`              | `TRUE` when mouse input is available.                 |
| `INPUT_ANY()`            | `1` when a key, keypad, joystick, or button is active. |
| `INPUT_ANY(wait)`        | `INPUT_ANY()`, but blocks first when `wait` is `1`. |
| `INPUT_JOY_ENABLE(port, enabled)` | Include or exclude a joystick port in `INPUT_ANY` and `INPUT_CLEAR`. |
| `INPUT_CLEAR()`          | `1` when no key, keypad, joystick, or button is active. |
| `INPUT_CLEAR(wait)`      | `INPUT_CLEAR()`, but blocks first when `wait` is `1`. |
| `JOY(port)`              | Direction bits from a joystick port numbered from zero. |
| `JOY_HORIZ(port)`        | Left or right direction bits from a joystick port numbered from zero. |
| `JOY_BUTTON(port, button)`   | `1` if pressed, `0` otherwise.                      |
| `INPUT_JOY_KEYMAP(port, button, up, down, left, right, button_key)` | Combine a joystick with optional held keys. |
| `JOY_SET_DEADZONE(value)` | Set the analog joystick dead zone.                |
| `PADDLE(axis)`           | Raw analog axis value.                              |
| `KEYPAD_CODE(port)`      | Raw keypad code, or `KEYPAD_NONE`.                  |

Input constants:

| Name                    | What it means                                       |
| ----------------------- | --------------------------------------------------- |
| `JOY_SELECTION_REQUIRED` | `TRUE` when joystick ports must be enabled explicitly for input prompts. |
| `JOY_PORTS`             | Number of joystick ports for `JOY(port)`.           |
| `JOY_DEFAULT_PORT`      | Default joystick port for gameplay input.           |
| `JOY_BUTTONS`           | Buttons per joystick port for `JOY_BUTTON(port, button)`. |
| `INPUT_JOY_BUTTON`      | Button bit returned by `INPUT_JOY_KEYMAP`.          |
| `JOY_DEADZONE_DEFAULT`  | Initial analog joystick dead zone.                  |
| `PADDLE_AXES`           | Number of analog axes for `PADDLE(axis)`.           |
| `ANALOG_AXIS_PAIR_NAME` | Name for an X/Y analog axis pair.                   |
| `MOUSE_BUTTONS`         | Number of mouse buttons; `0` when none.             |
| `KEYPAD_PORTS`          | Number of keypad ports for `KEYPAD_CODE(port)`.     |
| `KEYPAD_KEYS`           | Number of keypad key codes, excluding `KEYPAD_NONE`. |
| `KEYPAD_FUNCTION_KEYS`  | Number of nonnumeric keypad key codes.              |
| `KEYPAD_TEXT_INPUT_SUPPORTED` | `TRUE` when keypad numeric `INPUT` is available. |
| `KEYPAD_STRING_INPUT_SUPPORTED` | `TRUE` when keypad string `INPUT` is available. |
| `KEY_SPACE`               | ASCII keyboard space code.                         |
| `KEY_0` through `KEY_9` | ASCII keyboard digit codes.                         |
| `KEY_A` through `KEY_Z` | Uppercase ASCII keyboard letter codes.              |
| `KEY_LOWER_A`, `KEY_LOWER_Z`, `KEY_LOWER_TO_UPPER_DELTA` | Values used to convert lowercase ASCII letters to uppercase. |

`KEYPAD_CODE(port)` returns `KEYPAD_NONE`, `KEYPAD_0` through
`KEYPAD_9`, `KEYPAD_ASTERISK`, `KEYPAD_POUND`, `KEYPAD_START`,
`KEYPAD_PAUSE`, or `KEYPAD_RESET`. Missing keypad ports return
`KEYPAD_NONE`.

Keyboard constants `KEY_SPACE`, `KEY_0` through `KEY_9`, and `KEY_A`
through `KEY_Z` are useful with `KEY()`, `INKEY_CODE()`, and
`RAWKEY_CODE()` on targets that report ASCII character codes.
Targets also define constants such as `KEY_ENTER`, `KEY_BACKSPACE`,
`KEY_DELETE`, `KEY_ESCAPE`, `KEY_TAB`, cursor keys, and function keys
when available. Some targets add `RAWKEY_*` constants for keys whose raw
codes differ from their typed character codes.

Test joysticks against `JOY_UP`, `JOY_DOWN`, `JOY_LEFT`, `JOY_RIGHT` with bitwise `&`:

```basic
IF JOY(0) & JOY_LEFT <> 0 THEN ...
```

Missing joystick ports and buttons return 0. On analog joystick targets,
`JOY` still returns the shared direction bits.

`JOY_HORIZ(port)` returns a mask containing only the `JOY_LEFT` and
`JOY_RIGHT` direction bits. Use it when a game does not need vertical
movement. Analog targets may avoid reading the vertical axis.

`INPUT_JOY_KEYMAP` returns the `JOY_UP`, `JOY_DOWN`, `JOY_LEFT`, and
`JOY_RIGHT` bits from the selected joystick and mapped keys. It adds
`INPUT_JOY_BUTTON` when the selected joystick button or mapped button
key is held. Use `KEY_NONE` for any key that should not be mapped. Key
arguments are ignored when `KEY_HELD_SUPPORTED` is `FALSE`.

Analog targets start with `JOY_DEADZONE_DEFAULT`; larger
`JOY_SET_DEADZONE(n)` values ignore more center drift, smaller values
make the stick more sensitive, and digital targets accept the call
without changing state.

`MOUSE_BUTTONS` reports the button count, not the current button state.
Mouse position and button state are target specific.

`KEY()`, `INKEY()`, and `INKEY_CODE()` return typed characters rather
than keys that are currently held down.
`KEY()` blocks until a character is available. `INKEY()` and
`INKEY_CODE()` return immediately. Use them for prompts, menus, and text
input.

`RAWKEY()` and `RAWKEY_CODE()` are for checking game controls. They
report a held key on every poll when the target can read the keyboard
directly. On other targets, raw key input may match `KEY()`, `INKEY()`,
and `INKEY_CODE()`.

`KEY_HELD(code)` checks one key without blocking or consuming typed
input. It accepts `KEY_SPACE`, `KEY_0` through `KEY_9`, and `KEY_A`
through `KEY_Z`. Letter codes identify the key regardless of Shift or
character case. Check `KEY_HELD_SUPPORTED` before using it. Some
keyboards cannot detect every combination of held keys.

`INPUT_ANY()` is for "press anything" prompts. It checks held keys
without consuming a character waiting for `INKEY`. It also checks
keypads and enabled joystick ports. Pass `0` to poll and `1` to wait.
To prevent an already held button from carrying into a prompt,
wait for clear input before waiting for new input:

```basic
INPUT_CLEAR 1
INPUT_ANY 1
```

`INPUT_JOY_ENABLE(port, enabled)` controls which joystick ports participate
in `INPUT_ANY` and `INPUT_CLEAR`. Ports start disabled when
`JOY_SELECTION_REQUIRED` is `TRUE`, otherwise they start enabled.
Keyboard and keypad input remain available. Direct `JOY`, `JOY_BUTTON`,
and `INPUT_JOY_KEYMAP` reads are unaffected.

After the player selects a joystick interface:

```basic
INPUT_JOY_ENABLE PORT, TRUE
INPUT_ANY 1
INPUT_JOY_ENABLE PORT, FALSE
```

`INPUT(value)` reads and echoes one line. The keyboard's Backspace or
Delete key erases the previous character. The type of `value` tells it
whether to read a number or a string. It returns `1` after storing a
value. Invalid numeric text, numeric overflow, or text that does not fit
returns `0` and leaves `value` unchanged. The call does not print an
error or try again.

On a keypad, `INPUT` supports unsigned integer variables. Targets with
`KEYPAD_STRING_INPUT_SUPPORTED` also support strings. Press the target's
keypad key used to accept input. The target pages list keypad details
and input counts.

## Game helpers

These helpers are for portable game code.

### LFSR

LFSR provides fast, repeatable random sequences for games. Starting
with the same seed produces the same sequence.

| Call              | Returns       | What it does                                  |
| ----------------- | ------------- | --------------------------------------------- |
| `LFSR_SEED(seed)` | none          | Seed the byte or word generator.              |
| `LFSR_NEXT()`     | `U8` or `U16` | Next repeatable random byte or word.          |
| `LFSR_RANGE(hi)`  | `U8`          | Repeatable random value from 0 through `hi`.  |

`LFSR_SEED` and `LFSR_NEXT` have `U8` and `U16` forms. The seed or result
type chooses the form. Use `U8(...)` or `U16(...)` when a literal leaves
the type unclear. The two forms keep separate sequences. `LFSR_RANGE`
uses the `U8` form.

### Collision

| Call                                                   | Returns | What it does                         |
| ------------------------------------------------------ | ------- | ------------------------------------ |
| `RECT_CONTAINS_POINT(x, y, w, h, px, py)`              | `U8`    | `1` if the rectangle contains the point.  |
| `RECT_OVERLAP(x1, y1, w1, h1, x2, y2, w2, h2)`         | `U8`    | `1` if two rectangles overlap.       |

Rectangles include their top left point and exclude `x + w`, `y + h`.

### Sprites

#### Importing sprite artwork

Use `@INCLUDE_SPRITE` to import sprite artwork and prepare it for the selected
target:

```basic
@INCLUDE_SPRITE HERO "assets/hero.sprites"
@INCLUDE_SPRITE ENEMY "ENEMY.SPR" "assets/enemy.sprites"
@INCLUDE_SPRITE WALK "assets/walk.png" TRANSPARENT_COLOR YELLOW
@INCLUDE_SPRITE LARGE_HERO "assets/large_hero.png" FRAME_SIZE 32 56 TRANSPARENT_COLOR YELLOW

PROC MAIN
	HERO_APPLY_FRAME 0, U16(0), U8(0)

	IF ENEMY_LOAD THEN
		ENEMY_APPLY_FRAME 1, U16(0), U8(0)
	ENDIF
ENDPROC
```

The importer detects the source format from its contents. The generated
interface is the same for every source format.

`TRANSPARENT_COLOR` accepts a standard color name or an index from 0 through
15. Pixels using that color become transparent. The option applies to every
source format. PNG palette transparency is used automatically.

`FRAME_SIZE width height` joins native sprite slots into one logical sprite.
PNG sheets are divided into frames of that size. SpritePad files group
consecutive sprites. Slots are arranged from left to right, then top to
bottom, with the logical sprite position at the top left.

For example, a 32 by 56 C64 frame uses six 24 by 21 sprite slots:

```text
0 1
2 3
4 5
```

Every logical frame uses the same number of slots. Edge slots are padded with
transparent pixels when a PNG frame does not fill their full area.
These are real hardware slots, not multiplexed slots. A six slot C64 sprite
uses six of the eight available hardware sprites.

The generated operations keep those slots together:

```basic
PROC MAIN
	NEXT_ID AS U8

	NEXT_ID = LARGE_HERO_APPLY_FRAME(U8(0), U16(0), U8(0))
	LARGE_HERO_MOVE U8(0), U16(100), U8(80)
	LARGE_HERO_SHOW U8(0)
	SPRITES_FLUSH
	LARGE_HERO_HIDE U8(0)
ENDPROC
```

`LARGE_HERO_SLOT_COUNT` is six. `LARGE_HERO_APPLY_FRAME` uses IDs 0
through 5 and returns 6, the first unused sprite ID. Its return value may be
ignored when another ID is not needed.

With no range, every frame and animation is imported. To import only part of
a sprite set, use a zero based frame range:

```basic
@INCLUDE_SPRITE HERO "assets/hero.sprites" FIRST_FRAME 20 FRAME_COUNT 4
```

The selected frames become frames 0 through 3. Animations contained entirely
in the range are kept and renumbered. Animations needing frames outside the
range are omitted. With `FRAME_SIZE`, ranges count logical frames and
SpritePad animations must align with logical frame boundaries.

Animation ranges are imported when the source provides them. Otherwise use
`SPRITE_ANIMATION_FRAME` with frame ranges chosen by the program. Supported
source formats and target packing are documented on the target pages.

For `@INCLUDE_SPRITE HERO "assets/hero.sprites"`, crustyBASIC creates:

| Name | Meaning |
| --- | --- |
| `HERO_WIDTH` / `HERO_HEIGHT` | Logical frame size. |
| `HERO_FRAME_COUNT` | Number of imported logical frames. |
| `HERO_SLOT_COUNT` | Number of consecutive sprite IDs used by one logical frame. |
| `HERO_BYTES_PER_FRAME` | Packed bytes for one logical frame. |
| `HERO_TILE_WIDTH` / `HERO_TILE_HEIGHT` | Target tile shape, or zero when frames are not tile based. |
| `HERO_DATA` | Target packed frame bytes. |
| `HERO_COLORS` | Source color for each slot in each frame. |
| `HERO_MODES` | `SPRITE_MODE_HIRES` or `SPRITE_MODE_MULTICOLOR` for each slot in each frame. |
| `HERO_EXPAND_X` / `HERO_EXPAND_Y` | Imported expansion flags for each slot in each frame. |
| `HERO_OVERLAYS` / `HERO_OVERLAY_DISTANCE` | Imported overlay metadata. |
| `HERO_BACKGROUND` / `HERO_SHARED_COLOR_1` / `HERO_SHARED_COLOR_2` | Source project colors. |
| `HERO_PALETTE` | Source palette as red, green, blue byte triples. |
| `HERO_APPLY_FRAME(id, frame, palette)` | Install one logical frame starting at `id`. Returns the first unused ID. `palette` is used on palette targets. |
| `HERO_MOVE(id, x, y)` | Move every slot as one logical sprite. |
| `HERO_SHOW(id)` / `HERO_HIDE(id)` | Show or hide every slot. |
| `HERO_ANIMATION_COUNT` | Number of imported animation slots, or zero. |
| `HERO_ANIMATION_FIRSTS` / `HERO_ANIMATION_LASTS` / `HERO_ANIMATION_HOLDS` | Imported range and timing data when animations are present. |
| `HERO_ANIMATION_MODES` / `HERO_ANIMATION_OVERLAYS` / `HERO_ANIMATION_VALID` | Imported animation flags when animations are present. |
| `HERO_ANIMATION_FRAME(animation, tick)` | Select a frame from an imported animation. Present when animations are present. |

The importer first converts source data into pixels, colors, flags, and
animation ranges. Target packing is a separate step, so different source
formats can produce the same generated interface.

The two-path form stages the packed frame bytes instead of embedding
`LABEL_DATA`. It creates `LABEL_LOAD`, `LABEL_LOAD(dst)`, and `LABEL_ADDR`.
`LABEL_APPLY_FRAME` uses the address selected by the most recent load.
Frame, color, mode, and animation metadata remain in the program.

#### Support and limits

| Name | What it does |
| --- | --- |
| `SPRITE_SUPPORTED`               | `TRUE` when the target supports the `SPRITE_*` API. |
| `SPRITE_KIND`                    | `SPRITE_KIND_HW` for the target's own sprites, or `SPRITE_KIND_POLYFILL` for sprites drawn by crustyBASIC. |
| `SPRITE_SURFACE_KIND`            | `SPRITE_SURFACE_KIND_PIXEL` uses pixels, `_CELL` uses cells, and `_KERNEL` follows the target page. |
| `SPRITE_COLOR_MODEL`             | How the target colors a sprite: `SPRITE_COLOR_MODEL_PER_PIXEL`, `_PER_CELL`, or `_POSITIONAL`. |
| `SPRITE_MAX_COUNT`               | Number of available sprite IDs.      |
| `SPRITE_WIDTH` / `SPRITE_HEIGHT` | Sprite size in the target's coordinate system. |
| `SPRITE_DATA_BYTES`              | Number of bytes copied by `SPRITE_DATA`. |
| `SPRITE_COLORS_PER_SPRITE`       | Most visible, nontransparent color codes supported by one sprite. |
| `SPRITE_X_MIN` / `SPRITE_X_MAX`  | Fully visible horizontal position range. |
| `SPRITE_Y_MIN` / `SPRITE_Y_MAX`  | Fully visible vertical position range. |
| `SPRITE_X_GRANULARITY` / `SPRITE_Y_GRANULARITY` | Steps used when snapping sprite coordinates. |

`SPRITE_MAX_COUNT` is the total number of movable sprites. On some
targets, certain sprite IDs cannot load custom shapes or must share
colors. Use `SPRITE_CAN_SET_SHAPE` and `SPRITE_COLOR_SHARED` when choosing an
ID. `SPRITE_NAME` returns the target's name for that sprite, or a generic
name such as `SPRITE0`.

The size and position limits use the target's screen coordinates. The
target page explains its sprite data shape and any differences between
sprite data and screen size.

#### Loading and displaying sprites

| Name / call | What it does |
| --- | --- |
| `SPRITES_ON()`                     | Prepare sprites for use without clearing the active display. |
| `SPRITES_RESET()`                  | Return all sprites to their starting settings. |
| `SPRITES_RESTORE()`                | Restore saved backgrounds before drawing beneath sprites, keeping their settings for the next flush. Available on some targets. |
| `SPRITE_CAN_SET_SHAPE(id)`       | `1` when the sprite ID accepts a custom shape through `SPRITE_DATA`. |
| `SPRITE_COLOR_SHARED(id)`        | `1` when changing this sprite's color can affect another sprite. |
| `SPRITE_NAME(id)`                | Target name for the sprite ID, or a generic name. |
| `SPRITE_DATA(id, addr)`          | Install data in the target's sprite format. |
| `SPRITE_DATA_8X8(id, addr)`      | Install an 8 byte, one color 8x8 shape. Each byte is one row, with its highest bit on the left. |
| `SPRITE_DATA_TILES(id, w, h, addr)` | Install a `w` by `h` sprite made from tiles. Software sprite targets read 8 bytes per tile in row order. |
| `SPRITE_INVERT_SUPPORTED`              | `TRUE` when `SPRITE_INVERT` is supported. |
| `SPRITE_INVERT(id)`              | Invert the data loaded for the sprite. |
| `SPRITE_ALIGN_X(x)` / `SPRITE_ALIGN_Y(y)` | Snap a coordinate to the target's sprite grid. |
| `SPRITE_MOVE(id, x, y)`          | Move a sprite.                       |
| `SPRITE_FLIP_X(id, flag)` / `SPRITE_FLIP_Y(id, flag)` | Turn horizontal or vertical flipping on or off where supported. |
| `SPRITE_EXPAND(id, x_double, y_double)` | Double a sprite's width or height where supported. |
| `SPRITE_PRIORITY(id, behind_bg)` | Choose whether a sprite appears in front of or behind the background where supported. |
| `SPRITE_SHOW(id)` / `SPRITE_HIDE(id)` | Show / hide a sprite.          |
| `SPRITES_OFF()`                  | Hide or disable all sprites.          |
| `SPRITES_FLUSH()`                | Apply any waiting sprite changes.   |

`SPRITE_DATA` copies bytes in the target's own sprite format without
converting them. A different sprite mode may read those bytes
differently, so use data made for the chosen mode. `SPRITE_DATA_8X8`
uses the portable one bit 8x8 format where supported.
`SPRITE_DATA_TILES` uses `w` and `h` at runtime on software sprite
targets. The data buffer contains `w * h` consecutive 8 byte tiles,
ordered left to right and then top to bottom. Storage needed while
drawing or restoring an individual sprite is allocated from those
dimensions where needed; there is no separate cell limit. Allocation
failure is not checked.

#### Colors and modes

| Name / call | What it does |
| --- | --- |
| `SPRITE_COLOR(id, color)` | Set sprite color where supported. |
| `SPRITE_MODE_HIRES` / `SPRITE_MODE_MULTICOLOR` | Sprite data modes used by the option, frame metadata, and runtime call. |
| `SPRITE_MODE_RUNTIME_SUPPORTED` | `TRUE` when individual sprites can change mode at runtime. |
| `SPRITE_MODE(id, mode)` | Change one sprite's mode where supported. |
| `SPRITE_BG(color)` | Set the background color used where a sprite overlaps cells that share colors. Does nothing elsewhere. |
| `SPRITE_COLOR2(color)` / `SPRITE_COLOR3(color)` | Set extra shared colors on targets with multicolor sprites. Does nothing elsewhere. |
| `SPRITE_PALETTE(id, palette)` | Select a sprite palette where supported. |
| `SPRITE_PALETTE_SET(palette, c0, c1, c2)` | Set sprite palette colors where supported. |
| `SPRITE_PALETTE_SET(palette, index, RGB(r, g, b))` | Set one visible sprite palette color where supported. |
| `SPRITE_PALETTE_RGB_SUPPORTED` | `TRUE` when the RGB form of `SPRITE_PALETTE_SET` is supported. |

When `SPRITE_PALETTE_RGB_SUPPORTED` is `TRUE`, the RGB form of
`SPRITE_PALETTE_SET` changes one visible palette entry. `index` starts
at zero and does not include the transparent entry.

`SPRITE_COLOR_MODEL` explains how sprite colors behave:

- `SPRITE_COLOR_MODEL_PER_PIXEL`: pixels can keep their own colors.
- `SPRITE_COLOR_MODEL_PER_CELL`: cells share colors, so a sprite can
  change the background color in cells it overlaps. `SPRITE_BG` chooses
  the color restored behind it.
- `SPRITE_COLOR_MODEL_POSITIONAL`: color depends on screen position, so
  `SPRITE_COLOR` may not produce the exact requested color.

You can always call `SPRITE_BG`, `SPRITE_COLOR2`, `SPRITE_COLOR3`, and
`SPRITE_MODE`. They do nothing when the target does not use them.
`@OPTION SPRITE_DEFAULT_MODE SPRITE_MODE_MULTICOLOR` selects the starting
mode. The default is `SPRITE_MODE_HIRES`. When
`SPRITE_MODE_RUNTIME_SUPPORTED` is `TRUE`, individual sprites can mix modes.

Multicolor shapes use two bit color codes, with `00` as transparent.
The other codes use `SPRITE_COLOR`, `SPRITE_COLOR2`, and
`SPRITE_COLOR3` as described on the target page. When sprites are drawn
into a bitmap, `SPRITE_BG` chooses the background used when sprite and
bitmap colors are combined.

`@OPTION SOFT_SPRITE_OPAQUE TRUE` hides the background in a software
sprite's empty shape pixels. The default is `FALSE`, which lets it show
through. On some targets, opaque sprites can share shape storage and fit
more sprites. Background restoration when sprites move is separate from
opacity. This option has no effect on hardware sprites.

On targets that save sprite backgrounds, hide affected sprites and apply
pending changes with `SPRITES_FLUSH` before drawing beneath them. Otherwise,
moving a sprite can restore an older copy of the background over your changes.

#### Collision

| Call | What it does |
| --- | --- |
| `SPRITE_HIT(id)` | Sprite/sprite collision flag where supported, otherwise `0`. |
| `SPRITE_HIT_BG(id)` | Sprite/background collision flag where supported, otherwise `0`. |

#### Animation

| Name / call | What it does |
| --- | --- |
| `SPRITE_ANIMATION_FRAME(first, last, hold, mode, tick)` | Select a frame from a numeric range. |
| `SPRITE_ANIMATION_LOOP` / `_ONCE` / `_PING_PONG` | Range playback modes. |

`SPRITE_ANIMATION_FRAME` divides `tick` by `hold`, then selects a frame from
`first` through `last`. Loop wraps to the first frame, once stops at the last
frame, and ping pong reverses at both ends. `hold` is the number of ticks per
frame.

The [portable sprite showcase](../examples/__portable__/sprites.cbs)
combines movement, multiple shapes, colors, flipping, expansion, priority,
and all three animation modes. The
[C64 asset include showcase](../examples/c64/crustybasic/include_assets.cbs)
shows sprite, charset, charmap, tileset, and tilemap imports first from embedded
data and then from media on separate screens. Its custom charmap prompt waits
for Space before the program loads the assets from disk.

#### Software sprite count

Programs choose how many software sprite slots to allocate:

```basic
@OPTION SOFT_SPRITE_COUNT 1
```

This setting sizes the sprite state and update loops. `SPRITE_MAX_COUNT`
reports the selected count, with valid IDs from zero through one less than
that count. A shifted sprite may use extra cell storage, but it still uses
one sprite ID.

## Core builtins

Numeric helpers, bitwise and logical operators, `REAL` helpers, and
string functions are part of the core language. See
[LANGUAGE.md](LANGUAGE.md#core-builtins).

## REAL floating point

`REAL` is optional and target dependent.

`PI` is a `REAL` constant with the value `3.141592653589793`. It is
available when `REAL_SUPPORTED` is `TRUE`.

```basic
X! = PI
Y! = LOG(X!)
PRINT STR(Y!)
```

Declaring `REAL` on a target without floating point support is a compile
error.

Core `REAL` functions are documented in
[LANGUAGE.md](LANGUAGE.md#real-functions).

## Fixed point

These helpers provide fractional values without using `REAL`. They store
the value in a `U16`: the high byte is the whole number part and the low
byte is the fraction. This format is called unsigned Q8.8.

| Name | What it does |
| ---- | ------------ |
| `FX_ONE` | The value `1.0` (`256`). |
| `FX_HALF` | The value `0.5` (`128`). |
| `FX_MAX` | Largest Q8.8 value (`65535`). |
| `FX_FROM_INT(n)` | Convert a `U8` integer to Q8.8. |
| `FX_FROM_PARTS(i, f)` | Build Q8.8 from integer and fractional bytes. |
| `FX_INT(x)` | Return the integer byte. |
| `FX_FRAC(x)` | Return the fractional byte. |
| `FX_MUL(a, b)` | Multiply two Q8.8 values. |
| `FX_DIV(a, b)` | Divide two Q8.8 values. |

## Memory operations

### ADDR

`ADDR(name)`, `ADDR name`, and `&name` return an address using the size
needed by the target. Store it in a variable declared `AS ADDR`. These
forms work with variables, array elements, string literals, and no
argument PROCs outside `@BANK`. Use them with `MEMMOVE`, `MEMFILL`,
`DPOKE`, calls that need a PROC address such as `FRAME_INSTALL`, or
inline `ASM`.

```basic
BUF(255) AS U8
WORDS(15) AS U16
NAMES$(3) AS STRING * 16

PTR AS ADDR

PTR = ADDR BUF
PTR = ADDR(BUF)
PTR = &BUF
PTR = ADDR BUF[10]
PTR = &BUF[10]
PTR = ADDR WORDS[3]
PTR = ADDR(WORDS(3))
PTR = ADDR(NAMES$(2))
PTR = ADDR "LITERAL"

PROC FOO
    PRINT "IN PROC"
ENDPROC

PROC_PTR AS ADDR = ADDR FOO
```

`ADDR "LITERAL"` returns the address of read only text. Do not write to
that address. The bytes use the same target encoding as a `PRINT`
string, including `{NAME}` control code escapes and placeholders. See
[Literals](LANGUAGE.md#literals).

`ADDR(ARR(I))` returns the address of that element, not the start of the
array.

`ADDR(S$)` returns the address of the string's first character, so it
pairs directly with `MEMMOVE`, `MEMFILL`, and `BIND`. The same holds for
a string array element such as `ADDR(NAMES$(2))`. The address does not
include the string length; use `LEN` to read it.

`ADDR(FOO)` returns the starting address of a `PROC`. The `PROC` must
take no parameters and cannot be in a switched bank.

`ADDR` can take the address of `DIM` storage, typed `CONST` arrays, and
no argument PROCs outside `@BANK`. It cannot take the address of a scalar
`CONST` or a `PROC` parameter.

### PEEK, POKE, DPEEK, DPOKE

```basic
B = PEEK($2000)
POKE $2000, 6
W = DPEEK $2010
DPOKE($2020, $ABCD)
```

- `PEEK` reads one byte from an address.
- `POKE` writes one byte to an address.
- `DPEEK` reads two bytes from an address and returns a 16 bit value: low byte first, high byte second.
- `DPOKE` writes a 16 bit value to an address: low byte first, high byte second.

### MEMMOVE, MEMFILL, MEMCOMPARE

```basic
SRC(15) AS U8
DST(15) AS U8

MEMFILL(ADDR(SRC), 16, 0)
MEMMOVE(ADDR(SRC), ADDR(DST), 16)
IF MEMCOMPARE(ADDR(SRC), ADDR(DST), 16) THEN PRINT "SAME"
```

`MEMMOVE(src, dst, count)` copies bytes forward from source to destination.
It does not preserve overlapping source and destination ranges.

`MEMFILL(dst, count, value)` fills a range of bytes with one byte value.

`MEMCOMPARE(a, b, count)` returns `1` when two byte ranges match,
otherwise `0`.

None of these check addresses or counts. A bad address or an oversized
count reads or writes whatever is there.

`MEMSIZE_TEXT(size_kb)` returns a compact `K` or `MB` string for a KB
count, such as `64K` or `1 MB`.

### Asset include forms

Asset directives use the same two forms:

```basic
@INCLUDE_TYPE LABEL "source"
@INCLUDE_TYPE LABEL "runtime-file" "source"
```

The one-path form converts and embeds the asset. The two-path form converts
the asset, stages `runtime-file` beside the program or on its selected
medium, and does not embed a second copy. The source path is resolved next
to the source file first, then from the project folder. Filename case on the
host does not have to match.

For memory backed resources, the two-path form creates:

| Name | What it is |
| ---- | ---------- |
| `LABEL_FILE` | Runtime filename after adjustment for the selected medium. |
| `LABEL_FILE_LEN` | Staged byte count. |
| `LABEL_DATA` | Automatically allocated default buffer. |
| `LABEL_LOAD` | Load into the default buffer and return `TRUE` on success. |
| `LABEL_LOAD(dst)` | Load into memory chosen by the program. |
| `LABEL_ADDR` | Address used by the most recent successful load. |

`@INCLUDE_FILE` uses `LABEL` for its buffer instead of `LABEL_DATA`.
Image and audio loaders send data directly to their natural destination,
so they create only `LABEL_LOAD` and their filename metadata. Text creates
`LABEL_LOAD(name$, out text$)`.

The load calls require file support. Gate code that uses an external form
with `FILE_SUPPORTED` and any additional support value required by that
asset type.

### INCLUDE_FILE

`@INCLUDE_FILE` is the raw, untyped escape hatch. It copies bytes without
conversion:

```basic
@INCLUDE_FILE TABLE "assets/table.bin"
@INCLUDE_FILE PICTURE "PICTURE.IMG" "assets/picture.img"
@INCLUDE_FILE PAYLOAD "assets/program.prg" OFFSET 2 COUNT 256
```

The one-path form embeds a `CONST U8` array. It creates `LABEL_LEN`,
`LABEL_LAST`, `LABEL_ADDR`, and `LABEL_END`. The two-path form stages the
selected bytes and creates the runtime buffer and load calls described
above.

`OFFSET` and `COUNT` work with either form and accept decimal, `$` hex, and
`0x` hex literals. For example, `OFFSET 2` can skip a two byte file header.
Without `COUNT`, the directive uses every byte after the offset.

`@INCLUDE_FILE` treats the selected bytes as untyped data. It cannot validate
the file contents or reserve memory that those contents may use at runtime.

Use a typed directive when conversion or validation is wanted. For example,
`@INCLUDE_AUDIO` converts audio and `@INCLUDE_IMAGE` converts images.

An included asset only reserves a separate memory range when its format
declares where its bytes must live at runtime. Ordinary included arrays
and files have no such destination. All include converters use the same
placement rule: for a program loaded into RAM, crustyBASIC moves the
program code when its default position would overlap a declared range.
An explicit program placement remains fixed and reports a conflict
instead.

### INCLUDE_TEXT

`@INCLUDE_TEXT` keeps your writing in a plain text file instead of in
the program. Each block of text becomes a string constant.

```basic
@INCLUDE_TEXT ROOMS "rooms.txt"

PRINT ROOMS_CELLAR
```

A line starting with `.` names a block. Everything after it belongs to
that block until the next name:

```
.CELLAR
DARK. SOMETHING DRIPS.
A DOOR STANDS OPEN.

.HALL
A LONG HALL RUNS NORTH.
```

Lines in a block join together with a space between them, so you can
wrap your writing however you like and blank lines are just spacing.
Names have to be usable as BASIC identifiers, and you cannot use the
same name twice in one file.

For `@INCLUDE_TEXT LABEL "path"`, crustyBASIC creates:

| Name | What it is |
| ---- | ---------- |
| `LABEL_<NAME>` | `CONST` `STRING` holding that block's text. |
| `LABEL_COUNT` | How many blocks the file had, as `U16`. |

The label keeps two text files from clashing, so `.CELLAR` in the
example above becomes `ROOMS_CELLAR`.

These are ordinary string constants. Print them, compare them, or copy
them into a `DYNAMIC STRING` the same as any other string. The text goes
in with the rest of your program and costs no RAM until you copy it
somewhere.

Use the two-path form to keep the text outside the program binary:

```basic
@INCLUDE_TEXT ROOMS "ROOMS.TXT" "rooms.txt"

DESCRIPTION$ AS STRING * 255

IF ROOMS_LOAD("CELLAR", DESCRIPTION$) THEN
	PRINT DESCRIPTION$
ENDIF
```

This form creates `ROOMS_FILE` and
`ROOMS_LOAD(name$, out text$)`. It validates the same named blocks and
stages them in the selected target's text encoding, but does not generate
one constant per block. Changing only the external text changes the staged
file without changing the generated program source.

### EXTMEM

EXTMEM moves blocks between the program's normal RAM and extra memory
provided by the machine or an expansion device. A target can expose more
than one provider through handles.

The full EXTMEM address is `HANDLE:BANK:XADDR`. `HANDLE` and `BANK` are
`U8`, and `XADDR` is `U16`. Calls without a handle use `EXTMEM_DEFAULT`.
`COUNT = 0` means 65536 bytes.

| Name | What it does |
| ---- | ------------ |
| `EXTMEM_SUPPORTED` | `TRUE` when the target implements the portable extra-memory interface. |
| `EXTMEM_DEFAULT` | Handle used by calls that omit one. |
| `EXTMEM_AVAILABLE([handle])` | `1` when the selected provider is currently available. |
| `EXTMEM_STASH([handle,] count, localaddr, xaddr, bank)` | Copy local RAM to extra memory. |
| `EXTMEM_FETCH([handle,] count, localaddr, xaddr, bank)` | Copy extra memory to local RAM. |
| `EXTMEM_SWAP([handle,] count, localaddr, xaddr, bank)` | Exchange local and extra memory. |
| `EXTMEM_FILL([handle,] count, xaddr, bank, value)` | Fill extra memory with one byte value. |
| `EXTMEM_VERIFY([handle,] count, localaddr, xaddr, bank)` | Return `1` when the two ranges match. |
| `EXTMEM_POKE([handle,] bank, xaddr, value)` | Write one byte to extra memory. |
| `EXTMEM_PEEK([handle,] bank, xaddr)` | Read one byte from extra memory. |
| `EXTMEM_DETECT_KB([handle])` | Detect the selected provider's available size in KB. |
| `EXTMEM_HAS_KB([handle,] size_kb)` | Return `1` when the selected provider has at least that much memory. |
| `EXTMEM_BANK_KB([handle])` | Size of one `BANK` step for the selected provider. |

Use `EXTMEM_SUPPORTED` with `@IF` or `@REQUIRES` when a program requires
the interface. Check `EXTMEM_AVAILABLE()` before transferring data because
supported expansion hardware may not be attached. Without EXTMEM support,
the probe returns `0` and the transfer calls are unavailable.

Target pages list their provider handles. The handle is the first argument
when present:

```basic
IF EXTMEM_AVAILABLE(EXTMEM_REU) THEN
	EXTMEM_STASH EXTMEM_REU, 256, ADDR BUFFER, 0, 1
ENDIF
```

Some targets let `@OPTION EXTMEM provider` select the handle used by calls
that omit one. Explicit handles can still access another provider in the
same program.

#### Mapper storage and EXTMEM

A mapper can use extra memory internally without changing the `EXTMEM`
interface. Select the mapper's default storage with:

```basic
@OPTION MAPPER name
```

When a mapper supports it, select a provider and its physical bank
explicitly with:

```basic
@OPTION MAPPER name(PROVIDER, 0)
```

This physical bank selection does not renumber or reserve an `EXTMEM`
bank. Each EXTMEM handle keeps its own bank numbering. If a mapper and an
EXTMEM handle use the same expansion memory, choose physical banks that do
not overlap. The target page lists the mappings.

### Heap

The heap hands out blocks of memory while the program runs, from the RAM
left over past your code and data.

```basic
@REQUIRES HEAP_SUPPORTED @ELSE "this program needs a heap"

BUF AS ADDR
BUF = HEAP_ALLOC(200)

IF BUF = 0 THEN
    PRINT "OUT OF MEMORY"
    END
ENDIF

POKE BUF, 65
BUF = HEAP_RESIZE(BUF, 400)
HEAP_FREE BUF
```

| Name | What it does |
| ---- | ------------ |
| `HEAP_SUPPORTED` | `TRUE` when the target can host a heap. |
| `STRING_BIND_SUPPORTED` | `TRUE` when a string can be pointed at memory the program supplies, which `BIND` needs. |
| `HEAP_POINTER_BYTES` | Bytes one stored address takes on this machine. |
| `HEAP_ALLOC(size)` | Reserve `size` bytes and return the address, or `0` when the heap is full. |
| `HEAP_FREE(addr)` | Hand a block back. Freeing `0` does nothing. |
| `HEAP_RESIZE(addr, size)` | Change a block's size, keeping its contents. Returns the new address, which may differ, or `0` when the heap is full. A `size` of `0` frees the block. |
| `HEAP_PAYLOAD(addr)` | Usable bytes in a block, which can be more than you asked for. |
| `HEAP_AVAIL()` | Total free bytes. |
| `HEAP_LARGEST()` | Largest single allocation the heap can still satisfy. |
| `HEAP_CHECK()` | Walk the whole heap and return `1` when it is consistent. |

Freed blocks next to each other are merged as the heap is searched, so
`HEAP_AVAIL` counts each free block separately while `HEAP_LARGEST`
reports what a single request can actually get.

A block holds at most 65534 bytes. Sizes round up to an even number.

Arrays whose size is calculated while the program runs use this heap, and
so does every `DYNAMIC STRING`. See
[runtime-sized arrays](LANGUAGE.md#arrays) and
[strings](LANGUAGE.md#strings).

A `DYNAMIC STRING` needs only `HEAP_SUPPORTED`. Declaring one on a target
without a heap is a compile error naming the target.

`@OPTION HEAP_CHECKS TRUE` prints a message before ending when the heap
runs out or a bad address is released. It is off by default.

### BIND

```basic
BIND STRING$ TO ADDR, COUNT

RAW(79) AS U8
VIEW$ AS STRING * 80

BIND VIEW$ TO ADDR(RAW), 80
VIEW$ = "HELLO"
MID(VIEW$, 10, 1) = CHR(255)
```

`BIND` makes a string use an area of memory that you provide. The string
still cannot grow past its declared size. It can use `COUNT` bytes
starting at `ADDR`.

- The value before `TO` must be a string variable.
- `ADDR` and `COUNT` are treated as 16 bit values.
- If `COUNT` is a fixed number, it cannot be larger than the string's declared size.
- `ADDR` must point at writable RAM.

## Events

The event API lets a program respond to target events such as sprite
collisions or a light pen. When `EVENT_INTERRUPT_CALLBACKS` is `TRUE`, a
handler may interrupt normal program flow. Handlers must take no
arguments. Pass them as `ADDR(name)`.

| Name | What it does |
| --- | --- |
| `EVENT_SUPPORTED` | `TRUE` when the target supports events. |
| `EVENT_INTERRUPT_CALLBACKS` | `TRUE` when handlers may interrupt normal program flow. |
| `EVENT_SPRITE_SPRITE` | Sprite to sprite collision event mask. |
| `EVENT_SPRITE_BG` | Sprite to background collision event mask. |
| `EVENT_LIGHTPEN` | Light pen event mask. |
| `EVENT_SUBSCRIBE(mask, addr)` | Subscribe a handler address for one or more event bits. |
| `EVENT_UNSUBSCRIBE(mask)` | Remove handlers and disable those event bits. |
| `EVENT_ENABLE(mask)` | Enable one or more event bits without changing handlers. |
| `EVENT_DISABLE(mask)` | Disable one or more event bits. |
| `EVENT_PENDING()` | Return event bits that have been recorded but not cleared. |
| `EVENT_ACK(mask)` | Clear pending event bits. |

Use one event value as `mask`, or add event values together to select
more than one.

On targets without event support, these calls do nothing and
`EVENT_PENDING()` returns `0`.

## CALL

```basic
CALL $2000
```

`CALL` runs the machine code routine at a fixed address. How you pass
values in, what registers it changes, and how it returns values depend
on the machine and the routine you are calling.

## USR

`USR(...)` is dialect specific. Its syntax and value passing follow the
selected BASIC dialect. See the USR section on the selected
[target page](targets/) for the exact form.

For crustyBASIC code, use `CALL` for a simple machine code routine at a
fixed address. Use inline `ASM` when you need to pass values or get a
result.

## UCI

The C64 Ultimate system enables UCI support automatically. A stock C64 fitted
with a 1541 Ultimate-II+ can enable it with
`@OPTION C64_ULTIMATE_SUPPORTED TRUE`. The option is compile-time support;
`UCI_AVAILABLE()` reads the UCI identification register at runtime.

| Call | What it does |
| --- | --- |
| `UCI_AVAILABLE()` | `1` when the Ultimate Command Interface responds at runtime. |

## FujiNet

Targets with FujiNet support define `FUJINET_SUPPORTED` and
`FUJINET_TRANSPORT`. The calls here send commands and exchange data with
the adapter.

| Name / call | What it does |
| --- | --- |
| `FUJINET_SUPPORTED` | `TRUE` when FujiNet support is available. |
| `FUJINET_TRANSPORT` | How the target communicates with FujiNet. |
| `FUJINET_TRANSPORT_NONE` / `FUJINET_TRANSPORT_SIO` / `FUJINET_TRANSPORT_DRIVEWIRE` | Transport constants. |
| `FUJINET_STATUS_OK` / `FUJINET_STATUS_IO_ERROR` / `FUJINET_STATUS_UNSUPPORTED` | Status constants. |
| `FUJINET_DIR_NONE` / `FUJINET_DIR_READ` / `FUJINET_DIR_WRITE` | Command direction constants. |
| `FUJINET_BUFFER_MAX` | Maximum shared buffer size. |
| `FUJINET_STATUS()` | Return the current adapter status. |
| `FUJINET_COMMAND(device, command, aux1, aux2, direction, count)` | Send one adapter command for the current target. |
| `FUJINET_EXCHANGE(device, command, aux1, aux2, direction, write_count, read_count)` | Send request bytes and optionally read reply bytes. |
| `FUJINET_BUFFER_CLEAR()` | Clear the shared command buffer. |
| `FUJINET_BUFFER_SET_LEN(count)` / `FUJINET_BUFFER_LEN()` | Set or read the active buffer length. |
| `FUJINET_BUFFER_POKE(offset, value)` / `FUJINET_BUFFER_PEEK(offset)` | Write or read one shared buffer byte. |
| `FUJINET_BUFFER_WRITE_STRING(data$)` / `FUJINET_BUFFER_READ_STRING(out data$)` | Write or read the buffer as a string. |

The target pages list the FujiNet device and command values used by
`FUJINET_COMMAND` and `FUJINET_EXCHANGE`.

## Hardware registers

Target pages under [`targets/`](targets/) list named addresses for their
built in chips. Use those names with `PEEK`, `POKE`, or inline `ASM` as
described on the target page.

## Target docs

The pages under [`targets/`](targets/) list each target's screen sizes,
input counts, image formats, timing, file support, chip names, and
system notes. Check each API section's `*_AVAILABLE` values when a
program must work on more than one target.
