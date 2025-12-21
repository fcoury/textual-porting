# Reactive Properties Analysis

## Overview

Python Textual uses Python descriptors to implement reactive properties. When a reactive property changes, it can automatically trigger:
- Widget repaint
- Layout recalculation
- Recomposition
- Watcher callbacks
- CSS class toggling
- Binding updates

## Key Components

### 1. The Reactive Descriptor Classes

There are three reactive classes:

```python
# Reactive - base class with compute enabled
class Reactive(Generic[ReactiveType]):
    """Reactive descriptor with all options."""
    pass

# reactive - alias with init=True by default
class reactive(Reactive[ReactiveType]):
    """Create a reactive attribute (init=True)."""
    pass

# var - reactive with no auto-refresh
class var(Reactive[ReactiveType]):
    """Create a reactive attribute (with no auto-refresh)."""
    pass
```

### 2. Reactive Constructor Options

```python
# reactive.py:142-163
def __init__(
    self,
    default: ReactiveType | Callable[[], ReactiveType] | Initialize[ReactiveType],
    *,
    layout: bool = False,      # Trigger layout on change
    repaint: bool = True,      # Trigger repaint on change
    init: bool = False,        # Call watchers on initialize (post mount)
    always_update: bool = False,  # Call watchers even if value unchanged
    compute: bool = True,      # Run compute methods when attribute changed
    recompose: bool = False,   # Recompose widget on change
    bindings: bool = False,    # Refresh bindings on change
    toggle_class: str | None = None,  # Toggle CSS class based on truthiness
) -> None:
```

### 3. Usage Examples

```python
# Simple reactive with default value
counter = reactive(0)

# Reactive with callable default
items = reactive(list)  # Creates new list for each instance

# Reactive that triggers layout
size = reactive(100, layout=True)

# Reactive that recomposes widget
content = reactive("", recompose=True)

# Reactive that toggles CSS class
collapsed = reactive(True, toggle_class="-collapsed")

# Reactive with initialization callback
class Initialize(Generic[ReactiveType]):
    def __init__(self, callback: Callable[[ReactableType], ReactiveType]) -> None:
        self.callback = callback

names = reactive(Initialize(get_names))  # Calls get_names(self) for default
```

### 4. Internal Storage

Reactive values are stored with `_reactive_` prefix:

```python
# reactive.py:271
self.internal_name = f"_reactive_{name}"

# When getting:
return getattr(obj, f"_reactive_{name}")

# When setting:
setattr(obj, f"_reactive_{name}", value)
```

### 5. Watcher Methods

Watchers are methods named `watch_<name>` or `_watch_<name>`:

```python
class MyWidget(Widget):
    counter = reactive(0)

    # Public watcher - called after value changes
    def watch_counter(self, old_value: int, new_value: int) -> None:
        """Called when counter changes."""
        pass

    # Private watcher (alternative)
    def _watch_counter(self, value: int) -> None:
        """Can also take just the new value."""
        pass
```

Watcher signature options:
- `def watch_name()` - no arguments
- `def watch_name(value)` - new value only
- `def watch_name(old_value, value)` - both values

### 6. Validate Methods

Validators are methods named `validate_<name>` or `_validate_<name>`:

```python
class MyWidget(Widget):
    percentage = reactive(50)

    def validate_percentage(self, value: int) -> int:
        """Clamp value to 0-100."""
        return max(0, min(100, value))
```

### 7. Compute Methods

Computed properties are derived from other values:

```python
class MyWidget(Widget):
    width = reactive(100)
    height = reactive(50)

    # Computed reactive - read-only, value derived from compute method
    area = reactive(0)  # Default value, overridden by compute

    def compute_area(self) -> int:
        """Automatically called when width or height changes."""
        return self.width * self.height
```

Important notes:
- Computed reactives are read-only
- Setting a computed reactive raises `AttributeError`
- Compute methods are stored in `__computes` list

### 8. Data Binding

Data binding connects reactives across widgets:

```python
# dom.py:299-346
def data_bind(
    self,
    *reactives: Reactive[Any],
    **bind_vars: Reactive[Any] | object,
) -> Self:
    """Bind reactive data so that changes automatically propagate."""
    ...

# Usage
def compose(self) -> ComposeResult:
    yield WorldClock("Europe/London").data_bind(WorldClockApp.time)
    yield WorldClock("Europe/Paris").data_bind(WorldClockApp.time)
```

### 9. External Watchers

You can watch a reactive on another object:

```python
# reactive.py:505-532
def _watch(
    node: DOMNode,
    obj: Reactable,
    attribute_name: str,
    callback: WatchCallbackType,
    *,
    init: bool = True,
) -> None:
    """Watch a reactive variable on an object."""
    ...

# Called from DOMNode.watch()
self.watch(self.app, "title", self.update_title)
```

### 10. Mutate Method

For mutable values that can't be detected automatically:

```python
# dom.py:257-297
def mutate_reactive(self, reactive: Reactive[ReactiveType]) -> None:
    """Inform that a mutable reactive has been updated."""
    ...

# Usage
class MyWidget(Widget):
    items = reactive(list)

    def add_item(self, item: str) -> None:
        self.items.append(item)
        self.mutate_reactive(MyWidget.items)  # Trigger watchers
```

### 11. The Descriptor Protocol

```python
# __set_name__ - called when class is created
def __set_name__(self, owner: Type[MessageTarget], name: str) -> None:
    self._owner = owner
    self.name = name
    self.internal_name = f"_reactive_{name}"
    self.compute_name = f"compute_{name}"
    # Store default on class
    setattr(owner, f"_default_{name}", self._default)

# __get__ - called when attribute is accessed
def __get__(self, obj, obj_type):
    if obj is None:
        return self  # Class access
    # Initialize if needed
    if not hasattr(obj, self.internal_name):
        self._initialize_reactive(obj, self.name)
    # Run compute if exists
    if hasattr(obj, self.compute_name):
        value = getattr(obj, self.compute_name)()
        setattr(obj, self.internal_name, value)
    return getattr(obj, self.internal_name)

# __set__ - called when attribute is set
def __set__(self, obj, value):
    # Validate
    validate_fn = getattr(obj, f"validate_{name}", None)
    if validate_fn:
        value = validate_fn(value)
    # Check if changed
    if current_value != value:
        setattr(obj, self.internal_name, value)
        # Call watchers
        self._check_watchers(obj, name, current_value)
        # Trigger refresh
        obj.refresh(repaint=self._repaint, layout=self._layout)
```

## Rust Implementation Considerations

### Option 1: Derive Macro

```rust
#[derive(Reactive)]
struct MyWidget {
    #[reactive(layout = true)]
    width: u32,

    #[reactive(repaint = true, init = true)]
    label: String,

    #[reactive(toggle_class = "-collapsed")]
    collapsed: bool,
}

impl MyWidget {
    fn watch_width(&mut self, old: u32, new: u32) {
        // Called when width changes
    }

    fn validate_width(&self, value: u32) -> u32 {
        value.max(0).min(1000)
    }
}
```

### Option 2: Manual Implementation with Macros

```rust
struct MyWidget {
    _reactive_counter: Cell<i32>,
}

reactive_property!(MyWidget, counter: i32, {
    layout: false,
    repaint: true,
    default: 0,
});
```

### Option 3: Builder Pattern

```rust
impl MyWidget {
    pub fn counter(&self) -> i32 {
        self._reactive_counter.get()
    }

    pub fn set_counter(&mut self, value: i32) {
        let old = self._reactive_counter.get();
        let value = self.validate_counter(value);
        if old != value {
            self._reactive_counter.set(value);
            self.watch_counter(old, value);
            self.refresh(Refresh::REPAINT);
        }
    }
}
```

### Key Requirements for Rust Port

1. **Type-Safe Watchers**: Watcher callbacks with proper signatures
2. **Validation Hooks**: Transform values before setting
3. **Compute Properties**: Derived reactive values
4. **Change Detection**: Only trigger on actual changes
5. **CSS Class Toggling**: Automatic class updates
6. **Data Binding**: Cross-widget reactive connections
7. **Initialization**: First-time initialization callbacks

### Storage Patterns

```rust
// Option A: Cell for Copy types
struct Widget {
    _reactive_counter: Cell<i32>,
}

// Option B: RefCell for non-Copy types
struct Widget {
    _reactive_label: RefCell<String>,
}

// Option C: Wrapper type
struct Reactive<T> {
    value: RefCell<T>,
    watchers: RefCell<Vec<Box<dyn Fn(&T, &T)>>>,
}
```

### Watcher Registration

```rust
trait ReactiveWatchers {
    fn register_watchers(&mut self);
}

// Auto-generated by derive macro
impl ReactiveWatchers for MyWidget {
    fn register_watchers(&mut self) {
        // Register watch_counter, watch_label, etc.
    }
}
```

## Summary

| Python Concept | Rust Equivalent |
|---------------|-----------------|
| `reactive(default)` | `#[reactive] field: T` or `Reactive<T>` |
| `_reactive_name` storage | `_reactive_name: Cell<T>` |
| `watch_name(old, new)` | Trait method or callback |
| `validate_name(value)` | Trait method |
| `compute_name()` | Derived/computed field |
| `data_bind(Reactive)` | Signal/subscription pattern |
| `toggle_class` | Automatic in setter |
| `layout=True` | Attribute on derive macro |
