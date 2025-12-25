# DataTable Data Structures & Cursor System (Python Textual)

Source: `textual/src/textual/widgets/_data_table.py`

## Data Structures

### Identity & Keys
- `StringKey`: wraps an optional string; hashes by string value when present, otherwise by object identity.
  - Compares equal to raw strings containing the same value.
  - Enables stable identity independent of row/column ordering.
- `RowKey` / `ColumnKey`: subclasses of `StringKey` used as stable IDs.
- `CellKey`: tuple `(row_key, column_key)` identifying a cell regardless of display position.
- `Coordinate`: row/column index (display position). Sorting changes `Coordinate` but not keys.

### Core Storage
- `_data: dict[RowKey, dict[ColumnKey, CellType]]` stores actual cell values.
- `rows: dict[RowKey, Row]` holds row metadata.
- `columns: dict[ColumnKey, Column]` holds column metadata.
- `_row_locations: TwoWayDict[RowKey, int]` maps row key <-> row index.
- `_column_locations: TwoWayDict[ColumnKey, int]` maps column key <-> column index.

### Row/Column Metadata
- `Column`:
  - `key`, `label` (Rich `Text`), `width`, `content_width`, `auto_width`.
  - `get_render_width()` includes `cell_padding` on both sides.
- `Row`:
  - `key`, `height`, `label`, `auto_height`.
- Row label column uses `_label_column_key` + `_label_column`.
- Header row uses `_header_row_key` and `header_height`.

### Layout/Dimension Helpers
- `_y_offsets`: cached list of `(row_key, y)` for every rendered line in all rows.
- `_total_row_height`: length of `_y_offsets`.
- `_row_label_column_width`: render width of row label column (0 if hidden).

### Change Tracking & Caches
- `_update_count`: monotonically increases on data mutations and sort operations.
- `_new_rows` / `_updated_cells`: track rows/cells requiring dimension recalculation.
- Multiple LRU caches for row/cell/line renderables; keys include `update_count` and pseudo-classes.

## Cursor System

### Reactive State
- `cursor_type: Reactive[Literal["cell", "row", "column", "none"]]`.
- `cursor_coordinate: Reactive[Coordinate]` (current cursor position).
- `hover_coordinate: Reactive[Coordinate]` (mouse hover position).
- `show_cursor: Reactive[bool]` toggles both hover and keyboard cursor rendering.
- `_show_hover_cursor` toggles hover cursor visibility when using keyboard.
- `_pseudo_class_state` is part of cache keys so `:focus` styles invalidate properly.

### Movement & Clamping
- `move_cursor(row=None, column=None, animate=False, scroll=True)`:
  - Computes destination coordinate; optionally scrolls into view first.
  - Uses `call_after_refresh` when dimensions are pending.
- `validate_cursor_coordinate` -> `_clamp_cursor_coordinate` clamps to valid bounds.
- Action handlers (`cursor_up/down/left/right`, `page_up/down`, `home/end`):
  - Update coordinate and call `_scroll_cursor_into_view` where appropriate.

### Highlighting & Messages
- `watch_cursor_coordinate`:
  - Refreshes old/new cells and posts highlight messages based on `cursor_type`.
- `_highlight_coordinate` -> `CellHighlighted` message (with `CellKey`).
- `_highlight_row` -> `RowHighlighted` message.
- `_highlight_column` -> `ColumnHighlighted` message.
- `action_select_cursor` posts selected messages based on cursor type:
  - `CellSelected`, `RowSelected`, `ColumnSelected`, `HeaderSelected`, `RowLabelSelected`.

### Hover Behavior
- `_on_mouse_move` inspects style meta from segments (`row`, `column`) and updates hover.
- `_set_hover_cursor(active)` toggles `_show_hover_cursor` and refreshes the relevant region.

### Cursor Visibility & Mode Changes
- `watch_show_cursor`:
  - Clears caches, scrolls to cursor, and posts appropriate highlight when re-enabled.
- `watch_cursor_type`:
  - Clears hover, posts highlight, refreshes impacted cells from previous type.

### Scrolling Integration
- `_scroll_cursor_into_view`:
  - Computes Region based on cursor type (cell/row/column).
  - Uses fixed row/column offsets to keep cursor visible while scrolling.

## Implications for Rust Port
- Cursor state must be reactive and participate in cache keys.
- Cursor/hover selection requires per-cell metadata to map mouse events to coordinates.
- Cursor type changes should refresh prior highlighted region and post highlight events.
- Cursor movement must account for virtual size updates (defer scroll until after refresh).
