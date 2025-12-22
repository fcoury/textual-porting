# Basic Widgets Technical Design Document

## Executive Summary

This document provides a unified technical design for implementing Basic Widgets in textual-rs. It consolidates the detailed component designs into an implementation roadmap.

## Design Documents Index

| Document | Purpose |
|----------|---------|
| [static_design.md](static_design.md) | Static base widget with markup support |
| [reactive_integration_design.md](reactive_integration_design.md) | Reactive property patterns for all widgets |
| [default_css_design.md](default_css_design.md) | DEFAULT_CSS pattern and catalog |
| [message_types_design.md](message_types_design.md) | All widget message types |
| [validator_suggester_design.md](validator_suggester_design.md) | Input validation and suggestion systems |

## Analysis Documents Index

| Document | Widgets Analyzed |
|----------|------------------|
| [static_label_analysis.md](static_label_analysis.md) | Static, Label |
| [button_analysis.md](button_analysis.md) | Button |
| [input_analysis.md](input_analysis.md) | Input |
| [header_footer_analysis.md](header_footer_analysis.md) | Header, Footer |
| [form_progress_rule_analysis.md](form_progress_rule_analysis.md) | ToggleButton, Checkbox, RadioButton, Switch, ProgressBar, LoadingIndicator, Rule |
| [select_link_analysis.md](select_link_analysis.md) | Link, OptionList, Select, SelectionList |

## Widget Implementation Priority

### Tier 1: Foundation (Implement First)
1. **Static** - Base for Label, Link
2. **Label** - Simple text display with variants

### Tier 2: Core Interactivity
3. **Button** - Primary user action widget
4. **Input** - Text input with validation

### Tier 3: Form Controls
5. **ToggleButton** - Base for Checkbox/RadioButton
6. **Checkbox** - Boolean toggle
7. **RadioButton** - Exclusive selection item
8. **RadioSet** - Radio button container
9. **Switch** - Animated boolean toggle

### Tier 4: Selection Widgets
10. **OptionList** - Navigable option list
11. **Select** - Dropdown selection
12. **SelectionList** - Multi-select list

### Tier 5: Application Chrome
13. **Header** - App title and clock
14. **Footer** - Keybinding display

### Tier 6: Progress & Utility
15. **ProgressBar** - Progress display
16. **LoadingIndicator** - Animated loading state
17. **Rule** - Separator line
18. **Link** - Clickable URL

## Architecture Overview

### Widget Trait Hierarchy

```
Widget (trait)
├── Static
│   ├── Label
│   └── Link (markup=false)
├── ToggleButton
│   ├── Checkbox
│   └── RadioButton
├── ScrollView (existing)
│   ├── Input
│   └── OptionList
│       ├── Select (overlay)
│       └── SelectionList
├── Container (existing)
│   ├── Header
│   ├── Footer
│   └── RadioSet
└── Specialized
    ├── Button
    ├── Switch
    ├── ProgressBar
    ├── LoadingIndicator
    └── Rule
```

### Core Systems Integration

```
┌─────────────────────────────────────────────────────────────┐
│                        App                                   │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │   Screen    │  │StyleSystem  │  │    MessageQueue     │  │
│  │             │  │             │  │                     │  │
│  │ - widgets   │  │ - defaults  │  │ - pending messages  │  │
│  │ - focused   │  │ - user css  │  │ - bubble/capture    │  │
│  │ - bindings  │  │ - computed  │  │                     │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│                    WidgetRegistry                            │
│  ┌─────────────────────────────────────────────────────────┐│
│  │ SlotMap<WidgetId, Box<dyn Widget>>                      ││
│  │ SlotMap<WidgetId, TreeNode>  (parent/children)          ││
│  │ HashMap<WidgetId, ComputedStyle>                        ││
│  │ HashMap<WidgetId, WidgetRefreshState>                   ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

## Key Patterns

### 1. Composition over Inheritance

Since Rust lacks inheritance, widgets use composition:

```rust
pub struct Label {
    inner: Static,
    variant: Option<LabelVariant>,
}

impl Label {
    pub fn new(content: impl Into<ContentType>) -> Self {
        Self {
            inner: Static::new(content),
            variant: None,
        }
    }
}
```

### 2. Reactive Properties with Watchers

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
    }
}
```

### 3. Message Bubbling

```rust
impl Widget for MyWidget {
    fn on_message(&mut self, msg: &dyn Any) -> bool {
        if let Some(pressed) = msg.downcast_ref::<ButtonPressed>() {
            self.handle_button(pressed);
            return true;  // Stop bubbling
        }
        false  // Continue bubbling
    }
}
```

### 4. DEFAULT_CSS Registration

```rust
impl DefaultCss for Button {
    fn default_css() -> &'static str {
        r#"
        Button {
            background: $primary;
            height: 3;
        }
        "#
    }
}
```

## Implementation Guidelines

### File Organization

```
textual-rs/src/
├── widgets/
│   ├── mod.rs              # Widget exports
│   ├── static_widget.rs    # Static base widget
│   ├── label.rs            # Label (extends Static)
│   ├── button.rs           # Button widget
│   ├── input.rs            # Input widget
│   ├── toggle.rs           # ToggleButton base
│   ├── checkbox.rs         # Checkbox
│   ├── radio.rs            # RadioButton + RadioSet
│   ├── switch.rs           # Switch
│   ├── option_list.rs      # OptionList
│   ├── select.rs           # Select
│   ├── selection_list.rs   # SelectionList
│   ├── header.rs           # Header
│   ├── footer.rs           # Footer
│   ├── progress.rs         # ProgressBar + LoadingIndicator
│   ├── rule.rs             # Rule
│   └── link.rs             # Link
├── validation/
│   ├── mod.rs
│   ├── result.rs           # ValidationResult, Failure
│   ├── validator.rs        # Validator trait
│   └── validators/         # Built-in validators
│       ├── integer.rs
│       ├── number.rs
│       ├── length.rs
│       ├── regex.rs
│       └── url.rs
└── suggestion/
    ├── mod.rs
    ├── suggester.rs        # Suggester trait
    └── suggest_from_list.rs
```

### Testing Requirements

Each widget requires:
1. **Unit tests** - Constructor, property setters, validation
2. **Message tests** - Correct messages posted on actions
3. **Render tests** - Output matches expected
4. **Integration tests** - Works in widget tree

### Code Quality Standards

- All public APIs documented with rustdoc
- No `unwrap()` in library code (use `?` or explicit error handling)
- All reactive properties have watchers when required
- DEFAULT_CSS matches Python Textual exactly
- Messages match Python Textual API exactly

## Open Questions & Decisions

### Resolved

1. **Widget composition pattern** - Use composition with inner widget
2. **Message type safety** - Use `dyn Any` with downcasting
3. **Reactive implementation** - Use dedicated macro or manual setters
4. **CSS parsing** - Existing TCSS parser handles this

### Pending (for implementation phase)

1. **Clipboard integration** - Platform-specific implementation for Input
2. **URL opening** - Platform-specific for Link
3. **Animation timing** - Timer integration for Switch, LoadingIndicator
4. **Async suggestions** - Runtime integration for Suggester

## Success Criteria

Phase 2 is complete when:
- [ ] All design documents reviewed and approved
- [ ] No blocking architectural questions remain
- [ ] Implementation priority is agreed upon
- [ ] File organization is finalized

Phase 3 (Implementation) success criteria:
- [ ] All widgets compile and pass tests
- [ ] API matches Python Textual exactly
- [ ] DEFAULT_CSS matches Python Textual
- [ ] Example app demonstrates all widgets
