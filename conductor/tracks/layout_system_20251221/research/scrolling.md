# Analysis: Scrolling System

## Overview

Python Textual's scrolling system consists of:
1. **ScrollableContainer** - Base class for scrollable containers
2. **ScrollView** - Specialized scrollable for Line API widgets
3. **ScrollBar** - The actual scrollbar widget
4. **ScrollBarCorner** - Corner fill widget when both scrollbars present

## Container Widgets

### Container Hierarchy

```
Widget
├── Container          - layout: vertical, overflow: hidden
├── ScrollableContainer - layout: vertical, overflow: auto (both axes)
│   ├── VerticalScroll   - overflow-y: auto, overflow-x: hidden
│   └── HorizontalScroll - overflow-x: auto, overflow-y: hidden
├── Vertical           - layout: vertical, overflow: hidden
├── VerticalGroup      - height: auto, layout: vertical
├── Horizontal         - layout: horizontal, overflow: hidden
├── HorizontalGroup    - height: auto, layout: horizontal
├── Grid               - layout: grid
└── ItemGrid           - layout: grid, auto columns
```

### ScrollableContainer

The base scrollable container with keyboard bindings:

```python
class ScrollableContainer(Widget, can_focus=True):
    DEFAULT_CSS = """
    ScrollableContainer {
        width: 1fr;
        height: 1fr;
        layout: vertical;
        overflow: auto auto;
    }
    """

    BINDINGS = [
        Binding("up", "scroll_up"),
        Binding("down", "scroll_down"),
        Binding("left", "scroll_left"),
        Binding("right", "scroll_right"),
        Binding("home", "scroll_home"),
        Binding("end", "scroll_end"),
        Binding("pageup", "page_up"),
        Binding("pagedown", "page_down"),
        Binding("ctrl+pageup", "page_left"),
        Binding("ctrl+pagedown", "page_right"),
    ]
```

### Overflow CSS Property

The `overflow` property controls scrollbar visibility:
- `hidden` - No scrolling, clip content
- `scroll` - Always show scrollbar
- `auto` - Show scrollbar only when needed

```css
overflow: auto auto;        /* Both axes auto */
overflow-x: hidden;         /* Horizontal hidden */
overflow-y: auto;           /* Vertical auto */
```

## ScrollView

For Line API widgets (custom rendering that doesn't use child widgets):

```python
class ScrollView(ScrollableContainer):
    DEFAULT_CSS = """
    ScrollView {
        overflow-y: auto;
        overflow-x: auto;
    }
    """

    @property
    def is_scrollable(self) -> bool:
        return True  # Always scrollable

    @property
    def is_container(self) -> bool:
        return False  # No children, uses line API
```

### Virtual Size

ScrollView tracks virtual content size separately from viewport:

```python
def get_content_width(self, container, viewport) -> int:
    return self.virtual_size.width

def get_content_height(self, container, viewport, width) -> int:
    return self.virtual_size.height

def _size_updated(self, size, virtual_size, container_size, layout):
    # Track size changes
    # Update scroll position constraints
    # Refresh scrollbars
```

### Scroll Watchers

Reactive properties trigger updates when scroll position changes:

```python
def watch_scroll_x(self, old_value, new_value):
    if self.show_horizontal_scrollbar:
        self.horizontal_scrollbar.position = new_value
    if round(old_value) != round(new_value):
        self.refresh(self.size.region)

def watch_scroll_y(self, old_value, new_value):
    if self.show_vertical_scrollbar:
        self.vertical_scrollbar.position = new_value
    if round(old_value) != round(new_value):
        self.refresh(self.size.region)
```

### Scroll Methods

```python
def scroll_to(
    self,
    x: float | None = None,
    y: float | None = None,
    animate: bool = True,
    speed: float | None = None,
    duration: float | None = None,
    easing: EasingFunction | str | None = None,
    force: bool = False,
    on_complete: CallbackType | None = None,
    level: AnimationLevel = "basic",
    immediate: bool = False,
) -> None:
    """Scroll to absolute coordinate."""

def refresh_line(self, y: int) -> None:
    """Refresh a single line (for Line API)."""

def refresh_lines(self, y_start: int, line_count: int = 1) -> None:
    """Refresh multiple lines."""
```

## ScrollBar Widget

### ScrollBarRender

Handles the visual rendering of scrollbars with Unicode block characters:

```python
class ScrollBarRender:
    VERTICAL_BARS = ["▁", "▂", "▃", "▄", "▅", "▆", "▇", " "]
    HORIZONTAL_BARS = ["▉", "▊", "▋", "▌", "▍", "▎", "▏", " "]
    BLANK_GLYPH = " "

    def __init__(
        self,
        virtual_size: int = 100,    # Total content size
        window_size: int = 0,       # Visible viewport size
        position: float = 0,        # Current scroll position
        thickness: int = 1,         # Scrollbar width/height
        vertical: bool = True,      # Orientation
        style: StyleType = "bright_magenta on #555555",
    )
```

### Thumb Size Calculation

```python
@classmethod
def render_bar(cls, size, virtual_size, window_size, position, ...):
    if window_size and size and virtual_size and size != virtual_size:
        # Calculate thumb size ratio
        bar_ratio = virtual_size / size
        thumb_size = max(1, window_size / bar_ratio)

        # Calculate thumb position
        position_ratio = position / (virtual_size - window_size)
        position = (size - thumb_size) * position_ratio

        # Use sub-character positioning for smoothness
        start = int(position * len_bars)
        end = start + ceil(thumb_size * len_bars)

        # Apply partial characters for smooth edges
        start_index, start_bar = divmod(max(0, start), len_bars)
        end_index, end_bar = divmod(max(0, end), len_bars)
```

### ScrollBar Widget

```python
class ScrollBar(Widget):
    # Reactive properties
    window_virtual_size: Reactive[int] = Reactive(100)  # Content size
    window_size: Reactive[int] = Reactive(0)            # Viewport size
    position: Reactive[float] = Reactive(0)             # Scroll position
    mouse_over: Reactive[bool] = Reactive(False)        # Hover state
    grabbed: Reactive[Offset | None] = Reactive(None)   # Drag state

    def __init__(
        self,
        vertical: bool = True,
        name: str | None = None,
        thickness: int = 1,
    )
```

### Mouse Interaction

```python
def action_scroll_down(self):
    """Click below thumb."""
    self.post_message(ScrollDown() if self.vertical else ScrollRight())

def action_scroll_up(self):
    """Click above thumb."""
    self.post_message(ScrollUp() if self.vertical else ScrollLeft())

def action_grab(self):
    """Start dragging."""
    self.capture_mouse()

def _on_mouse_capture(self, event):
    self.grabbed = event.mouse_position
    self.grabbed_position = self.position

async def _on_mouse_move(self, event):
    if self.grabbed and self.window_size:
        # Calculate new scroll position from drag delta
        if self.vertical:
            delta = event._screen_y - self.grabbed.y
            y = self.grabbed_position + delta * (self.window_virtual_size / self.window_size)
            self.post_message(ScrollTo(y=y))
        else:
            delta = event._screen_x - self.grabbed.x
            x = self.grabbed_position + delta * (self.window_virtual_size / self.window_size)
            self.post_message(ScrollTo(x=x))
```

### Scroll Messages

```python
class ScrollMessage(Message, bubble=False):
    """Base class for scrollbar messages."""

class ScrollUp(ScrollMessage): ...
class ScrollDown(ScrollMessage): ...
class ScrollLeft(ScrollMessage): ...
class ScrollRight(ScrollMessage): ...

class ScrollTo(ScrollMessage):
    def __init__(self, x=None, y=None, animate=True):
        self.x = x
        self.y = y
        self.animate = animate
```

### ScrollBarCorner

Fills the corner when both scrollbars are visible:

```python
class ScrollBarCorner(Widget):
    def render(self):
        styles = self.parent.styles
        color = styles.scrollbar_corner_color
        return Blank(color)
```

## Styling

### Scrollbar CSS Properties

```css
Widget {
    scrollbar-background: #555555;
    scrollbar-background-hover: #666666;
    scrollbar-background-active: #777777;
    scrollbar-color: bright_magenta;
    scrollbar-color-hover: bright_magenta;
    scrollbar-color-active: bright_magenta;
    scrollbar-corner-color: #555555;
    scrollbar-size: 1 1;  /* vertical horizontal */
    scrollbar-gutter: stable;  /* Reserve space for scrollbar */
}
```

## Rust Implementation Notes

### Core Types

```rust
pub struct ScrollState {
    pub offset: Offset,        // Current scroll position (x, y)
    pub virtual_size: Size,    // Total content size
    pub viewport_size: Size,   // Visible area size
}

impl ScrollState {
    pub fn max_scroll_x(&self) -> i32 {
        (self.virtual_size.width as i32 - self.viewport_size.width as i32).max(0)
    }

    pub fn max_scroll_y(&self) -> i32 {
        (self.virtual_size.height as i32 - self.viewport_size.height as i32).max(0)
    }

    pub fn clamp_offset(&self, offset: Offset) -> Offset {
        Offset {
            x: offset.x.clamp(0, self.max_scroll_x()),
            y: offset.y.clamp(0, self.max_scroll_y()),
        }
    }
}
```

### ScrollBar Rendering

```rust
pub struct ScrollBarRenderer {
    pub virtual_size: u32,
    pub window_size: u32,
    pub position: f32,
    pub thickness: u16,
    pub vertical: bool,
}

impl ScrollBarRenderer {
    const VERTICAL_BARS: [char; 8] = ['▁', '▂', '▃', '▄', '▅', '▆', '▇', ' '];
    const HORIZONTAL_BARS: [char; 8] = ['▉', '▊', '▋', '▌', '▍', '▎', '▏', ' '];

    pub fn render(&self, size: u16) -> Vec<Segment> {
        if self.virtual_size <= self.window_size {
            // No scrollbar needed
            return self.render_empty(size);
        }

        // Calculate thumb size and position
        let bar_ratio = self.virtual_size as f32 / size as f32;
        let thumb_size = (self.window_size as f32 / bar_ratio).max(1.0);
        let max_position = self.virtual_size as f32 - self.window_size as f32;
        let position_ratio = self.position / max_position;
        let thumb_position = (size as f32 - thumb_size) * position_ratio;

        // Render with sub-character precision
        // ...
    }
}
```

### Scroll Actions

```rust
impl Widget for ScrollableContainer {
    fn action_scroll_up(&mut self) {
        let step = 1;  // Or configurable
        self.scroll_offset.y = (self.scroll_offset.y - step).max(0);
    }

    fn action_page_down(&mut self) {
        let page = self.viewport_size.height as i32;
        self.scroll_offset.y = (self.scroll_offset.y + page).min(self.max_scroll_y());
    }

    fn action_scroll_home(&mut self) {
        self.scroll_offset = Offset::ZERO;
    }

    fn action_scroll_end(&mut self) {
        self.scroll_offset.y = self.max_scroll_y();
    }
}
```

### Keyboard Bindings

```rust
const SCROLLABLE_BINDINGS: &[(&str, &str)] = &[
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
];
```
