# Reactive Property Integration Design

## Overview

This document defines how reactive properties integrate across all Basic Widgets, ensuring consistent behavior with Python Textual's `reactive` and `var` property systems.

## Property Types

### Python Textual Property Types

| Type | Behavior | Watcher | CSS Class Toggle |
|------|----------|---------|------------------|
| `reactive` | Triggers watcher on change | Yes (always) | Optional |
| `var` | Triggers watcher on change | Yes (always) | No |
| `var(init=False)` | Skips watcher on initialization | On change only | No |

### Rust Equivalents

```rust
/// Standard reactive property - triggers watcher on every change.
/// Use for properties that need UI updates.
#[reactive]
content: ContentType,

/// Reactive with layout flag - triggers layout recalculation.
#[reactive(layout = true)]
expanded: bool,

/// Reactive with CSS class toggle.
#[reactive(class = "-on")]
value: bool,

/// Reactive with init=false - skips watcher during construction.
#[reactive(init = false)]
value: SelectValue<T>,
```

## Widget Reactive Properties Catalog

### Static
```rust
// All trigger repaint only
content: ContentType,      // reactive (layout=true when changed via update())
expand: bool,              // reactive
shrink: bool,              // reactive
markup: bool,              // reactive
```

### Label (extends Static)
```rust
variant: Option<LabelVariant>,  // reactive (applies CSS class)
```

### Button
```rust
label: String,                  // reactive → watch_label (updates render)
variant: ButtonVariant,         // reactive → watch_variant (updates CSS class)
disabled: bool,                 // reactive (toggles -disabled class)
compact: bool,                  // reactive (toggles -textual-compact class)
flat: bool,                     // reactive (toggles -flat class)
```

### Input
```rust
value: String,                  // reactive
selection: Selection,           // reactive (cursor position and selection range)
placeholder: String,            // reactive
password: bool,                 // reactive
cursor_blink: bool,            // reactive
compact: bool,                  // reactive (toggles -textual-compact class)
// Internal
_cursor_visible: bool,          // reactive (internal, for blink state)
_suggestion: String,            // reactive (internal, for autocomplete)
```

### ToggleButton (base for Checkbox/RadioButton)
```rust
value: bool,                    // reactive → watch_value toggles -on, posts Changed
compact: bool,                  // reactive (toggles -textual-compact class)
```

**Critical Behavior:**
```rust
impl ToggleButton {
    fn watch_value(&mut self, value: bool) {
        self.set_class(value, "-on");
        self.post_message(Changed { toggle_button: self, value });
    }
}
```

### Switch
```rust
value: bool,                    // reactive → watch_value (animates, posts Changed)
_slider_position: f32,          // reactive → watch__slider_position toggles -on
```

**Initialization Behavior:**
```rust
impl Switch {
    pub fn new(value: bool, animate: bool) -> Self {
        let mut switch = Self::default();
        if value {
            switch._slider_position = 1.0;  // Triggers -on class
            switch.set_reactive_silent(|s| s.value = value);  // No watch_value
        }
        switch
    }
}
```

### OptionList
```rust
highlighted: Option<usize>,     // reactive
compact: bool,                  // reactive (toggles -textual-compact class)
```

### Select
```rust
// These are var, NOT reactive
value: SelectValue<T>,          // var(init=false)
expanded: bool,                 // var(init=false)
prompt: String,                 // var

// This IS reactive
compact: bool,                  // reactive (toggles -textual-compact class)
```

### Header
```rust
tall: bool,                     // reactive (toggles -tall class)
icon: String,                   // reactive
time_format: String,            // reactive
```

### Footer
```rust
compact: bool,                  // reactive (toggles -compact class)
show_command_palette: bool,     // reactive
_bindings_ready: bool,          // reactive (internal)
```

### ProgressBar
```rust
progress: f64,                  // reactive
total: Option<f64>,             // reactive
percentage: Option<f64>,        // reactive (computed)
```

### Rule
```rust
orientation: RuleOrientation,   // reactive (toggles -horizontal/-vertical class)
line_style: LineStyle,          // reactive
```

### Link
```rust
text: String,                   // reactive
url: String,                    // reactive
```

## Implementation Patterns

### Pattern 1: Standard Reactive (repaint only)

```rust
impl MyWidget {
    pub fn set_content(&mut self, content: impl Into<ContentType>) {
        let old = self.content.clone();
        self.content = content.into();
        if old != self.content {
            self.watch_content(old, &self.content);
            self.refresh_repaint();
        }
    }
}
```

### Pattern 2: Reactive with Layout

```rust
impl Static {
    pub fn update(&mut self, content: impl Into<ContentType>, layout: bool) {
        let old = self.content.clone();
        self.content = content.into();
        if old != self.content {
            self.visual_cache = None;
            if layout {
                self.refresh_layout();
            } else {
                self.refresh_repaint();
            }
        }
    }
}
```

### Pattern 3: Reactive with CSS Class Toggle

```rust
impl ToggleButton {
    pub fn set_value(&mut self, value: bool) {
        let old = self.value;
        self.value = value;
        if old != value {
            self.watch_value(value);
        }
    }

    fn watch_value(&mut self, value: bool) {
        self.set_class(value, "-on");
        self.post_message(Changed::new(self, value));
        self.refresh_repaint();
    }
}
```

### Pattern 4: var with init=false (Skip Initial Watcher)

```rust
impl Select<T> {
    pub fn new(options: Vec<(String, T)>, value: SelectValue<T>) -> Self {
        let mut select = Self {
            options,
            value: SelectValue::Blank,
            // ...
        };
        // Use set_reactive_silent to skip watcher on init
        select.value = value;  // Direct assignment, no watcher
        select
    }

    pub fn set_value(&mut self, value: SelectValue<T>) {
        let old = self.value.clone();
        self.value = value;
        if old != self.value {
            self.watch_value(old, &self.value);
        }
    }
}
```

### Pattern 5: Preventing Message Emission

```rust
impl ToggleButton {
    pub fn new(value: bool) -> Self {
        let mut button = Self::default();
        // Use prevent pattern to suppress Changed on init
        button.value = value;
        if value {
            button.add_class("-on");  // Set class directly, no message
        }
        button
    }
}
```

## CSS Class Toggles Summary

| Widget | Property | Class Toggled |
|--------|----------|---------------|
| Button | variant | `-primary` / `-success` / `-warning` / `-error` |
| Button | compact | `-textual-compact` |
| Button | flat | `-flat` |
| ToggleButton | value | `-on` |
| Switch | _slider_position (==1.0) | `-on` |
| Header | tall | `-tall` |
| Footer | compact | `-compact` |
| Input | compact | `-textual-compact` |
| OptionList | compact | `-textual-compact` |
| Select | compact | `-textual-compact` |
| Rule | orientation | `-horizontal` / `-vertical` |
| Input | validity | `-invalid` |

## Message Emission Summary

| Widget | Property | Message Emitted |
|--------|----------|-----------------|
| ToggleButton | value | Changed |
| Switch | value | Changed |
| Input | value | Changed (with validation) |
| Input | (submit) | Submitted |
| Input | (blur) | Blurred |
| Select | value | Changed |
| OptionList | highlighted | OptionHighlighted |
| OptionList | (select) | OptionSelected |
| SelectionList | (toggle) | SelectionToggled |
| SelectionList | selected | SelectedChanged |

## Watcher Naming Convention

Watchers follow Python Textual's naming convention:
- Public property `foo` → watcher method `watch_foo`
- Private property `_foo` → watcher method `watch__foo` (double underscore)

```rust
impl Switch {
    fn watch_value(&mut self, value: bool) { /* ... */ }
    fn watch__slider_position(&mut self, position: f32) { /* ... */ }
}
```

## Testing Requirements

1. **Watcher Invocation**: Verify watchers are called on property changes
2. **Init Suppression**: Verify `init=false` properties don't fire watchers on construction
3. **CSS Class Toggles**: Verify classes are added/removed correctly
4. **Message Emission**: Verify correct messages are posted
5. **Prevent Pattern**: Verify message suppression works during init
