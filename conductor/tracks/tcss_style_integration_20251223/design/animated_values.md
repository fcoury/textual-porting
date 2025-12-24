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
    /// Widget identifier.
    pub widget_id: String,
    /// Property being animated.
    pub property: String,
}
```

For integration with `WidgetId`:

```rust
impl AnimationKey {
    pub fn new(widget_id: WidgetId, property: impl Into<String>) -> Self {
        Self {
            widget_id: widget_id.0.to_string(),
            property: property.into(),
        }
    }
}
```

## Animatable Properties

Not all CSS properties can be animated. The following are supported:

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

Colors are interpolated component-wise:

```rust
impl Animatable for RgbaColor {
    fn lerp(&self, other: &Self, t: f64) -> Self {
        RgbaColor::rgba(
            lerp_u8(self.r, other.r, t),
            lerp_u8(self.g, other.g, t),
            lerp_u8(self.b, other.b, t),
            lerp_f32(self.a, other.a, t as f32),
        )
    }
}

fn lerp_u8(a: u8, b: u8, t: f64) -> u8 {
    (a as f64 + (b as f64 - a as f64) * t).round() as u8
}
```

## Animator Integration

### Getting Current Values

```rust
impl Animator {
    /// Get all current animation values for a widget.
    pub fn values_for(&self, widget_id: WidgetId) -> Vec<(String, AnimatedValue)> {
        let prefix = widget_id.0.to_string();

        self.animations
            .iter()
            .filter(|(key, _)| key.widget_id == prefix)
            .map(|(key, state)| {
                let value = AnimatedValue::from_state(state);
                (key.property.clone(), value)
            })
            .collect()
    }

    /// Get a specific animated value if active.
    pub fn get_value(&self, widget_id: WidgetId, property: &str) -> Option<f64> {
        let key = AnimationKey::new(widget_id, property);
        self.animations.get(&key).map(|state| state.current_value())
    }
}

/// Typed animated value.
pub enum AnimatedValue {
    Number(f64),
    Color(RgbaColor),
}
```

### Overlay Application

```rust
impl StyleManager {
    fn apply_animations(&self, widget_id: WidgetId, mut style: ComputedStyle) -> ComputedStyle {
        for (property, value) in self.animator.values_for(widget_id) {
            match property.as_str() {
                "opacity" => {
                    if let AnimatedValue::Number(v) = value {
                        style.opacity = Some(v);
                    }
                }
                "color" => {
                    if let AnimatedValue::Color(c) = value {
                        style.color = Some(c);
                    }
                }
                "background" => {
                    if let AnimatedValue::Color(c) = value {
                        style.background = Some(c);
                    }
                }
                // Future: width, height, margin, padding
                _ => {
                    // Unknown property - ignore
                }
            }
        }
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
        let key = AnimationKey::new(widget_id, property);
        self.animator.animate(key, start, end, duration, easing, Duration::ZERO);
    }
}
```

### Ticking Animations

Each frame, the animator is ticked:

```rust
// In run_managed() loop
let dt = last_frame.elapsed();
app.style_manager_mut().tick(dt);

impl StyleManager {
    pub fn tick(&mut self, dt: Duration) {
        self.animator.tick(dt);
    }
}
```

### Animation Completion

When an animation completes:
1. The final value is held until explicitly cleared
2. Or the animation is removed from the Animator

```rust
impl Animator {
    pub fn tick(&mut self, dt: Duration) {
        for (key, state) in &mut self.animations {
            state.advance(dt);
        }

        // Remove completed animations
        self.animations.retain(|_, state| !state.completed || state.fill_mode.holds_end());
    }
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
2. If `opacity` differs from current and has a transition, start animation
3. Animator interpolates from current to target

### Transition Detection

```rust
impl StyleManager {
    fn maybe_transition(
        &mut self,
        widget_id: WidgetId,
        old_style: &ComputedStyle,
        new_style: &ComputedStyle,
    ) {
        // Check if new_style has transition properties
        if let Some(transition) = &new_style.transition {
            for prop in transition.properties() {
                let old_val = old_style.get_property(prop);
                let new_val = new_style.get_property(prop);

                if old_val != new_val {
                    // Start transition animation
                    self.animate(
                        widget_id,
                        prop,
                        old_val.to_f64(),
                        new_val.to_f64(),
                        transition.duration,
                        transition.timing,
                    );
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

        // ALWAYS apply animations (not cached)
        self.apply_animations(widget_id, base)
    }
}
```

This ensures animations appear immediately, even when the base style is cached.

## Performance Considerations

### Per-Frame Overhead

- `tick()`: O(A) where A = active animations
- `values_for()`: O(A) filter + collect
- `apply_animations()`: O(P) where P = animated properties per widget

For typical apps with <100 concurrent animations, this is negligible.

### Optimization: Skip When No Animations

```rust
impl StyleManager {
    pub fn get_style(&mut self, ...) -> ComputedStyle {
        let base = /* cache or compute */;

        // Fast path: no animations for this widget
        if !self.animator.has_animations(widget_id) {
            return base;
        }

        self.apply_animations(widget_id, base)
    }
}

impl Animator {
    pub fn has_animations(&self, widget_id: WidgetId) -> bool {
        let prefix = widget_id.0.to_string();
        self.animations.keys().any(|k| k.widget_id == prefix)
    }
}
```

## Testing

```rust
#[test]
fn test_animation_overlay() {
    let mut sm = StyleManager::new();
    sm.load_user_stylesheet("Button { opacity: 1.0; }").unwrap();

    let meta = WidgetMeta::new("Button");
    let ancestors: &[&WidgetMeta] = &[];

    // Initial style
    let style1 = sm.get_style(WidgetId(1), &meta, ancestors, &Color::default());
    assert_eq!(style1.opacity, Some(1.0));

    // Start animation
    sm.animate(WidgetId(1), "opacity", 1.0, 0.5, Duration::from_millis(100), EasingFunction::Linear);

    // Tick halfway
    sm.tick(Duration::from_millis(50));

    // Style should show animated value
    let style2 = sm.get_style(WidgetId(1), &meta, ancestors, &Color::default());
    assert!((style2.opacity.unwrap() - 0.75).abs() < 0.01);
}

#[test]
fn test_animation_completion() {
    let mut sm = StyleManager::new();
    sm.animate(WidgetId(1), "opacity", 1.0, 0.0, Duration::from_millis(100), EasingFunction::Linear);

    // Tick past completion
    sm.tick(Duration::from_millis(150));

    let meta = WidgetMeta::new("Button");
    let style = sm.get_style(WidgetId(1), &meta, &[], &Color::default());

    // Animation should have reached target
    assert_eq!(style.opacity, Some(0.0));
}

#[test]
fn test_cached_base_with_animation() {
    let mut sm = StyleManager::new();
    sm.load_user_stylesheet("Button { opacity: 1.0; }").unwrap();
    sm.register_widget_defaults::<Button>().unwrap();

    let meta = WidgetMeta::new("Button");
    let ancestors: &[&WidgetMeta] = &[];

    // Warm cache
    let _ = sm.get_style(WidgetId(1), &meta, ancestors, &Color::default());

    // Start animation
    sm.animate(WidgetId(1), "opacity", 1.0, 0.0, Duration::from_millis(100), EasingFunction::Linear);
    sm.tick(Duration::from_millis(50));

    // Should use cached base + animation overlay
    let style = sm.get_style(WidgetId(1), &meta, ancestors, &Color::default());
    assert!((style.opacity.unwrap() - 0.5).abs() < 0.01);
}
```
