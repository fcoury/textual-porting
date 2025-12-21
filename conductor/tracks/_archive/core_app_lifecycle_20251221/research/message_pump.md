# Message Pump Analysis

## Overview

The MessagePump is the base class for all objects that process messages in Python Textual. This includes App, Screen, and Widget. It implements an async message queue for handling events and messages.

## Key Components

### 1. The MessagePump Class

```python
# message_pump.py:115-144
class MessagePump(metaclass=_MessagePumpMeta):
    """Base class which supplies a message pump."""

    def __init__(self, parent: MessagePump | None = None) -> None:
        self._parent = parent              # Parent in DOM tree
        self._running: bool = False        # Is processing messages
        self._closing: bool = False        # In process of closing
        self._closed: bool = False         # Fully closed
        self._disabled_messages: set[type[Message]] = set()
        self._pending_message: Message | None = None
        self._task: Task | None = None     # Asyncio task
        self._timers: WeakSet[Timer] = WeakSet()
        self._last_idle: float = time()
        self._max_idle: float | None = None
        self._is_mounted = False
        self._next_callbacks: list[events.Callback] = []
        self._thread_id: int = threading.get_ident()
        self.message_signal: Signal[Message] = Signal(self, "messages")
```

### 2. The Message Queue

```python
# message_pump.py:146-148
@cached_property
def _message_queue(self) -> Queue[Message | None]:
    return Queue()
```

The queue is:
- A custom `Queue` implementation (not asyncio.Queue)
- Uses `None` as a sentinel to signal closure
- Lazily created via cached_property

### 3. Message Class

```python
# message.py:23-57
class Message:
    """Base class for a message."""

    __slots__ = [
        "_sender",         # Who sent the message
        "time",            # When it was created
        "_forwarded",      # Was it forwarded
        "_no_default_action",  # Prevent default handlers
        "_stop_propagation",   # Stop bubbling
        "_prevent",        # Set of prevented message types
    ]

    bubble: ClassVar[bool] = True      # Will bubble up DOM
    verbose: ClassVar[bool] = False    # Log verbosity
    no_dispatch: ClassVar[bool] = False  # Skip dispatch
    namespace: ClassVar[str] = ""      # Handler namespace
    handler_name: ClassVar[str]        # Auto-generated: on_<message_name>
```

### 4. Posting Messages

```python
# message_pump.py:851-879
def post_message(self, message: Message) -> bool:
    """Posts a message on to this widget's queue."""
    if self._closing or self._closed:
        return False
    if not self.check_message_enabled(message):
        return False
    # Copy prevented message types to the message
    message._prevent.update(self._get_prevented_messages())
    if self._thread_id != threading.get_ident() and self.app._loop is not None:
        # Thread-safe posting
        loop = self.app._loop
        loop.call_soon_threadsafe(self._message_queue.put_nowait, message)
    else:
        self._message_queue.put_nowait(message)
    return True
```

Key points:
- Returns False if pump is closing/closed
- Checks if message type is disabled
- Copies prevented message types
- Thread-safe via `call_soon_threadsafe`

### 5. Message Processing Loop

```python
# message_pump.py:625-684
async def _process_messages_loop(self) -> None:
    """Process messages until the queue is closed."""
    while not self._closed:
        try:
            message = await self._get_message()
        except MessagePumpClosed:
            break

        # Combine pending messages that supersede this one
        while not (self._closed or self._closing):
            pending = self._peek_message()
            if pending is None or not message.can_replace(pending):
                break
            message = await self._get_message()

        try:
            await self._dispatch_message(message)
        except CancelledError:
            raise
        except Exception as error:
            self.app._handle_exception(error)
            break
        finally:
            self.message_signal.publish(message)
            self._message_queue.task_done()

            # Insert idle events when queue is empty
            if self._message_queue.empty() or (time expired):
                event = events.Idle()
                for _cls, method in self._get_dispatch_methods("on_idle", event):
                    await invoke(method, event)
                await self._flush_next_callbacks()
```

### 6. Message Dispatch

```python
# message_pump.py:698-731
async def _dispatch_message(self, message: Message) -> None:
    """Dispatch a message received from the message queue."""
    if message.no_dispatch:
        return

    # Allow message hooks to intercept
    message_hook = message_hook_context_var.get()
    message_hook(message)

    with self.prevent(*message._prevent):
        # Events vs Messages handled separately
        if isinstance(message, Event):
            await self.on_event(message)
        else:
            await self._on_message(message)
        if self._next_callbacks:
            await self._flush_next_callbacks()
```

### 7. Handler Resolution (MRO Walk)

```python
# message_pump.py:734-791
def _get_dispatch_methods(
    self, method_name: str, message: Message
) -> Iterable[tuple[type, Callable[[Message], Awaitable]]]:
    """Gets handlers from the MRO"""
    methods_dispatched: set[Callable] = set()
    message_mro = [_type for _type in message.__class__.__mro__ if issubclass(_type, Message)]

    for cls in self.__class__.__mro__:
        if message._no_default_action:
            break
        # Try decorated handlers first (@on decorator)
        decorated_handlers = cls.__dict__.get("_decorated_handlers")
        if decorated_handlers:
            for message_class in message_mro:
                handlers = decorated_handlers.get(message_class, [])
                for method, selectors in handlers:
                    # Check CSS selectors if present
                    if match(selector, node):
                        yield cls, method.__get__(self, cls)

        # Fall back to naming convention: on_<message_name> or _on_<message_name>
        method = cls.__dict__.get(f"_{method_name}") or cls.__dict__.get(method_name)
        if method is not None and not getattr(method, "_textual_on", None):
            yield cls, method.__get__(self, cls)
```

### 8. Message Bubbling

```python
# message_pump.py:801-830
async def _on_message(self, message: Message) -> None:
    """Called to process a message."""
    handler_name = message.handler_name

    # Look through the MRO to find a handler
    for cls, method in self._get_dispatch_methods(handler_name, message):
        await invoke(method, message)

    # Bubble messages up the DOM (if enabled on the message)
    if message.bubble and self._parent and not message._stop_propagation:
        if message._sender is not None and message._sender == self._parent:
            message.stop()  # Stop after parent
        if self.is_parent_active and self.is_attached:
            message._bubble_to(self._parent)

# message.py:151-158
def _bubble_to(self, widget: MessagePump) -> None:
    """Bubble to a widget (typically the parent)."""
    self._no_default_action = False
    widget.post_message(self)
```

### 9. Lifecycle Messages (Compose, Mount)

```python
# message_pump.py:579-604
async def _pre_process(self) -> bool:
    """Procedure to run before processing messages."""
    try:
        # Dispatch compose and mount messages first
        await self._dispatch_message(events.Compose())
        if self._prevented_messages_on_mount:
            with self.prevent(*self._prevented_messages_on_mount):
                await self._dispatch_message(events.Mount())
        else:
            await self._dispatch_message(events.Mount())
        self._post_mount()
    except Exception as error:
        self.app._handle_exception(error)
        return False
    finally:
        self._mounted_event.set()
        self._is_mounted = True
    return True
```

### 10. Scheduling Callbacks

```python
# Call after all messages processed
def call_later(self, callback, *args, **kwargs) -> bool:
    message = events.Callback(callback=partial(callback, *args, **kwargs))
    return self.post_message(message)

# Call immediately after current message
def call_next(self, callback, *args, **kwargs) -> None:
    callback_message = events.Callback(callback=partial(callback, *args, **kwargs))
    callback_message._prevent.update(self._get_prevented_messages())
    self._next_callbacks.append(callback_message)
    self.check_idle()

# Call after screen refresh
def call_after_refresh(self, callback, *args, **kwargs) -> bool:
    message = messages.InvokeLater(partial(callback, *args, **kwargs))
    return self.post_message(message)
```

### 11. Timers

```python
# message_pump.py:369-440
def set_timer(self, delay: float, callback=None, *, name=None, pause=False) -> Timer:
    """Call a function after a delay."""
    timer = Timer(self, delay, name=name, callback=callback, repeat=0, pause=pause)
    timer._start()
    self._timers.add(timer)
    return timer

def set_interval(self, interval: float, callback=None, *, name=None, repeat=0, pause=False) -> Timer:
    """Call a function at periodic intervals."""
    timer = Timer(self, interval, name=name, callback=callback, repeat=repeat, pause=pause)
    timer._start()
    self._timers.add(timer)
    return timer
```

### 12. Preventing Messages

```python
# message_pump.py:189-208
@contextmanager
def prevent(self, *message_types: type[Message]) -> Generator[None, None, None]:
    """A context manager to temporarily prevent message types from being posted."""
    if message_types:
        prevent_stack = self._prevent_message_types_stack
        prevent_stack.append(prevent_stack[-1].union(message_types))
        try:
            yield
        finally:
            prevent_stack.pop()
    else:
        yield

# Usage
with self.prevent(Input.Changed):
    input.value = "foo"  # Won't trigger Input.Changed
```

### 13. The @on Decorator

```python
# Stored via metaclass in _decorated_handlers
@on(Button.Pressed, "#submit")
async def handle_submit(self, event: Button.Pressed) -> None:
    pass

# Metaclass populates _decorated_handlers:
# { Button.Pressed: [(handle_submit, {"control": ("#submit",)})] }
```

## Handler Naming Convention

Messages auto-generate handler names:

```python
# message.py:62-86
def __init_subclass__(cls, bubble=True, verbose=False, no_dispatch=False, namespace=None):
    # Auto-generate handler_name
    if namespace is not None:
        cls.namespace = namespace
        name = f"{namespace}_{camel_to_snake(cls.__name__)}"
    else:
        qualname = cls.__qualname__.rsplit("<locals>.", 1)[-1]
        namespace = qualname.rsplit(".", 2)[-2:]
        name = "_".join(camel_to_snake(part) for part in namespace)
    cls.handler_name = f"on_{name}"

# Examples:
# Button.Pressed -> "on_button_pressed"
# Input.Changed -> "on_input_changed"
# Custom.MyMessage -> "on_custom_my_message"
```

## Message Flow

```
1. post_message(msg)
   -> Queue.put_nowait(msg)

2. _process_messages_loop()
   -> message = await Queue.get()
   -> await _dispatch_message(message)

3. _dispatch_message(message)
   -> if Event: await on_event(message)
   -> else: await _on_message(message)

4. _on_message(message)
   -> for handler in _get_dispatch_methods():
        await invoke(handler, message)
   -> if message.bubble:
        message._bubble_to(self._parent)

5. Bubbling continues until:
   - message.stop() called
   - No more parents
   - Not attached to DOM
```

## Rust Implementation Considerations

### Channel-Based Approach (Tokio)

```rust
use tokio::sync::mpsc;

struct MessagePump {
    sender: mpsc::UnboundedSender<Message>,
    receiver: Option<mpsc::UnboundedReceiver<Message>>,
    parent: Option<Weak<RefCell<dyn MessagePump>>>,
    running: bool,
    closing: bool,
    disabled_messages: HashSet<TypeId>,
    timers: Vec<Timer>,
    next_callbacks: Vec<Box<dyn FnOnce()>>,
}

impl MessagePump {
    pub fn post_message(&self, message: Message) -> bool {
        if self.closing {
            return false;
        }
        self.sender.send(message).is_ok()
    }

    pub async fn process_messages(&mut self) {
        while let Some(message) = self.receiver.as_mut().unwrap().recv().await {
            self.dispatch_message(message).await;
        }
    }
}
```

### Message Trait

```rust
pub trait Message: Any + Send {
    fn bubble(&self) -> bool { true }
    fn handler_name(&self) -> &'static str;
    fn stop_propagation(&mut self);
    fn prevent_default(&mut self);
}

// Auto-derive handler name
#[derive(Message)]
#[handler_name = "on_button_pressed"]
pub struct ButtonPressed {
    pub button: WidgetId,
}
```

### Handler Resolution

```rust
trait MessageHandler<M: Message> {
    fn handle(&mut self, message: &M) -> impl Future<Output = ()>;
}

// Registry of handlers per type
struct HandlerRegistry {
    handlers: HashMap<TypeId, Vec<Box<dyn Fn(&mut dyn Any, &dyn Message)>>>,
}
```

### Bubbling

```rust
async fn dispatch_message(&mut self, mut message: Box<dyn Message>) {
    // Find and call handlers
    if let Some(handlers) = self.handlers.get(&message.type_id()) {
        for handler in handlers {
            handler(self, &*message);
            if message.is_default_prevented() {
                break;
            }
        }
    }

    // Bubble to parent
    if message.bubble() && !message.is_stopped() {
        if let Some(parent) = self.parent.upgrade() {
            parent.borrow_mut().post_message(message);
        }
    }
}
```

### Callback Scheduling

```rust
impl MessagePump {
    pub fn call_later<F>(&self, callback: F) -> bool
    where
        F: FnOnce() + Send + 'static,
    {
        self.post_message(Box::new(CallbackMessage::new(callback)))
    }

    pub fn call_next<F>(&mut self, callback: F)
    where
        F: FnOnce() + 'static,
    {
        self.next_callbacks.push(Box::new(callback));
    }
}
```

## Key Requirements for Rust Port

1. **Async Message Queue**: Tokio mpsc channel
2. **Handler Resolution**: MRO-like trait dispatch or registry
3. **Bubbling**: Parent traversal with stop/prevent
4. **Thread Safety**: Send + Sync or proper synchronization
5. **Lifecycle Events**: Compose, Mount, Unmount ordering
6. **Timer Integration**: Tokio timers with callbacks
7. **Prevent Context**: Stack-based message prevention

## Summary

| Python Concept | Rust Equivalent |
|---------------|-----------------|
| `Queue` | `mpsc::UnboundedSender/Receiver` |
| `post_message()` | `sender.send()` |
| `_process_messages_loop` | `while recv().await` |
| Handler MRO walk | Trait dispatch or handler registry |
| `message.bubble` | Trait method |
| `message.stop()` | Mutable flag |
| `prevent(*types)` | Context/scope guard |
| `call_later/call_next` | Callback messages |
| `set_timer/set_interval` | `tokio::time::interval` |
