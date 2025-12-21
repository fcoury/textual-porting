# Analysis: CSS Size Properties

## Overview

Python Textual uses a `Scalar` type to represent CSS size values with various units. Scalars are resolved to concrete pixel values during layout based on container size, viewport size, and fraction unit calculations.

## Unit Types

```python
@unique
class Unit(Enum):
    CELLS = 1        # Explicit cell count: width: 10
    FRACTION = 2     # Fractional unit: width: 2fr
    PERCENT = 3      # Percentage: width: 50%
    WIDTH = 4        # Container width %: width: 50w
    HEIGHT = 5       # Container height %: height: 50h
    VIEW_WIDTH = 6   # Viewport width %: width: 25vw
    VIEW_HEIGHT = 7  # Viewport height %: height: 25vh
    AUTO = 8         # Auto-size: width: auto
```

### Unit Symbols

```python
UNIT_SYMBOL = {
    Unit.CELLS: "",      # No suffix: "10"
    Unit.FRACTION: "fr", # Fraction: "2fr"
    Unit.PERCENT: "%",   # Percent: "50%"
    Unit.WIDTH: "w",     # Width %: "50w"
    Unit.HEIGHT: "h",    # Height %: "50h"
    Unit.VIEW_WIDTH: "vw",   # Viewport width: "25vw"
    Unit.VIEW_HEIGHT: "vh",  # Viewport height: "25vh"
}
```

## Scalar Type

```python
class Scalar(NamedTuple):
    value: float       # Numeric value
    unit: Unit         # The unit type
    percent_unit: Unit # What % resolves to (WIDTH or HEIGHT)
```

### Properties

```python
@property
def is_cells(self) -> bool:
    """Check if explicit cell count."""
    return self.unit == Unit.CELLS

@property
def is_percent(self) -> bool:
    """Check if percentage unit."""
    return self.unit == Unit.PERCENT

@property
def is_fraction(self) -> bool:
    """Check if fraction unit (fr)."""
    return self.unit == Unit.FRACTION

@property
def is_auto(self) -> bool:
    """Check if auto-sized."""
    return self.unit == Unit.AUTO

@property
def cells(self) -> int | None:
    """Get explicit cell count, or None."""
    return int(self.value) if self.unit == Unit.CELLS else None

@property
def fraction(self) -> int | None:
    """Get fraction value, or None."""
    return int(self.value) if self.unit == Unit.FRACTION else None
```

## Parsing

```python
@classmethod
@lru_cache(maxsize=1024)
def parse(cls, token: str, percent_unit: Unit = Unit.WIDTH) -> Scalar:
    """Parse a string into a Scalar.

    Examples:
        "10"    -> Scalar(10.0, Unit.CELLS, percent_unit)
        "2fr"   -> Scalar(2.0, Unit.FRACTION, percent_unit)
        "50%"   -> Scalar(50.0, Unit.PERCENT, percent_unit)
        "25vw"  -> Scalar(25.0, Unit.VIEW_WIDTH, percent_unit)
        "auto"  -> Scalar(1.0, Unit.AUTO, Unit.AUTO)
    """
```

Pattern: `^(-?\d+\.?\d*)(fr|%|w|h|vw|vh)?$`

## Resolution

```python
@lru_cache(maxsize=4096)
def resolve(
    self,
    size: Size,           # Container size
    viewport: Size,       # Terminal/viewport size
    fraction_unit: Fraction | None = None,  # Size of 1fr
) -> Fraction:
    """Resolve scalar to actual dimension."""
```

### Resolution Functions

```python
def _resolve_cells(value, size, viewport, fraction_unit) -> Fraction:
    """width: 10 -> 10 cells"""
    return Fraction(value)

def _resolve_fraction(value, size, viewport, fraction_unit) -> Fraction:
    """width: 2fr -> 2 * fraction_unit cells"""
    return fraction_unit * Fraction(value)

def _resolve_width(value, size, viewport, fraction_unit) -> Fraction:
    """width: 50w -> 50% of container width"""
    return Fraction(value) * Fraction(size.width, 100)

def _resolve_height(value, size, viewport, fraction_unit) -> Fraction:
    """height: 50h -> 50% of container height"""
    return Fraction(value) * Fraction(size.height, 100)

def _resolve_view_width(value, size, viewport, fraction_unit) -> Fraction:
    """width: 25vw -> 25% of viewport width"""
    return Fraction(value) * Fraction(viewport.width, 100)

def _resolve_view_height(value, size, viewport, fraction_unit) -> Fraction:
    """height: 25vh -> 25% of viewport height"""
    return Fraction(value) * Fraction(viewport.height, 100)
```

## CSS Size Properties

### Primary Size Properties

| Property | Description |
|----------|-------------|
| `width` | Widget width |
| `height` | Widget height |
| `min-width` | Minimum width constraint |
| `min-height` | Minimum height constraint |
| `max-width` | Maximum width constraint |
| `max-height` | Maximum height constraint |

### Usage Examples

```css
.widget {
    /* Fixed size */
    width: 20;
    height: 10;

    /* Fractional sizing */
    width: 1fr;        /* Fill remaining space */
    height: 2fr;       /* Fill 2x as much as 1fr */

    /* Percentage of container */
    width: 50%;        /* 50% of container width */
    height: 100%;      /* Full container height */

    /* Viewport-relative */
    width: 80vw;       /* 80% of terminal width */
    height: 50vh;      /* 50% of terminal height */

    /* Constraints */
    min-width: 10;     /* At least 10 cells wide */
    max-width: 100;    /* At most 100 cells wide */
    min-height: 5;     /* At least 5 cells tall */
    max-height: 50vh;  /* At most 50% viewport height */

    /* Auto-size */
    height: auto;      /* Size to content */
}
```

## Fraction Unit Distribution

When laying out widgets with `fr` units:

1. **Sum all fr values**: `total_fr = sum(widget.fr_value for widget in children)`
2. **Calculate remaining space**: `remaining = container_size - fixed_sizes - margins`
3. **Calculate fr unit size**: `fr_size = remaining / total_fr`
4. **Apply to each widget**: `widget_size = widget.fr_value * fr_size`

### Handling Min/Max Constraints

```python
def resolve_fraction_unit(widget_styles, size, viewport, remaining_space, dimension):
    """Calculate the fr unit size, respecting min/max constraints."""
    remaining_fraction = sum(fr_values)

    while remaining_fraction > 0:
        changed = False
        fr_size = remaining_space / remaining_fraction

        for widget in fr_widgets:
            resolved = widget.fr_value * fr_size

            if resolved < widget.min_size:
                # Lock to minimum, remove from fr pool
                remaining_space -= widget.min_size
                remaining_fraction -= widget.fr_value
                widget.locked = True
                changed = True

            elif resolved > widget.max_size:
                # Lock to maximum, remove from fr pool
                remaining_space -= widget.max_size
                remaining_fraction -= widget.fr_value
                widget.locked = True
                changed = True

        if not changed:
            break

    return remaining_space / remaining_fraction
```

## ScalarOffset

For offset properties that have both X and Y components:

```python
class ScalarOffset(NamedTuple):
    x: Scalar
    y: Scalar

    def resolve(self, size: Size, viewport: Size) -> Offset:
        """Resolve to integer offset."""
        return Offset(
            round(self.x.resolve(size, viewport)),
            round(self.y.resolve(size, viewport)),
        )

# Used for:
# offset: 10 5;     -> ScalarOffset(Scalar(10, CELLS), Scalar(5, CELLS))
# offset: 10% 20%;  -> ScalarOffset(Scalar(10, PERCENT), Scalar(20, PERCENT))
```

## Rust Implementation Notes

### Scalar Type

```rust
#[derive(Debug, Clone, Copy, PartialEq)]
pub enum Unit {
    Cells,
    Fraction,
    Percent,
    Width,
    Height,
    ViewWidth,
    ViewHeight,
    Auto,
}

#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Scalar {
    pub value: f32,
    pub unit: Unit,
    pub percent_unit: Unit,
}

impl Scalar {
    pub fn cells(value: f32) -> Self {
        Self { value, unit: Unit::Cells, percent_unit: Unit::Width }
    }

    pub fn fraction(value: f32) -> Self {
        Self { value, unit: Unit::Fraction, percent_unit: Unit::Width }
    }

    pub fn auto() -> Self {
        Self { value: 1.0, unit: Unit::Auto, percent_unit: Unit::Auto }
    }

    pub fn is_fraction(&self) -> bool {
        self.unit == Unit::Fraction
    }

    pub fn is_auto(&self) -> bool {
        self.unit == Unit::Auto
    }

    pub fn resolve(&self, size: Size, viewport: Size, fraction_unit: i32) -> i32 {
        match self.unit {
            Unit::Cells => self.value as i32,
            Unit::Fraction => (self.value * fraction_unit as f32) as i32,
            Unit::Width | Unit::Percent if self.percent_unit == Unit::Width => {
                (self.value * size.width as f32 / 100.0) as i32
            }
            Unit::Height | Unit::Percent if self.percent_unit == Unit::Height => {
                (self.value * size.height as f32 / 100.0) as i32
            }
            Unit::ViewWidth => (self.value * viewport.width as f32 / 100.0) as i32,
            Unit::ViewHeight => (self.value * viewport.height as f32 / 100.0) as i32,
            Unit::Auto => 0,  // Auto is handled specially
            _ => 0,
        }
    }
}
```

### Parsing

```rust
impl Scalar {
    pub fn parse(s: &str) -> Result<Self, ScalarParseError> {
        if s.eq_ignore_ascii_case("auto") {
            return Ok(Self::auto());
        }

        // Regex: ^(-?\d+\.?\d*)(fr|%|w|h|vw|vh)?$
        // Or manual parsing for performance

        let (value_str, unit_str) = split_value_and_unit(s);
        let value: f32 = value_str.parse()?;

        let unit = match unit_str {
            "" => Unit::Cells,
            "fr" => Unit::Fraction,
            "%" => Unit::Percent,
            "w" => Unit::Width,
            "h" => Unit::Height,
            "vw" => Unit::ViewWidth,
            "vh" => Unit::ViewHeight,
            _ => return Err(ScalarParseError::InvalidUnit),
        };

        Ok(Self { value, unit, percent_unit: Unit::Width })
    }
}
```

### Caching

Python uses `@lru_cache` for both parsing and resolution. In Rust, consider:
- Parse caching: `HashMap<String, Scalar>` or `lru::LruCache`
- Resolution caching: Struct-level cache or memoization

For resolution, the inputs (size, viewport, fraction_unit) change frequently, so caching may not help as much as parsing caching.
