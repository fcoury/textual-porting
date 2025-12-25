# ListView Virtualization Strategy (Design)

## Goal
Match Python Textual semantics: ListView mounts all children and relies on ScrollView for clipping. No virtualization is required for parity.

## Design Decision
- **No virtualization** in initial Rust port.
- Implement ListView as a `VerticalScroll` with all `ListItem` children mounted.
- Highlighting and selection logic stays index-based, mirroring Python.

## Rationale
- Python Textual ListView does not virtualize.
- Introducing virtualization would risk behavior differences (focus, events, child queries).
- For large data sets, recommend using DataTable or a future VirtualList widget.

## API Notes
- Keep `index: Option<usize>` reactive.
- `highlighted_child()` returns the current ListItem by index.
- Selection/Highlight messages mirror Python.

## Future Considerations
- If performance becomes a concern, introduce a separate VirtualList widget.
