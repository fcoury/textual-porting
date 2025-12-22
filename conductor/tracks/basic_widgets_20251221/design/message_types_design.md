# Message Types Design

## Overview

This document defines all message types for Basic Widgets, matching Python Textual's message API exactly. Messages bubble up the widget tree until handled.

## Message Base Trait

```rust
use std::any::Any;

/// Base trait for all Textual messages.
pub trait Message: Any + Send + Sync + std::fmt::Debug {
    /// Returns true if this message should stop bubbling.
    fn stop_propagation(&self) -> bool {
        false
    }

    /// Mark this message to stop bubbling.
    fn prevent_default(&mut self) {}
}
```

## Widget Message Catalog

### Button

```rust
/// Message sent when a Button is pressed.
#[derive(Debug, Clone)]
pub struct ButtonPressed {
    /// Reference to the button that was pressed.
    pub button: WidgetRef<Button>,
}

impl ButtonPressed {
    pub fn new(button: WidgetRef<Button>) -> Self {
        Self { button }
    }
}

impl Message for ButtonPressed {}
```

### ToggleButton (Checkbox/RadioButton base)

```rust
/// Message sent when a ToggleButton value changes.
#[derive(Debug, Clone)]
pub struct ToggleButtonChanged {
    /// Reference to the toggle button.
    pub toggle_button: WidgetRef<ToggleButton>,
    /// The new value.
    pub value: bool,
}

impl ToggleButtonChanged {
    /// Alias for toggle_button (common Textual pattern).
    pub fn control(&self) -> &WidgetRef<ToggleButton> {
        &self.toggle_button
    }
}

impl Message for ToggleButtonChanged {}
```

### Checkbox

```rust
/// Message sent when a Checkbox value changes.
#[derive(Debug, Clone)]
pub struct CheckboxChanged {
    inner: ToggleButtonChanged,
}

impl CheckboxChanged {
    pub fn checkbox(&self) -> WidgetRef<Checkbox> {
        // Safe cast since Checkbox wraps ToggleButton
        self.inner.toggle_button.cast()
    }

    pub fn value(&self) -> bool {
        self.inner.value
    }
}

impl Message for CheckboxChanged {}
```

### RadioButton

```rust
/// Message sent when a RadioButton value changes.
#[derive(Debug, Clone)]
pub struct RadioButtonChanged {
    inner: ToggleButtonChanged,
}

impl RadioButtonChanged {
    pub fn radio_button(&self) -> WidgetRef<RadioButton> {
        self.inner.toggle_button.cast()
    }

    pub fn value(&self) -> bool {
        self.inner.value
    }
}

impl Message for RadioButtonChanged {}
```

### RadioSet

```rust
/// Message sent when the selected RadioButton in a RadioSet changes.
#[derive(Debug, Clone)]
pub struct RadioSetChanged {
    /// Reference to the RadioSet.
    pub radio_set: WidgetRef<RadioSet>,
    /// The index of the newly selected button.
    pub index: usize,
    /// Reference to the pressed RadioButton.
    pub pressed: WidgetRef<RadioButton>,
}

impl Message for RadioSetChanged {}
```

### Switch

```rust
/// Message sent when a Switch value changes.
#[derive(Debug, Clone)]
pub struct SwitchChanged {
    /// Reference to the switch.
    pub switch: WidgetRef<Switch>,
    /// The new value.
    pub value: bool,
}

impl Message for SwitchChanged {}
```

### Input

```rust
/// Message sent when Input value changes.
#[derive(Debug, Clone)]
pub struct InputChanged {
    /// Reference to the input.
    pub input: WidgetRef<Input>,
    /// The new value.
    pub value: String,
    /// Validation result (if validation was triggered).
    pub validation_result: Option<ValidationResult>,
}

impl InputChanged {
    /// Alias for input (common Textual pattern).
    pub fn control(&self) -> &WidgetRef<Input> {
        &self.input
    }
}

impl Message for InputChanged {}

/// Message sent when Input is submitted (Enter pressed).
#[derive(Debug, Clone)]
pub struct InputSubmitted {
    /// Reference to the input.
    pub input: WidgetRef<Input>,
    /// The submitted value.
    pub value: String,
    /// Validation result (if validation was triggered).
    pub validation_result: Option<ValidationResult>,
}

impl Message for InputSubmitted {}

/// Message sent when Input loses focus.
#[derive(Debug, Clone)]
pub struct InputBlurred {
    /// Reference to the input.
    pub input: WidgetRef<Input>,
    /// The current value.
    pub value: String,
    /// Validation result (if validation was triggered).
    pub validation_result: Option<ValidationResult>,
}

impl Message for InputBlurred {}
```

### OptionList

```rust
/// Base message for OptionList events.
#[derive(Debug, Clone)]
pub struct OptionMessage {
    /// Reference to the option list.
    pub option_list: WidgetRef<OptionList>,
    /// The option involved.
    pub option: Option<OptionData>,
    /// The option's ID (if set).
    pub option_id: Option<String>,
    /// The option's index in the list.
    pub option_index: usize,
}

/// Message sent when an option is highlighted (cursor moves).
#[derive(Debug, Clone)]
pub struct OptionHighlighted(pub OptionMessage);

impl std::ops::Deref for OptionHighlighted {
    type Target = OptionMessage;
    fn deref(&self) -> &Self::Target { &self.0 }
}

impl Message for OptionHighlighted {}

/// Message sent when an option is selected (Enter pressed).
#[derive(Debug, Clone)]
pub struct OptionSelected(pub OptionMessage);

impl std::ops::Deref for OptionSelected {
    type Target = OptionMessage;
    fn deref(&self) -> &Self::Target { &self.0 }
}

impl Message for OptionSelected {}
```

### Select

```rust
/// Message sent when Select value changes.
/// NOT a dataclass - uses explicit constructor.
#[derive(Debug, Clone)]
pub struct SelectChanged<T: Clone> {
    /// Reference to the select widget.
    pub select: WidgetRef<Select<T>>,
    /// The new value.
    pub value: SelectValue<T>,
}

impl<T: Clone> SelectChanged<T> {
    /// Alias for select (common Textual pattern).
    pub fn control(&self) -> &WidgetRef<Select<T>> {
        &self.select
    }
}

impl<T: Clone + Send + Sync + std::fmt::Debug + 'static> Message for SelectChanged<T> {}

/// Represents the selected value or blank state.
#[derive(Debug, Clone, PartialEq)]
pub enum SelectValue<T> {
    Blank,
    Selected(T),
}

impl<T> SelectValue<T> {
    pub fn is_blank(&self) -> bool {
        matches!(self, SelectValue::Blank)
    }
}
```

### SelectionList

```rust
/// Base for SelectionList messages.
#[derive(Debug, Clone)]
pub struct SelectionMessage<T: Clone> {
    /// Reference to the selection list.
    pub selection_list: WidgetRef<SelectionList<T>>,
    /// The selection involved.
    pub selection: Option<Selection<T>>,
    /// Index of the selection.
    pub selection_index: usize,
}

/// Message sent when a selection is highlighted (cursor moves).
#[derive(Debug, Clone)]
pub struct SelectionHighlighted<T: Clone>(pub SelectionMessage<T>);

impl<T: Clone + Send + Sync + std::fmt::Debug + 'static> Message for SelectionHighlighted<T> {}

/// Message sent when a selection is EXPLICITLY toggled.
///
/// Only sent for:
/// - User interaction (space key, click)
/// - toggle() or toggle_all() method calls
///
/// NOT sent for programmatic select()/deselect() calls.
#[derive(Debug, Clone)]
pub struct SelectionToggled<T: Clone>(pub SelectionMessage<T>);

impl<T: Clone + Send + Sync + std::fmt::Debug + 'static> Message for SelectionToggled<T> {}

/// Message sent when the collection of selected values changes.
///
/// Sent for ALL changes (user interaction AND programmatic API calls).
/// For bulk operations (select_all, deselect_all), only ONE message is sent.
#[derive(Debug, Clone)]
pub struct SelectedChanged<T: Clone> {
    /// Reference to the selection list.
    pub selection_list: WidgetRef<SelectionList<T>>,
}

impl<T: Clone + Send + Sync + std::fmt::Debug + 'static> Message for SelectedChanged<T> {}
```

### Link

```rust
/// Message sent when a Link is clicked/activated.
#[derive(Debug, Clone)]
pub struct LinkClicked {
    /// Reference to the link.
    pub link: WidgetRef<Link>,
    /// The URL to open.
    pub url: String,
}

impl Message for LinkClicked {}
```

## Widget Reference Type

```rust
/// A reference to a widget in the tree.
///
/// This is a lightweight handle that can be used to look up
/// the actual widget data from the registry.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct WidgetRef<T> {
    id: WidgetId,
    _phantom: std::marker::PhantomData<T>,
}

impl<T: Widget> WidgetRef<T> {
    pub fn new(id: WidgetId) -> Self {
        Self {
            id,
            _phantom: std::marker::PhantomData,
        }
    }

    pub fn id(&self) -> WidgetId {
        self.id
    }

    /// Cast to a different widget type.
    /// Unsafe: caller must ensure the underlying widget is of type U.
    pub fn cast<U: Widget>(&self) -> WidgetRef<U> {
        WidgetRef {
            id: self.id,
            _phantom: std::marker::PhantomData,
        }
    }
}
```

## Message Posting

```rust
impl dyn Widget {
    /// Post a message to bubble up the tree.
    fn post_message<M: Message>(&self, msg: M) {
        // Implementation accesses the message queue
        // Messages are processed in the event loop
    }
}
```

## Message Handling

```rust
impl Widget for MyWidget {
    fn on_message(&mut self, msg: &dyn Any) -> bool {
        // Handle specific message types
        if let Some(pressed) = msg.downcast_ref::<ButtonPressed>() {
            self.handle_button_press(pressed);
            return true; // Stop bubbling
        }

        if let Some(changed) = msg.downcast_ref::<InputChanged>() {
            self.handle_input_change(changed);
            return true;
        }

        false // Continue bubbling
    }
}
```

## Python Textual Compatibility Notes

### control Property

Many Python Textual messages have a `control` property as an alias for the widget reference. This is preserved in Rust:

```rust
impl InputChanged {
    pub fn control(&self) -> &WidgetRef<Input> {
        &self.input
    }
}
```

### Message Classes vs Structs

Python uses `@dataclass` for some messages but not others. In Rust, we use plain structs for all messages, implementing `Debug` and `Clone` manually when needed.

### Generic Messages

SelectChanged and SelectionList messages are generic over the value type `T`. This matches Python's `Generic[SelectType]` pattern.

## Testing Strategy

1. **Message Creation**: All messages can be constructed correctly
2. **Bubbling**: Messages bubble through widget tree
3. **Handling**: Messages are received and handled by listeners
4. **Stop Propagation**: Handled messages stop bubbling
5. **control Alias**: Alias methods work correctly
