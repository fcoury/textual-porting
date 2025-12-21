# Screen Stack Analysis

## Overview

Python Textual uses a screen stack to manage multiple screens, allowing screens to be pushed, popped, and switched. Screens can return results to their callers via callbacks.

## Key Components

### 1. Screen Stack Structure

```python
# app.py:613-614
self._screen_stacks: dict[str, list[Screen[Any]]] = {self.DEFAULT_MODE: []}
"""A stack of screens per mode."""
self._current_mode: str = self.DEFAULT_MODE
"""The current mode the app is in."""
```

The app maintains:
- A dictionary of screen stacks, one per "mode"
- A current mode string
- The current screen is always `_screen_stack[-1]`

### 2. Screen Class

```python
# screen.py:145-290
class Screen(Generic[ScreenResultType], Widget):
    """The base class for screens."""

    AUTO_FOCUS: ClassVar[str | None] = None
    """A selector to determine what to focus automatically."""

    CSS: ClassVar[str] = ""
    CSS_PATH: ClassVar[CSSPathType | None] = None
    TITLE: ClassVar[str | None] = None
    SUB_TITLE: ClassVar[str | None] = None

    focused: Reactive[Widget | None] = Reactive(None)
    title: Reactive[str | None] = Reactive(None, compute=False)
    sub_title: Reactive[str | None] = Reactive(None, compute=False)
    maximized: Reactive[Widget | None] = Reactive(None, layout=True)

    BINDINGS = [
        Binding("tab", "app.focus_next", "Focus Next", show=False),
        Binding("shift+tab", "app.focus_previous", "Focus Previous", show=False),
        Binding("ctrl+c", "screen.copy_text", "Copy selected text", show=False),
    ]

    def __init__(self, name=None, id=None, classes=None):
        self._modal = False
        super().__init__(name=name, id=id, classes=classes)
        self._compositor = Compositor()
        self._dirty_widgets: set[Widget] = set()
        self._result_callbacks: list[ResultCallback[ScreenResultType | None]] = []
```

### 3. Push Screen

```python
# app.py:2775-2834
def push_screen(
    self,
    screen: Screen[ScreenResultType] | str,
    callback: ScreenResultCallbackType[ScreenResultType] | None = None,
    wait_for_dismiss: bool = False,
) -> AwaitMount | asyncio.Future[ScreenResultType]:
    """Push a new screen on the screen stack."""

    # Create future for await result
    future: asyncio.Future[ScreenResultType] = asyncio.Future()

    # Suspend current screen
    self.app.capture_mouse(None)
    if self._screen_stack:
        self.screen.post_message(events.ScreenSuspend())
        self.screen.refresh()

    # Get or create the screen
    next_screen, await_mount = self._get_screen(screen)

    # Register callback for dismiss result
    next_screen._push_result_callback(message_pump, callback, future)

    # Load CSS and append to stack
    self._load_screen_css(next_screen)
    next_screen._update_auto_focus()
    self._screen_stack.append(next_screen)

    # Resume the new screen
    next_screen.post_message(events.ScreenResume())

    if wait_for_dismiss:
        return future  # For workers that need to await result
    else:
        return await_mount
```

### 4. Pop Screen

```python
# app.py:2957-2979
def pop_screen(self) -> AwaitComplete:
    """Pop the current screen from the stack, switch to previous screen."""

    screen_stack = self._screen_stack
    if len(screen_stack) <= 1:
        raise ScreenStackError("Can't pop screen; there must be at least one screen")

    previous_screen = screen_stack.pop()
    previous_screen._pop_result_callback()
    self.screen.post_message(events.ScreenResume())

    async def do_pop() -> None:
        await self._replace_screen(previous_screen)

    return AwaitComplete(do_pop()).call_next(self)
```

### 5. Switch Screen

```python
# app.py:2863-2893
def switch_screen(self, screen: Screen | str) -> AwaitComplete:
    """Switch to another screen by replacing the top of the stack."""

    next_screen, await_mount = self._get_screen(screen)
    if screen is self.screen or next_screen is self.screen:
        return AwaitComplete.nothing()

    self.app.capture_mouse(None)
    top_screen = self._screen_stack.pop()
    top_screen._pop_result_callback()

    self._load_screen_css(next_screen)
    self._screen_stack.append(next_screen)
    self.screen.post_message(events.ScreenResume())
    self.screen._push_result_callback(self.screen, None)

    async def do_switch() -> None:
        await await_mount()
        await self._replace_screen(top_screen)

    return AwaitComplete(do_switch()).call_next(self)
```

### 6. Screen Dismiss with Results

```python
# screen.py:105-141
class ResultCallback(Generic[ScreenResultType]):
    """Holds the details of a callback."""

    def __init__(
        self,
        requester: MessagePump,
        callback: ScreenResultCallbackType[ScreenResultType] | None,
        future: asyncio.Future[ScreenResultType] | None = None,
    ) -> None:
        self.requester = requester
        self.callback = callback
        self.future = future

    def __call__(self, result: ScreenResultType) -> None:
        """Call the callback, passing the given result."""
        if self.future is not None:
            self.future.set_result(result)
        if self.requester is not None and self.callback is not None:
            self.requester.call_next(self.callback, result)
        self.callback = None

# screen.py:1859-1878
def dismiss(self, result: ScreenResultType | None = None) -> AwaitComplete:
    """Dismiss the screen, optionally with a result.

    Only the active screen may be dismissed.
    """
    if not self.is_active:
        self.log.warning("Can't dismiss inactive screen")

    # Result is passed to the callback registered during push_screen
    ...
```

### 7. Result Callback Stack

```python
# screen.py:1254-1283
def _push_result_callback(
    self,
    requester: MessagePump,
    callback: ScreenResultCallbackType[ScreenResultType] | None,
    future: asyncio.Future[ScreenResultType | None] | None = None,
) -> None:
    """Add a result callback to the screen."""
    self._result_callbacks.append(
        ResultCallback[Optional[ScreenResultType]](requester, callback, future)
    )

def _pop_result_callback(self) -> None:
    """Remove the latest result callback from the stack."""
    self._result_callbacks.pop()
```

### 8. Modal Screens

```python
# screen.py:1974-1999
class ModalScreen(Screen[ScreenResultType]):
    """A screen with bindings that take precedence over the App's key bindings."""

    DEFAULT_CSS = """
    ModalScreen {
        layout: vertical;
        overflow-y: auto;
        background: $background 60%;  # Dim effect
    }
    """

    def __init__(self, name=None, id=None, classes=None):
        super().__init__(name=name, id=id, classes=classes)
        self._modal = True  # Key difference

# screen.py:2002-2011
class SystemModalScreen(ModalScreen[ScreenResultType], inherit_css=False):
    """A variant of ModalScreen for internal use."""
    # Used for system dialogs, command palette, etc.
```

Modal screens:
- Have `_modal = True`
- Get binding priority over app bindings
- Default styling dims the background (60% opacity)

### 9. Screen Events

```python
# Lifecycle events:
events.ScreenResume()   # Screen becomes active
events.ScreenSuspend()  # Screen is covered by another

# Events are posted automatically during push/pop/switch
```

### 10. Installed Screens

```python
# app.py
SCREENS: ClassVar[dict[str, Screen[Any] | Callable[[], Screen[Any]]]] = {}

def install_screen(self, screen: Screen, name: str) -> None:
    """Install a screen for later use."""
    self._installed_screens[name] = screen

def uninstall_screen(self, screen: Screen | str) -> str | None:
    """Uninstall a screen."""
    ...

def is_screen_installed(self, screen: Screen | str) -> bool:
    """Check if a screen is installed."""
    ...
```

Screens can be:
- Pushed by instance: `app.push_screen(MyScreen())`
- Pushed by name: `app.push_screen("settings")` (must be installed first)

### 11. Push Screen Wait Pattern

```python
# app.py:2846-2861
async def push_screen_wait(
    self, screen: Screen[ScreenResultType] | str
) -> ScreenResultType | Any:
    """Push a screen and wait for the result (from Screen.dismiss).

    Note: This method may only be called when running in a worker.
    """
    await self._flush_next_callbacks()
    return await asyncio.shield(self.push_screen(screen, wait_for_dismiss=True))
```

## Screen Stack Flow

```
1. App starts with default screen on stack
   _screen_stack = [DefaultScreen]

2. push_screen(SettingsScreen)
   - DefaultScreen receives ScreenSuspend
   - SettingsScreen added to stack
   - SettingsScreen receives ScreenResume
   _screen_stack = [DefaultScreen, SettingsScreen]

3. SettingsScreen.dismiss("saved")
   - Result callback called with "saved"
   - SettingsScreen removed from stack
   - DefaultScreen receives ScreenResume
   _screen_stack = [DefaultScreen]
```

## Rust Implementation Considerations

### Screen Stack Structure

```rust
struct App {
    screen_stacks: HashMap<String, Vec<Rc<RefCell<Screen>>>>,
    current_mode: String,
}

impl App {
    fn screen_stack(&self) -> &Vec<Rc<RefCell<Screen>>> {
        &self.screen_stacks[&self.current_mode]
    }

    fn screen(&self) -> Rc<RefCell<Screen>> {
        self.screen_stack().last().unwrap().clone()
    }
}
```

### Screen Type

```rust
pub struct Screen<R = ()> {
    widget: WidgetBase,
    modal: bool,
    result_callbacks: Vec<ResultCallback<R>>,
    compositor: Compositor,
    dirty_widgets: HashSet<WidgetId>,
    title: Option<String>,
    sub_title: Option<String>,
}

pub struct ResultCallback<R> {
    requester: Weak<RefCell<dyn MessagePump>>,
    callback: Option<Box<dyn FnOnce(Option<R>)>>,
    future: Option<oneshot::Sender<R>>,
}
```

### Push Screen

```rust
impl App {
    pub fn push_screen<R>(&mut self, screen: Screen<R>, callback: Option<Box<dyn FnOnce(Option<R>)>>) -> AwaitMount {
        // Suspend current screen
        if let Some(current) = self.screen_stack().last() {
            current.borrow_mut().post_message(ScreenSuspend);
        }

        // Setup result callback
        screen.push_result_callback(callback);

        // Add to stack
        self.screen_stack_mut().push(Rc::new(RefCell::new(screen)));

        // Resume new screen
        self.screen().borrow_mut().post_message(ScreenResume);

        AwaitMount::new(self.screen())
    }
}
```

### Pop Screen

```rust
impl App {
    pub fn pop_screen(&mut self) -> Result<AwaitComplete, ScreenStackError> {
        if self.screen_stack().len() <= 1 {
            return Err(ScreenStackError::CantPopLastScreen);
        }

        let popped = self.screen_stack_mut().pop().unwrap();
        popped.borrow_mut().pop_result_callback();

        // Resume previous screen
        self.screen().borrow_mut().post_message(ScreenResume);

        Ok(AwaitComplete::new(async move {
            // Cleanup popped screen
        }))
    }
}
```

### Dismiss with Result

```rust
impl<R> Screen<R> {
    pub fn dismiss(&mut self, result: Option<R>) -> AwaitComplete {
        if let Some(callback) = self.result_callbacks.pop() {
            if let Some(f) = callback.future {
                let _ = f.send(result);
            }
            if let Some(cb) = callback.callback {
                cb(result);
            }
        }

        // Trigger pop
        self.app().borrow_mut().pop_screen()
    }
}
```

### Modal Screens

```rust
pub struct ModalScreen<R = ()> {
    screen: Screen<R>,
}

impl<R> ModalScreen<R> {
    pub fn new() -> Self {
        let mut screen = Screen::new();
        screen.modal = true;
        Self { screen }
    }
}
```

## Key Requirements for Rust Port

1. **Generic Result Type**: `Screen<R>` where R is the dismiss result type
2. **Callback Stack**: Multiple callbacks can be registered per screen
3. **Future Support**: For `wait_for_dismiss` pattern using tokio oneshot
4. **Mode Support**: Multiple independent screen stacks
5. **Modal Flag**: For binding priority
6. **Screen Events**: ScreenResume, ScreenSuspend
7. **Installed Screens**: Name-to-screen mapping

## Summary

| Python Concept | Rust Equivalent |
|---------------|-----------------|
| `_screen_stacks: dict[str, list[Screen]]` | `HashMap<String, Vec<Rc<RefCell<Screen>>>>` |
| `push_screen(screen, callback)` | `push_screen(screen, Option<Box<dyn FnOnce(R)>>)` |
| `Screen[ScreenResultType]` | `Screen<R>` generic |
| `ResultCallback` | Struct with callback + oneshot::Sender |
| `dismiss(result)` | Triggers callback + pop |
| `ModalScreen` | Struct with `modal: bool = true` |
| `push_screen_wait()` | `oneshot::channel` + `.await` |
| `ScreenResume/ScreenSuspend` | Event enum variants |
