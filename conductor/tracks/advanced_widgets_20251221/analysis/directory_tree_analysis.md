# DirectoryTree File System Integration (Python Textual)

Source: `textual/src/textual/widgets/_directory_tree.py`

## Core Model
- Extends `Tree[DirEntry]`, where `DirEntry` holds:
  - `path: Path`
  - `loaded: bool` (prevents repeated loads)

## Icons & Styling
- Uses emoji icons for folders/files:
  - `ICON_NODE_EXPANDED = "📂 "`, `ICON_NODE = "📁 "`, `ICON_FILE = "📄 "`.
- Adds component classes:
  - `directory-tree--folder`, `directory-tree--file`, `directory-tree--extension`, `directory-tree--hidden`.
- DEFAULT_CSS styles folders bold, extensions italic, hidden entries dim.

## Loading & Async Model
- Maintains `_load_queue: asyncio.Queue[TreeNode]`.
- `@work(thread=True)` loads directory contents in a background thread (`_load_directory`).
- `_loader()` is an async worker that consumes `_load_queue` and populates nodes.
- `reload()` resets the queue, resets root node, and starts a fresh loader.
- `_add_to_load_queue(node)` marks `DirEntry.loaded = True`, enqueues node, returns `AwaitComplete` for join.

## Path Handling
- `PATH: Callable` defaults to `Path` to allow overriding.
- `validate_path` coerces input to `Path`.
- `_load_directory` calls `expanduser().resolve()` and sorts entries with dirs first.
- `_safe_is_dir` catches PermissionError and returns False.

## Population Rules
- `_populate_node(node, content)`:
  - Clears children, inserts entries with `allow_expand` set to `is_dir`.
  - Expands the node by default.
- `filter_paths(paths)` is overridable for hiding or customizing visibility.

## State Preservation on Reload
- `_reload(node)` captures:
  - expanded paths (for reopening)
  - currently highlighted path (cursor)
- After repopulating, restores expanded state and best matching cursor line.

## Selection Messages
- `FileSelected(node, path)` and `DirectorySelected(node, path)` posted on selection.
- `_on_tree_node_expanded`:
  - If directory, enqueue load.
  - If file, post FileSelected.
- `_on_tree_node_selected`:
  - If directory, post DirectorySelected.
  - If file, post FileSelected.

## Rendering Hooks
- Overrides `render_label` to:
  - Add file/folder icon.
  - Apply component class styles to folders/files/extensions/hidden.

## Implications for Rust Port
- Requires background loading (threaded) with cancellation.
- Must preserve expanded/cursor state on reload.
- Needs icon-aware label rendering and component-class styling.
- Should support filtering and permission-safe directory probing.
