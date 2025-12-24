# Example Porting Template

This template defines the standard structure for Rust example ports of Python Textual examples.

## File Layout

- Example code: `textual-rs/examples/<name>.rs`
- Example CSS: `textual-rs/examples/<name>.tcss` (if Python uses a .tcss file)
- Example assets (json/md): `textual-rs/examples/assets/<name>/...`
- Example tests: `textual-rs/tests/<name>_snapshot.rs`

## Required Pipeline

Each example must:
- Build a widget tree using `WidgetRegistry` and `ComposeContext`
- Register widget defaults in `StyleManager`
- Load example TCSS (and optional user CSS)
- Use `compute_layout_with_styles` for layout
- Render with `RenderContext::with_background_and_viewport`
- Call `render_with_context` for every widget (parent first, then children)

## Standard Skeleton (Rust)

```rust
use ratatui::Frame;
use textual_rs::compose::ComposeContext;
use textual_rs::geometry::{Rect, Size};
use textual_rs::layout::{compute_layout_with_styles, LayoutCache};
use textual_rs::render_context::RenderContext;
use textual_rs::style_manager::StyleManager;
use textual_rs::widgets::{Container, Grid};
use textual_rs::{App, WidgetId, WidgetRegistry};

const EXAMPLE_CSS: &str = include_str!("example.tcss");

struct ExampleApp {
    registry: WidgetRegistry,
    style_manager: StyleManager,
    root_id: WidgetId,
}

impl ExampleApp {
    fn new() -> Self {
        let mut registry = WidgetRegistry::new();
        let mut sm = StyleManager::new();

        // Register widget defaults
        sm.register_builtin_widgets().unwrap();

        // Load CSS
        sm.load_user_stylesheet(EXAMPLE_CSS).unwrap();

        let root_id = registry.add(Container::new().with_id("root"));
        let mut ctx = ComposeContext::new(&mut registry, root_id);

        // Compose children
        let _child_id = ctx.add(Grid::new());

        Self { registry, style_manager: sm, root_id }
    }

    fn render_tree(
        &self,
        widget_id: WidgetId,
        cache: &LayoutCache,
        frame: &mut Frame,
        context: &RenderContext,
    ) {
        let Some(rect) = cache.get(widget_id) else { return; };
        let area = ratatui::layout::Rect::new(
            rect.x.max(0) as u16,
            rect.y.max(0) as u16,
            rect.width as u16,
            rect.height as u16,
        );

        // Render parent first
        if let Some(widget) = self.registry.get(widget_id) {
            widget.render_with_context(widget_id, area, frame, context);
        }

        // Then render children
        for child_id in self.registry.get_children(widget_id).to_vec() {
            self.render_tree(child_id, cache, frame, context);
        }
    }
}

impl App for ExampleApp {
    type Message = ();

    fn update(&mut self, _msg: Self::Message) {}

    fn view(&self, frame: &mut Frame) {
        let area = frame.area();
        let viewport = Size::new(area.width, area.height);
        let theme_bg = self.style_manager.theme_background();
        let context = RenderContext::with_background_and_viewport(&self.style_manager, theme_bg, viewport);

        let layout_rect = Rect::new(0, 0, area.width, area.height);
        let cache = compute_layout_with_styles(&self.registry, self.root_id, layout_rect, &self.style_manager);

        self.render_tree(self.root_id, &cache, frame, &context);
    }
}
```

## Asset Mapping

- If the Python example uses `example.tcss`, include it as a const string or file in `examples/`.
- If the Python example uses JSON/MD data, copy it into `textual-rs/examples/assets/<name>/` and load it at runtime.

## Input Handling

- Use `crossterm::event` mappings consistent with Python (keys, mouse, modifiers)
- Keep focus handling in sync with widget pseudo-classes (`:focus`, `:hover`, etc.)

## Testing Requirements

Each example must have tests with:
- Snapshot output (string grid)
- At least one style assertion (bg/fg/border) for key widgets
- Layout sanity checks (rect sizes / spans)

Use the conventions defined in `design/example_test_harness.md`.
