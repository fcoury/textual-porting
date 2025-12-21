# Analysis: Python Textual _animator.py

## Overview

The `_animator.py` file implements the animation system for CSS property transitions. It manages interpolation between values over time with easing functions.

## Key Components

### 1. Animatable Protocol

```python
@runtime_checkable
class Animatable(Protocol):
    """Protocol for objects that can have their intrinsic values animated."""

    def blend(
        self: ReturnType, destination: ReturnType, factor: float
    ) -> ReturnType:
        """Blend between self and destination by factor (0.0-1.0)"""
        ...
```

Types implementing Animatable:
- `Color` - RGB/A interpolation
- `Scalar` - Dimension values
- `Spacing` - Padding/margin
- `ScalarOffset` - X/Y offsets

### 2. Animation Classes

**Base Animation:**
```python
class Animation(ABC):
    on_complete: CallbackType | None = None

    @abstractmethod
    def __call__(self, time: float, app_animation_level: AnimationLevel) -> bool:
        """Run animation step, return True if complete."""

    @abstractmethod
    async def stop(self, complete: bool = True) -> None:
        """Stop the animation."""
```

**SimpleAnimation:**
```python
@dataclass
class SimpleAnimation(Animation):
    obj: object              # Target object
    attribute: str           # Attribute name
    start_time: float        # Animation start time
    duration: float          # Animation duration
    start_value: float | Animatable
    end_value: float | Animatable
    final_value: object      # Value to set when complete
    easing: EasingFunction
    on_complete: CallbackType | None = None
    level: AnimationLevel = "full"  # "none" | "basic" | "full"
```

### 3. Animator Class

```python
class Animator:
    def __init__(self, app: App, frames_per_second: int = 60):
        self._animations: dict[AnimationKey, Animation] = {}
        self._scheduled: dict[AnimationKey, Timer] = {}
        self._timer = Timer(app, 1/frames_per_second, ...)

    def animate(
        self,
        obj: object,
        attribute: str,
        value: Any,
        *,
        final_value: object = ...,
        duration: float | None = None,
        speed: float | None = None,
        easing: EasingFunction | str = DEFAULT_EASING,
        delay: float = 0.0,
        on_complete: CallbackType | None = None,
        level: AnimationLevel = "full",
    ) -> None:
        ...
```

### 4. BoundAnimator

Convenience wrapper bound to a specific object:

```python
class BoundAnimator:
    def __init__(self, animator: Animator, obj: object):
        self._animator = animator
        self._obj = obj

    def __call__(self, attribute: str, value: ..., **kwargs):
        """Animate attribute on the bound object."""
        return self._animator.animate(self._obj, attribute, value, **kwargs)
```

## Animation Flow

1. **Schedule Animation**
   ```python
   animator.animate(obj, "color", Color.parse("red"), duration=0.3)
   ```

2. **Timer Tick (60 fps)**
   ```python
   def __call__(self):
       for animation_key in animation_keys:
           animation = self._animations[animation_key]
           animation_complete = animation(animation_time, app_animation_level)
           if animation_complete:
               del self._animations[animation_key]
               if animation.on_complete:
                   animation.on_complete()
   ```

3. **Value Interpolation**
   ```python
   def __call__(self, time: float, ...) -> bool:
       factor = min(1.0, (time - self.start_time) / self.duration)
       eased_factor = self.easing(factor)

       if isinstance(self.start_value, Animatable):
           value = self.start_value.blend(self.end_value, eased_factor)
       else:
           # Linear interpolation for floats
           value = self.start_value + (self.end_value - self.start_value) * eased_factor

       setattr(self.obj, self.attribute, value)
       return factor >= 1
   ```

## Easing Functions

From `_easing.py`:

```python
EASING = {
    "linear": lambda x: x,
    "in_sine": lambda x: 1 - cos((x * pi) / 2),
    "out_sine": lambda x: sin((x * pi) / 2),
    "in_out_sine": lambda x: -(cos(pi * x) - 1) / 2,
    "in_quad": lambda x: x * x,
    "out_quad": lambda x: 1 - (1 - x) ** 2,
    # ... many more
}
```

## Animation Levels

```python
AnimationLevel = Literal["none", "basic", "full"]
```

- `"none"`: Skip all animations, set final value immediately
- `"basic"`: Only essential animations
- `"full"`: All animations enabled

## Rust Implementation Strategy

### Traits

```rust
pub trait Animatable: Clone {
    fn blend(&self, destination: &Self, factor: f64) -> Self;
}

impl Animatable for Color {
    fn blend(&self, dest: &Self, factor: f64) -> Self {
        Color::new(
            lerp(self.r, dest.r, factor),
            lerp(self.g, dest.g, factor),
            lerp(self.b, dest.b, factor),
            lerp(self.a, dest.a, factor),
        )
    }
}
```

### Animator

```rust
pub struct Animator {
    animations: HashMap<AnimationKey, Box<dyn Animation>>,
    scheduled: HashMap<AnimationKey, TimerId>,
    fps: u32,
}

impl Animator {
    pub fn animate<T: Animatable>(
        &mut self,
        obj: &mut dyn HasAttribute<T>,
        attribute: &str,
        value: T,
        duration: Duration,
        easing: EasingFn,
    );

    pub fn tick(&mut self, time: Instant);
}
```

### Animation

```rust
pub trait Animation: Send {
    fn tick(&mut self, time: Instant, level: AnimationLevel) -> bool;
    fn stop(&mut self, complete: bool);
}

pub struct SimpleAnimation<T: Animatable> {
    target: Weak<RefCell<dyn HasAttribute<T>>>,
    attribute: String,
    start_time: Instant,
    duration: Duration,
    start_value: T,
    end_value: T,
    final_value: T,
    easing: EasingFn,
}
```

### Integration with Styles

```rust
impl RenderStyles {
    pub fn animate(&mut self, attribute: &str, value: StyleValue, transition: &Transition) {
        if self.is_animatable(attribute) {
            self.animator.animate(
                &mut self.base,
                attribute,
                value,
                transition.duration,
                transition.easing,
            );
        } else {
            self.base.set(attribute, value);
        }
    }
}
```
