# Header & Footer Widget Analysis

## Header Widget (`_header.py`)

### Overview
Header is a composite widget containing icon, title, and optional clock. It dynamically displays App/Screen title and subtitle.

### Component Widgets
1. **HeaderIcon** - Icon on left, opens command palette on click
2. **HeaderTitle** - Centered title/subtitle display (extends Static)
3. **HeaderClock** / **HeaderClockSpace** - Optional clock on right

### Class Definition
```python
class Header(Widget):
    # Docks to top, full width, height: 1 (or 3 when tall)
```

### Reactive Properties
- `tall: reactive[bool] = False` - Toggle `-tall` class for 3-line header
- `icon: reactive[str] = "⭘"` - Icon character
- `time_format: reactive[str] = "%X"` - strftime format for clock

### Constructor
```python
def __init__(
    self,
    show_clock: bool = False,
    *,
    name: str | None = None,
    id: str | None = None,
    classes: str | None = None,
    icon: str | None = None,
    time_format: str | None = None,
):
```

### Title Resolution
```python
@property
def screen_title(self) -> str:
    """Gets title from Screen.title or App.title"""

@property
def screen_sub_title(self) -> str:
    """Gets subtitle from Screen.sub_title or App.sub_title"""

def format_title(self) -> Content:
    """Delegates to App.format_title()"""
```

### Compose Output
```python
def compose(self) -> ComposeResult:
    yield HeaderIcon().data_bind(Header.icon)
    yield HeaderTitle()
    yield HeaderClock() if show_clock else HeaderClockSpace()
```

### Key Features
- Click to toggle `-tall` class (single vs 3-line header)
- HeaderIcon opens command palette (`app.command_palette` action)
- HeaderClock refreshes every second via `set_interval`
- Watches App and Screen title/sub_title reactively

### DEFAULT_CSS
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

---

## Footer Widget (`_footer.py`)

### Overview
Footer extends **ScrollableContainer** and dynamically displays active key bindings from the focused widget chain. It recomposes when bindings change.

### Class Definition
```python
class Footer(ScrollableContainer, can_focus=False, can_focus_children=False):
```

### Component Widgets
1. **FooterKey** - Individual key binding display (key + description)
2. **FooterLabel** - Text for binding groups
3. **KeyGroup** - Container for grouped bindings

### Reactive Properties
- `compact: reactive[bool] = False` - Toggle `-compact` class
- `show_command_palette: reactive[bool] = True` - Show command palette key
- `combine_groups: reactive[bool] = True` - Group related bindings
- `_bindings_ready: reactive[bool] = False` - Internal: bindings loaded

### Constructor
```python
def __init__(
    self,
    *children: Widget,
    name: str | None = None,
    id: str | None = None,
    classes: str | None = None,
    disabled: bool = False,
    show_command_palette: bool = True,
    compact: bool = False,
):
```

### Binding Collection
```python
def compose(self) -> ComposeResult:
    active_bindings = self.screen.active_bindings
    bindings = [
        (binding, enabled, tooltip)
        for (_, binding, enabled, tooltip) in active_bindings.values()
        if binding.show
    ]
    # Groups bindings by action and binding.group
    # Yields FooterKey widgets for each binding
```

### Key Features
- Subscribes to `screen.bindings_updated_signal` on mount
- Calls `recompose()` when bindings change
- Supports binding groups with shared labels
- Command palette key docked to right
- Horizontal scrolling for many bindings

### FooterKey Widget
```python
class FooterKey(Widget):
    COMPONENT_CLASSES = {"footer-key--key", "footer-key--description"}

    def __init__(self, key, key_display, description, action, disabled, tooltip, classes):
        ...

    def on_mouse_down(self):
        self.app.simulate_key(self.key)  # Simulate key press on click
```

### DEFAULT_CSS (Footer)
```css
Footer {
    layout: horizontal;
    color: $footer-foreground;
    background: $footer-background;
    dock: bottom;
    height: 1;
    scrollbar-size: 0 0;
}
```

### Theme Variables Required
- `$footer-foreground`, `$footer-background`
- `$footer-key-foreground`, `$footer-key-background`
- `$footer-description-foreground`, `$footer-description-background`
- `$footer-item-background`
- `$block-hover-background`

---

## Rust Implementation Considerations

### Header
```rust
pub struct Header {
    show_clock: bool,
    tall: bool,
    icon: String,
    time_format: String,
}

impl Header {
    pub fn compose(&self) -> Vec<Box<dyn Widget>> {
        vec![
            Box::new(HeaderIcon::new(self.icon.clone())),
            Box::new(HeaderTitle::new()),
            if self.show_clock {
                Box::new(HeaderClock::new(self.time_format.clone()))
            } else {
                Box::new(HeaderClockSpace::new())
            },
        ]
    }
}
```

### Footer Binding Integration
```rust
pub struct Footer {
    compact: bool,
    show_command_palette: bool,
}

impl Footer {
    fn on_bindings_changed(&mut self, screen: &Screen) {
        // Recompose with new bindings
        let active_bindings = screen.active_bindings();
        // Filter for show=true, group by action, create FooterKey widgets
    }
}
```

### Key Challenges
1. Header needs reactive watch on App/Screen title changes
2. Footer needs signal subscription for binding updates
3. FooterKey needs `simulate_key` on click
4. HeaderClock needs 1-second refresh timer
5. Both use data_bind for property synchronization
