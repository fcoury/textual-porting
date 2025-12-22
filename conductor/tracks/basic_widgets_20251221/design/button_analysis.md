# Button Widget Analysis

## Overview
Button is a focusable widget that supports variants, actions, and click animations. It has extensive DEFAULT_CSS with two style modes: default (3D-like) and flat.

## Class Definition
```python
class Button(Widget, can_focus=True):
    ALLOW_SELECT = False
```

## Variants
```python
ButtonVariant = Literal["default", "primary", "success", "warning", "error"]
```

## Reactive Properties
- `label: reactive[ContentText]` - Text displayed in button (auto-converts to Content)
- `variant: reactive[str] = "default"` - Visual variant
- `compact: reactive[bool] = False` - Toggle `-textual-compact` class
- `flat: reactive[bool] = False` - Use flat style instead of 3D

## Instance Attributes
- `action: str | None` - Action to run instead of posting Pressed message
- `active_effect_duration: float = 0.2` - Button press animation duration

## Constructor Parameters
```python
def __init__(
    self,
    label: ContentText | None = None,  # Defaults to css_identifier_styled
    variant: ButtonVariant = "default",
    *,
    name: str | None = None,
    id: str | None = None,
    classes: str | None = None,
    disabled: bool = False,
    tooltip: RenderableType | None = None,
    action: str | None = None,
    compact: bool = False,
    flat: bool = False,
):
```

## Messages
```python
class Pressed(Message):
    button: Button  # The button that was pressed

    @property
    def control(self) -> Button:  # Alias for button
```

## Bindings
```python
BINDINGS = [Binding("enter", "press", "Press button", show=False)]
```

## Methods

### Public
- `press() -> Self` - Programmatically press the button, triggers animation and message/action
- `action_press()` - Action handler for "press" binding

### Validators/Watchers
- `validate_variant(variant)` - Ensures valid variant, raises `InvalidButtonVariant`
- `watch_variant(old, new)` - Updates CSS classes when variant changes
- `watch_flat(flat)` - Toggles `-style-flat`/`-style-default` classes
- `validate_label(label)` - Converts to Content type

### Class Methods (Factory constructors)
- `Button.success(...)` - Create success variant button
- `Button.warning(...)` - Create warning variant button
- `Button.error(...)` - Create error variant button

## Action vs Message
- If `action` is `None`: Posts `Button.Pressed(self)` message
- If `action` is set: Calls `app.run_action(action, default_namespace=self._parent)`

## Active Effect Animation
1. Adds `-active` class to button
2. Sets timer for `active_effect_duration`
3. Removes `-active` class after timer

## DEFAULT_CSS Structure

### Base Styles
```css
Button {
    width: auto;
    min-width: 16;
    height: auto;
    line-pad: 1;
    text-align: center;
    content-align: center middle;
}
```

### Two Style Systems

#### Flat Style (`-style-flat`)
- Simple flat appearance with hover color change
- Uses `$surface`, `$primary`, etc.
- Variants use muted backgrounds: `$primary-muted`, `$text-primary`

#### Default Style (`-style-default`)
- 3D appearance with border lighting
- Uses `border-top: tall $surface-lighten-1` and `border-bottom: tall $surface-darken-1`
- Active state flips top/bottom borders
- Variants use full color: `$primary`, `$primary-lighten-3`, `$primary-darken-3`

### Theme Variables Required
- `$surface`, `$surface-lighten-1`, `$surface-darken-1`, `$surface-darken-2`
- `$primary`, `$primary-lighten-3`, `$primary-darken-2`, `$primary-darken-3`, `$primary-muted`
- `$success`, `$success-lighten-2`, `$success-darken-2`, `$success-darken-3`, `$success-muted`
- `$warning`, `$warning-lighten-2`, `$warning-darken-2`, `$warning-darken-3`, `$warning-muted`
- `$error`, `$error-lighten-2`, `$error-darken-3`, `$error-muted`
- `$button-foreground`, `$button-color-foreground`, `$button-focus-text-style`
- `$foreground`, `$background`
- `$text-*` variants for flat style

### Pseudo-classes Used
- `:hover` - Mouse over
- `:focus` - Keyboard focus
- `:disabled` - Disabled state
- `.-active` - Button pressed (animation state)
- `.-textual-compact` - Compact mode

---

## Rust Implementation Considerations

### ButtonVariant Enum
```rust
pub enum ButtonVariant {
    Default,
    Primary,
    Success,
    Warning,
    Error,
}

impl ButtonVariant {
    pub fn as_class(&self) -> &'static str {
        match self {
            Self::Default => "-default",
            Self::Primary => "-primary",
            Self::Success => "-success",
            Self::Warning => "-warning",
            Self::Error => "-error",
        }
    }
}
```

### Button Struct
```rust
pub struct Button {
    label: Content,
    variant: ButtonVariant,
    compact: bool,
    flat: bool,
    action: Option<String>,
    active_effect_duration: f32,
    disabled: bool,
}
```

### Pressed Message
```rust
#[derive(Clone, Debug)]
pub struct ButtonPressed {
    pub button_id: WidgetId,
}
```

### Key Features to Implement
1. Reactive property system for label, variant, compact, flat
2. CSS class management for variants and states
3. Action execution via App::run_action
4. Timer-based active effect animation
5. Enter key binding for press action
6. Factory constructors (success, warning, error)
