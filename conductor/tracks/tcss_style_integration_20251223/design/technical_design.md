# TCSS Style Integration - Technical Design Document

## Executive Summary

This document describes the integration of the existing TCSS styling engine with the widget rendering pipeline. The goal is to connect the fully-implemented TCSS system (selectors, properties, animations, themes, hot reloading) to widgets so they render using computed CSS styles instead of hardcoded values.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         run_managed()                            │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    Frame Loop                               │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  │ │
│  │  │ tick()   │→ │ poll()   │→ │ layout() │→ │  render()  │  │ │
│  │  │ Animator │  │ HotReload│  │          │  │            │  │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └────────────┘  │ │
│  └───────────────────────────────────┬────────────────────────┘ │
│                                       │                          │
│  ┌────────────────────────────────────▼───────────────────────┐ │
│  │                      StyleManager                           │ │
│  │  ┌────────────┐ ┌──────────────┐ ┌────────────────────────┐│ │
│  │  │ StyleSheet │ │ThemeRegistry │ │  Animator              ││ │
│  │  │ (merged)   │ │              │ │  (animations)          ││ │
│  │  └────────────┘ └──────────────┘ └────────────────────────┘│ │
│  │  ┌────────────┐ ┌──────────────────────────────────────────┐│ │
│  │  │StyleCache  │ │  HotReloadManager                       ││ │
│  │  │(per-widget)│ │  (file watching)                        ││ │
│  │  └────────────┘ └──────────────────────────────────────────┘│ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                     RenderContext                            │ │
│  │  • style_manager reference                                   │ │
│  │  • parent_background (for auto colors)                       │ │
│  │  • ancestors (root → parent)                                 │ │
│  └───────────────────────────────────┬─────────────────────────┘ │
│                                       │                          │
│  ┌────────────────────────────────────▼───────────────────────┐ │
│  │                      Widgets                                 │ │
│  │  ┌────────────────────────────────────────────────────────┐ │ │
│  │  │ fn render_with_context(id, area, frame, &RenderContext)│ │ │
│  │  │     let style = context.style_for(self.id, self);      │ │ │
│  │  │     let ratatui_style = style.to_ratatui_style();      │ │ │
│  │  │     // render with style...                            │ │ │
│  │  └────────────────────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## Key Components

### 1. StyleManager

Central coordinator owning all styling concerns.

**Owns:**
- `StyleSheet` - Merged widget defaults + user CSS
- `ThemeRegistry` - Active theme and switching
- `Animator` - Transitions and @keyframes
- `transition_overlays` - ActiveTransition map for non-numeric transitions
- `HotReloadManager` - File watching (optional)
- `StyleCache` - Per-widget computed style cache

**Key Methods:**
```rust
pub fn register_widget_defaults<W: Widget>(&mut self);
pub fn load_user_stylesheet(&mut self, source: &str) -> Result<(), StyleError>;
pub fn get_style(&self, widget_id, widget, ancestors, parent_bg) -> ComputedStyle;
pub fn tick(&mut self, now: Instant);
pub fn poll_hot_reload(&mut self) -> bool;
pub fn set_theme(&mut self, name: &str) -> Result<(), ThemeError>;
pub fn invalidate_widget(&mut self, widget_id: WidgetId);
pub fn invalidate_all(&mut self);
```

**Rule Origin:** StyleSheet entries record whether a rule came from DEFAULT_CSS
or user CSS so `Specificity.is_user_css` can be injected during matching.

**Design Doc:** [style_manager.md](./style_manager.md)

### 2. RenderContext

Bridge between StyleManager and widgets during rendering.

**Contains:**
- Reference to `StyleManager`
- `parent_background` for auto color resolution
- `ancestors` chain (root → parent order)

**Key Methods:**
```rust
pub fn style_for(&self, widget_id, widget) -> ComputedStyle;
pub fn child_context(&self, parent, parent_style) -> Self;
```

**Design Doc:** [render_context.md](./render_context.md)

### 3. StyleCache

Per-widget computed style cache with multi-factor invalidation.

**Cache Key:** `WidgetId`

**Validation Factors:**
- `theme_version` - Matches current StyleManager version
- `ancestor_hash` - Hash of ancestor chain styling inputs
- `widget_hash` - Hash of widget's own styling inputs
- `parent_bg_hash` - Included when auto colors are present

StyleCache uses interior mutability so `StyleManager::get_style(&self)` can update
cache stats without requiring a mutable borrow.

**Design Doc:** [style_cache.md](./style_cache.md)

### 4. WidgetMeta Extension

Existing `WidgetMeta` extended with `pseudo_classes`:

```rust
pub struct WidgetMeta {
    pub type_name: String,
    pub id: Option<String>,
    pub classes: Vec<String>,
    pub pseudo_classes: HashSet<String>,  // NEW
}
```

### 5. Widget Trait Extension

New trait methods for TCSS integration:

```rust
pub trait Widget {
    const DEFAULT_CSS: &'static str = "";

    fn pseudo_classes(&self) -> HashSet<String> { HashSet::new() }

    fn widget_meta(&self) -> WidgetMeta {
        WidgetMeta {
            type_name: self.widget_type_name().to_string(),
            id: self.id().map(|s| s.to_string()),
            classes: self.classes().to_vec(),
            pseudo_classes: self.pseudo_classes(),
        }
    }

    fn render_with_context(&self, id: WidgetId, area: Rect, frame: &mut Frame, context: &RenderContext) {
        self.render(area, frame);  // Default: backwards compatible
    }
}
```

## Style Computation Pipeline

```
1. Widget::widget_meta()
   └─► Build WidgetMeta with current pseudo-classes

2. StyleCache::get()
   ├─► Cache HIT: return cached ComputedStyle (validated against parent background when auto colors are used)
   └─► Cache MISS: continue to step 3

3. compute_style_resolved()
   ├─► Match selectors against widget + ancestors
   ├─► Apply cascade (origin: default vs user, then !important, then specificity)
   ├─► Resolve theme variables ($primary, $background, etc.)
   └─► Resolve auto colors against parent background

4. StyleCache::insert()
   └─► Store computed style with validation factors

5. Animator::get_value()
   └─► Get active numeric animation values for widget properties

6. apply_animations()
   └─► Overlay numeric animations + transition overlays on computed style

7. Return final ComputedStyle
```

## Invalidation Strategy

| Event | Scope | Action |
|-------|-------|--------|
| Class added/removed | Widget | `invalidate_widget(id)` |
| Pseudo-class change | Widget | `invalidate_widget(id)` |
| Theme switch | All | `invalidate_all()` + increment version |
| Hot reload | All | `invalidate_all()` + increment version |
| Widget mount | N/A | Compute on first access |
| Widget unmount | Widget | `cache.remove(id)` |
| Ancestor change | Descendants | Lazy (hash mismatch on access) |

## Integration Points

### ManagedWidgetApp Trait

```rust
pub trait ManagedWidgetApp: App {
    // Existing methods...

    fn style_manager(&self) -> &StyleManager;
    fn style_manager_mut(&mut self) -> &mut StyleManager;

    /// Render with style context (default delegates to App::view).
    fn view_with_context(&self, frame: &mut Frame, _context: &RenderContext) {
        self.view(frame);
    }
}
```

### run_managed() Loop

```rust
pub fn run_managed<A>(mut app: A) -> Result<(), TerminalError> {
    // ... setup ...

    loop {
        let now = Instant::now();

        // NEW: Tick animator
        app.style_manager_mut().tick(now);

        // NEW: Poll hot reload
        if app.style_manager_mut().poll_hot_reload() {
            app.style_manager_mut().invalidate_all();
        }

        // Existing: Layout
        if needs_layout { /* ... */ }

        // NEW: Create render context
        let context = RenderContext::new(app.style_manager());

        // Render with context
        terminal.draw(|f| {
            app.view_with_context(f, &context);
        })?;

        // Existing: Event handling
        // ...
    }
}
```

### App Initialization

```rust
fn new() -> Self {
    let mut style_manager = StyleManager::new();

    // 1. Register built-in widget defaults
    style_manager.register_builtin_widgets().unwrap();

    // 2. Load user stylesheet
    style_manager.load_user_stylesheet_file("app.tcss").unwrap();

    // 3. Enable hot reload (dev mode)
    style_manager.enable_hot_reload(vec!["./styles"]);

    Self {
        style_manager,
        // ...
    }
}
```

## ComputedStyle to ratatui::Style

Narrow adapter converting only paint-layer properties:

```rust
impl ComputedStyle {
    pub fn to_ratatui_style(&self) -> ratatui::style::Style {
        let mut style = ratatui::style::Style::default();

        if let Some(color) = &self.color {
            style = style.fg(color.into());
        }
        if let Some(bg) = &self.background {
            style = style.bg(bg.into());
        }

        let mut modifiers = Modifier::empty();
        if let Some(ts) = &self.text_style {
            if ts.bold { modifiers |= Modifier::BOLD; }
            if ts.italic { modifiers |= Modifier::ITALIC; }
            // ... etc
        }

        if !modifiers.is_empty() {
            style = style.add_modifier(modifiers);
        }

        style
    }
}
```

**NOT converted** (handled by layout system):
- width, height, min-*, max-*
- margin, padding, border
- display, visibility, overflow

## Migration Path

### Phase 1: Infrastructure
- Add StyleManager to ManagedWidgetApp
- Update run_managed() with tick/poll/context
- Add render_with_context() to Widget trait

### Phase 2: Widget Migration
For each widget:
1. Add `DEFAULT_CSS` constant
2. Override `pseudo_classes()` if interactive
3. Override `render_with_context()` to use styles

### Phase 3: User Adoption
- Document StyleManager API
- Provide example apps
- Migration guide for existing apps

## Testing Strategy

### Unit Tests
- StyleCache hit/miss/invalidation
- Ancestor hash computation
- Animation overlay
- Theme switching

### Integration Tests
- Full style pipeline
- Hot reload
- Theme switching at runtime
- Animation lifecycle

### Widget Tests
- Each widget with DEFAULT_CSS
- Pseudo-class styling
- User CSS override

## Performance Considerations

- **Cache hit path:** O(1) HashMap lookup + 3 comparisons
- **Cache miss path:** O(R×S) selector matching
- **Animation tick:** O(A) where A = active animations
- **Memory:** ~250-450 bytes per cached widget

## Existing Components

All core components exist in `style.rs`:

| Component | Status |
|-----------|--------|
| StyleSheet | ✅ Complete |
| ThemeRegistry | ✅ Complete |
| Animator | ✅ Complete |
| HotReloadManager | ✅ Complete |
| WidgetMeta | ⚠️ Missing pseudo_classes |
| compute_style_resolved | ✅ Complete |
| ComputedStyle | ✅ Complete |

## Files Modified

### New Files
- `src/style_manager.rs` - StyleManager struct
- `src/render_context.rs` - RenderContext struct

### Modified Files
- `src/widget.rs` - Add DEFAULT_CSS, pseudo_classes(), widget_meta(), render_with_context()
- `src/style.rs` - Add pseudo_classes to WidgetMeta, add to_ratatui_style()
- `src/style.rs` - Extend stylesheet rules with origin metadata (default vs user)
- `src/app.rs` - Add style_manager to ManagedWidgetApp, update run_managed()
- `src/widgets/*.rs` - Add DEFAULT_CSS and render_with_context() to each widget

## Detailed Design Documents

1. [StyleManager Design](./style_manager.md)
2. [RenderContext Design](./render_context.md)
3. [Style Cache Design](./style_cache.md)
4. [Ancestor Chain Design](./ancestor_chain.md)
5. [Animated Values Design](./animated_values.md)

## Acceptance Criteria

1. ✅ StyleManager exists and holds all components
2. ✅ Widget::DEFAULT_CSS available on trait
3. ✅ register_widget_defaults() parses and merges CSS
4. ✅ load_user_stylesheet() loads with higher specificity
5. ⬜ 3+ widgets define DEFAULT_CSS and render using computed styles
6. ✅ ComputedStyle::to_ratatui_style() converts properties
7. ✅ RenderContext provides style_for() access
8. ⬜ Animator ticked in run_managed()
9. ⬜ Style invalidation works for all triggers
10. ⬜ Hot reload triggers style refresh
11. ⬜ Theme switching changes appearance
12. ⬜ All existing tests pass
13. ⬜ New integration tests verify pipeline
