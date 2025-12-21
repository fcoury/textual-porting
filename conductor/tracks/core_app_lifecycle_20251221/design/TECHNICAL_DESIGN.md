# Technical Design Document: Core App Lifecycle

## Executive Summary

This document describes the core application lifecycle architecture for textual-rs, a Rust port of Python Textual. It covers the foundational systems that all widgets and applications depend on: compose pattern, reactive properties, message pump, screen stack, DOM queries, and layout integration.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Core Systems](#core-systems)
3. [API Reference](#api-reference)
4. [Integration Examples](#integration-examples)
5. [Implementation Order](#implementation-order)
6. [Dependencies](#dependencies)

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                          Application                             │
├─────────────────────────────────────────────────────────────────┤
│                        Screen Manager                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   Screen    │  │   Screen    │  │   Screen    │   ...        │
│  │   Stack     │  │   Stack     │  │   Stack     │              │
│  │  (Mode A)   │  │  (Mode B)   │  │  (Mode C)   │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
├─────────────────────────────────────────────────────────────────┤
│                        Active Screen                             │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     Widget Registry                        │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐       │  │
│  │  │ Widget  │  │ Widget  │  │ Widget  │  │ Widget  │  ...  │  │
│  │  │ + Pump  │  │ + Pump  │  │ + Pump  │  │ + Pump  │       │  │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘       │  │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                     Message Bus (Tokio)                          │
├─────────────────────────────────────────────────────────────────┤
│                     Layout Engine                                │
├─────────────────────────────────────────────────────────────────┤
│                     Render Engine (Ratatui)                      │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
User Input → Message Bus → Widget Handler → State Change
     ↓                                          ↓
Refresh Flags ← Reactive Properties ← Setter Methods
     ↓
Idle Event → Layout Engine → Compositor → Render
```

## Core Systems

### 1. Compose Pattern

**Purpose**: Declaratively define widget tree structure.

**Key Types**:
- `ComposeContext<'a>` - Builder context for widget trees
- `Compose` trait - Widgets that can compose children

**Example**:
```rust
impl Compose for MyApp {
    fn compose(&self, ctx: &mut ComposeContext) {
        ctx.add(Header::new("My App"));
        ctx.add_with(Container::new(), |ctx| {
            ctx.add(Label::new("Hello"));
            ctx.add(Button::new("Click"));
        });
        ctx.add_if(self.show_footer, Footer::new());
    }
}
```

**Design Document**: [compose_pattern.md](compose_pattern.md)

---

### 2. Reactive Properties

**Purpose**: Automatic UI updates when widget state changes.

**Key Types**:
- `#[derive(Reactive)]` - Derive macro for reactive widgets
- `ReactiveFlags` - Bitflags for repaint/layout triggers
- `Reactive` trait - Widget with reactive properties

**Example**:
```rust
#[derive(Debug, Reactive)]
pub struct Counter {
    #[reactive]
    count: i32,

    #[reactive(layout = true)]
    visible: bool,
}

impl Counter {
    fn watch_count(&mut self, old: i32, new: &i32) {
        println!("Count changed: {} -> {}", old, new);
    }
}
```

**Design Document**: [reactive_properties.md](reactive_properties.md)

---

### 3. Message Pump

**Purpose**: Async event processing and message bubbling.

**Key Types**:
- `Message` trait - Base trait for all messages
- `MessagePump` - Per-widget message queue
- `MessageEnvelope` - Message with metadata
- `MessageHandler` trait - Handler implementation

**Example**:
```rust
#[derive(Debug, Message)]
pub struct ButtonPressed {
    pub button_id: WidgetId,
}

#[async_trait]
impl MessageHandler for MyWidget {
    async fn handle_message(&mut self, envelope: &mut MessageEnvelope) -> bool {
        if let Some(msg) = envelope.message.as_any().downcast_ref::<ButtonPressed>() {
            self.count += 1;
            return true;
        }
        false
    }
}
```

**Design Document**: [message_pump.md](message_pump.md)

---

### 4. Screen Stack

**Purpose**: Manage multiple screens with push/pop navigation.

**Key Types**:
- `Screen<R>` trait - Screen with dismiss result type
- `ScreenStack` - Stack of screens per mode
- `ScreenManager` - Manages multiple stacks
- `ResultCallback<R>` - Callback for screen results

**Example**:
```rust
// Push screen with callback
app.push_screen(
    ConfirmDialog::new("Delete?"),
    Some(ResultCallback::new().with_callback(|result| {
        if result { delete_file(); }
    })),
).await;

// Push screen and wait
let result = app.push_screen_wait(ConfirmDialog::new("Delete?")).await?;
if result {
    delete_file();
}
```

**Design Document**: [screen_stack.md](screen_stack.md)

---

### 5. DOM Queries

**Purpose**: CSS-like widget tree queries.

**Key Types**:
- `Selector` - Parsed CSS selector
- `DOMQuery<'a, T>` - Query builder
- `TypedQueryResults<'a, T>` - Type-safe results

**Example**:
```rust
// Query by selector
for id in widget.query(".button.active")? {
    registry.get_mut(id).set_disabled(false);
}

// Query one with type
let submit: &Button = widget.query_one_typed::<Button>("#submit")?;

// Batch operations
widget.query(".item")?
    .add_class(&["selected"])
    .refresh();
```

**Design Document**: [dom_query.md](dom_query.md)

---

### 6. Layout Integration

**Purpose**: Connect widget changes to layout recalculation.

**Key Types**:
- `RefreshFlags` - What needs updating
- `WidgetRefreshState` - Per-widget refresh tracking
- `LayoutInfo` - Layout computation result

**Example**:
```rust
// Refresh triggers layout
widget.refresh(RefreshFlags::LAYOUT | RefreshFlags::REPAINT);

// Idle processing
async fn on_idle(&mut self, _: Idle) {
    self.check_refresh();  // Process pending updates
}

// Size update callback
fn size_updated(&mut self, info: LayoutInfo) -> bool {
    self.size = info.size;
    self.post_message(Resize { ... });
    true
}
```

**Design Document**: [layout_integration.md](layout_integration.md)

---

## API Reference

### Widget Trait (Extended)

```rust
pub trait Widget: Any + Debug + Reactive {
    // Core
    fn render(&self, area: Rect, frame: &mut Frame);
    fn id(&self) -> WidgetId;
    fn type_name(&self) -> &'static str;

    // Layout
    fn layout(&self) -> LayoutHints { LayoutHints::default() }
    fn focusable(&self) -> bool { false }

    // State
    fn refresh_state(&self) -> &WidgetRefreshState;
    fn refresh(&self, flags: RefreshFlags);

    // Type casting
    fn as_any(&self) -> &dyn Any;
    fn as_any_mut(&mut self) -> &mut dyn Any;

    // Classes
    fn has_class(&self, class: &str) -> bool;
    fn add_class(&mut self, class: &str);
    fn remove_class(&mut self, class: &str);
    fn toggle_class(&mut self, class: &str);
}
```

### App Trait

```rust
pub trait App: Compose + MessageHandler {
    /// Application title.
    fn title(&self) -> &str { "Textual App" }

    /// CSS styles for the application.
    fn css(&self) -> &str { "" }

    /// Key bindings.
    fn bindings(&self) -> &[Binding] { &[] }

    /// Run the application.
    async fn run(&mut self) -> Result<(), AppError>;
}
```

### Screen Trait

```rust
pub trait Screen<R = ()>: ScreenBase {
    /// Dismiss this screen with a result.
    fn dismiss(&mut self, result: R);
}

pub trait ScreenBase: Widget {
    fn is_modal(&self) -> bool { false }
    fn title(&self) -> Option<&str> { None }
    fn sub_title(&self) -> Option<&str> { None }
    fn auto_focus(&self) -> Option<&str> { None }
}
```

### Message Trait

```rust
pub trait Message: Any + Send + Sync + Debug {
    fn bubble(&self) -> bool { true }
    fn handler_name(&self) -> &'static str;
    fn type_name(&self) -> &'static str;
    fn as_any(&self) -> &dyn Any;
    fn as_any_mut(&mut self) -> &mut dyn Any;
}
```

## Integration Examples

### Complete Application

```rust
use textual_rs::prelude::*;

#[derive(Debug, Reactive)]
struct CounterApp {
    #[reactive]
    count: i32,
}

impl CounterApp {
    fn new() -> Self {
        Self { count: 0 }
    }

    fn increment(&mut self) {
        self.set_count(self.count() + 1);
    }

    fn decrement(&mut self) {
        self.set_count(self.count() - 1);
    }
}

impl Compose for CounterApp {
    fn compose(&self, ctx: &mut ComposeContext) {
        ctx.add(Header::new("Counter App"));
        ctx.add_with(Container::new(), |ctx| {
            ctx.add(Label::new(format!("Count: {}", self.count())));
            ctx.add(Button::new("Increment").with_id("inc"));
            ctx.add(Button::new("Decrement").with_id("dec"));
        });
    }
}

#[async_trait]
impl MessageHandler for CounterApp {
    async fn handle_message(&mut self, envelope: &mut MessageEnvelope) -> bool {
        if let Some(msg) = envelope.message.as_any().downcast_ref::<ButtonPressed>() {
            let button = self.query_one_typed::<Button>(&format!("#{}", msg.button_id));
            match button.and_then(|b| b.id()) {
                Some("inc") => self.increment(),
                Some("dec") => self.decrement(),
                _ => return false,
            }
            return true;
        }
        false
    }
}

impl App for CounterApp {
    fn title(&self) -> &str { "Counter" }

    fn bindings(&self) -> &[Binding] {
        &[
            Binding::new("q", "quit").with_description("Quit"),
            Binding::new("+", "increment").with_description("Increment"),
            Binding::new("-", "decrement").with_description("Decrement"),
        ]
    }
}

#[tokio::main]
async fn main() -> Result<(), AppError> {
    CounterApp::new().run().await
}
```

### Modal Dialog

```rust
#[derive(Debug)]
struct ConfirmDialog {
    state: ScreenState<bool>,
    message: String,
}

impl ConfirmDialog {
    fn new(message: &str) -> Self {
        let mut state = ScreenState::new();
        state.modal = true;
        Self { state, message: message.to_string() }
    }
}

impl Compose for ConfirmDialog {
    fn compose(&self, ctx: &mut ComposeContext) {
        ctx.add_with(VerticalGroup::new(), |ctx| {
            ctx.add(Label::new(&self.message));
            ctx.add_with(HorizontalGroup::new(), |ctx| {
                ctx.add(Button::new("Yes").with_id("yes"));
                ctx.add(Button::new("No").with_id("no"));
            });
        });
    }
}

impl Screen<bool> for ConfirmDialog {
    fn dismiss(&mut self, result: bool) {
        self.state.dismiss(result);
    }
}

impl ScreenBase for ConfirmDialog {
    fn is_modal(&self) -> bool { true }
    fn title(&self) -> Option<&str> { Some("Confirm") }
}

// Usage
let confirmed = app.push_screen_wait(ConfirmDialog::new("Delete file?")).await?;
```

### Dynamic List with Queries

```rust
#[derive(Debug, Reactive)]
struct TodoApp {
    #[reactive(recompose = true)]
    items: Vec<TodoItem>,
}

impl Compose for TodoApp {
    fn compose(&self, ctx: &mut ComposeContext) {
        ctx.add(Header::new("Todo List"));
        ctx.add_with(VerticalGroup::new().with_id("list"), |ctx| {
            for item in &self.items {
                ctx.add(TodoItemWidget::new(item.clone()));
            }
        });
        ctx.add(Input::new().with_id("new-item"));
        ctx.add(Button::new("Add").with_id("add"));
    }
}

impl TodoApp {
    fn select_all(&mut self) {
        self.query(".todo-item")
            .unwrap()
            .add_class(&["selected"]);
    }

    fn clear_completed(&mut self) {
        let completed: Vec<_> = self.query(".todo-item.completed")
            .unwrap()
            .ids()
            .to_vec();

        for id in completed {
            self.registry().remove(id);
        }
        self.refresh(RefreshFlags::LAYOUT);
    }
}
```

## Implementation Order

Based on dependencies, implement in this order:

### Week 1: Foundation
1. **ReactiveFlags** - Simple bitflags
2. **WidgetRefreshState** - Refresh tracking
3. **Message trait** + derive macro

### Week 2: Message System
4. **MessageEnvelope** - Message wrapper
5. **MessagePump** - Per-widget queue
6. **MessageHandler** trait
7. Idle event processing

### Week 3: Compose & Lifecycle
8. **ComposeContext** - Builder
9. **Compose** trait
10. Mount/Unmount integration
11. Layout trigger points

### Week 4: Reactive Properties
12. **#[derive(Reactive)]** macro
13. Watcher/Validator methods
14. Computed properties
15. Refresh integration

### Week 5: Screens
16. **ScreenBase** + **Screen** traits
17. **ScreenStack** + **ScreenManager**
18. Push/Pop/Switch operations
19. Result callbacks + await pattern

### Week 6: Queries
20. **Selector** parsing
21. **DOMQuery** implementation
22. Type-safe results
23. Batch operations

## Dependencies

### External Crates
- `tokio` - Async runtime + channels
- `async-trait` - Async trait methods
- `bitflags` - RefreshFlags
- `slotmap` - Widget storage
- `ratatui` - Rendering

### Internal Dependencies
```
Message Pump
    ↑
Reactive Properties → Refresh Flags
    ↑                     ↑
Compose Pattern      Layout Integration
    ↑                     ↑
Screen Stack ←───────────┘
    ↑
DOM Queries
```

## Summary

This design provides a complete widget lifecycle architecture for textual-rs:

| System | Purpose | Key Innovation |
|--------|---------|----------------|
| Compose | Widget trees | Builder with closures |
| Reactive | Auto-updates | Derive macro |
| Message Pump | Async events | Tokio channels |
| Screen Stack | Navigation | Generic results |
| DOM Queries | Widget finding | CSS selectors |
| Layout Integration | Refresh | Cell-based flags |

All systems are designed to work together seamlessly, providing a Python Textual-like experience with Rust's performance and safety guarantees.
