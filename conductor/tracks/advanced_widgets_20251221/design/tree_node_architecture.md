# TreeNode<T> Generic Architecture (Design)

## Goals
- Match Python Textual semantics: node identity, parent/child relations, expand/collapse, hover/selection state.
- Support stable `NodeId` lookups and cursor by line index.
- Enable Tree to flatten visible nodes into lines for render and hit testing.

## Core Types

### NodeId
- Stable identifier for node lookup.
- Prefer `slotmap::Key` or `u64` newtype.

```rust
#[derive(Clone, Copy, Debug, Eq, Hash, PartialEq)]
pub struct NodeId(slotmap::Key);
```

### TreeNode<T>
- Stored in Tree arena; nodes reference children by `NodeId`.

```rust
pub struct TreeNode<T> {
    pub id: NodeId,
    pub parent: Option<NodeId>,
    pub children: Vec<NodeId>,
    pub label: Content,
    pub data: Option<T>,
    pub expanded: bool,
    pub allow_expand: bool,
    pub hover: bool,
    pub selected: bool,
    pub line_index: i32,
    pub updates: u64,
}
```

### Tree<T>
```rust
pub struct Tree<T> {
    pub root: NodeId,
    pub nodes: SlotMap<NodeId, TreeNode<T>>,
    pub cursor_line: i32,
    pub cursor_node: Option<NodeId>,
    pub hover_line: i32,
    pub show_root: bool,
    pub show_guides: bool,
    pub guide_depth: u16,
    pub auto_expand: bool,
}
```

## Node Management
- `add(parent, label, data, before/after, expand, allow_expand)` inserts node in parent’s child list.
- `add_leaf` sets `allow_expand=false`.
- `remove(node)` deletes node and descendants; root removal forbidden.
- `clear(root)` removes all children.

## Line Flattening
- Build `_tree_lines: Vec<TreeLine>` where each line holds:
  - `path: Vec<NodeId>` from root to node
  - `last: bool` (last sibling)
- During build, set `line_index` on each node for cursor mapping.
- `show_root=false` starts traversal at root’s children.

## Cursor & Hover
- Cursor is line-based (matches Python). `cursor_line` maps to node via `_tree_lines`.
- Hover from mouse events uses line metadata; sets node `hover` state.
- `watch_cursor_line` toggles `selected` flags and posts `NodeHighlighted`.

## Messages
- `NodeSelected`, `NodeHighlighted`, `NodeExpanded`, `NodeCollapsed` messages include NodeId + data.
- `auto_expand` toggles expansion on selection.

## Styling Hooks
- Component classes: `tree--cursor`, `tree--guides`, `tree--highlight`, etc.
- `render_label(node, base_style, label_style)` to customize icons.

## Open Questions
- Do we store `Content` per node, or accept `impl Into<Content>` on insert?
- Should `TreeNode` expose `siblings()` as iterator for convenience?
- Do we need weak references for parent traversal? (likely no with arena)
