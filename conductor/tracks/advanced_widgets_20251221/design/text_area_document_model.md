# TextArea Document Model (Design)

## Goals
- Preserve Python Textual semantics: document-space cursor, wrapped navigation, selection ranges.
- Support syntax-aware documents (tree-sitter) when available, with fallback to plain text.
- Keep editing operations centralized for undo/redo.

## Model Choice

### Option A: Rope (ropey)
**Pros**
- Efficient edits in large documents.
- Cheap slicing and line lookup.

**Cons**
- Additional dependency + complexity.
- Requires custom mapping for tabs and wrapped offsets.

### Option B: Vec<String> (line-based)
**Pros**
- Matches Python model closely.
- Simple line operations, easy to map to wrapping logic.
- Good for typical TUI-sized documents.

**Cons**
- Larger edits on big docs can be slower.

### Decision
**Start with Vec<String> line model** to match Python parity and simplify porting. Provide a `DocumentBase` trait so a future Rope-backed implementation can be swapped in.

## Core Types

### DocumentBase Trait
```rust
pub trait DocumentBase {
    fn replace_range(&mut self, start: Location, end: Location, text: &str) -> EditResult;
    fn text(&self) -> String;
    fn newline(&self) -> Newline;
    fn lines(&self) -> &[String];
    fn get_line(&self, index: usize) -> &str;
    fn get_text_range(&self, start: Location, end: Location) -> String;
    fn get_size(&self, tab_width: u16) -> Size;
    fn line_count(&self) -> usize;
    fn start(&self) -> Location;
    fn end(&self) -> Location;
}
```

### Document (Vec<String>)
- Stores:
  - `newline: Newline` (`\n`, `\r\n`, `\r`)
  - `lines: Vec<String>` (no newline chars)
- If source ends with newline, append empty trailing line.

### SyntaxAwareDocument
- Wraps `Document` + tree-sitter parser/tree.
- Exposes `query_syntax_tree` and `prepare_query`.

### WrappedDocument
- Contains a reference to Document and computed wrap info:
  - per-line wrap offsets
  - per-line tab widths
  - mapping between (row, col) <-> (x, y) offsets

### DocumentNavigator
- Uses WrappedDocument to move cursor in wrapped space.
- Stores `last_x_offset` to preserve horizontal position in vertical movement.

## Editing Pipeline
- `Edit` object: `{ insert, start, end, maintain_selection_offset }`.
- `TextArea.edit(edit)`:
  1. `document.replace_range`
  2. update selection (if maintain offset)
  3. push to `EditHistory`
  4. update wrapped document for affected range
  5. rebuild highlight map

## Selection Model
- `Selection { start: Location, end: Location }`
- Cursor is always `Selection.end`.
- Selection is always in document-space.

## Open Questions
- Do we want to store `Content` lines in the document to avoid reformatting, or keep raw strings?
- Should WrappedDocument own precomputed cell-width for each line segment?
