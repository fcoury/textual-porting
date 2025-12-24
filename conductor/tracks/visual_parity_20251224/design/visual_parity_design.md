# Visual Parity Technical Design

## Overview

This document describes the design for achieving visual parity between Python Textual and Rust textual-rs. The focus is on the rendering pipeline integration points, not the CSS parsing layer (which is largely complete).

## 1. Style-to-Layout Mapping

### Current State
- `LayoutHints` struct exists in `geometry.rs` with: width, height, min_size, max_size, dock, display, alignment
- `ComputedStyle` has equivalent fields: width, height, min_width, min_height, max_width, max_height, display
- **Gap**: No automatic bridging from `ComputedStyle` to `LayoutHints`

### Design

#### 1.1 LayoutHints::from_computed_style()

Add a method to convert ComputedStyle dimensions to LayoutHints:

```rust
/// Context for resolving CSS units to absolute values.
pub struct ResolutionContext {
    /// Parent content box dimensions (for % units).
    pub parent: Size,
    /// Viewport dimensions (for vw/vh units).
    pub viewport: Size,
}

impl LayoutHints {
    /// Creates layout hints from computed CSS styles.
    ///
    /// Converts CSS dimension properties to layout constraints:
    /// - `width`/`height` → Constraint::Fixed or Constraint::Auto
    /// - `min-width`/`min-height` → min_size
    /// - `max-width`/`max-height` → max_size
    /// - `display: none` → Display::None
    ///
    /// Unit resolution:
    /// - `%` units resolve relative to parent content box
    /// - `vw`/`vh` units resolve relative to viewport
    /// - `w`/`h` units resolve relative to parent width/height
    pub fn from_computed_style(style: &ComputedStyle, ctx: &ResolutionContext) -> Self {
        let width = scalar_to_constraint(style.width, ctx.parent.width, ctx);
        let height = scalar_to_constraint(style.height, ctx.parent.height, ctx);

        let min_size = match (style.min_width, style.min_height) {
            (Some(w), Some(h)) => Some(Size::new(
                resolve_scalar(w, ctx.parent.width, ctx),
                resolve_scalar(h, ctx.parent.height, ctx),
            )),
            _ => None,
        };

        // Similar for max_size...

        Self {
            width,
            height,
            min_size,
            max_size,
            display: style.display.unwrap_or(Display::Block),
            ..Default::default()
        }
    }
}
```

#### 1.2 Scalar Resolution

```rust
fn scalar_to_constraint(
    scalar: Option<Scalar>,
    parent_dimension: u16,
    ctx: &ResolutionContext,
) -> Constraint {
    match scalar {
        None => Constraint::Auto,
        Some(s) if s.unit == Unit::Auto => Constraint::Auto,
        Some(s) => Constraint::Fixed(resolve_scalar(s, parent_dimension, ctx)),
    }
}

/// Resolves a CSS scalar to an absolute cell value.
///
/// Unit semantics:
/// - `%` → relative to parent content box dimension
/// - `w` → relative to parent width
/// - `h` → relative to parent height
/// - `vw` → relative to viewport width
/// - `vh` → relative to viewport height
/// - cells (no unit) → absolute value
fn resolve_scalar(scalar: Scalar, parent_dimension: u16, ctx: &ResolutionContext) -> u16 {
    match scalar.unit {
        Unit::Cells => scalar.value as u16,
        Unit::Percent => (scalar.value * parent_dimension as f64 / 100.0) as u16,
        Unit::Width => (scalar.value * ctx.parent.width as f64 / 100.0) as u16,
        Unit::Height => (scalar.value * ctx.parent.height as f64 / 100.0) as u16,
        Unit::ViewWidth => (scalar.value * ctx.viewport.width as f64 / 100.0) as u16,
        Unit::ViewHeight => (scalar.value * ctx.viewport.height as f64 / 100.0) as u16,
        Unit::Auto => 0, // Handled by Constraint::Auto
        Unit::Fraction => 0, // Handled separately in grid
    }
}
```

#### 1.3 Grid CSS Properties

Add to ComputedStyle:
```rust
pub struct ComputedStyle {
    // ... existing fields ...

    // Grid container properties
    pub grid_size_columns: Option<u16>,
    pub grid_size_rows: Option<u16>,
    pub grid_gutter_horizontal: Option<u16>,
    pub grid_gutter_vertical: Option<u16>,
    pub grid_columns: Option<Vec<Scalar>>,  // 1fr 1fr 2fr
    pub grid_rows: Option<Vec<Scalar>>,     // auto 1fr auto

    // Grid item properties
    pub column_span: Option<u16>,
    pub row_span: Option<u16>,
}
```

Conversion to GridConfig:
```rust
impl GridConfig {
    pub fn from_computed_style(style: &ComputedStyle) -> Option<Self> {
        // Only create if grid properties are set
        if style.grid_size_columns.is_none() && style.grid_columns.is_none() {
            return None;
        }

        let columns = match &style.grid_columns {
            Some(cols) => scalars_to_track_template(cols),
            None => GridTrackTemplate::repeat(
                style.grid_size_columns.unwrap_or(1) as usize,
                TrackSize::Fr(1),
            ),
        };

        // Similar for rows...

        Some(GridConfig {
            columns,
            rows,
            column_gap: style.grid_gutter_horizontal.unwrap_or(0),
            row_gap: style.grid_gutter_vertical.unwrap_or(0),
            placements: HashMap::new(),
        })
    }
}
```

## 2. Border Rendering Pipeline

### Current State
- `BorderKind` enum exists with most border types including `tall`
- `Border` struct has four `BorderEdge` fields (top, right, bottom, left)
- `ComputedStyle.border` is `Option<Border>`
- **Gap**: `block` border kind not yet parsed
- **Gap**: No rendering of borders from computed styles

### Design

#### 2.1 Add `block` Border Kind Parsing

The `BorderKind::Block` variant needs to be added and wired to the parser:

```rust
// In BorderKind::from_str() - add this match arm:
"block" => Some(BorderKind::Block),
```

#### 2.2 Border Character Maps

```rust
/// Returns the border characters for a given border kind.
pub fn border_chars(kind: BorderKind) -> BorderChars {
    match kind {
        BorderKind::Solid => BorderChars {
            top: '─', bottom: '─', left: '│', right: '│',
            top_left: '┌', top_right: '┐', bottom_left: '└', bottom_right: '┘',
        },
        BorderKind::Round => BorderChars {
            top: '─', bottom: '─', left: '│', right: '│',
            top_left: '╭', top_right: '╮', bottom_left: '╰', bottom_right: '╯',
        },
        BorderKind::Double => BorderChars {
            top: '═', bottom: '═', left: '║', right: '║',
            top_left: '╔', top_right: '╗', bottom_left: '╚', bottom_right: '╝',
        },
        BorderKind::Tall => BorderChars {
            // Tall uses half-block characters for top/bottom
            top: '▔', bottom: '▁', left: '│', right: '│',
            top_left: '▔', top_right: '▔', bottom_left: '▁', bottom_right: '▁',
        },
        BorderKind::Block => BorderChars {
            // Block uses full block for all edges
            top: '█', bottom: '█', left: '█', right: '█',
            top_left: '█', top_right: '█', bottom_left: '█', bottom_right: '█',
        },
        // ... other variants
    }
}
```

#### 2.3 Per-Edge Border Rendering

Add per-edge fields to ComputedStyle:
```rust
pub struct ComputedStyle {
    // ... existing border field ...
    pub border: Option<Border>,

    // Per-edge overrides (higher specificity)
    pub border_top: Option<BorderEdge>,
    pub border_right: Option<BorderEdge>,
    pub border_bottom: Option<BorderEdge>,
    pub border_left: Option<BorderEdge>,
}
```

Border rendering in widget paint:
```rust
fn render_borders(
    frame: &mut Frame,
    area: Rect,
    style: &ComputedStyle,
) {
    let border = style.effective_border();
    if border.is_none() {
        return;
    }

    // Render each edge independently
    if !border.top.is_none() {
        render_top_border(frame, area, &border.top);
    }
    if !border.bottom.is_none() {
        render_bottom_border(frame, area, &border.bottom);
    }
    // ... left and right
}
```

## 3. Content Alignment and Line-Pad

### Current State
- No `content-align` property
- No `line-pad` property
- Text is rendered at top-left by default

### Design

#### 3.1 Add Properties to ComputedStyle

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

pub struct ComputedStyle {
    // ... existing fields ...

    /// Horizontal content alignment within the widget.
    pub content_align_horizontal: Option<AlignHorizontal>,
    /// Vertical content alignment within the widget.
    pub content_align_vertical: Option<AlignVertical>,
    /// Padding added to left and right of text lines.
    pub line_pad: Option<u16>,
}
```

#### 3.2 CSS Parsing

```rust
// In declaration parser
"content-align" => {
    // Parse: "center middle" or "left top"
    let (h, v) = parse_content_align(value)?;
    declarations.push(Declaration::ContentAlignHorizontal(h));
    declarations.push(Declaration::ContentAlignVertical(v));
}
"content-align-horizontal" => {
    declarations.push(Declaration::ContentAlignHorizontal(
        AlignHorizontal::from_str(value)?
    ));
}
"content-align-vertical" => {
    declarations.push(Declaration::ContentAlignVertical(
        AlignVertical::from_str(value)?
    ));
}
"line-pad" => {
    declarations.push(Declaration::LinePad(parse_int(value)?));
}
```

#### 3.3 Rendering Integration

Content alignment is applied during rendering, not layout.

**Important**: Width measurement must use `unicode_width::UnicodeWidthStr` to correctly
handle box-drawing characters, emoji, and other multi-cell glyphs. Using `String::len()`
would cause alignment drift with non-ASCII content.

**Dependency Note**: The `unicode-width` crate is already in the dependency tree (via ratatui).
Implementation should either:
1. Re-export from ratatui if available, or
2. Add as a direct dependency (already transitively included, so no new download)

```rust
use unicode_width::UnicodeWidthStr;

/// Aligns content within a given area.
///
/// Uses unicode-width for correct cell-width measurement of box-drawing
/// characters and other multi-byte glyphs.
pub fn align_content(
    content: &[String],
    area: Rect,
    h_align: AlignHorizontal,
    v_align: AlignVertical,
) -> Rect {
    let content_height = content.len() as u16;
    // Use unicode-width for correct cell measurement
    let content_width = content
        .iter()
        .map(|s| s.width())
        .max()
        .unwrap_or(0) as u16;

    let x = match h_align {
        AlignHorizontal::Left => area.x,
        AlignHorizontal::Center => area.x + (area.width.saturating_sub(content_width)) / 2,
        AlignHorizontal::Right => area.x + area.width.saturating_sub(content_width),
    };

    let y = match v_align {
        AlignVertical::Top => area.y,
        AlignVertical::Middle => area.y + (area.height.saturating_sub(content_height)) / 2,
        AlignVertical::Bottom => area.y + area.height.saturating_sub(content_height),
    };

    Rect::new(x, y, content_width, content_height)
}
```

Line padding is applied to text content. Text is clipped to fit within the effective width:
```rust
use unicode_width::UnicodeWidthStr;

/// Applies line-pad to text and clips to fit within area.
fn apply_line_pad(text: &str, line_pad: u16, max_width: u16) -> String {
    let effective_width = max_width.saturating_sub(line_pad * 2) as usize;
    let pad = " ".repeat(line_pad as usize);

    // Clip text if it exceeds effective width
    let clipped = if text.width() > effective_width {
        // Truncate to fit (simplified; real impl should handle grapheme clusters)
        let mut width = 0;
        let truncated: String = text
            .chars()
            .take_while(|c| {
                width += c.to_string().width();
                width <= effective_width
            })
            .collect();
        truncated
    } else {
        text.to_string()
    };

    format!("{}{}{}", pad, clipped, pad)
}
```

## 4. Tint and Text-Opacity

### Current State
- `opacity` is parsed and computed but only approximated with DIM/HIDDEN
- No `tint` or `background-tint` properties
- No `text-opacity` property

### Design

#### 4.1 Add Properties to ComputedStyle

```rust
pub struct ComputedStyle {
    // ... existing fields ...

    /// Tint color to overlay on top of the widget.
    pub tint: Option<RgbaColor>,
    /// Tint color to blend with the background.
    pub background_tint: Option<RgbaColor>,
    /// Text opacity (0.0 - 1.0).
    pub text_opacity: Option<f64>,
}
```

#### 4.2 Color Blending

Tint application order (matching Python):
1. Render content with base colors
2. Apply `background-tint` to background
3. Apply `text-opacity` to foreground
4. Apply `tint` as final overlay

```rust
/// Blends a tint color onto a base color using alpha compositing.
/// The result is a solid RGB color (alpha is baked in).
fn apply_tint(base: &RgbaColor, tint: &RgbaColor) -> RgbaColor {
    // Alpha blending: result = tint * tint.a + base * (1 - tint.a)
    let alpha = tint.a as f64 / 255.0;
    RgbaColor {
        r: ((tint.r as f64 * alpha) + (base.r as f64 * (1.0 - alpha))) as u8,
        g: ((tint.g as f64 * alpha) + (base.g as f64 * (1.0 - alpha))) as u8,
        b: ((tint.b as f64 * alpha) + (base.b as f64 * (1.0 - alpha))) as u8,
        a: 255, // Result is fully opaque after blending
    }
}

/// Applies text opacity by darkening the color toward the background.
/// Returns modified RGB (not true alpha).
fn apply_text_opacity(color: &RgbaColor, opacity: f64, bg: &RgbaColor) -> RgbaColor {
    // Blend text color toward background based on opacity
    let inv_opacity = 1.0 - opacity;
    RgbaColor {
        r: ((color.r as f64 * opacity) + (bg.r as f64 * inv_opacity)) as u8,
        g: ((color.g as f64 * opacity) + (bg.g as f64 * inv_opacity)) as u8,
        b: ((color.b as f64 * opacity) + (bg.b as f64 * inv_opacity)) as u8,
        a: 255,
    }
}
```

#### 4.3 Terminal Rendering and ratatui Approximation

ratatui does not support true RGBA - all colors are solid RGB. The blending functions
above produce pre-composited RGB values. For additional visual feedback:

```rust
/// Converts blended colors to ratatui Style with modifier approximations.
fn style_with_opacity(
    fg: RgbaColor,
    bg: RgbaColor,
    text_opacity: f64,
) -> ratatui::style::Style {
    let mut style = Style::default()
        .fg(Color::Rgb(fg.r, fg.g, fg.b))
        .bg(Color::Rgb(bg.r, bg.g, bg.b));

    // Add modifiers for very low opacity as visual hint
    if text_opacity < 0.1 {
        style = style.add_modifier(Modifier::HIDDEN);
    } else if text_opacity < 0.5 {
        style = style.add_modifier(Modifier::DIM);
    }

    style
}
```

**Note**: The blending approach means `tint: red 50%` will produce a visually correct
result by pre-blending the red onto the background, even though ratatui doesn't
support alpha channels natively.

## 5. Pseudo-Class Selector Support

### Current State
- Uses class workarounds: `.-focused`, `.-disabled`, `.-active`
- No true pseudo-class selectors (`:focus`, `:hover`, `:disabled`)
- No SCSS-like nesting (`&:focus`)

### Design

#### 5.1 Selector Enhancement

Add pseudo-class to Selector AST:
```rust
#[derive(Debug, Clone, PartialEq)]
pub enum PseudoClass {
    Focus,
    Hover,
    Active,
    Disabled,
    Inline,
    // Container queries
    // FirstChild,
    // LastChild,
}

#[derive(Debug, Clone, Default)]
pub struct SimpleSelector {
    pub widget_type: Option<String>,
    pub id: Option<String>,
    pub classes: Vec<String>,
    pub pseudo_classes: Vec<PseudoClass>,  // NEW
}
```

#### 5.2 Selector Parsing

```rust
fn parse_pseudo_class(input: &str) -> IResult<&str, PseudoClass> {
    preceded(
        char(':'),
        alt((
            map(tag("focus"), |_| PseudoClass::Focus),
            map(tag("hover"), |_| PseudoClass::Hover),
            map(tag("active"), |_| PseudoClass::Active),
            map(tag("disabled"), |_| PseudoClass::Disabled),
            map(tag("inline"), |_| PseudoClass::Inline),
        )),
    )(input)
}
```

#### 5.3 Widget State → Pseudo-Class Mapping

```rust
impl Widget for Button {
    fn pseudo_classes(&self) -> Vec<PseudoClass> {
        let mut classes = Vec::new();
        if self.is_focused {
            classes.push(PseudoClass::Focus);
        }
        if self.is_hovered {
            classes.push(PseudoClass::Hover);
        }
        if self.is_pressed {
            classes.push(PseudoClass::Active);
        }
        if self.is_disabled {
            classes.push(PseudoClass::Disabled);
        }
        classes
    }
}
```

#### 5.4 Selector Matching

```rust
fn matches_pseudo_classes(
    selector: &SimpleSelector,
    widget_pseudo_classes: &[PseudoClass],
) -> bool {
    selector.pseudo_classes.iter().all(|required| {
        widget_pseudo_classes.contains(required)
    })
}
```

## 6. Digits Glyph Parity

### Current State
- Rust uses box-drawing characters (`┌─┐`)
- Python uses rounded corners (`╭─╮`)
- Python has more glyph variants (A-F hex, currency symbols)

### Design

#### 6.1 Update Digit Patterns

Replace in `calculator.rs` or create `Digits` widget:

```rust
/// 3x3 digit patterns matching Python Textual (rounded corners).
const DIGIT_PATTERNS: [&[&str]; 10] = [
    &["╭─╮", "│ │", "╰─╯"], // 0
    &["  ╷", "  │", "  ╵"], // 1
    &["╭─╮", "┌─╯", "╰─╸"], // 2
    &["╭─╮", " ─┤", "╰─╯"], // 3
    &["╷ ╷", "╰─┤", "  ╵"], // 4
    &["╭─╸", "╰─╮", "╰─╯"], // 5
    &["╭─╸", "├─╮", "╰─╯"], // 6
    &["╭─╮", "  │", "  ╵"], // 7
    &["╭─╮", "├─┤", "╰─╯"], // 8
    &["╭─╮", "╰─┤", "╰─╯"], // 9
];

const MINUS_PATTERN: &[&str] = &["   ", "───", "   "];
const PERIOD_PATTERN: &[&str] = &[" ", " ", "●"];
```

#### 6.2 Optional: Create Digits Widget

```rust
pub struct Digits {
    value: String,
    classes: HashSet<String>,
}

impl Digits {
    pub const DEFAULT_CSS: &'static str = r#"
Digits {
    width: 1fr;
    height: auto;
    text-align: left;
}
"#;

    pub fn new(value: impl Into<String>) -> Self {
        Self {
            value: value.into(),
            classes: HashSet::new(),
        }
    }

    pub fn set_value(&mut self, value: impl Into<String>) {
        self.value = value.into();
    }
}
```

## 7. Integration Points

### 7.1 RenderContext Enhancement

```rust
impl<'a> RenderContext<'a> {
    /// Get computed style with all properties resolved.
    pub fn computed_style(&self, id: WidgetId, widget: &impl Widget) -> ComputedStyle {
        self.style_manager.get_style(id, widget)
    }

    /// Get layout hints derived from computed style.
    ///
    /// Requires ResolutionContext for proper unit resolution:
    /// - `%` units resolve relative to parent content box
    /// - `vw`/`vh` units resolve relative to viewport
    pub fn layout_hints(
        &self,
        id: WidgetId,
        widget: &impl Widget,
        ctx: &ResolutionContext,
    ) -> LayoutHints {
        let style = self.computed_style(id, widget);
        LayoutHints::from_computed_style(&style, ctx)
    }

    /// Get grid config if widget has grid properties.
    pub fn grid_config(&self, id: WidgetId, widget: &impl Widget) -> Option<GridConfig> {
        let style = self.computed_style(id, widget);
        GridConfig::from_computed_style(&style)
    }
}
```

### 7.2 Widget Rendering

Widgets should implement `render_with_context()` from the Widget trait for styled rendering.
The signature matches the existing trait definition:

```rust
impl Widget for Button {
    /// Renders the button using computed styles from RenderContext.
    ///
    /// This method is called by the render loop when a RenderContext is available.
    /// Falls back to render() if no context is provided.
    fn render_with_context(
        &self,
        id: WidgetId,
        area: Rect,
        frame: &mut Frame,
        context: &RenderContext,
    ) {
        let style = context.computed_style(id, self);

        // Apply borders
        render_borders(frame, area, &style);

        // Calculate content area (inside borders/padding)
        let content_area = style.content_area(area);

        // Apply content alignment
        let aligned_area = align_content(
            &[self.label.clone()],
            content_area,
            style.content_align_horizontal.unwrap_or_default(),
            style.content_align_vertical.unwrap_or_default(),
        );

        // Apply line-pad to text with clipping
        let text = apply_line_pad(
            &self.label,
            style.line_pad.unwrap_or(0),
            aligned_area.width,
        );

        // Get effective colors
        let base_fg = style.effective_color();
        let base_bg = style.effective_background();

        // Apply background-tint
        let bg = match &style.background_tint {
            Some(tint) => apply_tint(&base_bg, tint),
            None => base_bg,
        };

        // Apply text-opacity (blend fg toward bg)
        let fg = apply_text_opacity(
            &base_fg,
            style.text_opacity.unwrap_or(1.0),
            &bg,
        );

        // Apply tint overlay
        let final_bg = match &style.tint {
            Some(tint) => apply_tint(&bg, tint),
            None => bg,
        };

        // Convert to ratatui style with modifier approximations
        let ratatui_style = style_with_opacity(fg, final_bg, style.text_opacity.unwrap_or(1.0));
        let paragraph = Paragraph::new(text).style(ratatui_style);
        frame.render_widget(paragraph, aligned_area);
    }

    /// Default render without styling context.
    fn render(&self, frame: &mut Frame, area: Rect) {
        // Fallback: render with hardcoded styles (existing behavior)
        // ...
    }
}
```

## 8. Implementation Priority

1. **P0 - Calculator Parity**
   - `content-align` parsing and rendering
   - `line-pad` parsing and rendering
   - Digits glyph patterns (rounded corners)

2. **P1 - Button Parity**
   - `border: block` parsing
   - Pseudo-class selectors (`:focus`, `:hover`, `:active`)
   - `tint` property

3. **P2 - Layout Integration**
   - Style-to-layout mapping for dimensions
   - Grid CSS properties wiring
   - `column-span`, `row-span`

4. **P3 - Full Parity**
   - `text-opacity`, `background-tint`
   - SCSS-like nested selectors
   - Theme variable derivation
