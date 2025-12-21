# Analysis: Arrangement Algorithm

## Overview

Python Textual's `_arrange.py` module contains the core `arrange()` function that orchestrates the complete widget layout process. It handles layers, split widgets, docked widgets, and finally layout widgets in a specific order.

## Main `arrange()` Function

```python
def arrange(
    widget: Widget,
    children: Sequence[Widget],
    size: Size,
    viewport: Size,
    optimal: bool = False,
) -> DockArrangeResult:
```

### Parameters

- `widget`: The parent container widget
- `children`: Child widgets to arrange
- `size`: Available area for layout
- `viewport`: Terminal size (for percentage calculations)
- `optimal`: If True, use greedy=False for box model calculations (content-fitting mode)

### Returns

```python
@dataclass
class DockArrangeResult:
    placements: list[WidgetPlacement]  # All widget positions
    widgets: set[Widget]               # Widgets that were arranged
    scroll_spacing: Spacing            # Space to exclude from scrollable area
```

## Algorithm Overview

### 1. Filter Display Widgets

```python
display_widgets = list(filter(_get_display, children))
```

Only widgets with `display: true` (visible) are considered.

### 2. Organize into Layers

```python
layers = _build_layers(display_widgets)
# Returns: {"default": [...], "overlay": [...], ...}
```

Widgets are grouped by their `layer` property. Each layer is processed independently.

### 3. Process Each Layer

For each layer:

#### 3a. Handle Split Widgets

Split widgets are like docks but reduce the available area for all subsequent widgets:

```python
non_split_widgets, split_widgets = partition(_get_split, widgets)
if split_widgets:
    _split_placements, dock_region = _arrange_split_widgets(
        split_widgets, size, viewport
    )
    placements.extend(_split_placements)
else:
    dock_region = size.region
```

#### 3b. Handle Docked Widgets

Docked widgets attach to edges and **do reduce the layout area** for subsequent widgets. The returned `dock_spacing` is used to shrink `dock_region` before laying out regular widgets:

```python
layout_widgets, dock_widgets = partition(_get_dock, non_split_widgets)
if dock_widgets:
    _dock_placements, dock_spacing = _arrange_dock_widgets(
        dock_widgets, dock_region, viewport, greedy=not optimal
    )
    placements.extend(_dock_placements)
    # IMPORTANT: dock_region is shrunk by dock_spacing before layout
    dock_region = dock_region.shrink(dock_spacing)
```

**NOTE**: The key difference between dock and split is:
- **Split**: Physically divides the region, each split widget gets exclusive space
- **Dock**: Multiple widgets can dock to the same edge (they overlap each other), and the combined spacing reduces the layout area

#### 3c. Handle Layout Widgets

Regular widgets are arranged using the container's layout:

```python
if layout_widgets:
    layout_placements = widget.process_layout(
        widget.layout.arrange(
            widget, layout_widgets, dock_region.size, greedy=not optimal
        )
    )
```

#### 3d. Apply Alignment

```python
if styles.align_horizontal != "left" or styles.align_vertical != "top":
    bounding_region = WidgetPlacement.get_bounds(layout_placements)
    placement_offset += styles._align_size(bounding_region.size, container_size)
```

#### 3e. Translate and Apply Absolute Positioning

```python
if placement_offset:
    layout_placements = WidgetPlacement.translate(layout_placements, placement_offset)
WidgetPlacement.apply_absolute(layout_placements)
placements.extend(layout_placements)
```

## Dock Widget Arrangement

```python
def _arrange_dock_widgets(
    dock_widgets: Sequence[Widget],
    region: Region,
    viewport: Size,
    greedy: bool = True,
) -> tuple[list[WidgetPlacement], Spacing]:
```

### Algorithm

```python
top = right = bottom = left = 0

for dock_widget in dock_widgets:
    edge = dock_widget.styles.dock  # "top", "right", "bottom", "left"

    # Get widget's box model
    box_model = dock_widget._get_box_model(size, viewport, ...)
    widget_width = int(box_model.width) + margin.width
    widget_height = int(box_model.height) + margin.height

    # Position based on edge
    if edge == "bottom":
        dock_region = Region(0, height - widget_height, widget_width, widget_height)
        bottom = max(bottom, widget_height)
    elif edge == "top":
        dock_region = Region(0, 0, widget_width, widget_height)
        top = max(top, widget_height)
    elif edge == "left":
        dock_region = Region(0, 0, widget_width, widget_height)
        left = max(left, widget_width)
    elif edge == "right":
        dock_region = Region(width - widget_width, 0, widget_width, widget_height)
        right = max(right, widget_width)

    # Apply margin and offset
    dock_region = dock_region.shrink(margin)
    placements.append(WidgetPlacement(dock_region, offset, ...))

dock_spacing = Spacing(top, right, bottom, left)
return (placements, dock_spacing)
```

### Key Points

- Docked widgets get Z-order `TOP_Z` (highest, always on top)
- Multiple widgets can dock to same edge (they stack)
- Returns spacing that reduces the content area

## Split Widget Arrangement

```python
def _arrange_split_widgets(
    split_widgets: Sequence[Widget],
    size: Size,
    viewport: Size,
) -> tuple[list[WidgetPlacement], Region]:
```

### Algorithm

Split widgets work like docks but actually divide the available region:

```python
view_region = size.region

for split_widget in split_widgets:
    split = split_widget.styles.split  # "top", "right", "bottom", "left"

    if split == "bottom":
        view_region, split_region = view_region.split_horizontal(-widget_height)
    elif split == "top":
        split_region, view_region = view_region.split_horizontal(widget_height)
    elif split == "left":
        split_region, view_region = view_region.split_vertical(widget_width)
    elif split == "right":
        view_region, split_region = view_region.split_vertical(-widget_width)

    placements.append(WidgetPlacement(split_region, ...))

return placements, view_region  # view_region is the remaining space
```

### Difference from Dock

- **Split**: Physically divides the region. Each split widget carves out exclusive space, and the remaining `view_region` is passed to subsequent processing.
- **Dock**: Widgets are positioned at edges and may overlap each other if multiple widgets dock to the same edge. The combined `dock_spacing` reduces the layout area for regular widgets. The `scroll_spacing` from docking is used to calculate the scrollable content area.

## Layers

Widgets can belong to different layers, processed in order:

```python
layers = _build_layers(display_widgets)
# {"default": [widget1, widget2], "overlay": [widget3], ...}

for widgets in layers.values():
    # Process each layer independently
```

This allows overlay widgets to be rendered after base widgets.

**IMPORTANT**: Python dicts maintain insertion order (Python 3.7+), so layers are processed in the order widgets were added. In Rust, `HashMap` does **not** guarantee order. Use `IndexMap` from the `indexmap` crate if layer processing order matters, or collect layer names and sort them before iteration.

## Rust Implementation Notes

### Data Flow

```rust
pub fn arrange(
    registry: &WidgetRegistry,
    parent_id: WidgetId,
    children: &[WidgetId],
    size: Size,
    viewport: Size,
    optimal: bool,
) -> ArrangeResult {
    // 1. Filter visible widgets
    let display_widgets: Vec<_> = children
        .iter()
        .filter(|id| registry.get_display(*id))
        .copied()
        .collect();

    // 2. Build layers
    let layers = build_layers(&display_widgets, registry);

    // 3. Process each layer
    for widgets in layers.values() {
        // 3a. Handle split widgets
        let (non_split, split_widgets) = partition_by_split(widgets, registry);
        let (split_placements, dock_region) = arrange_split_widgets(
            &split_widgets, size, viewport, registry
        );
        placements.extend(split_placements);

        // 3b. Handle docked widgets
        let (layout_widgets, dock_widgets) = partition_by_dock(&non_split, registry);
        let (dock_placements, dock_spacing) = arrange_dock_widgets(
            &dock_widgets, dock_region, viewport, registry
        );
        placements.extend(dock_placements);
        let dock_region = dock_region.shrink(dock_spacing);

        // 3c. Handle layout widgets
        let layout = registry.get_layout(parent_id);
        let layout_placements = layout.arrange(
            registry, parent_id, &layout_widgets, dock_region.size, !optimal
        );

        // 3d. Apply alignment
        // 3e. Apply offset and absolute positioning
        placements.extend(layout_placements);
    }

    ArrangeResult { placements, widgets, scroll_spacing }
}
```

### Helper Functions

```rust
fn partition_by_dock(
    widgets: &[WidgetId],
    registry: &WidgetRegistry,
) -> (Vec<WidgetId>, Vec<WidgetId>) {
    widgets.iter().partition(|id| {
        registry.get_styles(*id).is_docked()
    })
}

// Use IndexMap to preserve insertion order (like Python dict)
fn build_layers(
    widgets: &[WidgetId],
    registry: &WidgetRegistry,
) -> IndexMap<String, Vec<WidgetId>> {
    let mut layers: IndexMap<String, Vec<WidgetId>> = IndexMap::new();
    for &widget_id in widgets {
        let layer = registry.get_layer(widget_id);
        layers.entry(layer).or_default().push(widget_id);
    }
    layers
}
```

### Key Constants

```rust
const TOP_Z: i32 = i32::MAX;  // Z-order for docked widgets
```

### Region Operations

Need `Region::split_horizontal()` and `Region::split_vertical()`:

```rust
impl Rect {
    pub fn split_horizontal(&self, at: i16) -> (Rect, Rect) {
        let at = if at < 0 { self.height as i16 + at } else { at };
        let top = Rect::new(self.x, self.y, self.width, at as u16);
        let bottom = Rect::new(self.x, self.y + at, self.width, self.height - at as u16);
        (top, bottom)
    }

    pub fn split_vertical(&self, at: i16) -> (Rect, Rect) {
        let at = if at < 0 { self.width as i16 + at } else { at };
        let left = Rect::new(self.x, self.y, at as u16, self.height);
        let right = Rect::new(self.x + at, self.y, self.width - at as u16, self.height);
        (left, right)
    }
}
```
