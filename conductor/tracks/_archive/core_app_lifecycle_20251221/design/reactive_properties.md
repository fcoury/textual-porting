# Reactive Properties Design

## Overview

This document defines the design for a reactive property system in textual-rs that mirrors Python Textual's reactive pattern. Reactive properties automatically trigger UI updates (repaint/layout) when their values change, support watchers and validators, and enable data binding between widgets.

## Goals

1. **Ergonomic API**: Similar feel to Python Textual's `reactive(default)`
2. **Type Safety**: Full Rust type checking and ownership semantics
3. **Performance**: Minimal overhead for property access
4. **Integration**: Works with existing Widget trait and WidgetRegistry

## Design Decisions

### 1. Derive Macro Approach

We'll use a derive macro `#[derive(Reactive)]` with field attributes rather than a runtime descriptor pattern. This provides compile-time checking and zero-cost abstractions.

```rust
#[derive(Reactive)]
struct MyWidget {
    #[reactive(repaint = true)]
    label: String,

    #[reactive(layout = true)]
    visible: bool,

    #[reactive(init = true)]
    counter: i32,
}
```

### 2. Storage Model

Each reactive field `foo` generates:
- A private storage field `_reactive_foo: T`
- A getter method `foo(&self) -> &T`
- A setter method `set_foo(&mut self, value: T)`

```rust
// Generated code
impl MyWidget {
    pub fn label(&self) -> &String {
        &self._reactive_label
    }

    pub fn set_label(&mut self, value: String) {
        let old = std::mem::replace(&mut self._reactive_label, value);
        if old != self._reactive_label {
            self._on_reactive_change("label", ReactiveFlags::REPAINT);
            self.watch_label(old, &self._reactive_label);
        }
    }
}
```

### 3. Watcher Pattern

Watchers are optional methods with a specific naming convention:

```rust
impl MyWidget {
    // Optional: called when label changes
    fn watch_label(&mut self, old: String, new: &String) {
        // React to change
    }

    // Optional: validate/transform before setting
    fn validate_counter(&self, value: i32) -> i32 {
        value.clamp(0, 100)
    }
}
```

## Core Types

### ReactiveFlags

```rust
bitflags::bitflags! {
    /// Flags indicating what should happen when a reactive property changes.
    #[derive(Clone, Copy, Debug, PartialEq, Eq)]
    pub struct ReactiveFlags: u8 {
        /// Trigger a repaint (re-render) of the widget
        const REPAINT = 0b0001;
        /// Trigger a layout recalculation
        const LAYOUT = 0b0010;
        /// Call watchers even on first initialization
        const INIT = 0b0100;
        /// Always call watchers, even if value unchanged
        const ALWAYS = 0b1000;
    }
}
```

### ReactiveContext

```rust
/// Context passed to widgets when reactive properties change.
/// Allows scheduling refreshes without direct App access.
pub trait ReactiveContext {
    /// Schedule a refresh with the given flags.
    fn schedule_refresh(&mut self, flags: ReactiveFlags);

    /// Check if we're currently in the initialization phase.
    fn is_initializing(&self) -> bool;
}
```

### The Reactive Trait

```rust
/// Trait implemented by widgets with reactive properties.
pub trait Reactive {
    /// Called when any reactive property changes.
    /// The widget should schedule appropriate refreshes.
    fn on_reactive_change(&mut self, property: &str, flags: ReactiveFlags);

    /// Initialize all reactive properties with their defaults.
    /// Called after widget construction.
    fn init_reactives(&mut self);
}
```

## Derive Macro Specification

### Syntax

```rust
#[derive(Reactive)]
struct MyWidget {
    // Basic reactive with repaint (default)
    #[reactive]
    text: String,

    // Reactive with layout trigger
    #[reactive(layout = true)]
    size: Size,

    // Reactive with multiple flags
    #[reactive(repaint = true, layout = true)]
    content: Vec<String>,

    // Reactive with init (call watcher on mount)
    #[reactive(init = true)]
    selected: bool,

    // Reactive that always triggers watcher
    #[reactive(always = true)]
    mutable_data: Vec<Item>,

    // Non-reactive fields are left alone
    internal_cache: HashMap<String, Value>,
}
```

### Generated Code

For each `#[reactive]` field, the macro generates:

```rust
impl MyWidget {
    // Getter
    pub fn text(&self) -> &String {
        &self._reactive_text
    }

    // Setter with change detection
    pub fn set_text(&mut self, value: String) {
        // Optional: validate
        let value = if Self::HAS_VALIDATE_TEXT {
            self.validate_text(value)
        } else {
            value
        };

        let old = std::mem::replace(&mut self._reactive_text, value);

        // Check for change (or always if ALWAYS flag)
        if old != self._reactive_text {
            // Trigger refresh
            self._on_reactive_change("text", ReactiveFlags::REPAINT);

            // Call watcher if it exists
            if Self::HAS_WATCH_TEXT {
                self.watch_text(old, &self._reactive_text);
            }
        }
    }
}
```

### Constructor Transformation

The derive macro also generates a constructor helper:

```rust
impl MyWidget {
    /// Create with default reactive values
    pub fn new() -> Self {
        Self {
            _reactive_text: String::new(),
            _reactive_size: Size::default(),
            // ... other reactive fields ...
            internal_cache: HashMap::new(), // non-reactive unchanged
        }
    }
}
```

## Computed Properties

Computed properties are derived from other reactive values:

```rust
#[derive(Reactive)]
struct Rectangle {
    #[reactive(layout = true)]
    width: u32,

    #[reactive(layout = true)]
    height: u32,

    // Read-only computed property
    #[computed]
    area: u32,
}

impl Rectangle {
    // Required: compute method
    fn compute_area(&self) -> u32 {
        self.width * self.height
    }
}
```

Generated code:

```rust
impl Rectangle {
    pub fn area(&self) -> u32 {
        self.compute_area()
    }

    // No setter - computed properties are read-only
}
```

## Integration with Widget Trait

### Widget Trait Extension

```rust
pub trait Widget: Any + Debug + Reactive {
    fn render(&self, area: Rect, frame: &mut Frame);
    fn on_message(&mut self, msg: &dyn Any) -> bool { false }
    fn layout(&self) -> LayoutHints { LayoutHints::default() }
    fn focusable(&self) -> bool { false }
    fn as_any(&self) -> &dyn Any;
    fn as_any_mut(&mut self) -> &mut dyn Any;
}
```

### Refresh Mechanism

Widgets need access to a refresh scheduler. This is handled via the widget's context:

```rust
impl MyWidget {
    fn _on_reactive_change(&mut self, property: &str, flags: ReactiveFlags) {
        // During widget operation, this is provided by App
        if let Some(ctx) = self._reactive_context.as_mut() {
            ctx.schedule_refresh(self._id, flags);
        }
    }
}
```

## Data Binding

Data binding allows reactive properties to stay synchronized across widgets:

```rust
// In Python Textual:
// yield WorldClock("London").data_bind(WorldClockApp.time)

// In Rust, using a Binding type:
pub struct Binding<T> {
    source_id: WidgetId,
    property: &'static str,
    _marker: PhantomData<T>,
}

impl MyWidget {
    pub fn bind_time(&mut self, binding: Binding<DateTime>) {
        self._bindings.push(binding);
    }
}
```

The App handles synchronization during the update cycle.

## Example Usage

### Basic Widget

```rust
#[derive(Debug, Reactive)]
pub struct Counter {
    #[reactive]
    count: i32,

    #[reactive(layout = true)]
    visible: bool,
}

impl Counter {
    pub fn new(initial: i32) -> Self {
        Self {
            _reactive_count: initial,
            _reactive_visible: true,
        }
    }

    fn watch_count(&mut self, old: i32, new: &i32) {
        println!("Count changed from {} to {}", old, new);
    }

    fn validate_count(&self, value: i32) -> i32 {
        value.max(0)  // Ensure non-negative
    }
}

impl Widget for Counter {
    fn render(&self, area: Rect, frame: &mut Frame) {
        if self.visible() {
            let text = format!("Count: {}", self.count());
            // ... render text ...
        }
    }

    fn on_message(&mut self, msg: &dyn Any) -> bool {
        if let Some(Increment) = msg.downcast_ref() {
            self.set_count(self.count() + 1);
            return true;
        }
        false
    }
}
```

### Toggle Class Pattern

Python Textual's `toggle_class` can be implemented via a watcher:

```rust
#[derive(Debug, Reactive)]
pub struct Collapsible {
    #[reactive]
    collapsed: bool,
}

impl Collapsible {
    fn watch_collapsed(&mut self, _old: bool, &collapsed: &bool) {
        // Toggle CSS class equivalent
        if collapsed {
            self.add_class("-collapsed");
        } else {
            self.remove_class("-collapsed");
        }
    }
}
```

## Implementation Plan

### Phase 1: Core Types (Day 1)
- [ ] Define `ReactiveFlags` bitflags
- [ ] Define `Reactive` trait
- [ ] Create `reactive` attribute macro

### Phase 2: Derive Macro (Day 2-3)
- [ ] Parse `#[reactive(...)]` attributes
- [ ] Generate storage fields
- [ ] Generate getters/setters
- [ ] Handle watcher methods
- [ ] Handle validator methods

### Phase 3: Integration (Day 4)
- [ ] Add reactive context to widgets
- [ ] Connect to refresh mechanism
- [ ] Add computed property support

### Phase 4: Data Binding (Day 5)
- [ ] Define `Binding` type
- [ ] Implement binding synchronization
- [ ] Add `data_bind` method pattern

## Alternatives Considered

### 1. Runtime Descriptors (Rejected)
Python-style descriptors don't translate well to Rust's ownership model. Would require `RefCell` everywhere and runtime overhead.

### 2. Signal-based Reactivity (Deferred)
Libraries like `leptos` use signals. Could be added later but adds complexity. Our simpler approach aligns with Python Textual's model.

### 3. Observable Pattern (Rejected)
Full observer pattern with subscription lists adds memory overhead and complexity for our use case where the App centrally manages updates.

## Open Questions

1. **Thread Safety**: Should reactive properties be `Send + Sync`? Currently assuming single-threaded UI.

2. **Async Watchers**: Should watchers be able to be async? Python Textual supports async watchers.

3. **Batch Updates**: Should we batch multiple reactive changes before triggering refresh? Python Textual does this via the idle mechanism.

## Summary

| Python Concept | Rust Implementation |
|---------------|---------------------|
| `reactive(default)` | `#[reactive] field: T` |
| `_reactive_name` storage | `_reactive_name: T` field |
| Descriptor `__get__` | `fn name(&self) -> &T` |
| Descriptor `__set__` | `fn set_name(&mut self, T)` |
| `watch_name(old, new)` | `fn watch_name(&mut self, old: T, new: &T)` |
| `validate_name(value)` | `fn validate_name(&self, T) -> T` |
| `compute_name()` | `#[computed]` + `fn compute_name(&self) -> T` |
| `layout=True` | `#[reactive(layout = true)]` |
| `repaint=True` | `#[reactive(repaint = true)]` (default) |
