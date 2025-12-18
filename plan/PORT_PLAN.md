# Textual → Rust Port Plan (Parity‑First)

## Constraints
- Parity with Python Textual is primary.
- Direct terminal I/O (no TUI frameworks).
- Idiomatic Rust API preferred, while preserving compatibility where it helps parity.

## Architecture Map (Python Reference)
- App lifecycle + screen stack: `textual/src/textual/app.py`
- Message pump + dispatch: `textual/src/textual/message_pump.py`
- DOM tree + queries: `textual/src/textual/dom.py`
- Widgets + screens: `textual/src/textual/widget.py`, `textual/src/textual/screen.py`
- Events/messages: `textual/src/textual/events.py`, `textual/src/textual/message.py`
- CSS engine: `textual/src/textual/css/*`
- Layout engine: `textual/src/textual/layout.py`, `textual/src/textual/layouts/*`, `_arrange.py`
- Rendering/compositor: `textual/src/textual/_compositor.py`, `strip.py`
- Drivers + input: `textual/src/textual/driver.py`, `textual/src/textual/drivers/*`
- Reactivity: `textual/src/textual/reactive.py`
- Timers/animation/workers: `textual/src/textual/timer.py`, `_animator.py`, `worker_manager.py`

## Rust Module Sketch
- `app/` — App, run loop, screen stack, timers/animations
- `dom/` — DOM tree, ids/classes, queries, focus
- `messages/` — Message/Event, queue, bubbling, dispatch
- `reactive/` — reactive attributes, watchers, validation
- `css/` — tokenizer/parser, selectors, stylesheet, style resolution
- `layout/` — layouts, docking, spatial map
- `render/` — strips, cell buffer, compositor, diffing
- `driver/` — terminal mode, input reader, xterm parsing, raw output
- `widgets/` — core widgets and screens

## Phases

### Phase A — Foundations
- Geometry + box model: `Size`, `Offset`, `Region`, `Spacing`, `BoxModel` equivalents
- Style primitives: `Color`, `Style`, style merge/opacity, SGR emission
- Direct terminal I/O: alt screen, raw mode, cursor control, clear/update
- Render primitives: `Strip` + `Cell` buffer + minimal diff

### Phase B — Message Pump + DOM
- Per‑node message queue and dispatch
- Bubbling / prevent_default / stop propagation
- DOM tree + ids/classes + query selectors (subset first)

### Phase C — Rendering Core
- Widget render → strips → composited screen buffer
- Partial updates / diff

### Phase D — CSS + Layout
- TCSS parsing + selector matching + specificity
- Layouts: vertical/horizontal + dock + layers, then grid

### Phase E — Widgets + Input
- `Widget`/`Screen` + compose/mount lifecycle
- Focus + keybindings + mouse events
- Essential widgets: `Static`, `Label`, `Container`, `ScrollView`

### Phase F — Reactivity + Animations + Workers
- Reactive properties, watchers, validation
- Timers/animations with refresh scheduling
- Worker manager and task lifecycle

## Test Strategy
- Snapshot rendering tests (buffer → golden)
- Layout placement tests (regions vs expected)
- Event propagation tests (bubbling order)
- Input parsing tests (xterm sequences)

## Open Decisions
- Whether to add a compatibility layer mirroring Python naming.
- Whether to split into workspace (`textual-core`, `textual`) or single crate.
