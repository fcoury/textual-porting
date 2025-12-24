# Animated Value Overlay Design

## Overview

The animated value overlay applies dynamic animation values on top of computed CSS styles. This enables smooth property transitions without recomputing the entire style cascade each frame.

## Order of Operations

Per the spec (FR-6), style computation follows this order:

1. **Build WidgetMeta** with current pseudo-classes
2. **Compute base style** via `compute_style_resolved()` (theme variables, auto colors)
3. **Query Animator** for active animations on this widget
4. **Apply animated values** as property overrides
5. **Return final ComputedStyle**

```
                ┌─────────────────┐
                │  WidgetMeta     │
                │  + ancestors    │
                └────────┬────────┘
                         │
                         ▼
            ┌────────────────────────┐
            │ compute_style_resolved │
            │  - Match selectors     │
            │  - Apply cascade       │
            │  - Resolve theme vars  │
            │  - Resolve auto colors │
            └────────────┬───────────┘
                         │
                         ▼
                ┌────────────────┐
                │  Base Style    │
                │  (from CSS)    │
                └────────┬───────┘
                         │
                         ▼
            ┌────────────────────────┐
            │  Apply Animator Values │
            │  - Query active anims  │
            │  - Overlay on base     │
            └────────────┬───────────┘
                         │
                         ▼
                ┌────────────────┐
                │  Final Style   │
                │  (rendered)    │
                └────────────────┘
```

## Animation Key Structure

The existing `AnimationKey` identifies an animation:

```rust
// From style.rs
#[derive(Clone, Debug, Hash, PartialEq, Eq)]
pub struct AnimationKey {
    /// Target object ID.
    pub object_id: u64,
    /// Property being animated.
    pub property: String,
}
```

For integration with `WidgetId`, derive a stable `u64` object_id from the slotmap
key data (e.g., `widget_id.data().as_ffi()`), and use that with `AnimationKey::new`.

```rust
impl AnimationKey {
    pub fn new(object_id: u64, property: impl Into<String>) -> Self {
        Self {
            object_id,
            property: property.into(),
        }
    }
}
```

## Animatable Properties

Not all CSS properties can be animated. The authoritative list is defined by
`is_property_animatable()` / `ANIMATABLE_PROPERTIES` in `style.rs`.

Common categories:

| Property | Animation Type | Notes |
|----------|---------------|-------|
| `opacity` | f64 → f64 | Linear interpolation |
| `color` | Color → Color | RGB component lerp |
| `background` | Color → Color | RGB component lerp |
| `width` | Scalar → Scalar | For fixed units only |
| `height` | Scalar → Scalar | For fixed units only |
| `margin-*` | Scalar → Scalar | Individual sides |
| `padding-*` | Scalar → Scalar | Individual sides |

### Color Animation

Colors are interpolated via the existing `Animatable for Color` implementation
in `style.rs`, which blends RGB (+ alpha) and snaps ANSI/auto colors at 50%.

## Animator Integration

### Getting Current Values

```rust
impl Animator {
    /// Get a specific animated value if active.
    pub fn get_value(&self, key: &AnimationKey) -> Option<f64>;
}
```

### Overlay Application

```rust
impl StyleManager {
    fn apply_animations(&self, widget_id: WidgetId, mut style: ComputedStyle) -> ComputedStyle {
    // Requires `use slotmap::Key` for data()/as_ffi().
    let object_id = widget_id.data().as_ffi();

        if let Some(opacity) = self.animator.get_value(&AnimationKey::new(object_id, "opacity")) {
            style.opacity = Some(opacity);
        }

        // Non-numeric properties (color, spacing, etc.) use the transition overlay
        // path (ActiveTransition + interpolate_transition) instead of Animator.
        style
    }
}
```

## Animation Lifecycle

### Starting an Animation

```rust
// Via StyleManager API
style_manager.animate(
    widget_id,
    "opacity",
    0.0,  // start
    1.0,  // end
    Duration::from_millis(300),
    EasingFunction::EaseOut,
);

// Internally creates AnimationKey and delegates to Animator
impl StyleManager {
    pub fn animate(
        &mut self,
        widget_id: WidgetId,
        property: &str,
        start: f64,
        end: f64,
        duration: Duration,
        easing: EasingFunction,
    ) {
        let object_id = widget_id.data().as_ffi();
        let key = AnimationKey::new(object_id, property);
        self.animator.animate(key, start, end, duration, easing, Duration::ZERO);
    }
}
```

### Ticking Animations

Each frame, the animator is ticked:

```rust
// In run_managed() loop
let now = Instant::now();
app.style_manager_mut().tick(now);

impl StyleManager {
    pub fn tick(&mut self, now: Instant) {
        self.animator.tick(now);
        self.tick_transition_overlays();
    }
}
```

### Animation Completion

When an animation completes:
1. The final value is held until explicitly cleared
2. Or the animation is removed from the Animator

For non-numeric transitions stored as `ActiveTransition`, `StyleManager` should
tick and remove completed overlays during `tick_transition_overlays()`.

```rust
fn tick_transition_overlays(&mut self) {
    self.transition_overlays.retain(|_, t| {
        let _events = t.tick();
        !t.is_complete()
    });
}
```

```rust
impl Animator {
    /// Tick animations and return any keys that completed.
    pub fn tick(&mut self, now: Instant) -> Vec<AnimationKey>;
}
```

## CSS Transitions Support

CSS transitions are triggered by property changes:

```css
Button {
    transition: opacity 300ms ease-out;
}

Button:hover {
    opacity: 0.8;
}
```

When hover state changes:
1. Style recomputes with new pseudo-class
2. Diff old/new style values for animatable properties
3. If a transition is defined, start a numeric animation or register an
   ActiveTransition overlay for non-numeric values

### Transition Detection

```rust
impl StyleManager {
    fn maybe_transition(
        &mut self,
        widget_id: WidgetId,
        old_style: &ComputedStyle,
        new_style: &ComputedStyle,
    ) {
        let controller = TransitionController::from_transitions(
            new_style
                .transition
                .clone()
                .unwrap_or_else(TransitionSet::new),
        );

        for (prop, old_val, new_val) in diff_style_values(old_style, new_style) {
            if let Some(params) = controller.get_transition_params(&prop, &old_val, &new_val) {
                let pending = PendingTransition::new(
                    widget_id.data().as_ffi(),
                    prop.clone(),
                    old_val.clone(),
                    new_val.clone(),
                    params,
                );

                // Numeric values go through Animator; others use ActiveTransition overlay
                if !pending.start(&mut self.animator) {
                    self.transition_overlays
                        .insert((widget_id, prop.clone()), ActiveTransition::new(
                            widget_id.data().as_ffi(),
                            prop,
                            old_val,
                            new_val,
                            params,
                        ));
                }
            }
        }
    }
}
```

## Cache Interaction

Animated values bypass the style cache:

```rust
impl StyleManager {
    pub fn get_style(
        &mut self,
        widget_id: WidgetId,
        widget: &WidgetMeta,
        ancestors: &[&WidgetMeta],
        parent_background: &Color,
    ) -> ComputedStyle {
        // Cache lookup for BASE style
        let base = if let Some(cached) = self.cache.get(...) {
            cached.clone()
        } else {
            let computed = compute_style_resolved(...);
            self.cache.insert(..., computed.clone());
            computed
        };

// ALWAYS apply animations/transition overlays (not cached)
self.apply_animations(widget_id, base)
    }
}
```

This ensures animations appear immediately, even when the base style is cached.

## Performance Considerations

### Per-Frame Overhead

- `tick()`: O(A) where A = active animations
- `get_value()`: O(1) per animated numeric property
- `apply_animations()`: O(P) where P = animated properties per widget

For typical apps with <100 concurrent animations, this is negligible.

### Optimization: Skip When No Animations

```rust
impl StyleManager {
    pub fn get_style(&self, ...) -> ComputedStyle {
        let base = /* cache or compute */;

        // Fast path: no numeric animations and no transition overlays
        let object_id = widget_id.data().as_ffi();
        let has_overlay = self
            .transition_overlays
            .keys()
            .any(|(id, _)| *id == widget_id);

        if !self.animator.is_animating(&AnimationKey::new(object_id, "opacity"))
            && !has_overlay
        {
            return base;
        }

        self.apply_animations(widget_id, base)
    }
}
```

## Testing

```rust
#[test]
fn test_animation_overlay() {
    let mut sm = StyleManager::new();
    sm.load_user_stylesheet("Button { opacity: 1.0; }").unwrap();

    let mut registry = WidgetRegistry::new();
    let meta = WidgetMeta::new("Button");
    let ancestors: &[&WidgetMeta] = &[];
    let widget_id = registry.add(Button::new("Test"));

    // Initial style
    let style1 = sm.get_style(widget_id, &meta, ancestors, &Color::default());
    assert_eq!(style1.opacity, Some(1.0));

    // Start animation
    sm.animate(widget_id, "opacity", 1.0, 0.5, Duration::from_millis(100), EasingFunction::Linear);

    // Tick halfway
    sm.tick(Instant::now() + Duration::from_millis(50));

    // Style should show animated value
    let style2 = sm.get_style(widget_id, &meta, ancestors, &Color::default());
    assert!((style2.opacity.unwrap() - 0.75).abs() < 0.01);
}

#[test]
fn test_animation_completion() {
    let mut sm = StyleManager::new();
    let mut registry = WidgetRegistry::new();
    let widget_id = registry.add(Button::new("Test"));
    sm.animate(widget_id, "opacity", 1.0, 0.0, Duration::from_millis(100), EasingFunction::Linear);

    // Tick past completion
    sm.tick(Instant::now() + Duration::from_millis(150));
    let meta = WidgetMeta::new("Button");
    let style = sm.get_style(widget_id, &meta, &[], &Color::default());

    // Animation should have reached target
    assert_eq!(style.opacity, Some(0.0));
}

#[test]
fn test_cached_base_with_animation() {
    let mut sm = StyleManager::new();
    sm.load_user_stylesheet("Button { opacity: 1.0; }").unwrap();
    sm.register_widget_defaults::<Button>().unwrap();

    let mut registry = WidgetRegistry::new();
    let meta = WidgetMeta::new("Button");
    let ancestors: &[&WidgetMeta] = &[];

    // Warm cache
    let widget_id = registry.add(Button::new("Test"));
    let _ = sm.get_style(widget_id, &meta, ancestors, &Color::default());

    // Start animation
    sm.animate(widget_id, "opacity", 1.0, 0.0, Duration::from_millis(100), EasingFunction::Linear);
    sm.tick(Instant::now() + Duration::from_millis(50));

    // Should use cached base + animation overlay
    let style = sm.get_style(widget_id, &meta, ancestors, &Color::default());
    assert!((style.opacity.unwrap() - 0.5).abs() < 0.01);
}
```
