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
    width: auto;
    height: auto;
    layout: vertical;
}

HorizontalGroup {
    width: auto;
    height: auto;
    layout: horizontal;
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

5. **VerticalGroup / HorizontalGroup** - Collapsing variants
   - Use `width: auto; height: auto` instead of `1fr`

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

1. Center, Middle, Right (trivial alignment wrappers)
2. VerticalGroup, HorizontalGroup (collapsing variants)
3. ScrollableContainer (keyboard bindings)
4. VerticalScroll, HorizontalScroll (overflow presets)
5. ItemGrid (advanced features)
