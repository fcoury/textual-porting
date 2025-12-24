# Python Textual Style Integration Reference

## Overview

Python Textual has a mature style integration system. This document analyzes their patterns to inform our Rust implementation.

## DEFAULT_CSS Pattern

### Widget Definition
```python
class Widget:
    DEFAULT_CSS = """
    Widget {
        scrollbar-background: $scrollbar-background;
        scrollbar-color: $scrollbar;
        background: transparent;
    }
    """
```

Every widget class can define a `DEFAULT_CSS` class variable containing TCSS rules.

### Key Properties

| Property | Value | Purpose |
|----------|-------|---------|
| `DEFAULT_CSS` | TCSS string | Widget's default styles |
| `SCOPED_CSS` | bool (True) | If True, CSS affects only widget and children |
| CSS Specificity | Lower than app CSS | App CSS always wins over DEFAULT_CSS |

### Specificity Hierarchy
1. App's `CSS` class variable (highest)
2. External CSS files (`CSS_PATH`)
3. Widget's `DEFAULT_CSS` (lowest)

This matches Textual's load order in `App._process_messages`: it reads `CSS_PATH`,
then adds widget DEFAULT_CSS, then adds App `CSS`, so later sources override earlier
ones. This aligns with our `is_user_css` flag approach.

## Style Attachment to DOMNode

```python
class DOMNode:
    def __init__(self):
        self._css_styles: Styles = Styles(self)        # From CSS rules
        self._inline_styles: Styles = Styles(self)     # Inline/programmatic
        self.styles: RenderStyles = RenderStyles(
            self, self._css_styles, self._inline_styles
        )
```

### Three-Layer Style System
1. **CSS Styles**: Rules from parsed stylesheets
2. **Inline Styles**: Programmatic `widget.styles.color = "red"`
3. **RenderStyles**: Combined view that merges both

### Component Styles
Widgets can have named sub-components with their own styles:
```python
self._component_styles: dict[str, RenderStyles] = {}
```

## Style Computation

### On-Demand Computation
Textual computes styles lazily via property descriptors:
```python
# In Styles class
get_rule(rule_name, default)  # Get individual rule
get_rules()                    # Get complete RulesMap copy
```

### Caching Mechanism
```python
class RenderStyles:
    _cache_key: tuple        # Combined update counters
    _gutter: Spacing | None  # Cached padding + border spacing
    _rich_style: Style       # Cached Rich Style object
```

Cache invalidation is driven by `_updates` counter that increments on any change.

### Ancestor Traversal
Rich style objects are computed by walking ancestors:
```python
# In rich_style property
for node in self.ancestors:
    opacity *= node.styles.opacity  # Multiplicative composition
```

## Pseudo-Class Handling

### State Tracking
```python
_has_hover_style: bool
_has_focus_within: bool
_has_order_style: bool
_has_odd_or_even: bool
```

### Evaluation
```python
_PSEUDO_CLASSES = {
    "hover": lambda widget: widget.is_hovered,
    "focus": lambda widget: widget.has_focus,
    "disabled": lambda widget: widget.disabled,
    # ...
}

def get_pseudo_classes(self) -> FrozenSet[str]:
    return frozenset(
        name for name, check in _PSEUDO_CLASSES.items()
        if check(self)
    )
```

### Style Update Trigger
```python
def _update_styles(self):
    """Called whenever CSS classes / pseudo classes change."""
    self.app.update_styles(self)
```

This propagates to the app-level style system for global handling.

## Render-Time Style Access

### Via styles Property
```python
def render(self) -> RenderableType:
    # Access computed styles
    bg = self.styles.background
    color = self.styles.color

    # Use in rendering
    return Panel(content, style=self.rich_style)
```

### Visual Style Object
```python
class VisualStyle:
    background: Color
    color: Color
    bold: bool
    italic: bool
    underline: bool
    # ...
```

### _Styled Wrapper
```python
class _Styled:
    """Wrapper that applies styles to renderables during console rendering."""
```

## Animation Integration

### Via RenderStyles
```python
def animate(self, attribute: str, value: T, duration: float, ...):
    """Animate a style attribute."""
    self._node.app.animate(...)
```

Animations are delegated to the app's animator, just like our design.

## Key Patterns for Rust Implementation

### Pattern 1: DEFAULT_CSS as Constant
- Static string on Widget type
- Parsed once at app startup
- Lower specificity than user CSS

**Rust equivalent**:
```rust
trait Widget {
    const DEFAULT_CSS: &'static str = "";
}
```

### Pattern 2: Styles Object on Widget
- Widget owns its style state
- Combined view of CSS + inline

**Rust equivalent**:
```rust
pub struct RenderContext<'a> {
    pub style_manager: &'a StyleManager,
    // ...
}

fn style_for(&self, widget: &impl Widget) -> ComputedStyle
```

### Pattern 3: Lazy Computation with Caching
- Styles computed on-demand
- Cache invalidated by update counter

**Rust equivalent**:
```rust
struct StyleCacheEntry {
    computed: ComputedStyle,
    ancestor_hash: u64,
    theme_version: u64,
}
```

### Pattern 4: Pseudo-Class as Functions
- Map of name → lambda that evaluates widget state
- Called to build pseudo-class set

**Rust equivalent**:
```rust
fn pseudo_classes(&self) -> HashSet<String> {
    let mut set = HashSet::new();
    if self.focused { set.insert("focus".to_string()); }
    if self.disabled { set.insert("disabled".to_string()); }
    set
}
```

### Pattern 5: App-Level Style Updates
- Style changes propagate to app
- App coordinates invalidation

**Rust equivalent**:
```rust
impl StyleManager {
    pub fn invalidate_widget(&mut self, widget_id: WidgetId);
    pub fn invalidate_all(&mut self);
}
```

## Differences from Python Textual

| Aspect | Python Textual | Our Rust Design |
|--------|---------------|-----------------|
| Widget owns styles | Yes (RenderStyles field) | No (StyleManager owns) |
| Inline styles | Separate object | Not in initial scope |
| Component styles | Named sub-components | Not in initial scope |
| Style access | `widget.styles.X` | `context.style_for(widget)` |
| Animation | Via RenderStyles.animate() | Via StyleManager.animate() |
| Cache location | On widget | In StyleManager |

## Recommendations

1. **Keep DEFAULT_CSS pattern** - It mirrors Python Textual and is intuitive
2. **Centralize in StyleManager** - Unlike Python's widget-owned styles, centralize for better control
3. **Use ancestor hash for cache** - More robust than Python's counter approach
4. **Support pseudo_classes() method** - Essential for interactive widgets

## Sources

- [Textual CSS Guide](https://textual.textualize.io/guide/CSS/)
- [Textual Styles Guide](https://textual.textualize.io/guide/styles/)
- [Widget.py Source](https://github.com/Textualize/textual/blob/main/src/textual/widget.py)
- [DOMNode Source](https://github.com/Textualize/textual/blob/main/src/textual/dom.py)
- [Styles.py Source](https://github.com/Textualize/textual/blob/main/src/textual/css/styles.py)
