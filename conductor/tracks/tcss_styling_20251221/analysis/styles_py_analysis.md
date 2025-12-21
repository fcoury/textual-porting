# Analysis: Python Textual css/styles.py

## Overview

The `styles.py` file is the core of Textual's CSS system. It defines all CSS properties supported by TCSS and provides the infrastructure for style application and animation.

## Key Components

### 1. RulesMap TypedDict

Defines all 70+ CSS properties supported in TCSS:

```python
class RulesMap(TypedDict, total=False):
    # Display/Layout
    display: Display  # "block" | "none"
    visibility: Visibility  # "visible" | "hidden"
    layout: Layout  # "vertical" | "horizontal" | "grid"

    # Colors
    auto_color: bool
    color: Color
    background: Color
    background_tint: Color
    tint: Color

    # Text
    text_style: Style  # Rich Style (bold, italic, etc.)
    text_opacity: float
    text_align: TextAlign  # "start" | "end" | "center" | "justify"
    text_wrap: TextWrap
    text_overflow: TextOverflow

    # Box Model
    padding: Spacing
    margin: Spacing
    offset: ScalarOffset
    position: str  # "relative" | "absolute"

    # Borders (4 edges)
    border_top: tuple[str, Color]
    border_right: tuple[str, Color]
    border_bottom: tuple[str, Color]
    border_left: tuple[str, Color]

    # Outline (same structure as border)
    outline_top: tuple[str, Color]
    outline_right: tuple[str, Color]
    outline_bottom: tuple[str, Color]
    outline_left: tuple[str, Color]

    # Sizing
    box_sizing: BoxSizing  # "border-box" | "content-box"
    width: Scalar
    height: Scalar
    min_width: Scalar
    min_height: Scalar
    max_width: Scalar
    max_height: Scalar

    # Docking
    dock: str  # "top" | "right" | "bottom" | "left" | "none"
    split: str

    # Overflow
    overflow_x: Overflow  # "scroll" | "hidden" | "auto"
    overflow_y: Overflow

    # Layers
    layers: tuple[str, ...]
    layer: str

    # Transitions
    transitions: dict[str, Transition]

    # Scrollbar
    scrollbar_color: Color
    scrollbar_color_hover: Color
    scrollbar_color_active: Color
    scrollbar_corner_color: Color
    scrollbar_background: Color
    scrollbar_background_hover: Color
    scrollbar_background_active: Color
    scrollbar_gutter: ScrollbarGutter
    scrollbar_size_vertical: int
    scrollbar_size_horizontal: int
    scrollbar_visibility: ScrollbarVisibility

    # Alignment
    align_horizontal: AlignHorizontal  # "left" | "center" | "right"
    align_vertical: AlignVertical  # "top" | "middle" | "bottom"
    content_align_horizontal: AlignHorizontal
    content_align_vertical: AlignVertical

    # Grid
    grid_size_rows: int
    grid_size_columns: int
    grid_gutter_horizontal: int
    grid_gutter_vertical: int
    grid_rows: tuple[Scalar, ...]
    grid_columns: tuple[Scalar, ...]
    row_span: int
    column_span: int

    # Links
    link_color: Color
    auto_link_color: bool
    link_background: Color
    link_style: Style
    link_color_hover: Color
    auto_link_color_hover: bool
    link_background_hover: Color
    link_style_hover: Style

    # Border titles
    border_title_align: AlignHorizontal
    border_subtitle_align: AlignHorizontal
    auto_border_title_color: bool
    border_title_color: Color
    border_title_background: Color
    border_title_style: Style
    auto_border_subtitle_color: bool
    border_subtitle_color: Color
    border_subtitle_background: Color
    border_subtitle_style: Style

    # Visual effects
    opacity: float
    hatch: tuple[str, Color] | Literal["none"]
    overlay: Overlay
    constrain_x: Constrain
    constrain_y: Constrain
    keyline: tuple[str, Color]
    expand: Expand
    line_pad: int
```

### 2. StylesBase Class

Base class providing property descriptors and core functionality:

```python
class StylesBase:
    ANIMATABLE = {
        "offset", "padding", "margin",
        "width", "height", "min_width", "min_height", "max_width", "max_height",
        "auto_color", "color", "background", "background_tint",
        "opacity", "text_opacity", "tint",
        "scrollbar_color", "scrollbar_color_hover", "scrollbar_color_active",
        "scrollbar_background", "scrollbar_background_hover", "scrollbar_background_active",
        "scrollbar_visibility",
        "link_color", "link_background", "link_color_hover", "link_background_hover",
        "text_wrap", "text_overflow", "line_pad",
    }

    # Property descriptors (examples)
    display = StringEnumProperty(VALID_DISPLAY, "block", layout=True, display=True)
    visibility = StringEnumProperty(VALID_VISIBILITY, "visible", layout=True)
    layout = LayoutProperty()
    color = ColorProperty(Color(255, 255, 255))
    background = ColorProperty(Color(0, 0, 0, 0))
    width = ScalarProperty(percent_unit=Unit.WIDTH)
    height = ScalarProperty(percent_unit=Unit.HEIGHT)
    # ... etc
```

Key methods:
- `has_rule(rule_name: str) -> bool`
- `clear_rule(rule_name: str) -> bool`
- `get_rules() -> RulesMap`
- `set_rule(rule_name: str, value: object) -> bool`
- `get_rule(rule_name: str, default: object) -> object`
- `merge(other: StylesBase) -> None`
- `is_animatable(rule: str) -> bool`
- `parse(css: str, read_from: CSSLocation) -> Styles`

### 3. Styles Dataclass

Concrete implementation storing rules:

```python
@dataclass
class Styles(StylesBase):
    node: DOMNode | None = None
    _rules: RulesMap = field(default_factory=RulesMap)
    _updates: int = 0  # Change counter for cache invalidation
    important: set[str] = field(default_factory=set)  # !important rules
```

Key features:
- **Rule extraction with specificity**: `extract_rules(specificity, is_default_rules, tie_breaker)`
- **CSS serialization**: `css_lines` and `css` properties
- **Merge support**: Combines rules from multiple sources

### 4. RenderStyles Class

Combines base (CSS) and inline (runtime) styles:

```python
class RenderStyles(StylesBase):
    def __init__(self, node: DOMNode, base: Styles, inline_styles: Styles):
        self.node = node
        self._base_styles = base
        self._inline_styles = inline_styles
        self._animate: BoundAnimator | None = None
```

Key features:
- **Style resolution**: Inline styles override base styles
- **Animation support**: `animate()` method for property animations
- **Cache key**: `_cache_key` for efficient re-rendering

## Property Descriptor System

Properties are defined using descriptors from `_style_properties.py`:

| Descriptor | Purpose | Example |
|------------|---------|---------|
| `StringEnumProperty` | Enum values | display, visibility |
| `ColorProperty` | Color values | color, background |
| `ScalarProperty` | Dimension values | width, height |
| `SpacingProperty` | 1-4 value spacing | padding, margin |
| `BorderProperty` | Border shorthand | border |
| `BoxProperty` | Single edge border | border_top |
| `OffsetProperty` | X/Y offset | offset |
| `IntegerProperty` | Integer values | row_span |
| `BooleanProperty` | Boolean values | auto_color |
| `FractionalProperty` | 0.0-1.0 values | opacity |
| `StyleFlagsProperty` | Rich text styles | text_style |
| `LayoutProperty` | Layout type | layout |
| `TransitionsProperty` | Transition dict | transitions |

## Rust Implementation Implications

### Data Structures Needed

1. **RulesMap equivalent**: `HashMap<String, StyleValue>` or typed struct
2. **StyleValue enum**: All possible property value types
3. **Styles struct**: Holds rules + important set + update counter
4. **RenderStyles struct**: Combines base + inline with resolution

### Key Features to Implement

1. **Property descriptors**: Rust traits for type-safe property access
2. **Specificity handling**: For cascade resolution
3. **Animation support**: ANIMATABLE set + animation hooks
4. **CSS serialization**: Back to TCSS string format
5. **Merge semantics**: Inline overrides base styles

### Type Mapping

| Python Type | Rust Type |
|-------------|-----------|
| `Color` | `textual_rs::color::Color` |
| `Scalar` | `textual_rs::css::Scalar` |
| `Spacing` | `textual_rs::geometry::Spacing` |
| `Style` (Rich) | `ratatui::style::Style` |
| `tuple[str, Color]` | `(String, Color)` or `Border` struct |
