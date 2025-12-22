# Form Controls, Progress, & Rule Widgets Analysis

## ToggleButton (Base Class)

### Overview
ToggleButton is the base class for Checkbox and RadioButton. Extends Static, adds toggle functionality.

### Class Definition
```python
class ToggleButton(Static, can_focus=True):
    BUTTON_LEFT = "▐"
    BUTTON_INNER = "X"  # Overridden by subclasses
    BUTTON_RIGHT = "▌"
```

### Reactive Properties
- `value: reactive[bool] = False` - Toggle state
- `compact: reactive[bool] = False` - Compact mode

### Constructor
```python
def __init__(
    self,
    label: ContentText = "",
    value: bool = False,
    button_first: bool = True,  # Button position relative to label
    *,
    name, id, classes, disabled, tooltip, compact
):
```

### Bindings
```python
BINDINGS = [Binding("enter,space", "toggle_button", "Toggle", show=False)]
```

### Component Classes
- `toggle--button` - The checkbox/radio indicator
- `toggle--label` - The text label

### Messages
```python
class Changed(Message):
    _toggle_button: ToggleButton
    value: bool
```

---

## Checkbox

### Overview
Thin wrapper around ToggleButton. Default BUTTON_INNER is "X".

```python
class Checkbox(ToggleButton):
    class Changed(ToggleButton.Changed):
        @property
        def checkbox(self) -> Checkbox: ...
```

---

## RadioButton

### Overview
Extends ToggleButton with filled circle indicator.

```python
class RadioButton(ToggleButton):
    BUTTON_INNER = "\u25cf"  # ●

    class Changed(ToggleButton.Changed):
        @property
        def radio_button(self) -> RadioButton: ...
```

---

## Switch

### Overview
Boolean toggle with animated sliding effect. Uses ScrollBarRender for slider visualization.

### Class Definition
```python
class Switch(Widget, can_focus=True):
```

### Reactive Properties
- `value: reactive[bool] = False` - On/Off state
- `_slider_position: reactive[float] = 0.0` - Animation position (0.0-1.0)

### Constructor
```python
def __init__(
    self,
    value: bool = False,
    *,
    animate: bool = True,  # Enable sliding animation
    name, id, classes, disabled, tooltip
):
```

### Animation
```python
def watch_value(self, value: bool) -> None:
    target = 1.0 if value else 0.0
    if self._should_animate:
        self.animate("_slider_position", target, duration=0.3)
    else:
        self._slider_position = target
    self.post_message(self.Changed(self, self.value))
```

### Rendering
Uses `ScrollBarRender` with virtual_size=100, window_size=50, position based on _slider_position.

### Messages
```python
class Changed(Message):
    switch: Switch
    value: bool
```

### Bindings
```python
BINDINGS = [Binding("enter,space", "toggle_switch", "Toggle", show=False)]
```

---

## ProgressBar

### Overview
Composite widget with Bar, PercentageStatus, and ETAStatus components. Supports determinate and indeterminate modes.

### Class Definition
```python
class ProgressBar(Widget, can_focus=False):
```

### Reactive Properties
- `progress: reactive[float] = 0.0` - Current progress (steps)
- `total: reactive[float | None] = None` - Total steps (None = indeterminate)
- `percentage: reactive[float | None] = None` - Computed percentage
- `gradient: reactive[Gradient | None] = None` - Optional gradient

### Constructor
```python
def __init__(
    self,
    total: float | None = None,
    *,
    show_bar: bool = True,
    show_percentage: bool = True,
    show_eta: bool = True,
    name, id, classes, disabled, clock, gradient
):
```

### Methods
```python
def advance(self, advance: float = 1) -> None:
    """Advance progress by given amount."""

def update(self, *, total=..., progress=..., advance=...) -> None:
    """Update progress bar state."""
```

### Component Classes (Bar widget)
- `bar--bar` - Normal bar style
- `bar--complete` - 100% complete style
- `bar--indeterminate` - Animated indeterminate style

### Indeterminate Animation
- Auto-refresh at 15 FPS
- Sliding highlight effect using Clock

### ETA Calculation
Uses `ETA` class with time sampling for estimated completion.

---

## LoadingIndicator

### Overview
Animated loading dots with gradient color effect.

### Class Definition
```python
class LoadingIndicator(Widget):
```

### Animation
- Auto-refresh at 16 FPS
- 5 animated dots with gradient color cycling
- Blocks all input events when displayed

```python
def render(self) -> RenderResult:
    dot = "\u25cf"  # ●
    gradient = Gradient(...)
    blends = [(elapsed * speed - dot_number / 8) % 1 for dot_number in range(5)]
    # Create colored dots based on blend values
```

### CSS Classes
- `-textual-loading-indicator` - Special layer positioning

---

## Rule

### Overview
Separator line (like HTML `<hr>`), horizontal or vertical.

### Class Definition
```python
class Rule(Widget, can_focus=False):
```

### Reactive Properties
- `orientation: reactive[RuleOrientation] = "horizontal"` - horizontal | vertical
- `line_style: reactive[LineStyle] = "solid"` - Line character style

### LineStyle Options
```python
LineStyle = Literal["ascii", "blank", "dashed", "double", "heavy", "hidden", "none", "solid", "thick"]

_HORIZONTAL_LINE_CHARS = {
    "ascii": "-", "blank": " ", "dashed": "╍", "double": "═",
    "heavy": "━", "hidden": " ", "none": " ", "solid": "─", "thick": "█"
}
```

### Factory Methods
```python
@classmethod
def horizontal(cls, line_style="solid", ...) -> Rule: ...

@classmethod
def vertical(cls, line_style="solid", ...) -> Rule: ...
```

### CSS Classes
- `-horizontal` - Added when horizontal
- `-vertical` - Added when vertical

---

## Rust Implementation Considerations

### ToggleButton/Checkbox/RadioButton
```rust
pub struct ToggleButton {
    label: String,
    value: bool,
    button_first: bool,
    compact: bool,
}

pub struct Checkbox(ToggleButton);
pub struct RadioButton(ToggleButton);
```

### Switch
```rust
pub struct Switch {
    value: bool,
    slider_position: f32,
    animate: bool,
}

impl Switch {
    fn toggle(&mut self) {
        self.value = !self.value;
        // Trigger animation or direct update
    }
}
```

### ProgressBar
```rust
pub struct ProgressBar {
    progress: f64,
    total: Option<f64>,
    show_bar: bool,
    show_percentage: bool,
    show_eta: bool,
    eta_calculator: ETA,
}

impl ProgressBar {
    pub fn advance(&mut self, amount: f64) { ... }
    pub fn percentage(&self) -> Option<f64> { ... }
}
```

### Rule
```rust
pub enum RuleOrientation { Horizontal, Vertical }
pub enum LineStyle { Ascii, Blank, Dashed, Double, Heavy, Hidden, None, Solid, Thick }

pub struct Rule {
    orientation: RuleOrientation,
    line_style: LineStyle,
}
```

### Key Challenges
1. ToggleButton rendering with button + label composition
2. Switch animation using ScrollBarRender equivalent
3. ProgressBar ETA calculation with time sampling
4. LoadingIndicator gradient animation at 16 FPS
5. Rule Unicode line characters for each style
