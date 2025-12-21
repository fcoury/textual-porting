# Design: ScrollView Architecture

## Overview

ScrollView is a container widget that enables scrolling when its content exceeds the visible viewport. It manages virtual content size, scroll offsets, and scrollbar widgets.

## Core Concepts

### Virtual Size vs Viewport Size

```
┌─────────────────────────────────────┐
│          Virtual Content            │
│  ┌─────────────────────┐            │
│  │     Viewport        │            │
│  │  (visible area)     │ ◄─ scroll_y
│  │                     │            │
│  └─────────────────────┘            │
│         ▲                           │
│         └── scroll_x                │
│                                     │
│                                     │
└─────────────────────────────────────┘
```

- **Virtual Size**: Total size of all content (may be larger than viewport)
- **Viewport Size**: Visible area (the ScrollView's actual size minus scrollbars)
- **Scroll Offset**: Current scroll position (x, y)

## Data Structures

```rust
/// Scroll state for a scrollable container.
#[derive(Debug, Clone, Default)]
pub struct ScrollState {
    /// Current scroll position.
    pub offset: Offset,
    /// Total size of virtual content.
    pub virtual_size: Size,
    /// Size of the visible viewport.
    pub viewport_size: Size,
}

impl ScrollState {
    /// Maximum scroll offset in X direction.
    pub fn max_scroll_x(&self) -> i16 {
        (self.virtual_size.width as i16 - self.viewport_size.width as i16).max(0)
    }

    /// Maximum scroll offset in Y direction.
    pub fn max_scroll_y(&self) -> i16 {
        (self.virtual_size.height as i16 - self.viewport_size.height as i16).max(0)
    }

    /// Clamp offset to valid scroll range.
    pub fn clamp_offset(&self, offset: Offset) -> Offset {
        Offset {
            x: offset.x.clamp(0, self.max_scroll_x()),
            y: offset.y.clamp(0, self.max_scroll_y()),
        }
    }

    /// Whether horizontal scrolling is possible.
    pub fn can_scroll_x(&self) -> bool {
        self.virtual_size.width > self.viewport_size.width
    }

    /// Whether vertical scrolling is possible.
    pub fn can_scroll_y(&self) -> bool {
        self.virtual_size.height > self.viewport_size.height
    }
}
```

## ScrollView Widget

```rust
/// A container that enables scrolling for content larger than its viewport.
pub struct ScrollView {
    /// Unique widget ID.
    id: WidgetId,
    /// Child widget (the content to scroll).
    content: WidgetId,
    /// Current scroll state.
    scroll: ScrollState,
    /// Vertical scrollbar widget (if visible).
    v_scrollbar: Option<WidgetId>,
    /// Horizontal scrollbar widget (if visible).
    h_scrollbar: Option<WidgetId>,
    /// Scrollbar corner widget (when both scrollbars visible).
    corner: Option<WidgetId>,
}
```

### Overflow Property

The `overflow` CSS property controls scrollbar behavior:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum Overflow {
    /// Clip content, no scrolling.
    Hidden,
    /// Always show scrollbar.
    Scroll,
    /// Show scrollbar only when needed.
    #[default]
    Auto,
}

/// Overflow settings for both axes.
#[derive(Debug, Clone, Copy, Default)]
pub struct OverflowSettings {
    pub x: Overflow,
    pub y: Overflow,
}
```

## Layout Integration

ScrollView participates in layout in two ways:

### 1. As a Container (receiving layout)

```rust
impl ScrollView {
    fn arrange_content(
        &mut self,
        context: &LayoutContext,
        available_size: Size,
    ) -> Vec<WidgetPlacement> {
        let styles = context.styles(self.id);
        let overflow = styles.overflow;

        // Determine scrollbar visibility
        let show_v = self.should_show_vscrollbar(overflow.y, available_size);
        let show_h = self.should_show_hscrollbar(overflow.x, available_size);

        // Calculate viewport size (excluding scrollbars)
        let scrollbar_width = styles.scrollbar_size.0;
        let scrollbar_height = styles.scrollbar_size.1;

        let viewport_width = if show_v {
            available_size.width.saturating_sub(scrollbar_width)
        } else {
            available_size.width
        };
        let viewport_height = if show_h {
            available_size.height.saturating_sub(scrollbar_height)
        } else {
            available_size.height
        };

        let viewport_size = Size::new(viewport_width, viewport_height);

        // Get content's natural size
        let content_width = context.content_width(self.content, viewport_size);
        let content_height = context.content_height(self.content, viewport_size, content_width);

        // Update scroll state
        self.scroll.viewport_size = viewport_size;
        self.scroll.virtual_size = Size::new(content_width, content_height);
        self.scroll.offset = self.scroll.clamp_offset(self.scroll.offset);

        // Build placements
        let mut placements = Vec::new();

        // Content placement (offset by scroll position)
        placements.push(WidgetPlacement {
            widget_id: self.content,
            region: Rect::new(
                -self.scroll.offset.x,
                -self.scroll.offset.y,
                content_width,
                content_height,
            ),
            offset: Offset::ZERO,
            margin: Spacing::ZERO,
            order: 0,
            fixed: false,
        });

        // Vertical scrollbar (fixed, doesn't scroll)
        if show_v {
            placements.push(WidgetPlacement {
                widget_id: self.v_scrollbar.unwrap(),
                region: Rect::new(
                    viewport_width as i16,
                    0,
                    scrollbar_width,
                    if show_h { viewport_height } else { available_size.height },
                ),
                offset: Offset::ZERO,
                margin: Spacing::ZERO,
                order: 100,
                fixed: true,
            });
        }

        // Horizontal scrollbar
        if show_h {
            placements.push(WidgetPlacement {
                widget_id: self.h_scrollbar.unwrap(),
                region: Rect::new(
                    0,
                    viewport_height as i16,
                    if show_v { viewport_width } else { available_size.width },
                    scrollbar_height,
                ),
                offset: Offset::ZERO,
                margin: Spacing::ZERO,
                order: 100,
                fixed: true,
            });
        }

        // Corner (when both scrollbars visible)
        if show_v && show_h {
            placements.push(WidgetPlacement {
                widget_id: self.corner.unwrap(),
                region: Rect::new(
                    viewport_width as i16,
                    viewport_height as i16,
                    scrollbar_width,
                    scrollbar_height,
                ),
                offset: Offset::ZERO,
                margin: Spacing::ZERO,
                order: 100,
                fixed: true,
            });
        }

        placements
    }
}
```

### 2. Scrollbar Visibility Logic

```rust
impl ScrollView {
    fn should_show_vscrollbar(&self, overflow: Overflow, size: Size) -> bool {
        match overflow {
            Overflow::Hidden => false,
            Overflow::Scroll => true,
            Overflow::Auto => self.scroll.virtual_size.height > size.height,
        }
    }

    fn should_show_hscrollbar(&self, overflow: Overflow, size: Size) -> bool {
        match overflow {
            Overflow::Hidden => false,
            Overflow::Scroll => true,
            Overflow::Auto => self.scroll.virtual_size.width > size.width,
        }
    }
}
```

## Scroll Operations

```rust
impl ScrollView {
    /// Scroll to absolute position.
    pub fn scroll_to(&mut self, x: Option<i16>, y: Option<i16>) {
        if let Some(x) = x {
            self.scroll.offset.x = x;
        }
        if let Some(y) = y {
            self.scroll.offset.y = y;
        }
        self.scroll.offset = self.scroll.clamp_offset(self.scroll.offset);
    }

    /// Scroll by relative amount.
    pub fn scroll_by(&mut self, dx: i16, dy: i16) {
        self.scroll.offset.x += dx;
        self.scroll.offset.y += dy;
        self.scroll.offset = self.scroll.clamp_offset(self.scroll.offset);
    }

    /// Scroll to make a widget visible.
    pub fn scroll_visible(&mut self, widget_region: Rect) {
        // Calculate target offset to make widget visible
        let mut target = self.scroll.offset;

        // Horizontal visibility
        if widget_region.x < self.scroll.offset.x {
            target.x = widget_region.x;
        } else if widget_region.right() > self.scroll.offset.x + self.scroll.viewport_size.width as i16 {
            target.x = widget_region.right() - self.scroll.viewport_size.width as i16;
        }

        // Vertical visibility
        if widget_region.y < self.scroll.offset.y {
            target.y = widget_region.y;
        } else if widget_region.bottom() > self.scroll.offset.y + self.scroll.viewport_size.height as i16 {
            target.y = widget_region.bottom() - self.scroll.viewport_size.height as i16;
        }

        self.scroll.offset = self.scroll.clamp_offset(target);
    }

    /// Scroll to home (top-left).
    pub fn scroll_home(&mut self) {
        self.scroll.offset = Offset::ZERO;
    }

    /// Scroll to end (bottom-right).
    pub fn scroll_end(&mut self) {
        self.scroll.offset = Offset {
            x: self.scroll.max_scroll_x(),
            y: self.scroll.max_scroll_y(),
        };
    }

    /// Scroll up by one line.
    pub fn scroll_up(&mut self) {
        self.scroll_by(0, -1);
    }

    /// Scroll down by one line.
    pub fn scroll_down(&mut self) {
        self.scroll_by(0, 1);
    }

    /// Scroll up by one page.
    pub fn page_up(&mut self) {
        let page = self.scroll.viewport_size.height as i16;
        self.scroll_by(0, -page);
    }

    /// Scroll down by one page.
    pub fn page_down(&mut self) {
        let page = self.scroll.viewport_size.height as i16;
        self.scroll_by(0, page);
    }
}
```

## ScrollBar Widget

```rust
/// A scrollbar widget.
pub struct ScrollBar {
    id: WidgetId,
    /// Orientation.
    vertical: bool,
    /// Current scroll position (0.0 to 1.0).
    position: f32,
    /// Size of the thumb relative to track (0.0 to 1.0).
    thumb_size: f32,
    /// Whether mouse is over the scrollbar.
    hovered: bool,
    /// Drag state.
    grabbed: Option<GrabState>,
}

struct GrabState {
    /// Mouse position when grab started.
    start_pos: Offset,
    /// Scroll position when grab started.
    start_scroll: f32,
}

impl ScrollBar {
    /// Update scrollbar from scroll state.
    pub fn update_from_scroll(&mut self, scroll: &ScrollState) {
        if self.vertical {
            let max = scroll.max_scroll_y() as f32;
            self.position = if max > 0.0 {
                scroll.offset.y as f32 / max
            } else {
                0.0
            };
            self.thumb_size = if scroll.virtual_size.height > 0 {
                (scroll.viewport_size.height as f32 / scroll.virtual_size.height as f32)
                    .clamp(0.1, 1.0)
            } else {
                1.0
            };
        } else {
            // Similar for horizontal
        }
    }

    /// Render the scrollbar.
    pub fn render(&self, size: Size) -> Vec<Segment> {
        ScrollBarRenderer {
            vertical: self.vertical,
            position: self.position,
            thumb_size: self.thumb_size,
            hovered: self.hovered,
            size,
        }.render()
    }
}
```

### ScrollBar Rendering

Uses Unicode block characters for sub-character precision:

```rust
struct ScrollBarRenderer {
    vertical: bool,
    position: f32,
    thumb_size: f32,
    hovered: bool,
    size: Size,
}

impl ScrollBarRenderer {
    const VERTICAL_BLOCKS: [char; 8] = ['▁', '▂', '▃', '▄', '▅', '▆', '▇', '█'];
    const HORIZONTAL_BLOCKS: [char; 8] = ['▏', '▎', '▍', '▌', '▋', '▊', '▉', '█'];

    fn render(&self) -> Vec<Segment> {
        let track_size = if self.vertical { self.size.height } else { self.size.width } as f32;
        let thumb_cells = (self.thumb_size * track_size).max(1.0);
        let thumb_start = self.position * (track_size - thumb_cells);

        // Render track with thumb at position
        // Use block characters for smooth edges
        // ...
    }
}
```

## Keyboard Bindings

```rust
const SCROLL_BINDINGS: &[(&str, &str)] = &[
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

## Message Flow

```
User Input (key/mouse)
       │
       ▼
   ScrollView
       │
   ┌───┴───┐
   │       │
   ▼       ▼
ScrollBar  Content
   │
   ▼
ScrollTo/ScrollBy message
       │
       ▼
   Update ScrollState
       │
       ▼
   Trigger Re-render
```

## Design Decisions

### 1. Scroll Offset as Negative Content Position
Content is positioned at negative scroll offset rather than translating the viewport. This matches Python Textual and simplifies hit-testing.

### 2. Fixed Scrollbars
Scrollbars have `fixed: true` in their placement so they don't move with the scrolling content.

### 3. Scrollbar as Separate Widget
Scrollbars are separate widgets rather than drawn by ScrollView, enabling:
- Independent styling
- Mouse interaction handling
- Reuse in other contexts

### 4. Lazy Scrollbar Creation
Scrollbar widgets are created only when needed (overflow: auto and content exceeds viewport), reducing widget count for non-scrolling containers.
