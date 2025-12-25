# ListView Virtualization Strategy (Python Textual)

Source: `textual/src/textual/widgets/_list_view.py`

## Summary
- **No explicit virtualization.** `ListView` is a `VerticalScroll` container that mounts all `ListItem` children as real widgets.
- Scrolling is handled by the container; the list does not recycle rows or render only visible items.

## How it Works
- `ListView` is a `VerticalScroll` with `can_focus=True`, `can_focus_children=False`.
- All children are `ListItem` widgets; they stay mounted in the DOM.
- Highlighting is index-based (`index` reactive) and toggles the `ListItem.highlighted` flag.
- CSS applies highlight/hover styles directly to ListItem widgets.

## Implications
- Performance depends on number of mounted `ListItem`s.
- For very large lists, Textual uses other widgets (e.g. `DataTable` or custom virtualized widgets) rather than `ListView`.

## Porting Guidance (Rust)
- Implement ListView as a simple scroll container with full child mounting.
- Defer virtualization to other widgets (or introduce a new virtualized ListView if needed later).
- Keep selection/highlight logic index-based and map to child widget state.
