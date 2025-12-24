# Render Pipeline Analysis

## Current App Traits

### 1. `App` (Base Trait)
```rust
pub trait App {
    type Message;
    fn update(&mut self, msg: Self::Message);
    fn view(&self, frame: &mut Frame);
}
```
- Minimal trait for basic applications
- `view()` receives a `Frame` for rendering
- No access to widget registry or layout

### 2. `WidgetApp` (With Mouse)
```rust
pub trait WidgetApp: App {
    fn registry(&self) -> &WidgetRegistry;
    fn registry_mut(&mut self) -> &mut WidgetRegistry;
    fn layout_cache(&self) -> &LayoutCache;
    fn root(&self) -> WidgetId;
    fn on_mouse(&mut self, _event: TargetedMouseEvent) {}
}
```
- Adds widget registry access
- Adds layout cache for hit-testing
- No LayoutManager (manual layout)

### 3. `ManagedWidgetApp` (Automatic Layout)
```rust
pub trait ManagedWidgetApp: App {
    fn registry(&self) -> &WidgetRegistry;
    fn registry_mut(&mut self) -> &mut WidgetRegistry;
    fn layout_manager(&self) -> &LayoutManager;
    fn layout_manager_mut(&mut self) -> &mut LayoutManager;
    fn root(&self) -> WidgetId;
    fn on_mouse(&mut self, _event: TargetedMouseEvent) {}
    fn check_layout(&mut self) {}
    fn layout_cache(&self) -> &LayoutCache { self.layout_manager().cache() }
}
```
- Automatic layout management
- `check_layout()` hook before each frame
- **This is where StyleManager should be added**

## Run Functions

### `run_managed()` Loop (Target for Integration)

```rust
pub fn run_managed<A>(mut app: A) -> Result<(), TerminalError>
where
    A: ManagedWidgetApp,
    A::Message: From<Event>,
{
    let mut terminal_guard = TerminalGuard::new()?;
    let terminal = terminal_guard.terminal();

    // Initial layout
    app.layout_manager_mut().invalidate_all();

    loop {
        // 1. Get terminal size
        let size = terminal.size()?;
        let container = Rect::new(0, 0, size.width, size.height);

        // 2. Check layout
        app.check_layout();

        // 3. Run layout if needed
        let needs_layout = app.layout_manager().needs_layout();
        let root_id = app.root();

        if needs_layout {
            let cache = compute_layout(app.registry(), root_id, container);
            app.layout_manager_mut().set_cache(cache, root_id, container);
        }

        // 4. RENDER - app.view(f)
        terminal.draw(|f| app.view(f))?;

        // 5. Handle events
        if event::poll(std::time::Duration::from_millis(16))? {
            let event = event::read()?;
            // ... handle key, mouse events ...
            app.update(A::Message::from(event));
        }
    }
}
```

## Integration Points for StyleManager

### Required Changes to `ManagedWidgetApp`

```rust
pub trait ManagedWidgetApp: App {
    // ... existing methods ...

    // NEW: Access to StyleManager
    fn style_manager(&self) -> &StyleManager;
    fn style_manager_mut(&mut self) -> &mut StyleManager;
}
```

### Required Changes to `run_managed()`

```rust
loop {
    // ... get size, check_layout ...

    // NEW: Tick animator
    let dt = last_frame.elapsed();
    app.style_manager_mut().tick(dt);

    // NEW: Poll hot reload
    if app.style_manager_mut().poll_hot_reload() {
        app.style_manager_mut().invalidate_all();
        // Widgets will recompute styles on next render
    }

    // Layout if needed (unchanged)
    if needs_layout {
        let cache = compute_layout(app.registry(), root_id, container);
        app.layout_manager_mut().set_cache(cache, root_id, container);
    }

    // NEW: Create RenderContext
    let render_context = RenderContext {
        style_manager: app.style_manager(),
        parent_background: Color::default(),
        ancestors: vec![],
    };

    // Render with context (needs design decision)
    terminal.draw(|f| app.view(f))?;  // How to pass context?

    // ... handle events ...
}
```

## Design Challenge: Passing RenderContext to Widgets

### Current Widget Rendering

Widgets are rendered via `view()` which uses the registry:
```rust
fn view(&self, frame: &mut Frame) {
    // Walk widget tree and call widget.render(area, frame)
    if let Some(widget) = self.registry().get(id) {
        widget.render(area, frame);
    }
}
```

### Options for Style Context Access

**Option A: Thread-local context**
```rust
thread_local! {
    static RENDER_CONTEXT: RefCell<Option<RenderContext<'static>>> = RefCell::new(None);
}

// Set before render, widgets access via thread-local
```
- Pros: No signature changes
- Cons: Lifetime complexity, global state

**Option B: Render method signature change**
```rust
fn render(&self, area: Rect, frame: &mut Frame, context: &RenderContext);
```
- Pros: Explicit, testable
- Cons: Breaking change to all widgets

**Option C: Store context in App, widgets query via registry**
```rust
// Widget gets context from some accessible location
fn render(&self, area: Rect, frame: &mut Frame) {
    // How does widget get context? Need some mechanism.
}
```

**Recommendation: Option B with migration**
- Add `render_with_context()` to Widget trait with default that delegates to `render()`
- Update run loop to call `render_with_context()`
- Widgets migrate incrementally

## Timing Considerations

The render loop runs at ~60 FPS (16ms poll interval).

For StyleManager integration:
1. `tick()` - should be O(active_animations), very fast
2. `poll_hot_reload()` - uses file mtime, should be fast
3. `get_style()` - should use cache, only compute on invalidation

## Summary: Required Work

1. **Add to ManagedWidgetApp trait:**
   - `style_manager()` and `style_manager_mut()`

2. **Update run_managed() to:**
   - Track frame timing for animator
   - Call `style_manager.tick(dt)` each frame
   - Call `poll_hot_reload()` and invalidate if changed
   - Create/pass RenderContext (design TBD)

3. **Design RenderContext access pattern:**
   - How widgets get context during render
   - Thread-local vs signature change vs other

4. **App initialization:**
   - Where/when to call `register_builtin_widgets()`
   - Where/when to load user stylesheet
