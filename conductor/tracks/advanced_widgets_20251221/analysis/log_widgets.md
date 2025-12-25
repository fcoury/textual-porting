# Log & RichLog Scrolling/Formatting (Python Textual)

Sources:
- `textual/src/textual/widgets/_log.py`
- `textual/src/textual/widgets/_rich_log.py`

## Log (plain text)
- Stores raw `_lines: list[str]` plus `_width` and `_updates`.
- `max_lines` prunes old lines and rewrites render cache keys.
- `auto_scroll` drives scroll-to-end behavior; respects scrollbar grab state.
- Line widths computed with `rich.cells.cell_len` after processing control chars.
- Uses `LRUCache[int, Strip]` for rendered lines; cache cleared on style updates and selection changes.
- Selection uses `Selection` and `screen--selection` style to highlight selection region.
- `_update_size` runs in a background thread to update maximum width for new lines.
- `render_line` crops by scroll_x and applies offsets.

## RichLog (renderables)
- Stores rendered `lines: list[Strip]` (already rendered with Rich).
- `DeferredRender` queue if size not known (writes during compose/mount).
- `write()` accepts renderable or object; if string, can apply markup/highlight.
- Measures renderable width with `rich.measure.measure_renderables` to determine render width.
- Supports `expand`/`shrink` behavior; respects `min_width`.
- Uses `Segment.split_lines` to create strips; maintains `_widest_line_width`.
- `max_lines` trims list and increases `_start_line` offset.
- `render_line` uses cache keyed by `(line_index, scroll_x, width, widest_width)`.

## Common Scrolling Behavior
- Both are `ScrollView` subclasses with vertical scroll enabled.
- Both update `virtual_size` for width/height changes.
- Auto-scroll to end after writes unless disabled or user scrolled away.

## Implications for Rust Port
- Log: keep raw lines + text width calculation; background width updates optional but helpful for large inputs.
- RichLog: renderables to strips; need deferred rendering until widget size known.
- Both should support `max_lines` trimming with consistent scrolling offsets.
- Keep selection rendering for Log; RichLog does not handle selection in Python.
