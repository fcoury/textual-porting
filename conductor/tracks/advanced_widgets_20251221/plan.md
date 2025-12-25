# Plan: Advanced Widgets

## Phase 1: Analysis & Research [checkpoint: b4b6ac5]
- [x] Task: Study Python Textual _data_table.py (108KB) in depth (162ec23)
- [x] Task: Document DataTable data structures and cursor system (a057e1e)
- [x] Task: Study _tree.py (52KB) node management (c298918)
- [x] Task: Analyze _directory_tree.py file system integration (da38d24)
- [x] Task: Study _text_area.py (100KB) document model (ceef420)
- [x] Task: Analyze TextArea syntax highlighting approach (60e689b)
- [x] Task: Study _markdown.py rendering pipeline (ade2921)
- [x] Task: Analyze _list_view.py virtualization strategy (9e62c2d)
- [x] Task: Study _log.py and _rich_log.py scrolling and formatting (1718b8c)
- [x] Task: Research Rust syntax highlighting libraries (tree-sitter, syntect) (a603537)
- [x] Task: Research Rust Markdown parsers (pulldown-cmark, comrak) (9eec4a6)
- [ ] Task: Conductor - User Manual Verification 'Analysis Complete' (Protocol in workflow.md)

## Phase 2: Design & Planning
- [x] Task: Design DataTable data model (Row, Column, Cell, Coordinate) (820d808)
- [x] Task: Design TreeNode<T> generic architecture (553b892)
- [x] Task: Design TextArea document model (rope vs Vec<String>) (881e526)
- [x] Task: Design syntax highlighting integration (abfc3f5)
- [x] Task: Design Markdown AST to terminal rendering (c681c70)
- [x] Task: Design virtualization strategy for ListView (3824766)
- [x] Task: Write technical design document (1d72e5e)
- [ ] Task: Conductor - User Manual Verification 'Design Approved' (Protocol in workflow.md)

## Phase 3: Implementation

### 3.1 DataTable Data Structures
- [ ] Task: Implement RowKey and ColumnKey types
- [ ] Task: Implement Coordinate type
- [ ] Task: Implement CellType trait/enum
- [ ] Task: Implement Row and Column structs
- [ ] Task: Write tests for data structures

### 3.2 DataTable Core
- [ ] Task: Implement DataTable struct extending ScrollView
- [ ] Task: Implement add_column() and add_columns()
- [ ] Task: Implement add_row() and add_rows()
- [ ] Task: Implement remove_row() and remove_column()
- [ ] Task: Implement clear()
- [ ] Task: Implement get_cell() and get_cell_at()
- [ ] Task: Implement update_cell() and update_cell_at()
- [ ] Task: Write tests for data management

### 3.3 DataTable Display
- [ ] Task: Implement cursor_type property
- [ ] Task: Implement cursor navigation
- [ ] Task: Implement show_header and show_row_labels
- [ ] Task: Implement fixed_rows and fixed_columns
- [ ] Task: Implement zebra_stripes
- [ ] Task: Implement column width calculation
- [ ] Task: Implement cell rendering
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for display

### 3.4 DataTable Sorting & Messages
- [ ] Task: Implement sort() with multi-column support
- [ ] Task: Implement CellHighlighted message
- [ ] Task: Implement CellSelected message
- [ ] Task: Implement RowHighlighted message
- [ ] Task: Implement RowSelected message
- [ ] Task: Implement ColumnHighlighted message
- [ ] Task: Implement ColumnSelected message
- [ ] Task: Implement HeaderSelected message
- [ ] Task: Write tests for sorting and messages

### 3.5 TreeNode
- [ ] Task: Implement TreeNode<T> generic struct
- [ ] Task: Implement parent/children relationships
- [ ] Task: Implement label and data properties
- [ ] Task: Implement is_expanded, is_root, allow_expand
- [ ] Task: Implement add() and add_leaf()
- [ ] Task: Implement remove()
- [ ] Task: Implement expand(), collapse(), toggle()
- [ ] Task: Write tests for TreeNode

### 3.6 Tree Widget
- [ ] Task: Implement Tree<T> extending ScrollView
- [ ] Task: Implement root property
- [ ] Task: Implement cursor_node
- [ ] Task: Implement show_root and show_guides
- [ ] Task: Implement guide line rendering
- [ ] Task: Implement clear() and reset()
- [ ] Task: Implement get_node_at_line()
- [ ] Task: Implement get_node_by_id()
- [ ] Task: Implement select_node()
- [ ] Task: Implement NodeSelected message
- [ ] Task: Implement NodeExpanded message
- [ ] Task: Implement NodeCollapsed message
- [ ] Task: Implement NodeHighlighted message
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Tree

### 3.7 DirectoryTree
- [ ] Task: Implement DirEntry struct
- [ ] Task: Implement DirectoryTree extending Tree<DirEntry>
- [ ] Task: Implement path property
- [ ] Task: Implement show_hidden
- [ ] Task: Implement lazy directory loading
- [ ] Task: Implement reload()
- [ ] Task: Implement filter_paths()
- [ ] Task: Implement FileSelected message
- [ ] Task: Implement DirectorySelected message
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for DirectoryTree

### 3.8 TextArea Document Model
- [ ] Task: Implement Location (row, column) type
- [ ] Task: Implement Selection type for TextArea
- [ ] Task: Implement Document struct (lines storage)
- [ ] Task: Implement cursor management
- [ ] Task: Implement selection management
- [ ] Task: Write tests for document model

### 3.9 TextArea Core
- [ ] Task: Implement TextArea extending ScrollView
- [ ] Task: Implement text property
- [ ] Task: Implement read_only property
- [ ] Task: Implement load_text() and clear()
- [ ] Task: Implement insert()
- [ ] Task: Implement delete()
- [ ] Task: Implement get_text_range()
- [ ] Task: Implement select_line() and select_all()
- [ ] Task: Implement Changed message
- [ ] Task: Implement SelectionChanged message
- [ ] Task: Write tests for core editing

### 3.10 TextArea Display
- [ ] Task: Implement show_line_numbers
- [ ] Task: Implement soft_wrap
- [ ] Task: Implement indent_width
- [ ] Task: Implement tab_behavior
- [ ] Task: Implement cursor rendering
- [ ] Task: Implement selection highlighting
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for display

### 3.11 TextArea Syntax Highlighting
- [ ] Task: Integrate syntax highlighting library
- [ ] Task: Implement language property
- [ ] Task: Implement theme property
- [ ] Task: Implement token-based rendering
- [ ] Task: Write tests for syntax highlighting

### 3.12 Markdown Parsing
- [ ] Task: Integrate Markdown parser library
- [ ] Task: Implement Markdown AST types
- [ ] Task: Implement heading parsing
- [ ] Task: Implement list parsing
- [ ] Task: Implement code block parsing
- [ ] Task: Implement link parsing
- [ ] Task: Write tests for parsing

### 3.13 Markdown Rendering
- [ ] Task: Implement Markdown widget
- [ ] Task: Implement heading rendering
- [ ] Task: Implement list rendering
- [ ] Task: Implement code block rendering with highlighting
- [ ] Task: Implement link rendering
- [ ] Task: Implement table rendering
- [ ] Task: Implement update() method
- [ ] Task: Implement TableOfContentsUpdated message
- [ ] Task: Implement LinkClicked message
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for rendering

### 3.14 MarkdownViewer
- [ ] Task: Implement MarkdownViewer extending Markdown
- [ ] Task: Implement show_table_of_contents
- [ ] Task: Implement table of contents panel
- [ ] Task: Implement goto_anchor()
- [ ] Task: Implement TableOfContentsSelected message
- [ ] Task: Write tests for MarkdownViewer

### 3.15 ListItem
- [ ] Task: Implement ListItem widget
- [ ] Task: Implement highlighted property
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for ListItem

### 3.16 ListView
- [ ] Task: Implement ListView extending ScrollView
- [ ] Task: Implement index property
- [ ] Task: Implement virtualized rendering
- [ ] Task: Implement append()
- [ ] Task: Implement remove()
- [ ] Task: Implement clear()
- [ ] Task: Implement keyboard navigation
- [ ] Task: Implement Highlighted message
- [ ] Task: Implement Selected message
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for ListView

### 3.17 Log Widget
- [ ] Task: Implement Log extending ScrollView
- [ ] Task: Implement max_lines property
- [ ] Task: Implement auto_scroll property
- [ ] Task: Implement highlight and markup properties
- [ ] Task: Implement write_line() method
- [ ] Task: Implement write_lines() method
- [ ] Task: Implement clear() method
- [ ] Task: Implement efficient line buffer management
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Log

### 3.18 RichLog Widget
- [ ] Task: Implement RichLog extending ScrollView
- [ ] Task: Implement max_lines and auto_scroll properties
- [ ] Task: Implement highlight and markup properties
- [ ] Task: Implement min_width property
- [ ] Task: Implement write() for Rich renderables
- [ ] Task: Implement deferred rendering for performance
- [ ] Task: Implement width calculation for wrapping
- [ ] Task: Implement clear() method
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for RichLog

- [ ] Task: Conductor - User Manual Verification 'Implementation Complete' (Protocol in workflow.md)

## Phase 4: Testing & Verification
- [ ] Task: Create DataTable example with 10,000 rows
- [ ] Task: Create Tree example with file browser
- [ ] Task: Create TextArea example with code editing
- [ ] Task: Create Markdown example with documentation
- [ ] Task: Create ListView example with large list
- [ ] Task: Create Log/RichLog example with streaming output
- [ ] Task: Performance benchmark all widgets
- [ ] Task: Run all tests and fix failures
- [ ] Task: Conductor - User Manual Verification 'Track Complete' (Protocol in workflow.md)
