# Specification: Advanced Widgets

## Overview
This track implements the most complex widgets in Textual: data tables, tree views, multi-line text editors, markdown rendering, and virtualized lists. These widgets require sophisticated rendering, data management, and user interaction handling.

## Goals
- Implement DataTable with sorting, selection, and cursors
- Implement Tree and DirectoryTree for hierarchical data
- Implement TextArea with syntax highlighting
- Implement Markdown (basic rendering) and MarkdownViewer (with navigation/ToC)
- Implement ListView with virtualization
- Implement Log and RichLog (scrolling text display with Rich formatting)

## Reference: Python Textual Advanced Widgets

### DataTable (108KB)
```python
class DataTable(ScrollView):
    """A powerful data table with rows, columns, and cursors."""

    # Cursors
    cursor_type: Literal["cell", "row", "column", "none"]
    cursor_coordinate: Coordinate
    cursor_row: int
    cursor_column: int

    # Selection
    show_cursor: bool
    show_header: bool
    show_row_labels: bool
    fixed_rows: int
    fixed_columns: int
    zebra_stripes: bool

    # Data management
    def add_column(self, label: str, *, key: str = None, width: int = None) -> ColumnKey: ...
    def add_row(self, *cells, key: str = None, height: int = None) -> RowKey: ...
    def add_columns(self, *labels) -> list[ColumnKey]: ...
    def add_rows(self, rows: Iterable) -> list[RowKey]: ...
    def remove_row(self, row_key: RowKey) -> None: ...
    def remove_column(self, column_key: ColumnKey) -> None: ...
    def clear(self, columns: bool = False) -> None: ...
    def get_cell(self, row_key: RowKey, column_key: ColumnKey) -> CellType: ...
    def get_cell_at(self, coordinate: Coordinate) -> CellType: ...
    def update_cell(self, row_key: RowKey, column_key: ColumnKey, value: CellType) -> None: ...
    def update_cell_at(self, coordinate: Coordinate, value: CellType) -> None: ...

    # Sorting
    def sort(self, *columns, reverse: bool = False) -> None: ...

    # Messages
    class CellHighlighted(Message): ...
    class CellSelected(Message): ...
    class RowHighlighted(Message): ...
    class RowSelected(Message): ...
    class ColumnHighlighted(Message): ...
    class ColumnSelected(Message): ...
    class HeaderSelected(Message): ...
    class RowLabelSelected(Message): ...
```

### Tree (52KB)
```python
class Tree(ScrollView, Generic[TreeDataType]):
    """A hierarchical tree view with expandable nodes."""

    root: TreeNode[TreeDataType]
    cursor_node: TreeNode[TreeDataType]
    show_root: bool
    show_guides: bool
    guide_depth: int
    auto_expand: bool

    # Tree manipulation
    def clear(self) -> None: ...
    def reset(self, label: str, data: TreeDataType = None) -> None: ...
    def get_node_at_line(self, line: int) -> TreeNode[TreeDataType] | None: ...
    def get_node_by_id(self, node_id: NodeID) -> TreeNode[TreeDataType]: ...
    def select_node(self, node: TreeNode[TreeDataType]) -> None: ...

    # Messages
    class NodeSelected(Message): ...
    class NodeExpanded(Message): ...
    class NodeCollapsed(Message): ...
    class NodeHighlighted(Message): ...

class TreeNode(Generic[TreeDataType]):
    """A node in a Tree."""
    tree: Tree[TreeDataType]
    parent: TreeNode[TreeDataType] | None
    data: TreeDataType
    label: TextType
    is_expanded: bool
    is_root: bool
    allow_expand: bool

    def add(self, label: str, data: TreeDataType = None) -> TreeNode[TreeDataType]: ...
    def add_leaf(self, label: str, data: TreeDataType = None) -> TreeNode[TreeDataType]: ...
    def remove(self) -> None: ...
    def expand(self) -> None: ...
    def collapse(self) -> None: ...
    def toggle(self) -> None: ...
```

### DirectoryTree (20KB)
```python
class DirectoryTree(Tree[DirEntry]):
    """A tree view of a file system directory."""

    path: str | Path
    show_hidden: bool

    class FileSelected(Message): ...
    class DirectorySelected(Message): ...

    def reload(self) -> None: ...
    def filter_paths(self, paths: Iterable[Path]) -> Iterable[Path]: ...
```

### TextArea (100KB)
```python
class TextArea(ScrollView):
    """A multi-line text editor with optional syntax highlighting."""

    text: str
    language: str | None  # For syntax highlighting
    theme: str
    show_line_numbers: bool
    read_only: bool
    tab_behavior: Literal["focus", "indent"]
    indent_width: int
    soft_wrap: bool

    # Selection
    selection: Selection
    cursor_location: Location
    cursor_at_first_row: bool
    cursor_at_last_row: bool

    # Actions
    def load_text(self, text: str) -> None: ...
    def clear(self) -> None: ...
    def insert(self, text: str) -> None: ...
    def delete(self, start: Location, end: Location) -> str: ...
    def get_text_range(self, start: Location, end: Location) -> str: ...
    def select_line(self, line_index: int) -> None: ...
    def select_all(self) -> None: ...

    # Messages
    class Changed(Message): ...
    class SelectionChanged(Message): ...

class Location(NamedTuple):
    row: int
    column: int
```

### Markdown (53KB)
```python
class Markdown(Widget):
    """Render Markdown content."""

    # Renders Markdown to terminal
    # Supports headings, lists, code blocks, links, etc.

    class TableOfContentsUpdated(Message): ...
    class TableOfContentsSelected(Message): ...
    class LinkClicked(Message): ...

    def update(self, markdown: str) -> None: ...
    def goto_anchor(self, anchor: str) -> bool: ...

class MarkdownViewer(Markdown):
    """Markdown with navigation and table of contents."""
    show_table_of_contents: bool
```

### ListView (14KB)
```python
class ListView(ScrollView):
    """A vertical list of items with virtualization."""

    index: int | None  # Currently highlighted index

    class Highlighted(Message): ...
    class Selected(Message): ...

    def append(self, item: ListItem) -> None: ...
    def clear(self) -> None: ...
    def remove(self, index: int) -> None: ...

class ListItem(Widget):
    """An item in a ListView."""
    highlighted: bool
```

### Log (12KB)
```python
class Log(ScrollView):
    """Scrolling text log widget."""

    max_lines: int | None
    auto_scroll: bool
    highlight: bool
    markup: bool

    def write_line(self, line: str) -> None: ...
    def write_lines(self, lines: Iterable[str]) -> None: ...
    def clear(self) -> None: ...

    # Efficient line management for large logs
    # Automatic scrolling to bottom on new content
```

### RichLog (12KB)
```python
class RichLog(ScrollView):
    """Scrolling log with Rich renderable support."""

    max_lines: int | None
    auto_scroll: bool
    highlight: bool
    markup: bool
    min_width: int

    def write(self, content: RenderableType, width: int = None) -> None: ...
    def clear(self) -> None: ...

    # Supports Rich renderables (tables, panels, syntax highlighting)
    # Deferred rendering for performance
    # Width calculation for proper wrapping
```

## Deliverables

### Phase 1: Analysis & Research
- Study _data_table.py in detail (108KB)
- Study _tree.py architecture (52KB)
- Study _directory_tree.py file system integration
- Study _text_area.py editor implementation (100KB)
- Study _markdown.py rendering approach
- Study _list_view.py virtualization

### Phase 2: Design & Planning
- Design DataTable data structures (RowKey, ColumnKey, Coordinate)
- Design TreeNode with generic data type
- Design TextArea document model
- Design Markdown parser integration
- Design virtualization strategy for ListView
- Plan syntax highlighting integration

### Phase 3: Implementation

#### 3.1 DataTable
- Data structures (RowKey, ColumnKey, Coordinate, CellType)
- Row and column management
- Cursor types (cell, row, column, none)
- Fixed rows/columns
- Zebra stripes
- Sorting
- All messages (CellSelected, RowSelected, etc.)

#### 3.2 Tree
- TreeNode generic type
- Tree structure management
- Expand/collapse
- Guide lines rendering
- Cursor navigation
- All messages

#### 3.3 DirectoryTree
- Extend Tree with file system
- Path loading
- Hidden files filter
- FileSelected/DirectorySelected messages
- Lazy loading of directories

#### 3.4 TextArea
- Document model (lines, cursor, selection)
- Syntax highlighting (tree-sitter integration?)
- Line numbers
- Soft wrap
- Undo/redo
- All editing operations
- Changed/SelectionChanged messages

#### 3.5 Markdown
- Markdown parsing
- Heading rendering
- Code block rendering
- List rendering
- Link handling
- Table of contents
- Anchor navigation

#### 3.6 ListView
- Virtualized rendering
- ListItem management
- Highlighted/Selected messages
- Keyboard navigation
- Append/remove/clear operations

### Phase 4: Testing & Verification
- Unit tests for each widget
- Performance tests for large data sets
- Example apps for each widget

## Success Criteria
- [ ] DataTable handles 10,000+ rows smoothly
- [ ] Tree supports deep hierarchies
- [ ] DirectoryTree browses file system
- [ ] TextArea edits multi-line text with highlighting
- [ ] Markdown renders all common Markdown features
- [ ] ListView virtualizes long lists

## Dependencies
- All Tier 1-3 tracks
- ScrollView (from Layout System)
- Syntax highlighting library (for TextArea)
- Markdown parser library

## Technical Challenges
- DataTable: Efficient cell rendering with many rows/columns
- Tree: Memory-efficient representation of large trees
- TextArea: Syntax highlighting performance
- ListView: Virtual scrolling without layout thrashing
