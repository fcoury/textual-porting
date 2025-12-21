# Analysis: Python Textual css/_style_properties.py

## Overview

Property descriptors in `_style_properties.py` enable type-safe CSS property access with validation, conversion, and automatic refresh triggering. They abstract the complexity of accepting various input types while ensuring type safety.

## Property Descriptor Architecture

### Base: GenericProperty

```python
class GenericProperty(Generic[PropertyGetType, PropertySetType]):
    def __init__(
        self,
        default: PropertyGetType,
        layout: bool = False,        # Refresh layout on change
        refresh_children: bool = False,
    ) -> None:
        ...

    def validate_value(self, value: object) -> PropertyGetType:
        """Override in subclasses to validate/transform values"""
        return cast(PropertyGetType, value)

    def __get__(self, obj: StylesBase, objtype) -> PropertyGetType:
        return obj.get_rule(self.name, self.default)

    def __set__(self, obj: StylesBase, value: PropertySetType | None) -> None:
        if value is None:
            obj.clear_rule(self.name)
            obj.refresh(layout=self.layout, children=self.refresh_children)
            return
        new_value = self.validate_value(value)
        if obj.set_rule(self.name, new_value):
            obj.refresh(layout=self.layout, children=self.refresh_children)
```

## Property Descriptors Catalog

### 1. Simple Value Properties

| Property | Get Type | Set Type | Description |
|----------|----------|----------|-------------|
| `IntegerProperty` | `int` | `int \| float` | Integer values (row_span) |
| `BooleanProperty` | `bool` | `bool` | Boolean values (auto_color) |
| `FractionalProperty` | `float` | `float \| str` | 0.0-1.0 values (opacity) |
| `NameProperty` | `str` | `str` | Single name (layer) |
| `NameListProperty` | `tuple[str, ...]` | `str \| tuple[str]` | Space-separated names (layers) |

### 2. Scalar Properties

```python
class ScalarProperty:
    def __init__(
        self,
        units: set[Unit] | None = None,  # Allowed units
        percent_unit: Unit = Unit.WIDTH,  # How to interpret %
        allow_auto: bool = True,
    ) -> None:
        ...
```

**Input formats:**
- `int/float`: Interpreted as cells (e.g., `10` → `10 cells`)
- `Scalar`: Direct Scalar object
- `str`: Parsed string (e.g., `"50%"`, `"10vw"`, `"auto"`)

### 3. Spacing Property

```python
class SpacingProperty:
    # Accepts 1, 2, 4 values like CSS
    # "1"       → Spacing(1, 1, 1, 1)
    # "1 2"     → Spacing(1, 2, 1, 2)
    # "1 2 3 4" → Spacing(1, 2, 3, 4)
```

### 4. Box/Border Properties

**Single edge (BoxProperty):**
```python
class BoxProperty:
    # Get: tuple[EdgeType, Color]
    # Set: tuple[EdgeType, str | Color] | "none"
    # Example: ("round", Color.parse("green"))
```

**Full border (BorderProperty):**
```python
class BorderProperty:
    # Get: Edges (NamedTuple with top, right, bottom, left)
    # Set: Single tuple or sequence of 1, 2, or 4 tuples

class Edges(NamedTuple):
    top: tuple[EdgeType, Color]
    right: tuple[EdgeType, Color]
    bottom: tuple[EdgeType, Color]
    left: tuple[EdgeType, Color]

    @property
    def spacing(self) -> Spacing:
        """Calculate spacing from border widths"""
```

### 5. Color Property

```python
class ColorProperty:
    # Get: Color
    # Set: Color | str | None

    # String parsing:
    # - Named: "red", "blue"
    # - Hex: "#ff0000", "#f00"
    # - RGB: "rgb(255, 0, 0)"
    # - With opacity: "red 50%"
```

**ScrollbarColorProperty** extends this to also refresh scrollbar widgets.

### 6. Enum Property

```python
class StringEnumProperty(Generic[EnumType]):
    def __init__(
        self,
        valid_values: set[str],
        default: EnumType,
        layout: bool = False,
        refresh_children: bool = False,
        refresh_parent: bool = False,
        display: bool = False,  # Forces nodes update
    ) -> None:
        ...
```

**OverflowProperty** extends this with scrollbar refresh hook.

### 7. Layout Property

```python
class LayoutProperty:
    # Get: Layout | None
    # Set: str | Layout | None

    # Accepts layout name ("vertical", "horizontal", "grid")
    # or Layout object directly
```

### 8. Offset Property

```python
class OffsetProperty:
    # Get: ScalarOffset
    # Set: tuple[int | str, int | str] | ScalarOffset | None

    # Example inputs:
    # (10, 20)        → 10 cells, 20 cells
    # ("0.5vw", "10") → 0.5 viewport width, 10 cells
```

### 9. Style Flags Property

```python
class StyleFlagsProperty:
    # Get: Style (Rich Style)
    # Set: Style | str | None

    # String parsing: "bold italic underline"
    # Valid flags: bold, italic, underline, strike, reverse, dim, none
```

### 10. Transitions Property

```python
class TransitionsProperty:
    # Get: dict[str, Transition]
    # Set: dict[str, Transition] | None
```

### 11. Compound Properties

**AlignProperty:**
```python
class AlignProperty:
    # Combines align_horizontal and align_vertical
    # Get: tuple[AlignHorizontal, AlignVertical]
    # Set: tuple[AlignHorizontal, AlignVertical]
```

**HatchProperty:**
```python
class HatchProperty:
    # Get: tuple[str, Color] | Literal["none"]
    # Set: tuple[str, Color | str] | Literal["none"] | None
```

## Refresh Semantics

Properties trigger appropriate refreshes:

| Refresh Type | Description |
|--------------|-------------|
| `layout=True` | Recalculate widget positions |
| `children=True` | Also refresh child widgets |
| `parent=True` | Also refresh parent widget |
| `repaint=True` (default) | Trigger repaint |

## Rust Implementation Strategy

### Trait-Based Approach

```rust
trait StyleProperty<T> {
    fn get(&self, styles: &Styles) -> T;
    fn set(&mut self, styles: &mut Styles, value: impl Into<Option<T>>) -> bool;
    fn default(&self) -> T;
}
```

### Type Conversion

```rust
// Scalar conversion
impl From<i32> for Scalar { ... }
impl From<f64> for Scalar { ... }
impl FromStr for Scalar { ... }

// Color conversion
impl From<&str> for Color { ... }
impl From<(u8, u8, u8)> for Color { ... }
```

### Validation

```rust
fn validate_enum<T>(value: &str, valid: &HashSet<&str>) -> Result<T, StyleValueError>
fn validate_scalar_units(scalar: &Scalar, allowed: &HashSet<Unit>) -> Result<(), StyleValueError>
```

### Refresh Triggering

```rust
struct PropertyMeta {
    layout: bool,
    refresh_children: bool,
    refresh_parent: bool,
}

fn set_property(styles: &mut Styles, name: &str, value: T, meta: &PropertyMeta) {
    if styles.set_rule(name, value) {
        styles.refresh(RefreshFlags {
            layout: meta.layout,
            children: meta.refresh_children,
            parent: meta.refresh_parent,
        });
    }
}
```

## Key Patterns

1. **Null handling**: Setting to `None` clears the rule
2. **Type coercion**: Accept multiple input types, store canonical type
3. **Validation**: Fail fast with helpful error messages
4. **Change detection**: Only refresh if value actually changed
5. **Compound properties**: Delegate to component properties
