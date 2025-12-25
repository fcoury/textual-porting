# Advanced Widgets: Technical Design

## Summary
This track ports Python Textual’s advanced widgets to Rust with high parity. The design follows Python’s structure: stable keys for data tables, arena-backed tree nodes with line flattening, a line-based document model for TextArea, tree-sitter driven syntax highlighting, and markdown-it–style widget trees for Markdown. ListView intentionally avoids virtualization for parity.

## Architecture Overview

### DataTable
- Data model uses stable keys (`RowKey`, `ColumnKey`) independent of sorting order.
- Rows/columns stored in HashMaps; ordering stored in vectors for reordering without data movement.
- Cells stored per-row in nested maps.
- Cursor and hover operate on display coordinates; keys remain stable.

### Tree / DirectoryTree
- Tree nodes are stored in an arena (SlotMap) with `NodeId` handles.
- `TreeLine` flattening creates a visible list of nodes and assigns `line_index` per node.
- Cursor is line-based (matches Python), and hover uses line metadata for hit testing.
- DirectoryTree adds async loading with queues and preserves expanded/cursor state on reload.

### TextArea
- Document model is line-based (`Vec<String>`) to match Python behavior.
- `WrappedDocument` provides visual wrapping and offset mapping.
- `DocumentNavigator` moves the cursor in wrapped space while keeping selection in document space.
- Edits flow through a single edit API for undo/redo and consistent rewrap.

### Syntax Highlighting
- Primary path uses tree-sitter with highlight queries loaded from `.scm` files.
- Highlight spans stored per line; applied before selection/cursor styles.
- Fallback to plain text (or syntect) when tree-sitter not available.

### Markdown
- Parsing via `pulldown-cmark` (GFM features enabled).
- Tokens/events mapped to Markdown block widgets (headers, lists, tables, fences, etc.).
- Inline spans use component-class styling and link click metadata.
- TOC built from headings and rendered with Tree widget.

### ListView
- No virtualization (matches Python). All ListItem children remain mounted.

## Key Design Documents
- DataTable model: `design/data_table_model.md`
- TreeNode architecture: `design/tree_node_architecture.md`
- TextArea document model: `design/text_area_document_model.md`
- Syntax highlighting: `design/syntax_highlighting_integration.md`
- Markdown rendering: `design/markdown_rendering.md`
- ListView virtualization: `design/list_view_virtualization.md`

## Phased Implementation Plan (Phase 3)
1) DataTable data structures + cursor + rendering caches.
2) Tree + DirectoryTree node management and line flattening.
3) TextArea document + navigation + selection.
4) Syntax highlighting integration with tree-sitter.
5) Markdown widgets and TOC.
6) ListView and ListItem.
7) Log + RichLog rendering.

## Compatibility & Parity Notes
- All widgets preserve Python semantics (keys, cursor behavior, selection, TOC, etc.).
- Styling uses TCSS component classes to match Python CSS.
- Avoid virtualization in ListView to prevent behavior drift.

## Open Questions
- Should we add an optional rope-based document implementation after parity?
- Do we want syntect fallback, or tree-sitter only?
- How aggressively should we optimize caching for DataTable/Tree before parity is achieved?
