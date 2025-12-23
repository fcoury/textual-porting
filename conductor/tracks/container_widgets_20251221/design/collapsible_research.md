# Collapsible Widget Research

## Architecture Overview

```
Collapsible
├── CollapsibleTitle (focusable header bar)
│   ├── Symbol (▼ or ▶)
│   └── Label text
└── Contents (container for child widgets)
```

## CollapsibleTitle Class

Extends `Static` with focusability.

### Properties
- `collapsed: bool` - Synced with parent state
- `label: Content` - Header text (accepts ContentText)

### Keyboard Bindings
```
Enter → action_toggle_collapsible()
```

### DEFAULT_CSS
```css
CollapsibleTitle {
    width: auto;
    height: auto;
    padding: 0 1;

    &:hover {
        background: $surface-lighten-1;
        color: $text;
    }

    &:focus {
        background: $block-cursor-background;
        color: $block-cursor-foreground;
    }
}
```

### Rendering
Symbol changes based on collapsed state:
- Collapsed: `▶` (right-pointing triangle)
- Expanded: `▼` (down-pointing triangle)

## Collapsible Class

### Reactive Properties
- `collapsed: bool` - Default: True (starts collapsed)
- `title: str` - Header text for CollapsibleTitle

### Messages
| Message | Trigger |
|---------|---------|
| `Toggled` | Base class for toggle events (has `control` property) |
| `Expanded` | Content expands (collapsed → false) |
| `Collapsed` | Content collapses (collapsed → true) |

### Methods
- `expand()` - Set collapsed to False
- `collapse()` - Set collapsed to True
- `toggle()` - Flip collapsed state

### Compose Structure
```python
def compose(self):
    yield CollapsibleTitle(
        label=self._title,
        collapsed=self._collapsed,
        id="title"
    )
    yield Contents(*self._contents)
```

### DEFAULT_CSS
```css
Collapsible {
    width: 1fr;
    height: auto;

    /* Hide contents when collapsed */
    &.-collapsed > Contents {
        display: none;
    }

    /* Focus-within effect */
    &:focus-within {
        background: $surface-lighten-1;
    }
}

Contents {
    width: 1fr;
    height: auto;
    padding-left: 1;  /* Indent content */
}
```

## Animation Behavior

**Key Finding**: Python Textual Collapsible has NO explicit animation!

State changes are immediate:
1. `collapsed` reactive property changes
2. `-collapsed` CSS class is toggled
3. `display: none` hides/shows Contents instantly
4. `scroll_visible()` is called after refresh to ensure visibility

Animation could be added via:
- CSS transitions on height
- Animate max-height from 0 to auto
- Use our Timer system for frame-by-frame animation

## State Change Flow

```
1. User presses Enter on CollapsibleTitle
   ↓
2. action_toggle_collapsible() called
   ↓
3. Posts Collapsible.Toggled message
   ↓
4. Parent Collapsible handles message
   ↓
5. collapsed property flips
   ↓
6. _watch_collapsed() triggers:
   - Updates CollapsibleTitle.collapsed
   - Adds/removes .-collapsed class
   - Posts Expanded or Collapsed message
   - Calls scroll_visible() after refresh
```

## Implementation Plan for textual-rs

### Phase 1: CollapsibleTitle Widget
1. Create CollapsibleTitle struct
2. Implement label property with Content/ContentText
3. Implement collapsed property
4. Implement Enter key binding → action_toggle_collapsible
5. Implement rendering (symbol + label)
6. Add DEFAULT_CSS

### Phase 2: Contents Container
1. Simple container widget
2. Holds child widgets from Collapsible

### Phase 3: Collapsible Widget
1. Create Collapsible struct with collapsed, title properties
2. Implement compose with CollapsibleTitle + Contents
3. Implement expand(), collapse(), toggle() methods
4. Add Toggled, Expanded, Collapsed messages
5. Implement -collapsed class toggling
6. Add scroll_visible call after state change
7. Add DEFAULT_CSS

### Phase 4: Optional Animation
1. Use Timer system for smooth expand/collapse
2. Animate max-height from 0 to content height
3. Or use CSS transitions if supported

## Comparison with textual-rs Timer System

Our Timer system from Basic Widgets can power smooth animations:

```rust
// Example animation approach
impl Collapsible {
    fn start_expand_animation(&mut self) {
        self.animation_timer = Some(
            self.set_interval(
                Duration::from_millis(16), // ~60fps
                "expand_animation"
            )
        );
        self.target_height = self.contents.natural_height();
        self.current_height = 0;
    }

    fn on_timer_tick(&mut self, event: &TimerTick) {
        if event.timer_name == "expand_animation" {
            self.current_height += (self.target_height - self.current_height) / 4;
            if (self.target_height - self.current_height).abs() < 1 {
                self.current_height = self.target_height;
                self.animation_timer.take().map(|t| t.stop());
            }
            self.refresh();
        }
    }
}
```

Animation is optional - basic implementation can use instant toggle.
