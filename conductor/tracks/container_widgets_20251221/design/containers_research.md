# Container Widgets Research

## Python Textual Container Hierarchy

Based on analysis of `containers.py` from Python Textual:

```
Widget
├── Container           # Base container with vertical layout
├── ScrollableContainer # Scrollable with keyboard bindings
├── Vertical           # Vertical layout (expanding)
├── VerticalGroup      # Vertical layout (collapsing)
├── VerticalScroll     # Vertical scroll only
├── Horizontal         # Horizontal layout (expanding)
├── HorizontalGroup    # Horizontal layout (collapsing)
├── HorizontalScroll   # Horizontal scroll only
├── Center             # Horizontal centering
├── Right              # Right alignment
├── Middle             # Vertical centering
├── CenterMiddle       # Both axis centering
├── Grid               # CSS Grid layout
└── ItemGrid           # Reactive grid with constraints
```

## DEFAULT_CSS Patterns

### Container
```css
Container {
    width: 1fr;
    height: 1fr;
    layout: vertical;
}
```

### Vertical / Horizontal
```css
Vertical {
    width: 1fr;
    height: 1fr;
    layout: vertical;
}

Horizontal {
    width: 1fr;
    height: 1fr;
    layout: horizontal;
}
```

### VerticalGroup / HorizontalGroup
```css
VerticalGroup {
    width: 1fr;
    height: auto;
    layout: vertical;
    overflow: hidden hidden;
}

HorizontalGroup {
    width: 1fr;
    height: auto;
    layout: horizontal;
    overflow: hidden hidden;
}
```

### Scrollable Containers
```css
ScrollableContainer {
    /* Inherits from Widget */
}

VerticalScroll {
    overflow-x: hidden;
    overflow-y: auto;
}

HorizontalScroll {
    overflow-y: hidden;
    overflow-x: auto;
}
```

### Alignment Containers
```css
Center {
    align-horizontal: center;
    width: 1fr;
    height: auto;
}

Middle {
    align-vertical: middle;
    width: auto;
    height: 1fr;
}

CenterMiddle {
    align: center middle;
    width: 1fr;
    height: 1fr;
}

Right {
    align-horizontal: right;
    width: 1fr;
    height: auto;
}
```

### Grid
```css
Grid {
    width: 1fr;
    height: 1fr;
    layout: grid;
}
```

### ItemGrid
```css
ItemGrid {
    width: 1fr;
    height: auto;
    layout: grid;
}
```

## Widget API Details

### Alignment Containers (Center, Middle, Right, CenterMiddle)

All alignment containers are simple wrappers with no custom properties, methods, or messages beyond what Widget provides.

| Widget | Purpose | Properties | Methods | Messages |
|--------|---------|------------|---------|----------|
| Center | Horizontally center children | Inherited only | Inherited only | None |
| Middle | Vertically center children | Inherited only | Inherited only | None |
| CenterMiddle | Center on both axes | Inherited only | Inherited only | None |
| Right | Right-align children | Inherited only | Inherited only | None |

### Group Containers (VerticalGroup, HorizontalGroup)

Group containers are non-expanding variants that shrink to content height while filling width.

| Widget | Purpose | Properties | Methods | Messages |
|--------|---------|------------|---------|----------|
| VerticalGroup | Vertical layout, auto height | Inherited only | Inherited only | None |
| HorizontalGroup | Horizontal layout, auto height | Inherited only | Inherited only | None |

**Key Difference from Vertical/Horizontal**:
- `height: auto` instead of `height: 1fr` (shrink to content)
- `overflow: hidden hidden` (no scrolling)

### ItemGrid

ItemGrid is an advanced grid container with reactive properties for dynamic column configuration.

**Reactive Properties**:
| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `stretch_height` | bool | false | Expand widgets to fill row height |
| `min_column_width` | int \| None | None | Minimum column width constraint |
| `max_column_width` | int \| None | None | Maximum column width constraint |
| `regular` | bool | false | Force equal items per row |

**Constructor Parameters**:
```python
ItemGrid(
    *children,
    stretch_height: bool = False,
    min_column_width: int | None = None,
    max_column_width: int | None = None,
    regular: bool = False,
    name: str | None = None,
    id: str | None = None,
    classes: str | None = None,
    disabled: bool = False,
)
```

**Methods**:
- `pre_layout()` - Applies reactive properties to GridLayout instance

**Messages**: None (inherited only)

## Existing textual-rs Infrastructure

### Already Implemented (scroll.rs)

1. **ScrollView Widget** - Full-featured scrollable container
   - `scroll_to(x, y)` - Scroll to absolute position
   - `scroll_by(dx, dy)` - Scroll by relative amount
   - `scroll_home()` - Jump to top-left
   - `scroll_end()` - Jump to bottom-right
   - `scroll_visible(region)` - Ensure region is visible

2. **ScrollBar Widget** - Visual scrollbar
   - Vertical and horizontal orientations
   - Track and thumb styling
   - Position tracking (0.0-1.0)

3. **ScrollState** - Core scroll state tracking
   - offset, virtual_size, viewport_size
   - `scroll_percent_x/y()`, `thumb_size_x/y()`
   - `page_up/down/left/right()`

4. **OverflowSettings** - Scrollbar visibility control
   - `Hidden`, `Scroll`, `Auto` modes
   - Independent X/Y control

### Already Implemented (container.rs)

1. **Container** - Generic grouping widget
2. **Horizontal** - Left-to-right layout
3. **Vertical** - Top-to-bottom layout
4. **Grid** - CSS Grid-like 2D layout

All support: ID, CSS classes, borders, width/height constraints, dock positioning.

## Gap Analysis

### What We Have
- Basic Container, Horizontal, Vertical, Grid widgets
- ScrollView with full scroll functionality
- ScrollBar widget
- Layout system with VerticalLayout, HorizontalLayout, GridLayout

### What We Need to Add

1. **VerticalScroll / HorizontalScroll** - Specialized scroll containers
   - Extend ScrollView with specific overflow settings
   - Match Python API

2. **ScrollableContainer** - Base scrollable with keyboard bindings
   - Keyboard navigation (arrows, page up/down, home/end)
   - Match Python API methods

3. **Center / Middle / CenterMiddle** - Alignment containers
   - Use align styles
   - Simple wrappers

4. **Right** - Right alignment container

5. **VerticalGroup / HorizontalGroup** - Non-expanding variants
   - Use `height: auto` instead of `height: 1fr` (shrink to content height)
   - Add `overflow: hidden hidden` (no scrolling)

6. **ItemGrid** - Advanced grid with reactive properties
   - `min_column_width`, `max_column_width`
   - `stretch_height`, `regular` options

## Keyboard Bindings for ScrollableContainer

From Python Textual:
```python
BINDINGS = [
    Binding("up", "scroll_up", "Scroll Up", show=False),
    Binding("down", "scroll_down", "Scroll Down", show=False),
    Binding("left", "scroll_left", "Scroll Left", show=False),
    Binding("right", "scroll_right", "Scroll Right", show=False),
    Binding("home", "scroll_home", "Scroll Home", show=False),
    Binding("end", "scroll_end", "Scroll End", show=False),
    Binding("pageup", "page_up", "Page Up", show=False),
    Binding("pagedown", "page_down", "Page Down", show=False),
    Binding("ctrl+pageup", "page_left", "Page Left", show=False),
    Binding("ctrl+pagedown", "page_right", "Page Right", show=False),
]
```

## Implementation Strategy

1. **Leverage existing infrastructure** - ScrollView, ScrollBar, ScrollState are complete
2. **Create thin wrappers** - Most containers are just CSS defaults + layout type
3. **Focus on API parity** - Match Python Textual method signatures
4. **Add keyboard bindings** - Use existing binding system from Basic Widgets

## Priority Order

1. Center, Middle, Right, CenterMiddle (trivial alignment wrappers)
2. VerticalGroup, HorizontalGroup (collapsing variants)
3. ScrollableContainer (keyboard bindings)
4. VerticalScroll, HorizontalScroll (overflow presets)
5. ItemGrid (advanced features)

## Detailed Implementation Plans

### Plan: Alignment Containers (Center, Middle, Right, CenterMiddle)

These are trivial wrappers that just set DEFAULT_CSS for alignment.

**Implementation**:
1. Create each struct extending Container base
2. Set DEFAULT_CSS with appropriate align properties
3. No custom properties, methods, or messages needed
4. Write basic tests for alignment behavior

**Estimated Complexity**: Trivial (CSS-only difference)

```rust
// Example: Center
pub struct Center {
    // Inherits from Container
}

impl Center {
    pub const DEFAULT_CSS: &'static str = r#"
        Center {
            align-horizontal: center;
            width: 1fr;
            height: auto;
        }
    "#;
}
```

### Plan: Group Containers (VerticalGroup, HorizontalGroup)

Variants of Vertical/Horizontal that shrink to content.

**Implementation**:
1. Create each struct extending Container base
2. Set DEFAULT_CSS with `height: auto` and `overflow: hidden hidden`
3. Set layout type to vertical/horizontal
4. No custom properties, methods, or messages needed
5. Write tests for shrink-to-content behavior

**Estimated Complexity**: Trivial (CSS-only difference)

```rust
// Example: VerticalGroup
pub struct VerticalGroup {
    // Inherits from Container
}

impl VerticalGroup {
    pub const DEFAULT_CSS: &'static str = r#"
        VerticalGroup {
            width: 1fr;
            height: auto;
            layout: vertical;
            overflow: hidden hidden;
        }
    "#;
}
```

### Plan: ScrollableContainer

Base for all scrollable containers with keyboard navigation.

**Implementation**:
1. Create ScrollableContainer struct with `can_focus = true`
2. Extend or wrap ScrollView
3. Add keyboard bindings for scroll navigation
4. Implement scroll action methods
5. Write tests for keyboard navigation

**Keyboard Bindings**:
```rust
bindings![
    ("up", "scroll_up"),
    ("down", "scroll_down"),
    ("left", "scroll_left"),
    ("right", "scroll_right"),
    ("home", "scroll_home"),
    ("end", "scroll_end"),
    ("pageup", "page_up"),
    ("pagedown", "page_down"),
    ("ctrl+pageup", "page_left"),
    ("ctrl+pagedown", "page_right"),
]
```

**Estimated Complexity**: Medium (keyboard handling)

### Plan: VerticalScroll / HorizontalScroll

Specialized scroll containers with single-axis scrolling.

**Implementation**:
1. Create each struct extending ScrollableContainer
2. Set DEFAULT_CSS with appropriate overflow settings
3. Inherit keyboard bindings from ScrollableContainer
4. Write tests for scroll behavior

**Estimated Complexity**: Trivial (CSS-only difference from ScrollableContainer)

```rust
// Example: VerticalScroll
pub struct VerticalScroll {
    // Inherits from ScrollableContainer
}

impl VerticalScroll {
    pub const DEFAULT_CSS: &'static str = r#"
        VerticalScroll {
            overflow-x: hidden;
            overflow-y: auto;
        }
    "#;
}
```

### Plan: ItemGrid

Advanced grid with reactive column configuration.

**Implementation**:
1. Create ItemGrid struct extending Grid
2. Add reactive properties:
   - `stretch_height: bool`
   - `min_column_width: Option<u16>`
   - `max_column_width: Option<u16>`
   - `regular: bool`
3. Implement `pre_layout()` to apply properties to GridLayout
4. Implement watchers for reactive property changes
5. Write tests for dynamic column behavior

**Estimated Complexity**: Medium (reactive properties + layout integration)

```rust
pub struct ItemGrid {
    stretch_height: bool,
    min_column_width: Option<u16>,
    max_column_width: Option<u16>,
    regular: bool,
    // ... other fields
}

impl ItemGrid {
    pub fn new() -> Self { ... }

    pub fn with_stretch_height(mut self, stretch: bool) -> Self { ... }
    pub fn with_min_column_width(mut self, width: u16) -> Self { ... }
    pub fn with_max_column_width(mut self, width: u16) -> Self { ... }
    pub fn with_regular(mut self, regular: bool) -> Self { ... }

    fn pre_layout(&mut self, layout: &mut GridLayout) {
        // Apply column constraints based on reactive properties
    }
}
```
