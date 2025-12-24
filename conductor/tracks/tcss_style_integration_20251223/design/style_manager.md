# StyleManager Design

## Overview

`StyleManager` is the central coordinator for all styling concerns. It owns and orchestrates the existing style system components to provide a unified API for widget style computation.

## Existing Components (Already Implemented)

All core components exist in `style.rs`:

| Component | Line | Status |
|-----------|------|--------|
| `StyleSheet` | 4129 | ✅ Complete (parsing, matching, @keyframes) |
| `ThemeRegistry` | 6911 | ✅ Complete (built-in themes, switching) |
| `Animator` | 3376 | ✅ Complete (animations, tick, easing) |
| `HotReloadManager` | 7582 | ✅ Complete (file watching, polling) |
| `WidgetMeta` | 4270 | ⚠️ Missing `pseudo_classes` field |
| `compute_style_resolved` | 4826 | ✅ Complete (auto color resolution) |
| `ComputedStyle` | 4490 | ✅ Complete (all properties) |

## StyleManager Structure

```rust
pub struct StyleManager {
    /// Widget default stylesheet (DEFAULT_CSS from widgets).
    defaults: StyleSheet,

    /// User stylesheet (inline or file).
    user: Option<ReloadableStylesheet>,

    /// Effective stylesheet (defaults + user).
    stylesheet: StyleSheet,

    /// Active theme and theme switching.
    theme_registry: ThemeRegistry,

    /// Central animator for transitions and @keyframes.
    animator: Animator,

    /// Active transition overlays for non-numeric properties.
    transition_overlays: HashMap<(WidgetId, String), ActiveTransition>,

    /// File watching for .tcss changes (dev mode).
    hot_reload: Option<HotReloadManager>,

    /// Per-widget style cache with invalidation.
    cache: StyleCache,

    /// Current theme version (for cache invalidation).
    theme_version: u64,

    /// Path to user stylesheet (for hot reload).
    user_stylesheet_path: Option<PathBuf>,
}
```

## StyleCache Structure

```rust
pub struct StyleCache {
    /// Cached styles by widget ID.
    entries: RefCell<HashMap<WidgetId, StyleCacheEntry>>,

    /// Cache statistics (hits/misses/invalidations).
    stats: RefCell<CacheStats>,
}

pub struct StyleCacheEntry {
    /// The computed style.
    computed: ComputedStyle,

    /// Hash of ancestor chain at computation time.
    ancestor_hash: u64,

    /// Theme version at computation time.
    theme_version: u64,

    /// Hash of the widget's own styling inputs.
    widget_hash: u64,

    /// Hash of the parent background used for auto colors.
    parent_bg_hash: u64,
}
```

## Public API

### Construction

```rust
impl StyleManager {
    /// Create a new StyleManager with default settings.
    pub fn new() -> Self;

    /// Create with a builder pattern.
    pub fn builder() -> StyleManagerBuilder;
}

impl StyleManagerBuilder {
    /// Set initial theme.
    pub fn theme(self, name: &str) -> Self;

    /// Enable hot reload with watch paths.
    pub fn hot_reload(self, paths: Vec<PathBuf>) -> Self;

    /// Set animator FPS.
    pub fn fps(self, fps: u32) -> Self;

    /// Build the StyleManager.
    pub fn build(self) -> StyleManager;
}
```

### Widget Registration

```rust
impl StyleManager {
    /// Register a widget type's DEFAULT_CSS.
    ///
    /// Parses the CSS and registers rules as Default origin (lowest priority).
    pub fn register_widget_defaults<W: Widget>(&mut self) -> Result<(), StyleError>;

    /// Register all built-in widgets at once.
    pub fn register_builtin_widgets(&mut self) -> Result<(), StyleError>;
}
```

### User Stylesheet

```rust
impl StyleManager {
    /// Load user stylesheet from source string.
    ///
    /// Rules are registered as User origin (higher than defaults).
    pub fn load_user_stylesheet(&mut self, source: &str) -> Result<(), StyleError>;

    /// Load user stylesheet from file path (enables hot reload tracking).
    pub fn load_user_stylesheet_file(&mut self, path: impl AsRef<Path>) -> Result<(), StyleError>;
}
```

`load_user_stylesheet_file` should store a `ReloadableStylesheet` in `self.user`
and call `rebuild_stylesheet()` to merge `defaults` + `user`.

### Style Computation

```rust
impl StyleManager {
    /// Get computed style for a widget.
    ///
    /// Uses cached value if valid, otherwise computes fresh.
    /// Applies animated values as overlay on computed style.
    pub fn get_style(
        &self,
        widget_id: WidgetId,
        widget: &WidgetMeta,
        ancestors: &[&WidgetMeta],
        parent_background: &Color,
    ) -> ComputedStyle;

    /// Get current theme version (for external cache validation).
    pub fn theme_version(&self) -> u64;
}
```

### Theme Management

```rust
impl StyleManager {
    /// Get the active theme name.
    pub fn current_theme(&self) -> &str;

    /// Switch to a different theme.
    ///
    /// Increments theme_version and invalidates all cached styles.
    pub fn set_theme(&mut self, name: &str) -> Result<(), ThemeError>;

    /// Get available theme names.
    pub fn available_themes(&self) -> Vec<&str>;

    /// Register a custom theme.
    pub fn register_theme(&mut self, theme: Theme);
}
```

### Animation

```rust
impl StyleManager {
    /// Tick the animator. Called each frame by the run loop.
    pub fn tick(&mut self, now: Instant);

    /// Start an animation on a widget property.
    pub fn animate(
        &mut self,
        widget_id: WidgetId,
        property: &str,
        start_value: f64,
        end_value: f64,
        duration: Duration,
        easing: EasingFunction,
    );

    /// Cancel an animation.
    pub fn cancel_animation(&mut self, widget_id: WidgetId, property: &str);

    /// Check if a widget has active animations.
    pub fn has_animations(&self, widget_id: WidgetId) -> bool;
}
```

`animate()` is numeric-only (f64) via the existing Animator. Non-numeric transitions
use `ActiveTransition` overlays. `tick()` should also advance those overlays and
purge completed ones.

### Hot Reload

```rust
impl StyleManager {
    /// Enable hot reload with watch paths.
    pub fn enable_hot_reload(&mut self, paths: Vec<PathBuf>);

    /// Disable hot reload.
    pub fn disable_hot_reload(&mut self);

    /// Poll for file changes. Returns true if stylesheet was reloaded.
    ///
    /// If changes detected:
    /// 1. Reloads user stylesheet from file
    /// 2. Rebuilds effective stylesheet (defaults + user)
    /// 3. Increments theme_version
    /// 4. Invalidates all cached styles
    pub fn poll_hot_reload(&mut self) -> bool;
}
```

### Cache Invalidation

```rust
impl StyleManager {
    /// Invalidate cached style for a specific widget.
    ///
    /// Called when widget's classes, id, or pseudo-classes change.
    pub fn invalidate_widget(&mut self, widget_id: WidgetId);

    /// Invalidate all cached styles.
    ///
    /// Called on theme switch, hot reload, or global state change.
    pub fn invalidate_all(&mut self);
}
```

## Implementation Details

### Stylesheet Merging

Widget defaults and user CSS are combined into a single effective stylesheet,
but rule origin must be preserved to set `Specificity.is_user_css` at match time.

**Recommended approach (matches Python Textual behavior):**
- Extend `StyleSheet` to store rule origin + source order:

```rust
pub enum RuleOrigin {
    Default,
    User,
}

pub struct RuleEntry {
    pub rule: Rule,
    pub origin: RuleOrigin,
    pub source_order: u32,
}
```

Then, when matching:
1. Compute base specificity from selectors
2. Inject `is_user_css` based on `origin`
3. Use `source_order` as the final tie-breaker

This avoids mutating `Rule` and keeps source ordering deterministic.

```rust
fn rebuild_stylesheet(&mut self) {
    self.stylesheet = StyleSheet::merge_with_origins(&self.defaults, self.user.as_ref());
}
```

### Matching Rules with Origin

When gathering matching rules, inject origin into specificity:

```rust
let base = rule.specificity();
let specificity = base.with_user_css(origin == RuleOrigin::User);
```

### Cache Key Computation

The cache uses widget ID as primary key. Cache entries are validated against:

1. **Widget Hash**: type_name + id + classes + pseudo-classes
2. **Ancestor Hash**: Computed from ancestor chain (types, ids, classes, pseudo-classes)
3. **Theme Version**: Incremented on theme switch or hot reload
4. **Parent Background Hash**: Included when auto colors are present

```rust
fn compute_ancestor_hash(ancestors: &[&WidgetMeta]) -> u64 {
    use std::hash::{Hash, Hasher};
    use std::collections::hash_map::DefaultHasher;

    let mut hasher = DefaultHasher::new();

    for ancestor in ancestors {
        ancestor.type_name.hash(&mut hasher);
        if let Some(id) = &ancestor.id {
            id.hash(&mut hasher);
        }
        // Sort classes for deterministic hash
        let mut classes: Vec<_> = ancestor.classes.iter().collect();
        classes.sort();
        for class in classes {
            class.hash(&mut hasher);
        }
        // Sort pseudo-classes for deterministic hash
        let mut pseudo: Vec<_> = ancestor.pseudo_classes.iter().collect();
        pseudo.sort();
        for pc in pseudo {
            pc.hash(&mut hasher);
        }
    }

    hasher.finish()
}
```

### Animated Value Overlay

After computing base style, animated values are applied:

```rust
fn apply_animated_values(
    &self,
    widget_id: WidgetId,
    mut style: ComputedStyle,
) -> ComputedStyle {
    // Animator uses object_id (u64) + property name.
    // Requires `use slotmap::Key` for data()/as_ffi().
    let object_id = widget_id.data().as_ffi();

    // Numeric properties (Animator stores f64)
    if let Some(opacity) = self.animator.get_value(&AnimationKey::new(object_id, "opacity")) {
        style.opacity = Some(opacity);
    }

    // Non-numeric properties (color, spacing, etc.) should use the
    // TransitionController + ActiveTransition / interpolate_transition path.
    // This is handled by a separate transition overlay map (see animated_values.md).

    style
}
```

### Theme Variable Injection

When parsing stylesheets, theme variables are injected:

```rust
fn parse_with_theme(&self, source: &str) -> Result<StyleSheet, StyleError> {
    let theme = self.theme_registry.current()?;
    let variables = theme.to_stylesheet_variables();
    StyleSheet::parse_with_variables(source, &variables)
}
```

## WidgetMeta Extension

The existing `WidgetMeta` needs to be extended:

```rust
pub struct WidgetMeta {
    pub type_name: String,
    pub classes: Vec<String>,
    pub id: Option<String>,
    pub pseudo_classes: HashSet<String>,  // NEW
}
```

## Error Types

```rust
pub enum StyleError {
    /// CSS parse error.
    ParseError(String),
    /// Theme not found.
    ThemeError(ThemeError),
    /// File I/O error.
    IoError(std::io::Error),
}
```

## Thread Safety

`StyleManager` is **not** thread-safe by design. It should be owned by the main application struct and accessed on the main thread. Internally, the cache may use interior mutability so `get_style(&self)` can update cache stats without requiring a mutable borrow.

If thread-safe access is needed in the future, wrap in `Arc<Mutex<StyleManager>>`.

## Testing Strategy

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn test_register_widget_defaults() {
        let mut sm = StyleManager::new();
        sm.register_widget_defaults::<Button>().unwrap();

        let mut registry = WidgetRegistry::new();
        let meta = WidgetMeta::new("Button");
        let widget_id = registry.add(Button::new("Test"));
        let style = sm.get_style(widget_id, &meta, &[], &Color::default());

        // Verify default styles are applied
        assert!(style.background.is_some());
    }

    #[test]
    fn test_user_css_overrides_defaults() {
        let mut sm = StyleManager::new();
        sm.register_widget_defaults::<Button>().unwrap();
        sm.load_user_stylesheet("Button { background: red; }").unwrap();

        let mut registry = WidgetRegistry::new();
        let meta = WidgetMeta::new("Button");
        let widget_id = registry.add(Button::new("Test"));
        let style = sm.get_style(widget_id, &meta, &[], &Color::default());

        // User CSS should win
        assert_eq!(style.background.unwrap().r, 255);
    }

    #[test]
    fn test_theme_switch_invalidates_cache() {
        let mut sm = StyleManager::new();
        let v1 = sm.theme_version();

        sm.set_theme("textual-light").unwrap();
        let v2 = sm.theme_version();

        assert!(v2 > v1);
    }

    #[test]
    fn test_cache_hit() {
        let mut sm = StyleManager::new();
        sm.register_widget_defaults::<Button>().unwrap();

        let mut registry = WidgetRegistry::new();
        let meta = WidgetMeta::new("Button");
        let ancestors: &[&WidgetMeta] = &[];
        let widget_id = registry.add(Button::new("Test"));

        // First call computes
        let _ = sm.get_style(widget_id, &meta, ancestors, &Color::default());

        // Second call should hit cache (implementation detail, but verifiable via metrics)
        let _ = sm.get_style(widget_id, &meta, ancestors, &Color::default());
    }
}
```
