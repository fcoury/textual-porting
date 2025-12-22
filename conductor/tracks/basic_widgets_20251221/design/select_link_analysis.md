# Select, OptionList, and Link Widgets Analysis

## Link Widget

### Overview
Simple clickable link that opens a URL. Extends Static with `markup=False` (link text is NOT parsed as markup).

### Class Definition
```python
class Link(Static, can_focus=True):
    def __init__(self, text, *, url=None, ...):
        super().__init__(text, markup=False)  # Disables markup parsing for link text
```

### Reactive Properties
- `text: reactive[str] = ""` - Display text
- `url: reactive[str] = ""` - URL to open (defaults to text if not provided)

### Constructor
```python
def __init__(
    self,
    text: str,
    *,
    url: str | None = None,  # If None, uses text as URL
    tooltip: str | None = None,
    name, id, classes, disabled
):
```

### Bindings
```python
BINDINGS = [Binding("enter", "open_link", "Open link")]
```

### Methods
```python
def action_open_link(self) -> None:
    if self.url:
        self.app.open_url(self.url)  # Platform URL opener
```

### DEFAULT_CSS
```css
Link {
    width: auto;
    height: auto;
    min-height: 1;
    color: $text-accent;
    text-style: underline;
    &:hover { color: $accent; }
    &:focus { text-style: bold reverse; }
}
```

---

## OptionList Widget

### Overview
Navigable list of options extending ScrollView. Used as base for Select dropdown.

### Class Definition
```python
class OptionList(ScrollView, can_focus=True):
```

### Constructor
```python
def __init__(
    self,
    *content: OptionListContent,
    name: str | None = None,
    id: str | None = None,
    classes: str | None = None,
    disabled: bool = False,
    tooltip: RenderableType | None = None,
    markup: bool = True,      # Enable markup parsing in option prompts
    compact: bool = False,    # Compact display mode (toggles -compact class)
):
```

### Reactive Properties
- `highlighted: reactive[int | None]` - Currently highlighted option index
- `compact: reactive[bool] = False` - Toggles `-compact` CSS class

### OptionListContent Type
```python
OptionListContent: TypeAlias = "Option | VisualType | None"
# - Option: A selectable option
# - VisualType: Any Rich renderable (auto-wrapped as Option)
# - None: Creates a separator line (divider)
```

### Option Class
```python
class Option:
    def __init__(
        self,
        prompt: VisualType,  # Text or renderable
        id: str | None = None,
        disabled: bool = False
    ):
        self._prompt = prompt
        self._id = id
        self.disabled = disabled
        self._divider = False  # Is separator line
```

### Key Bindings
```python
BINDINGS = [
    Binding("down", "cursor_down"),
    Binding("up", "cursor_up"),
    Binding("enter", "select"),
    Binding("home", "first"),
    Binding("end", "last"),
    Binding("pageup", "page_up"),
    Binding("pagedown", "page_down"),
]
```

### Messages
```python
class OptionMessage(Message):
    """Base class for OptionList messages."""
    option_list: OptionList
    option: Option
    option_id: str | None    # The option's ID (if set)
    option_index: int        # The option's index in the list

class OptionHighlighted(OptionMessage):
    """Sent when an option is highlighted (cursor moves)."""

class OptionSelected(OptionMessage):
    """Sent when an option is selected (Enter pressed)."""
```

### Key Methods
```python
def add_option(self, item: OptionListContent = None) -> Self:
    """Add option. If item is None, adds a separator line."""

def add_options(self, items: Iterable[OptionListContent]) -> Self:
    """Add multiple options. None values become separators."""

def get_option(self, option_id: str) -> Option:
    """Get option by ID. Raises OptionDoesNotExist if not found."""

def get_option_at_index(self, index: int) -> Option:
    """Get option at index. Raises OptionDoesNotExist if invalid."""

def remove_option(self, option_id: str) -> Self:
    """Remove option by ID. Raises OptionDoesNotExist if not found."""

def remove_option_at_index(self, index: int) -> Self:
    """Remove option at index. Raises OptionDoesNotExist if invalid."""
```

### Errors
```python
class DuplicateID(Exception):
    """Raised when adding an option with an ID that already exists."""

class OptionDoesNotExist(Exception):
    """Raised when referencing an option by ID or index that doesn't exist."""
```

### Component Classes
- `option-list--option` - Individual option styling
- `option-list--option-highlighted` - Highlighted option
- `option-list--option-disabled` - Disabled option
- `option-list--separator` - Divider line

---

## Select Widget

### Overview
Dropdown selection widget combining a button with OptionList overlay.

### Class Definition
```python
SelectType = TypeVar("SelectType")

class Select(Generic[SelectType], Widget):
```

### Special Values
```python
class NoSelection:
    """Flags unselected state."""

BLANK = NoSelection()  # Select.BLANK
```

### Reactive Properties
- `value: reactive[SelectType | NoSelection]` - Selected value or BLANK
- `expanded: reactive[bool] = False` - Dropdown open state
- `prompt: reactive[str] = "Select"` - Default prompt text

### Constructor
```python
def __init__(
    self,
    options: Iterable[tuple[str, SelectType]],  # (display_text, value) tuples
    *,
    prompt: str = "Select",
    allow_blank: bool = True,
    value: SelectType | NoSelection = BLANK,
    name, id, classes, disabled, tooltip
):
```

### Messages
```python
@dataclass
class Changed(Message, Generic[SelectType]):
    select: Select[SelectType]
    value: SelectType | NoSelection
```

### Internal Components
- **SelectCurrent** - Shows current selection (button)
- **SelectOverlay** - Extends OptionList with search functionality

### Type-to-Search
SelectOverlay supports typing to jump to matching options:
- Timer resets search after 0.7s delay
- Substring matching favors earlier matches

### Errors
- `InvalidSelectValueError` - Unknown option value
- `EmptySelectError` - No options and allow_blank=False

---

## SelectionList Widget

### Overview
Multi-select list where options can be individually toggled. Extends OptionList with checkbox indicators.

### Selection Type
```python
class Selection(Generic[SelectionType], Option):
    def __init__(
        self,
        prompt: ContentText,
        value: SelectionType,
        initial_state: bool = False,  # Initially selected?
        id: str | None = None,
        disabled: bool = False
    ):
```

### Component Classes
- `selection-list--button` - Default button style
- `selection-list--button-selected` - Selected button
- `selection-list--button-highlighted` - Highlighted button
- `selection-list--button-selected-highlighted` - Both selected and highlighted

### Bindings
```python
BINDINGS = [Binding("space", "select", "Toggle option", show=False)]
```

### Messages (ALL THREE)

#### SelectionHighlighted
```python
class SelectionHighlighted(SelectionMessage[MessageSelectionType]):
    """Message sent when a selection is highlighted (cursor moves)."""
    selection_list: SelectionList
    selection: Selection
    selection_index: int
```

#### SelectionToggled
```python
class SelectionToggled(SelectionMessage[MessageSelectionType]):
    """Message sent when a selection is EXPLICITLY toggled.

    Only sent for:
    - User interaction (space key, click)
    - toggle() or toggle_all() method calls

    NOT sent for programmatic select()/deselect() calls.
    A message is sent for EACH option toggled (even in bulk toggle_all).
    """
    selection_list: SelectionList
    selection: Selection
    selection_index: int
```

#### SelectedChanged
```python
@dataclass
class SelectedChanged(Generic[MessageSelectionType], Message):
    """Message sent when the collection of selected values changes.

    Sent for ALL changes (user interaction AND programmatic API calls).
    For bulk operations (select_all, deselect_all), only ONE message is sent.
    """
    selection_list: SelectionList
```

---

## Rust Implementation Considerations

### Link
```rust
pub struct Link {
    text: String,
    url: String,
}

impl Link {
    pub fn open_link(&self, app: &App) {
        if !self.url.is_empty() {
            app.open_url(&self.url);  // Platform-specific
        }
    }
}
```

### Option
```rust
pub struct Option {
    prompt: ContentType,
    id: Option<String>,
    disabled: bool,
    is_divider: bool,
}
```

### OptionList
```rust
pub struct OptionList {
    options: Vec<Option>,
    highlighted: Option<usize>,
}

// Messages
pub struct OptionHighlighted { index: usize }
pub struct OptionSelected { index: usize }
```

### Select
```rust
pub enum SelectValue<T> {
    Blank,
    Selected(T),
}

pub struct Select<T> {
    options: Vec<(String, T)>,  // (display, value)
    value: SelectValue<T>,
    prompt: String,
    allow_blank: bool,
    expanded: bool,
}
```

### Key Challenges
1. Generic type parameter for Select<T>
2. OptionList virtualization for large lists
3. SelectOverlay type-to-search with timer
4. Platform URL opening for Link
5. SelectionList multi-select state management
