# Ancestor Chain Construction Design

## Overview

The ancestor chain is the list of widgets from the root to the immediate parent of the current widget. It's essential for:
1. Matching descendant/child CSS selectors
2. Propagating parent background for auto color resolution
3. Cache invalidation when ancestors change

## Order Convention

Ancestors are ordered **root → parent** (not parent → root):

```
Widget tree:  Screen > Container > Horizontal > Button

When rendering Button:
  ancestors[0] = Screen     (root)
  ancestors[1] = Container
  ancestors[2] = Horizontal (immediate parent)

Button is NOT in ancestors (it's the current widget being rendered)
```

This matches CSS selector semantics where `A B` means "B is a descendant of A".

## Construction During Render Traversal

### RenderContext Propagation

The ancestor chain is built incrementally through `RenderContext::child_context()`:

```rust
impl<'a> RenderContext<'a> {
    /// Create a child context with the current widget added to ancestors.
    pub fn child_context(&self, parent: &impl Widget, parent_style: &ComputedStyle) -> Self {
        let mut ancestors = self.ancestors.clone();
        ancestors.push(parent.widget_meta());

        Self {
            style_manager: self.style_manager,
            parent_background: parent_style.background
                .clone()
                .unwrap_or_else(|| self.parent_background.clone()),
            ancestors,
        }
    }
}
```

### Render Flow

```rust
// Root level (run_managed loop)
let context = RenderContext::new(style_manager);
// context.ancestors = []

// Screen renders
fn render_with_context(&self, id: WidgetId, frame: &mut Frame, ctx: &RenderContext) {
    let style = ctx.style_for(id, self);
    // ctx.ancestors = [] when computing Screen's style

    let child_ctx = ctx.child_context(self, &style);
    // child_ctx.ancestors = [Screen]

    for (child_id, child_area, child) in children_with_layout() {
        child.render_with_context(child_id, child_area, frame, &child_ctx);
    }
}

// Container renders (child of Screen)
fn render_with_context(&self, id: WidgetId, frame: &mut Frame, ctx: &RenderContext) {
    let style = ctx.style_for(id, self);
    // ctx.ancestors = [Screen] when computing Container's style

    let child_ctx = ctx.child_context(self, &style);
    // child_ctx.ancestors = [Screen, Container]

    for (child_id, child_area, child) in children_with_layout() {
        child.render_with_context(child_id, child_area, frame, &child_ctx);
    }
}

// Button renders (child of Container)
fn render_with_context(&self, id: WidgetId, frame: &mut Frame, ctx: &RenderContext) {
    let style = ctx.style_for(id, self);
    // ctx.ancestors = [Screen, Container] when computing Button's style
    // CSS selector "Screen Container Button" can now match
}
```

## WidgetMeta for Ancestors

Each ancestor is represented as a `WidgetMeta`:

```rust
pub struct WidgetMeta {
    pub type_name: String,
    pub id: Option<String>,
    pub classes: Vec<String>,
    pub pseudo_classes: HashSet<String>,
}
```

### Widget Trait Method

Widgets provide their metadata via `widget_meta()`:

```rust
pub trait Widget {
    fn widget_meta(&self) -> WidgetMeta {
        WidgetMeta {
            type_name: self.widget_type_name().to_string(),
            id: self.id().map(|s| s.to_string()),
            classes: self.classes().to_vec(),
            pseudo_classes: self.pseudo_classes(),
        }
    }
}
```

## Memory Considerations

### Cloning Overhead

Each `child_context()` clones the entire ancestor chain:

```rust
let mut ancestors = self.ancestors.clone();  // O(n) where n = depth
ancestors.push(parent.widget_meta());         // O(1) amortized
```

For a tree of depth D with W widgets per level:
- Total clones: O(D² × W)
- Memory: O(D) per context

### Optimization: Shared Ancestors

For deep trees, consider reference-counted sharing:

```rust
pub struct RenderContext<'a> {
    // Instead of Vec<WidgetMeta>
    ancestors: Rc<Vec<WidgetMeta>>,
}

impl<'a> RenderContext<'a> {
    pub fn child_context(&self, parent: &impl Widget, parent_style: &ComputedStyle) -> Self {
        let mut new_ancestors = (*self.ancestors).clone();
        new_ancestors.push(parent.widget_meta());

        Self {
            ancestors: Rc::new(new_ancestors),
            // ...
        }
    }
}
```

This is a micro-optimization; only implement if profiling shows ancestor cloning as a bottleneck.

### Optimization: COW Vectors

Use copy-on-write semantics:

```rust
use std::borrow::Cow;

pub struct RenderContext<'a> {
    ancestors: Cow<'a, [WidgetMeta]>,
}
```

This avoids cloning until modification is needed.

## Selector Matching

### Descendant Selector (`A B`)

"B is a descendant of A" at any depth:

```rust
fn matches_descendant(
    widget: &WidgetMeta,      // B
    ancestors: &[&WidgetMeta], // [..., A, ...]
    ancestor_selector: &CompoundSelector, // A
    widget_selector: &CompoundSelector,   // B
) -> bool {
    // Widget must match B
    if !widget.matches_compound(widget_selector) {
        return false;
    }

    // Some ancestor must match A
    ancestors.iter().any(|anc| anc.matches_compound(ancestor_selector))
}
```

### Child Selector (`A > B`)

"B is a direct child of A":

```rust
fn matches_child(
    widget: &WidgetMeta,
    ancestors: &[&WidgetMeta],
    parent_selector: &CompoundSelector,
    widget_selector: &CompoundSelector,
) -> bool {
    // Widget must match B
    if !widget.matches_compound(widget_selector) {
        return false;
    }

    // Immediate parent must match A
    ancestors.last()
        .map(|parent| parent.matches_compound(parent_selector))
        .unwrap_or(false)
}
```

### Complex Selector Chain

For selectors like `A B > C D`:

```rust
// Already implemented in WidgetMeta::matches_complex_with_ancestors()
// See style.rs:4324-4349
```

## Edge Cases

### Root Widget

The root widget has an empty ancestor chain:

```rust
let root_ctx = RenderContext::new(style_manager);
assert!(root_ctx.ancestors.is_empty());
```

Selectors requiring ancestors won't match the root:
- `Container Button` - won't match if Button is root
- `* Button` - universal selector still won't provide an ancestor

### Single-Widget App

Apps with just one widget:

```rust
fn view(&self, frame: &mut Frame, ctx: &RenderContext) {
    // ancestors = []
    let style = ctx.style_for(self.root_id, &self.root_widget);
    // Only type/class/id selectors can match
}
```

### Dynamic Reparenting

If a widget is moved to a different parent:

```rust
// Widget was: Screen > OldParent > Widget
// Now is:    Screen > NewParent > Widget

// The ancestor chain changes, so:
// 1. Old cached style becomes stale (ancestor_hash mismatch)
// 2. New style is computed with new ancestor chain
// 3. New style is cached
```

The cache handles this automatically via `ancestor_hash`.

## Testing

```rust
#[test]
fn test_ancestor_chain_construction() {
    let sm = StyleManager::new();
    let ctx = RenderContext::new(&sm);

    assert!(ctx.ancestors.is_empty());

    let screen = Screen::new();
    let ctx1 = ctx.child_context(&screen, &ComputedStyle::default());
    assert_eq!(ctx1.ancestors.len(), 1);
    assert_eq!(ctx1.ancestors[0].type_name, "Screen");

    let container = Container::new();
    let ctx2 = ctx1.child_context(&container, &ComputedStyle::default());
    assert_eq!(ctx2.ancestors.len(), 2);
    assert_eq!(ctx2.ancestors[0].type_name, "Screen");
    assert_eq!(ctx2.ancestors[1].type_name, "Container");
}

#[test]
fn test_descendant_selector_matching() {
    // Stylesheet: Container Button { color: red; }
    let mut sm = StyleManager::new();
    sm.load_user_stylesheet("Container Button { color: red; }").unwrap();

    let screen = WidgetMeta::new("Screen");
    let container = WidgetMeta::new("Container");
    let button = WidgetMeta::new("Button");
    let mut registry = WidgetRegistry::new();
    let widget_id = registry.add(Button::new("Test"));

    // Button with Container ancestor
    let ancestors: Vec<&WidgetMeta> = vec![&screen, &container];
    let style = sm.get_style(widget_id, &button, &ancestors, &Color::default());

    assert_eq!(style.color.unwrap().r, 255);  // Red matched
}

#[test]
fn test_child_selector_matching() {
    // Stylesheet: Container > Button { color: blue; }
    let mut sm = StyleManager::new();
    sm.load_user_stylesheet("Container > Button { color: blue; }").unwrap();

    let screen = WidgetMeta::new("Screen");
    let container = WidgetMeta::new("Container");
    let horizontal = WidgetMeta::new("Horizontal");
    let button = WidgetMeta::new("Button");
    let mut registry = WidgetRegistry::new();
    let widget_id1 = registry.add(Button::new("Test"));
    let widget_id2 = registry.add(Button::new("Other"));

    // Button as direct child of Container
    let ancestors1: Vec<&WidgetMeta> = vec![&screen, &container];
    let style1 = sm.get_style(widget_id1, &button, &ancestors1, &Color::default());
    assert_eq!(style1.color.unwrap().b, 255);  // Blue matched

    // Button as grandchild of Container (through Horizontal)
    let ancestors2: Vec<&WidgetMeta> = vec![&screen, &container, &horizontal];
    let style2 = sm.get_style(widget_id2, &button, &ancestors2, &Color::default());
    assert!(style2.color.is_none());  // Selector didn't match
}
```
