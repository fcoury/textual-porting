# DEFAULT_CSS Pattern Design

## Overview

Python Textual widgets define default styles via `DEFAULT_CSS` class attributes. This document defines how Rust widgets provide equivalent default styling while integrating with the TCSS (Textual CSS) system.

## Python Textual Pattern

```python
class Button(Widget):
    DEFAULT_CSS = """
    Button {
        background: $primary;
        color: $text-primary;
        border: tall $background-lighten-1;
        height: 3;
        min-width: 16;
        &:hover { background: $primary-darken-1; }
        &:focus { text-style: bold; }
        &.-primary { background: $primary; }
    }
    """
```

## Rust Design

### Option 1: Const String (Recommended)

```rust
impl Button {
    /// Default CSS for Button widget.
    /// Registered with the style system during widget registration.
    pub const DEFAULT_CSS: &'static str = r#"
        Button {
            background: $primary;
            color: $text-primary;
            border: tall $background-lighten-1;
            height: 3;
            min-width: 16;
        }
        Button:hover {
            background: $primary-darken-1;
        }
        Button:focus {
            text-style: bold;
        }
        Button.-primary {
            background: $primary;
        }
    "#;
}
```

### Registration Trait

```rust
/// Trait for widgets that provide default CSS.
pub trait DefaultCss {
    /// Returns the default CSS for this widget type.
    /// This CSS is loaded at lowest priority, allowing user overrides.
    fn default_css() -> &'static str;

    /// Returns the widget type name for CSS selector matching.
    fn css_type_name() -> &'static str;
}

impl DefaultCss for Button {
    fn default_css() -> &'static str {
        Self::DEFAULT_CSS
    }

    fn css_type_name() -> &'static str {
        "Button"
    }
}
```

### Style System Integration

```rust
impl StyleSystem {
    /// Register a widget type's default CSS.
    pub fn register_widget_defaults<W: DefaultCss>(&mut self) {
        let css = W::default_css();
        let type_name = W::css_type_name();

        // Parse and add to default stylesheet (lowest priority)
        if let Ok(rules) = self.parse_css(css) {
            self.default_rules.extend(rules);
        }
    }

    /// Called during app initialization to register all widget defaults.
    pub fn register_all_defaults(&mut self) {
        self.register_widget_defaults::<Static>();
        self.register_widget_defaults::<Label>();
        self.register_widget_defaults::<Button>();
        self.register_widget_defaults::<Input>();
        self.register_widget_defaults::<Header>();
        self.register_widget_defaults::<Footer>();
        self.register_widget_defaults::<Checkbox>();
        self.register_widget_defaults::<RadioButton>();
        self.register_widget_defaults::<Switch>();
        self.register_widget_defaults::<Select>();
        self.register_widget_defaults::<OptionList>();
        self.register_widget_defaults::<SelectionList>();
        self.register_widget_defaults::<ProgressBar>();
        self.register_widget_defaults::<LoadingIndicator>();
        self.register_widget_defaults::<Rule>();
        self.register_widget_defaults::<Link>();
    }
}
```

## DEFAULT_CSS Catalog

### Static
```css
Static {
    height: auto;
}
```

### Label
```css
Label {
    width: auto;
    height: auto;
    min-height: 1;
}
Label.success {
    color: $text-success;
    background: $success-muted;
}
Label.error {
    color: $text-error;
    background: $error-muted;
}
Label.warning {
    color: $text-warning;
    background: $warning-muted;
}
Label.primary {
    color: $text-primary;
    background: $primary-muted;
}
Label.secondary {
    color: $text-secondary;
    background: $secondary-muted;
}
Label.accent {
    color: $text-accent;
    background: $accent-muted;
}
```

### Button
```css
/* Base Button styles (exact Python Textual parity) */
Button {
    width: auto;
    min-width: 16;
    height: auto;
    line-pad: 1;
    text-align: center;
    content-align: center middle;
}

/* ===== FLAT STYLE (-style-flat) ===== */
Button.-style-flat {
    text-style: bold;
    color: auto 90%;
    background: $surface;
    border: block $surface;
}
Button.-style-flat:hover {
    background: $primary;
    border: block $primary;
}
Button.-style-flat:focus {
    text-style: $button-focus-text-style;
}
Button.-style-flat.-active {
    background: $surface;
    border: block $surface;
    tint: $background 30%;
}
Button.-style-flat:disabled {
    color: auto 50%;
}
/* Flat style variants */
Button.-style-flat.-primary {
    background: $primary-muted;
    border: block $primary-muted;
    color: $text-primary;
}
Button.-style-flat.-primary:hover {
    color: $text;
    background: $primary;
    border: block $primary;
}
Button.-style-flat.-success {
    background: $success-muted;
    border: block $success-muted;
    color: $text-success;
}
Button.-style-flat.-success:hover {
    color: $text;
    background: $success;
    border: block $success;
}
Button.-style-flat.-warning {
    background: $warning-muted;
    border: block $warning-muted;
    color: $text-warning;
}
Button.-style-flat.-warning:hover {
    color: $text;
    background: $warning;
    border: block $warning;
}
Button.-style-flat.-error {
    background: $error-muted;
    border: block $error-muted;
    color: $text-error;
}
Button.-style-flat.-error:hover {
    color: $text;
    background: $error;
    border: block $error;
}

/* ===== DEFAULT STYLE (-style-default) - 3D appearance ===== */
Button.-style-default {
    text-style: bold;
    color: $button-foreground;
    background: $surface;
    border: none;
    border-top: tall $surface-lighten-1;
    border-bottom: tall $surface-darken-1;
}
Button.-style-default.-textual-compact {
    border: none !important;
}
Button.-style-default:disabled {
    text-opacity: 0.6;
}
Button.-style-default:focus {
    text-style: $button-focus-text-style;
    background-tint: $foreground 5%;
}
Button.-style-default:hover {
    border-top: tall $surface;
    background: $surface-darken-1;
}
Button.-style-default.-active {
    background: $surface;
    border-bottom: tall $surface-lighten-1;
    border-top: tall $surface-darken-1;
    tint: $background 30%;
}
/* Default style -primary variant */
Button.-style-default.-primary {
    color: $button-color-foreground;
    background: $primary;
    border-top: tall $primary-lighten-3;
    border-bottom: tall $primary-darken-3;
}
Button.-style-default.-primary:hover {
    background: $primary-darken-2;
    border-top: tall $primary;
}
Button.-style-default.-primary.-active {
    background: $primary;
    border-bottom: tall $primary-lighten-3;
    border-top: tall $primary-darken-3;
}
/* Default style -success variant */
Button.-style-default.-success {
    color: $button-color-foreground;
    background: $success;
    border-top: tall $success-lighten-2;
    border-bottom: tall $success-darken-3;
}
Button.-style-default.-success:hover {
    background: $success-darken-2;
    border-top: tall $success;
}
Button.-style-default.-success.-active {
    background: $success;
    border-bottom: tall $success-lighten-2;
    border-top: tall $success-darken-2;
}
/* Default style -warning variant */
Button.-style-default.-warning {
    color: $button-color-foreground;
    background: $warning;
    border-top: tall $warning-lighten-2;
    border-bottom: tall $warning-darken-3;
}
Button.-style-default.-warning:hover {
    background: $warning-darken-2;
    border-top: tall $warning;
}
Button.-style-default.-warning.-active {
    background: $warning;
    border-bottom: tall $warning-lighten-2;
    border-top: tall $warning-darken-2;
}
/* Default style -error variant */
Button.-style-default.-error {
    color: $button-color-foreground;
    background: $error;
    border-top: tall $error-lighten-2;
    border-bottom: tall $error-darken-3;
}
Button.-style-default.-error:hover {
    background: $error-darken-1;
    border-top: tall $error;
}
Button.-style-default.-error.-active {
    background: $error;
    border-bottom: tall $error-lighten-2;
    border-top: tall $error-darken-2;
}
```

### Input
```css
Input {
    background: $surface;
    color: $foreground;
    padding: 0 2;
    border: tall $border-blurred;
    width: 100%;
    height: 3;
    scrollbar-size-horizontal: 0;
}
Input:focus {
    border: tall $border;
}
Input.-invalid {
    border: tall $error 60%;
}
Input.-textual-compact {
    border: none;
    height: 1;
}
```

### Header
```css
Header {
    dock: top;
    width: 100%;
    background: $panel;
    color: $foreground;
    height: 1;
}
Header.-tall {
    height: 3;
}
```

### Footer
```css
Footer {
    layout: horizontal;
    color: $footer-foreground;
    background: $footer-background;
    dock: bottom;
    height: 1;
    scrollbar-size: 0 0;
}
Footer.-compact {
    /* Compact styling */
}
```

### ToggleButton (Checkbox/RadioButton base)
```css
ToggleButton {
    width: auto;
    height: auto;
    padding: 0 1;
}
ToggleButton:focus {
    text-style: bold;
}
ToggleButton.-on {
    /* Checked/selected state */
}
```

### Switch
```css
Switch {
    width: auto;
    height: auto;
    padding: 0 1;
    border: tall $border-blurred;
}
Switch:focus {
    border: tall $border;
}
Switch.-on {
    /* On state styling */
}
```

### OptionList
```css
OptionList {
    width: 100%;
    height: auto;
    max-height: 100%;
    background: $surface;
    border: tall $border-blurred;
}
OptionList:focus {
    border: tall $border;
}
OptionList.-textual-compact {
    border: none;
    max-height: none;
}
```

### Select
```css
Select {
    width: 100%;
    height: auto;
}
Select.-textual-compact {
    /* Compact styling */
}
```

### ProgressBar
```css
ProgressBar {
    width: 100%;
    height: 1;
}
```

### LoadingIndicator
```css
LoadingIndicator {
    width: 100%;
    height: 100%;
    content-align: center middle;
    background: $surface 80%;
}
```

### Rule
```css
Rule {
    color: $secondary;
}
Rule.-horizontal {
    height: 1;
    margin: 1 0;
    width: 1fr;
}
Rule.-vertical {
    width: 1;
    margin: 0 2;
    height: 1fr;
}
```

### Link
```css
Link {
    width: auto;
    height: auto;
    min-height: 1;
    color: $text-accent;
    text-style: underline;
}
Link:hover {
    color: $accent;
}
Link:focus {
    text-style: bold reverse;
}
```

## Theme Variables Required

The DEFAULT_CSS uses these theme variables that must be defined:

### Base Colors
- `$primary`, `$primary-lighten-3`, `$primary-darken-2`, `$primary-darken-3`
- `$secondary`
- `$success`, `$success-lighten-2`, `$success-darken-2`, `$success-darken-3`
- `$warning`, `$warning-lighten-2`, `$warning-darken-2`, `$warning-darken-3`
- `$error`, `$error-lighten-2`, `$error-darken-1`, `$error-darken-3`
- `$surface`, `$surface-lighten-1`, `$surface-darken-1`, `$surface-darken-2`, `$panel`
- `$foreground`, `$background`
- `$border`, `$border-blurred`
- `$accent`
- `$text` (general text color)

### Button-specific
- `$button-foreground` - Default style text color
- `$button-color-foreground` - Variant text color (primary/success/etc)
- `$button-focus-text-style` - Text style on focus

### Semantic Text Colors
- `$text-primary`, `$text-secondary`, `$text-accent`
- `$text-success`, `$text-warning`, `$text-error`

### Muted Colors (for flat button variants)
- `$primary-muted`, `$secondary-muted`, `$accent-muted`
- `$success-muted`, `$warning-muted`, `$error-muted`

### Footer-specific
- `$footer-foreground`, `$footer-background`

## CSS Priority Order

1. **User CSS** (highest priority) - App's custom stylesheet
2. **Screen CSS** - Screen-level styles
3. **Widget DEFAULT_CSS** (lowest priority) - Built-in defaults

## Implementation Notes

### Nested Selectors

Python Textual uses SCSS-like nesting (`&:hover`, `&.-class`). Our TCSS parser should either:
1. Support nesting directly
2. Expand nested rules during parsing

### Pseudo-class Support

Required pseudo-classes:
- `:hover` - Mouse hover state
- `:focus` - Keyboard focus state
- `:disabled` - Widget disabled state

### Component Classes

Some widgets use component classes for internal parts:
- `input--cursor`
- `input--placeholder`
- `input--suggestion`
- `toggle--button`
- `toggle--label`
- `option-list--option`
- `option-list--option-highlighted`
- `selection-list--button`

These are typically styled using `&>` child combinator syntax.

## Testing Strategy

1. **Parse Validation**: All DEFAULT_CSS strings parse without errors
2. **Selector Matching**: Type selectors match widget types
3. **Priority Order**: User CSS overrides defaults
4. **Theme Variables**: All variables resolve to colors
5. **State Selectors**: Pseudo-classes apply correctly
