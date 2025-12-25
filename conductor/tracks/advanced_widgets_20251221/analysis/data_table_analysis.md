# DataTable Analysis (Python Textual)

Source: `textual/src/textual/widgets/_data_table.py`

## Scope & Responsibilities
- Owns a tabular data model with row/column keys that are stable across sorting.
- Handles rendering, scrolling, selection, cursoring, and hover state.
- Applies TCSS via component classes (cursor, header, fixed rows/cols, zebra stripes).
- Implements heavy caching to keep per-cell rendering fast.

## Key Types & Identity
- `StringKey` wraps optional string; supports equality with raw strings and stable hashing.
- `RowKey` / `ColumnKey` are `StringKey` subclasses.
- `CellKey` is `(row_key, column_key)`.
- `Coordinate` is used for row/column indices; keys are used for identity.

## Core Data Structures
- `_data: dict[RowKey, dict[ColumnKey, CellType]]` holds actual cell values.
- `columns: dict[ColumnKey, Column]` and `rows: dict[RowKey, Row]` store metadata.
- `_row_locations` and `_column_locations` are `TwoWayDict` mapping key <-> index.
- Row label column uses a special key (`_label_column_key`).
- Header uses a special row key (`_header_row_key`).

## Row/Column Sizing
- Column width: explicit `width` or `auto_width` based on max content width.
- Row height: explicit height or auto-height (`height == 0` until measured).
- `cell_padding` is horizontal padding on each side of every cell.
- `header_height` is reactive and used to reserve top rows.

## Rendering Pipeline (High Level)
- ScrollView is leveraged; DataTable implements `render_line(y)` with cached strips.
- Uses `Segment`/`Strip` layers and `console.render_lines` for renderable cells.
- `Padding(cell, (0, cell_padding))` provides column padding at render time.
- Base style is derived from component styles + cursor/hover overlays.

## Component Classes (TCSS)
- Declares `COMPONENT_CLASSES` including:
  - `datatable--cursor`, `datatable--hover`, `datatable--fixed`, `datatable--header`,
  - `datatable--fixed-cursor`, `datatable--header-cursor`, `datatable--header-hover`,
  - `datatable--odd-row`, `datatable--even-row`.
- `DEFAULT_CSS` defines nested selectors (`&:focus`, `& > .datatable--cursor`, etc.).
- `PseudoClasses` (focus/hover/disabled) influence style cache keys.

## Caching Strategy
- Multiple LRU caches with different scopes:
  - `_cell_render_cache` keyed by (row_key, col_key, style, cursor/hover flags, update_count, pseudo_class_state).
  - `_row_render_cache` keyed by (row_key, line_no, style, cursor/hover, update_count, pseudo_class_state).
  - `_line_cache` keyed by row/column indices and style to avoid recomputing strips.
  - `_row_renderable_cache` keyed by (update_count, row_index) to avoid reformatting.
  - `_offset_cache` keyed by update_count for y-offset mapping.
  - `_ordered_row_cache` keyed by (num_rows, update_count) for sorted row ordering.
- `_update_count` increments on data mutations (including sort) to invalidate caches.

## Cursor / Hover Behavior
- `cursor_type`: "cell" | "row" | "column" | "none".
- `cursor_coordinate` and `hover_coordinate` are reactive.
- Hover cursor uses style meta from segments (`row`, `column`) to track hover.
- `show_cursor` gates selection and message posting.

## Messages
- Emits `CellHighlighted`, `CellSelected`, `RowHighlighted`, `RowSelected`,
  `ColumnHighlighted`, `ColumnSelected`, `HeaderSelected`, `RowLabelSelected`.
- Messages are only emitted when cursor type matches intent and cursor is visible.

## Sorting
- `sort(*columns, key=None, reverse=False)` reorders `_row_locations` only.
- Sorting does not mutate `_data`; keys remain stable.

## Notable Rendering Details
- Row labels are rendered as a fixed column and may be styled like headers.
- Fixed rows and columns are rendered separately from scrollable body.
- Header row is row index -1 and gets its own height.
- `zebra_stripes` selects odd/even row component classes.

## Implications for Rust Port
- Data model should preserve key identity independent of row/column position.
- Must support component-class based styling with nested selectors.
- Cache keys must include pseudo-classes, cursor/hover state, and update count.
- Rendering should handle per-row multi-line height and per-cell padding.
- Hover/cursor logic depends on metadata in render segments for hit-testing.
