# Analysis: Vertical and Horizontal Layouts

## Overview

Python Textual's `VerticalLayout` and `HorizontalLayout` classes arrange widgets along a single axis. Both inherit from the abstract `Layout` base class and implement the `arrange()` method.

## Layout Base Class (`layout.py`)

### Key Types

```python
@dataclass
class DockArrangeResult:
    placements: list[WidgetPlacement]
    widgets: set[Widget]
    scroll_spacing: Spacing
    _spatial_map: SpatialMap[WidgetPlacement] | None

class WidgetPlacement(NamedTuple):
    region: Region       # Position and size
    offset: Offset       # Additional offset (from CSS offset)
    margin: Spacing      # Widget margins
    widget: Widget       # The widget itself
    order: int = 0       # Z-order
    fixed: bool = False  # Fixed position
    overlay: bool = False # Screen overlay
    absolute: bool = False # Absolute positioning
```

### Layout Trait

```python
class Layout(ABC):
    name: ClassVar[str] = ""

    @abstractmethod
    def arrange(
        self,
        parent: Widget,
        children: list[Widget],
        size: Size,
        greedy: bool = True,
    ) -> ArrangeResult:
        """Generate layout map for children within parent."""

    def get_content_width(self, widget, container, viewport) -> int:
        """Calculate content width by arranging children."""

    def get_content_height(self, widget, container, viewport, width) -> int:
        """Calculate content height by arranging children."""
```

## VerticalLayout (`vertical.py`)

### Algorithm

1. **Pre-layout hook**: `parent.pre_layout(self)` - allows widgets to prepare for layout
2. **Collect margins**: Gather margins from children, excluding `overlay: screen` widgets
3. **Calculate resolve margin**: Sum of vertical margins for constraint solving
4. **Resolve box models**: Call `resolve_box_models()` with `resolve_dimension="height"`
5. **Collapse margins**: Larger of adjacent bottom/top margins wins
6. **Position widgets**: Stack top-to-bottom, respecting margins and offsets

### Key Details

- Uses `Fraction` for precise positioning (avoids float rounding)
- Supports `overlay: screen` - widgets positioned independently
- Supports `position: absolute` - widgets positioned absolutely
- Handles CSS `offset` for widget displacement

### Code Structure

```python
def arrange(self, parent, children, size, greedy=True):
    # 1. Get child styles and margins
    child_styles = [child.styles for child in children]
    box_margins = [styles.margin for styles in child_styles if not overlay]

    # 2. Calculate total margin space
    resolve_margin = Size(max_horizontal_margin, sum_vertical_margins)

    # 3. Resolve all heights via box models
    box_models = resolve_box_models(
        [styles.height for styles in child_styles],
        children, size, viewport, resolve_margin,
        resolve_dimension="height", greedy=greedy
    )

    # 4. Position each widget
    y = first_margin_top
    for widget, (width, height, margin), next_margin in ...:
        region = Region(margin.left, y, width, height)
        y += height + collapsed_margin
        placements.append(WidgetPlacement(region, offset, margin, widget, ...))
```

## HorizontalLayout (`horizontal.py`)

### Algorithm

Nearly identical to VerticalLayout but:
- Resolves `width` instead of `height`
- Stacks left-to-right instead of top-to-bottom
- Horizontal margins collapse instead of vertical

### Code Structure

```python
def arrange(self, parent, children, size, greedy=True):
    # 1. Resolve widths
    box_models = resolve_box_models(
        [styles.width for styles in child_styles],
        children, size, viewport, resolve_margin,
        resolve_dimension="width", greedy=greedy
    )

    # 2. Position left-to-right
    x = first_margin_left
    for widget, (width, height, margin), next_margin in ...:
        region = Region(x, margin.top, width, height)
        x += width + collapsed_margin
        placements.append(WidgetPlacement(region, offset, margin, widget, ...))
```

## Box Model Resolution (`_resolve.py`)

### `resolve_box_models()`

Core function that converts CSS dimensions to pixel values:

1. **Fixed dimensions**: Resolve directly (px, %, etc.)
2. **Fraction units**: Handle `fr` units by distributing remaining space
3. **Min/max constraints**: Apply min-width/height, max-width/height

### `resolve_fraction_unit()`

Calculates the size of 1fr unit:

```python
def resolve_fraction_unit(widget_styles, size, viewport_size, remaining_space, resolve_dimension):
    # Sum all fr values
    total_fraction = sum(scalar.value for scalar in fr_scalars)

    # Initial fr value
    fraction_unit = remaining_space / total_fraction

    # Iteratively apply min/max constraints
    while remaining_fraction > 0:
        for each fr widget:
            resolved = scalar.resolve(size, viewport, fraction_unit)
            if resolved < min_value:
                remaining_space -= min_value
                remaining_fraction -= scalar.value
            elif resolved > max_value:
                remaining_space -= max_value
                remaining_fraction -= scalar.value

        # Recalculate fraction_unit with remaining space
        fraction_unit = remaining_space / remaining_fraction
```

### Key Insights for Rust Port

1. **Fraction-based math**: Use `Fraction` type for precise sub-pixel positioning
2. **Two-pass layout**: First resolve box models, then position widgets
3. **Margin collapsing**: Adjacent margins collapse (larger wins)
4. **Greedy expansion**: `greedy=True` expands widgets to fill space
5. **Overlay/absolute**: Special handling for screen overlays and absolute positioning

## StreamLayout (`stream.py`)

A simplified, faster vertical layout for long lists:
- All widgets fill width (1fr)
- All heights are `auto`
- Caches placements for performance
- No absolute positioning or overlays

## Rust Implementation Notes

### Data Structures

```rust
pub struct WidgetPlacement {
    pub region: Rect,
    pub offset: Offset,
    pub margin: Spacing,
    pub widget_id: WidgetId,
    pub order: i32,
    pub fixed: bool,
    pub overlay: bool,
    pub absolute: bool,
}

pub trait Layout {
    fn name(&self) -> &'static str;

    fn arrange(
        &self,
        registry: &WidgetRegistry,
        parent_id: WidgetId,
        children: &[WidgetId],
        size: Size,
        greedy: bool,
    ) -> Vec<WidgetPlacement>;

    fn get_content_width(&self, ...) -> u16;
    fn get_content_height(&self, ...) -> u16;
}
```

### Fraction Handling

Options:
1. Use `num-rational` crate for exact fractions
2. Use fixed-point arithmetic (e.g., multiply by 1000)
3. Use floats with careful rounding

Recommendation: Start with `i64` fixed-point (multiply by 256), convert to pixels at final step.

### Box Model Resolution

```rust
pub fn resolve_box_models(
    dimensions: &[Option<Scalar>],
    widgets: &[WidgetId],
    registry: &WidgetRegistry,
    size: Size,
    viewport: Size,
    margin: Size,
    dimension: Dimension,
    greedy: bool,
) -> Vec<BoxModel> {
    // Similar algorithm to Python
}
```
