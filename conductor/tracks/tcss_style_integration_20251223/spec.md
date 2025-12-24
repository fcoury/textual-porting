# Specification: TCSS Style Integration

## Overview

This track integrates the existing TCSS styling engine (selectors, properties, animations, transitions, themes, hot reloading) with the widget rendering pipeline. Currently, the TCSS system is fully implemented but not connected to widgets—widgets use hardcoded `ratatui::Style` values instead of computed TCSS styles.

The goal is to wire the TCSS engine into the widget system so that:
1. Widgets can declare `DEFAULT_CSS` for their default styles
2. User stylesheets cascade over widget defaults
3. `ComputedStyle` flows to widgets during render
4. Animations and transitions are driven by a central Animator
5. Hot reloading triggers automatic style refresh

## Functional Requirements

### FR-1: StyleManager

A central `StyleManager` struct that owns and coordinates all styling concerns:

- **Stylesheet**: Merged stylesheet (widget defaults + user CSS)
- **ThemeRegistry**: Active theme and theme switching
- **Animator**: Central animator for transitions and @keyframes
- **HotReloadManager**: File watching for `.tcss` changes (dev mode)
- **Style Cache**: Per-widget computed style cache with invalidation

The `StyleManager` provides:

```rust
impl StyleManager {
    /// Compute resolved style for a widget. Uses compute_style_resolved() internally
    /// to ensure theme variables, auto colors, and animated values are applied.
    pub fn get_style(&self, widget: &WidgetMeta, ancestors: &[&WidgetMeta]) -> ComputedStyle;

    /// Tick the animator. Called each frame by the run loop.
    pub fn tick(&mut self, dt: Duration);

    /// Poll for .tcss file changes. Returns true if stylesheet was reloaded.
    pub fn poll_hot_reload(&mut self) -> bool;

    /// Switch the active theme. Invalidates all cached styles.
    pub fn set_theme(&mut self, name: &str);

    /// Register a widget type's DEFAULT_CSS. Called once at startup.
    pub fn register_widget_defaults<W: Widget>(&mut self);

    /// Load user stylesheet from source string or file.
    pub fn load_user_stylesheet(&mut self, source: &str) -> Result<(), StyleError>;

    /// Invalidate cached styles for a specific widget.
    pub fn invalidate_widget(&mut self, widget_id: WidgetId);

    /// Invalidate all cached styles (theme switch, hot reload).
    pub fn invalidate_all(&mut self);

    /// Get current theme version (for cache invalidation checks).
    pub fn theme_version(&self) -> u64;
}
```

**Critical**: `get_style()` must use `compute_style_resolved()` (not `compute_style()`) to ensure:
- Theme variables are substituted
- Auto colors are resolved against parent background
- Animated property values are applied from the Animator

### FR-2: Widget DEFAULT_CSS

Widgets declare default styles via a static constant:

```rust
trait Widget {
    const DEFAULT_CSS: &'static str = "";
    // ...existing methods...
}
```

**Registration mechanism**: At app startup, widget types must be registered with the StyleManager:

```rust
// In app initialization
style_manager.register_widget_defaults::<Button>();
style_manager.register_widget_defaults::<Label>();
style_manager.register_widget_defaults::<Input>();
// ... etc
```

The `register_widget_defaults::<W>()` method:
1. Parses `W::DEFAULT_CSS` once
2. Merges rules into the stylesheet with `is_user_css = 0` (lowest specificity)
3. Uses `W::widget_type_name()` to scope rules appropriately

A convenience macro or function can register all built-in widgets:
```rust
style_manager.register_builtin_widgets(); // registers Button, Label, Input, etc.
```

**App initialization sequence**:
1. Create StyleManager
2. Call `register_builtin_widgets()` (or individual `register_widget_defaults::<W>()`)
3. Load user stylesheet via `load_user_stylesheet()`
4. Start app with `run_managed()`

### FR-3: ComputedStyle to ratatui::Style Adapter

A narrow conversion function for the paint layer:

```rust
impl ComputedStyle {
    /// Convert color/text properties to ratatui::Style for rendering.
    /// Layout properties (padding, margin, dimensions) are NOT included—
    /// those are handled by the layout system.
    pub fn to_ratatui_style(&self) -> ratatui::style::Style { ... }
}
```

Converts only:
- `color` → `fg`
- `background` → `bg`
- `text_style` (bold, italic, underline, etc.) → `Modifier`
- `opacity` → dim modifier if < 1.0

Does NOT convert (handled by layout system):
- width, height, min-*, max-*
- margin, padding, border
- display, visibility, overflow

### FR-4: Render-Time Style Access via RenderContext

**Decision**: Use a context-based approach to avoid changing every widget's `render()` signature.

A `RenderContext` is passed through the render pipeline:

```rust
pub struct RenderContext<'a> {
    pub style_manager: &'a StyleManager,
    pub parent_background: Color,
    pub ancestors: Vec<WidgetMeta>,
}

impl<'a> RenderContext<'a> {
    /// Get computed style for a widget, using cached value if valid.
    pub fn style_for(&self, widget: &impl Widget) -> ComputedStyle {
        let meta = widget.widget_meta();
        self.style_manager.get_style(&meta, &self.ancestors_as_refs())
    }
}
```

**Ancestor chain order**: The `ancestors` slice is ordered from root to immediate parent:
```rust
ancestors[0]        // root widget (e.g., Screen)
ancestors[1]        // first child of root
...
ancestors[n-1]      // immediate parent of current widget
```
This matches CSS selector semantics where `A B` means "B descendant of A" with A being an ancestor.

Widgets access styles by:
1. Storing a reference to the `RenderContext` (thread-local or passed through)
2. Calling `context.style_for(self)` in their `render()` method

**Migration path**: Existing widgets continue to work with hardcoded styles. Widgets can incrementally adopt TCSS by querying the context.

### FR-5: WidgetMeta Construction

To compute styles, we need `WidgetMeta` for each widget:

```rust
pub struct WidgetMeta {
    pub id: Option<String>,
    pub classes: HashSet<String>,
    pub widget_type: String,
    pub pseudo_classes: HashSet<String>,
}
```

Add to Widget trait:

```rust
trait Widget {
    // ... existing methods ...

    /// Construct WidgetMeta for style computation.
    fn widget_meta(&self) -> WidgetMeta {
        WidgetMeta {
            id: self.id().map(|s| s.to_string()),
            classes: self.classes().clone(),
            widget_type: self.widget_type_name().to_string(),
            pseudo_classes: self.pseudo_classes(),
        }
    }

    /// Return current pseudo-classes (focus, hover, active, disabled, etc.)
    fn pseudo_classes(&self) -> HashSet<String> {
        HashSet::new()
    }
}
```

### FR-6: Animator Integration

The `Animator` is owned by `StyleManager` and ticked in the render loop:

```rust
// In run_managed()
loop {
    let dt = last_frame.elapsed();
    app.style_manager_mut().tick(dt);

    // ... existing layout/render logic
}
```

**Order of operations** when computing a widget's final style:
1. Build `WidgetMeta` with current pseudo-classes
2. Call `compute_style_resolved(widget, ancestors, stylesheet, parent_bg)` to get base style with theme variables and auto colors resolved
3. Query Animator for active animations on this widget
4. Apply animated values as property overrides to the resolved style
5. Return final `ComputedStyle`

This ensures theme resolution happens on the base style, and animations override the resolved values (not raw CSS values).

**Animated value resolution**: The Animator maintains a map of `(WidgetId, PropertyName) -> CurrentValue`. When `get_style()` is called:

1. Compute base style via `compute_style_resolved()`
2. Check Animator for any active animations on this widget
3. Apply animated values as overrides to the ComputedStyle
4. Return the merged result

This avoids recomputing selectors each frame—only the animated property values are updated.

Widgets can start animations via:
```rust
style_manager.animate(
    widget_id,
    "opacity",
    0.0,
    1.0,
    Duration::from_millis(300),
    EasingFunction::EaseOut
);
```

### FR-7: Style Invalidation Triggers

Styles must be recomputed when any of these change:

| Trigger | Action |
|---------|--------|
| Class added/removed | `invalidate_widget(widget_id)` |
| ID changed | `invalidate_widget(widget_id)` |
| Pseudo-class change (focus, hover, active, disabled) | `invalidate_widget(widget_id)` |
| Widget mounted | Compute initial style |
| Widget unmounted | Remove from cache |
| Theme switched | `invalidate_all()` |
| Hot reload | `invalidate_all()` |
| Ancestor class/state change | Invalidate descendants (for descendant selectors) |

**Integration with WidgetRefreshState**: When a widget's style is invalidated:
```rust
widget.refresh_state.refresh_styles_required.set(true);
```

The render loop checks this flag and recomputes styles before rendering.

### FR-8: Style Caching

Cache key: `(WidgetId, AncestorChainHash, ThemeVersion)`

```rust
struct StyleCacheEntry {
    computed: ComputedStyle,
    ancestor_hash: u64,      // Hash of ancestor chain (types, classes, ids)
    theme_version: u64,      // StyleManager::theme_version() at computation time
}
```

**Ancestor hash inputs** (for deterministic invalidation):
- For each ancestor: `(widget_type, id, classes sorted alphabetically, pseudo_classes sorted alphabetically)`
- Hash computed in root→parent order
- Use a stable hasher (e.g., `DefaultHasher` with consistent seed)

Cache invalidation occurs when:
- Widget's classes, id, or pseudo-classes change → remove entry
- Any ancestor's state changes → remove entry (ancestor_hash won't match)
- Theme switches → increment theme_version, all entries stale
- Hot reload → increment theme_version, all entries stale

### FR-9: Hot Reload Integration

In development mode, `StyleManager` watches `.tcss` files:

```rust
// In app initialization
style_manager.enable_hot_reload(vec!["./styles"]);

// In run_managed() loop
if style_manager.poll_hot_reload() {
    // Stylesheet was reloaded
    style_manager.invalidate_all();
    // All widgets will recompute styles on next render
}
```

### FR-10: ManagedWidgetApp Extension

Extend the `ManagedWidgetApp` trait to include style management:

```rust
pub trait ManagedWidgetApp: App {
    // ... existing methods ...

    fn style_manager(&self) -> &StyleManager;
    fn style_manager_mut(&mut self) -> &mut StyleManager;
}
```

The `run_managed()` function is updated to:
1. Tick the animator each frame
2. Poll hot reload (if enabled)
3. Construct `RenderContext` with `StyleManager` reference
4. Pass context through render pipeline

## Non-Functional Requirements

### NFR-1: Performance

- DEFAULT_CSS parsing happens once at startup via `register_widget_defaults()`
- Style computation is cached per-widget with smart invalidation
- Animator tick is O(active_animations), not O(all_widgets)
- Hot reload polling uses filesystem mtime, not content hashing
- Cache hit path is a simple HashMap lookup + version check

### NFR-2: Backwards Compatibility

- Existing widgets continue to work without modification (hardcoded styles still render)
- `DEFAULT_CSS` defaults to empty string
- `pseudo_classes()` defaults to empty set
- `widget_meta()` has a default implementation
- Widgets can incrementally adopt TCSS by querying `RenderContext`
- **Minimal migration**: widgets that want TCSS styling add a `context.style_for(self)` call

### NFR-3: Testability

- `StyleManager` can be constructed without file I/O for tests
- Animator can be stepped manually with `tick(Duration)`
- `ComputedStyle` is `PartialEq` for assertions
- Mock `RenderContext` can be created for widget render tests

## Acceptance Criteria

1. **StyleManager exists** and holds StyleSheet, ThemeRegistry, Animator, HotReloadManager, and style cache
2. **Widget::DEFAULT_CSS** is available on the trait with empty default
3. **register_widget_defaults()** parses and merges widget default CSS
4. **load_user_stylesheet()** loads and merges user CSS with higher specificity
5. **At least 3 widgets** (Button, Label, Input) define DEFAULT_CSS and render using computed styles
6. **ComputedStyle::to_ratatui_style()** converts color/text properties correctly
7. **RenderContext** provides `style_for()` access to computed styles
8. **Animator is ticked** in run_managed() and animated values are applied to ComputedStyle
9. **Style invalidation** works for class changes, pseudo-class changes, and theme switches
10. **Hot reload works**: changing a .tcss file triggers style refresh without restart
11. **Theme switching** changes widget appearance at runtime
12. **All existing tests pass** (backwards compatibility)
13. **New integration tests** verify style computation, caching, animation, and invalidation

## Out of Scope

- Changing the layout system to read from ComputedStyle (layout already works)
- Adding new CSS properties beyond what's already parsed
- CSS-in-Rust macros or compile-time style checking
- Visual regression testing infrastructure
- Performance profiling/optimization beyond basic caching
- Automatic discovery of DEFAULT_CSS (explicit registration required)
