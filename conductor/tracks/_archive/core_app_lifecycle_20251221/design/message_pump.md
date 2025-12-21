# Message Pump Architecture Design

## Overview

The message pump is the async event processing system that drives all widget communication in textual-rs. Every App, Screen, and Widget has its own message queue, and messages bubble up through the DOM tree. This design uses Tokio channels for async message passing.

## Goals

1. **Async-First**: Built on Tokio for efficient async message processing
2. **Type-Safe Messages**: Compile-time checked message types
3. **Bubbling**: Messages propagate up the widget tree
4. **Handler Resolution**: Clean handler dispatch via traits
5. **Thread Safety**: Safe cross-thread message posting

## Design Decisions

### 1. Tokio MPSC Channels

Each message pump owns an unbounded MPSC channel:

```rust
use tokio::sync::mpsc;

pub struct MessagePump {
    sender: mpsc::UnboundedSender<Box<dyn Message>>,
    receiver: mpsc::UnboundedReceiver<Box<dyn Message>>,
}
```

**Why unbounded?**
- Matches Python Textual behavior (no backpressure)
- Prevents deadlocks in widget tree traversal
- Message coalescing handles high-frequency updates

### 2. Message Trait

```rust
use std::any::Any;

/// Base trait for all messages in the system.
pub trait Message: Any + Send + Sync + std::fmt::Debug {
    /// Whether this message bubbles up to parent widgets.
    fn bubble(&self) -> bool { true }

    /// The handler method name (e.g., "on_button_pressed").
    fn handler_name(&self) -> &'static str;

    /// Type name for debugging.
    fn type_name(&self) -> &'static str {
        std::any::type_name::<Self>()
    }

    /// Downcast to concrete type.
    fn as_any(&self) -> &dyn Any;
    fn as_any_mut(&mut self) -> &mut dyn Any;
}
```

### 3. Message Metadata

```rust
/// Mutable state carried with each message.
pub struct MessageMeta {
    /// The widget that sent this message.
    pub sender: WidgetId,
    /// When the message was created.
    pub time: Instant,
    /// Whether bubbling should stop.
    pub stop_propagation: bool,
    /// Whether default handling should be skipped.
    pub prevent_default: bool,
    /// Message types to prevent (inherited from sender context).
    pub prevented_types: HashSet<TypeId>,
}

impl MessageMeta {
    pub fn new(sender: WidgetId) -> Self {
        Self {
            sender,
            time: Instant::now(),
            stop_propagation: false,
            prevent_default: false,
            prevented_types: HashSet::new(),
        }
    }

    /// Stop this message from bubbling further.
    pub fn stop(&mut self) {
        self.stop_propagation = true;
    }

    /// Prevent default handlers from running.
    pub fn prevent_default(&mut self) {
        self.prevent_default = true;
    }
}
```

### 4. Message Derive Macro

```rust
/// Derive macro for creating messages.
/// Generates handler_name and trait implementations.
#[derive(Debug, Message)]
#[message(bubble = true)]
pub struct ButtonPressed {
    pub button_id: WidgetId,
}

// Generated:
impl Message for ButtonPressed {
    fn bubble(&self) -> bool { true }
    fn handler_name(&self) -> &'static str { "on_button_pressed" }
    fn as_any(&self) -> &dyn Any { self }
    fn as_any_mut(&mut self) -> &mut dyn Any { self }
}
```

## Core Types

### MessagePump

```rust
use tokio::sync::mpsc;
use std::collections::HashSet;
use std::any::TypeId;

/// The message processing engine for a widget.
pub struct MessagePump {
    /// Channel sender (cloneable for external posting).
    sender: mpsc::UnboundedSender<MessageEnvelope>,
    /// Channel receiver (owned by processor).
    receiver: mpsc::UnboundedReceiver<MessageEnvelope>,
    /// Parent pump for bubbling (weak reference to avoid cycles).
    parent: Option<mpsc::UnboundedSender<MessageEnvelope>>,
    /// Widget ID for this pump.
    widget_id: WidgetId,
    /// Is the pump running.
    running: bool,
    /// Is the pump closing.
    closing: bool,
    /// Disabled message types.
    disabled_messages: HashSet<TypeId>,
    /// Callbacks to run after current message.
    next_callbacks: Vec<Box<dyn FnOnce() + Send>>,
    /// Stack of prevented message types.
    prevent_stack: Vec<HashSet<TypeId>>,
}

/// Wrapper around a message with its metadata.
pub struct MessageEnvelope {
    pub message: Box<dyn Message>,
    pub meta: MessageMeta,
}
```

### MessagePump Implementation

```rust
impl MessagePump {
    pub fn new(widget_id: WidgetId) -> Self {
        let (sender, receiver) = mpsc::unbounded_channel();
        Self {
            sender,
            receiver,
            parent: None,
            widget_id,
            running: false,
            closing: false,
            disabled_messages: HashSet::new(),
            next_callbacks: Vec::new(),
            prevent_stack: vec![HashSet::new()],
        }
    }

    /// Get a sender for posting messages to this pump.
    pub fn sender(&self) -> mpsc::UnboundedSender<MessageEnvelope> {
        self.sender.clone()
    }

    /// Set the parent pump for message bubbling.
    pub fn set_parent(&mut self, parent: mpsc::UnboundedSender<MessageEnvelope>) {
        self.parent = Some(parent);
    }

    /// Post a message to this pump's queue.
    pub fn post_message<M: Message + 'static>(&self, message: M) -> bool {
        if self.closing {
            return false;
        }

        // Check if message type is disabled
        let type_id = TypeId::of::<M>();
        if self.disabled_messages.contains(&type_id) {
            return false;
        }

        let envelope = MessageEnvelope {
            message: Box::new(message),
            meta: MessageMeta::new(self.widget_id),
        };

        self.sender.send(envelope).is_ok()
    }

    /// Post a boxed message (for bubbling).
    pub fn post_boxed(&self, envelope: MessageEnvelope) -> bool {
        if self.closing {
            return false;
        }
        self.sender.send(envelope).is_ok()
    }
}
```

### Message Processing Loop

```rust
impl MessagePump {
    /// Run the message processing loop.
    pub async fn run<H: MessageHandler>(&mut self, handler: &mut H) {
        self.running = true;

        // Pre-process: dispatch Compose and Mount events
        self.dispatch_lifecycle_events(handler).await;

        // Main loop
        while let Some(envelope) = self.receiver.recv().await {
            if self.closing {
                break;
            }

            // Dispatch the message
            self.dispatch_message(envelope, handler).await;

            // Flush next callbacks
            self.flush_next_callbacks().await;

            // Check for idle (queue empty)
            if self.receiver.is_empty() {
                handler.on_idle().await;
            }
        }

        self.running = false;
    }

    async fn dispatch_message<H: MessageHandler>(
        &mut self,
        mut envelope: MessageEnvelope,
        handler: &mut H,
    ) {
        // Check if message type is prevented
        let type_id = (*envelope.message).type_id();
        if self.current_prevented().contains(&type_id) {
            return;
        }

        // Dispatch to handler
        let handled = handler.handle_message(&mut envelope).await;

        // Bubble to parent if not stopped
        if envelope.message.bubble()
            && !envelope.meta.stop_propagation
            && !envelope.meta.prevent_default
        {
            if let Some(parent) = &self.parent {
                let _ = parent.send(envelope);
            }
        }
    }

    fn current_prevented(&self) -> &HashSet<TypeId> {
        self.prevent_stack.last().unwrap()
    }

    async fn flush_next_callbacks(&mut self) {
        let callbacks = std::mem::take(&mut self.next_callbacks);
        for callback in callbacks {
            callback();
        }
    }
}
```

## Handler Trait

```rust
use async_trait::async_trait;

/// Trait for handling messages.
#[async_trait]
pub trait MessageHandler {
    /// Handle a message. Returns true if handled.
    async fn handle_message(&mut self, envelope: &mut MessageEnvelope) -> bool;

    /// Called when the message queue is empty.
    async fn on_idle(&mut self) {}

    /// Called during mount phase.
    async fn on_mount(&mut self) {}

    /// Called during unmount phase.
    async fn on_unmount(&mut self) {}
}
```

### Handler Implementation Pattern

Widgets implement message handling via the `on_*` method convention:

```rust
#[derive(Debug)]
struct MyWidget {
    count: i32,
}

#[async_trait]
impl MessageHandler for MyWidget {
    async fn handle_message(&mut self, envelope: &mut MessageEnvelope) -> bool {
        let message = &envelope.message;

        // Try each handler based on type
        if let Some(msg) = message.as_any().downcast_ref::<ButtonPressed>() {
            self.on_button_pressed(msg, &mut envelope.meta).await;
            return true;
        }

        if let Some(msg) = message.as_any().downcast_ref::<InputChanged>() {
            self.on_input_changed(msg, &mut envelope.meta).await;
            return true;
        }

        false
    }
}

impl MyWidget {
    async fn on_button_pressed(&mut self, msg: &ButtonPressed, meta: &mut MessageMeta) {
        self.count += 1;
        // meta.stop() to prevent bubbling
    }

    async fn on_input_changed(&mut self, msg: &InputChanged, meta: &mut MessageMeta) {
        // Handle input change
    }
}
```

### Handler Macro (Future Enhancement)

```rust
// Macro to simplify handler implementation
message_handlers! {
    impl MessageHandler for MyWidget {
        async fn on_button_pressed(&mut self, msg: &ButtonPressed) {
            self.count += 1;
        }

        async fn on_input_changed(&mut self, msg: &InputChanged) {
            // Handle input
        }
    }
}
```

## Lifecycle Events

```rust
/// Compose event - sent to build widget tree.
#[derive(Debug, Message)]
#[message(bubble = false)]
pub struct Compose;

/// Mount event - sent after widget is added to DOM.
#[derive(Debug, Message)]
#[message(bubble = false)]
pub struct Mount;

/// Unmount event - sent before widget is removed.
#[derive(Debug, Message)]
#[message(bubble = false)]
pub struct Unmount;

/// Idle event - sent when message queue is empty.
#[derive(Debug, Message)]
#[message(bubble = false)]
pub struct Idle;
```

### Lifecycle Dispatch

```rust
impl MessagePump {
    async fn dispatch_lifecycle_events<H: MessageHandler>(&mut self, handler: &mut H) {
        // Compose
        let compose = MessageEnvelope {
            message: Box::new(Compose),
            meta: MessageMeta::new(self.widget_id),
        };
        self.dispatch_message(compose, handler).await;

        // Mount
        handler.on_mount().await;
        let mount = MessageEnvelope {
            message: Box::new(Mount),
            meta: MessageMeta::new(self.widget_id),
        };
        self.dispatch_message(mount, handler).await;
    }
}
```

## Callback Scheduling

```rust
impl MessagePump {
    /// Schedule a callback to run after the queue is processed.
    pub fn call_later<F>(&self, callback: F) -> bool
    where
        F: FnOnce() + Send + 'static,
    {
        #[derive(Debug)]
        struct CallbackMessage(Option<Box<dyn FnOnce() + Send>>);

        impl Message for CallbackMessage {
            fn bubble(&self) -> bool { false }
            fn handler_name(&self) -> &'static str { "on_callback" }
            fn as_any(&self) -> &dyn Any { self }
            fn as_any_mut(&mut self) -> &mut dyn Any { self }
        }

        self.post_message(CallbackMessage(Some(Box::new(callback))))
    }

    /// Schedule a callback to run immediately after current message.
    pub fn call_next<F>(&mut self, callback: F)
    where
        F: FnOnce() + Send + 'static,
    {
        self.next_callbacks.push(Box::new(callback));
    }
}
```

## Message Prevention

```rust
impl MessagePump {
    /// Prevent message types within a scope.
    pub fn prevent<F, R>(&mut self, types: &[TypeId], f: F) -> R
    where
        F: FnOnce() -> R,
    {
        let mut prevented = self.current_prevented().clone();
        prevented.extend(types.iter().copied());
        self.prevent_stack.push(prevented);

        let result = f();

        self.prevent_stack.pop();
        result
    }

    /// Disable a message type permanently for this pump.
    pub fn disable_message<M: Message + 'static>(&mut self) {
        self.disabled_messages.insert(TypeId::of::<M>());
    }

    /// Re-enable a disabled message type.
    pub fn enable_message<M: Message + 'static>(&mut self) {
        self.disabled_messages.remove(&TypeId::of::<M>());
    }
}
```

## Timer Integration

```rust
use tokio::time::{interval, Duration, Interval};

/// A repeating or one-shot timer.
pub struct Timer {
    pump_sender: mpsc::UnboundedSender<MessageEnvelope>,
    widget_id: WidgetId,
    interval: Duration,
    repeat: bool,
    paused: bool,
    handle: Option<tokio::task::JoinHandle<()>>,
}

impl Timer {
    pub fn new(
        pump_sender: mpsc::UnboundedSender<MessageEnvelope>,
        widget_id: WidgetId,
        delay: Duration,
        repeat: bool,
    ) -> Self {
        Self {
            pump_sender,
            widget_id,
            interval: delay,
            repeat,
            paused: false,
            handle: None,
        }
    }

    pub fn start(&mut self) {
        let sender = self.pump_sender.clone();
        let widget_id = self.widget_id;
        let interval_duration = self.interval;
        let repeat = self.repeat;

        self.handle = Some(tokio::spawn(async move {
            tokio::time::sleep(interval_duration).await;
            loop {
                let envelope = MessageEnvelope {
                    message: Box::new(TimerTick),
                    meta: MessageMeta::new(widget_id),
                };
                if sender.send(envelope).is_err() {
                    break;
                }
                if !repeat {
                    break;
                }
                tokio::time::sleep(interval_duration).await;
            }
        }));
    }

    pub fn pause(&mut self) {
        self.paused = true;
        if let Some(handle) = self.handle.take() {
            handle.abort();
        }
    }

    pub fn resume(&mut self) {
        if self.paused {
            self.paused = false;
            self.start();
        }
    }
}

impl MessagePump {
    /// Set a one-shot timer.
    pub fn set_timer(&self, delay: Duration) -> Timer {
        let mut timer = Timer::new(self.sender.clone(), self.widget_id, delay, false);
        timer.start();
        timer
    }

    /// Set a repeating interval timer.
    pub fn set_interval(&self, interval: Duration) -> Timer {
        let mut timer = Timer::new(self.sender.clone(), self.widget_id, interval, true);
        timer.start();
        timer
    }
}
```

## Thread-Safe Posting

```rust
/// A thread-safe handle for posting messages from any thread.
#[derive(Clone)]
pub struct MessageSender {
    sender: mpsc::UnboundedSender<MessageEnvelope>,
    widget_id: WidgetId,
}

impl MessageSender {
    pub fn new(sender: mpsc::UnboundedSender<MessageEnvelope>, widget_id: WidgetId) -> Self {
        Self { sender, widget_id }
    }

    /// Post a message from any thread.
    pub fn post<M: Message + 'static>(&self, message: M) -> bool {
        let envelope = MessageEnvelope {
            message: Box::new(message),
            meta: MessageMeta::new(self.widget_id),
        };
        self.sender.send(envelope).is_ok()
    }
}
```

## Integration with Widget

```rust
pub struct Widget {
    id: WidgetId,
    pump: MessagePump,
    // ... other fields
}

impl Widget {
    pub fn new(id: WidgetId) -> Self {
        Self {
            id,
            pump: MessagePump::new(id),
        }
    }

    pub fn post_message<M: Message + 'static>(&self, message: M) -> bool {
        self.pump.post_message(message)
    }

    pub fn sender(&self) -> MessageSender {
        MessageSender::new(self.pump.sender(), self.id)
    }
}
```

## Example Usage

### Defining Custom Messages

```rust
#[derive(Debug, Message)]
pub struct CounterIncremented {
    pub new_value: i32,
}

#[derive(Debug, Message)]
#[message(bubble = false)]
pub struct RefreshRequested;
```

### Handling Messages

```rust
struct Counter {
    count: i32,
    pump: MessagePump,
}

#[async_trait]
impl MessageHandler for Counter {
    async fn handle_message(&mut self, envelope: &mut MessageEnvelope) -> bool {
        if let Some(_) = envelope.message.as_any().downcast_ref::<ButtonPressed>() {
            self.count += 1;
            self.pump.post_message(CounterIncremented { new_value: self.count });
            return true;
        }
        false
    }
}
```

### Parent Listening to Child

```rust
struct App {
    pump: MessagePump,
}

#[async_trait]
impl MessageHandler for App {
    async fn handle_message(&mut self, envelope: &mut MessageEnvelope) -> bool {
        // Bubbled from child Counter
        if let Some(msg) = envelope.message.as_any().downcast_ref::<CounterIncremented>() {
            println!("Counter is now: {}", msg.new_value);
            return true;
        }
        false
    }
}
```

## Comparison with Python Textual

| Python Textual | Rust Implementation |
|---------------|---------------------|
| `asyncio.Queue` | `tokio::sync::mpsc::unbounded_channel` |
| `post_message(msg)` | `pump.post_message(msg)` |
| `_process_messages_loop()` | `pump.run(handler).await` |
| `Message` class | `Message` trait + derive macro |
| `message.stop()` | `meta.stop()` |
| `message.prevent_default()` | `meta.prevent_default()` |
| `on_button_pressed` method | `handle_message` dispatch |
| `@on(Message)` decorator | Future: `#[on(Message)]` attribute |
| `call_later()` | `pump.call_later(callback)` |
| `call_next()` | `pump.call_next(callback)` |
| `set_timer()` | `pump.set_timer(duration)` |
| `prevent(*types)` context | `pump.prevent(&types, \|\| { ... })` |

## Implementation Plan

### Phase 1: Core Types
- [ ] Define `Message` trait
- [ ] Define `MessageMeta` struct
- [ ] Define `MessageEnvelope` struct
- [ ] Create `Message` derive macro

### Phase 2: MessagePump
- [ ] Implement `MessagePump` struct
- [ ] Implement `post_message()`
- [ ] Implement message processing loop
- [ ] Implement `MessageHandler` trait

### Phase 3: Bubbling & Prevention
- [ ] Implement parent reference and bubbling
- [ ] Implement `stop_propagation`
- [ ] Implement `prevent_default`
- [ ] Implement `prevent()` scope

### Phase 4: Scheduling
- [ ] Implement `call_later()`
- [ ] Implement `call_next()`
- [ ] Implement `Timer` struct
- [ ] Implement `set_timer()` and `set_interval()`

### Phase 5: Lifecycle
- [ ] Implement `Compose`, `Mount`, `Unmount` events
- [ ] Implement lifecycle dispatch order
- [ ] Implement `Idle` event

## Open Questions

1. **Message Coalescing**: Should we coalesce duplicate messages like Python does with `can_replace()`?

2. **Handler Registry**: Should handlers be registered dynamically or rely on trait dispatch?

3. **Async Handlers**: Should all handlers be async, or support sync handlers too?

4. **Error Handling**: How should handler errors be propagated?

## Summary

This design provides:
- **Async message processing** via Tokio channels
- **Type-safe messages** with derive macro
- **Bubbling** through parent references
- **Lifecycle events** matching Python Textual
- **Callback scheduling** for deferred execution
- **Timer support** for delayed/repeated messages
