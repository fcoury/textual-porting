# Tree Node Management (Python Textual)

Source: `textual/src/textual/widgets/_tree.py`

## Core Model
- `TreeNode<T>` represents a node in the tree with:
  - `id: NodeID` (monotonic integer wrapped in `NewType`).
  - `label: Text` (processed via `Tree.process_label`).
  - `data: T | None` payload.
  - `parent: TreeNode | None`, `_children: list[TreeNode]`.
  - `expanded`, `allow_expand` flags.
  - `_hover`, `_selected` flags for UI state.
  - `_updates` counter (per-node invalidation); `_line` cache of rendered line index.

- `Tree<T>` holds:
  - `root: TreeNode<T>`.
  - `_tree_nodes: dict[NodeID, TreeNode]` for ID lookup.
  - `_current_id` counter for node IDs.
  - `_tree_lines_cached: list[_TreeLine] | None` for flattening visible lines.
  - `_line_cache: LRUCache` for rendered lines.
  - `cursor_line`, `hover_line`, `cursor_node` state.

## Node Management
- `TreeNode.add(...)` inserts new child at index or before/after a sibling.
- `TreeNode.add_leaf(...)` adds non-expandable node.
- `TreeNode.remove()` removes node (cannot remove root; raises `RemoveRootError`).
- `TreeNode.remove_children()` clears children.
- `Tree.clear()` resets tree while keeping root label/data.
- `Tree.reset(label, data)` clears and replaces root label/data.
- `_add_node(parent, label, data, expand)` creates new node, registers it, updates `_updates`.

## Expand/Collapse
- `TreeNode.expand()` / `collapse()` toggle node visibility and post messages:
  - `Tree.NodeExpanded`, `Tree.NodeCollapsed`.
- `expand_all()` / `collapse_all()` cascade to descendants.
- `toggle()` / `toggle_all()` invert and optionally cascade.
- `Tree._toggle_node(node)` respects `allow_expand`.

## Cursor & Selection
- `cursor_line` is the selected line index (-1 = none).
- `cursor_node` is tracked separately; set in `watch_cursor_line`.
- `move_cursor(node, animate)` sets cursor_line from node._line and scrolls.
- `move_cursor_to_line(line)` maps line -> node -> move.
- `select_node(node)` moves cursor + posts `NodeSelected`.
- `unselect()` resets cursor_line to -1.

## Line Construction (Flattening)
- `_TreeLine` holds `path: list[TreeNode]` and `last` flag; `node` property = last in path.
- `Tree._build()`:
  - DFS traversal to build visible lines.
  - Sets each node’s `_line` to its line index.
  - Respects `show_root` and `node._expanded`.
  - Computes `virtual_size` from max label width + guide width.
  - Adjusts cursor_line if out of range.

## Rendering-Relevant State
- `hover_line` and `cursor_line` drive visual feedback and component classes.
- `watch_hover_line` toggles `node._hover` and refreshes nodes.
- `watch_cursor_line` toggles `node._selected` and posts `NodeHighlighted` if changed.
- `render_label(node, base_style, style)` prefixes expand/collapse icon if allowed.

## Messages
- `NodeSelected`, `NodeHighlighted`, `NodeExpanded`, `NodeCollapsed`.
- `NodeSelected` also triggers `auto_expand` via `@on(NodeSelected)`.

## Styling Hooks (Component Classes)
- `tree--cursor`, `tree--guides`, `tree--guides-hover`, `tree--guides-selected`,
  `tree--highlight`, `tree--highlight-line`, `tree--label`.
- Line segments include style meta: `line`, `node`, and `toggle` for click detection.

## Implications for Rust Port
- Need stable NodeID registry and path-based flattened list for render.
- Must preserve node-level `_updates` to key caches for hover/selection redraws.
- Cursor is line-based; node -> line mapping is essential for scrolling.
- Expand/collapse and insert/remove operations must invalidate cached line list.
- Mouse interactions depend on segment metadata (`line` and `toggle`).
