# Timer System Design

## Overview

The timer system provides scheduled message delivery for widgets, enabling:
- **One-shot timers**: Execute after a delay (e.g., debouncing, timeouts)
- **Interval timers**: Execute repeatedly (e.g., animations, polling, clocks)

This builds on the existing `MessagePump` infrastructure in `src/pump.rs`.

## Goals

1. **Async-native**: Built on Tokio for efficient async timing
2. **Message-based**: Timers deliver messages through the existing pump system
3. **Controllable**: Timers can be paused, resumed, and stopped
4. **Widget-scoped**: Each timer is associated with a specific widget
5. **RAII cleanup**: Timers automatically stop when dropped

## Existing Infrastructure

The following already exists in `src/pump.rs`:
- `MessagePump` - async message processing engine
- `MessageSender` - thread-safe handle for posting messages
- `PumpMessage` trait - base trait for all messages
- `MessageEnvelope` - wrapper with metadata

## Design

### TimerId

```rust
/// Unique identifier for a timer.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct TimerId(u64);

impl TimerId {
    /// Generate a new unique timer ID.
    pub fn new() -> Self {
        use std::sync::atomic::{AtomicU64, Ordering};
        static COUNTER: AtomicU64 = AtomicU64::new(0);
        Self(COUNTER.fetch_add(1, Ordering::Relaxed))
    }
}

impl Default for TimerId {
    fn default() -> Self {
        Self::new()
    }
}
```

### TimerTick Message

```rust
use std::time::{Duration, Instant};

/// Message sent when a timer fires.
#[derive(Debug, Clone)]
pub struct TimerTick {
    /// Unique identifier for this timer.
    pub timer_id: TimerId,
    /// Number of times this timer has fired (1-indexed).
    pub count: u64,
    /// Time elapsed since the timer was started (measured at delivery time).
    pub elapsed: Duration,
    /// The instant when the timer was started.
    pub started_at: Instant,
}

impl PumpMessage for TimerTick {
    fn bubble(&self) -> bool {
        false // Timer events don't bubble
    }

    fn handler_name(&self) -> &'static str {
        "on_timer"
    }

    fn as_any(&self) -> &dyn Any { self }
    fn as_any_mut(&mut self) -> &mut dyn Any { self }
}
```

### Timer State

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicBool, AtomicU64, Ordering};
use tokio::sync::Notify;

/// Shared state between Timer handle and spawned async task.
struct TimerState {
    /// Whether the timer has been stopped.
    stopped: AtomicBool,
    /// Whether the timer is currently paused.
    paused: AtomicBool,
    /// Whether the async task has finished (for one-shot or channel closed).
    finished: AtomicBool,
    /// Notification for pause/resume state changes.
    pause_notify: Notify,
    /// Tick count, shared with async task.
    tick_count: AtomicU64,
    /// When the timer was started.
    started_at: std::sync::RwLock<Option<Instant>>,
}

impl TimerState {
    fn new() -> Self {
        Self {
            stopped: AtomicBool::new(false),
            paused: AtomicBool::new(false),
            finished: AtomicBool::new(false),
            pause_notify: Notify::new(),
            tick_count: AtomicU64::new(0),
            started_at: std::sync::RwLock::new(None),
        }
    }
}
```

### Repeat Configuration

```rust
use std::num::NonZeroU64;

/// Repeat configuration for interval timers.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Repeat {
    /// Fire once, then finish.
    Once,
    /// Fire indefinitely.
    Infinite,
    /// Fire a fixed number of times.
    Count(NonZeroU64),
}
```

### Timer Struct

```rust
use std::time::Duration;
use tokio::task::JoinHandle;

/// A timer that sends messages at scheduled intervals.
///
/// Timers are RAII: they automatically stop when dropped.
pub struct Timer {
    /// Unique identifier for this timer.
    id: TimerId,
    /// Optional name for debugging.
    name: Option<String>,
    /// Duration between ticks (or delay for one-shot).
    interval: Duration,
    /// Repeat configuration.
    repeat: Repeat,
    /// Shared state with the async task.
    state: Arc<TimerState>,
    /// Handle to the spawned timer task.
    handle: Option<JoinHandle<()>>,
    /// Sender for posting timer messages.
    sender: MessageSender,
}
```

### Timer Implementation

```rust
use std::num::NonZeroU64;
use tokio::time::{interval_at, MissedTickBehavior, Instant as TokioInstant};

impl Timer {
    /// Create a new one-shot timer (does not start automatically).
    pub fn new_oneshot(sender: MessageSender, delay: Duration) -> Self {
        Self {
            id: TimerId::new(),
            name: None,
            interval: delay,
            repeat: Repeat::Once,
            state: Arc::new(TimerState::new()),
            handle: None,
            sender,
        }
    }

    /// Create a new repeating timer (does not start automatically).
    pub fn new_interval(sender: MessageSender, interval: Duration, repeat: Repeat) -> Self {
        Self {
            id: TimerId::new(),
            name: None,
            interval,
            repeat,
            state: Arc::new(TimerState::new()),
            handle: None,
            sender,
        }
    }

    /// Set an optional name for debugging.
    pub fn with_name(mut self, name: impl Into<String>) -> Self {
        self.name = Some(name.into());
        self
    }

    /// Get the timer's unique ID.
    pub fn id(&self) -> TimerId {
        self.id
    }

    /// Get the timer's name, if set.
    pub fn name(&self) -> Option<&str> {
        self.name.as_deref()
    }

    /// Get the time when the timer was started (if started).
    pub fn started_at(&self) -> Option<Instant> {
        *self.state.started_at.read().unwrap()
    }

    /// Get the number of times this timer has fired.
    pub fn tick_count(&self) -> u64 {
        self.state.tick_count.load(Ordering::Relaxed)
    }

    /// Check if the timer is currently running (started and not finished/stopped).
    pub fn is_running(&self) -> bool {
        self.handle.is_some()
            && !self.state.stopped.load(Ordering::Relaxed)
            && !self.state.finished.load(Ordering::Relaxed)
    }

    /// Check if the timer is paused.
    pub fn is_paused(&self) -> bool {
        self.state.paused.load(Ordering::Relaxed)
    }

    /// Check if the timer has finished (one-shot completed or repeat count exhausted).
    pub fn is_finished(&self) -> bool {
        self.state.finished.load(Ordering::Relaxed)
    }

    /// Start the timer.
    pub fn start(&mut self) {
        if let Some(handle) = &self.handle {
            if !handle.is_finished() {
                return; // Already started
            }
            self.handle = None;
        }

        // Reset state for fresh start
        self.state.stopped.store(false, Ordering::Relaxed);
        self.state.paused.store(false, Ordering::Relaxed);
        self.state.finished.store(false, Ordering::Relaxed);
        self.state.tick_count.store(0, Ordering::Relaxed);

        let started_at = Instant::now();
        *self.state.started_at.write().unwrap() = Some(started_at);

        let sender = self.sender.clone();
        let timer_id = self.id;
        let interval_duration = self.interval;
        let repeat = self.repeat;
        let state = Arc::clone(&self.state);

        self.handle = Some(tokio::spawn(async move {
            match repeat {
                Repeat::Once => {
                    Self::run_oneshot(sender, timer_id, interval_duration, state, started_at).await;
                }
                Repeat::Infinite => {
                    Self::run_interval(sender, timer_id, interval_duration, None, state, started_at).await;
                }
                Repeat::Count(max_count) => {
                    Self::run_interval(sender, timer_id, interval_duration, Some(max_count), state, started_at).await;
                }
            }
        }));
    }

    async fn run_oneshot(
        sender: MessageSender,
        timer_id: TimerId,
        delay: Duration,
        state: Arc<TimerState>,
        started_at: Instant,
    ) {
        // Wait for the delay, but check for stop/pause
        let deadline = TokioInstant::now() + delay;

        loop {
            tokio::select! {
                _ = tokio::time::sleep_until(deadline) => {
                    break;
                }
                _ = state.pause_notify.notified() => {
                    // Check if stopped
                    if state.stopped.load(Ordering::Relaxed) {
                        state.finished.store(true, Ordering::Relaxed);
                        return;
                    }
                    // If paused, wait for resume
                    while state.paused.load(Ordering::Relaxed) {
                        state.pause_notify.notified().await;
                        if state.stopped.load(Ordering::Relaxed) {
                            state.finished.store(true, Ordering::Relaxed);
                            return;
                        }
                    }
                }
            }
        }

        // Check if stopped during sleep
        if state.stopped.load(Ordering::Relaxed) {
            state.finished.store(true, Ordering::Relaxed);
            return;
        }

        // Fire the tick
        state.tick_count.fetch_add(1, Ordering::Relaxed);
        let tick = TimerTick {
            timer_id,
            count: 1,
            elapsed: started_at.elapsed(),
            started_at,
        };
        let _ = sender.post(tick);
        state.finished.store(true, Ordering::Relaxed);
    }

    async fn run_interval(
        sender: MessageSender,
        timer_id: TimerId,
        interval_duration: Duration,
        max_count: Option<NonZeroU64>,
        state: Arc<TimerState>,
        started_at: Instant,
    ) {
        // Use tokio::time::Interval with skip behavior to reduce drift
        let mut interval = interval_at(TokioInstant::now() + interval_duration, interval_duration);
        interval.set_missed_tick_behavior(MissedTickBehavior::Skip);

        let mut count = 0u64;

        loop {
            tokio::select! {
                _ = interval.tick() => {
                    // Check if stopped
                    if state.stopped.load(Ordering::Relaxed) {
                        break;
                    }

                    // If paused, skip delivery but keep schedule ticking.
                    if state.paused.load(Ordering::Relaxed) {
                        continue;
                    }

                    count += 1;
                    state.tick_count.store(count, Ordering::Relaxed);

                    let tick = TimerTick {
                        timer_id,
                        count,
                        elapsed: started_at.elapsed(),
                        started_at,
                    };

                    if !sender.post(tick) {
                        // Channel closed
                        break;
                    }

                    // Check if we've reached max count
                    if let Some(max) = max_count {
                        if count >= max.get() {
                            break;
                        }
                    }
                }
                _ = state.pause_notify.notified() => {
                    if state.stopped.load(Ordering::Relaxed) {
                        break;
                    }
                }
            }
        }

        state.finished.store(true, Ordering::Relaxed);
    }

    /// Pause the timer (can be resumed).
    ///
    /// Note: For interval timers, time continues to elapse and ticks are skipped
    /// while paused. When resumed, the next tick fires at the next normal boundary.
    pub fn pause(&mut self) {
        self.state.paused.store(true, Ordering::Relaxed);
        self.state.pause_notify.notify_waiters();
    }

    /// Resume a paused timer.
    pub fn resume(&mut self) {
        self.state.paused.store(false, Ordering::Relaxed);
        self.state.pause_notify.notify_waiters();
    }

    /// Stop the timer permanently.
    pub fn stop(&mut self) {
        self.state.stopped.store(true, Ordering::Relaxed);
        self.state.finished.store(true, Ordering::Relaxed);
        self.state.pause_notify.notify_waiters();
        if let Some(handle) = self.handle.take() {
            handle.abort();
        }
    }

    /// Reset the timer: stops and restarts from now with count reset to 0.
    ///
    /// This is equivalent to calling `stop()` followed by `start()`.
    /// The timer restarts immediately from "now" - remaining time is not preserved.
    pub fn reset(&mut self) {
        self.stop();
        // Clear the handle so start() will work
        self.handle = None;
        self.start();
    }
}

impl Drop for Timer {
    fn drop(&mut self) {
        self.stop();
    }
}
```

### MessagePump Extensions

```rust
use std::num::NonZeroU64;
use std::sync::{Arc, Weak};
use std::collections::HashSet;
use parking_lot::Mutex;

impl MessagePump {
    /// Create a one-shot timer that fires after the given delay.
    ///
    /// Returns a Timer that can be used to cancel the scheduled callback.
    /// The timer is automatically stopped when dropped (RAII).
    pub fn set_timer(&self, delay: Duration) -> Timer {
        let mut timer = Timer::new_oneshot(self.sender(), delay);
        timer.start();
        self.register_timer(&timer);
        timer
    }

    /// Create a repeating timer that fires at the given interval indefinitely.
    ///
    /// Returns a Timer that can be paused, resumed, or stopped.
    /// The timer is automatically stopped when dropped (RAII).
    pub fn set_interval(&self, interval: Duration) -> Timer {
        let mut timer = Timer::new_interval(self.sender(), interval, Repeat::Infinite);
        timer.start();
        self.register_timer(&timer);
        timer
    }

    /// Create a repeating timer that fires a specific number of times.
    ///
    /// `count` must be non-zero.
    ///
    /// Returns a Timer that can be paused, resumed, or stopped.
    pub fn set_interval_count(&self, interval: Duration, count: NonZeroU64) -> Timer {
        let mut timer = Timer::new_interval(self.sender(), interval, Repeat::Count(count));
        timer.start();
        self.register_timer(&timer);
        timer
    }

    /// Register a timer in the weak registry (for cleanup on close).
    fn register_timer(&self, timer: &Timer) {
        // Implementation detail: store Weak<TimerState> in internal registry.
        // This allows close() to stop any remaining timers without owning them.
    }

    /// Stop all registered timers (called during close).
    fn stop_all_timers(&self) {
        // Iterate through weak references, set stopped=true, and notify.
        // Prune dead Weak entries during iteration.
    }
}
```

## File Organization

Create a new module at `src/timer.rs`:
- `TimerId` struct
- `TimerTick` message
- `TimerState` (private)
- `Timer` struct and implementation

Add to `src/pump.rs`:
- `MessagePump::set_timer()`, `set_interval()`, `set_interval_count()` methods
- Internal weak timer registry

Re-export from `src/lib.rs`:
```rust
pub mod timer;
pub use timer::{Repeat, Timer, TimerId, TimerTick};
```

## Usage Examples

### One-Shot Timer (Debounce)

```rust
impl MyWidget {
    fn on_input_changed(&mut self) {
        // Cancel any existing debounce timer (RAII: just drop it)
        self.debounce_timer = None;

        // Set a new 300ms debounce timer
        self.debounce_timer = Some(self.pump.set_timer(Duration::from_millis(300)));
    }

    async fn handle_message(&mut self, envelope: &mut MessageEnvelope) -> bool {
        if let Some(tick) = envelope.message.as_any().downcast_ref::<TimerTick>() {
            if Some(tick.timer_id) == self.debounce_timer.as_ref().map(|t| t.id()) {
                self.perform_search();
                return true;
            }
        }
        false
    }
}
```

### Interval Timer (Animation)

```rust
impl LoadingIndicator {
    fn on_mount(&mut self) {
        // Start animation at 100ms intervals
        self.animation_timer = Some(
            self.pump.set_interval(Duration::from_millis(100))
                .with_name("loading-animation")
        );
    }

    fn on_unmount(&mut self) {
        // RAII: timer stops when dropped
        self.animation_timer = None;
    }

    async fn handle_message(&mut self, envelope: &mut MessageEnvelope) -> bool {
        if let Some(tick) = envelope.message.as_any().downcast_ref::<TimerTick>() {
            // Use tick.count for animation frame
            self.frame = (tick.count as usize) % self.frames.len();
            self.refresh();
            return true;
        }
        false
    }
}
```

### Clock Widget with Elapsed Time

```rust
impl Clock {
    fn on_mount(&mut self) {
        self.timer = Some(self.pump.set_interval(Duration::from_secs(1)));
    }

    async fn handle_message(&mut self, envelope: &mut MessageEnvelope) -> bool {
        if let Some(tick) = envelope.message.as_any().downcast_ref::<TimerTick>() {
            // Use tick.elapsed for uptime display
            let uptime = tick.elapsed.as_secs();
            self.update_display(uptime);
            return true;
        }
        false
    }
}
```

## Integration with Widgets

Widgets that need timers will:
1. Store `Option<Timer>` for each timer they manage
2. Call `pump.set_timer()` or `pump.set_interval()` to create timers
3. Handle `TimerTick` messages in their message handler
4. Rely on RAII (Drop) for cleanup - explicit `stop()` is optional

## Testing Strategy

1. **Unit Tests**:
   - Timer creation and ID uniqueness
   - Start/stop/pause/resume state transitions
   - TimerTick message fields (count, elapsed, started_at)
   - `is_running()` returns false after one-shot completes
   - `is_finished()` returns true after completion

2. **Integration Tests**:
   - Timer fires after delay (within tolerance)
   - Interval timer fires repeatedly with correct count
   - Paused timer doesn't fire
   - Resumed timer continues correctly
   - Stopped timer cleans up properly
   - Timer drops cleanly (RAII)
   - MissedTickBehavior::Skip reduces drift

3. **Async Tests**:
   - Use `tokio::test` for async timer behavior
   - Test timing accuracy (within ~10ms tolerance for UI)
   - Test pause/resume with Notify (no busy-wait)

## Implementation Plan

### Phase 1: Core Types
- [ ] Create `src/timer.rs` module
- [ ] Add `TimerId` struct with atomic counter
- [ ] Add `TimerTick` message with elapsed/started_at
- [ ] Write tests for TimerId uniqueness

### Phase 2: Timer Struct
- [ ] Implement `TimerState` with Notify for pause/resume
- [ ] Implement `Timer` struct with correct is_running/is_finished
- [ ] Implement `start()` with tokio::time::Interval + MissedTickBehavior::Skip
- [ ] Implement `stop()`, `pause()`, `resume()`, `reset()`
- [ ] Implement `Drop` for RAII cleanup
- [ ] Add optional `name` field
- [ ] Write unit tests

### Phase 3: MessagePump Integration
- [ ] Add `set_timer()` method
- [ ] Add `set_interval()` method
- [ ] Add `set_interval_count()` method
- [ ] Add weak timer registry for close() cleanup
- [ ] Write integration tests

### Phase 4: Verification
- [ ] Test with real widget (LoadingIndicator)
- [ ] Verify RAII cleanup works (no timer leaks)
- [ ] Verify pause/resume doesn't busy-wait
- [ ] Document usage patterns

## Design Decisions

### 1. Timer Naming
**Decision**: Yes, add optional names via `with_name()` builder method.

Optional string names help debugging/logging. The field is `Option<String>` to avoid allocation when not used.

### 2. Timer Callbacks vs Messages
**Decision**: Messages only.

Consistent with Python Textual's conceptual model. Keeps execution on the message loop. If ergonomics are needed, add convenience methods at the widget level later.

### 3. Timer Ownership and Registry
**Decision**: RAII primary + weak registry in MessagePump.

- RAII (Drop) is the primary cleanup mechanism - idiomatic Rust
- Weak registry in MessagePump allows `close()` to stop any remaining timers
- Users don't need to remember `on_unmount()` cleanup - just drop the timer

### 4. File Organization
**Decision**: Separate module at `src/timer.rs`.

Better separation of concerns. `src/pump.rs` is already large. Timers can be tested and evolved independently.

### 5. Interval Timing
**Decision**: Use `tokio::time::Interval` with `MissedTickBehavior::Skip`.

Reduces drift and matches Python Textual's `skip=True` behavior. For one-shot, `sleep` is fine.

### 6. Pause/Resume Implementation
**Decision**: Use `tokio::sync::Notify` instead of busy-wait.

Immediate pause/resume without CPU burn. Uses `tokio::select!` for responsive state changes.

Note: Current design continues the interval while paused and *skips* delivery during pause. When resumed, the next tick fires at the next normal interval boundary.

### 7. Repeat Count
**Decision**: `Repeat` enum with `Count(NonZeroU64)` for bounded timers.

This avoids ambiguity around zero, keeps the API clear, and allows `set_interval_count()` to reject zero at the type level.

### 8. Reset Semantics
**Decision**: `reset()` stops and restarts from "now" with count reset to 0.

Remaining time is not preserved. This is the simplest and most predictable behavior. Documented explicitly in the method.

## Comparison with Python Textual

| Python Textual | Rust Implementation |
|---------------|---------------------|
| `set_timer(delay)` | `pump.set_timer(Duration)` |
| `set_interval(interval)` | `pump.set_interval(Duration)` |
| `timer.stop()` | `timer.stop()` or drop |
| `timer.pause()` | `timer.pause()` |
| `timer.resume()` | `timer.resume()` |
| `Timer.time` | `TimerTick.elapsed` |
| `Timer.count` | `TimerTick.count` |
| `timer.timer_id` | `timer.id()` |
| N/A | `timer.name()` (optional) |
| N/A | `timer.with_name()` (builder) |

## Summary

This design provides:
- **One-shot and interval timers** via `set_timer()` and `set_interval()`
- **Full control** with pause/resume/stop
- **RAII cleanup** - timers stop when dropped
- **Message-based delivery** through existing pump infrastructure
- **Time metadata** in TimerTick (elapsed, started_at)
- **Optional naming** for debugging
- **Efficient pause/resume** using Notify (no busy-wait)
- **Reduced drift** using tokio::time::Interval with Skip behavior
- **Weak registry** in MessagePump for close() cleanup
