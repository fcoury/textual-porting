# Bindings & Actions Analysis

## Overview

Python Textual uses a binding system where keys are mapped to actions. Actions are dispatched to handler methods (`action_*`) on widgets. The system supports priority bindings, namespaced actions, and dynamic action states.

## Key Components

### 1. Binding Class

```python
# binding.py:54-98
@dataclass(frozen=True)
class Binding:
    """The configuration of a key binding."""

    key: str              # Key to bind: "ctrl+s", "escape", "j,down"
    action: str           # Action to bind to: "quit", "app.quit", "save('draft')"
    description: str = "" # Description for Footer
    show: bool = True     # Show in Footer
    key_display: str | None = None  # Custom key display
    priority: bool = False  # Priority binding (checked from app down)
    tooltip: str = ""     # Tooltip in footer
    id: str | None = None  # For keymap overrides
    system: bool = False  # System binding (hidden from key panel)

    @dataclass(frozen=True)
    class Group:
        """Binding group for grouping related bindings."""
        description: str = ""
        compact: bool = False
```

### 2. BINDINGS Class Variable

```python
# Widgets, Screens, and Apps define BINDINGS
class MyWidget(Widget):
    BINDINGS = [
        Binding("ctrl+s", "save", "Save file"),
        Binding("ctrl+q", "quit", "Quit", priority=True),
        ("escape", "cancel", "Cancel"),  # Tuple shorthand
        ("j,down", "move_down"),  # Multiple keys to same action
    ]
```

### 3. BindingsMap

```python
# binding.py:184-394
class BindingsMap:
    """Manage a set of bindings."""

    def __init__(self, bindings: Iterable[BindingType] | None = None):
        self.key_to_bindings: dict[str, list[Binding]] = {}
        for binding in Binding.make_bindings(bindings or {}):
            self.key_to_bindings.setdefault(binding.key, []).append(binding)

    def bind(self, keys: str, action: str, description: str = "", ...) -> None:
        """Bind keys to an action dynamically."""

    def get_bindings_for_key(self, key: str) -> list[Binding]:
        """Get bindings for a given key."""
```

### 4. Action Parsing

```python
# actions.py:25-59
@lru_cache(maxsize=1024)
def parse(action: str) -> ActionParseResult:
    """Parses an action string.

    Returns:
        (namespace, action_name, args)
    """
    # "app.quit" -> ("app", "quit", ())
    # "save('draft')" -> ("", "save", ("draft",))
    # "screen.toggle_dark" -> ("screen", "toggle_dark", ())
```

Action format:
- `action_name` - dispatches to current namespace
- `namespace.action_name` - dispatches to specific namespace
- `action_name(args)` - action with arguments

### 5. Action Namespaces

```python
# app.py:627
self._action_targets = {"app", "screen", "focused"}
```

Built-in namespaces:
- `app` - The App instance
- `screen` - Current Screen
- `focused` - Currently focused widget

### 6. Binding Chain

The binding chain determines which bindings are active based on focus:

```python
# screen.py:400-438
@property
def _binding_chain(self) -> list[tuple[DOMNode, BindingsMap]]:
    """Binding chain from this screen."""
    focused = self.focused

    if focused is None:
        # No focus: screen + app bindings
        namespace_bindings = [
            (self, self._bindings.copy()),
            (self.app, self.app._bindings.copy()),
        ]
    else:
        # With focus: walk from focused widget up to app
        namespace_bindings = [
            (node, node._bindings.copy())
            for node in focused.ancestors_with_self
        ]

    # Filter consumed keys (e.g., Input widgets consume letter keys)
    for namespace, bindings_map in namespace_bindings:
        for filter_namespace in filter_namespaces:
            for key in list(bindings_map.key_to_bindings):
                if filter_namespace.check_consume_key(key, ...):
                    del bindings_map.key_to_bindings[key]

    return namespace_bindings
```

### 7. Modal Binding Chain

```python
# screen.py:441-447
@property
def _modal_binding_chain(self) -> list[tuple[DOMNode, BindingsMap]]:
    """The binding chain, ignoring everything before the last modal."""
    binding_chain = self._binding_chain
    for index, (node, _bindings) in enumerate(binding_chain, 1):
        if node.is_modal:
            return binding_chain[:index]
    return binding_chain
```

Modal screens cut off the binding chain, preventing parent bindings from being reached.

### 8. Key Event Handling

```python
# app.py:3805-3848
async def _check_bindings(self, key: str, priority: bool = False) -> bool:
    """Handle a key press."""

    # Direction depends on priority flag
    binding_chain = (
        reversed(self.screen._binding_chain)  # App -> focused (priority)
        if priority
        else self.screen._binding_chain  # focused -> App (normal)
    )

    for namespace, bindings in binding_chain:
        try:
            bindings_for_key = bindings.get_bindings_for_key(key)
        except NoBinding:
            continue

        for binding in bindings_for_key:
            if binding.priority != priority:
                continue
            # Check if action is enabled
            action_state = self._check_action_state(binding.action, namespace)
            if not action_state:
                continue
            # Run the action
            await self.run_action(binding.action, namespace)
            return True

    return False
```

### 9. Running Actions

```python
# app.py:4100-4150
async def run_action(
    self,
    action: str | ActionParseResult,
    default_namespace: DOMNode | None = None,
) -> bool:
    """Perform an action.

    Args:
        action: Action string like "app.quit" or "save('draft')"
        default_namespace: Default node to run action on
    """
    # Parse the action
    destination, action_name, params = actions.parse(action)

    # Resolve the namespace
    if destination == "app":
        action_target = self
    elif destination == "screen":
        action_target = self.screen
    elif destination == "focused":
        action_target = self.focused
    else:
        action_target = default_namespace

    # Find and call the action method
    method_name = f"action_{action_name}"
    if hasattr(action_target, method_name):
        method = getattr(action_target, method_name)
        await invoke(method, *params)
        return True

    return False
```

### 10. Action Method Convention

```python
class MyWidget(Widget):
    BINDINGS = [
        Binding("ctrl+s", "save('draft')"),
        Binding("ctrl+d", "toggle('debug_mode')"),
    ]

    async def action_save(self, mode: str = "normal") -> None:
        """Handler for 'save' action."""
        ...

    async def action_toggle(self, attr_name: str) -> None:
        """Handler for 'toggle' action."""
        value = getattr(self, attr_name)
        setattr(self, attr_name, not value)
```

### 11. Check Action State

```python
# dom.py:1835-1854
def check_action(
    self, action: str, parameters: tuple[object, ...]
) -> bool | None:
    """Check if an action is enabled.

    Returns:
        True - action enabled and shown
        None - action disabled but shown (grayed out)
        False - action disabled and hidden
    """
    return True  # Default: enabled
```

Override in widgets to dynamically enable/disable actions:

```python
class UndoWidget(Widget):
    def check_action(self, action: str, parameters: tuple) -> bool | None:
        if action == "undo":
            return len(self.undo_stack) > 0
        return True
```

### 12. Skip Action

```python
# actions.py:14-15
class SkipAction(Exception):
    """Raise in an action to skip the action (and allow parent bindings to run)."""

# Usage in action method:
async def action_save(self) -> None:
    if not self.can_save:
        raise SkipAction()  # Let parent handle it
    ...
```

### 13. Key Dispatch (key_* methods)

```python
# _dispatch_key.py:12-60
async def dispatch_key(node: DOMNode, event: events.Key) -> bool:
    """Dispatch a key event to method.

    This calls method named 'key_<event.key>' on a node if it exists.
    """
    handler = getattr(node, f"key_{key}", None) or getattr(node, f"_key_{key}", None)
    if handler:
        await invoke(handler, event)
        return True
    return False
```

For raw key handling (bypass binding system):

```python
class MyWidget(Widget):
    def key_space(self, event: events.Key) -> None:
        """Called when space is pressed."""
        event.prevent_default()  # Prevent binding from running
```

## Binding Resolution Flow

```
Key Press Event
    |
    v
1. Priority bindings checked (App -> focused)
    - Only bindings with priority=True
    |
    v
2. key_* method dispatch on focused widget
    |
    v
3. Normal bindings checked (focused -> App)
    - Only bindings with priority=False
    - Stopped by modal screens
    |
    v
4. Action parsed and run
    - Namespace resolved
    - action_* method called
```

## Rust Implementation Considerations

### Binding Struct

```rust
#[derive(Clone)]
pub struct Binding {
    pub key: String,
    pub action: String,
    pub description: String,
    pub show: bool,
    pub key_display: Option<String>,
    pub priority: bool,
    pub tooltip: String,
    pub id: Option<String>,
    pub system: bool,
}

impl Binding {
    pub fn new(key: impl Into<String>, action: impl Into<String>) -> Self {
        Self {
            key: key.into(),
            action: action.into(),
            ..Default::default()
        }
    }
}
```

### BindingsMap

```rust
pub struct BindingsMap {
    key_to_bindings: HashMap<String, Vec<Binding>>,
}

impl BindingsMap {
    pub fn bind(&mut self, key: &str, action: &str, description: &str) {
        self.key_to_bindings
            .entry(key.to_string())
            .or_default()
            .push(Binding::new(key, action).with_description(description));
    }

    pub fn get_bindings_for_key(&self, key: &str) -> Option<&[Binding]> {
        self.key_to_bindings.get(key).map(|v| v.as_slice())
    }
}
```

### Action Parsing

```rust
pub struct ParsedAction {
    pub namespace: String,
    pub name: String,
    pub args: Vec<serde_json::Value>,
}

pub fn parse_action(action: &str) -> Result<ParsedAction, ActionError> {
    // "app.quit" -> ("app", "quit", [])
    // "save('draft')" -> ("", "save", ["draft"])
}
```

### Action Dispatch Trait

```rust
pub trait ActionHandler {
    fn handle_action(&mut self, action: &str, args: &[serde_json::Value]) -> Result<bool, ActionError>;
    fn check_action(&self, action: &str, args: &[serde_json::Value]) -> Option<bool>;
}

// Macro to generate action handlers
action_handlers! {
    impl ActionHandler for MyWidget {
        fn action_save(&mut self, mode: String) { ... }
        fn action_toggle(&mut self, attr: String) { ... }
    }
}
```

### Binding Chain

```rust
fn binding_chain(&self) -> Vec<(Rc<dyn DOMNode>, BindingsMap)> {
    let mut chain = Vec::new();

    if let Some(focused) = self.focused() {
        // Walk from focused up to app
        let mut node: Option<Rc<dyn DOMNode>> = Some(focused);
        while let Some(n) = node {
            chain.push((n.clone(), n.bindings().clone()));
            node = n.parent();
        }
    } else {
        // No focus: screen + app
        chain.push((self.screen(), self.screen().bindings().clone()));
        chain.push((self.app(), self.app().bindings().clone()));
    }

    chain
}
```

## Key Requirements for Rust Port

1. **Binding Storage**: HashMap of key to list of bindings
2. **Action Parsing**: Regex or parser for action syntax
3. **Namespace Resolution**: app, screen, focused + custom
4. **Priority Handling**: Two-pass key handling
5. **Modal Cut-off**: Binding chain truncation
6. **Action State**: Dynamic enable/disable/hide
7. **SkipAction**: Exception equivalent for propagation

## Summary

| Python Concept | Rust Equivalent |
|---------------|-----------------|
| `Binding(key, action)` | `Binding { key, action, ... }` |
| `BINDINGS = [...]` | `const BINDINGS: &[Binding]` or trait method |
| `BindingsMap` | `HashMap<String, Vec<Binding>>` |
| `action_*` methods | Trait method or enum dispatch |
| `check_action` | Trait method returning `Option<bool>` |
| `_binding_chain` | Method returning `Vec<(Node, BindingsMap)>` |
| `SkipAction` | `Result<SkipAction, ...>` |
| `priority=True` | First pass before normal bindings |
