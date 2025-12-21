# Compose Pattern Design

## Overview

The compose pattern allows widgets to declaratively define their child widget tree. In Python Textual, this is done via a generator that yields widgets. In Rust, we need an approach that maintains type safety while providing a similar ergonomic experience.

## Goals

1. **Declarative**: Widget trees defined at widget creation time
2. **Type Safe**: Compile-time checking of widget types
3. **Nested Containers**: Support for grouping widgets under containers
4. **Dynamic**: Support conditional and iterative widget creation
5. **Recomposable**: Ability to re-run compose and replace children

## Design Decision: Builder Pattern with Closures

After evaluating three approaches (iterator, macro, builder), we choose a **builder pattern with closures** for the following reasons:

1. **Nesting**: Closures naturally express parent-child relationships
2. **Type Safety**: Rust's type system ensures valid widget trees
3. **Flexibility**: Easy to add conditional logic
4. **No Proc Macro**: Simpler implementation without custom syntax

## Core Types

### ComposeContext

```rust
/// Context for building a widget tree during composition.
pub struct ComposeContext<'a> {
    registry: &'a mut WidgetRegistry,
    parent_id: WidgetId,
    children: Vec<WidgetId>,
}

impl<'a> ComposeContext<'a> {
    /// Add a widget as a child.
    pub fn add<W: Widget + 'static>(&mut self, widget: W) -> WidgetId {
        let id = self.registry.add(widget);
        self.registry.set_parent(id, Some(self.parent_id));
        self.children.push(id);
        id
    }

    /// Add a widget and compose its children.
    pub fn add_with<W, F>(&mut self, widget: W, children: F) -> WidgetId
    where
        W: Widget + 'static,
        F: FnOnce(&mut ComposeContext),
    {
        let id = self.registry.add(widget);
        self.registry.set_parent(id, Some(self.parent_id));
        self.children.push(id);

        // Create nested context for children
        let mut child_ctx = ComposeContext {
            registry: self.registry,
            parent_id: id,
            children: Vec::new(),
        };
        children(&mut child_ctx);

        id
    }

    /// Conditionally add a widget.
    pub fn add_if<W: Widget + 'static>(&mut self, condition: bool, widget: W) -> Option<WidgetId> {
        if condition {
            Some(self.add(widget))
        } else {
            None
        }
    }

    /// Add multiple widgets from an iterator.
    pub fn add_iter<W, I>(&mut self, widgets: I) -> Vec<WidgetId>
    where
        W: Widget + 'static,
        I: IntoIterator<Item = W>,
    {
        widgets.into_iter().map(|w| self.add(w)).collect()
    }
}
```

### The Compose Trait

```rust
/// Trait for widgets that can compose child widgets.
pub trait Compose: Widget {
    /// Define the child widgets for this widget.
    ///
    /// This method is called when the widget is mounted and when
    /// `recompose()` is called.
    fn compose(&self, ctx: &mut ComposeContext) {
        // Default: no children
    }
}
```

## Usage Examples

### Basic Composition

```rust
#[derive(Debug)]
struct MyApp {
    show_footer: bool,
}

impl Compose for MyApp {
    fn compose(&self, ctx: &mut ComposeContext) {
        ctx.add(Header::new("My App"));
        ctx.add(Label::new("Welcome!"));
        ctx.add(Button::new("Click Me"));

        if self.show_footer {
            ctx.add(Footer::new());
        }
    }
}
```

### Nested Containers

```rust
impl Compose for SettingsScreen {
    fn compose(&self, ctx: &mut ComposeContext) {
        ctx.add(Header::new("Settings"));

        // Vertical group with children
        ctx.add_with(VerticalGroup::new(), |ctx| {
            ctx.add(Label::new("User Settings"));
            ctx.add(Input::new("username"));
            ctx.add(Input::new("email"));
        });

        // Horizontal group for buttons
        ctx.add_with(HorizontalGroup::new(), |ctx| {
            ctx.add(Button::new("Save"));
            ctx.add(Button::new("Cancel"));
        });
    }
}
```

### Dynamic Lists

```rust
impl Compose for TodoList {
    fn compose(&self, ctx: &mut ComposeContext) {
        ctx.add(Header::new("Todo List"));

        ctx.add_with(VerticalGroup::new(), |ctx| {
            for item in &self.items {
                ctx.add(TodoItem::new(item.clone()));
            }
        });

        ctx.add(Button::new("Add Item"));
    }
}
```

### Conditional Rendering

```rust
impl Compose for Dashboard {
    fn compose(&self, ctx: &mut ComposeContext) {
        ctx.add(Header::new("Dashboard"));

        // Show loading or content
        if self.loading {
            ctx.add(LoadingIndicator::new());
        } else {
            ctx.add_with(ContentArea::new(), |ctx| {
                ctx.add(StatsPanel::new(&self.stats));
                ctx.add(RecentActivity::new(&self.activity));
            });
        }

        ctx.add_if(self.show_footer, Footer::new());
    }
}
```

## Container Widgets

Standard container widgets that group children:

### VerticalGroup

```rust
#[derive(Debug)]
pub struct VerticalGroup {
    gap: u16,
}

impl VerticalGroup {
    pub fn new() -> Self {
        Self { gap: 0 }
    }

    pub fn with_gap(gap: u16) -> Self {
        Self { gap }
    }
}

impl Widget for VerticalGroup {
    fn render(&self, area: Rect, frame: &mut Frame) {
        // Children are rendered by the layout system
    }

    fn layout(&self) -> LayoutHints {
        LayoutHints::new().with_direction(Direction::Vertical)
    }
}

impl Compose for VerticalGroup {
    // Children are added via add_with() in parent's compose
}
```

### HorizontalGroup

```rust
#[derive(Debug)]
pub struct HorizontalGroup {
    gap: u16,
}

impl Widget for HorizontalGroup {
    fn layout(&self) -> LayoutHints {
        LayoutHints::new().with_direction(Direction::Horizontal)
    }
}
```

### Grid

```rust
#[derive(Debug)]
pub struct Grid {
    columns: usize,
    gap: u16,
}

impl Grid {
    pub fn new(columns: usize) -> Self {
        Self { columns, gap: 0 }
    }
}
```

## Lifecycle Integration

### Initial Composition (Mount)

When a widget is mounted, its `compose()` method is called:

```rust
impl WidgetRegistry {
    pub async fn mount(&mut self, widget_id: WidgetId) {
        // Get widget and compose children
        if let Some(widget) = self.get(widget_id) {
            if let Some(composable) = widget.as_any().downcast_ref::<dyn Compose>() {
                let mut ctx = ComposeContext {
                    registry: self,
                    parent_id: widget_id,
                    children: Vec::new(),
                };
                composable.compose(&mut ctx);
            }
        }

        // Post mount event
        self.post_message(widget_id, MountEvent);
    }
}
```

### Recomposition

Recomposition removes existing children and re-runs compose:

```rust
impl WidgetRegistry {
    pub async fn recompose(&mut self, widget_id: WidgetId) {
        // Remove existing children
        let children = self.get_children(widget_id).to_vec();
        for child_id in children {
            self.remove_subtree(child_id).await;
        }

        // Re-compose
        self.mount(widget_id).await;

        // Trigger layout
        self.schedule_layout(widget_id);
    }
}
```

Widgets can trigger recomposition via reactive properties:

```rust
#[derive(Debug, Reactive)]
struct ItemList {
    #[reactive(recompose = true)]
    items: Vec<String>,
}
```

## Pending Children Pattern

Like Python Textual, widgets can accept children in their constructor:

```rust
impl VerticalGroup {
    pub fn with_children<I, W>(children: I) -> Self
    where
        I: IntoIterator<Item = W>,
        W: Widget + 'static,
    {
        Self {
            pending_children: children.into_iter().map(|w| Box::new(w) as _).collect(),
            ..Default::default()
        }
    }
}
```

During mounting, pending children are added before `compose()` runs:

```rust
impl WidgetRegistry {
    pub async fn mount(&mut self, widget_id: WidgetId) {
        // 1. Mount pending children first
        let pending = self.take_pending_children(widget_id);
        for child in pending {
            let child_id = self.add(child);
            self.set_parent(child_id, Some(widget_id));
        }

        // 2. Run compose() for additional children
        // ...
    }
}
```

## Alternative: Macro-Based DSL (Future Enhancement)

For even more ergonomic syntax, a proc macro could provide:

```rust
compose! {
    Header::new("My App"),
    VerticalGroup {
        Label::new("Hello"),
        Button::new("Click"),
    },
    if self.show_footer {
        Footer::new(),
    },
}
```

This would expand to the builder pattern calls. Deferred to future iteration.

## Comparison with Python Textual

| Python Textual | Rust Implementation |
|---------------|---------------------|
| `def compose(self) -> ComposeResult:` | `fn compose(&self, ctx: &mut ComposeContext)` |
| `yield Widget()` | `ctx.add(Widget::new())` |
| `with Container():` | `ctx.add_with(Container::new(), \|ctx\| { ... })` |
| `yield from iterable` | `ctx.add_iter(iterable)` |
| Conditional `if` in generator | `ctx.add_if(condition, widget)` |
| `compose_add_child()` override | Implement custom `Compose` logic |
| `*children` in constructor | `with_children()` builder method |
| `recompose()` | `registry.recompose(widget_id)` |

## Implementation Plan

### Phase 1: Core Types
- [ ] Define `ComposeContext` struct
- [ ] Define `Compose` trait
- [ ] Add basic `add()` method

### Phase 2: Nesting
- [ ] Implement `add_with()` for nested containers
- [ ] Add `add_if()` for conditionals
- [ ] Add `add_iter()` for lists

### Phase 3: Integration
- [ ] Connect compose to mount lifecycle
- [ ] Implement `recompose()`
- [ ] Add pending children pattern

### Phase 4: Standard Containers
- [ ] `VerticalGroup`
- [ ] `HorizontalGroup`
- [ ] `Grid`
- [ ] `ScrollableContainer`

## Open Questions

1. **Widget References**: Should `add()` return a handle for later mutation?

2. **Async Compose**: Should `compose()` be async to support data fetching?

3. **Compose Caching**: Should compose results be cached and diffed?

## Summary

The builder pattern with closures provides:
- **Clear syntax** for widget tree construction
- **Type safety** via Rust's type system
- **Natural nesting** via closure scoping
- **Familiar patterns** for Rust developers
- **Easy conditional/iterative** widget creation
