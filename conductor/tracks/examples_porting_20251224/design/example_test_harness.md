# Example Test Harness Conventions

This document defines a standard test harness for ported examples, focused on layout, content, and style parity.

## Test File Location

- `textual-rs/tests/<example>_snapshot.rs`
- For doc-only examples, group under `textual-rs/tests/docs_examples/<example>_snapshot.rs`

## Required Imports

```rust
use ratatui::backend::TestBackend;
use ratatui::layout::Rect as RatatuiRect;
use ratatui::Terminal;
use textual_rs::geometry::{Rect as TextualRect, Size};
use textual_rs::layout::{compute_layout_with_styles, LayoutCache};
use textual_rs::render_context::RenderContext;
use textual_rs::style_manager::StyleManager;
use textual_rs::{WidgetId, WidgetRegistry};
```

## Standard Helpers

- `test_terminal(width, height)`
- `buffer_to_string(buffer)`
- `center_bg(buffer, rect)`

## Render Order (Critical)

Always render the parent before children:

```rust
fn render_tree(
    registry: &WidgetRegistry,
    widget_id: WidgetId,
    cache: &LayoutCache,
    frame: &mut Frame,
    context: &RenderContext,
) {
    let Some(rect) = cache.get(widget_id) else { return; };
    let area = RatatuiRect::new(rect.x.max(0) as u16, rect.y.max(0) as u16, rect.width as u16, rect.height as u16);

    if let Some(widget) = registry.get(widget_id) {
        widget.render_with_context(widget_id, area, frame, context);
    }

    for child_id in registry.get_children(widget_id).to_vec() {
        render_tree(registry, child_id, cache, frame, context);
    }
}
```

## Layout + Render Setup

```rust
let theme_bg = style_manager.theme_background();
let viewport = Size::new(width, height);
let context = RenderContext::with_background_and_viewport(&style_manager, theme_bg, viewport);

let layout_rect = TextualRect::new(0, 0, width, height);
let cache = compute_layout_with_styles(&registry, root_id, layout_rect, &style_manager);

terminal.draw(|frame| {
    render_tree(&registry, root_id, &cache, frame, &context);
}).unwrap();
```

## Required Assertions (Minimum)

Each example test must include:

1. **Content presence**
   - At least one key label or widget text appears in output.

2. **Layout sanity**
   - Root or key widgets have non-zero rects.

3. **Style assertion**
   - Compare computed style to a buffer cell, e.g.:
     ```rust
     let widget = registry.get(button_id).unwrap();
     let style = context.style_for(button_id, widget).to_ratatui_style();
     let bg = style.bg.unwrap();
     let rect = cache.get(button_id).unwrap();
     let cell_bg = center_bg(terminal.backend().buffer(), rect);
     assert_eq!(cell_bg, bg);
     ```

## Snapshot Output (Optional)

Printing the snapshot string is allowed for debugging, but tests should not rely
solely on snapshot string comparisons for parity.
