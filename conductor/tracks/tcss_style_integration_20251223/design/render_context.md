# RenderContext Design

## Overview

`RenderContext` provides widgets with access to computed styles during rendering without changing the existing `render()` method signature. It acts as a bridge between the `StyleManager` and individual widgets.

## Design Decision

Per the spec, we use a **context-based approach** rather than modifying the `render()` signature. This enables incremental migration - existing widgets continue to work with hardcoded styles while new widgets can opt-in to TCSS styling.

## RenderContext Structure

```rust
pub struct RenderContext<'a> {
    /// Reference to the StyleManager for style computation.
    style_manager: &'a StyleManager,

    /// Current parent's background color (for auto color resolution).
    parent_background: Color,

    /// Ancestor chain from root to current parent.
    /// Used for descendant/child selector matching.
    ancestors: Vec<WidgetMeta>,
}
```

Note: `StyleManager::get_style(&self)` uses interior mutability for caching,
so RenderContext can hold an immutable reference.

## Public API

### Construction

```rust
impl<'a> RenderContext<'a> {
    /// Create a new RenderContext at the root level.
    pub fn new(style_manager: &'a StyleManager) -> Self {
        Self {
            style_manager,
            parent_background: Color::default(),
            ancestors: Vec::new(),
        }
    }

    /// Create a new RenderContext with initial background.
    pub fn with_background(style_manager: &'a StyleManager, background: Color) -> Self {
        Self {
            style_manager,
            parent_background: background,
            ancestors: Vec::new(),
        }
    }
}
```

### Style Access

```rust
impl<'a> RenderContext<'a> {
    /// Get computed style for a widget.
    ///
    /// This is the primary method widgets use to get their TCSS styles.
    pub fn style_for(&self, widget_id: WidgetId, widget: &impl Widget) -> ComputedStyle {
        // widget_id must be the registry WidgetId (not the CSS id string)
        let meta = widget.widget_meta();
        let ancestor_refs: Vec<&WidgetMeta> = self.ancestors.iter().collect();

        self.style_manager.get_style(
            widget_id,
            &meta,
            &ancestor_refs,
            &self.parent_background,
        )
    }

    /// Get computed style and convert to ratatui::Style.
    ///
    /// Convenience method for the common case.
    pub fn ratatui_style_for(&self, widget_id: WidgetId, widget: &impl Widget) -> ratatui::style::Style {
        self.style_for(widget_id, widget).to_ratatui_style()
    }
}
```

### Child Context Creation

```rust
impl<'a> RenderContext<'a> {
    /// Create a child context for rendering nested widgets.
    ///
    /// Pushes the parent widget onto the ancestor chain and updates
    /// the parent background color.
    pub fn child_context(&self, parent: &impl Widget, parent_style: &ComputedStyle) -> Self {
        let mut ancestors = self.ancestors.clone();
        ancestors.push(parent.widget_meta());

        let parent_background = parent_style
            .background
            .clone()
            .unwrap_or_else(|| self.parent_background.clone());

        Self {
            style_manager: self.style_manager,
            parent_background,
            ancestors,
        }
    }

    /// Create a child context with explicit background override.
    pub fn child_context_with_background(
        &self,
        parent: &impl Widget,
        background: Color,
    ) -> Self {
        let mut ancestors = self.ancestors.clone();
        ancestors.push(parent.widget_meta());

        Self {
            style_manager: self.style_manager,
            parent_background: background,
            ancestors,
        }
    }
}
```

### Introspection

```rust
impl<'a> RenderContext<'a> {
    /// Get the current parent background color.
    pub fn parent_background(&self) -> &Color {
        &self.parent_background
    }

    /// Get the ancestor chain depth.
    pub fn depth(&self) -> usize {
        self.ancestors.len()
    }

    /// Get the current theme name.
    pub fn current_theme(&self) -> &str {
        self.style_manager.current_theme()
    }
}
```

## Context Flow Through Render Tree

### Conceptual Flow

```
run_managed() loop
    │
    ├─► Create RenderContext (root)
    │
    └─► terminal.draw(|frame| {
            app.view_with_context(frame, &context)  // Pass context to app
        })
            │
            └─► Container::render_with_context(id, area, frame, &context)
                    │
                    ├─► let style = context.style_for(id, self);
                    ├─► let child_ctx = context.child_context(self, &style);
                    │
                    └─► for child in children {
                            child.render_with_context(child_id, child_area, frame, &child_ctx);
                        }
```

### Widget Render Pattern

```rust
impl Widget for Container {
    fn render_with_context(&self, id: WidgetId, area: Rect, frame: &mut Frame, context: &RenderContext) {
        // 1. Get own computed style
        let style = context.style_for(id, self);
        let ratatui_style = style.to_ratatui_style();

        // 2. Render self with computed style
        let block = Block::default()
            .style(ratatui_style)
            .borders(Borders::ALL);
        frame.render_widget(block, area);

        // 3. Create child context (updates ancestors + background)
        let child_ctx = context.child_context(self, &style);

        // 4. Render children with child context
        let inner = block.inner(area);
        for (child_id, child) in &self.children {
            if let Some(child_area) = self.layout_cache.get(child_id) {
                child.render_with_context(*child_id, *child_area, frame, &child_ctx);
            }
        }
    }
}
```

## Migration Strategy

### Approach 1: New render_with_context Method (Recommended)

Add a new method to the Widget trait with a default implementation:

```rust
pub trait Widget: Any + std::fmt::Debug {
    // Existing method - unchanged for backwards compatibility
    fn render(&self, area: Rect, frame: &mut Frame);

    // NEW: Render with style context
    fn render_with_context(&self, id: WidgetId, area: Rect, frame: &mut Frame, context: &RenderContext) {
        // Default: delegate to existing render() for backwards compatibility
        self.render(area, frame);
    }
}
```

The render pipeline calls `render_with_context()`. Widgets that want TCSS styling override it.

### Approach 2: Optional Context Parameter (Alternative)

```rust
pub trait Widget: Any + std::fmt::Debug {
    fn render(&self, id: WidgetId, area: Rect, frame: &mut Frame, context: Option<&RenderContext>) {
        // Widgets check for context and use it if available
    }
}
```

This requires updating all widgets but provides a single method signature.

### Recommendation

**Approach 1** is recommended because:
- Zero breaking changes to existing widgets
- Widgets migrate at their own pace
- Clear opt-in semantics

## Ancestor Chain Order

Per the spec, ancestors are ordered from **root to immediate parent**:

```rust
// Widget tree: Screen > Container > Horizontal > Button
// When rendering Button:
ancestors = [
    WidgetMeta { type_name: "Screen", ... },    // index 0 (root)
    WidgetMeta { type_name: "Container", ... }, // index 1
    WidgetMeta { type_name: "Horizontal", ... }, // index 2 (parent)
]
// Button is NOT in ancestors (it's the current widget)
```

This matches CSS selector semantics where `A B` means "B descendant of A".

## Thread-Local Alternative (Not Recommended)

The spec mentions a thread-local option that was rejected:

```rust
// NOT RECOMMENDED - shown for completeness
thread_local! {
    static RENDER_CONTEXT: RefCell<Option<RenderContext<'static>>> = RefCell::new(None);
}

fn with_context<F, R>(f: F) -> R where F: FnOnce(&RenderContext) -> R {
    RENDER_CONTEXT.with(|ctx| {
        let ctx = ctx.borrow();
        f(ctx.as_ref().expect("No render context"))
    })
}
```

**Why rejected:**
- Lifetime complexity (requires `'static` or unsafe)
- Hidden global state
- Hard to test
- Doesn't compose well with async

## Testing

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn test_render_context_style_for() {
        let mut sm = StyleManager::new();
        sm.load_user_stylesheet("Button { color: red; }").unwrap();

        let mut registry = WidgetRegistry::new();
        let ctx = RenderContext::new(&sm);
        let button = Button::new("Test");
        let id = registry.add(button);

        let style = ctx.style_for(id, registry.get(id).unwrap());
        assert_eq!(style.color.unwrap().r, 255);
    }

    #[test]
    fn test_render_context_child_context() {
        let sm = StyleManager::new();
        let ctx = RenderContext::new(&sm);

        let container = Container::new();
        let container_style = ComputedStyle {
            background: Some(Color::rgb(0, 0, 255)),
            ..Default::default()
        };

        let child_ctx = ctx.child_context(&container, &container_style);

        assert_eq!(child_ctx.parent_background().r, 0);
        assert_eq!(child_ctx.parent_background().b, 255);
        assert_eq!(child_ctx.depth(), 1);
    }

    #[test]
    fn test_ancestor_chain_order() {
        let sm = StyleManager::new();
        let ctx = RenderContext::new(&sm);

        let screen = Screen::new();
        let container = Container::new();

        let ctx1 = ctx.child_context(&screen, &ComputedStyle::default());
        let ctx2 = ctx1.child_context(&container, &ComputedStyle::default());

        // Screen is at index 0, Container at index 1
        assert_eq!(ctx2.ancestors[0].type_name, "Screen");
        assert_eq!(ctx2.ancestors[1].type_name, "Container");
    }
}
```

## Integration with run_managed()

```rust
pub fn run_managed<A>(mut app: A) -> Result<(), TerminalError>
where
    A: ManagedWidgetApp,
    A::Message: From<Event>,
{
    // ... existing setup ...

    loop {
        let now = Instant::now();

        // Tick animator
        app.style_manager_mut().tick(now);

        // Poll hot reload
        if app.style_manager_mut().poll_hot_reload() {
            app.style_manager_mut().invalidate_all();
        }

        // ... existing layout logic ...

        // Create render context
        let context = RenderContext::new(app.style_manager());

        // Render with context
        terminal.draw(|f| {
            app.view_with_context(f, &context);
        })?;

        // ... existing event handling ...
    }
}
```

`ManagedWidgetApp` should provide a default `view_with_context` implementation
that delegates to `App::view`, so existing apps remain compatible.
