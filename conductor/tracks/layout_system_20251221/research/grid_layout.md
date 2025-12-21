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
- `grid_gutter_horizontal`: Gap between rows (confusing naming - it's the horizontal line between rows)
- `grid_gutter_vertical`: Gap between columns (confusing naming - it's the vertical line between columns)
- `grid_size_columns`: Number of columns (table mode)
- `grid_size_rows`: Number of rows (table mode)

**IMPORTANT NOTE**: The gutter naming is counterintuitive in Textual:
- `gutter_vertical` is used for column spacing (vertical lines between columns)
- `gutter_horizontal` is used for row spacing (horizontal lines between rows)

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
    gutter_horizontal = styles.grid_gutter_horizontal  # Between rows
    gutter_vertical = styles.grid_gutter_vertical      # Between columns

    # Determine number of columns
    table_size_columns = max(1, styles.grid_size_columns)
    container_width = size.width

    # Handle max_column_width: limit container to fit max columns
    if max_column_width is not None:
        # Calculate how many columns fit at max width
        max_cols = max(1, container_width // max_column_width)
        # Limit to actual number of children
        max_cols = min(len(children), max_cols)
        container_width = max_cols * max_column_width
        size = Size(container_width, size.height)

    # Handle min_column_width: calculate how many columns fit
    if min_column_width is not None:
        # How many columns fit given minimum width + gutter?
        table_size_columns = max(
            1,
            (container_width + gutter_vertical) // (min_column_width + gutter_vertical)
        )
        # Don't exceed number of children
        table_size_columns = min(table_size_columns, len(children))

        # If regular mode, ensure no remainder in last row
        if self.regular:
            while len(children) % table_size_columns and table_size_columns > 1:
                table_size_columns -= 1
```

### 2. Cell Placement

Auto-placement algorithm (similar to CSS Grid auto-placement):

```python
# Maps (col, row) -> (Widget, is_origin_cell)
cell_map: dict[tuple[int, int], tuple[Widget, bool]] = {}
# Maps Widget -> (start_col, start_row, col_span, row_span) - NOTE: actual spans, not span-1
cell_size_map: dict[Widget, tuple[int, int, int, int]] = {}

def cell_coords(column_count):
    """Generator yielding (col, row) in left-to-right, top-to-bottom order."""
    row = 0
    while True:
        for column in range(column_count):
            yield (column, row)
        row += 1

def widget_coords(col_start, row_start, col_span, row_span):
    """Get all (col, row) cells occupied by a widget."""
    return {
        (col, row)
        for col in range(col_start, col_start + col_span)
        for row in range(row_start, row_start + row_span)
    }

# Initialize coordinate generator and first position
coord_gen = cell_coords(table_size_columns)
cell_coord = next(coord_gen)

for child in children:
    column_span = child.styles.column_span or 1
    row_span = child.styles.row_span or 1

    # Find first slot where widget fits without overlapping
    while True:
        column, row = cell_coord
        coords = widget_coords(column, row, column_span, row_span)
        if cell_map.keys().isdisjoint(coords):
            # Place widget in all its cells
            for coord in coords:
                cell_map[coord] = (child, coord == (column, row))  # Mark origin
            # Store actual spans (Python stores span-1 for later indexing)
            cell_size_map[child] = (column, row, column_span - 1, row_span - 1)
            break
        cell_coord = next(coord_gen)
    cell_coord = next(coord_gen)
```

**NOTE on span storage**: Textual stores `column_span - 1` and `row_span - 1` in `cell_size_map`. This is because later when computing cell bounds, it adds these values back to the start position to get the end index. For example, a widget at column 0 with span 2 is stored as `(0, row, 1, ...)` and accessed as `columns[0 + 1]` to get the second column.

### 3. Auto Column Width

For `auto` columns, find the maximum content width:

```python
for column, scalar in enumerate(column_scalars):
    if scalar.is_auto:
        width = 0
        for row in range(len(row_scalars)):
            cell_entry = cell_map.get((column, row))
            if cell_entry is not None:
                widget, is_origin = cell_entry
                # Only count single-span widgets for auto sizing
                if widget.styles.column_span == 1:
                    width = max(
                        width,
                        widget.get_content_width(size, viewport) + widget.styles.gutter.width
                    )
        column_scalars[column] = Scalar.from_number(width)
```

### 4. Resolve Column Widths

```python
# NOTE: gutter_vertical is used for columns (vertical lines between columns)
columns = resolve(
    column_scalars,        # [Scalar("1fr"), Scalar("200"), ...]
    size.width,            # Total available width
    gutter_vertical,       # Gap between columns (confusing name)
    size, viewport,
    expand=self.expand,
    shrink=self.shrink,
    minimums=column_minimums,
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
            cell_entry = cell_map.get((column, row))
            if cell_entry is not None:
                widget, is_origin = cell_entry
                # Only count single-span widgets for auto sizing
                if widget.styles.row_span == 1:
                    column_width = columns[column][1]
                    gutter_width, gutter_height = widget.styles.gutter.totals
                    widget_height = widget.get_content_height(
                        size, viewport, column_width - gutter_width
                    ) + gutter_height
                    height = max(height, widget_height)
        row_scalars[row] = Scalar.from_number(height)
```

### 6. Resolve Row Heights

```python
# NOTE: gutter_horizontal is used for rows (horizontal lines between rows)
rows = resolve(row_scalars, size.height, gutter_horizontal, size, viewport)
```

### 7. Generate Placements

```python
max_column = len(columns) - 1
max_row = len(rows) - 1

# cell_size_map stores (start_col, start_row, span_col_offset, span_row_offset)
# where span_col_offset = column_span - 1, span_row_offset = row_span - 1
for widget, (column, row, col_span_offset, row_span_offset) in cell_size_map.items():
    x = columns[column][0]
    if row > max_row:
        break
    y = rows[row][0]

    # Get end cell position using stored offset
    # If widget spans 2 columns: col_span_offset = 1, so we access columns[column + 1]
    end_col = min(max_column, column + col_span_offset)
    end_row = min(max_row, row + row_span_offset)
    x2, cell_width = columns[end_col]
    y2, cell_height = rows[end_row]

    # Total cell size includes all spanned cells
    cell_size = Size(cell_width + x2 - x, cell_height + y2 - y)

    # Get widget's box model within the cell
    box_width, box_height, margin = widget._get_box_model(
        cell_size, viewport, Fraction(cell_size.width), Fraction(cell_size.height),
        constrain_width=True, greedy=greedy
    )

    # Create region, apply offset and margin
    region = Region(x, y, int(box_width + margin.width), int(box_height + margin.height))
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

    # 1. Resolve fixed dimensions, mark fractions as None
    resolved = [
        (scalar, None) if scalar.is_fraction
        else (scalar, scalar.resolve(size, viewport))
        for scalar in dimensions
    ]

    # 2. Calculate total gutter space
    total_gutter = gutter * (len(dimensions) - 1)

    # 3. Calculate fraction unit
    total_fraction = sum(scalar.value for scalar, frac in resolved if frac is None)
    if total_fraction > 0:
        consumed = sum(frac for _, frac in resolved if frac is not None)
        remaining = max(0, total - total_gutter - consumed)
        fraction_unit = remaining / total_fraction
        resolved_fractions = [
            scalar.value * fraction_unit if frac is None else frac
            for scalar, frac in resolved
        ]
    else:
        resolved_fractions = [frac for _, frac in resolved]

    # 4. Apply expand/shrink
    if expand or shrink:
        total_space = total - total_gutter
        used_space = sum(resolved_fractions)
        if expand and used_space < total_space:
            # Distribute remaining space proportionally
            ...
        if shrink and used_space > total_space:
            # Reduce proportionally, respecting minimums
            ...

    # 5. Convert to offsets and lengths via accumulation
    offsets = [0] + list(accumulate(
        value for fraction in resolved_fractions for value in (fraction, gutter)
    ))
    return [(offsets[i*2], offsets[i*2+1] - offsets[i*2]) for i in range(len(dimensions))]
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

**IMPORTANT**: The naming is confusing in Textual:
```css
grid-gutter-horizontal: 2;  /* Gap between ROWS (horizontal lines) */
grid-gutter-vertical: 1;    /* Gap between COLUMNS (vertical lines) */
```

In the resolve() calls:
- Column widths use `gutter_vertical` (vertical lines between columns)
- Row heights use `gutter_horizontal` (horizontal lines between rows)

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

// Cell placement info - store actual spans for clarity
struct CellPlacement {
    column: usize,
    row: usize,
    column_span: usize,  // Actual span, not span-1
    row_span: usize,     // Actual span, not span-1
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

    // Cell coordinate generator
    fn next_coord(coord: (usize, usize), columns: usize) -> (usize, usize) {
        let (col, row) = coord;
        if col + 1 < columns {
            (col + 1, row)
        } else {
            (0, row + 1)
        }
    }

    fn widget_cells(col: usize, row: usize, col_span: usize, row_span: usize) -> HashSet<(usize, usize)> {
        let mut cells = HashSet::new();
        for c in col..col + col_span {
            for r in row..row + row_span {
                cells.insert((c, r));
            }
        }
        cells
    }

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
                placements.insert(child_id, CellPlacement {
                    column: col,
                    row,
                    column_span: col_span,
                    row_span: row_span,
                });
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
    // 2. Calculate fraction unit from remaining space (guard against div-by-zero)
    // 3. Resolve fraction values
    // 4. Apply expand/shrink constraints
    // 5. Return (offset, length) pairs
}
```

### Key Implementation Notes

1. **Gutter naming**: Be aware that `gutter_vertical` is for columns, `gutter_horizontal` is for rows
2. **Span storage**: Python stores span-1 for indexing convenience; in Rust, store actual spans and subtract 1 when indexing
3. **HashMap ordering**: Unlike Python dict, Rust HashMap has no ordering; use `IndexMap` if order matters
4. **Bounds checking**: Always clamp column/row indices to max_column/max_row to avoid OOB
