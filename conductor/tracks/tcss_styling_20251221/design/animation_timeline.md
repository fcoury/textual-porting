# Design: Animation Timeline Architecture

## Overview

This document defines the animation system architecture for TCSS transitions and animations. The system supports property interpolation, easing functions, and a 60fps timer-based update loop.

**Key Constraint:** Duration and speed are mutually exclusive - exactly one must be specified.

## Animation Types

### Transition

CSS transitions animate property changes automatically:

```rust
/// Definition from CSS transition property
#[derive(Debug, Clone, PartialEq)]
pub struct Transition {
    /// Property to transition (or "all")
    pub property: TransitionProperty,
    /// Duration of the transition
    pub duration: Duration,
    /// Easing function
    pub easing: EasingFunction,
    /// Delay before starting
    pub delay: Duration,
}

#[derive(Debug, Clone, PartialEq)]
pub enum TransitionProperty {
    /// Specific property name
    Named(PropertyName),
    /// All animatable properties
    All,
}

impl Default for Transition {
    fn default() -> Self {
        Transition {
            property: TransitionProperty::All,
            duration: Duration::from_millis(250),
            easing: EasingFunction::Linear,
            delay: Duration::ZERO,
        }
    }
}
```

### Animation Instance

A running animation tracking current state:

```rust
/// Key for identifying active animations
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct AnimationKey {
    /// Target object ID
    pub object_id: ObjectId,
    /// Property being animated
    pub property: PropertyName,
}

/// Base trait for all animations
pub trait Animation: Send {
    /// Advance animation, returns true if complete
    fn tick(&mut self, time: Instant, level: AnimationLevel) -> bool;

    /// Stop the animation
    fn stop(&mut self, complete: bool);

    /// Get callback to run on completion
    fn on_complete(&self) -> Option<&AnimationCallback>;
}

/// Simple property animation
pub struct SimpleAnimation<T: Animatable> {
    /// Start time of animation
    start_time: Instant,
    /// Animation duration
    duration: Duration,
    /// Starting value
    start_value: T,
    /// Target value for interpolation
    end_value: T,
    /// Final value to set (may differ from end_value)
    final_value: T,
    /// Easing function
    easing: EasingFunction,
    /// Completion callback
    on_complete: Option<AnimationCallback>,
    /// Animation level (none, basic, full)
    level: AnimationLevel,
    /// Setter function to apply value
    setter: Box<dyn FnMut(T) + Send>,
}

impl<T: Animatable> Animation for SimpleAnimation<T> {
    fn tick(&mut self, time: Instant, app_level: AnimationLevel) -> bool {
        // Skip if animation level too low
        if !self.should_animate(app_level) {
            (self.setter)(self.final_value.clone());
            return true;
        }

        // Calculate progress
        let elapsed = time.duration_since(self.start_time);
        let factor = (elapsed.as_secs_f64() / self.duration.as_secs_f64())
            .clamp(0.0, 1.0);

        // Apply easing
        let eased_factor = self.easing.apply(factor);

        // Interpolate value
        let value = self.start_value.blend(&self.end_value, eased_factor);
        (self.setter)(value);

        // Check if complete
        if factor >= 1.0 {
            (self.setter)(self.final_value.clone());
            true
        } else {
            false
        }
    }

    fn stop(&mut self, complete: bool) {
        if complete {
            (self.setter)(self.final_value.clone());
        }
    }

    fn on_complete(&self) -> Option<&AnimationCallback> {
        self.on_complete.as_ref()
    }
}
```

### Animation Level

Controls which animations run:

```rust
/// Global animation level setting
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Default)]
pub enum AnimationLevel {
    /// No animations - set final values immediately
    None,
    /// Basic/essential animations only
    Basic,
    /// All animations enabled
    #[default]
    Full,
}

impl SimpleAnimation<T> {
    fn should_animate(&self, app_level: AnimationLevel) -> bool {
        match self.level {
            AnimationLevel::None => false,
            AnimationLevel::Basic => app_level >= AnimationLevel::Basic,
            AnimationLevel::Full => app_level >= AnimationLevel::Full,
        }
    }
}
```

## Easing Functions

```rust
/// Standard easing functions
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum EasingFunction {
    #[default]
    Linear,

    // Sine
    InSine,
    OutSine,
    InOutSine,

    // Quadratic
    InQuad,
    OutQuad,
    InOutQuad,

    // Cubic
    InCubic,
    OutCubic,
    InOutCubic,

    // Quartic
    InQuart,
    OutQuart,
    InOutQuart,

    // Quintic
    InQuint,
    OutQuint,
    InOutQuint,

    // Exponential
    InExpo,
    OutExpo,
    InOutExpo,

    // Circular
    InCirc,
    OutCirc,
    InOutCirc,

    // Back (overshoot)
    InBack,
    OutBack,
    InOutBack,

    // Elastic
    InElastic,
    OutElastic,
    InOutElastic,

    // Bounce
    InBounce,
    OutBounce,
    InOutBounce,
}

impl EasingFunction {
    /// Apply easing to a factor (0.0 to 1.0)
    pub fn apply(&self, t: f64) -> f64 {
        use std::f64::consts::PI;

        match self {
            EasingFunction::Linear => t,

            // Sine
            EasingFunction::InSine => 1.0 - ((t * PI) / 2.0).cos(),
            EasingFunction::OutSine => ((t * PI) / 2.0).sin(),
            EasingFunction::InOutSine => -((PI * t).cos() - 1.0) / 2.0,

            // Quadratic
            EasingFunction::InQuad => t * t,
            EasingFunction::OutQuad => 1.0 - (1.0 - t).powi(2),
            EasingFunction::InOutQuad => {
                if t < 0.5 {
                    2.0 * t * t
                } else {
                    1.0 - (-2.0 * t + 2.0).powi(2) / 2.0
                }
            }

            // Cubic
            EasingFunction::InCubic => t * t * t,
            EasingFunction::OutCubic => 1.0 - (1.0 - t).powi(3),
            EasingFunction::InOutCubic => {
                if t < 0.5 {
                    4.0 * t * t * t
                } else {
                    1.0 - (-2.0 * t + 2.0).powi(3) / 2.0
                }
            }

            // Quartic
            EasingFunction::InQuart => t * t * t * t,
            EasingFunction::OutQuart => 1.0 - (1.0 - t).powi(4),
            EasingFunction::InOutQuart => {
                if t < 0.5 {
                    8.0 * t * t * t * t
                } else {
                    1.0 - (-2.0 * t + 2.0).powi(4) / 2.0
                }
            }

            // Quintic
            EasingFunction::InQuint => t * t * t * t * t,
            EasingFunction::OutQuint => 1.0 - (1.0 - t).powi(5),
            EasingFunction::InOutQuint => {
                if t < 0.5 {
                    16.0 * t * t * t * t * t
                } else {
                    1.0 - (-2.0 * t + 2.0).powi(5) / 2.0
                }
            }

            // Exponential
            EasingFunction::InExpo => {
                if t == 0.0 { 0.0 } else { 2.0_f64.powf(10.0 * t - 10.0) }
            }
            EasingFunction::OutExpo => {
                if t == 1.0 { 1.0 } else { 1.0 - 2.0_f64.powf(-10.0 * t) }
            }
            EasingFunction::InOutExpo => {
                if t == 0.0 {
                    0.0
                } else if t == 1.0 {
                    1.0
                } else if t < 0.5 {
                    2.0_f64.powf(20.0 * t - 10.0) / 2.0
                } else {
                    (2.0 - 2.0_f64.powf(-20.0 * t + 10.0)) / 2.0
                }
            }

            // Circular
            EasingFunction::InCirc => 1.0 - (1.0 - t * t).sqrt(),
            EasingFunction::OutCirc => (1.0 - (t - 1.0).powi(2)).sqrt(),
            EasingFunction::InOutCirc => {
                if t < 0.5 {
                    (1.0 - (1.0 - (2.0 * t).powi(2)).sqrt()) / 2.0
                } else {
                    ((1.0 - (-2.0 * t + 2.0).powi(2)).sqrt() + 1.0) / 2.0
                }
            }

            // Back (overshoot)
            EasingFunction::InBack => {
                const C1: f64 = 1.70158;
                const C3: f64 = C1 + 1.0;
                C3 * t * t * t - C1 * t * t
            }
            EasingFunction::OutBack => {
                const C1: f64 = 1.70158;
                const C3: f64 = C1 + 1.0;
                1.0 + C3 * (t - 1.0).powi(3) + C1 * (t - 1.0).powi(2)
            }
            EasingFunction::InOutBack => {
                const C1: f64 = 1.70158;
                const C2: f64 = C1 * 1.525;
                if t < 0.5 {
                    ((2.0 * t).powi(2) * ((C2 + 1.0) * 2.0 * t - C2)) / 2.0
                } else {
                    ((2.0 * t - 2.0).powi(2) * ((C2 + 1.0) * (t * 2.0 - 2.0) + C2) + 2.0) / 2.0
                }
            }

            // Elastic
            EasingFunction::InElastic => {
                const C4: f64 = (2.0 * PI) / 3.0;
                if t == 0.0 {
                    0.0
                } else if t == 1.0 {
                    1.0
                } else {
                    -(2.0_f64.powf(10.0 * t - 10.0)) * ((t * 10.0 - 10.75) * C4).sin()
                }
            }
            EasingFunction::OutElastic => {
                const C4: f64 = (2.0 * PI) / 3.0;
                if t == 0.0 {
                    0.0
                } else if t == 1.0 {
                    1.0
                } else {
                    2.0_f64.powf(-10.0 * t) * ((t * 10.0 - 0.75) * C4).sin() + 1.0
                }
            }
            EasingFunction::InOutElastic => {
                const C5: f64 = (2.0 * PI) / 4.5;
                if t == 0.0 {
                    0.0
                } else if t == 1.0 {
                    1.0
                } else if t < 0.5 {
                    -(2.0_f64.powf(20.0 * t - 10.0) * ((20.0 * t - 11.125) * C5).sin()) / 2.0
                } else {
                    (2.0_f64.powf(-20.0 * t + 10.0) * ((20.0 * t - 11.125) * C5).sin()) / 2.0 + 1.0
                }
            }

            // Bounce
            EasingFunction::InBounce => 1.0 - EasingFunction::OutBounce.apply(1.0 - t),
            EasingFunction::OutBounce => {
                const N1: f64 = 7.5625;
                const D1: f64 = 2.75;
                if t < 1.0 / D1 {
                    N1 * t * t
                } else if t < 2.0 / D1 {
                    let t = t - 1.5 / D1;
                    N1 * t * t + 0.75
                } else if t < 2.5 / D1 {
                    let t = t - 2.25 / D1;
                    N1 * t * t + 0.9375
                } else {
                    let t = t - 2.625 / D1;
                    N1 * t * t + 0.984375
                }
            }
            EasingFunction::InOutBounce => {
                if t < 0.5 {
                    (1.0 - EasingFunction::OutBounce.apply(1.0 - 2.0 * t)) / 2.0
                } else {
                    (1.0 + EasingFunction::OutBounce.apply(2.0 * t - 1.0)) / 2.0
                }
            }
        }
    }

    /// Parse from CSS string
    pub fn from_str(s: &str) -> Option<Self> {
        match s.to_lowercase().replace('-', "_").as_str() {
            "linear" => Some(EasingFunction::Linear),
            "in_sine" | "ease_in" => Some(EasingFunction::InSine),
            "out_sine" | "ease_out" => Some(EasingFunction::OutSine),
            "in_out_sine" | "ease_in_out" => Some(EasingFunction::InOutSine),
            "in_quad" => Some(EasingFunction::InQuad),
            "out_quad" => Some(EasingFunction::OutQuad),
            "in_out_quad" => Some(EasingFunction::InOutQuad),
            "in_cubic" => Some(EasingFunction::InCubic),
            "out_cubic" => Some(EasingFunction::OutCubic),
            "in_out_cubic" => Some(EasingFunction::InOutCubic),
            // ... etc
            _ => None,
        }
    }
}
```

## Animator

The central animation controller:

```rust
/// Animation callback type
pub type AnimationCallback = Box<dyn FnOnce() + Send>;

/// Central animation controller
pub struct Animator {
    /// Active animations by key
    animations: HashMap<AnimationKey, Box<dyn Animation>>,
    /// Scheduled animations (delayed start)
    scheduled: HashMap<AnimationKey, ScheduledAnimation>,
    /// Target frames per second
    fps: u32,
    /// Timer handle for animation loop
    timer_handle: Option<TimerHandle>,
    /// Current animation level
    level: AnimationLevel,
}

struct ScheduledAnimation {
    animation: Box<dyn Animation>,
    start_at: Instant,
}

impl Animator {
    pub fn new(fps: u32) -> Self {
        Animator {
            animations: HashMap::new(),
            scheduled: HashMap::new(),
            fps,
            timer_handle: None,
            level: AnimationLevel::Full,
        }
    }

    /// Start an animation
    ///
    /// # Arguments
    /// * `key` - Unique animation identifier
    /// * `start_value` - Initial value
    /// * `end_value` - Target value
    /// * `timing` - Duration OR speed (mutually exclusive)
    /// * `easing` - Easing function
    /// * `delay` - Start delay
    /// * `on_complete` - Completion callback
    /// * `setter` - Function to apply interpolated values
    pub fn animate<T: Animatable + 'static>(
        &mut self,
        key: AnimationKey,
        start_value: T,
        end_value: T,
        timing: AnimationTiming,
        easing: EasingFunction,
        delay: Duration,
        on_complete: Option<AnimationCallback>,
        setter: impl FnMut(T) + Send + 'static,
    ) {
        // Cancel any existing animation for this key
        self.cancel(&key);

        // Calculate duration from timing
        let duration = match timing {
            AnimationTiming::Duration(d) => d,
            AnimationTiming::Speed(speed) => {
                let distance = start_value.distance_to(&end_value);
                Duration::from_secs_f64(distance / speed)
            }
        };

        let animation = Box::new(SimpleAnimation {
            start_time: Instant::now(),
            duration,
            start_value,
            end_value: end_value.clone(),
            final_value: end_value,
            easing,
            on_complete,
            level: AnimationLevel::Full,
            setter: Box::new(setter),
        });

        if delay.is_zero() {
            self.animations.insert(key, animation);
            self.ensure_timer_running();
        } else {
            self.scheduled.insert(key, ScheduledAnimation {
                animation,
                start_at: Instant::now() + delay,
            });
        }
    }

    /// Cancel an animation
    pub fn cancel(&mut self, key: &AnimationKey) {
        self.animations.remove(key);
        self.scheduled.remove(key);
    }

    /// Stop an animation, optionally completing it
    pub fn stop(&mut self, key: &AnimationKey, complete: bool) {
        if let Some(mut animation) = self.animations.remove(key) {
            animation.stop(complete);
            if complete {
                if let Some(callback) = animation.on_complete() {
                    // Note: in real impl, would need to handle this differently
                }
            }
        }
        self.scheduled.remove(key);
    }

    /// Process one animation frame
    pub fn tick(&mut self, time: Instant) {
        // Promote scheduled animations that are ready
        let ready: Vec<_> = self.scheduled
            .iter()
            .filter(|(_, s)| time >= s.start_at)
            .map(|(k, _)| k.clone())
            .collect();

        for key in ready {
            if let Some(scheduled) = self.scheduled.remove(&key) {
                self.animations.insert(key, scheduled.animation);
            }
        }

        // Tick active animations
        let mut completed = Vec::new();
        for (key, animation) in &mut self.animations {
            if animation.tick(time, self.level) {
                completed.push(key.clone());
            }
        }

        // Remove completed animations and run callbacks
        for key in completed {
            if let Some(animation) = self.animations.remove(&key) {
                if let Some(callback) = animation.on_complete() {
                    // callback();  // Would need ownership
                }
            }
        }

        // Stop timer if no animations remain
        if self.animations.is_empty() && self.scheduled.is_empty() {
            self.stop_timer();
        }
    }

    /// Check if any animations are active
    pub fn is_animating(&self) -> bool {
        !self.animations.is_empty() || !self.scheduled.is_empty()
    }

    fn ensure_timer_running(&mut self) {
        if self.timer_handle.is_none() {
            let interval = Duration::from_secs_f64(1.0 / self.fps as f64);
            // self.timer_handle = Some(start_timer(interval, ...));
        }
    }

    fn stop_timer(&mut self) {
        if let Some(handle) = self.timer_handle.take() {
            // handle.cancel();
        }
    }
}

/// Timing specification (duration XOR speed)
#[derive(Debug, Clone, Copy)]
pub enum AnimationTiming {
    /// Fixed duration
    Duration(Duration),
    /// Speed (units per second)
    Speed(f64),
}
```

## BoundAnimator

Convenience wrapper for animating a specific object:

```rust
/// Animator bound to a specific object
pub struct BoundAnimator<'a> {
    animator: &'a mut Animator,
    object_id: ObjectId,
}

impl<'a> BoundAnimator<'a> {
    pub fn new(animator: &'a mut Animator, object_id: ObjectId) -> Self {
        BoundAnimator { animator, object_id }
    }

    /// Animate a property on the bound object
    pub fn animate<T: Animatable + 'static>(
        &mut self,
        property: PropertyName,
        start_value: T,
        end_value: T,
        duration: Duration,
        easing: EasingFunction,
        setter: impl FnMut(T) + Send + 'static,
    ) {
        let key = AnimationKey {
            object_id: self.object_id.clone(),
            property,
        };

        self.animator.animate(
            key,
            start_value,
            end_value,
            AnimationTiming::Duration(duration),
            easing,
            Duration::ZERO,
            None,
            setter,
        );
    }
}
```

## Custom Animation Hook

Objects can customize animation creation:

```rust
/// Trait for objects that can customize their animations
pub trait CustomAnimatable {
    /// Create custom animation for a property change
    ///
    /// Return None to use default SimpleAnimation
    fn create_animation(
        &self,
        property: PropertyName,
        start_value: StyleValue,
        end_value: StyleValue,
        start_time: Instant,
        duration: Option<Duration>,
        speed: Option<f64>,
        easing: EasingFunction,
        on_complete: Option<AnimationCallback>,
        level: AnimationLevel,
    ) -> Option<Box<dyn Animation>>;
}
```

## Integration with Styles

```rust
impl RenderStyles {
    /// Apply a property change with potential animation
    pub fn set_animated(
        &mut self,
        property: PropertyName,
        value: StyleValue,
        animator: &mut Animator,
        object_id: ObjectId,
    ) {
        // Check for transition definition
        if let Some(transition) = self.get_transition(&property) {
            if property.is_animatable() {
                let current = self.get(property);

                // Only animate if value actually changed
                if current != value {
                    let key = AnimationKey { object_id, property };

                    // Check for custom animation hook
                    // if let Some(anim) = self.create_custom_animation(...) {
                    //     animator.add(key, anim);
                    // } else {
                        animator.animate(
                            key,
                            current.clone(),
                            value.clone(),
                            AnimationTiming::Duration(transition.duration),
                            transition.easing,
                            transition.delay,
                            None,
                            move |v| { /* apply value */ },
                        );
                    // }

                    return;
                }
            }
        }

        // No animation - set directly
        self.set(property, value);
    }

    /// Get transition for a property
    fn get_transition(&self, property: &PropertyName) -> Option<&Transition> {
        // Check for specific property transition
        if let Some(t) = self.transitions.get(property) {
            return Some(t);
        }
        // Check for "all" transition
        self.transitions.get(&TransitionProperty::All)
    }
}
```

## Frame Rate and Timing

```rust
/// Default animation frame rate
pub const DEFAULT_FPS: u32 = 60;

/// Minimum frame duration (16.67ms at 60fps)
pub const MIN_FRAME_DURATION: Duration = Duration::from_micros(16_667);

impl Animator {
    /// Get frame interval
    pub fn frame_interval(&self) -> Duration {
        Duration::from_secs_f64(1.0 / self.fps as f64)
    }

    /// Set target FPS
    pub fn set_fps(&mut self, fps: u32) {
        self.fps = fps.max(1).min(120);  // Clamp to reasonable range
        // Restart timer with new interval if running
        if self.timer_handle.is_some() {
            self.stop_timer();
            self.ensure_timer_running();
        }
    }
}
```
