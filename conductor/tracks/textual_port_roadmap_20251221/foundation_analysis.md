# Foundation Analysis

## Python Textual Repository Structure

### Core Modules (`src/textual/`)

| Module | Size | Description |
|--------|------|-------------|
| `app.py` | 178KB | Main App class - lifecycle, compose, run loop, screens |
| `widget.py` | 173KB | Base Widget class - rendering, layout, events |
| `screen.py` | 73KB | Screen management, compositor integration |
| `dom.py` | 63KB | DOM tree, node hierarchy, queries |
| `content.py` | 62KB | Content/text rendering with Rich integration |
| `message_pump.py` | 32KB | Message/event dispatch system |
| `events.py` | 26KB | Event types (key, mouse, resize, focus) |
| `reactive.py` | 18KB | Reactive properties system |
| `binding.py` | 14KB | Key bindings and actions |
| `geometry.py` | 41KB | Offset, Region, Size, Spacing |
| `timer.py` | 6KB | Timer/interval system |
| `worker.py` | 14KB | Background worker threads |

### CSS System (`src/textual/css/`)

| Module | Size | Description |
|--------|------|-------------|
| `styles.py` | 51KB | Style properties and computed styles |
| `stylesheet.py` | 27KB | Stylesheet parsing and application |
| `_style_properties.py` | 43KB | Individual CSS property definitions |
| `_styles_builder.py` | 49KB | Style construction |
| `parse.py` | 17KB | CSS parsing |
| `query.py` | 16KB | CSS selector queries |
| `tokenize.py` | 11KB | CSS tokenization |
| `scalar.py` | 10KB | Scalar values (px, %, fr, etc.) |

### Layout System (`src/textual/layouts/`)

| Module | Description |
|--------|-------------|
| `vertical.py` | Vertical stacking layout |
| `horizontal.py` | Horizontal stacking layout |
| `grid.py` | CSS Grid-like layout |
| `stream.py` | Streaming/flow layout |

### Widgets (`src/textual/widgets/`)

#### Basic Display Widgets
- `_static.py` (3KB) - Static text display
- `_label.py` (2KB) - Simple label
- `_placeholder.py` (6KB) - Placeholder widget
- `_rule.py` (7KB) - Horizontal/vertical rule
- `_digits.py` (3KB) - Large digit display
- `_sparkline.py` (4KB) - Inline charts

#### Interactive Widgets
- `_button.py` (15KB) - Clickable button
- `_input.py` (41KB) - Text input field
- `_masked_input.py` (26KB) - Masked input
- `_checkbox.py` (1KB) - Checkbox toggle
- `_radio_button.py` (1KB) - Radio button
- `_radio_set.py` (11KB) - Radio button group
- `_switch.py` (6KB) - Toggle switch
- `_toggle_button.py` (9KB) - Toggle button

#### Container Widgets
- `containers.py` (9KB) - Vertical, Horizontal, Grid, ScrollableContainer
- `_collapsible.py` (8KB) - Collapsible sections
- `_tabbed_content.py` (24KB) - Tabbed content
- `_tabs.py` (27KB) - Tab bar
- `_content_switcher.py` (4KB) - Content switching

#### Data Display Widgets
- `_data_table.py` (108KB) - Full data table with sorting, selection
- `_tree.py` (52KB) - Tree view
- `_directory_tree.py` (20KB) - File system tree
- `_list_view.py` (14KB) - List view
- `_list_item.py` (1KB) - List item
- `_option_list.py` (34KB) - Option list
- `_select.py` (24KB) - Dropdown select
- `_selection_list.py` (25KB) - Multi-select list

#### Rich Content Widgets
- `_markdown.py` (53KB) - Markdown rendering
- `_rich_log.py` (12KB) - Rich text log
- `_log.py` (12KB) - Simple log
- `_text_area.py` (100KB) - Multi-line text editor
- `_pretty.py` (1KB) - Pretty-printed content

#### Utility Widgets
- `_header.py` (6KB) - App header
- `_footer.py` (11KB) - App footer with bindings
- `_loading_indicator.py` (3KB) - Loading spinner
- `_progress_bar.py` (13KB) - Progress bar
- `_toast.py` (6KB) - Toast notifications
- `_tooltip.py` (0.5KB) - Tooltips
- `_link.py` (2KB) - Clickable links
- `_welcome.py` (2KB) - Welcome screen

---

## textual-rs Foundation Capabilities

### Core Modules

| Module | Lines | Description |
|--------|-------|-------------|
| `geometry.rs` | ~600 | Size, Rect, Constraint, LayoutHints |
| `layout.rs` | ~1200 | Dock layout system, LayoutCache |
| `style.rs` | ~1100 | TCSS parsing, Style, StyleSheet |
| `widget.rs` | ~300 | Widget trait, WidgetId, WidgetRegistry |
| `focus.rs` | ~300 | FocusManager, FocusableRegistry, tab navigation |
| `mouse.rs` | ~200 | MouseEvent, hit testing, dispatch |
| `app.rs` | ~150 | App trait, WidgetApp trait, run loop |
| `message.rs` | ~70 | Message trait for event passing |
| `terminal.rs` | ~60 | Terminal setup/cleanup with crossterm |

### Implemented Widgets

| Widget | Features |
|--------|----------|
| `Label` | Text display, alignment, styling |
| `Header` | Title bar with app name |
| `Footer` | Key binding display |
| `Button` | Click events, focus states, pressed feedback |
| `Input` | Text entry, cursor, placeholder, submit/change events |

### Key Features Built

1. **Geometry System**
   - Size with width/height
   - Rect with position and size
   - Constraint enum (Fixed, AtLeast, AtMost, Between, Unbounded)
   - LayoutHints for widget sizing preferences

2. **Dock Layout**
   - Dock enum (Top, Bottom, Left, Right, Fill)
   - Recursive layout computation
   - Layout caching with invalidation

3. **TCSS Styling**
   - CSS property parsing (color, background, border, margin, padding)
   - Selector matching (type, class, ID, pseudo-classes)
   - Style cascading and specificity
   - Pseudo-class support (:hover, :focus, :active)

4. **Focus Management**
   - Tab order tracking
   - Forward/backward navigation
   - Focus state callbacks

5. **Mouse Support**
   - Mouse event capture (click, scroll, move)
   - Hit testing against widget bounds
   - Event targeting to widgets

6. **App Lifecycle**
   - Basic run loop with crossterm
   - Event polling
   - Terminal raw mode management

---

## Gap Analysis: Python Textual vs textual-rs

### Tier 1: Core App Lifecycle (HIGH PRIORITY)

| Feature | Python Textual | textual-rs | Gap |
|---------|---------------|------------|-----|
| App.compose() | ✅ Async compose method | ❌ None | Need compose pattern |
| App.mount() | ✅ Dynamic widget mounting | ❌ None | Need mount/unmount |
| Screen stack | ✅ Push/pop screens | ❌ None | Need screen management |
| Message passing | ✅ Async message pump | ⚠️ Basic trait | Need full async dispatch |
| Actions | ✅ String-based actions | ❌ None | Need action system |
| Bindings | ✅ Key bindings | ❌ None | Need binding system |
| Workers | ✅ Background threads | ❌ None | Need worker system |
| Reactive | ✅ Reactive properties | ❌ None | Need reactive system |
| DOM queries | ✅ CSS selectors | ❌ None | Need query system |
| Timers | ✅ set_timer/set_interval | ❌ None | Need timer system |

### Tier 2: Layout System

| Feature | Python Textual | textual-rs | Gap |
|---------|---------------|------------|-----|
| Vertical layout | ✅ | ⚠️ Via Dock | Need proper layout |
| Horizontal layout | ✅ | ⚠️ Via Dock | Need proper layout |
| Grid layout | ✅ | ❌ | Need grid implementation |
| Scrolling | ✅ ScrollView | ❌ | Need scroll support |
| Docking | ✅ dock: top/bottom/left/right | ✅ | Done |
| Arrangement | ✅ _arrange.py | ⚠️ Basic | Need full arrange |

### Tier 2: TCSS Styling

| Feature | Python Textual | textual-rs | Gap |
|---------|---------------|------------|-----|
| Basic properties | ✅ | ✅ | Done |
| Selectors | ✅ Complex selectors | ⚠️ Basic | Need combinators |
| Pseudo-classes | ✅ :hover, :focus, etc | ⚠️ Basic | Partial |
| Animations | ✅ CSS animations | ❌ | Need animation system |
| Transitions | ✅ CSS transitions | ❌ | Need transitions |
| Variables | ✅ CSS variables | ❌ | Need variables |
| Themes | ✅ Theme system | ❌ | Need theme support |

### Tier 3: Basic Widgets

| Widget | Python Textual | textual-rs | Gap |
|--------|---------------|------------|-----|
| Static | ✅ | ❌ | Need implementation |
| Label | ✅ | ✅ | Done |
| Button | ✅ | ✅ | Done |
| Input | ✅ | ✅ | Done (basic) |
| Checkbox | ✅ | ❌ | Need implementation |
| RadioButton | ✅ | ❌ | Need implementation |
| Switch | ✅ | ❌ | Need implementation |
| ProgressBar | ✅ | ❌ | Need implementation |
| Rule | ✅ | ❌ | Need implementation |
| Link | ✅ | ❌ | Need implementation |

### Tier 3: Container Widgets

| Widget | Python Textual | textual-rs | Gap |
|--------|---------------|------------|-----|
| Vertical | ✅ | ❌ | Need implementation |
| Horizontal | ✅ | ❌ | Need implementation |
| Grid | ✅ | ❌ | Need implementation |
| ScrollableContainer | ✅ | ❌ | Need scrolling |
| TabbedContent | ✅ | ❌ | Need implementation |
| Collapsible | ✅ | ❌ | Need implementation |

### Tier 4: Advanced Widgets

| Widget | Python Textual | textual-rs | Gap |
|--------|---------------|------------|-----|
| DataTable | ✅ | ❌ | Complex - 108KB in Python |
| Tree | ✅ | ❌ | Complex - 52KB in Python |
| DirectoryTree | ✅ | ❌ | Needs Tree first |
| TextArea | ✅ | ❌ | Very complex - 100KB |
| Markdown | ✅ | ❌ | Rich integration needed |
| Select | ✅ | ❌ | Dropdown behavior |
| OptionList | ✅ | ❌ | List with selection |

---

## Recommended Track Priorities

### Track 1: Core App Lifecycle
- compose() pattern
- mount/unmount
- Screen stack
- Message pump
- Key bindings
- Actions

### Track 2: Layout System
- Vertical/Horizontal layouts
- Grid layout
- Scroll view
- Viewport management

### Track 3: TCSS Styling
- Selector combinators
- CSS animations
- Transitions
- Variables
- Themes

### Track 4: Basic Widgets
- Static, Checkbox, RadioButton, Switch
- ProgressBar, Rule, Link
- Loading indicators

### Track 5: Container Widgets
- Vertical, Horizontal, Grid containers
- ScrollableContainer
- TabbedContent, Collapsible

### Track 6: Advanced Widgets
- DataTable
- Tree/DirectoryTree
- TextArea
- Markdown
- Select/OptionList
