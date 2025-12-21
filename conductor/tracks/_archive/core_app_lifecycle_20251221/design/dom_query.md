# DOM Query API Design

## Overview

The DOM query API provides a fluent interface for finding and manipulating widgets in the widget tree. It supports CSS selector syntax for filtering, type-safe widget casting, and batch operations on matched widgets.

## Goals

1. **CSS Selector Support**: Query by ID, class, type, and combinators
2. **Type Safety**: Compile-time type checking for widget types
3. **Fluent API**: Chainable methods for filtering and actions
4. **Lazy Evaluation**: Only compute results when needed
5. **JQuery-like Experience**: Familiar API for web developers

## Design Decisions

### 1. Iterator-Based with Lazy Evaluation

Queries are evaluated lazily when results are accessed:

```rust
// Query is defined but not executed
let mut query = registry.query(root_id, ".button")?;

// Now the tree is walked
for id in query.iter() {
    // Access widgets via registry
    if let Some(button) = registry.get(id) {
        // ...
    }
}
```

### 2. WidgetId-Based Results (Not Generic)

**Design Change**: DOMQuery returns `WidgetId`s rather than carrying a generic type parameter. This avoids Rust's `Sized` constraint issues with `dyn Widget`.

```rust
// Returns DOMQuery (WidgetIds)
let mut query = registry.query(root_id, "*")?;
let ids: Vec<WidgetId> = query.to_vec();

// For typed access, use TypedQueryResults separately
let results = registry.query_typed::<Button>(root_id, "Button")?;
if let Some(button) = results.first() {
    // button is &Button
}
```

**Rationale**: Using `DOMQuery<'a, T = dyn Widget>` causes compilation errors because `dyn Widget` is not `Sized`. Since queries return `WidgetId`s anyway, the type parameter adds complexity without benefit.

### 3. CSS Selector Subset

We support a practical subset of CSS selectors:
- Type selectors: `Button`, `Label`
- ID selectors: `#my-widget`
- Class selectors: `.active`, `.hidden`
- Descendant combinator: `Container Button`
- Child combinator: `Container > Button`
- Attribute selectors (future): `[disabled]`

### 4. Widget Metadata Access

Widgets that support ID and class queries must provide this metadata. Rather than extending the base Widget trait (which would be invasive), we use runtime downcasting to check for metadata on widgets that support it (e.g., Container).

```rust
// Container supports ID and classes
let container = Container::new()
    .with_id("main")
    .with_class("active");

// Other widgets matched by type name only
let label = Label::new("Hello"); // Matches "Label" selector
```

### 5. Immutable Queries, Separate Mutation

**Design Change**: DOMQuery holds an immutable reference to the registry. Batch mutation operations are provided separately on WidgetRegistry, taking the query results as input.

```rust
// Query is immutable
let mut query = registry.query(root_id, ".item")?;
let ids = query.to_vec();

// Mutation is separate
for id in &ids {
    if let Some(widget) = registry.get_mut(*id) {
        // mutate widget
    }
}
```

**Rationale**: Holding `&mut WidgetRegistry` in DOMQuery would prevent having multiple queries or other operations active simultaneously.

## Core Types

### Selector Parsing

```rust
/// A parsed CSS selector.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum Selector {
    /// Match any widget: `*`
    Universal,
    /// Match by widget type name: `Button`
    Type(String),
    /// Match by ID: `#my-id`
    Id(String),
    /// Match by class: `.active`
    Class(String),
    /// Compound selector: `Button.active#submit`
    Compound(Vec<Selector>),
    /// Descendant combinator: `Container Button`
    Descendant(Box<Selector>, Box<Selector>),
    /// Child combinator: `Container > Button`
    Child(Box<Selector>, Box<Selector>),
}

impl Selector {
    /// Parse a selector string.
    pub fn parse(s: &str) -> Result<Selector, SelectorError> {
        // Parse CSS selector syntax
        // Handles: *, Type, #id, .class, Type.class, Parent Child, Parent > Child
        todo!()
    }

    /// Check if a widget matches this selector.
    /// Uses MatchContext for widget metadata access.
    pub fn matches(&self, widget: &dyn Widget, context: &MatchContext<'_>) -> bool {
        match self {
            Selector::Universal => true,
            Selector::Type(name) => {
                // Match by type name (handles full path or simple name)
                let type_name = std::any::type_name_of_val(widget);
                type_name == name || type_name.ends_with(&format!("::{}", name))
            }
            Selector::Id(id) => context.widget_id.as_deref() == Some(id.as_str()),
            Selector::Class(class) => context.widget_classes.contains(&class.as_str()),
            Selector::Compound(parts) => parts.iter().all(|p| p.matches(widget, context)),
            Selector::Descendant(ancestor_sel, descendant_sel) => {
                descendant_sel.matches(widget, context) &&
                context.ancestors.iter().any(|&id| {
                    context.registry.get(id).map_or(false, |w| {
                        let ctx = MatchContext::for_widget(context.registry, id, w);
                        ancestor_sel.matches(w, &ctx)
                    })
                })
            }
            Selector::Child(parent_sel, child_sel) => {
                child_sel.matches(widget, context) &&
                context.ancestors.first().map_or(false, |&id| {
                    context.registry.get(id).map_or(false, |w| {
                        let ctx = MatchContext::for_widget(context.registry, id, w);
                        parent_sel.matches(w, &ctx)
                    })
                })
            }
        }
    }
}

/// Context for matching selectors against a widget.
/// Provides access to widget metadata (ID, classes) via downcasting.
pub struct MatchContext<'a> {
    pub registry: &'a WidgetRegistry,
    pub widget_id: Option<String>,      // From Container.id() etc.
    pub widget_classes: Vec<&'a str>,   // From Container.classes() etc.
    pub ancestors: Vec<WidgetId>,       // Parent chain
}

impl<'a> MatchContext<'a> {
    /// Create context for a widget, extracting metadata via downcasting.
    pub fn for_widget(registry: &'a WidgetRegistry, id: WidgetId, widget: &dyn Widget) -> Self {
        let (widget_id, classes) = extract_widget_metadata(widget);
        let ancestors = registry.ancestors(id).collect();
        Self { registry, widget_id, widget_classes: classes, ancestors }
    }
}

/// Extract ID and classes from widgets that support them (e.g., Container).
fn extract_widget_metadata(widget: &dyn Widget) -> (Option<String>, Vec<&str>) {
    // Try downcasting to Container, Horizontal, Vertical, etc.
    if let Some(c) = widget.as_any().downcast_ref::<Container>() {
        return (c.id().map(String::from), c.classes().iter().map(|s| s.as_str()).collect());
    }
    // Add other widget types as needed
    (None, Vec::new())
}
```

### DOMQuery

```rust
/// A query over the widget tree.
/// Returns WidgetIds - no generic type parameter to avoid Sized issues.
pub struct DOMQuery<'a> {
    /// The registry to search.
    registry: &'a WidgetRegistry,
    /// The starting widget ID.
    start: WidgetId,
    /// Include filters (all must match).
    filters: Vec<Selector>,
    /// Exclude filters (none must match).
    excludes: Vec<Selector>,
    /// Whether to search deeply (descendants) or shallow (children only).
    deep: bool,
    /// Cached results.
    cache: Option<Vec<WidgetId>>,
}

impl<'a> DOMQuery<'a> {
    /// Create a new deep query starting from a widget.
    pub fn new(registry: &'a WidgetRegistry, start: WidgetId) -> Self {
        Self {
            registry,
            start,
            filters: Vec::new(),
            excludes: Vec::new(),
            deep: true,
            cache: None,
        }
    }

    /// Create a shallow query (immediate children only).
    pub fn children(registry: &'a WidgetRegistry, start: WidgetId) -> Self {
        Self {
            registry,
            start,
            filters: Vec::new(),
            excludes: Vec::new(),
            deep: false,
            cache: None,
        }
    }

    /// Add a filter selector.
    pub fn filter(mut self, selector: &str) -> Result<Self, SelectorError> {
        self.filters.push(Selector::parse(selector)?);
        self.cache = None;
        Ok(self)
    }

    /// Add an exclusion selector.
    pub fn exclude(mut self, selector: &str) -> Result<Self, SelectorError> {
        self.excludes.push(Selector::parse(selector)?);
        self.cache = None;
        Ok(self)
    }

    /// Get all matching widget IDs.
    pub fn ids(&mut self) -> &[WidgetId] {
        if self.cache.is_none() {
            self.cache = Some(self.execute());
        }
        self.cache.as_ref().unwrap()
    }

    /// Execute the query.
    fn execute(&self) -> Vec<WidgetId> {
        let mut results = Vec::new();

        let widget_ids: Vec<WidgetId> = if self.deep {
            self.registry.walk_descendants(self.start).collect()
        } else {
            self.registry.get_children(self.start).to_vec()
        };

        for id in widget_ids {
            if let Some(widget) = self.registry.get(id) {
                let context = MatchContext::for_widget(self.registry, id, widget);

                let matches_filters = self.filters.is_empty() ||
                    self.filters.iter().all(|f| f.matches(widget, &context));

                let matches_excludes = self.excludes.iter()
                    .any(|e| e.matches(widget, &context));

                if matches_filters && !matches_excludes {
                    results.push(id);
                }
            }
        }

        results
    }
}
```

### Query Iterator

```rust
impl<'a> DOMQuery<'a> {
    /// Iterate over matching widget IDs.
    pub fn iter(&mut self) -> impl Iterator<Item = WidgetId> + '_ {
        self.ids().iter().copied()
    }

    /// Convert to a vector of IDs.
    pub fn to_vec(&mut self) -> Vec<WidgetId> {
        self.ids().to_vec()
    }

    /// Get the number of matching widgets.
    pub fn len(&mut self) -> usize {
        self.ids().len()
    }

    /// Check if no widgets match.
    pub fn is_empty(&mut self) -> bool {
        self.ids().is_empty()
    }
}
```

### Result Access Methods

```rust
impl<'a> DOMQuery<'a> {
    /// Get the first matching widget ID.
    pub fn first(&mut self) -> Option<WidgetId> {
        self.ids().first().copied()
    }

    /// Get the first matching widget ID, or error if none.
    pub fn first_or_err(&mut self) -> Result<WidgetId, QueryError> {
        self.first().ok_or(QueryError::NoMatches)
    }

    /// Get the last matching widget ID.
    pub fn last(&mut self) -> Option<WidgetId> {
        self.ids().last().copied()
    }

    /// Get the only matching widget ID (error if 0 or >1).
    pub fn only_one(&mut self) -> Result<WidgetId, QueryError> {
        match self.ids().len() {
            0 => Err(QueryError::NoMatches),
            1 => Ok(self.ids()[0]),
            n => Err(QueryError::TooManyMatches(n)),
        }
    }

    /// Get widget ID by index.
    pub fn get(&mut self, index: usize) -> Option<WidgetId> {
        self.ids().get(index).copied()
    }
}
```

### Type-Safe Casting

Type-safe access is provided via `TypedQueryResults`, created separately from the registry:

```rust
/// Query results with type-safe widget access.
pub struct TypedQueryResults<'a, T: Widget> {
    registry: &'a WidgetRegistry,
    ids: Vec<WidgetId>,
    _marker: PhantomData<T>,
}

impl<'a, T: Widget + 'static> TypedQueryResults<'a, T> {
    /// Create from registry and widget IDs.
    pub fn new(registry: &'a WidgetRegistry, ids: Vec<WidgetId>) -> Self {
        Self { registry, ids, _marker: PhantomData }
    }

    /// Get the first widget of the expected type.
    pub fn first(&self) -> Option<&T> {
        for id in &self.ids {
            if let Some(widget) = self.registry.get(*id) {
                if let Some(typed) = widget.as_any().downcast_ref::<T>() {
                    return Some(typed);
                }
            }
        }
        None
    }

    /// Get the first widget, or error if not found.
    pub fn first_or_err(&self) -> Result<&T, QueryError> {
        self.first().ok_or(QueryError::NoMatches)
    }

    /// Get all matching IDs.
    pub fn ids(&self) -> &[WidgetId] {
        &self.ids
    }

    /// Number of results.
    pub fn len(&self) -> usize {
        self.ids.len()
    }

    /// Check if empty.
    pub fn is_empty(&self) -> bool {
        self.ids.is_empty()
    }
}
```

### Batch Operations (Future)

**Note**: Batch operations require mutable registry access. Since DOMQuery holds an immutable reference, batch operations are performed separately:

```rust
// Query first (immutable)
let mut query = registry.query(root, ".item")?;
let ids = query.to_vec();

// Then mutate separately
for id in &ids {
    if let Some(widget) = registry.get_mut(*id) {
        // Apply mutations via downcasting
        if let Some(container) = widget.as_any_mut().downcast_mut::<Container>() {
            // container.add_class("selected"); // if Container supports this
        }
    }
}
```

Future enhancement: Add batch operation helpers on WidgetRegistry:

```rust
impl WidgetRegistry {
    /// Remove all widgets matching the IDs.
    pub fn remove_all(&mut self, ids: &[WidgetId]) -> Vec<Box<dyn Widget>> {
        ids.iter().filter_map(|id| self.remove(*id)).collect()
    }
}
```

## Registry Methods

Query methods are provided on WidgetRegistry rather than Widget (since widgets don't hold a reference to their registry):

```rust
impl WidgetRegistry {
    /// Query descendants matching a selector.
    pub fn query(&self, start: WidgetId, selector: &str) -> Result<DOMQuery<'_>, QueryError> {
        DOMQuery::new(self, start).filter(selector).map_err(Into::into)
    }

    /// Query immediate children matching a selector.
    pub fn query_children(&self, start: WidgetId, selector: &str) -> Result<DOMQuery<'_>, QueryError> {
        DOMQuery::children(self, start).filter(selector).map_err(Into::into)
    }

    /// Query for a single widget ID matching the selector.
    pub fn query_one(&self, start: WidgetId, selector: &str) -> Result<WidgetId, QueryError> {
        self.query(start, selector)?.first_or_err()
    }

    /// Query for exactly one widget (error if 0 or >1).
    pub fn query_exactly_one(&self, start: WidgetId, selector: &str) -> Result<WidgetId, QueryError> {
        self.query(start, selector)?.only_one()
    }

    /// Query with typed results.
    pub fn query_typed<T: Widget + 'static>(
        &self,
        start: WidgetId,
        selector: &str,
    ) -> Result<TypedQueryResults<'_, T>, QueryError> {
        let mut query = self.query(start, selector)?;
        Ok(TypedQueryResults::new(self, query.to_vec()))
    }

    /// Query for a single widget of a specific type.
    pub fn query_one_typed<T: Widget + 'static>(
        &self,
        start: WidgetId,
        selector: &str,
    ) -> Result<&T, QueryError> {
        self.query_typed::<T>(start, selector)?.first_or_err()
    }
}
```

## Usage Examples

### Basic Queries

```rust
// Query all buttons
for id in app.query("Button")? {
    println!("Found button: {:?}", id);
}

// Query by ID
let submit = app.query_one("#submit")?;

// Query by class
let active_items = app.query(".active")?;

// Compound selector
let active_buttons = app.query("Button.active")?;
```

### Type-Safe Access

```rust
// Get typed button reference
let submit: &Button = app.query_one_typed::<Button>("#submit")?;
submit.label = "Click Me".into();

// Iterate over typed widgets
for button in app.query("Button")?.as_type::<Button>().iter() {
    button.set_disabled(false);
}
```

### Chained Filtering

```rust
// Filter and exclude
let visible_buttons = app.query("Button")?
    .filter(".visible")?
    .exclude(".disabled")?;

// Get first match
if let Some(id) = visible_buttons.first() {
    app.focus(id);
}
```

### Batch Operations

```rust
// Add class to all matches
app.query(".item")?
    .add_class(&["selected", "highlighted"]);

// Remove class from all
app.query(".item.old")?
    .remove_class(&["old"]);

// Toggle class
app.query("Button")?
    .toggle_class("active");

// Set styles
app.query("Label")?
    .set_styles("color: red; font-weight: bold;")?;

// Refresh all
app.query(".dirty")?
    .refresh();

// Remove widgets
let removed = app.query(".temporary")?
    .remove();
```

### Descendant vs Child Queries

```rust
// Query all descendants (deep)
let all_buttons = container.query("Button")?;

// Query immediate children only (shallow)
let direct_children = container.query_children("Button")?;
```

### Error Handling

```rust
// Handle missing widget
match app.query_one("#nonexistent") {
    Ok(id) => println!("Found: {:?}", id),
    Err(QueryError::NoMatches) => println!("Widget not found"),
    Err(e) => return Err(e),
}

// Ensure exactly one match
let unique = app.query(".unique")?.only_one()?;
```

## Comparison with Python Textual

| Python Textual | Rust Implementation |
|---------------|---------------------|
| `query(selector)` | `query(selector)?` |
| `query_one(selector)` | `query_one(selector)?` |
| `query_children(selector)` | `query_children(selector)?` |
| `DOMQuery[Widget]` | `DOMQuery<'a, Widget>` |
| `.filter(selector)` | `.filter(selector)?` |
| `.exclude(selector)` | `.exclude(selector)?` |
| `.first()` | `.first()` |
| `.last()` | `.last()` |
| `.only_one()` | `.only_one()?` |
| `.results(Type)` | `.as_type::<Type>().iter()` |
| `.add_class(names)` | `.add_class(&names)` |
| `.remove_class(names)` | `.remove_class(&names)` |
| `.set_styles(css)` | `.set_styles(css)?` |
| `.refresh()` | `.refresh()` |
| `.remove()` | `.remove()` |

## Implementation Plan

### Phase 1: Selector Parsing
- [ ] Define `Selector` enum
- [ ] Implement basic selector parsing (type, id, class)
- [ ] Implement compound selectors
- [ ] Implement descendant/child combinators

### Phase 2: DOMQuery Core
- [ ] Implement `DOMQuery` struct
- [ ] Implement tree walking
- [ ] Implement filter matching
- [ ] Implement lazy evaluation

### Phase 3: Result Access
- [ ] Implement `first()`, `last()`, `only_one()`
- [ ] Implement iterator interface
- [ ] Implement type-safe casting

### Phase 4: Batch Operations
- [ ] Implement `add_class()`, `remove_class()`
- [ ] Implement `set_styles()`
- [ ] Implement `refresh()`
- [ ] Implement `remove()`

### Phase 5: Widget Integration
- [ ] Add `query()` method to Widget
- [ ] Add `query_one()` method
- [ ] Add `query_children()` method

## Open Questions

1. **Selector Caching**: Should parsed selectors be cached?

2. **Result Caching**: How long should query results be cached?

3. **Attribute Selectors**: Should we support `[attr=value]` syntax?

4. **Pseudo-Classes**: Should we support `:first-child`, `:hover`, etc.?

## Summary

This design provides:
- **CSS selector syntax** for familiar querying
- **Type-safe widget access** via generics
- **Lazy evaluation** for performance
- **Fluent API** for chaining operations
- **Batch operations** for efficient updates
- **JQuery-like experience** for web developers
