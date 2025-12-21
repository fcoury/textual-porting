# Layout Integration Design

## Overview

Layout integration connects the widget lifecycle with the layout system. When widgets are mounted, resized, or have reactive properties change, the layout system must be notified to recalculate positions and sizes. This design defines the trigger points, refresh flags, and idle processing mechanism.

## Goals

1. **Automatic Triggering**: Layout updates happen automatically on state changes
2. **Batched Updates**: Multiple changes are batched for efficiency
3. **Flag-Based**: Simple flags indicate what needs updating
4. **Idle Processing**: Updates are applied during idle events
5. **Minimal Recomputation**: Only recompute what changed

## Design Decisions

### 1. Cell-Based Refresh Flags

Use `Cell<bool>` for interior mutability on refresh flags, allowing immutable widget references to schedule refreshes:

```rust
struct WidgetState {
    layout_required: Cell<bool>,
    repaint_required: Cell<bool>,
    recompose_required: Cell<bool>,
}
```

### 2. Idle Event Processing

Refresh actions are deferred to idle events (when message queue is empty):

```rust
// Widget changes state
widget.set_visible(true);  // Sets layout_required = true

// Later, when queue is empty
fn on_idle(&mut self) {
    self.check_refresh();  // Processes pending refresh
}
```

### 3. Refresh Propagation

Layout changes propagate up the tree while repaint changes propagate down:
- Layout: Child change affects parent sizing
- Repaint: Parent change affects child rendering

## Core Types

### RefreshFlags

```rust
use std::cell::Cell;

bitflags::bitflags! {
    /// Flags indicating what refresh actions are needed.
    #[derive(Clone, Copy, Debug, Default)]
    pub struct RefreshFlags: u8 {
        /// Widget needs to be repainted.
        const REPAINT = 0b0001;
        /// Widget needs layout recalculation.
        const LAYOUT = 0b0010;
        /// Widget needs to re-compose children.
        const RECOMPOSE = 0b0100;
        /// Widget styles need refresh.
        const STYLES = 0b1000;
    }
}
```

### WidgetRefreshState

```rust
/// Tracks pending refresh operations for a widget.
#[derive(Default)]
pub struct WidgetRefreshState {
    /// Layout recalculation is needed.
    pub layout_required: Cell<bool>,
    /// Number of layout updates (for tracking changes).
    pub layout_updates: Cell<u32>,
    /// Widget needs repaint.
    pub repaint_required: Cell<bool>,
    /// Widget needs to re-run compose().
    pub recompose_required: Cell<bool>,
    /// Styles need to be recalculated.
    pub refresh_styles_required: Cell<bool>,
    /// Scroll position needs update.
    pub scroll_required: Cell<bool>,
}

impl WidgetRefreshState {
    pub fn new() -> Self {
        Self::default()
    }

    /// Check if any refresh is pending.
    pub fn needs_refresh(&self) -> bool {
        self.layout_required.get()
            || self.repaint_required.get()
            || self.recompose_required.get()
            || self.refresh_styles_required.get()
    }

    /// Clear all refresh flags.
    pub fn clear(&self) {
        self.layout_required.set(false);
        self.repaint_required.set(false);
        self.recompose_required.set(false);
        self.refresh_styles_required.set(false);
        self.scroll_required.set(false);
    }
}
```

### Refresh Method

```rust
impl Widget {
    /// Request a refresh with the specified options.
    pub fn refresh(&self, flags: RefreshFlags) {
        let state = self.refresh_state();

        if flags.contains(RefreshFlags::LAYOUT) && !state.layout_required.get() {
            state.layout_required.set(true);
            state.layout_updates.set(state.layout_updates.get() + 1);
        }

        if flags.contains(RefreshFlags::RECOMPOSE) {
            state.recompose_required.set(true);
            // Recompose is handled immediately via call_next
            self.call_next(|| self.check_recompose());
            return;
        }

        if flags.contains(RefreshFlags::REPAINT) {
            self.set_dirty();
            self.clear_cached_dimensions();
            state.repaint_required.set(true);
        }

        if flags.contains(RefreshFlags::STYLES) {
            state.refresh_styles_required.set(true);
        }

        // Trigger idle check
        self.check_idle();
    }

    /// Convenience method for repaint only.
    pub fn refresh_repaint(&self) {
        self.refresh(RefreshFlags::REPAINT);
    }

    /// Convenience method for layout refresh.
    pub fn refresh_layout(&self) {
        self.refresh(RefreshFlags::REPAINT | RefreshFlags::LAYOUT);
    }
}
```

## Layout Trigger Points

### 1. Mount/Unmount

```rust
impl WidgetRegistry {
    /// Mount a widget and trigger layout.
    pub async fn mount(&mut self, widget_id: WidgetId) {
        // ... mounting logic ...

        // Schedule layout after mount
        if let Some(parent) = self.parent(widget_id) {
            self.get(parent).refresh(RefreshFlags::LAYOUT);
        }
    }

    /// Unmount a widget and trigger layout.
    pub async fn unmount(&mut self, widget_id: WidgetId) {
        // ... unmounting logic ...

        // Schedule layout after unmount
        if let Some(parent) = self.parent(widget_id) {
            self.get(parent).refresh(RefreshFlags::LAYOUT);
        }
    }
}
```

### 2. Reactive Properties with layout=true

```rust
// In the derive macro generated code
impl MyWidget {
    pub fn set_size(&mut self, value: Size) {
        let old = std::mem::replace(&mut self._reactive_size, value);
        if old != self._reactive_size {
            // Trigger layout because layout=true
            self.refresh(RefreshFlags::REPAINT | RefreshFlags::LAYOUT);
            self.watch_size(old, &self._reactive_size);
        }
    }
}

// Usage:
#[derive(Reactive)]
struct MyWidget {
    #[reactive(layout = true)]
    size: Size,
}
```

### 3. Child Insertion/Removal

```rust
impl Widget {
    /// Insert a child widget at a position.
    pub fn insert_child(&mut self, child: WidgetId, position: usize) {
        self.children_mut().insert(position, child);
        self.refresh(RefreshFlags::LAYOUT);
    }

    /// Remove a child widget.
    pub fn remove_child(&mut self, child: WidgetId) {
        self.children_mut().retain(|&id| id != child);
        self.refresh(RefreshFlags::LAYOUT);
    }
}
```

### 4. Style Changes

```rust
impl Widget {
    /// Set inline styles.
    pub fn set_styles(&mut self, styles: &str) -> Result<(), StyleError> {
        self.parse_and_apply_styles(styles)?;

        // Check if layout-affecting properties changed
        if self.styles_affect_layout() {
            self.refresh(RefreshFlags::LAYOUT | RefreshFlags::REPAINT);
        } else {
            self.refresh(RefreshFlags::REPAINT);
        }

        Ok(())
    }

    fn styles_affect_layout(&self) -> bool {
        // Check if width, height, padding, margin, etc. changed
        self.has_layout_style_change()
    }
}
```

### 5. Resize Events

```rust
impl Screen {
    /// Handle terminal resize.
    pub fn on_resize(&mut self, new_size: Size) {
        if self.size != new_size {
            self.size = new_size;
            // Full layout refresh needed
            self.refresh_layout();
        }
    }
}
```

## Idle Processing

```rust
impl Widget {
    /// Called when message queue is empty.
    pub async fn on_idle(&mut self, _event: Idle) {
        self.check_refresh();
    }

    /// Process pending refresh operations.
    fn check_refresh(&mut self) {
        let state = self.refresh_state();

        // Skip if closing or detached
        if self.is_closing() || !self.is_attached() {
            return;
        }

        let screen = match self.screen() {
            Some(s) => s,
            None => return,
        };

        // Process style refresh
        if state.refresh_styles_required.get() {
            state.refresh_styles_required.set(false);
            self.call_later(|| self.update_styles());
        }

        // Process scroll refresh
        if state.scroll_required.get() {
            state.scroll_required.set(false);
            if !state.layout_required.get() {
                screen.post_message(UpdateScroll);
            }
        }

        // Process repaint
        if state.repaint_required.get() {
            state.repaint_required.set(false);
            if self.is_visible() {
                screen.post_message(Update { widget: self.id() });
            }
        }

        // Layout is processed at screen level
        if state.layout_required.get() {
            screen.schedule_layout();
        }
    }
}
```

## Screen Layout Refresh

```rust
impl Screen {
    /// Schedule a layout refresh.
    pub fn schedule_layout(&mut self) {
        if !self.layout_scheduled.get() {
            self.layout_scheduled.set(true);
            self.call_next(|| self.refresh_layout());
        }
    }

    /// Perform layout refresh.
    pub fn refresh_layout(&mut self) {
        self.layout_scheduled.set(false);

        let size = self.outer_size();
        if size.is_zero() {
            return;
        }

        // Update compositor with dirty widgets
        self.compositor.update_widgets(&self.dirty_widgets);
        self.dirty_widgets.clear();

        // Perform reflow
        let exposed_widgets = self.compositor.reflow(self, size);

        // Notify widgets of size changes
        for (widget_id, layout_info) in exposed_widgets {
            if let Some(widget) = self.get_widget(widget_id) {
                widget.size_updated(layout_info);
            }
        }
    }
}
```

## Size Update Callback

```rust
/// Layout information for a widget after layout computation.
pub struct LayoutInfo {
    /// The widget's computed size.
    pub size: Size,
    /// The virtual (scrollable) size.
    pub virtual_size: Size,
    /// The container size.
    pub container_size: Size,
    /// The widget's position relative to parent.
    pub position: Position,
}

impl Widget {
    /// Called when the widget's size is updated after layout.
    pub fn size_updated(&mut self, info: LayoutInfo) -> bool {
        // Clear layout cache
        self.clear_layout_cache();

        // Check if anything actually changed
        let changed = self.size != info.size
            || self.virtual_size != info.virtual_size
            || self.container_size != info.container_size;

        if changed {
            // Mark as dirty if size changed
            if self.size != info.size {
                self.set_dirty();
            }

            // Update stored values
            self.size = info.size;
            self.virtual_size = info.virtual_size;
            self.container_size = info.container_size;

            // Update scroll if scrollable
            if self.is_scrollable() {
                self.scroll_update(info.virtual_size);
            }

            // Post resize event
            self.post_message(Resize {
                size: info.size,
                virtual_size: info.virtual_size,
                container_size: info.container_size,
            });

            return true;
        }

        false
    }
}
```

## Layout Cache Invalidation

```rust
/// Cached layout computations.
pub struct LayoutCache {
    /// Cached content size.
    content_size: Option<Size>,
    /// Cached min/max constraints.
    constraints: Option<Constraints>,
    /// Cache version (increments on invalidation).
    version: u32,
}

impl Widget {
    /// Invalidate the layout cache.
    pub fn clear_layout_cache(&mut self) {
        self.layout_cache.content_size = None;
        self.layout_cache.constraints = None;
        self.layout_cache.version += 1;
    }

    /// Clear cached dimensions (for repaint).
    pub fn clear_cached_dimensions(&mut self) {
        self.cached_width = None;
        self.cached_height = None;
    }
}
```

## Integration with Layout Engine

```rust
/// Integration point with the layout engine from the Layout System track.
pub trait LayoutEngine {
    /// Compute layout for a widget tree.
    fn compute_layout(
        &mut self,
        root: WidgetId,
        available_size: Size,
        registry: &WidgetRegistry,
    ) -> HashMap<WidgetId, LayoutInfo>;
}

impl Screen {
    /// Integrate with layout engine during reflow.
    pub fn reflow(&mut self, size: Size) -> HashMap<WidgetId, LayoutInfo> {
        self.layout_engine.compute_layout(
            self.root_widget,
            size,
            &self.registry,
        )
    }
}
```

## Dirty Widget Tracking

```rust
impl Screen {
    /// Mark a widget as dirty (needs repaint).
    pub fn mark_dirty(&mut self, widget_id: WidgetId) {
        self.dirty_widgets.insert(widget_id);
    }

    /// Get all dirty widgets.
    pub fn dirty_widgets(&self) -> &HashSet<WidgetId> {
        &self.dirty_widgets
    }

    /// Clear dirty widget set.
    pub fn clear_dirty(&mut self) {
        self.dirty_widgets.clear();
    }
}

impl Widget {
    /// Mark this widget as dirty.
    pub fn set_dirty(&self) {
        if let Some(screen) = self.screen() {
            screen.borrow_mut().mark_dirty(self.id());
        }
    }
}
```

## Example Flow

```rust
// 1. Widget property changes
widget.set_visible(true);  // reactive with layout=true

// 2. Generated setter triggers refresh
fn set_visible(&mut self, value: bool) {
    self._reactive_visible = value;
    self.refresh(RefreshFlags::LAYOUT | RefreshFlags::REPAINT);
}

// 3. Refresh sets flags and schedules idle check
fn refresh(&self, flags: RefreshFlags) {
    state.layout_required.set(true);
    state.repaint_required.set(true);
    self.check_idle();
}

// 4. When message queue empties, on_idle fires
async fn on_idle(&mut self, _: Idle) {
    self.check_refresh();
}

// 5. check_refresh processes flags
fn check_refresh(&mut self) {
    if state.layout_required.get() {
        screen.schedule_layout();
    }
    if state.repaint_required.get() {
        screen.post_message(Update { widget: self.id() });
    }
}

// 6. Screen performs layout
fn refresh_layout(&mut self) {
    let layouts = self.compositor.reflow(self, size);
    for (id, info) in layouts {
        widget.size_updated(info);
    }
}

// 7. Widget receives Resize event
async fn on_resize(&mut self, event: Resize) {
    // Handle resize...
}
```

## Comparison with Python Textual

| Python Textual | Rust Implementation |
|---------------|---------------------|
| `_layout_required` | `Cell<bool>` in `WidgetRefreshState` |
| `_repaint_required` | `Cell<bool>` in `WidgetRefreshState` |
| `refresh(layout=True)` | `refresh(RefreshFlags::LAYOUT)` |
| `check_idle()` | `check_idle()` posts Prompt message |
| `_on_idle()` | `on_idle()` async handler |
| `_check_refresh()` | `check_refresh()` processes flags |
| `_refresh_layout()` | `refresh_layout()` calls layout engine |
| `_size_updated()` | `size_updated(info)` callback |

## Implementation Plan

### Phase 1: Refresh Flags
- [ ] Define `RefreshFlags` bitflags
- [ ] Define `WidgetRefreshState` struct
- [ ] Add refresh state to Widget

### Phase 2: Refresh Method
- [ ] Implement `refresh()` method
- [ ] Implement `check_idle()` trigger
- [ ] Implement convenience methods

### Phase 3: Idle Processing
- [ ] Implement `on_idle()` handler
- [ ] Implement `check_refresh()` method
- [ ] Connect to message pump

### Phase 4: Trigger Points
- [ ] Add layout trigger on mount/unmount
- [ ] Add layout trigger on child changes
- [ ] Add layout trigger in reactive macro
- [ ] Add layout trigger on style changes

### Phase 5: Screen Integration
- [ ] Implement `schedule_layout()`
- [ ] Implement `refresh_layout()`
- [ ] Implement dirty widget tracking
- [ ] Implement `size_updated()` callback

### Phase 6: Layout Engine Integration
- [ ] Define `LayoutEngine` trait
- [ ] Connect to Layout System track
- [ ] Implement `reflow()` method

## Open Questions

1. **Batching Window**: How long should we wait before processing batched updates?

2. **Priority Refreshes**: Should some refreshes skip the idle queue?

3. **Async Layout**: Should layout computation be async for complex trees?

4. **Partial Layout**: Can we avoid full reflow for local changes?

## Summary

This design provides:
- **Automatic layout triggering** from lifecycle events
- **Batched updates** via idle processing
- **Cell-based flags** for interior mutability
- **Clear integration points** with layout engine
- **Efficient dirty tracking** for minimal recomputation
