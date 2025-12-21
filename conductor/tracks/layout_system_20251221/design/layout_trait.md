# Design: Layout Trait

## Overview

The `Layout` trait is the core abstraction for arranging widgets within a container. It defines how child widgets are sized and positioned within the available space.

## Trait Definition

```rust
/// Result of laying out widgets within a container.
#[derive(Debug, Clone)]
pub struct WidgetPlacement {
    /// The widget being placed.
    pub widget_id: WidgetId,
    /// The region (position and size) assigned to the widget.
    pub region: Rect,
    /// Additional offset to apply (for scrolling, absolute positioning).
    pub offset: Offset,
    /// The widget's margin (already accounted for in region).
    pub margin: Spacing,
    /// Rendering order (higher = on top).
    pub order: i32,
    /// Whether this widget is fixed (doesn't scroll with content).
    pub fixed: bool,
}

/// The core layout trait that all layout algorithms implement.
pub trait Layout: Debug + Send + Sync {
    /// Arrange child widgets within the available size.
    ///
    /// # Arguments
    /// * `context` - Layout context with registry access and viewport info
    /// * `parent_id` - The container widget's ID
    /// * `children` - Child widget IDs to arrange
    /// * `size` - Available size for layout
    /// * `greedy` - If true, widgets expand to fill space; if false, fit to content
    ///
    /// # Returns
    /// A vector of widget placements with positions and sizes.
    fn arrange(
        &self,
        context: &LayoutContext,
        parent_id: WidgetId,
        children: &[WidgetId],
        size: Size,
        greedy: bool,
    ) -> Vec<WidgetPlacement>;

    /// Get the content width for this layout (for auto sizing parent containers).
    ///
    /// This is called when the parent has `width: auto` and needs to know
    /// how wide its content would be. Default returns 0 (unknown).
    fn get_content_width(
        &self,
        context: &LayoutContext,
        children: &[WidgetId],
        container: Size,
    ) -> u16 {
        // Default implementation: sum of child content widths (vertical)
        // or max of child content widths (horizontal) - override as needed
        let _ = (context, children, container);
        0
    }

    /// Get the content height for this layout given a width (for auto sizing).
    ///
    /// This is called when the parent has `height: auto` and needs to know
    /// how tall its content would be at a given width.
    fn get_content_height(
        &self,
        context: &LayoutContext,
        children: &[WidgetId],
        container: Size,
        width: u16,
    ) -> u16 {
        // Default implementation: sum of child content heights (vertical)
        // Override for horizontal layouts, grids, etc.
        let _ = (context, children, container, width);
        0
    }

    /// Get the name of this layout (for debugging).
    fn name(&self) -> &'static str;
}
```

## Supporting Types

```rust
/// Context provided to layout algorithms.
pub struct LayoutContext<'a> {
    /// Access to widget registry for querying styles and content sizes.
    pub registry: &'a WidgetRegistry,
    /// The terminal/viewport size (for vw/vh units).
    pub viewport: Size,
}

impl<'a> LayoutContext<'a> {
    /// Get a widget's styles.
    pub fn styles(&self, id: WidgetId) -> &Styles {
        self.registry.get_styles(id)
    }

    /// Get a widget's content width (for auto sizing).
    pub fn content_width(&self, id: WidgetId, container: Size) -> u16 {
        self.registry.get_content_width(id, container, self.viewport)
    }

    /// Get a widget's content height given a width (for auto sizing).
    pub fn content_height(&self, id: WidgetId, container: Size, width: u16) -> u16 {
        self.registry.get_content_height(id, container, self.viewport, width)
    }

    /// Get a widget's box model (width, height, margin).
    pub fn box_model(
        &self,
        id: WidgetId,
        container: Size,
        fraction_width: Option<f32>,
        fraction_height: Option<f32>,
        greedy: bool,
    ) -> BoxModel {
        self.registry.get_box_model(id, container, self.viewport, fraction_width, fraction_height, greedy)
    }
}

/// Box model result from size calculations.
#[derive(Debug, Clone, Copy)]
pub struct BoxModel {
    /// Computed width (not including margin).
    pub width: u16,
    /// Computed height (not including margin).
    pub height: u16,
    /// Margin around the widget.
    pub margin: Spacing,
}

/// Spacing for margins, padding, borders.
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq)]
pub struct Spacing {
    pub top: u16,
    pub right: u16,
    pub bottom: u16,
    pub left: u16,
}

impl Spacing {
    pub const ZERO: Self = Self { top: 0, right: 0, bottom: 0, left: 0 };

    pub fn new(top: u16, right: u16, bottom: u16, left: u16) -> Self {
        Self { top, right, bottom, left }
    }

    /// Total horizontal space (left + right).
    pub fn width(&self) -> u16 {
        self.left + self.right
    }

    /// Total vertical space (top + bottom).
    pub fn height(&self) -> u16 {
        self.top + self.bottom
    }
}
```

## Layout Result Processing

After layout returns placements, the parent widget may need to process them:

```rust
impl WidgetPlacement {
    /// Get the bounding region of all placements.
    pub fn get_bounds(placements: &[WidgetPlacement]) -> Rect {
        if placements.is_empty() {
            return Rect::ZERO;
        }

        let mut min_x = i16::MAX;
        let mut min_y = i16::MAX;
        let mut max_x = i16::MIN;
        let mut max_y = i16::MIN;

        for p in placements {
            min_x = min_x.min(p.region.x);
            min_y = min_y.min(p.region.y);
            max_x = max_x.max(p.region.x + p.region.width as i16);
            max_y = max_y.max(p.region.y + p.region.height as i16);
        }

        Rect::new(min_x, min_y, (max_x - min_x) as u16, (max_y - min_y) as u16)
    }

    /// Translate all placements by an offset.
    pub fn translate(placements: &mut [WidgetPlacement], offset: Offset) {
        for p in placements {
            p.region.x += offset.x;
            p.region.y += offset.y;
        }
    }

    /// Apply absolute positioning to placements that have it set.
    pub fn apply_absolute(placements: &mut [WidgetPlacement], context: &LayoutContext) {
        for p in placements {
            let styles = context.styles(p.widget_id);
            if let Some(abs_offset) = styles.offset {
                let resolved = abs_offset.resolve(
                    Size::new(p.region.width, p.region.height),
                    context.viewport,
                );
                p.region.x += resolved.x;
                p.region.y += resolved.y;
            }
        }
    }
}
```

## Layout Selection

Widgets select their layout based on their `layout` style property:

```rust
/// Available layout algorithms.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum LayoutType {
    #[default]
    Vertical,
    Horizontal,
    Grid,
}

impl LayoutType {
    /// Get the layout implementation for this type.
    pub fn get_layout(&self) -> Box<dyn Layout> {
        match self {
            LayoutType::Vertical => Box::new(VerticalLayout),
            LayoutType::Horizontal => Box::new(HorizontalLayout),
            LayoutType::Grid => Box::new(GridLayout::default()),
        }
    }
}
```

## Integration with Arrange Algorithm

The Layout trait integrates into the broader arrange algorithm:

```
arrange(parent, children, size, viewport)
├── Filter visible widgets (display: true)
├── Build layers (group by layer property)
└── For each layer:
    ├── Handle split widgets (physically divide region)
    ├── Handle docked widgets (reduce layout area)
    └── Call layout.arrange() for remaining widgets
        ├── Apply alignment (horizontal, vertical)
        ├── Translate placements
        └── Apply absolute positioning
```

## Absolute and Overlay Positioning

Widgets with absolute positioning or overlay behavior are excluded from normal layout flow:

```rust
/// Position type for widgets.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum Position {
    /// Normal flow - participates in parent layout.
    #[default]
    Relative,
    /// Absolute positioning relative to parent.
    /// Removed from normal flow; other widgets don't see it.
    Absolute,
}

/// Check if a widget should participate in layout flow.
pub fn in_layout_flow(context: &LayoutContext, id: WidgetId) -> bool {
    let styles = context.styles(id);
    // Absolute-positioned and overlay widgets don't participate in flow
    styles.position != Position::Absolute && !styles.overlay
}
```

### Filtering During Arrange

The arrange algorithm filters out non-flow widgets before calling Layout:

```rust
// In arrange():
let flow_children: Vec<WidgetId> = children
    .iter()
    .filter(|&id| in_layout_flow(context, *id))
    .copied()
    .collect();

// Layout only operates on flow children
let placements = layout.arrange(context, parent_id, &flow_children, size, greedy);

// Absolute/overlay widgets are handled separately
let absolute_children: Vec<WidgetId> = children
    .iter()
    .filter(|&id| !in_layout_flow(context, *id))
    .copied()
    .collect();

for id in absolute_children {
    let styles = context.styles(id);
    let box_model = context.box_model(id, size, None, None, false);

    // Position relative to parent using offset styles (top, left, etc.)
    let x = styles.left.unwrap_or(0);
    let y = styles.top.unwrap_or(0);

    placements.push(WidgetPlacement {
        widget_id: id,
        region: Rect::new(x, y, box_model.width, box_model.height),
        offset: Offset::ZERO,
        margin: box_model.margin,
        order: if styles.overlay { i32::MAX - 1 } else { 0 },
        fixed: styles.overlay,  // Overlay widgets don't scroll
    });
}
```

## Design Decisions

### 1. WidgetId vs Widget Reference

We use `WidgetId` instead of widget references because:
- Avoids lifetime complexity
- Allows layouts to be cached and reused
- Registry provides all necessary data through LayoutContext

### 2. Greedy Parameter

The `greedy` parameter controls whether widgets:
- `true` (default): Expand to fill available space
- `false`: Size to fit content (used for "optimal" sizing)

This affects how fractional units and auto sizing are resolved.

### 3. Separate Offset Field

The `offset` field in WidgetPlacement is separate from the region for:
- Scroll offset application
- Absolute positioning
- Layer-specific translations

### 4. Order for Z-Index

The `order` field determines rendering order:
- Higher values render on top
- Docked widgets use `i32::MAX` (TOP_Z)
- Regular widgets use their natural order

## Next Steps

1. Design constraint solving for flex (fr) distribution
2. Implement VerticalLayout
3. Implement HorizontalLayout
4. Implement GridLayout
