# Phase D Example Port Plan — Clock

## 1) Chosen example (Python)
- `textual/examples/clock.py`
- Why: smallest surface area (single widget, timer, CSS align) while still exercising lifecycle + styles.

## 2) Expected Rust example shape
Target file: `textual-rs/examples/clock.rs`

Sketch:
- `struct ClockApp { app: DomApp, quit: bool }`
- `ClockApp::new()` builds a root `DomNode` with class `root` and one `Digits`-like widget node (or a `Label` placeholder if we stage it).
- `on_ready` equivalent schedules a 1s interval to update the widget text.
- CSS:
  - `Screen { align: center middle; }`
  - `Digits { width: auto; }`

Expected Rust API (idealized):
- `App::on_ready(&mut self)` or a `Runtime::schedule_interval`.
- `DomApp::query_one("Digits")` or `DomNode::query_one` + update text.
- `Digits` widget (or `Label` with big glyphs) that supports `update(text)`.

## 3) Missing pieces by layer

### Runtime / App lifecycle
- `on_ready` hook equivalent for initial setup.
- Interval/timer scheduling in the app loop (per-second callback).
- A safe way to mutate DOM from timers (message pump or queued updates).

### DOM / Widgets
- `Digits` widget (Textual uses rich digits); at minimum, a Rust widget that renders large digits.
- `query_one` by type or selector returning a widget handle (or `DomNode` with a label).
- `update(text)` API that triggers a redraw / invalidation.

### CSS / Styles
- Ensure `align: center middle` works at the root `Screen` level.
- `width: auto` respected for the digits widget.
- `Digits { width: auto; }` needs to be parsed and applied (already supported for width/height, but verify in layout stage).

### Layout / Rendering
- Widget can report intrinsic size (digits width/height) so `auto` sizing behaves.
- Center alignment should work for a single widget with intrinsic size.

### Examples / UX
- Example should run via `cargo run --example clock`.
- Demo should look close to Textual’s `Digits` (big clock digits centered).

## 4) Parallel vertical plan (slices)

### Slice A — Minimal functional clock (Label-based)
- Add `examples/clock.rs` using a `Label` node (no Digits yet).
- Implement interval scheduling in `AppLoop`/runtime.
- Add a `query_one`-like helper for DOM nodes or keep a direct handle to the label node.
- Verify CSS align center middle works for a single widget.
- Outcome: working clock text (small) updates every second.

### Slice B — Digits widget + intrinsic sizing
- Implement `Digits` widget rendering (ASCII block digits or 7‑seg style).
- Provide intrinsic size hint so `width: auto` and `height: auto` resolve to the digits size.
- Add a widget test to ensure `Digits` updates redraw and size changes propagate.
- Outcome: big centered clock similar to Textual.

### Slice C — Polish + parity checks
- Align CSS behavior with Textual for `Screen` and `Digits` selectors.
- Add tests for timer scheduling (tick-driven update).
- Verify no regressions in existing layout tests.

## Acceptance criteria
- `cargo run --example clock` renders a centered clock with large digits.
- Digits update every second without flicker or crashes.
- CSS `align: center middle` and `width: auto` are honored in the example.
- New tests cover interval scheduling and Digits sizing/update behavior.
