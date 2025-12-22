# Select, OptionList, and Link Widgets Analysis

## Link Widget

### Overview
Simple clickable link that opens a URL. Extends Static.

### Class Definition
```python
class Link(Static, can_focus=True):
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

### Reactive Properties
- `highlighted: reactive[int | None]` - Currently highlighted option index

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
class OptionHighlighted(Message):
    option_list: OptionList
    option: Option
    option_index: int

class OptionSelected(Message):
    option_list: OptionList
    option: Option
    option_index: int
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
Multi-select list where options can be individually toggled.

### Key Features
- Options have checkboxes
- Multiple selection allowed
- `selected` property returns list of selected values

### Messages
```python
class SelectedChanged(Message):
    selection_list: SelectionList
    # Contains newly selected/deselected items
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
