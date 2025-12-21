# Screen Stack Design

## Overview

The screen stack manages multiple screens in a layered fashion, allowing screens to be pushed, popped, and switched. Screens can return results to their callers via callbacks or async futures. Modal screens take priority over app-level bindings.

## Goals

1. **Stack-Based Navigation**: Push/pop/switch screens naturally
2. **Result Passing**: Screens can dismiss with a result for the caller
3. **Modal Support**: Modal screens get binding priority
4. **Mode Support**: Multiple independent screen stacks
5. **Type-Safe Results**: Generic result types with compile-time checking

## Design Decisions

### 1. Rc<RefCell> for Screen References

Screens need to be shared between the stack and widgets that reference them. Using `Rc<RefCell<Screen>>` provides:
- Shared ownership between stack and active references
- Interior mutability for event handling
- No lifetime complexity across async boundaries

### 2. Generic Result Types

```rust
pub struct Screen<R = ()> {
    // R is the dismiss result type
}

// Usage:
pub struct ConfirmDialog;
impl Screen<bool> for ConfirmDialog { ... }  // Returns bool

pub struct FilePickerScreen;
impl Screen<PathBuf> for FilePickerScreen { ... }  // Returns path
```

### 3. Oneshot Channels for Await Pattern

For the `push_screen_wait()` pattern, we use `tokio::sync::oneshot` channels:

```rust
let (tx, rx) = oneshot::channel();
app.push_screen(dialog, Some(tx));
let result = rx.await;  // Blocks until dismiss
```

## Core Types

### Screen Trait

```rust
use std::any::Any;

/// Base trait for all screens.
pub trait ScreenBase: Widget + Any {
    /// Whether this screen is modal (takes binding priority).
    fn is_modal(&self) -> bool { false }

    /// Get the screen's title.
    fn title(&self) -> Option<&str> { None }

    /// Get the screen's subtitle.
    fn sub_title(&self) -> Option<&str> { None }

    /// The CSS selector for auto-focus on mount.
    fn auto_focus(&self) -> Option<&str> { None }
}

/// A screen that can return a result when dismissed.
pub trait Screen<R = ()>: ScreenBase {
    /// Dismiss this screen with a result.
    fn dismiss(&mut self, result: R);
}
```

### ScreenState

```rust
/// Internal state for screen management.
pub struct ScreenState<R = ()> {
    /// The widget ID for this screen.
    widget_id: WidgetId,
    /// Whether this screen is modal.
    modal: bool,
    /// Title of the screen.
    title: Option<String>,
    /// Subtitle of the screen.
    sub_title: Option<String>,
    /// Compositor for rendering.
    compositor: Compositor,
    /// Widgets that need repaint.
    dirty_widgets: HashSet<WidgetId>,
    /// Currently focused widget.
    focused: Option<WidgetId>,
    /// Stack of result callbacks.
    result_callbacks: Vec<ResultCallback<R>>,
    /// Selector for auto-focus.
    auto_focus: Option<String>,
}
```

### ResultCallback

```rust
use tokio::sync::oneshot;

/// Callback for screen dismiss results.
pub struct ResultCallback<R> {
    /// The requester that pushed this screen.
    requester: Option<MessageSender>,
    /// Callback function to invoke with result.
    callback: Option<Box<dyn FnOnce(R) + Send>>,
    /// Future sender for await pattern.
    future: Option<oneshot::Sender<R>>,
}

impl<R> ResultCallback<R> {
    pub fn new() -> Self {
        Self {
            requester: None,
            callback: None,
            future: None,
        }
    }

    pub fn with_callback<F>(mut self, callback: F) -> Self
    where
        F: FnOnce(R) + Send + 'static,
    {
        self.callback = Some(Box::new(callback));
        self
    }

    pub fn with_future(mut self, tx: oneshot::Sender<R>) -> Self {
        self.future = Some(tx);
        self
    }

    /// Invoke the callback with the result.
    pub fn call(self, result: R) {
        if let Some(tx) = self.future {
            let _ = tx.send(result);
        }
        if let Some(callback) = self.callback {
            callback(result);
        }
    }
}
```

### ScreenStack

```rust
use std::rc::Rc;
use std::cell::RefCell;

/// A stack of screens for a single mode.
pub struct ScreenStack {
    screens: Vec<Rc<RefCell<dyn ScreenBase>>>,
}

impl ScreenStack {
    pub fn new() -> Self {
        Self { screens: Vec::new() }
    }

    /// Get the active (top) screen.
    pub fn active(&self) -> Option<Rc<RefCell<dyn ScreenBase>>> {
        self.screens.last().cloned()
    }

    /// Push a screen onto the stack.
    pub fn push(&mut self, screen: Rc<RefCell<dyn ScreenBase>>) {
        self.screens.push(screen);
    }

    /// Pop the top screen from the stack.
    pub fn pop(&mut self) -> Option<Rc<RefCell<dyn ScreenBase>>> {
        if self.screens.len() > 1 {
            self.screens.pop()
        } else {
            None // Can't pop the last screen
        }
    }

    /// Get the screen below the active one.
    pub fn previous(&self) -> Option<Rc<RefCell<dyn ScreenBase>>> {
        if self.screens.len() > 1 {
            self.screens.get(self.screens.len() - 2).cloned()
        } else {
            None
        }
    }

    /// Get all screens in the stack.
    pub fn iter(&self) -> impl Iterator<Item = &Rc<RefCell<dyn ScreenBase>>> {
        self.screens.iter()
    }

    /// Number of screens in the stack.
    pub fn len(&self) -> usize {
        self.screens.len()
    }

    /// Check if there's only one screen.
    pub fn is_at_root(&self) -> bool {
        self.screens.len() <= 1
    }
}
```

### ScreenManager

```rust
use std::collections::HashMap;

/// Manages multiple screen stacks (modes) for an app.
pub struct ScreenManager {
    /// Screen stacks by mode name.
    stacks: HashMap<String, ScreenStack>,
    /// Current active mode.
    current_mode: String,
    /// Installed screen factories.
    installed_screens: HashMap<String, Box<dyn Fn() -> Box<dyn ScreenBase>>>,
}

impl ScreenManager {
    pub const DEFAULT_MODE: &'static str = "_default";

    pub fn new() -> Self {
        let mut stacks = HashMap::new();
        stacks.insert(Self::DEFAULT_MODE.to_string(), ScreenStack::new());

        Self {
            stacks,
            current_mode: Self::DEFAULT_MODE.to_string(),
            installed_screens: HashMap::new(),
        }
    }

    /// Get the current screen stack.
    pub fn current_stack(&self) -> &ScreenStack {
        &self.stacks[&self.current_mode]
    }

    /// Get the current screen stack mutably.
    pub fn current_stack_mut(&mut self) -> &mut ScreenStack {
        self.stacks.get_mut(&self.current_mode).unwrap()
    }

    /// Get the active screen.
    pub fn screen(&self) -> Option<Rc<RefCell<dyn ScreenBase>>> {
        self.current_stack().active()
    }

    /// Switch to a different mode.
    pub fn switch_mode(&mut self, mode: &str) {
        if !self.stacks.contains_key(mode) {
            self.stacks.insert(mode.to_string(), ScreenStack::new());
        }
        self.current_mode = mode.to_string();
    }

    /// Get the current mode name.
    pub fn current_mode(&self) -> &str {
        &self.current_mode
    }
}
```

## Screen Operations

### Push Screen

```rust
impl ScreenManager {
    /// Push a new screen onto the stack.
    pub async fn push_screen<S, R>(
        &mut self,
        screen: S,
        callback: Option<ResultCallback<R>>,
        message_bus: &MessageBus,
    ) -> Result<(), ScreenError>
    where
        S: Screen<R> + 'static,
        R: 'static,
    {
        // Suspend current screen
        if let Some(current) = self.screen() {
            message_bus.post(current.borrow().widget_id(), ScreenSuspend);
            current.borrow_mut().refresh();
        }

        // Setup result callback
        let mut screen_state = screen.state_mut();
        if let Some(cb) = callback {
            screen_state.result_callbacks.push(cb);
        }

        // Add to stack
        let screen_rc = Rc::new(RefCell::new(screen));
        self.current_stack_mut().push(screen_rc.clone());

        // Mount and resume new screen
        message_bus.post(screen_rc.borrow().widget_id(), ScreenResume);

        Ok(())
    }
}
```

### Pop Screen

```rust
impl ScreenManager {
    /// Pop the current screen from the stack.
    pub async fn pop_screen(&mut self, message_bus: &MessageBus) -> Result<(), ScreenError> {
        if self.current_stack().is_at_root() {
            return Err(ScreenError::CantPopLastScreen);
        }

        // Pop and cleanup
        let popped = self.current_stack_mut().pop().unwrap();

        // Resume previous screen
        if let Some(previous) = self.screen() {
            message_bus.post(previous.borrow().widget_id(), ScreenResume);
        }

        // Unmount popped screen
        message_bus.post(popped.borrow().widget_id(), ScreenUnmount);

        Ok(())
    }
}
```

### Switch Screen

```rust
impl ScreenManager {
    /// Replace the top screen with a new one.
    pub async fn switch_screen<S, R>(
        &mut self,
        screen: S,
        message_bus: &MessageBus,
    ) -> Result<(), ScreenError>
    where
        S: Screen<R> + 'static,
        R: 'static,
    {
        // Pop current (without result callback)
        let popped = self.current_stack_mut().pop();

        // Push new screen
        let screen_rc = Rc::new(RefCell::new(screen));
        self.current_stack_mut().push(screen_rc.clone());

        // Resume new screen
        message_bus.post(screen_rc.borrow().widget_id(), ScreenResume);

        // Cleanup old screen
        if let Some(old) = popped {
            message_bus.post(old.borrow().widget_id(), ScreenUnmount);
        }

        Ok(())
    }
}
```

### Dismiss with Result

```rust
impl<R> ScreenState<R> {
    /// Dismiss this screen with a result.
    pub fn dismiss(&mut self, result: R) {
        // Pop and invoke the callback
        if let Some(callback) = self.result_callbacks.pop() {
            callback.call(result);
        }
    }
}

// Convenience method on Screen
impl<S, R> ScreenOps<R> for S
where
    S: Screen<R>,
{
    fn dismiss(&mut self, result: R) {
        self.state_mut().dismiss(result);
        // The ScreenManager will handle the actual pop
        // by listening to a Dismiss message
    }
}
```

### Push Screen and Wait

```rust
impl ScreenManager {
    /// Push a screen and wait for its result.
    pub async fn push_screen_wait<S, R>(
        &mut self,
        screen: S,
        message_bus: &MessageBus,
    ) -> Result<R, ScreenError>
    where
        S: Screen<R> + 'static,
        R: 'static,
    {
        let (tx, rx) = oneshot::channel();

        let callback = ResultCallback::new().with_future(tx);
        self.push_screen(screen, Some(callback), message_bus).await?;

        rx.await.map_err(|_| ScreenError::DismissedWithoutResult)
    }
}
```

## Screen Events

```rust
/// Screen lifecycle events.
#[derive(Debug, Clone, Message)]
#[message(bubble = false)]
pub struct ScreenResume;

#[derive(Debug, Clone, Message)]
#[message(bubble = false)]
pub struct ScreenSuspend;

#[derive(Debug, Clone, Message)]
#[message(bubble = false)]
pub struct ScreenUnmount;
```

## Modal Screens

```rust
/// A modal screen that takes binding priority.
pub struct ModalScreen<R = ()> {
    state: ScreenState<R>,
}

impl<R> ModalScreen<R> {
    pub fn new() -> Self {
        let mut state = ScreenState::new();
        state.modal = true;
        Self { state }
    }
}

impl<R> ScreenBase for ModalScreen<R> {
    fn is_modal(&self) -> bool {
        true
    }
}
```

### Modal Binding Behavior

When checking bindings, modal screens cut off the binding chain:

```rust
impl ScreenManager {
    /// Get the binding chain respecting modal screens.
    pub fn binding_chain(&self) -> Vec<(WidgetId, BindingsMap)> {
        let mut chain = Vec::new();

        if let Some(screen) = self.screen() {
            let screen_ref = screen.borrow();

            // Walk from focused widget up
            // ... collect bindings ...

            // If screen is modal, stop here (don't include app bindings)
            if screen_ref.is_modal() {
                return chain;
            }
        }

        // Include app bindings
        chain.push((self.app_id, self.app_bindings.clone()));
        chain
    }
}
```

## Installed Screens

```rust
impl ScreenManager {
    /// Install a screen factory by name.
    pub fn install_screen<F, S>(&mut self, name: &str, factory: F)
    where
        F: Fn() -> S + 'static,
        S: ScreenBase + 'static,
    {
        self.installed_screens.insert(
            name.to_string(),
            Box::new(move || Box::new(factory()) as Box<dyn ScreenBase>),
        );
    }

    /// Uninstall a screen by name.
    pub fn uninstall_screen(&mut self, name: &str) -> bool {
        self.installed_screens.remove(name).is_some()
    }

    /// Check if a screen is installed.
    pub fn is_screen_installed(&self, name: &str) -> bool {
        self.installed_screens.contains_key(name)
    }

    /// Get a screen by name (creates new instance).
    pub fn get_screen(&self, name: &str) -> Option<Box<dyn ScreenBase>> {
        self.installed_screens.get(name).map(|f| f())
    }
}
```

## Example Usage

### Basic Screen Push/Pop

```rust
// Define a settings screen
struct SettingsScreen {
    state: ScreenState<()>,
}

impl SettingsScreen {
    fn new() -> Self {
        Self {
            state: ScreenState::new(),
        }
    }
}

impl ScreenBase for SettingsScreen {
    fn title(&self) -> Option<&str> { Some("Settings") }
}

impl Screen<()> for SettingsScreen {
    fn dismiss(&mut self, result: ()) {
        self.state.dismiss(result);
    }
}

// Push and pop
app.push_screen(SettingsScreen::new(), None).await;
// Later...
app.pop_screen().await;
```

### Screen with Result

```rust
// Confirmation dialog that returns bool
struct ConfirmDialog {
    state: ScreenState<bool>,
    message: String,
}

impl ConfirmDialog {
    fn new(message: &str) -> Self {
        let mut state = ScreenState::new();
        state.modal = true;  // Modal for focus
        Self {
            state,
            message: message.to_string(),
        }
    }

    fn confirm(&mut self) {
        self.dismiss(true);
    }

    fn cancel(&mut self) {
        self.dismiss(false);
    }
}

impl Screen<bool> for ConfirmDialog {
    fn dismiss(&mut self, result: bool) {
        self.state.dismiss(result);
    }
}

// Usage with callback
app.push_screen(
    ConfirmDialog::new("Delete file?"),
    Some(ResultCallback::new().with_callback(|confirmed| {
        if confirmed {
            delete_file();
        }
    })),
).await;

// Usage with await
let confirmed = app.push_screen_wait(ConfirmDialog::new("Delete file?")).await?;
if confirmed {
    delete_file();
}
```

### Mode-Based Navigation

```rust
// Define modes
app.screen_manager.switch_mode("editor");
app.push_screen(EditorScreen::new(), None).await;

app.screen_manager.switch_mode("preview");
app.push_screen(PreviewScreen::new(), None).await;

// Switch between modes preserves separate stacks
app.screen_manager.switch_mode("editor");  // Back to editor
```

## Comparison with Python Textual

| Python Textual | Rust Implementation |
|---------------|---------------------|
| `_screen_stacks: dict[str, list[Screen]]` | `HashMap<String, ScreenStack>` |
| `push_screen(screen, callback)` | `push_screen(screen, Option<ResultCallback<R>>)` |
| `Screen[ScreenResultType]` | `Screen<R>` trait |
| `ResultCallback` class | `ResultCallback<R>` struct |
| `dismiss(result)` | `state.dismiss(result)` |
| `ModalScreen` | `ModalScreen<R>` with `modal: true` |
| `push_screen_wait()` | `push_screen_wait().await` with oneshot |
| `ScreenResume/ScreenSuspend` | Event structs |
| `install_screen()` | `install_screen(name, factory)` |
| Modes (`_current_mode`) | `switch_mode(name)` |

## Implementation Plan

### Phase 1: Core Types
- [ ] Define `ScreenBase` trait
- [ ] Define `Screen<R>` trait
- [ ] Define `ScreenState<R>` struct
- [ ] Define `ResultCallback<R>` struct

### Phase 2: Screen Stack
- [ ] Implement `ScreenStack`
- [ ] Implement `ScreenManager`
- [ ] Add mode support

### Phase 3: Operations
- [ ] Implement `push_screen()`
- [ ] Implement `pop_screen()`
- [ ] Implement `switch_screen()`
- [ ] Implement `dismiss()`

### Phase 4: Advanced Features
- [ ] Implement `push_screen_wait()` with oneshot
- [ ] Implement `install_screen()` / `uninstall_screen()`
- [ ] Implement modal binding chain cutoff

### Phase 5: Events
- [ ] Define `ScreenResume`, `ScreenSuspend`, `ScreenUnmount`
- [ ] Integrate with message pump

## Open Questions

1. **Screen Identity**: Should screens have unique IDs separate from widget IDs?

2. **Screen Caching**: Should dismissed screens be cached for quick re-push?

3. **Transition Animations**: How to integrate screen transition animations?

4. **Error Handling**: What happens if push/pop fails mid-operation?

## Summary

This design provides:
- **Type-safe screen results** with generics
- **Stack-based navigation** with push/pop/switch
- **Modal support** for dialogs and overlays
- **Async await pattern** for screen results
- **Mode support** for multi-context apps
- **Installed screens** for lazy instantiation
