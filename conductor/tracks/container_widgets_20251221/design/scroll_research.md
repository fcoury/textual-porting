# ScrollableContainer and Scroll Methods Research

## Python Textual ScrollBar Implementation

### Message Types
All inherit from `ScrollMessage` with `bubble=False`:

| Message | Purpose |
|---------|---------|
| `ScrollUp` | Posted when clicking above handle in vertical scrollbars |
| `ScrollDown` | Posted when clicking below handle in vertical scrollbars |
| `ScrollLeft` | Posted when clicking left of handle in horizontal scrollbars |
| `ScrollRight` | Posted when clicking right of handle in horizontal scrollbars |
| `ScrollTo` | Sent during click-and-drag with optional x/y coordinates and animate flag |

### ScrollBar Properties
- `window_virtual_size`: Total scrollable content size (default: 100)
- `window_size`: Visible viewport size (default: 0)
- `position`: Current scroll position (default: 0.0)
- `mouse_over`: Tracks hover state
- `grabbed`: Stores mouse position when actively dragging

### Position Calculation
```
new_position = initial_position + (pixel_offset × virtual_size / window_size)
thumb_size = max(1, window_size / (virtual_size / size))
```

## Python Textual Widget Scroll Methods

### Primary Scroll Methods
- `scroll_to(x, y, animate, speed, duration, easing, force, on_complete)` - Scroll to absolute coordinates
- `scroll_relative(x, y, animate)` - Scroll relative to current position
- `set_scroll(x, y)` - Immediate scroll without validation

### Scroll Position Properties
- `scroll_x`, `scroll_y`: Reactive floats tracking position
- `scroll_target_x`, `scroll_target_y`: Animation destination
- `scroll_offset`: Computed Offset(round(scroll_x), round(scroll_y))
- `max_scroll_x`, `max_scroll_y`: Maximum allowed scroll values
- `is_vertical_scroll_end`, `is_horizontal_scroll_end`: Boundary checks

## Existing textual-rs Infrastructure

### ScrollState (Complete)
```rust
pub struct ScrollState {
    pub offset: Offset,
    pub virtual_size: Size,
    pub viewport_size: Size,
}
```

Methods implemented:
- `max_scroll_x()`, `max_scroll_y()` - Maximum scroll values
- `clamp_offset()` - Constrain offset to valid range
- `can_scroll_x()`, `can_scroll_y()` - Check if scrolling possible
- `scroll_percent_x()`, `scroll_percent_y()` - Position as percentage
- `thumb_size_x()`, `thumb_size_y()` - Scrollbar thumb sizes
- `scroll_to(x, y)` - Absolute positioning
- `scroll_by(dx, dy)` - Relative scrolling
- `scroll_up/down/left/right()` - Single line/column
- `page_up/down/left/right()` - Page-based scrolling
- `scroll_home()`, `scroll_end()` - Jump to edges
- `scroll_visible(region)` - Ensure region is visible
- `set_virtual_size()`, `set_viewport_size()` - Size management

### ScrollBar Widget (Complete)
```rust
pub struct ScrollBar {
    vertical: bool,
    position: f32,
    thumb_size: f32,
    track_style: Style,
    thumb_style: Style,
}
```

Methods:
- `vertical()`, `horizontal()` - Constructors
- `update_from_scroll(&ScrollState)` - Sync with scroll state
- `with_track_style()`, `with_thumb_style()` - Styling
- `is_vertical()`, `position()`, `thumb_size()` - Getters

### ScrollView Widget (Complete)
```rust
pub struct ScrollView {
    id: Option<String>,
    classes: Vec<String>,
    scroll: ScrollState,
    overflow: OverflowSettings,
    border: bool,
    border_style: Style,
    layout_hints: LayoutHints,
}
```

Methods:
- `new()`, `vertical()`, `horizontal()` - Constructors
- `with_id()`, `with_class()` - Identification
- `with_overflow()` - Scrollbar visibility
- `with_border()`, `with_border_style()` - Styling
- `with_width()`, `with_height()`, `with_dock()` - Layout
- `scroll()`, `scroll_mut()` - Access scroll state
- `scroll_to()`, `scroll_by()`, `scroll_visible()`, `scroll_home()`, `scroll_end()` - Scrolling
- `show_vertical_scrollbar()`, `show_horizontal_scrollbar()` - Visibility checks

### Test Coverage
60+ comprehensive tests covering:
- Overflow settings
- ScrollState calculations
- ScrollBar rendering
- ScrollView operations

## Gap Analysis

### Already Have
1. Complete ScrollState with all methods
2. Complete ScrollBar with styling
3. Complete ScrollView with overflow settings
4. 60+ tests

### Need to Add for ScrollableContainer

1. **Keyboard Bindings** - ScrollableContainer should have:
   ```
   up -> scroll_up
   down -> scroll_down
   left -> scroll_left
   right -> scroll_right
   home -> scroll_home
   end -> scroll_end
   pageup -> page_up
   pagedown -> page_down
   ctrl+pageup -> page_left
   ctrl+pagedown -> page_right
   ```

2. **Focusability** - ScrollableContainer has `can_focus=True`
   - Current ScrollView is NOT focusable
   - Need to make ScrollableContainer focusable

3. **Mouse Interaction Messages** (Optional for now):
   - ScrollUp, ScrollDown, ScrollLeft, ScrollRight, ScrollTo
   - These are for scrollbar drag interaction

## Implementation Plan

1. **VerticalScroll** - Thin wrapper around ScrollView::vertical()
   - Add keyboard bindings for vertical scrolling
   - Make focusable

2. **HorizontalScroll** - Thin wrapper around ScrollView::horizontal()
   - Add keyboard bindings for horizontal scrolling
   - Make focusable

3. **ScrollableContainer** - Base with all keyboard bindings
   - Full keyboard navigation
   - Can be extended by specialized containers

4. **Scroll-to-widget** - Method to scroll widget into view
   - Calculate widget's position in virtual space
   - Use scroll_visible() with widget's region
