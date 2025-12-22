# Static & Label Widget Analysis

## Static Widget (`_static.py`)

### Overview
Static is the base class for simple content display widgets. It handles rendering of text, Rich renderables, and Visual objects.

### Properties
- `content: VisualType` - The content to display (string, Rich renderable, or Visual)
- `expand: bool = False` - Expand content to fill container
- `shrink: bool = False` - Shrink content to fit container
- `markup: bool = True` - Enable Rich markup parsing

### Internal State
- `__content: VisualType` - Original content as set
- `__visual: Visual | None` - Lazy-evaluated Visual object for rendering

### Methods
- `update(content, layout=True)` - Update content and trigger refresh
- `render() -> RenderResult` - Returns the visual for rendering
- `visual` property - Lazy-creates Visual via `visualize()`

### DEFAULT_CSS
```css
Static {
    height: auto;
}
```

### Key Implementation Notes
- Extends `Widget` with `inherit_bindings=False`
- Uses `visualize()` helper to convert content → Visual
- Setting `content` clears cached dimensions and triggers layout refresh

---

## Label Widget (`_label.py`)

### Overview
Label is a thin wrapper around Static that adds variant styling support.

### Properties (in addition to Static)
- `variant: LabelVariant | None` - Visual variant for styling

### LabelVariant Type
```python
LabelVariant = Literal["success", "error", "warning", "primary", "secondary", "accent"]
```

### Implementation
- Simply adds variant as CSS class during `__init__`
- All variant styling done via DEFAULT_CSS using theme variables

### DEFAULT_CSS
```css
Label {
    width: auto;
    height: auto;
    min-height: 1;

    &.success {
        color: $text-success;
        background: $success-muted;
    }
    &.error {
        color: $text-error;
        background: $error-muted;
    }
    &.warning {
        color: $text-warning;
        background: $warning-muted;
    }
    &.primary {
        color: $text-primary;
        background: $primary-muted;
    }
    &.secondary {
        color: $text-secondary;
        background: $secondary-muted;
    }
    &.accent {
        color: $text-accent;
        background: $accent-muted;
    }
}
```

### Theme Variables Required
- `$text-success`, `$success-muted`
- `$text-error`, `$error-muted`
- `$text-warning`, `$warning-muted`
- `$text-primary`, `$primary-muted`
- `$text-secondary`, `$secondary-muted`
- `$text-accent`, `$accent-muted`

---

## Rust Implementation Considerations

### VisualType Equivalent
Need to define an enum or trait for content types:
```rust
pub enum ContentType {
    Text(String),
    Styled(StyledContent), // Rich markup parsed
}
```

### Static Struct Design
```rust
pub struct Static {
    content: ContentType,
    expand: bool,
    shrink: bool,
    markup: bool,
}

impl Static {
    pub fn update(&mut self, content: impl Into<ContentType>) { ... }
}
```

### Label Design
```rust
pub enum LabelVariant {
    Success,
    Error,
    Warning,
    Primary,
    Secondary,
    Accent,
}

pub struct Label {
    static_base: Static, // Or embed Static fields directly
    variant: Option<LabelVariant>,
}
```
