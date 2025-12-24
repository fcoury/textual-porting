# ComputedStyle to ratatui::Style Mapping

## Overview

This document maps `ComputedStyle` fields from our TCSS system to the capabilities of `ratatui::Style`. The conversion is a **narrow adapter** - it only handles paint-layer properties (colors and text effects), not layout properties.

## ratatui::Style Structure

```rust
pub struct Style {
    pub fg: Option<Color>,          // Foreground color
    pub bg: Option<Color>,          // Background color
    pub underline_color: Option<Color>,
    pub add_modifier: Modifier,     // Text effects to add
    pub sub_modifier: Modifier,     // Text effects to remove
}
```

### Available Modifiers
- `Modifier::BOLD`
- `Modifier::DIM`
- `Modifier::ITALIC`
- `Modifier::UNDERLINED`
- `Modifier::SLOW_BLINK`
- `Modifier::RAPID_BLINK`
- `Modifier::REVERSED`
- `Modifier::HIDDEN`
- `Modifier::CROSSED_OUT`

## ComputedStyle Fields by Category

### 1. Color Properties → ratatui::Style

| ComputedStyle Field | ratatui Mapping | Notes |
|---------------------|-----------------|-------|
| `color: Option<Color>` | `fg: Option<Color>` | Direct mapping after RgbaColor → ratatui::Color conversion |
| `background: Option<Color>` | `bg: Option<Color>` | Direct mapping after color conversion |
| `auto_color: bool` | N/A | Resolved before reaching ratatui (by compute_style_resolved) |
| `auto_background: bool` | N/A | Resolved before reaching ratatui |

### 2. Text Properties → ratatui::Modifier

| ComputedStyle Field | ratatui Mapping | Notes |
|---------------------|-----------------|-------|
| `text_style.bold` | `Modifier::BOLD` | Direct |
| `text_style.dim` | `Modifier::DIM` | Direct |
| `text_style.italic` | `Modifier::ITALIC` | Direct |
| `text_style.underline` | `Modifier::UNDERLINED` | Direct |
| `text_style.underline2` | `Modifier::UNDERLINED` | Double underline not supported, use single |
| `text_style.blink` | `Modifier::SLOW_BLINK` | Direct |
| `text_style.blink2` | `Modifier::RAPID_BLINK` | Direct |
| `text_style.reverse` | `Modifier::REVERSED` | Direct |
| `text_style.strike` | `Modifier::CROSSED_OUT` | Direct |
| `text_style.overline` | **Not supported** | ratatui doesn't have overline |

### 3. Display Properties → ratatui::Style

| ComputedStyle Field | ratatui Mapping | Notes |
|---------------------|-----------------|-------|
| `opacity` | `Modifier::DIM` if < 1.0 | Approximation only - full opacity control not available |
| `visibility` | **Not applicable** | Handled by render pipeline, not style |
| `display` | **Not applicable** | Handled by layout system |

### 4. Layout Properties (NOT converted)

These are handled by the layout system, not the paint layer:

- `width`, `height`, `min_width`, `min_height`, `max_width`, `max_height`
- `margin`, `padding`, `border`
- `text_align` (affects content positioning, not style)
- `overflow_x`, `overflow_y`, `scrollbar_gutter`
- All scrollbar-* properties

### 5. Animation/Transition Properties (NOT converted)

These drive the Animator, not the paint layer:

- `animation`, `animation_name`, `animation_duration`, etc.
- `transition`, `transition_property`, `transition_duration`, etc.

## Color Type Conversion

### Our Color Type (RgbaColor)

```rust
pub struct RgbaColor {
    pub r: u8,
    pub g: u8,
    pub b: u8,
    pub a: f32,
    pub ansi: Option<String>,
    pub auto: bool,
}
```

### ratatui Color Type

```rust
pub enum Color {
    Reset,
    Black, Red, Green, Yellow, Blue, Magenta, Cyan, Gray,
    DarkGray, LightRed, LightGreen, LightYellow, LightBlue, LightMagenta, LightCyan, White,
    Rgb(u8, u8, u8),
    Indexed(u8),
}
```

### Conversion Strategy

```rust
impl From<&RgbaColor> for ratatui::style::Color {
    fn from(color: &RgbaColor) -> Self {
        // 1. Check for ANSI named color.
        // RgbaColor::ansi stores names with the "ansi_" prefix (e.g., "ansi_red").
        if let Some(ansi_name) = &color.ansi {
            match ansi_name.as_str() {
                "ansi_default" => return ratatui::style::Color::Reset,
                "ansi_black" => return ratatui::style::Color::Black,
                "ansi_red" => return ratatui::style::Color::Red,
                "ansi_green" => return ratatui::style::Color::Green,
                "ansi_yellow" => return ratatui::style::Color::Yellow,
                "ansi_blue" => return ratatui::style::Color::Blue,
                "ansi_magenta" => return ratatui::style::Color::Magenta,
                "ansi_cyan" => return ratatui::style::Color::Cyan,
                "ansi_white" => return ratatui::style::Color::White,
                _ => {}
            }
        }

        // 2. Convert RGB to ratatui Rgb
        // Note: alpha is lost - ratatui doesn't support alpha
        ratatui::style::Color::Rgb(color.r, color.g, color.b)
    }
}
```

**Limitations**:
- Alpha channel is ignored (ratatui doesn't support transparency)
- Auto colors must be resolved before conversion

## Proposed to_ratatui_style() Implementation

```rust
impl ComputedStyle {
    /// Convert color and text properties to ratatui::Style for rendering.
    ///
    /// Layout properties (width, height, margin, padding, border, etc.)
    /// are NOT included - those are handled by the layout system.
    pub fn to_ratatui_style(&self) -> ratatui::style::Style {
        let mut style = ratatui::style::Style::default();

        // Convert colors
        if let Some(color) = &self.color {
            style = style.fg(color.into());
        }
        if let Some(bg) = &self.background {
            style = style.bg(bg.into());
        }

        // Convert text style to modifiers
        let mut modifiers = ratatui::style::Modifier::empty();

        if let Some(ts) = &self.text_style {
            if ts.bold { modifiers |= ratatui::style::Modifier::BOLD; }
            if ts.dim { modifiers |= ratatui::style::Modifier::DIM; }
            if ts.italic { modifiers |= ratatui::style::Modifier::ITALIC; }
            if ts.underline || ts.underline2 { modifiers |= ratatui::style::Modifier::UNDERLINED; }
            if ts.blink { modifiers |= ratatui::style::Modifier::SLOW_BLINK; }
            if ts.blink2 { modifiers |= ratatui::style::Modifier::RAPID_BLINK; }
            if ts.reverse { modifiers |= ratatui::style::Modifier::REVERSED; }
            if ts.strike { modifiers |= ratatui::style::Modifier::CROSSED_OUT; }
            // Note: ts.overline not supported by ratatui
        }

        // Approximate opacity with DIM modifier
        if let Some(opacity) = self.opacity {
            if opacity < 1.0 && opacity > 0.0 {
                modifiers |= ratatui::style::Modifier::DIM;
            } else if opacity == 0.0 {
                modifiers |= ratatui::style::Modifier::HIDDEN;
            }
        }

        if !modifiers.is_empty() {
            style = style.add_modifier(modifiers);
        }

        style
    }
}
```

## Property Support Summary

| Category | Properties | ratatui Support |
|----------|------------|-----------------|
| Colors | color, background | ✅ Full (minus alpha) |
| Text Effects | bold, dim, italic, underline, blink, reverse, strike | ✅ Full |
| Text Effects | underline2, overline | ⚠️ Partial/None |
| Opacity | opacity | ⚠️ Approximated via DIM/HIDDEN |
| Layout | All dimension/box properties | ❌ Not applicable |
| Animation | All animation/transition properties | ❌ Not applicable |
| Scrollbar | All scrollbar properties | ❌ Not applicable |

## Integration Notes

### Order of Operations

1. `compute_style_resolved()` - resolves theme variables and auto colors
2. Animator applies animated property values
3. `to_ratatui_style()` converts to ratatui format
4. Widget uses ratatui Style for rendering

### What to_ratatui_style() Does NOT Do

- Resolve auto colors (must be done by compute_style_resolved first)
- Handle visibility (render pipeline decides whether to render)
- Apply layout (layout system handles positioning)
- Manage focus/hover states (pseudo-classes determine which rules apply)

### Testing Strategy

```rust
#[test]
fn test_to_ratatui_style_colors() {
    let mut style = ComputedStyle::new();
    style.color = Some(RgbaColor::rgb(255, 0, 0));
    style.background = Some(RgbaColor::rgb(0, 0, 255));

    let ratatui = style.to_ratatui_style();
    assert_eq!(ratatui.fg, Some(ratatui::style::Color::Rgb(255, 0, 0)));
    assert_eq!(ratatui.bg, Some(ratatui::style::Color::Rgb(0, 0, 255)));
}

#[test]
fn test_to_ratatui_style_text_modifiers() {
    let mut style = ComputedStyle::new();
    style.text_style = Some(TextStyle {
        bold: true,
        italic: true,
        ..TextStyle::NONE
    });

    let ratatui = style.to_ratatui_style();
    assert!(ratatui.add_modifier.contains(ratatui::style::Modifier::BOLD));
    assert!(ratatui.add_modifier.contains(ratatui::style::Modifier::ITALIC));
}

#[test]
fn test_to_ratatui_style_opacity_dim() {
    let mut style = ComputedStyle::new();
    style.opacity = Some(0.5);

    let ratatui = style.to_ratatui_style();
    assert!(ratatui.add_modifier.contains(ratatui::style::Modifier::DIM));
}
```

## Sources

- [ratatui::style module](https://docs.rs/ratatui/latest/ratatui/style/)
- [ratatui::style::Modifier](https://docs.rs/ratatui/latest/ratatui/style/struct.Modifier.html)
