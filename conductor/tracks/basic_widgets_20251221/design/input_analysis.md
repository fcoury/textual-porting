# Input Widget Analysis

## Overview
Input is a full-featured text input widget that **extends ScrollView** (not Widget directly). It supports selection, validation, suggestions, and extensive keyboard bindings.

## Class Definition
```python
class Input(ScrollView):  # NOTE: Extends ScrollView!
```

## Reactive Properties
- `value: reactive[str] = ""` - Current text value
- `selection: reactive[Selection]` - Current selection range
- `placeholder: reactive[str] = ""` - Placeholder text
- `password: reactive[bool] = False` - Mask input as password
- `cursor_blink: reactive[bool] = True` - Enable cursor blinking
- `compact: reactive[bool] = False` - Compact mode (toggle `-textual-compact`)
- `_cursor_visible: reactive[bool]` - Internal cursor visibility state
- `_suggestion: reactive[str] = ""` - Current autocomplete suggestion

## Non-Reactive Properties (var)
- `restrict: str | None = None` - Regex pattern to restrict input
- `type: InputType` - Input type ("text", "integer", "number")
- `max_length: int | None = None` - Maximum character length
- `valid_empty: bool = False` - Allow empty values to pass validation

## Selection Type
```python
class Selection(NamedTuple):
    start: int
    end: int

    @classmethod
    def cursor(cls, cursor_position: int) -> Selection:
        """Create a selection from a cursor position."""

    @property
    def is_empty(self) -> bool:
        """Return True if the selection is empty."""
```

## InputType
```python
InputType = Literal["integer", "number", "text"]

_RESTRICT_TYPES = {
    "integer": r"[-+]?(?:\d*|\d+_)*",
    "number": r"[-+]?(?:\d*|\d+_)*\.?(?:\d*|\d+_)*(?:\d[eE]?[-+]?(?:\d*|\d+_)*)?",
    "text": None,
}
```

## Constructor Parameters
```python
def __init__(
    self,
    value: str | None = None,
    placeholder: str = "",
    highlighter: Highlighter | None = None,
    password: bool = False,
    *,
    restrict: str | None = None,
    type: InputType = "text",
    max_length: int = 0,
    suggester: Suggester | None = None,
    validators: Validator | Iterable[Validator] | None = None,
    validate_on: Iterable[InputValidationOn] | None = None,
    valid_empty: bool = False,
    select_on_focus: bool = True,
    name: str | None = None,
    id: str | None = None,
    classes: str | None = None,
    disabled: bool = False,
    tooltip: RenderableType | None = None,
    compact: bool = False,
):
```

## Messages
```python
@dataclass
class Changed(Message):
    input: Input
    value: str
    validation_result: ValidationResult | None = None

@dataclass
class Submitted(Message):
    input: Input
    value: str
    validation_result: ValidationResult | None = None

@dataclass
class Blurred(Message):
    input: Input
    value: str
    validation_result: ValidationResult | None = None
```

## InputValidationOn
```python
InputValidationOn = Literal["blur", "changed", "submitted"]
```

## Extensive Key Bindings
```python
BINDINGS = [
    # Cursor movement
    Binding("left", "cursor_left"),
    Binding("shift+left", "cursor_left(True)"),  # With selection
    Binding("ctrl+left", "cursor_left_word"),
    Binding("ctrl+shift+left", "cursor_left_word(True)"),
    Binding("right", "cursor_right"),
    Binding("shift+right", "cursor_right(True)"),
    Binding("ctrl+right", "cursor_right_word"),
    Binding("ctrl+shift+right", "cursor_right_word(True)"),
    Binding("home,ctrl+a", "home"),
    Binding("end,ctrl+e", "end"),
    Binding("shift+home", "home(True)"),
    Binding("shift+end", "end(True)"),

    # Deletion
    Binding("backspace", "delete_left"),
    Binding("delete,ctrl+d", "delete_right"),
    Binding("ctrl+w", "delete_left_word"),
    Binding("ctrl+u", "delete_left_all"),
    Binding("ctrl+f", "delete_right_word"),
    Binding("ctrl+k", "delete_right_all"),

    # Selection
    Binding("ctrl+shift+a", "select_all"),

    # Clipboard
    Binding("ctrl+x", "cut"),
    Binding("ctrl+c", "copy"),
    Binding("ctrl+v", "paste"),

    # Submit
    Binding("enter", "submit"),
]
```

## Component Classes (for styling)
- `input--cursor` - The cursor character
- `input--placeholder` - Placeholder text
- `input--suggestion` - Autocomplete suggestion text
- `input--selection` - Selected text background

## DEFAULT_CSS
```css
Input {
    background: $surface;
    color: $foreground;
    padding: 0 2;
    border: tall $border-blurred;
    width: 100%;
    height: 3;
    scrollbar-size-horizontal: 0;

    &:focus { border: tall $border; }
    &.-invalid { border: tall $error 60%; }
    &.-textual-compact { border: none; height: 1; }

    &>.input--cursor { ... }
    &>.input--selection { ... }
    &>.input--placeholder, &>.input--suggestion { color: $text-disabled; }
}
```

---

## Validator System

### ValidationResult
```python
@dataclass
class ValidationResult:
    failures: Sequence[Failure] = field(default_factory=list)

    @staticmethod
    def merge(results: Sequence["ValidationResult"]) -> "ValidationResult"
    @staticmethod
    def success() -> ValidationResult
    @staticmethod
    def failure(failures: Sequence[Failure]) -> ValidationResult

    @property
    def is_valid(self) -> bool
    @property
    def failure_descriptions(self) -> list[str]
```

### Failure
```python
@dataclass
class Failure:
    validator: Validator
    value: str | None = None
    description: str | None = None
```

### Validator (ABC)
```python
class Validator(ABC):
    @abstractmethod
    def validate(self, value: str) -> ValidationResult:
        """Validate the given value."""

    # Built-in validators: Integer, Number, Length, Regex, URL, etc.
```

---

## Suggester System

### Suggester (ABC)
```python
class Suggester(ABC):
    cache: LRUCache[str, str | None] | None
    case_sensitive: bool

    def __init__(self, *, use_cache: bool = True, case_sensitive: bool = False)

    @abstractmethod
    async def get_suggestion(self, value: str) -> str | None:
        """Return a suggestion for the value, or None."""
```

### SuggestionReady Message
```python
@dataclass
class SuggestionReady(Message):
    value: str      # Input value suggestion is for
    suggestion: str # The suggestion text
```

### Built-in: SuggestFromList
```python
class SuggestFromList(Suggester):
    """Suggest from a fixed list of options."""
```

---

## Rust Implementation Considerations

### Selection Struct
```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub struct Selection {
    pub start: usize,
    pub end: usize,
}

impl Selection {
    pub fn cursor(position: usize) -> Self {
        Self { start: position, end: position }
    }

    pub fn is_empty(&self) -> bool {
        self.start == self.end
    }
}
```

### InputType Enum
```rust
pub enum InputType {
    Text,
    Integer,
    Number,
}
```

### Validator Trait
```rust
pub trait Validator: Send + Sync {
    fn validate(&self, value: &str) -> ValidationResult;
}

pub struct ValidationResult {
    pub failures: Vec<Failure>,
}

impl ValidationResult {
    pub fn success() -> Self { Self { failures: vec![] } }
    pub fn is_valid(&self) -> bool { self.failures.is_empty() }
}
```

### Suggester Trait
```rust
#[async_trait]
pub trait Suggester: Send + Sync {
    async fn get_suggestion(&self, value: &str) -> Option<String>;
}
```

### Key Challenges
1. Input extends ScrollView for horizontal scrolling of long text
2. Extensive cursor/selection management with cell-width handling (Unicode)
3. Timer-based cursor blinking
4. Clipboard integration (ctrl+c/v/x)
5. Async suggester with caching
6. Multiple validation triggers (blur, changed, submitted)
