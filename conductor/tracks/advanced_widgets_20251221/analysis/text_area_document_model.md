# TextArea Document Model (Python Textual)

Source: `textual/src/textual/widgets/_text_area.py` and `textual/src/textual/document/*`

## Core Document Abstractions
- `DocumentBase`: abstract interface required by TextArea.
  - `replace_range(start, end, text) -> EditResult`
  - `text`, `lines`, `newline`, `line_count`, `start`, `end`
  - `get_line`, `get_text_range`, `get_size(tab_width)`
- `Document`: plain text implementation.
  - Splits text into `_lines` (no newline chars); preserves trailing newline by appending empty line.
  - Tracks newline style (`\n`, `\r\n`, or `\r`).
- `SyntaxAwareDocument`: wraps tree-sitter parsing; can answer `query_syntax_tree`.
- `WrappedDocument`: view of document with wrapping + tab expansion.
  - Provides offset <-> location mapping for visual coordinates.
- `DocumentNavigator`: navigation helper that moves the cursor in *wrapped* visual space while keeping selection in document space.
- `EditHistory`: undo/redo checkpoint stack; populated by `Edit` operations.

## TextArea State Wiring
- `document: DocumentBase` is the source of truth for text.
- `wrapped_document: WrappedDocument` is rebuilt from `document` when:
  - text changes, tab width changes, or wrap width changes.
- `navigator: DocumentNavigator` depends on `wrapped_document` and must be recreated when wrapped content updates.
- Selection state: `selection: Selection` (document-space start/end).
  - `Selection.end` is always the cursor location.

## Document Construction (`_set_document`)
- When language is set and tree-sitter is available:
  - Resolve a `Language` (user-registered or built-in).
  - Attempt `SyntaxAwareDocument(text, language)`, fallback to `Document` on failure.
  - Build `_highlight_query` via tree-sitter and cache highlights per line.
- Without tree-sitter or no language: use `Document`.
- Always rebuild `wrapped_document`, `navigator`, highlights, and reset cursor to `(0, 0)`.

## Wrapping & Layout
- `wrap_width` = content width minus gutter and cursor width (when soft wrap enabled).
- `_rewrap_and_refresh_virtual_size()`:
  - Calls `wrapped_document.wrap(wrap_width, tab_width)`.
  - Clears line cache and recomputes virtual size based on wrapped height.
- `WrappedDocument` provides:
  - `location_to_offset` and `offset_to_location` mapping for cursor/selection.
  - `get_offsets` / `get_tab_widths` for per-line rendering decisions.

## Cursor & Selection Model
- Cursor lives in `selection.end` (document-space).
- Navigation uses `DocumentNavigator` to interpret movement in wrapped space.
- `move_cursor(location, select=False)` updates `Selection`.
- `record_cursor_width` stores last x-offset (cell width) to preserve column on vertical movement.

## Editing Model
- Editing API built on `Edit` objects:
  - `insert(text, location)` -> `Edit(text, start, end)`
  - `delete(start, end)` -> `Edit("", start, end)`
  - `replace(insert, start, end)`
- `edit(Edit)` performs:
  - `document.replace_range` (document-space change)
  - selection offset maintenance (optional)
  - history checkpointing via `EditHistory`
  - wrapped document rewrap/rebuild for affected range
  - highlight map rebuild

## Syntax Highlighting & Themes
- `_highlight_query` set only for syntax-aware documents.
- `_build_highlight_map()`:
  - Executes tree-sitter query and stores per-line highlight ranges.
- Highlighting combines:
  - theme styles (from `TextAreaTheme`) and
  - component class styles (`text-area--selection`, `text-area--cursor`, etc.).

## Rendering Implications
- Rendering uses document-space lines plus wrapped offsets to produce visible lines.
- Selections and cursor are rendered by mapping document locations to wrapped offsets.
- Gutter (line numbers) is computed from document line count and `line_number_start`.

## Implications for Rust Port
- Must preserve document as source of truth, with separate wrapped view.
- Navigation must be wrapping-aware; cursor stored in document coords.
- Editing operations should flow through a single edit abstraction to keep history, wrapping, and highlights in sync.
- Tree-sitter integration is optional but should follow the same fallback behavior.
