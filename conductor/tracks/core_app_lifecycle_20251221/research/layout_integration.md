# Layout Integration Analysis

## Overview

Python Textual integrates layout with the widget lifecycle through reactive properties, refresh flags, and idle event processing. Layout is triggered automatically when widgets are mounted, resized, or their styles change.

## Key Components

### 1. Widget Refresh Flags

```python
# widget.py:432-437
self._layout_required = False     # Layout needs recomputation
self._layout_updates = 0          # Counter for layout updates
self._repaint_required = False    # Widget needs repaint
self._scroll_required = False     # Scroll position update needed
self._recompose_required = False  # Need to re-run compose()
self._refresh_styles_required = False  # CSS styles need refresh
```

### 2. The refresh() Method

```python
# widget.py:4247-4299
def refresh(
    self,
    *regions: Region,
    repaint: bool = True,
    layout: bool = False,
    recompose: bool = False,
) -> Self:
    """Initiate a refresh of the widget.

    Args:
        regions: Additional screen regions to mark as dirty.
        repaint: Repaint the widget (will call render() again).
        layout: Also layout widgets in the view.
        recompose: Re-compose the widget (remove and re-mount children).
    """
    if layout and not self._layout_required:
        self._layout_required = True
        self._layout_updates += 1

    if recompose:
        self._recompose_required = True
        self.call_next(self._check_recompose)
        return self

    if not self._is_mounted:
        self._repaint_required = True
        self.check_idle()
        return self

    self._layout_cache.clear()
    if repaint:
        self._set_dirty(*regions)
        self.clear_cached_dimensions()
        self._rich_style_cache.clear()
        self._repaint_required = True

    self.check_idle()
    return self
```

### 3. Reactive Properties with Layout

```python
# widget.py:352-376
virtual_size: Reactive[Size] = Reactive(Size(0, 0), layout=True)
"""The virtual (scrollable) size of the widget."""

show_vertical_scrollbar: Reactive[bool] = Reactive(False, layout=True)
"""Show a vertical scrollbar?"""

show_horizontal_scrollbar: Reactive[bool] = Reactive(False, layout=True)
"""Show a horizontal scrollbar?"""
```

When a reactive property with `layout=True` changes, the widget's layout is automatically invalidated.

### 4. Idle Event Processing

```python
# widget.py:4460-4490
async def _on_idle(self, event: events.Idle) -> None:
    """Called when there are no more events on the queue."""
    self._check_refresh()

def _check_refresh(self) -> None:
    """Check if a refresh was requested."""
    if self._parent is not None and not self._closing:
        try:
            screen = self.screen
        except NoScreen:
            pass
        else:
            if self._refresh_styles_required:
                self._refresh_styles_required = False
                self.call_later(self._update_styles)
            if self._scroll_required:
                self._scroll_required = False
                if not self._layout_required:
                    if self.styles.keyline[0] != "none":
                        self._set_dirty()
                    screen.post_message(messages.UpdateScroll())
            if self._repaint_required:
                self._repaint_required = False
                if self.display:
                    screen.post_message(messages.Update(self))
```

### 5. Mount Triggers Layout

```python
# widget.py:154-163 (AwaitMount)
async def __call__(self) -> None:
    aws = [
        create_task(widget._mounted_event.wait(), name="await mount")
        for widget in self._widgets
    ]
    if aws:
        await wait(aws)
        self._parent.refresh(layout=True)  # Layout after mounting
        try:
            self._parent.app._update_mouse_over(self._parent.screen)
        except NoScreen:
            pass
```

### 6. Dynamic Widget Changes Trigger Layout

```python
# widget.py:1624-1629 (inserting child)
def _insert_child(self, child: Widget, target: Widget, after: bool) -> None:
    if after:
        self._nodes._insert(self._nodes.index(target) + 1, child)
    else:
        self._nodes._insert(self._nodes.index(target), child)
    self.refresh(layout=True)

# widget.py:735-740 (cover widget)
def _cover(self, widget: Widget) -> None:
    self._cover_widget = widget
    widget._parent = self
    widget._start_messages()
    widget._post_register(self.app)
    self.app.stylesheet.apply(widget)
    self.refresh(layout=True)
```

### 7. Screen Layout Refresh

```python
# screen.py:1285-1340
def _refresh_layout(self, size: Size | None = None, scroll: bool = False) -> None:
    """Refresh the layout (can change size and positions of widgets)."""
    size = self.outer_size if size is None else size
    if self.app.is_inline:
        size = size.with_height(self.app._get_inline_height())
    if not size:
        return

    self._compositor.update_widgets(self._dirty_widgets)
    self._update_timer.pause()

    # Reflow and arrange widgets
    if scroll and not self._layout_widgets:
        exposed_widgets = self._compositor.reflow_visible(self, size)
        # Send resize events to exposed widgets
        for widget, (region, order, clip, virtual_size, container_size) in exposed_widgets.items():
            widget._size_updated(region.size, virtual_size, container_size)
            widget.post_message(ResizeEvent(region.size, virtual_size, container_size))
    else:
        # Full layout pass
        self._compositor.reflow(self, size)
```

### 8. Layout Invalidation Chain

```
Widget state changes (reactive, mount, style)
    |
    v
widget.refresh(layout=True)
    |
    v
_layout_required = True
    |
    v
check_idle() -> posts Prompt message
    |
    v
Idle event when queue empty
    |
    v
_check_refresh() -> posts Update/UpdateScroll
    |
    v
Screen._refresh_layout()
    |
    v
Compositor.reflow() -> arrange(), compute sizes
    |
    v
Widget._size_updated() for changed widgets
    |
    v
Resize events to affected widgets
```

### 9. Styles and Layout

```python
# widget.py:4040-4049
async def _on_styles_updated(self) -> None:
    """Called when styles are updated."""
    try:
        if self.is_attached:
            self.screen._update_styles()
    except NoScreen:
        pass
    self._update_styles()
```

Style changes can trigger layout when layout-affecting properties change (width, height, padding, margin, etc.).

### 10. Size Update Callback

```python
# widget.py:4051-4082
def _size_updated(
    self, size: Size, virtual_size: Size, container_size: Size, layout: bool = True
) -> bool:
    """Called when the widget's size is updated.

    Returns:
        True if a resize event should be sent.
    """
    self._layout_cache.clear()
    if (
        self._size != size
        or self.virtual_size != virtual_size
        or self._container_size != container_size
    ):
        if self._size != size:
            self._set_dirty()
        self._size = size
        if layout:
            self.virtual_size = virtual_size
        else:
            self.set_reactive(Widget.virtual_size, virtual_size)
        self._container_size = container_size
        if self.is_scrollable:
            self._scroll_update(virtual_size)
        return True
    return False
```

## Layout Trigger Points

| Trigger | Code Location | Description |
|---------|--------------|-------------|
| Mount | `AwaitMount.__call__` | `refresh(layout=True)` after widgets mounted |
| Child insert | `_insert_child` | `refresh(layout=True)` when child added |
| Cover widget | `_cover` | `refresh(layout=True)` when cover set |
| Uncover widget | `_uncover` | `refresh(layout=True)` when cover removed |
| Reactive with layout=True | `Reactive.__set__` | Calls `refresh(layout=True)` |
| Style update | `_on_styles_updated` | May trigger layout if layout properties change |
| Resize event | Screen/App | Full layout reflow |

## Rust Implementation Considerations

### Refresh Flags

```rust
struct Widget {
    layout_required: Cell<bool>,
    layout_updates: Cell<u32>,
    repaint_required: Cell<bool>,
    scroll_required: Cell<bool>,
    recompose_required: Cell<bool>,
    refresh_styles_required: Cell<bool>,
}

impl Widget {
    pub fn refresh(&self, repaint: bool, layout: bool, recompose: bool) {
        if layout && !self.layout_required.get() {
            self.layout_required.set(true);
            self.layout_updates.set(self.layout_updates.get() + 1);
        }

        if recompose {
            self.recompose_required.set(true);
            self.call_next(|| self.check_recompose());
            return;
        }

        if repaint {
            self.set_dirty();
            self.clear_cached_dimensions();
            self.repaint_required.set(true);
        }

        self.check_idle();
    }
}
```

### Layout Integration with Reactive

```rust
// In reactive macro
#[reactive(layout = true)]
pub virtual_size: Size,

// Generated setter
pub fn set_virtual_size(&mut self, value: Size) {
    if self._reactive_virtual_size != value {
        self._reactive_virtual_size = value;
        self.refresh(repaint: true, layout: true, recompose: false);
        self.watch_virtual_size(old_value, value);
    }
}
```

### Idle Processing

```rust
impl Widget {
    async fn on_idle(&mut self, _event: IdleEvent) {
        self.check_refresh();
    }

    fn check_refresh(&mut self) {
        if self.parent.is_some() && !self.closing {
            if let Ok(screen) = self.screen() {
                if self.refresh_styles_required.get() {
                    self.refresh_styles_required.set(false);
                    self.call_later(|| self.update_styles());
                }
                if self.scroll_required.get() {
                    self.scroll_required.set(false);
                    if !self.layout_required.get() {
                        screen.post_message(UpdateScroll);
                    }
                }
                if self.repaint_required.get() {
                    self.repaint_required.set(false);
                    if self.display() {
                        screen.post_message(Update(self.id()));
                    }
                }
            }
        }
    }
}
```

### Mount Layout Trigger

```rust
impl AwaitMount {
    pub async fn await_mount(self) {
        for widget in &self.widgets {
            widget.mounted_event().await;
        }
        self.parent.refresh(repaint: false, layout: true, recompose: false);
    }
}
```

## Key Requirements for Rust Port

1. **Refresh Flags**: Cell-based flags for layout/repaint state
2. **Idle Processing**: Check flags during idle and post appropriate messages
3. **Layout Trigger Points**: Mount, child changes, reactive updates
4. **Reactive layout=true**: Auto-trigger layout on property change
5. **Size Update Callback**: Notify widgets of size changes
6. **Layout Cache**: Invalidate on layout changes

## Summary

| Python Concept | Rust Equivalent |
|---------------|-----------------|
| `_layout_required` | `Cell<bool>` |
| `refresh(layout=True)` | Method setting flag + `check_idle()` |
| `_on_idle` | Async handler checking flags |
| `Reactive(layout=True)` | Macro attribute triggering `refresh()` |
| `_size_updated` | Method called by layout engine |
| `check_idle()` | Posts `Prompt` message to trigger idle |
