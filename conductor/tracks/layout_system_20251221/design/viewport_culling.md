# Design: Viewport Culling Strategy

## Overview

Viewport culling is an optimization that skips rendering widgets that are completely outside the visible area. This is essential for performance when dealing with large scrollable content (e.g., lists with thousands of items).

## Problem Statement

When a ScrollView contains many widgets, rendering all of them every frame is wasteful:
- Off-screen widgets produce no visible output
- Processing them consumes CPU time
- Memory for their render output is allocated unnecessarily

## Coordinate Spaces

**CRITICAL:** Understanding coordinate spaces is essential for correct culling.

### Coordinate Space Definitions

1. **Screen Space**: Absolute coordinates on the terminal (0,0 = top-left)
2. **Parent Space**: Coordinates relative to parent widget's origin
3. **Content Space**: Coordinates within scrollable content (can be negative when scrolled)

### ScrollView Coordinate Mapping

```
Screen Space           Parent Space          Content Space
┌────────────┐        ┌────────────┐        ┌────────────────┐
│ (50,10)    │   →    │ (0,0)      │   →    │ (scroll_x,     │
│   ScrollView        │   ScrollView         │  scroll_y)     │
│            │        │            │        │     = visible  │
└────────────┘        └────────────┘        └────────────────┘
```

When content is placed at `(-scroll_x, -scroll_y)` in parent space:
- Content at `(0, 0)` in content space appears at `(-scroll_x, -scroll_y)` in parent space
- The visible region in **content space** is `(scroll_x, scroll_y, viewport_w, viewport_h)`
- The visible region in **parent space** is `(0, 0, viewport_w, viewport_h)`

## Culling Strategy

### 1. Visible Region Calculation

```rust
/// Calculate the visible region in CONTENT SPACE for a scrollable container.
/// This is where the content coordinates that are currently visible fall.
pub fn visible_content_region(scroll: &ScrollState) -> Rect {
    Rect::new(
        scroll.offset.x,
        scroll.offset.y,
        scroll.viewport_size.width,
        scroll.viewport_size.height,
    )
}

/// Calculate the visible region in PARENT SPACE for a scrollable container.
/// This is always (0, 0) relative to the ScrollView's origin.
pub fn visible_parent_region(scroll: &ScrollState) -> Rect {
    Rect::new(0, 0, scroll.viewport_size.width, scroll.viewport_size.height)
}
```

### 2. Intersection Test

```rust
impl Rect {
    /// Check if this rect intersects with another.
    pub fn intersects(&self, other: &Rect) -> bool {
        self.x < other.x + other.width as i16 &&
        self.x + self.width as i16 > other.x &&
        self.y < other.y + other.height as i16 &&
        self.y + self.height as i16 > other.y
    }

    /// Get the intersection of two rects (if any).
    pub fn intersection(&self, other: &Rect) -> Option<Rect> {
        if !self.intersects(other) {
            return None;
        }

        let x = self.x.max(other.x);
        let y = self.y.max(other.y);
        let right = (self.x + self.width as i16).min(other.x + other.width as i16);
        let bottom = (self.y + self.height as i16).min(other.y + other.height as i16);

        Some(Rect::new(x, y, (right - x) as u16, (bottom - y) as u16))
    }
}
```

### 3. Culling During Render

**IMPORTANT:** Child clip must **intersect** with parent clip, not replace it.
This ensures nested scrollables don't render outside their parent's bounds.

```rust
/// Render tree with culling.
///
/// clip_region is in SCREEN SPACE - the absolute screen coordinates
/// that are valid for rendering.
pub fn render_with_culling(
    root: WidgetId,
    registry: &WidgetRegistry,
    placement_cache: &PlacementCache,
    clip_region: Rect,
    buffer: &mut RenderBuffer,
) {
    // Stack holds (widget_id, clip_in_screen_space, transform_to_screen)
    let mut stack = vec![(root, clip_region, Offset::ZERO)];

    while let Some((widget_id, current_clip, screen_offset)) = stack.pop() {
        let Some(placement) = placement_cache.get_placement(widget_id) else {
            continue;
        };
        let region = placement.region;

        // Transform widget region to screen space
        let screen_region = Rect::new(
            region.x + screen_offset.x,
            region.y + screen_offset.y,
            region.width,
            region.height,
        );

        // Culling: skip if widget is completely outside clip region
        let Some(visible) = screen_region.intersection(&current_clip) else {
            continue;  // Widget is off-screen, skip it entirely
        };

        // Render this widget (clipped to visible portion)
        let widget = registry.get_widget(widget_id);
        widget.render(visible, buffer);

        // Calculate child clip and transform
        // NOTE: PlacementCache provides metadata (fixed flag) for culling decisions
        let is_scrollable = registry.get_scroll_state(widget_id).is_some();

        for child_id in registry.children(widget_id) {
            // Get placement metadata to check if child is fixed
            let child_placement = placement_cache.get_placement(child_id);
            let is_fixed = child_placement.map(|p| p.fixed).unwrap_or(false);

            let (child_clip, child_offset) = if is_scrollable {
                let scroll = registry.get_scroll_state(widget_id).unwrap();

                // Scrollable container: clip to viewport AND intersect with parent clip
                let viewport_in_screen = Rect::new(
                    screen_region.x,
                    screen_region.y,
                    scroll.viewport_size.width,
                    scroll.viewport_size.height,
                );
                // CRITICAL: Intersect with parent clip to prevent rendering outside parent bounds
                let clipped_viewport = viewport_in_screen.intersection(&current_clip)
                    .unwrap_or(Rect::ZERO);

                if is_fixed {
                    // Fixed children (scrollbars) don't scroll - use parent's screen offset
                    // They're positioned relative to the ScrollView, not the content
                    let child_screen_offset = Offset {
                        x: screen_offset.x + region.x,
                        y: screen_offset.y + region.y,
                    };
                    // Fixed children clip to full parent region, not just viewport
                    (current_clip.intersection(&screen_region).unwrap_or(Rect::ZERO), child_screen_offset)
                } else {
                    // Scrolling children: scroll offset is ALREADY in their region coordinates
                    // (content placed at negative offset by ScrollView layout)
                    // So just use the parent's screen position as the offset base
                    let child_screen_offset = Offset {
                        x: screen_offset.x + region.x,
                        y: screen_offset.y + region.y,
                    };
                    (clipped_viewport, child_screen_offset)
                }
            } else {
                // Non-scrollable: children inherit current clip, offset by widget position
                let child_screen_offset = Offset {
                    x: screen_offset.x + region.x,
                    y: screen_offset.y + region.y,
                };
                (visible, child_screen_offset)
            };

            stack.push((child_id, child_clip, child_offset));
        }
    }
}
```

## Dirty Region Tracking

Beyond culling, we can also track which regions have changed and only re-render those.

### Dirty Region Types

```rust
/// A region that needs re-rendering.
#[derive(Debug, Clone)]
pub enum DirtyRegion {
    /// Everything needs re-rendering.
    Full,
    /// Specific rectangles need re-rendering.
    Partial(Vec<Rect>),
    /// Nothing needs re-rendering.
    None,
}

impl DirtyRegion {
    /// Mark a rectangle as dirty.
    pub fn mark_dirty(&mut self, rect: Rect) {
        match self {
            DirtyRegion::Full => {}  // Already fully dirty
            DirtyRegion::Partial(rects) => {
                // Could merge overlapping rects for efficiency
                rects.push(rect);
            }
            DirtyRegion::None => {
                *self = DirtyRegion::Partial(vec![rect]);
            }
        }
    }

    /// Mark everything as dirty.
    pub fn mark_full(&mut self) {
        *self = DirtyRegion::Full;
    }

    /// Clear dirty regions (after rendering).
    pub fn clear(&mut self) {
        *self = DirtyRegion::None;
    }
}
```

### Tracking Dirty Widgets

```rust
/// Track which widgets need re-rendering.
#[derive(Debug, Default)]
pub struct DirtyTracker {
    /// Widgets that have changed.
    dirty_widgets: HashSet<WidgetId>,
    /// Regions that need re-rendering.
    dirty_region: DirtyRegion,
}

impl DirtyTracker {
    /// Mark a widget as needing re-render.
    pub fn mark_widget_dirty(&mut self, id: WidgetId, layout_cache: &LayoutCache) {
        self.dirty_widgets.insert(id);
        if let Some(region) = layout_cache.get(id) {
            self.dirty_region.mark_dirty(region);
        }
    }

    /// Mark all widgets as dirty (e.g., after resize).
    pub fn mark_all_dirty(&mut self) {
        self.dirty_region.mark_full();
    }

    /// Check if a widget needs re-rendering.
    pub fn is_dirty(&self, id: WidgetId) -> bool {
        self.dirty_widgets.contains(&id) || matches!(self.dirty_region, DirtyRegion::Full)
    }

    /// Check if a region overlaps any dirty area.
    pub fn region_is_dirty(&self, region: &Rect) -> bool {
        match &self.dirty_region {
            DirtyRegion::Full => true,
            DirtyRegion::Partial(rects) => rects.iter().any(|r| r.intersects(region)),
            DirtyRegion::None => false,
        }
    }

    /// Clear after rendering.
    pub fn clear(&mut self) {
        self.dirty_widgets.clear();
        self.dirty_region.clear();
    }
}
```

## Optimized Render Loop

```rust
pub fn render_frame(
    root: WidgetId,
    registry: &WidgetRegistry,
    layout_cache: &LayoutCache,
    dirty: &mut DirtyTracker,
    buffer: &mut RenderBuffer,
) {
    // If nothing is dirty, skip rendering entirely
    if matches!(dirty.dirty_region, DirtyRegion::None) {
        return;
    }

    // Calculate visible region
    let viewport = Rect::new(0, 0, buffer.width, buffer.height);

    // Render with culling
    render_tree(root, registry, layout_cache, viewport, dirty, buffer);

    // Clear dirty tracking
    dirty.clear();
}

fn render_tree(
    widget_id: WidgetId,
    registry: &WidgetRegistry,
    layout_cache: &LayoutCache,
    clip: Rect,
    dirty: &DirtyTracker,
    buffer: &mut RenderBuffer,
) {
    let Some(region) = layout_cache.get(widget_id) else {
        return;
    };

    // Culling check 1: off-screen
    let Some(visible) = region.intersection(&clip) else {
        return;
    };

    // Culling check 2: not dirty (only for partial updates)
    if !matches!(dirty.dirty_region, DirtyRegion::Full)
        && !dirty.region_is_dirty(&visible)
    {
        // Widget's visible region hasn't changed, skip
        // But still need to process children in case they're dirty
        for child_id in registry.children(widget_id) {
            render_tree(child_id, registry, layout_cache, visible, dirty, buffer);
        }
        return;
    }

    // Render this widget
    let widget = registry.get_widget(widget_id);
    widget.render(visible, buffer);

    // Render children
    for child_id in registry.children(widget_id) {
        render_tree(child_id, registry, layout_cache, visible, dirty, buffer);
    }
}
```

## Special Cases

### 1. Scrolling Optimization

When only the scroll offset changes (not content), we can use a blit/copy optimization:

```rust
/// Optimize scroll by copying existing buffer content.
pub fn scroll_blit(
    buffer: &mut RenderBuffer,
    dx: i16,
    dy: i16,
    viewport: Rect,
) {
    // Copy existing content by scroll delta
    buffer.copy_region(
        viewport,
        Offset { x: dx, y: dy },
    );

    // Mark newly exposed regions as dirty
    if dy > 0 {
        // Scrolled down, top strip is new
        mark_dirty(Rect::new(viewport.x, viewport.y, viewport.width, dy as u16));
    } else if dy < 0 {
        // Scrolled up, bottom strip is new
        mark_dirty(Rect::new(
            viewport.x,
            viewport.y + viewport.height as i16 + dy,
            viewport.width,
            (-dy) as u16,
        ));
    }
    // Similar for horizontal
}
```

### 2. Widget Virtualization

For very large lists, don't even create widgets for off-screen items:

```rust
/// Virtual list that only creates widgets for visible items.
pub struct VirtualList {
    /// Total number of items.
    item_count: usize,
    /// Height of each item (fixed for simplicity).
    item_height: u16,
    /// Currently materialized widgets.
    visible_widgets: Vec<(usize, WidgetId)>,
}

impl VirtualList {
    /// Update which items are visible based on scroll.
    pub fn update_visible(&mut self, scroll: &ScrollState, registry: &mut WidgetRegistry) {
        let first_visible = (scroll.offset.y as usize) / self.item_height as usize;
        let visible_count = (scroll.viewport_size.height / self.item_height) as usize + 2;
        let last_visible = (first_visible + visible_count).min(self.item_count);

        // Remove widgets for items that scrolled out of view
        self.visible_widgets.retain(|(idx, id)| {
            if *idx < first_visible || *idx >= last_visible {
                registry.remove(*id);
                false
            } else {
                true
            }
        });

        // Create widgets for newly visible items
        for idx in first_visible..last_visible {
            if !self.visible_widgets.iter().any(|(i, _)| *i == idx) {
                let widget_id = self.create_item_widget(idx, registry);
                self.visible_widgets.push((idx, widget_id));
            }
        }
    }
}
```

## Performance Characteristics

| Scenario | Without Culling | With Culling |
|----------|-----------------|--------------|
| 100 widgets, all visible | O(100) | O(100) |
| 1000 widgets, 10 visible | O(1000) | O(10) |
| Scroll by 1 line | Full re-render | Partial (1 line) |
| Widget content change | Full re-render | Widget only |

## Design Decisions

### 1. Intersection-Based Culling
Simple bounding box intersection is fast (O(1) per widget) and handles most cases. More sophisticated spatial data structures (quadtrees, R-trees) are overkill for typical UI hierarchies.

### 2. Conservative Dirty Tracking
We track rectangular regions rather than individual pixels. This is simpler and still provides significant optimization for partial updates.

### 3. Top-Down Traversal with Clip Propagation
Clip regions are propagated down the tree, allowing early termination when an entire subtree is off-screen.

### 4. Optional Virtualization
Full virtualization is only needed for extreme cases (10,000+ items). Basic culling handles most real-world scenarios.

## Integration Points

1. **PlacementCache**: Stores full `WidgetPlacement` for positions and metadata (fixed flag, order)
2. **ScrollState**: Defines the visible region for scrollable containers
3. **RenderBuffer**: Receives only visible content
4. **DirtyTracker**: Tracks what needs re-rendering

### PlacementCache Design

The culling algorithm needs placement metadata (specifically the `fixed` flag) to correctly handle scrollbar children. PlacementCache stores full placements:

```rust
/// Cache storing layout results with full placement metadata.
pub struct PlacementCache {
    /// Widget placements indexed by widget ID.
    placements: HashMap<WidgetId, WidgetPlacement>,
}

impl PlacementCache {
    /// Get a widget's region (backwards compat).
    pub fn get(&self, id: WidgetId) -> Option<Rect> {
        self.placements.get(&id).map(|p| p.region)
    }

    /// Get full placement metadata.
    pub fn get_placement(&self, id: WidgetId) -> Option<&WidgetPlacement> {
        self.placements.get(&id)
    }

    /// Store a placement.
    pub fn insert(&mut self, placement: WidgetPlacement) {
        self.placements.insert(placement.widget_id, placement);
    }
}
```
