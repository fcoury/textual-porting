# DataTable Data Model (Design)

## Goals
- Preserve Python Textual semantics: stable row/column keys regardless of sorting or reordering.
- Support user-supplied string keys for rows/columns (lookup by key string).
- Keep data structure close to Python to simplify parity and test porting.
- Avoid expensive full-table reshapes on column changes.

## Core Types

### Keys
- `RowKey` and `ColumnKey` are stable opaque identifiers.
- Keys must remain valid across sorting, reordering, and cursor movement.
- User-facing string keys are supported via a lookup map.

Proposed Rust shapes:
```rust
#[derive(Clone, Copy, Debug, Eq, Hash, PartialEq)]
pub struct RowKey(slotmap::Key); // or a newtype around u64

#[derive(Clone, Copy, Debug, Eq, Hash, PartialEq)]
pub struct ColumnKey(slotmap::Key);
```

Associated lookup maps:
```rust
row_keys_by_name: HashMap<String, RowKey>
column_keys_by_name: HashMap<String, ColumnKey>
```

Row/column metadata stores `user_key: Option<String>` for reverse lookup.

### Coordinate
- Use existing `Coordinate` type from core (row, column indices).
- Coordinate is *display position* and may change when sorted.

### Cell Value
- Python accepts str/float/renderable; it formats to a renderable per cell.
- For Rust parity, define a `CellValue` enum with common variants and a fallback:
```rust
pub enum CellValue {
    Text(Content),
    Str(String),
    Int(i64),
    Float(f64),
    Renderable(Box<dyn Renderable>),
    Empty,
}
```
- A `CellFormatter` trait can map `CellValue` -> `Content` for rendering.

## Data Structures

### Row/Column Metadata
```rust
pub struct Column {
    pub key: ColumnKey,
    pub label: Content,
    pub width: u16,
    pub content_width: u16,
    pub auto_width: bool,
    pub user_key: Option<String>,
}

pub struct Row {
    pub key: RowKey,
    pub height: u16,
    pub label: Option<Content>,
    pub auto_height: bool,
    pub user_key: Option<String>,
}
```

### Storage
- Mirror Python: `_data[row_key][column_key]`.
- Rust equivalent:
```rust
pub struct DataTableModel {
    pub rows: HashMap<RowKey, Row>,
    pub columns: HashMap<ColumnKey, Column>,
    pub data: HashMap<RowKey, HashMap<ColumnKey, CellValue>>,

    // Display order (sorting reorders these, not the data)
    pub row_order: Vec<RowKey>,
    pub column_order: Vec<ColumnKey>,

    // Fast lookup
    pub row_index: HashMap<RowKey, usize>,
    pub column_index: HashMap<ColumnKey, usize>,
}
```

Rationale:
- Keeps updates localized (no reallocation of entire table on add/remove column).
- Sorting reorders `row_order` only (matches Python `_row_locations`).
- Rows/columns remain stable via keys.

### Special Keys
- Header row and row-label column are not part of `rows`/`columns`.
- Use dedicated fields instead of inserting sentinel entries:
```rust
pub struct DataTableModel {
    pub header_row_key: RowKey,
    pub label_column_key: ColumnKey,
}
```
- These keys are used for rendering and style caching, not for data storage.

## Invariants
- `row_order.len() == rows.len()` and each entry exists in `rows`.
- `column_order.len() == columns.len()` and each entry exists in `columns`.
- `row_index`/`column_index` mirror the order vectors.
- `data` may omit cells; missing values resolve to `CellValue::Empty`.

## Key Operations

### Add Column
- Create ColumnKey (new or from user string).
- Insert into `columns`, `column_order`, `column_index`.
- For each existing row: insert default cell into `data[row_key][column_key]`.
- Update `content_width` based on label + existing cells.

### Add Row
- Create RowKey (new or from user string).
- Insert into `rows`, `row_order`, `row_index`.
- Initialize `data[row_key]` with provided cells mapped to columns.
- Track `auto_height` if height is None.

### Sort
- Reorder `row_order` only (stable keys preserved).
- Rebuild `row_index` accordingly.

## Compatibility Notes
- Use `HashMap<String, RowKey>` and `HashMap<String, ColumnKey>` to match Python’s behavior of accepting string keys for lookup.
- Provide helpers:
  - `row_key_from_str(&str) -> Option<RowKey>`
  - `column_key_from_str(&str) -> Option<ColumnKey>`

## Open Questions (Phase 2)
- Should `CellValue` be a trait object (`Box<dyn Renderable>`) or a sealed enum?
- Should we store `Content` directly to avoid repeated formatting?
- Do we need a small-string optimization for user keys?
