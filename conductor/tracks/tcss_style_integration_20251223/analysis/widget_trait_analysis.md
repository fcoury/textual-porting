# Widget Trait Analysis

## Current Widget Trait Methods (widget.rs)

The `Widget` trait in `textual-rs/src/widget.rs` currently has these methods:

### Core Methods
| Method | Signature | Default | Purpose |
|--------|-----------|---------|---------|
| `render` | `fn render(&self, area: Rect, frame: &mut Frame)` | Required | Render widget to frame |
| `on_message` | `fn on_message(&mut self, msg: &dyn Any) -> bool` | `false` | Handle messages |
| `as_any` | `fn as_any(&self) -> &dyn Any` | Required | Downcasting support |
| `as_any_mut` | `fn as_any_mut(&mut self) -> &mut dyn Any` | Required | Mutable downcasting |

### Layout Methods
| Method | Signature | Default | Purpose |
|--------|-----------|---------|---------|
| `layout` | `fn layout(&self) -> LayoutHints` | `LayoutHints::default()` | Size constraints |
| `layout_type` | `fn layout_type(&self) -> Option<LayoutType>` | `None` | Container layout type |
| `get_content_width` | `fn get_content_width(&self) -> u16` | `0` | Natural content width |
| `get_content_height` | `fn get_content_height(&self) -> u16` | `0` | Natural content height |

### Focus & Interaction
| Method | Signature | Default | Purpose |
|--------|-----------|---------|---------|
| `focusable` | `fn focusable(&self) -> bool` | `false` | Can receive focus |

### Identity (Already Exists - Useful for Styling)
| Method | Signature | Default | Purpose |
|--------|-----------|---------|---------|
| `widget_type_name` | `fn widget_type_name(&self) -> &'static str` | `"Widget"` | CSS type selector matching |
| `id` | `fn id(&self) -> Option<&str>` | `None` | CSS ID selector matching |

### Timer Support
| Method | Signature | Default | Purpose |
|--------|-----------|---------|---------|
| `as_timer_widget` | `fn as_timer_widget(&self) -> Option<&dyn TimerWidget>` | `None` | Timer interface |
| `as_timer_widget_mut` | `fn as_timer_widget_mut(&mut self) -> Option<&mut dyn TimerWidget>` | `None` | Mutable timer |

### Child Management
| Method | Signature | Default | Purpose |
|--------|-----------|---------|---------|
| `as_display_override` | `fn as_display_override(&self) -> Option<&dyn ChildDisplayOverride>` | `None` | Control child visibility |
| `as_child_slotter` | `fn as_child_slotter(&self) -> Option<&dyn ChildSlotter>` | `None` | Re-parent children |

## What's Missing for TCSS Integration

### 1. DEFAULT_CSS Constant
**Need to add:**
```rust
const DEFAULT_CSS: &'static str = "";
```
- Static string for widget's default styles
- Parsed once at startup
- Merged into stylesheet with lowest specificity

### 2. Classes Access
**Need to add:**
```rust
fn classes(&self) -> &[String] { &[] }  // or HashSet<String>
```
- Currently implemented per-widget (Button, Label have `classes: Vec<String>`)
- Need trait-level access for WidgetMeta construction
- Note: Could return empty slice by default for backwards compatibility

### 3. Pseudo-classes
**Need to add:**
```rust
fn pseudo_classes(&self) -> HashSet<String> { HashSet::new() }
```
- Returns current pseudo-class state (focus, hover, active, disabled)
- Each widget implements based on its state
- Default returns empty set

### 4. WidgetMeta Construction
**Note:** `style.rs` already defines `WidgetMeta` with fields:
`type_name: String`, `classes: Vec<String>`, and `id: Option<String>`.
For TCSS integration we will extend it to include `pseudo_classes` (and keep
`type_name` unless we decide to rename for clarity).

**Need to add:**
```rust
fn widget_meta(&self) -> WidgetMeta {
    WidgetMeta {
        type_name: self.widget_type_name().to_string(),
        classes: self.classes().to_vec(),
        id: self.id().map(|s| s.to_string()),
        pseudo_classes: self.pseudo_classes(),
    }
}
```
- Convenience method for style computation
- Has default implementation using other trait methods

## Widget Implementation Patterns

Each widget currently has:
- `classes: Vec<String>` field (not on trait)
- `add_class(&mut self, class: impl Into<String>)` method
- `remove_class(&mut self, class: &str)` method
- `has_class(&self, class: &str) -> bool` method
- `refresh_state: WidgetRefreshState` field

Example from Button:
```rust
pub struct Button {
    // ...
    classes: Vec<String>,
    refresh_state: WidgetRefreshState,
    // ...
}

impl Button {
    pub fn classes(&self) -> &[String] { &self.classes }
    pub fn add_class(&mut self, class: impl Into<String>) { ... }
    pub fn remove_class(&mut self, class: &str) { ... }
    pub fn has_class(&self, class: &str) -> bool { ... }
}
```

## Migration Strategy

1. **Add trait methods with defaults** - no breaking changes
2. **Widgets can override** `classes()` and `pseudo_classes()` for proper behavior
3. **widget_meta() has default** that uses other trait methods
4. **DEFAULT_CSS is opt-in** - empty string by default

## Summary of Required Additions

```rust
trait Widget: Any + std::fmt::Debug {
    // NEW: Default CSS for this widget type
    const DEFAULT_CSS: &'static str = "";

    // NEW: Access to CSS classes
    fn classes(&self) -> &[String] { &[] }

    // NEW: Current pseudo-class state
    fn pseudo_classes(&self) -> std::collections::HashSet<String> {
        std::collections::HashSet::new()
    }

    // NEW: Construct WidgetMeta for style computation
    fn widget_meta(&self) -> crate::style::WidgetMeta {
        crate::style::WidgetMeta {
            type_name: self.widget_type_name().to_string(),
            classes: self.classes().to_vec(),
            id: self.id().map(|s| s.to_string()),
            pseudo_classes: self.pseudo_classes(),
        }
    }

    // ... existing methods unchanged ...
}
```
