# Design: TCSS Property Types

## Overview

This document defines the Rust type system for all TCSS (Textual CSS) properties. The design supports 70+ CSS properties with type-safe parsing, validation, and value storage.

## Property Value Types

### Core Value Enums

```rust
/// All possible CSS property values
#[derive(Debug, Clone, PartialEq)]
pub enum StyleValue {
    // Scalars and dimensions
    Scalar(Scalar),
    ScalarOffset(ScalarOffset),
    Spacing(Spacing),

    // Colors
    Color(Color),

    // Text and display
    TextAlign(TextAlign),
    TextStyle(TextStyle),
    ContentAlign(ContentAlign),

    // Layout
    Display(Display),
    Visibility(Visibility),
    Overflow(Overflow),
    BoxSizing(BoxSizing),

    // Borders
    Border(Border),
    BorderEdge(BorderEdge),
    Outline(Outline),

    // Layout system
    Layout(LayoutKind),
    Dock(DockEdge),

    // Grid (integers)
    GridSizeRows(i32),
    GridSizeColumns(i32),
    GridGutterHorizontal(i32),
    GridGutterVertical(i32),
    GridRows(Vec<Scalar>),
    GridColumns(Vec<Scalar>),

    // Scrollbar (separate horizontal/vertical integers)
    ScrollbarSizeHorizontal(i32),
    ScrollbarSizeVertical(i32),
    ScrollbarGutter(ScrollbarGutter),

    // Special
    Transition(Transition),
    Hatch(Hatch),
    Keyline(Keyline),

    // Primitives
    Integer(i32),
    Float(f64),
    Bool(bool),
    String(String),
    NameList(Vec<String>),

    // Auto/None/Initial
    Auto,
    None,
    Initial,
}
```

### Scalar Type

```rust
/// Represents a dimensional value with unit
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Scalar {
    pub value: f64,
    pub unit: Unit,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Unit {
    // Absolute
    Cells,      // Default unit (character cells)

    // Relative to parent
    Percent,    // % of parent dimension
    Width,      // % of parent width (w)
    Height,     // % of parent height (h)

    // Relative to viewport
    ViewWidth,  // % of viewport width (vw)
    ViewHeight, // % of viewport height (vh)

    // Special
    Auto,       // Automatic sizing
    Fraction,   // Grid fraction (fr)
}

impl Scalar {
    pub const ZERO: Scalar = Scalar { value: 0.0, unit: Unit::Cells };
    pub const AUTO: Scalar = Scalar { value: 0.0, unit: Unit::Auto };

    pub fn cells(value: f64) -> Self {
        Scalar { value, unit: Unit::Cells }
    }

    pub fn percent(value: f64) -> Self {
        Scalar { value, unit: Unit::Percent }
    }

    pub fn is_auto(&self) -> bool {
        self.unit == Unit::Auto
    }
}
```

### Spacing Type (Box Model)

```rust
/// Four-sided spacing for margin/padding
#[derive(Debug, Clone, Copy, PartialEq, Default)]
pub struct Spacing {
    pub top: Scalar,
    pub right: Scalar,
    pub bottom: Scalar,
    pub left: Scalar,
}

impl Spacing {
    pub const ZERO: Spacing = Spacing {
        top: Scalar::ZERO,
        right: Scalar::ZERO,
        bottom: Scalar::ZERO,
        left: Scalar::ZERO,
    };

    /// CSS-style 1-value: all sides same
    pub fn all(value: Scalar) -> Self {
        Spacing { top: value, right: value, bottom: value, left: value }
    }

    /// CSS-style 2-value: vertical, horizontal
    pub fn symmetric(vertical: Scalar, horizontal: Scalar) -> Self {
        Spacing { top: vertical, right: horizontal, bottom: vertical, left: horizontal }
    }

    /// CSS-style 4-value: top, right, bottom, left
    pub fn new(top: Scalar, right: Scalar, bottom: Scalar, left: Scalar) -> Self {
        Spacing { top, right, bottom, left }
    }
}
```

### Border Types

```rust
/// Border edge style
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum BorderKind {
    #[default]
    None,
    Ascii,
    Blank,
    Double,
    Dashed,
    Heavy,
    Hidden,
    Hkey,
    Inner,
    Outer,
    Panel,
    Round,
    Solid,
    Tall,
    Thick,
    Vkey,
    Wide,
}

/// Single border edge (style + color)
#[derive(Debug, Clone, PartialEq)]
pub struct BorderEdge {
    pub kind: BorderKind,
    pub color: Color,
}

/// All four border edges
#[derive(Debug, Clone, PartialEq)]
pub struct Border {
    pub top: BorderEdge,
    pub right: BorderEdge,
    pub bottom: BorderEdge,
    pub left: BorderEdge,
}
```

### Text Properties

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum TextAlign {
    #[default]
    Start,   // Logical start (LTR: left, RTL: right)
    End,     // Logical end (LTR: right, RTL: left)
    Left,
    Center,
    Right,
    Justify,
}

/// Rich text style flags
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub struct TextStyle {
    pub bold: bool,
    pub dim: bool,
    pub italic: bool,
    pub underline: bool,
    pub underline2: bool,  // Double underline
    pub blink: bool,
    pub blink2: bool,      // Rapid blink
    pub reverse: bool,
    pub strike: bool,
    pub overline: bool,
}

impl TextStyle {
    pub const NONE: TextStyle = TextStyle {
        bold: false, dim: false, italic: false, underline: false,
        underline2: false, blink: false, blink2: false,
        reverse: false, strike: false, overline: false,
    };

    /// Parse from space-separated flags
    pub fn parse(s: &str) -> Result<Self, StyleValueError> {
        let mut style = TextStyle::NONE;
        for flag in s.split_whitespace() {
            match flag {
                "bold" | "b" => style.bold = true,
                "dim" => style.dim = true,
                "italic" | "i" => style.italic = true,
                "underline" | "u" => style.underline = true,
                "underline2" | "uu" => style.underline2 = true,
                "blink" => style.blink = true,
                "blink2" => style.blink2 = true,
                "reverse" => style.reverse = true,
                "strike" | "s" => style.strike = true,
                "overline" | "o" => style.overline = true,
                "none" => return Ok(TextStyle::NONE),
                "not" => {} // Prefix for negation, handle separately
                _ => return Err(StyleValueError::InvalidStyleFlag(flag.into())),
            }
        }
        Ok(style)
    }
}
```

### Layout Properties

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum Display {
    #[default]
    Block,
    None,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum Visibility {
    #[default]
    Visible,
    Hidden,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum Overflow {
    #[default]
    Hidden,
    Auto,
    Scroll,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum BoxSizing {
    #[default]
    BorderBox,
    ContentBox,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum LayoutKind {
    Vertical,
    Horizontal,
    Grid,
    Stream,   // Note: Not supported in Python Textual's sibling selectors
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum DockEdge {
    #[default]
    None,
    Top,
    Right,
    Bottom,
    Left,
}
```

### Alignment Properties

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum AlignHorizontal {
    #[default]
    Left,
    Center,
    Right,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum AlignVertical {
    #[default]
    Top,
    Middle,
    Bottom,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub struct ContentAlign {
    pub horizontal: AlignHorizontal,
    pub vertical: AlignVertical,
}
```

### Transition and Animation

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct Transition {
    pub property: String,        // Property name or "all"
    pub duration: Duration,      // Transition duration
    pub easing: EasingFunction,  // Timing function
    pub delay: Duration,         // Start delay
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum EasingFunction {
    #[default]
    Linear,
    InSine,
    OutSine,
    InOutSine,
    InQuad,
    OutQuad,
    InOutQuad,
    InCubic,
    OutCubic,
    InOutCubic,
    InQuart,
    OutQuart,
    InOutQuart,
    InQuint,
    OutQuint,
    InOutQuint,
    InExpo,
    OutExpo,
    InOutExpo,
    InCirc,
    OutCirc,
    InOutCirc,
    InBack,
    OutBack,
    InOutBack,
    InElastic,
    OutElastic,
    InOutElastic,
    InBounce,
    OutBounce,
    InOutBounce,
}
```

### Special Properties

```rust
/// Hatch pattern fill
#[derive(Debug, Clone, PartialEq)]
pub struct Hatch {
    pub character: char,
    pub color: Color,
}

/// Keyline for canvas drawing
#[derive(Debug, Clone, PartialEq)]
pub struct Keyline {
    pub style: String,
    pub color: Color,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum ScrollbarGutter {
    #[default]
    Auto,
    Stable,
}
```

## Property Name Registry

```rust
/// All supported TCSS property names
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum PropertyName {
    // Dimensions
    Width,
    Height,
    MinWidth,
    MinHeight,
    MaxWidth,
    MaxHeight,

    // Box model
    Margin,
    MarginTop,
    MarginRight,
    MarginBottom,
    MarginLeft,
    Padding,
    PaddingTop,
    PaddingRight,
    PaddingBottom,
    PaddingLeft,

    // Borders
    Border,
    BorderTop,
    BorderRight,
    BorderBottom,
    BorderLeft,
    BorderTitleColor,
    BorderTitleBackground,
    BorderTitleStyle,
    BorderTitleAlign,
    BorderSubtitleColor,
    BorderSubtitleBackground,
    BorderSubtitleStyle,
    BorderSubtitleAlign,

    // Outline
    Outline,
    OutlineTop,
    OutlineRight,
    OutlineBottom,
    OutlineLeft,

    // Colors
    Color,
    Background,
    BackgroundTint,
    Tint,
    AutoColor,

    // Text
    TextAlign,
    TextStyle,
    TextOpacity,
    TextWrap,
    TextOverflow,

    // Layout
    Display,
    Visibility,
    Opacity,
    Layout,
    Dock,
    Layer,
    Layers,
    Overlay,
    ConstrainX,
    ConstrainY,
    Expand,
    LinePad,

    // Alignment
    AlignHorizontal,
    AlignVertical,
    ContentAlignHorizontal,
    ContentAlignVertical,

    // Sizing
    BoxSizing,

    // Overflow
    Overflow,
    OverflowX,
    OverflowY,

    // Offset
    Offset,
    OffsetX,
    OffsetY,

    // Grid (integers)
    GridSizeRows,
    GridSizeColumns,
    GridGutterHorizontal,
    GridGutterVertical,
    GridRows,
    GridColumns,
    ColumnSpan,
    RowSpan,

    // Scrollbar (separate horizontal/vertical)
    ScrollbarGutter,
    ScrollbarSizeHorizontal,
    ScrollbarSizeVertical,
    ScrollbarCornerColor,
    // Scrollbar background colors
    ScrollbarBackgroundColor,
    ScrollbarBackgroundColorHover,
    ScrollbarBackgroundColorActive,
    // Scrollbar thumb colors
    ScrollbarColor,
    ScrollbarColorHover,
    ScrollbarColorActive,

    // Links
    LinkColor,
    LinkBackgroundColor,
    LinkStyle,
    LinkHoverColor,
    LinkHoverBackgroundColor,
    LinkHoverStyle,
    // Auto-links (URLs detected automatically)
    AutoLinkColor,
    AutoLinkBackgroundColor,
    AutoLinkStyle,
    AutoLinkHoverColor,
    AutoLinkHoverBackgroundColor,
    AutoLinkHoverStyle,

    // Special
    Hatch,
    Keyline,

    // Transitions
    Transition,
}

impl PropertyName {
    /// Get the CSS string representation
    pub fn as_str(&self) -> &'static str {
        match self {
            PropertyName::Width => "width",
            PropertyName::Height => "height",
            PropertyName::MinWidth => "min-width",
            // ... etc
        }
    }

    /// Parse from CSS string
    pub fn from_str(s: &str) -> Option<Self> {
        match s {
            "width" => Some(PropertyName::Width),
            "height" => Some(PropertyName::Height),
            "min-width" => Some(PropertyName::MinWidth),
            // ... etc
            _ => None,
        }
    }

    /// Get expected value type for this property
    pub fn value_type(&self) -> PropertyValueType {
        match self {
            PropertyName::Width | PropertyName::Height |
            PropertyName::MinWidth | PropertyName::MinHeight |
            PropertyName::MaxWidth | PropertyName::MaxHeight => PropertyValueType::Scalar,

            PropertyName::Margin | PropertyName::Padding => PropertyValueType::Spacing,

            PropertyName::Color | PropertyName::Background |
            PropertyName::Tint => PropertyValueType::Color,

            PropertyName::Display => PropertyValueType::Display,
            PropertyName::Visibility => PropertyValueType::Visibility,
            PropertyName::Opacity | PropertyName::TextOpacity => PropertyValueType::Float,

            // ... etc
        }
    }

    /// Does this property trigger layout recalculation?
    pub fn triggers_layout(&self) -> bool {
        matches!(self,
            PropertyName::Width | PropertyName::Height |
            PropertyName::MinWidth | PropertyName::MinHeight |
            PropertyName::MaxWidth | PropertyName::MaxHeight |
            PropertyName::Margin | PropertyName::MarginTop |
            PropertyName::MarginRight | PropertyName::MarginBottom |
            PropertyName::MarginLeft | PropertyName::Padding |
            PropertyName::PaddingTop | PropertyName::PaddingRight |
            PropertyName::PaddingBottom | PropertyName::PaddingLeft |
            PropertyName::Display | PropertyName::Dock |
            PropertyName::Layout | PropertyName::GridColumns |
            PropertyName::GridRows | PropertyName::BoxSizing
        )
    }

    /// Is this property animatable?
    pub fn is_animatable(&self) -> bool {
        matches!(self,
            PropertyName::Width | PropertyName::Height |
            PropertyName::Color | PropertyName::Background |
            PropertyName::Opacity | PropertyName::Offset |
            PropertyName::Margin | PropertyName::Padding
        )
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum PropertyValueType {
    Scalar,
    Spacing,
    Color,
    Display,
    Visibility,
    Overflow,
    TextAlign,
    TextStyle,
    Layout,
    Dock,
    Border,
    BorderEdge,
    Integer,
    Float,
    Bool,
    String,
    NameList,
    ScalarList,
    Transition,
    Hatch,
    Keyline,
}
```

## Parsing and Validation

```rust
impl StyleValue {
    /// Parse a value for a specific property
    pub fn parse(property: PropertyName, input: &str) -> Result<Self, StyleValueError> {
        let input = input.trim();

        // Handle universal keywords
        match input {
            "auto" => return Ok(StyleValue::Auto),
            "none" => return Ok(StyleValue::None),
            "initial" => return Ok(StyleValue::Initial),
            _ => {}
        }

        match property.value_type() {
            PropertyValueType::Scalar => {
                let scalar = Scalar::parse(input)?;
                Ok(StyleValue::Scalar(scalar))
            }
            PropertyValueType::Spacing => {
                let spacing = Spacing::parse(input)?;
                Ok(StyleValue::Spacing(spacing))
            }
            PropertyValueType::Color => {
                let color = Color::parse(input)?;
                Ok(StyleValue::Color(color))
            }
            // ... etc
        }
    }
}

#[derive(Debug, Clone)]
pub enum StyleValueError {
    InvalidValue(String),
    InvalidUnit(String),
    InvalidColor(String),
    InvalidStyleFlag(String),
    OutOfRange { value: f64, min: f64, max: f64 },
    UnitNotAllowed { unit: Unit, allowed: Vec<Unit> },
}
```

## Animatable Trait

```rust
/// Trait for values that can be interpolated during animations
pub trait Animatable: Clone {
    /// Blend between self and destination by factor (0.0 to 1.0)
    fn blend(&self, destination: &Self, factor: f64) -> Self;

    /// Get distance to destination (for speed-based animations)
    fn distance_to(&self, destination: &Self) -> f64;
}

impl Animatable for Color {
    fn blend(&self, dest: &Self, factor: f64) -> Self {
        Color {
            r: lerp(self.r, dest.r, factor),
            g: lerp(self.g, dest.g, factor),
            b: lerp(self.b, dest.b, factor),
            a: lerp(self.a, dest.a, factor),
        }
    }

    fn distance_to(&self, dest: &Self) -> f64 {
        // Euclidean distance in RGB space
        let dr = (dest.r - self.r) as f64;
        let dg = (dest.g - self.g) as f64;
        let db = (dest.b - self.b) as f64;
        (dr * dr + dg * dg + db * db).sqrt()
    }
}

impl Animatable for Scalar {
    fn blend(&self, dest: &Self, factor: f64) -> Self {
        // Can only blend same units
        if self.unit != dest.unit {
            return dest.clone();
        }
        Scalar {
            value: lerp(self.value, dest.value, factor),
            unit: self.unit,
        }
    }

    fn distance_to(&self, dest: &Self) -> f64 {
        (dest.value - self.value).abs()
    }
}

impl Animatable for Spacing {
    fn blend(&self, dest: &Self, factor: f64) -> Self {
        Spacing {
            top: self.top.blend(&dest.top, factor),
            right: self.right.blend(&dest.right, factor),
            bottom: self.bottom.blend(&dest.bottom, factor),
            left: self.left.blend(&dest.left, factor),
        }
    }

    fn distance_to(&self, dest: &Self) -> f64 {
        // Manhattan distance
        self.top.distance_to(&dest.top) +
        self.right.distance_to(&dest.right) +
        self.bottom.distance_to(&dest.bottom) +
        self.left.distance_to(&dest.left)
    }
}

fn lerp(a: f64, b: f64, t: f64) -> f64 {
    a + (b - a) * t
}
```

## Refresh Semantics

```rust
/// Flags indicating what needs refreshing after style change
#[derive(Debug, Clone, Copy, Default)]
pub struct RefreshFlags {
    pub layout: bool,      // Recalculate widget positions
    pub repaint: bool,     // Trigger repaint
    pub children: bool,    // Refresh child widgets
    pub parent: bool,      // Refresh parent widget
    pub scrollbars: bool,  // Refresh scrollbar widgets
}

impl PropertyName {
    pub fn refresh_flags(&self) -> RefreshFlags {
        match self {
            // Layout-affecting properties
            PropertyName::Width | PropertyName::Height |
            PropertyName::MinWidth | PropertyName::MinHeight |
            PropertyName::MaxWidth | PropertyName::MaxHeight |
            PropertyName::Margin | PropertyName::Padding |
            PropertyName::Display | PropertyName::Dock |
            PropertyName::Layout | PropertyName::BoxSizing => {
                RefreshFlags { layout: true, repaint: true, ..Default::default() }
            }

            // Overflow affects scrollbars
            PropertyName::Overflow | PropertyName::OverflowX |
            PropertyName::OverflowY => {
                RefreshFlags { repaint: true, scrollbars: true, ..Default::default() }
            }

            // Scrollbar colors
            PropertyName::ScrollbarColor | PropertyName::ScrollbarBackgroundColor => {
                RefreshFlags { scrollbars: true, ..Default::default() }
            }

            // Visual-only properties
            _ => RefreshFlags { repaint: true, ..Default::default() }
        }
    }
}
```

## Integration with Styles

```rust
impl Styles {
    /// Set a property value with validation
    pub fn set(&mut self, property: PropertyName, value: StyleValue) -> Result<bool, StyleValueError> {
        // Validate value type matches property
        let expected = property.value_type();
        if !value.matches_type(expected) {
            return Err(StyleValueError::InvalidValue(
                format!("Expected {:?} for {:?}", expected, property)
            ));
        }

        // Store in rules map
        let changed = self.rules.insert(property, value);
        Ok(changed)
    }

    /// Get a property value with default fallback
    pub fn get(&self, property: PropertyName) -> StyleValue {
        self.rules.get(&property)
            .cloned()
            .unwrap_or_else(|| self.default_value(property))
    }

    /// Clear a property (revert to default)
    pub fn clear(&mut self, property: PropertyName) {
        self.rules.remove(&property);
    }
}
```
