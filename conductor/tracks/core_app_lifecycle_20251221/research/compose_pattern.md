# Compose Pattern Analysis

## Overview

Python Textual uses a `compose()` generator pattern for declarative widget tree construction. This is a core pattern used throughout the framework for building UI hierarchies.

## Key Components

### 1. ComposeResult Type

```python
# From app.py:159
ComposeResult = Iterable[Widget]
```

`ComposeResult` is simply an alias for `Iterable[Widget]`. In practice, `compose()` methods are generators that `yield` widgets.

### 2. The compose() Method

Both `App` and `Widget` define a `compose()` method:

```python
# Widget.compose() - widget.py:1631-1648
def compose(self) -> ComposeResult:
    """Called by Textual to create child widgets.

    This method is called when a widget is mounted or by setting `recompose=True` when
    calling `refresh()`.

    Example:
        def compose(self) -> ComposeResult:
            yield Header()
            yield Label("Press the button below:")
            yield Button()
            yield Footer()
    """
    yield from ()  # Default: no children

# App.compose() - app.py:1312-1317
def compose(self) -> ComposeResult:
    """Yield child widgets for a container.

    This method should be implemented in a subclass.
    """
    yield from ()
```

### 3. The compose() Helper Function

The `compose.py` module provides a helper function that processes the generator:

```python
# compose.py:12-99
def compose(node: App | Widget, compose_result: ComposeResult | None = None) -> list[Widget]:
    """Compose child widgets from a generator."""

    app = node.app
    nodes: list[Widget] = []
    compose_stack: list[Widget] = []  # For context manager nesting
    composed: list[Widget] = []       # For widgets added via context manager exit

    app._compose_stacks.append(compose_stack)
    app._composed.append(composed)

    iter_compose = iter(compose_result if compose_result is not None else node.compose())
    is_generator = hasattr(iter_compose, "throw")

    try:
        while True:
            try:
                child = next(iter_compose)
            except StopIteration:
                break

            # Validate widget type
            if not isinstance(child, Widget):
                raise MountError(f"Can't mount {type(child)}; expected a Widget instance.")

            # Handle composed widgets from context managers
            if composed:
                nodes.extend(composed)
                composed.clear()

            # If inside a context manager, add to parent container
            if compose_stack:
                compose_stack[-1].compose_add_child(child)
            else:
                nodes.append(child)

        if composed:
            nodes.extend(composed)
            composed.clear()
    finally:
        app._compose_stacks.pop()
        app._composed.pop()

    return nodes
```

### 4. Context Manager Pattern for Nesting

Widgets implement `__enter__` and `__exit__` to support the `with` syntax:

```python
# widget.py:977-994
def __enter__(self) -> Self:
    """Use as context manager when composing."""
    self.app._compose_stacks[-1].append(self)
    return self

def __exit__(self, exc_type, exc_val, exc_tb) -> None:
    """Exit compose context manager."""
    compose_stack = self.app._compose_stacks[-1]
    composed = compose_stack.pop()
    if compose_stack:
        # Nested: add to parent container
        compose_stack[-1].compose_add_child(composed)
    else:
        # Top-level: add to composed list
        self.app._composed[-1].append(composed)
```

This enables the following pattern:

```python
def compose(self) -> ComposeResult:
    with containers.HorizontalGroup():
        yield Label("Name:")
        yield Input()
    with containers.HorizontalGroup():
        yield Button("Submit")
        yield Button("Cancel")
```

### 5. compose_add_child Method

Each widget has a method to handle children during composition:

```python
# widget.py:859-870
def compose_add_child(self, widget: Widget) -> None:
    """Add a node to children.

    This is used by the compose process when it adds children.
    There is no need to use it directly, but you may want to override it in a subclass
    if you want children to be attached to a different node.
    """
    self._pending_children.append(widget)
```

Some widgets override this to redirect children (e.g., `TabbedContent`, `Collapsible`).

### 6. Widget Initialization with Children

Widgets accept `*children` in their constructor for immediate child specification:

```python
# widget.py:492-497
for child in children:
    if not isinstance(child, Widget):
        raise TypeError(...)
self._pending_children = list(children)
```

### 7. The Compose Lifecycle

When a widget is mounted, `_compose()` is called:

```python
# widget.py:4639-4651
async def _compose(self) -> None:
    try:
        widgets = [*self._pending_children, *compose(self)]
        self._pending_children.clear()
    except TypeError as error:
        raise TypeError(f"{self!r} compose() method returned an invalid result; {error}")
    except Exception as error:
        self.app._handle_exception(error)
    else:
        self._extend_compose(widgets)
        await self.mount_composed_widgets(widgets)
```

Key observations:
1. `_pending_children` (from constructor) comes first
2. Then `compose()` results are appended
3. All are mounted together via `mount_composed_widgets()`

### 8. Recompose Pattern

Widgets can be recomposed dynamically:

```python
# widget.py:1656-1668
async def recompose(self) -> None:
    """Recompose the widget.

    Recomposing will remove children and call `self.compose` again to remount.
    """
    if not self.is_attached or self._pruning:
        return

    async with self.batch():
        await self.query_children("*").exclude(".-textual-system").remove()
        if self.is_attached:
            compose_nodes = compose(self)
            await self.mount_all(compose_nodes)
```

## Real-World Examples

### Header Widget (Yields Children)

```python
# header.py:170-177
def compose(self) -> ComposeResult:
    yield HeaderIcon().data_bind(Header.icon)
    yield HeaderTitle()
    yield (
        HeaderClock().data_bind(Header.time_format)
        if self._show_clock
        else HeaderClockSpace()
    )
```

### Collapsible Widget (Context Manager Usage)

```python
# collapsible.py - uses compose_add_child override
def compose(self) -> ComposeResult:
    yield self._title
    with self.Contents():
        yield from self._contents_list

def compose_add_child(self, widget: Widget) -> None:
    """Redirect children to contents container."""
    self._contents.compose_add_child(widget)
```

## Rust Implementation Considerations

### Option 1: Iterator-Based Pattern

```rust
pub trait Compose {
    fn compose(&self) -> impl Iterator<Item = Box<dyn Widget>>;
}

// Usage
impl Compose for MyApp {
    fn compose(&self) -> impl Iterator<Item = Box<dyn Widget>> {
        [
            Box::new(Header::new()) as Box<dyn Widget>,
            Box::new(Label::new("Hello")),
            Box::new(Footer::new()),
        ].into_iter()
    }
}
```

### Option 2: Builder Pattern with Callbacks

```rust
pub trait Compose {
    fn compose(&self, builder: &mut ComposeBuilder);
}

impl Compose for MyApp {
    fn compose(&self, b: &mut ComposeBuilder) {
        b.add(Header::new());
        b.add(Label::new("Hello"));
        b.group(HorizontalGroup::new(), |b| {
            b.add(Button::new("OK"));
            b.add(Button::new("Cancel"));
        });
        b.add(Footer::new());
    }
}
```

### Option 3: Macro-Based DSL

```rust
compose! {
    Header::new(),
    Label::new("Hello"),
    HorizontalGroup {
        Button::new("OK"),
        Button::new("Cancel"),
    },
    Footer::new(),
}
```

### Key Requirements for Rust Port

1. **Type Safety**: All yielded items must be widgets
2. **Context Manager Equivalent**: Need way to group children under containers
3. **Pending Children**: Support constructor-provided children
4. **Recompose**: Support dynamic recomposition
5. **compose_add_child Override**: Some widgets redirect children to internal containers
6. **Error Propagation**: Handle errors during composition gracefully

### App-Level State

The App needs:
```rust
struct App {
    // Stack of containers being built via context manager pattern
    compose_stacks: Vec<Vec<Box<dyn Widget>>>,
    // Widgets added via context manager exit
    composed: Vec<Vec<Box<dyn Widget>>>,
}
```

## Summary

| Python Concept | Rust Equivalent |
|---------------|-----------------|
| `ComposeResult = Iterable[Widget]` | `impl Iterator<Item = Box<dyn Widget>>` or builder pattern |
| `yield Widget()` | Iterator `.next()` or `builder.add()` |
| `with Container():` | Closure-based grouping or macro |
| `compose_add_child` | Trait method override |
| `_pending_children` | `Vec<Box<dyn Widget>>` field |
| `recompose()` | Async method that clears and re-runs compose |
