# Specification: Basic Widgets

## Overview
This track implements the standard library of basic widgets matching Python Textual's API. **Important**: Existing textual-rs widgets (Button, Input, Label, Header, Footer) must be rebuilt to match Python Textual's properties, messages, and behavior.

## Goals
- **Rebuild existing widgets** to match Python Textual API exactly
- Implement display widgets (Static, Placeholder, Digits, Sparkline, Pretty)
- Implement form controls (Checkbox, RadioButton, RadioSet, Switch)
- Implement selection widgets (Select, SelectionList, OptionList)
- Implement progress widgets (ProgressBar, LoadingIndicator)
- Implement utility widgets (Rule, Link)

## ⚠️ REBUILD REQUIRED: Existing Widgets

The following widgets exist in textual-rs but are **not compatible** with Python Textual. They must be rebuilt from scratch to match the Python API:

### Button (REBUILD)
**Current textual-rs**: Basic button with simple click handling
**Python Textual requires**:
```python
class Button(Widget, can_focus=True):
    # Variants: "default", "primary", "success", "warning", "error"
    variant: reactive[ButtonVariant]
    label: reactive[TextType]
    disabled: reactive[bool]

    # Action support - can trigger action instead of message
    action: str | None

    # DEFAULT_CSS with all variant styles
    # Hover, focus, active, disabled states

    class Pressed(Message):
        button: Button
```

### Input (REBUILD)
**Current textual-rs**: Basic text input with cursor
**Python Textual requires**:
```python
class Input(ScrollView):  # NOTE: Extends ScrollView, not Widget!
    value: reactive[str]
    placeholder: reactive[str]
    password: reactive[bool]
    restrict: str | None  # Regex to restrict input
    type: InputType  # "integer", "number", "text"
    max_length: int | None

    # Selection support
    selection: Selection  # start, end indices
    cursor_position: int

    # Validation
    validators: list[Validator]
    validate_on: set[InputValidationOn]  # "blur", "changed", "submitted"
    valid: bool

    # Suggester (autocomplete)
    suggester: Suggester | None

    # Extensive key bindings for cursor movement, selection, word navigation
    BINDINGS: list[Binding]

    class Changed(Message):
        input: Input
        value: str
        validation_result: ValidationResult | None

    class Submitted(Message):
        input: Input
        value: str
        validation_result: ValidationResult | None
```

### Label (REBUILD)
**Current textual-rs**: Simple text display
**Python Textual requires**:
```python
class Label(Static):  # NOTE: Extends Static, not Widget!
    # Variants: "success", "error", "warning", "primary", "secondary", "accent"
    variant: LabelVariant | None

    # From Static base:
    content: VisualType
    expand: bool
    shrink: bool
    markup: bool  # Enable Rich markup parsing

    # DEFAULT_CSS with variant class styles
```

### Header (REBUILD)
**Current textual-rs**: Basic title bar
**Python Textual requires**:
```python
class Header(Widget):
    screen_title: str | None  # Can show screen's title
    screen_sub_title: str | None
    show_clock: bool
    time_format: str

    # Icon display
    icon: str

    # DEFAULT_CSS with styling
```

### Footer (REBUILD)
**Current textual-rs**: Basic key binding display
**Python Textual requires**:
```python
class Footer(Widget):
    # Automatically shows active bindings from focused widget chain
    # Supports compact mode
    compact: bool
    show_command_palette: bool

    # Key display formatting
    # Binding priority filtering

    # DEFAULT_CSS
```

## New Widgets to Implement

### Display Widgets

#### Static
```python
class Static(Widget):
    """Display static content (text, Rich renderables)."""
    content: VisualType
    expand: bool
    shrink: bool
    markup: bool

    def update(self, content: VisualType) -> None: ...
```

#### Placeholder
```python
class Placeholder(Widget):
    """Development placeholder showing widget info."""
    label: str | None
    variant: PlaceholderVariant  # "default", "size", "text"
```

#### Digits
```python
class Digits(Widget):
    """Display large 3x3 character digits."""
    value: str
```

#### Sparkline
```python
class Sparkline(Widget):
    """Inline mini chart."""
    data: Sequence[float]
    width: int | None
    min: float | None
    max: float | None
```

#### Pretty
```python
class Pretty(Widget):
    """Pretty-print Python objects."""
    object: Any
```

### Form Controls

#### Checkbox
```python
class Checkbox(Widget):
    value: reactive[bool]
    label: TextType

    BINDINGS = [Binding("space", "toggle")]

    class Changed(Message):
        checkbox: Checkbox
        value: bool
```

#### RadioButton
```python
class RadioButton(Widget):
    value: reactive[bool]
    label: TextType
```

#### RadioSet
```python
class RadioSet(Widget):
    """Container for RadioButtons with exclusive selection."""

    class Changed(Message):
        radio_set: RadioSet
        pressed: RadioButton
        index: int
```

#### Switch
```python
class Switch(Widget):
    value: reactive[bool]
    animate: bool  # Sliding animation

    class Changed(Message):
        switch: Switch
        value: bool
```

### Selection Widgets

#### Select
```python
class Select(Widget):
    """Dropdown selection widget."""
    value: reactive[SelectType]
    options: list[tuple[str, SelectType]]
    prompt: str
    allow_blank: bool

    class Changed(Message):
        select: Select
        value: SelectType
```

#### SelectionList
```python
class SelectionList(Widget):
    """Multi-select list."""
    selected: list[SelectionType]

    class SelectedChanged(Message): ...
```

#### OptionList
```python
class OptionList(Widget):
    """Vertical list of options."""
    highlighted: int | None

    class OptionHighlighted(Message): ...
    class OptionSelected(Message): ...
```

### Progress Widgets

#### ProgressBar
```python
class ProgressBar(Widget):
    progress: reactive[float]  # 0.0 to 1.0
    total: float
    show_bar: bool
    show_percentage: bool
    show_eta: bool

    def advance(self, amount: float) -> None: ...
```

#### LoadingIndicator
```python
class LoadingIndicator(Widget):
    """Animated loading spinner."""
    # Multiple animation frame styles
```

### Utility Widgets

#### Rule
```python
class Rule(Widget):
    orientation: Literal["horizontal", "vertical"]
    line_style: RuleLineStyle  # "solid", "dashed", "double", etc.
```

#### Link
```python
class Link(Widget):
    text: str
    url: str

    class Clicked(Message):
        link: Link
```

## Deliverables

### Phase 1: Analysis & Research
- Study each Python Textual widget implementation in detail
- Document exact API signatures, properties, messages
- Identify shared patterns (DEFAULT_CSS, reactive properties)
- Map validation and suggester systems

### Phase 2: Design & Planning
- Design Widget base compatibility layer
- Plan reactive property system integration
- Design message type hierarchy
- Plan DEFAULT_CSS approach in Rust

### Phase 3: Implementation
- **First**: Rebuild existing widgets (Static → Label → Button → Input → Header → Footer)
- **Then**: Implement new form controls
- **Then**: Implement selection widgets
- **Then**: Implement progress and utility widgets

### Phase 4: Testing & Verification
- API compatibility tests against Python Textual
- Unit tests for each widget
- Example app with all widgets

## Success Criteria
- [ ] All widgets match Python Textual API signatures
- [ ] Messages match Python Textual message types
- [ ] DEFAULT_CSS equivalent styling works
- [ ] Reactive properties trigger updates correctly
- [ ] Key bindings match Python Textual bindings

## Dependencies
- Core App Lifecycle (compose, mount, messages, reactive, bindings)
- Layout System (sizing, ScrollView for Input)
- TCSS Styling (DEFAULT_CSS, variants)

## Out of Scope (Tier 4: Advanced Widgets)
- DataTable (108KB - complex data grid)
- Tree / DirectoryTree (52KB - hierarchical views)
- TextArea (100KB - multi-line editor with syntax highlighting)
- Markdown / MarkdownViewer (53KB - document rendering)
- ListView (virtualized list)
