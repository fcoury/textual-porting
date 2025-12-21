# Analysis: Python Textual css/query.py

## Overview

The `query.py` file implements `DOMQuery`, a jQuery-inspired API for selecting and manipulating DOM nodes using CSS selectors.

## Key Components

### 1. DOMQuery Class

```python
@rich.repr.auto(angular=True)
class DOMQuery(Generic[QueryType]):
    __slots__ = ["_node", "_nodes", "_filters", "_excludes", "_deep"]

    def __init__(
        self,
        node: DOMNode,
        *,
        filter: str | None = None,
        exclude: str | None = None,
        deep: bool = True,
        parent: DOMQuery | None = None,
    ):
        self._node = node
        self._nodes: list[QueryType] | None = None  # Lazy evaluation
        # Copy filters/excludes from parent query for chaining
        self._filters: list[tuple[SelectorSet, ...]] = (
            parent._filters.copy() if parent else []
        )
        self._excludes: list[tuple[SelectorSet, ...]] = (
            parent._excludes.copy() if parent else []
        )
        self._deep = deep
```

### 2. Lazy Node Evaluation

Nodes are evaluated only when accessed:

```python
@property
def nodes(self) -> list[QueryType]:
    if self._nodes is None:
        # Walk the DOM tree
        initial_nodes = list(
            self._node.walk_children(Widget) if self._deep else self._node._nodes
        )
        # Apply filters (AND logic)
        nodes = [
            node for node in initial_nodes
            if all(match(selector_set, node) for selector_set in self._filters)
        ]
        # Apply excludes (OR logic for exclusion)
        nodes = [
            node for node in nodes
            if not any(match(selector_set, node) for selector_set in self._excludes)
        ]
        self._nodes = cast("list[QueryType]", nodes)
    return self._nodes
```

### 3. Selector Matching

Uses `parse_selectors()` and `match()` from css module:

```python
from textual.css.match import match
from textual.css.parse import parse_selectors

# Filter parsing
self._filters.append(parse_selectors(filter))

# Matching during node evaluation
all(match(selector_set, node) for selector_set in self._filters)
```

## Query Errors

```python
class QueryError(Exception): pass
class InvalidQueryFormat(QueryError): pass  # Bad selector syntax
class NoMatches(QueryError): pass           # No results
class TooManyMatches(QueryError): pass      # More than expected
class WrongType(QueryError): pass           # Type mismatch
```

## Query Methods

### Selection Methods

| Method | Description |
|--------|-------------|
| `filter(selector)` | Add filter, return new query |
| `exclude(selector)` | Add exclusion, return new query |
| `first()` | Get first match (raises NoMatches) |
| `last()` | Get last match (raises NoMatches) |
| `only_one()` | Get single match (raises TooManyMatches) |
| `results(filter_type)` | Iterator with optional type filter |

### Manipulation Methods

| Method | Description |
|--------|-------------|
| `set_class(add, *names)` | Add or remove classes |
| `add_class(*names)` | Add classes |
| `remove_class(*names)` | Remove classes |
| `toggle_class(*names)` | Toggle classes |
| `set_styles(css, **kwargs)` | Apply inline styles |
| `refresh()` | Trigger refresh |
| `focus()` | Focus first focusable |
| `blur()` | Blur if focused |
| `remove()` | Remove nodes from DOM |
| `set(display, visible, disabled, loading)` | Set common attributes |

### Type Safety

Generics ensure type safety:

```python
def first(self, expect_type: type[ExpectType] | None = None) -> QueryType | ExpectType:
    first = self.nodes[0]
    if expect_type is not None:
        if not isinstance(first, expect_type):
            raise WrongType(...)
    return first
```

## Selector Matching Algorithm

From `css/match.py` (imported):

1. **Selector Set**: Multiple selectors combined with `,`
2. **Selector**: Chain of simple selectors (e.g., `#id .class Type`)
3. **Simple Selector**: Single selector (`#id`, `.class`, `Type`, `:pseudo`)

**Supported Combinators:**
- `DESCENDENT` (space): Matches any descendant
- `CHILD` (`>`): Matches direct child only
- `SAME`: Chained simple selectors on same element (e.g., `Button.primary`)

**Note:** Sibling combinators (`~`, `+`) are NOT supported in Textual CSS.

Match process:
```python
def match(selector_set: tuple[SelectorSet, ...], node: DOMNode) -> bool:
    # Any selector in the set matches
    return any(
        _match_selector(selector, node)
        for selector_set in selector_sets
        for selector in selector_set
    )

def _match_selector(selector: Selector, node: DOMNode) -> bool:
    # Walk up the DOM tree matching each part of the selector
    # Handle DESCENDENT (walk ancestors) and CHILD (parent only) combinators
```

## Rust Implementation Strategy

### Data Structures

```rust
pub struct DOMQuery<T: Widget> {
    node: Rc<DOMNode>,
    nodes: OnceCell<Vec<Rc<T>>>,  // Lazy evaluation
    filters: Vec<SelectorSet>,
    excludes: Vec<SelectorSet>,
    deep: bool,
}

pub enum QueryError {
    InvalidFormat(String),
    NoMatches,
    TooManyMatches,
    WrongType { expected: &'static str, found: String },
}
```

### Key Traits

```rust
trait Query<T: Widget> {
    fn nodes(&self) -> &[Rc<T>];
    fn filter(self, selector: &str) -> Result<Self, QueryError>;
    fn exclude(self, selector: &str) -> Result<Self, QueryError>;
    fn first(&self) -> Result<Rc<T>, QueryError>;
    fn first_of<U: Widget>(&self) -> Result<Rc<U>, QueryError>;
}
```

### Selector Matching

```rust
fn match_selector_set(selectors: &[SelectorSet], node: &DOMNode) -> bool {
    selectors.iter().any(|ss| match_selector(ss, node))
}

fn match_selector(selector: &Selector, node: &DOMNode) -> bool {
    // Match simple selectors from right to left
    // Handle combinators and walk DOM tree
}
```
