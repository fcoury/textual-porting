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

## Dirty Tracking

There are two separate concerns for "dirty" tracking:

1. **Widget Dirty Tracking** - Which widgets need to re-render their content (widget IDs)
2. **Terminal Damage Tracking** - Which screen regions changed and need to be sent to terminal (screen-space rects)

These are handled separately because they operate in different coordinate spaces.

### Terminal Damage Tracking (Optional Optimization)

For terminal output optimization, track which screen regions changed:

```rust
/// Tracks damaged screen regions for terminal output optimization.
/// All rects are in SCREEN SPACE (absolute terminal coordinates).
#[derive(Debug, Clone, Default)]
pub struct DamageTracker {
    /// Damaged screen regions (collected during render).
    damaged_rects: Vec<Rect>,
    /// Whether full screen refresh is needed.
    full_damage: bool,
}

impl DamageTracker {
    /// Record damage at a screen-space rect (called during render).
    pub fn record_damage(&mut self, screen_rect: Rect) {
        if !self.full_damage {
            self.damaged_rects.push(screen_rect);
        }
    }

    /// Mark full screen as damaged.
    pub fn mark_full(&mut self) {
        self.full_damage = true;
    }

    /// Get damaged regions for terminal output.
    pub fn get_damage(&self) -> impl Iterator<Item = &Rect> {
        self.damaged_rects.iter()
    }

    /// Clear after terminal output.
    pub fn clear(&mut self) {
        self.damaged_rects.clear();
        self.full_damage = false;
    }
}
```

**Usage:** During `render_with_dirty_tracking`, after calling `widget.render()`, record `damage.record_damage(visible)` where `visible` is already in screen space.

### Widget Dirty Tracking

**IMPORTANT:** Dirty tracking uses widget IDs, not regions. Region-based tracking has coordinate space issues (placement.region is in parent space, but culling uses screen space). Widget ID tracking is simple and correct.

```rust
/// Track which widgets need re-rendering.
#[derive(Debug, Default)]
pub struct DirtyTracker {
    /// Widgets that have changed (primary mechanism).
    dirty_widgets: HashSet<WidgetId>,
    /// Whether everything needs re-rendering.
    full_redraw: bool,
}

impl DirtyTracker {
    /// Mark a widget as needing re-render.
    pub fn mark_widget_dirty(&mut self, id: WidgetId) {
        self.dirty_widgets.insert(id);
    }

    /// Mark multiple widgets as dirty (e.g., after scroll reveals new widgets).
    pub fn mark_widgets_dirty(&mut self, ids: impl IntoIterator<Item = WidgetId>) {
        self.dirty_widgets.extend(ids);
    }

    /// Mark all widgets as dirty (e.g., after resize).
    /// IMPORTANT: This populates dirty_widgets with all widget IDs so that
    /// culled widgets remain dirty after clear_full_redraw().
    pub fn mark_all_dirty(&mut self, all_widgets: impl IntoIterator<Item = WidgetId>) {
        self.full_redraw = true;
        self.dirty_widgets.extend(all_widgets);
    }

    /// Check if a widget needs re-rendering.
    pub fn is_dirty(&self, id: WidgetId) -> bool {
        self.full_redraw || self.dirty_widgets.contains(&id)
    }

    /// Check if anything is dirty.
    pub fn has_dirty(&self) -> bool {
        self.full_redraw || !self.dirty_widgets.is_empty()
    }

    /// Clear dirty flag for a specific widget (called after rendering it).
    /// IMPORTANT: Only clear widgets that were actually rendered, not culled ones.
    /// Culled widgets remain dirty so they render when scrolled into view.
    pub fn mark_rendered(&mut self, id: WidgetId) {
        self.dirty_widgets.remove(&id);
    }

    /// Clear full_redraw flag after a complete frame.
    /// Individual dirty_widgets are preserved for culled widgets.
    pub fn clear_full_redraw(&mut self) {
        self.full_redraw = false;
    }
}
```

**Critical:** When `mark_all_dirty()` is called, ALL widget IDs must be passed so they're added to `dirty_widgets`. This ensures culled widgets remain dirty after `clear_full_redraw()`.

**Note:** For terminal damage tracking (which screen regions changed), compute screen-space bounds during render traversal and collect into a separate damage list. This is orthogonal to widget dirty tracking.

## Optimized Render Loop

Dirty tracking integrates with `render_with_culling` by adding an early-exit check. The full screen-space transform and scroll/fixed handling from the main culling algorithm is preserved:

```rust
pub fn render_frame(
    root: WidgetId,
    registry: &WidgetRegistry,
    placement_cache: &PlacementCache,
    dirty: &mut DirtyTracker,
    damage: &mut DamageTracker,
    buffer: &mut RenderBuffer,
) {
    // If nothing is dirty, skip rendering entirely
    if !dirty.has_dirty() {
        return;
    }

    // Calculate visible region (screen space)
    let viewport = Rect::new(0, 0, buffer.width, buffer.height);

    // Render with culling + dirty tracking
    render_with_dirty_tracking(root, registry, placement_cache, viewport, dirty, damage, buffer);

    // Clear full_redraw flag (individual widget flags cleared during render)
    dirty.clear_full_redraw();
}

/// Render with both culling and dirty region optimization.
/// This is render_with_culling extended with dirty checks.
pub fn render_with_dirty_tracking(
    root: WidgetId,
    registry: &WidgetRegistry,
    placement_cache: &PlacementCache,
    clip_region: Rect,
    dirty: &mut DirtyTracker,
    damage: &mut DamageTracker,
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

        // Culling check 1: off-screen
        // NOTE: Culled widgets stay dirty so they render when scrolled into view
        let Some(visible) = screen_region.intersection(&current_clip) else {
            continue;
        };

        // Culling check 2: not dirty (only for partial updates)
        // Skip rendering but still process children in case they're dirty
        // Uses widget ID, not region, to avoid coordinate space issues
        if dirty.is_dirty(widget_id) {
            let widget = registry.get_widget(widget_id);
            widget.render(visible, buffer);

            // Mark widget as rendered (clears its dirty flag)
            dirty.mark_rendered(widget_id);

            // Record damage in screen space for terminal output optimization
            damage.record_damage(visible);
        }

        // Calculate child clip and transform (same as render_with_culling)
        let is_scrollable = registry.get_scroll_state(widget_id).is_some();

        for child_id in registry.children(widget_id) {
            let child_placement = placement_cache.get_placement(child_id);
            let is_fixed = child_placement.map(|p| p.fixed).unwrap_or(false);

            let (child_clip, child_offset) = if is_scrollable {
                let scroll = registry.get_scroll_state(widget_id).unwrap();
                let viewport_in_screen = Rect::new(
                    screen_region.x,
                    screen_region.y,
                    scroll.viewport_size.width,
                    scroll.viewport_size.height,
                );
                let clipped_viewport = viewport_in_screen.intersection(&current_clip)
                    .unwrap_or(Rect::ZERO);

                if is_fixed {
                    let child_screen_offset = Offset {
                        x: screen_offset.x + region.x,
                        y: screen_offset.y + region.y,
                    };
                    (current_clip.intersection(&screen_region).unwrap_or(Rect::ZERO), child_screen_offset)
                } else {
                    let child_screen_offset = Offset {
                        x: screen_offset.x + region.x,
                        y: screen_offset.y + region.y,
                    };
                    (clipped_viewport, child_screen_offset)
                }
            } else {
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

**Key differences from basic culling:**
- Adds `dirty` parameter to check if widget needs re-rendering (by widget ID)
- Skips `widget.render()` for clean widgets, but still processes children
- Maintains full screen-space transform and scroll/fixed handling
- Uses widget IDs (not regions) to avoid coordinate space mismatches
- Clears dirty flags only for rendered widgets (culled widgets stay dirty)
- Records screen-space damage for terminal output optimization

## Special Cases

### 1. Scroll Handling

Scroll requires two operations:
1. **Mark newly visible widgets dirty** - so they render with widget-based tracking
2. **Record terminal damage** - so the terminal output includes the changed region

```rust
/// Handle scroll offset change.
/// CRITICAL: Must mark newly visible widgets dirty for them to render.
pub fn handle_scroll(
    scroll_view: WidgetId,
    old_offset: Offset,
    new_offset: Offset,
    registry: &WidgetRegistry,
    placement_cache: &PlacementCache,
    dirty: &mut DirtyTracker,
) {
    let scroll = registry.get_scroll_state(scroll_view).unwrap();
    let viewport = scroll.viewport_size;

    // Calculate exposed regions in CONTENT SPACE (what's newly visible)
    let exposed_rects = calculate_exposed_regions(old_offset, new_offset, viewport);

    if exposed_rects.is_empty() {
        return;
    }

    // Get the content widget (ScrollView's single child that contains all scrollable content)
    let content_id = registry.children(scroll_view).next().unwrap();

    // Traverse content subtree, accumulating coordinates to content space
    let newly_visible = find_widgets_in_regions(
        content_id,
        Offset::ZERO,  // Content widget is at (0,0) in content space
        &exposed_rects,
        registry,
        placement_cache,
    );

    dirty.mark_widgets_dirty(newly_visible);
}

/// Calculate newly exposed regions based on scroll delta.
/// Returns rectangles in CONTENT SPACE.
fn calculate_exposed_regions(
    old_offset: Offset,
    new_offset: Offset,
    viewport: Size,
) -> Vec<Rect> {
    let mut exposed = Vec::new();
    let dx = new_offset.x - old_offset.x;
    let dy = new_offset.y - old_offset.y;

    // Vertical scroll
    if dy > 0 {
        // Scrolled down: bottom strip is newly visible
        exposed.push(Rect::new(
            new_offset.x,
            new_offset.y + viewport.height as i16 - dy,
            viewport.width,
            dy as u16,
        ));
    } else if dy < 0 {
        // Scrolled up: top strip is newly visible
        exposed.push(Rect::new(
            new_offset.x,
            new_offset.y,
            viewport.width,
            (-dy) as u16,
        ));
    }

    // Horizontal scroll
    if dx > 0 {
        // Scrolled right: right strip is newly visible
        exposed.push(Rect::new(
            new_offset.x + viewport.width as i16 - dx,
            new_offset.y,
            dx as u16,
            viewport.height,
        ));
    } else if dx < 0 {
        // Scrolled left: left strip is newly visible
        exposed.push(Rect::new(
            new_offset.x,
            new_offset.y,
            (-dx) as u16,
            viewport.height,
        ));
    }

    exposed
}

/// Find all widgets in a subtree that intersect any of the given regions.
/// Traverses descendants and accumulates coordinates to content space.
fn find_widgets_in_regions(
    widget_id: WidgetId,
    content_offset: Offset,  // Accumulated offset from content root
    exposed_rects: &[Rect],
    registry: &WidgetRegistry,
    placement_cache: &PlacementCache,
) -> Vec<WidgetId> {
    let mut result = Vec::new();

    let Some(placement) = placement_cache.get_placement(widget_id) else {
        return result;
    };

    // Transform widget region to CONTENT SPACE by adding accumulated offset
    let content_region = Rect::new(
        placement.region.x + content_offset.x,
        placement.region.y + content_offset.y,
        placement.region.width,
        placement.region.height,
    );

    // Check if this widget intersects any exposed region
    if exposed_rects.iter().any(|r| r.intersects(&content_region)) {
        result.push(widget_id);
    }

    // Recurse into children with updated offset
    let child_offset = Offset {
        x: content_offset.x + placement.region.x,
        y: content_offset.y + placement.region.y,
    };

    for child_id in registry.children(widget_id) {
        result.extend(find_widgets_in_regions(
            child_id,
            child_offset,
            exposed_rects,
            registry,
            placement_cache,
        ));
    }

    result
}
```

### 2. Scroll Blit (Terminal Optimization)

When only the scroll offset changes (not content), we can use a blit/copy optimization for terminal output:

```rust
/// Optimize scroll by copying existing buffer content.
/// NOTE: This only handles terminal damage; handle_scroll must also be
/// called to mark newly visible widgets dirty.
pub fn scroll_blit(
    buffer: &mut RenderBuffer,
    damage: &mut DamageTracker,
    dx: i16,
    dy: i16,
    viewport: Rect,
) {
    // Copy existing content by scroll delta
    buffer.copy_region(
        viewport,
        Offset { x: dx, y: dy },
    );

    // Record damage for newly exposed regions (screen-space)
    if dy > 0 {
        // Scrolled down, top strip is new
        damage.record_damage(Rect::new(viewport.x, viewport.y, viewport.width, dy as u16));
    } else if dy < 0 {
        // Scrolled up, bottom strip is new
        damage.record_damage(Rect::new(
            viewport.x,
            viewport.y + viewport.height as i16 + dy,
            viewport.width,
            (-dy) as u16,
        ));
    }
    // Similar for horizontal
}
```

### 3. Widget Virtualization

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

### 2. Two-Level Dirty Tracking
- **DirtyTracker** uses widget IDs (not regions) to track which widgets need re-rendering. This avoids coordinate space issues between parent-space placement and screen-space rendering.
- **DamageTracker** uses screen-space rectangles for terminal output optimization. Damage is recorded during render when coordinates are already transformed.

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
