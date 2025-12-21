# Analysis: Grid Layout

## Overview

Python Textual's `GridLayout` implements a CSS Grid-like 2D layout system. Widgets are placed in cells defined by rows and columns, with support for spanning multiple cells, auto-sizing, and gaps.

## GridLayout Class (`layouts/grid.py`)

### Configuration Options

```python
class GridLayout(Layout):
    min_column_width: int | None = None   # Minimum column width
    max_column_width: int | None = None   # Maximum column width
    stretch_height: bool = False          # Stretch cells to equal height
    regular: bool = False                 # No remainder in last row
    expand: bool = False                  # Expand grid to fill container
    shrink: bool = False                  # Shrink grid to fit container
    auto_minimum: bool = False            # Auto-detect min width when shrinking
    _grid_size: tuple[int, int] | None    # (columns, rows) after arrange
```

### Grid CSS Properties

From parent widget styles:
- `grid_rows`: List of Scalars for row heights (default: `["1fr"]` or `["auto"]`)
- `grid_columns`: List of Scalars for column widths (default: `["1fr"]`)
- `grid_gutter_horizontal`: Horizontal gap between cells
- `grid_gutter_vertical`: Vertical gap between cells
- `grid_size_columns`: Number of columns (table mode)
- `grid_size_rows`: Number of rows (table mode)

From child widget styles:
- `column_span`: Number of columns to span (default: 1)
- `row_span`: Number of rows to span (default: 1)

## Algorithm

### 1. Setup

```python
def arrange(self, parent, children, size, greedy=True):
    # Get grid templates from parent styles
    row_scalars = styles.grid_rows or [Scalar("1fr")]
    column_scalars = styles.grid_columns or [Scalar("1fr")]
    gutter_h = styles.grid_gutter_horizontal
    gutter_v = styles.grid_gutter_vertical

    # Handle min/max column width constraints
    table_size_columns = styles.grid_size_columns
    if min_column_width:
        table_size_columns = container_width // (min_column_width + gutter)
    if max_column_width:
        container_width = min(children, container_width // max_column_width) * max_column_width
```

### 2. Cell Placement

Auto-placement algorithm (similar to CSS Grid auto-placement):

```python
cell_map: dict[tuple[int, int], tuple[Widget, bool]] = {}
cell_size_map: dict[Widget, tuple[int, int, int, int]] = {}

next_coord = cell_coords(table_size_columns)  # Generator: (0,0), (1,0), (2,0), (0,1), ...

for child in children:
    column_span = child.styles.column_span or 1
    row_span = child.styles.row_span or 1

    # Find first slot where widget fits without overlapping
    while True:
        column, row = cell_coord
        coords = widget_coords(column, row, column_span, row_span)
        if no_overlap(cell_map, coords):
            # Place widget in all its cells
            for coord in coords:
                cell_map[coord] = (child, coord == cell_coord)  # Mark origin
            cell_size_map[child] = (column, row, column_span-1, row_span-1)
            break
        cell_coord = next(next_coord)
```

### 3. Auto Column Width

For `auto` columns, find the maximum content width:

```python
for column, scalar in enumerate(column_scalars):
    if scalar.is_auto:
        width = 0
        for row in range(len(row_scalars)):
            widget = cell_map.get((column, row))
            if widget and widget.styles.column_span == 1:
                width = max(width, widget.get_content_width() + gutter)
        column_scalars[column] = Scalar.from_number(width)
```

### 4. Resolve Column Widths

```python
columns = resolve(
    column_scalars,        # [Scalar("1fr"), Scalar("200"), ...]
    size.width,            # Total available width
    gutter_vertical,       # Gap between columns
    size, viewport,
    expand=self.expand,    # Expand columns to fill
    shrink=self.shrink,    # Shrink columns to fit
    minimums=column_minimums,  # Min widths for shrink
)
# Returns: [(offset, length), (offset, length), ...]
```

### 5. Auto Row Height

For `auto` rows, find the maximum content height:

```python
for row, scalar in enumerate(row_scalars):
    if scalar.is_auto:
        height = 0
        for column in range(len(column_scalars)):
            widget = cell_map.get((column, row))
            if widget and widget.styles.row_span == 1:
                column_width = columns[column][1]
                height = max(height, widget.get_content_height(column_width))
        row_scalars[row] = Scalar.from_number(height)
```

### 6. Resolve Row Heights

```python
rows = resolve(row_scalars, size.height, gutter_horizontal, size, viewport)
```

### 7. Generate Placements

```python
for widget, (column, row, column_span, row_span) in cell_size_map.items():
    x = columns[column][0]
    y = rows[row][0]

    # Handle spanning: get end cell positions
    x2, cell_width = columns[column + column_span]
    y2, cell_height = rows[row + row_span]
    cell_size = Size(cell_width + x2 - x, cell_height + y2 - y)

    # Get widget's box model within the cell
    box_width, box_height, margin = widget._get_box_model(
        cell_size, viewport, cell_size.width, cell_size.height
    )

    # Create region, apply offset and margin
    region = Region(x, y, box_width + margin.width, box_height + margin.height)
    region = region.crop_size(cell_size).shrink(margin) + offset

    placements.append(WidgetPlacement(region, offset, margin, widget, ...))
```

## The `resolve()` Function

Core function for distributing space across rows/columns:

```python
def resolve(
    dimensions: Sequence[Scalar],  # Column/row sizes
    total: int,                    # Total available space
    gutter: int,                   # Gap between items
    size: Size,
    viewport: Size,
    expand: bool = False,          # Expand to fill
    shrink: bool = False,          # Shrink to fit
    minimums: list[int] | None = None,  # Min sizes for shrink
) -> list[tuple[int, int]]:
    """Returns: [(offset, length), (offset, length), ...]"""

    # 1. Resolve fixed dimensions
    resolved = [scalar.resolve(size, viewport) if not is_fraction else None]

    # 2. Calculate fraction unit
    total_fraction = sum(fr values)
    total_gutter = gutter * (len(dimensions) - 1)
    consumed = sum(fixed sizes)
    remaining = total - total_gutter - consumed
    fraction_unit = remaining / total_fraction

    # 3. Resolve all fractions
    resolved_fractions = [fraction * fraction_unit for fraction in fr_values]

    # 4. Apply expand/shrink
    if expand and remaining_space > 0:
        # Distribute remaining space proportionally
    if shrink and excess_space > 0:
        # Reduce proportionally, respecting minimums

    # 5. Convert to offsets and lengths
    offsets = accumulate(widths + gutters)
    return [(offset, length) for ...]
```

## Key Features

### Spanning

Widgets can span multiple cells:
```python
# In TCSS
.widget {
    column-span: 2;
    row-span: 3;
}
```

The auto-placement algorithm respects spans when finding empty slots.

### Auto Sizing

- `auto` columns: Width based on content
- `auto` rows: Height based on content
- Only single-span widgets contribute to auto sizing

### Fractional Units

- `1fr`: 1 fraction of remaining space
- `2fr`: 2 fractions (twice as wide/tall as 1fr)
- Mixed: `[200, 1fr, 2fr]` = 200px fixed + remaining split 1:2

### Gaps (Gutters)

```css
grid-gutter-horizontal: 2;  /* Between columns */
grid-gutter-vertical: 1;    /* Between rows */
```

### Keyline Support

Special handling for keylines (visual borders):
- Adds 2px padding when keylines are enabled
- Adjusts gutter spacing to account for keyline overlap

## Rust Implementation Notes

### Data Structures

```rust
pub struct GridLayout {
    pub min_column_width: Option<u16>,
    pub max_column_width: Option<u16>,
    pub stretch_height: bool,
    pub regular: bool,
    pub expand: bool,
    pub shrink: bool,
    pub auto_minimum: bool,
    grid_size: Option<(usize, usize)>,
}

// Cell placement info
struct CellPlacement {
    column: usize,
    row: usize,
    column_span: usize,
    row_span: usize,
}
```

### Auto-Placement Algorithm

```rust
fn place_widgets(
    children: &[WidgetId],
    registry: &WidgetRegistry,
    columns: usize,
) -> HashMap<WidgetId, CellPlacement> {
    let mut occupied: HashSet<(usize, usize)> = HashSet::new();
    let mut placements = HashMap::new();

    let mut coord = (0, 0);

    for &child_id in children {
        let styles = registry.get_styles(child_id);
        let col_span = styles.column_span.unwrap_or(1);
        let row_span = styles.row_span.unwrap_or(1);

        loop {
            let (col, row) = coord;
            let cells = widget_cells(col, row, col_span, row_span);

            if occupied.is_disjoint(&cells) {
                occupied.extend(&cells);
                placements.insert(child_id, CellPlacement { ... });
                break;
            }

            coord = next_coord(coord, columns);
        }
        coord = next_coord(coord, columns);
    }

    placements
}
```

### Space Resolution

```rust
pub fn resolve_grid_dimension(
    scalars: &[Scalar],
    total: i32,
    gutter: i32,
    size: Size,
    viewport: Size,
    expand: bool,
    shrink: bool,
    minimums: Option<&[i32]>,
) -> Vec<(i32, i32)> {
    // 1. Resolve non-fraction values
    // 2. Calculate fraction unit from remaining space
    // 3. Resolve fraction values
    // 4. Apply expand/shrink constraints
    // 5. Return (offset, length) pairs
}
```

### Key Differences from Python

1. Use `HashMap` instead of `dict` for cell map
2. Use `HashSet` for occupied cell tracking
3. Iterator-based cell coordinate generation
4. Separate function for auto-sizing columns/rows
