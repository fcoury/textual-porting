# CSS Property Parity Audit: Python Textual vs Rust textual-rs

## Executive Summary

This audit compares CSS property support between Python Textual and Rust textual-rs.
Significant gaps exist in rendering-related properties that affect visual parity.

## CSS Property Comparison

### Color Properties

| Property | Python | Rust Parsed | Rust Computed | Rust Rendered | Notes |
|----------|--------|-------------|---------------|---------------|-------|
| `color` | ✅ | ✅ | ✅ | ✅ | Working |
| `background` | ✅ | ✅ | ✅ | ✅ | Working |
| `tint` | ✅ | ❌ | ❌ | ❌ | **GAP** - Overlay color blending |
| `background-tint` | ✅ | ❌ | ❌ | ❌ | **GAP** - Background tint overlay |
| `text-opacity` | ✅ | ❌ | ❌ | ❌ | **GAP** - Listed as animatable but not computed |
| `opacity` | ✅ | ✅ | ✅ | ⚠️ | Rendered approximately (DIM/HIDDEN), not true alpha |

### Layout Dimension Properties

| Property | Python | Rust Parsed | Rust Computed | Rust Rendered | Notes |
|----------|--------|-------------|---------------|---------------|-------|
| `width` | ✅ | ✅ | ✅ | ⚠️ | Computed but not used by layout |
| `height` | ✅ | ✅ | ✅ | ⚠️ | Computed but not used by layout |
| `min-width` | ✅ | ✅ | ✅ | ⚠️ | Computed but not used by layout |
| `min-height` | ✅ | ✅ | ✅ | ⚠️ | Computed but not used by layout |
| `max-width` | ✅ | ✅ | ✅ | ⚠️ | Computed but not used by layout |
| `max-height` | ✅ | ✅ | ✅ | ⚠️ | Computed but not used by layout |

### Box Model Properties

| Property | Python | Rust Parsed | Rust Computed | Rust Rendered | Notes |
|----------|--------|-------------|---------------|---------------|-------|
| `margin` | ✅ | ✅ | ✅ | ⚠️ | Computed but not used by layout |
| `padding` | ✅ | ✅ | ✅ | ⚠️ | Computed but not used by layout |
| `border` | ✅ | ✅ | ✅ | ⚠️ | Computed, needs render integration |
| `border-top` | ✅ | ✅ | ❌ | ❌ | **GAP** - Parsed but not carried into ComputedStyle |
| `border-bottom` | ✅ | ✅ | ❌ | ❌ | **GAP** - Parsed but not carried into ComputedStyle |
| `border-left` | ✅ | ✅ | ❌ | ❌ | **GAP** - Parsed but not carried into ComputedStyle |
| `border-right` | ✅ | ✅ | ❌ | ❌ | **GAP** - Parsed but not carried into ComputedStyle |
| `box-sizing` | ✅ | ❌ | ❌ | ❌ | **GAP** - Not implemented |

### Border Styles

| Style | Python | Rust Parsed | Rendered | Notes |
|-------|--------|-------------|----------|-------|
| `solid` | ✅ | ✅ | ⚠️ | Needs render integration |
| `round` | ✅ | ✅ | ⚠️ | Needs render integration |
| `double` | ✅ | ✅ | ⚠️ | Needs render integration |
| `dashed` | ✅ | ✅ | ⚠️ | Needs render integration |
| `heavy` | ✅ | ✅ | ⚠️ | Needs render integration |
| `tall` | ✅ | ✅ | ⚠️ | **IMPORTANT** - Used heavily in Button |
| `block` | ✅ | ❌ | ❌ | **GAP** - Not parsed, used in Python Button |
| `wide` | ✅ | ✅ | ⚠️ | Needs render integration |
| `inner` | ✅ | ✅ | ⚠️ | Needs render integration |
| `outer` | ✅ | ✅ | ⚠️ | Needs render integration |

### Text Properties

| Property | Python | Rust Parsed | Rust Computed | Rust Rendered | Notes |
|----------|--------|-------------|---------------|---------------|-------|
| `text-align` | ✅ | ✅ | ✅ | ⚠️ | Computed but not used in rendering |
| `text-style` | ✅ | ✅ | ✅ | ✅ | Working (bold, italic, etc.) |
| `content-align` | ✅ | ❌ | ❌ | ❌ | **GAP** - Vertical + horizontal alignment |
| `content-align-horizontal` | ✅ | ❌ | ❌ | ❌ | **GAP** |
| `content-align-vertical` | ✅ | ❌ | ❌ | ❌ | **GAP** |
| `line-pad` | ✅ | ❌ | ❌ | ❌ | **GAP** - Horizontal padding per line |

### Display Properties

| Property | Python | Rust Parsed | Rust Computed | Rust Rendered | Notes |
|----------|--------|-------------|---------------|---------------|-------|
| `display` | ✅ | ✅ | ✅ | ⚠️ | Computed but not used |
| `visibility` | ✅ | ✅ | ✅ | ⚠️ | Computed but not used |
| `overflow` | ✅ | ✅ | ✅ | ⚠️ | Computed but not used |
| `overflow-x` | ✅ | ✅ | ✅ | ⚠️ | Computed but not used |
| `overflow-y` | ✅ | ✅ | ✅ | ⚠️ | Computed but not used |

### Grid Properties

| Property | Python | Rust Parsed | Rust Computed | Rust Rendered | Notes |
|----------|--------|-------------|---------------|---------------|-------|
| `grid-size` | ✅ | ❌ | ❌ | ❌ | **GAP** - Grid layout exists, CSS wiring missing |
| `grid-columns` | ✅ | ❌ | ❌ | ❌ | **GAP** - Grid layout exists, CSS wiring missing |
| `grid-rows` | ✅ | ❌ | ❌ | ❌ | **GAP** - Grid layout exists, CSS wiring missing |
| `grid-gutter` | ✅ | ❌ | ❌ | ❌ | **GAP** - Grid layout exists, CSS wiring missing |
| `column-span` | ✅ | ❌ | ❌ | ❌ | **GAP** - Grid layout exists, CSS wiring missing |
| `row-span` | ✅ | ❌ | ❌ | ❌ | **GAP** - Grid layout exists, CSS wiring missing |

### Animation/Transition Properties

| Property | Python | Rust Parsed | Rust Computed | Rust Rendered | Notes |
|----------|--------|-------------|---------------|---------------|-------|
| `animation` | ✅ | ✅ | ✅ | ⚠️ | Infrastructure exists |
| `transition` | ✅ | ✅ | ✅ | ⚠️ | Infrastructure exists |

## Selector Parity

### Selector Types

| Selector | Python | Rust | Notes |
|----------|--------|------|-------|
| Type (`Button`) | ✅ | ✅ | Working |
| Class (`.class`) | ✅ | ✅ | Working |
| ID (`#id`) | ✅ | ✅ | Working |
| Pseudo-class (`:focus`) | ✅ | ❌ | **GAP** - Uses `.-focused` class workaround (semantic mismatch) |
| Pseudo-class (`:hover`) | ✅ | ❌ | **GAP** - No equivalent workaround |
| Pseudo-class (`:disabled`) | ✅ | ❌ | **GAP** - Uses `.-disabled` class workaround (semantic mismatch) |
| Pseudo-class (`:active`) | ✅ | ❌ | **GAP** - Uses `.-active` class, but no true pseudo-class |
| Pseudo-class (`:inline`) | ✅ | ❌ | **GAP** |
| Nested (`&:focus`) | ✅ | ❌ | **GAP** - Python supports SCSS-like nesting |
| Combinators (` `, `>`) | ✅ | ⚠️ | Descendant + child parsing exists; matching parity unverified |

## Theme Variable Comparison

### Python textual-dark Theme
```python
Theme(
    name="textual-dark",
    primary="#0178D4",
    secondary="#004578",
    accent="#ffa62b",
    warning="#ffa62b",
    error="#ba3c5b",
    success="#4EBF71",
    foreground="#e0e0e0",
)
```

### Rust ThemeRegistry (textual-dark)
```rust
Theme {
    name: "textual-dark",
    primary: "#0178D4",   // ✅ Match
    secondary: "#004578", // ✅ Match
    accent: "#ffa62b",    // ✅ Match
    warning: "#ffa62b",   // ✅ Match
    error: "#ba3c5b",     // ✅ Match
    success: "#4EBF71",   // ✅ Match
    foreground: "#e0e0e0",// ✅ Match
    dark: true,
}
```

### Theme Variables Gap

Python uses these derived variables that aren't implemented in Rust:
- `$primary-muted` - Muted variant of primary
- `$surface-lighten-1` - Lightened surface (derived color)
- `$primary-darken-2` - Darkened primary (derived color)
- `$text-primary` - Primary text color on primary bg
- `$text-success` - Success text color
- `$panel` - Panel background
- `$button-focus-text-style` - Button focus text style

Additional theme gaps that affect perceived brightness:
- `Theme::luminosity_spread` parity with Python ColorSystem derivation
- `Theme::text_alpha` / default text opacity parity

## Button DEFAULT_CSS Comparison

### Python Button DEFAULT_CSS (excerpt)
```css
Button {
    width: auto;
    min-width: 16;
    height: auto;
    line-pad: 1;              /* GAP */
    text-align: center;
    content-align: center middle; /* GAP */

    &.-style-flat {           /* SCSS nesting - GAP */
        text-style: bold;
        color: auto 90%;
        background: $surface;
        border: block $surface; /* border: block - GAP */
        &:hover {
            background: $primary;
        }
        &:focus {
            text-style: $button-focus-text-style;
        }
        &.-active {
            tint: $background 30%; /* tint - GAP */
        }
    }
}
```

### Rust Button DEFAULT_CSS
```css
Button {
    background: $surface;
    min-width: 16;
    height: 3;              /* Fixed height, not auto */
}

Button.-focused {           /* Flat selector, no nesting */
    text-style: bold;
}

Button.-disabled {
    opacity: 0.6;
}
```

### Key Differences
1. No `content-align` or `line-pad` for vertical centering
2. No SCSS-like nested selectors (`&:hover`, `&:focus`)
3. No `:hover`, `:focus`, `:active` pseudo-classes
4. No `tint` for state feedback
5. No `border: block` style
6. Fixed height vs auto height

## Digits Widget Comparison

### Python Digits Patterns (rounded)
```
╭─╮  ╺┓  ╭─╮
│ │   │  ┌─┘
╰─╯  ╺┻╸ └─╸
 0    1    2
```

### Rust Digits Patterns (box drawing)
```
┌─┐    ┐  ┌─┐
│ │    │  ┌─┘
└─┘    ┘  └─┘
 0    1    2
```

### Key Difference
- Python uses rounded corners (`╭`, `╮`, `╰`, `╯`)
- Rust uses sharp corners (`┌`, `┐`, `└`, `┘`)
- Python has more glyph variants (A-F hex, currency, etc.)

## Examples Inventory

### Python Examples with TCSS
| Example | TCSS File | Key Features |
|---------|-----------|--------------|
| calculator.py | calculator.tcss | Grid layout, column-span, content-align |
| five_by_five.py | five_by_five.tcss | Game grid |
| dictionary.py | dictionary.tcss | Text display |
| code_browser.py | code_browser.tcss | Syntax highlighting |

### Rust Examples
| Example | Uses TCSS | Notes |
|---------|-----------|-------|
| calculator.rs | No | Manual layout, hardcoded styles |
| styled_app.rs | Yes | Uses StyleManager, RenderContext |
| theme_demo.rs | Yes | Theme switching demo |
| animation_demo.rs | Yes | Animation demo |

## Priority Fixes for Visual Parity

### Critical (P0) - Required for Calculator
1. `content-align: center middle` - Vertical/horizontal centering
2. `line-pad` - Horizontal text padding per line
3. Digits glyph patterns - Use rounded corners

### High (P1) - Required for Button Parity
1. `border: block` style
2. `tint` property for state feedback
3. Pseudo-class selectors (`:hover`, `:focus`, `:active`)
4. SCSS-like nested selectors

### Medium (P2) - General Parity
1. Grid CSS properties (`grid-size`, `grid-columns`, etc.)
2. `column-span`, `row-span`
3. `text-opacity`
4. `background-tint`
5. Theme variable derivation (`$surface-lighten-1`, etc.)

### Low (P3) - Nice to Have
1. Per-edge border colors in rendering
2. `box-sizing` property
3. Full Python theme parity (nord, gruvbox, etc.)
