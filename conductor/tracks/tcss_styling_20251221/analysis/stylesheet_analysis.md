# Analysis: Python Textual css/stylesheet.py

## Overview

The `stylesheet.py` file implements the CSS cascade and specificity resolution system. It manages loading, parsing, and applying CSS rules to DOM nodes.

## Key Components

### 1. Stylesheet Class

```python
class Stylesheet:
    def __init__(self, *, variables: dict[str, str] | None = None):
        self._rules: list[RuleSet] = []
        self._rules_map: dict[str, list[RuleSet]] | None = None
        self._variables = variables or {}
        self.source: dict[CSSLocation, CssSource] = {}
        self._parse_cache: LRUCache[tuple, list[RuleSet]] = LRUCache(64)
```

### 2. CssSource NamedTuple

Tracks CSS origin for specificity:

```python
class CssSource(NamedTuple):
    content: str           # The CSS content
    is_defaults: bool      # Widget DEFAULT_CSS vs user CSS
    tie_breaker: int       # Priority for same-specificity rules
    scope: str             # CSS scope (widget type name)
```

### 3. Rules Map

Optimized rule lookup by selector name:

```python
@property
def rules_map(self) -> dict[str, list[RuleSet]]:
    """Maps selector names to matching rules for fast lookup."""
    # Keys: "#id", ".class", "TypeName"
    # Values: List of RuleSets that include that selector
```

## Cascade Algorithm

### Specificity Resolution

The `apply()` method implements the full CSS cascade:

1. **Filter Applicable Rules**
   ```python
   limit_rules = {
       rule
       for name in rules_map.keys() & node._selector_names
       for rule in rules_map[name]
   }
   ```

2. **Check Selector Match**
   ```python
   for rule in rules:
       for selector_set in rule.selector_set:
           if _check_selectors(selector_set.selectors, css_path_nodes):
               yield selector_set.specificity
   ```

3. **Collect Rules with Specificity**
   ```python
   rule_attributes: defaultdict[str, list[tuple[Specificity6, object]]]
   for key, rule_specificity, value in rule.styles.extract_rules(
       base_specificity, is_default_rules, tie_breaker
   ):
       rule_attributes[key].append((rule_specificity, value))
   ```

4. **Select Highest Specificity**
   ```python
   node_rules = {
       name: max(specificity_rules, key=itemgetter(0))[1]
       for name, specificity_rules in rule_attributes.items()
   }
   ```

### Specificity Types

```python
Specificity3 = tuple[int, int, int]  # (IDs, Classes, Types)
Specificity6 = tuple[int, int, int, int, int, int]
# (is_default, important, IDs, Classes, Types, tie_breaker)
```

The 6-tuple enables cascade priority:
1. `is_default`: 0 for user CSS, 1 for DEFAULT_CSS (user CSS wins)
2. `important`: !important rules get higher priority
3. `IDs`: ID selector count (#id)
4. `Classes`: Class/pseudo-class/attribute count (.class, :hover, [attr])
5. `Types`: Type selector count (Widget, Button)
6. `tie_breaker`: Source order for equal specificity

### Pseudo-Class Caching

Some pseudo-classes invalidate caching:

```python
_EXCLUDE_PSEUDO_CLASSES_FROM_CACHE = {
    "first-of-type", "last-of-type",
    "first-child", "last-child",
    "odd", "even",
    "focus-within", "empty",
}
```

## Style Application

### replace_rules()

Applies rules with optional animation:

```python
@classmethod
def replace_rules(cls, node: DOMNode, rules: RulesMap, animate: bool = False):
    if animate:
        # Check if property is animatable and has transition
        if is_animatable(key):
            transition = new_styles._get_transition(key)
            if transition:
                animator.animate(base, key, new_render_value, ...)
                continue
    # Default: direct assignment
    setattr(base_styles, key, new_value)
```

### Component Classes

Virtual nodes for styled components:

```python
def _process_component_classes(self, node: DOMNode):
    for component in sorted(component_classes):
        virtual_node = DOMNode(classes=component)
        virtual_node._attach(node)
        self.apply(virtual_node, animate=False)
        node._component_styles[component] = virtual_node.styles
```

## Rust Implementation Strategy

### Data Structures

```rust
struct Stylesheet {
    rules: Vec<RuleSet>,
    rules_map: HashMap<String, Vec<usize>>,  // Selector name → rule indices
    source: HashMap<CssLocation, CssSource>,
    variables: HashMap<String, String>,
    parse_cache: LruCache<CacheKey, Vec<RuleSet>>,
}

struct CssSource {
    content: String,
    is_defaults: bool,
    tie_breaker: i32,
    scope: String,
}

type Specificity6 = (i32, i32, i32, i32, i32, i32);
```

### Key Methods

```rust
impl Stylesheet {
    fn apply(&self, node: &mut DOMNode, animate: bool, cache: &mut StyleCache);
    fn check_rule(&self, rule: &RuleSet, path: &[&DOMNode]) -> Vec<Specificity3>;
    fn replace_rules(node: &mut DOMNode, rules: RulesMap, animate: bool);
}
```

### Performance Considerations

1. **Rules map indexing**: O(1) lookup by selector name
2. **LRU cache**: Avoid re-parsing identical CSS
3. **Style caching**: Cache by parent/classes/pseudo-classes
4. **Batch updates**: `update_nodes()` processes multiple nodes efficiently
