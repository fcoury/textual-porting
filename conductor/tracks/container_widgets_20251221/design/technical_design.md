# Container Widgets Technical Design

## Existing Infrastructure

### Already Implemented (container.rs)
- **Container** - Base container with id, classes, border, layout hints
- **Horizontal** - Wraps Container with `layout_type() = Horizontal`
- **Vertical** - Wraps Container with `layout_type() = Vertical`
- **Grid** - Wraps Container with GridConfig

### Already Implemented (scroll.rs)
- **ScrollState** - Full scroll position tracking with all methods
- **ScrollBar** - Visual scrollbar widget
- **ScrollView** - Scrollable container with overflow settings
- **Overflow/OverflowSettings** - Scrollbar visibility control

### Already Implemented (binding.rs)
- **Binding** - Key-to-action mapping
- **KeyPattern** - Parsed key patterns
- **BindingManager** - Key matching and dispatch
- **ActionHandler** trait - For widgets to handle actions

### Existing Constraint Enum (geometry.rs)
```rust
pub enum Constraint {
    Length(u16),      // Fixed size in cells
    Percentage(f32),  // 0.0 to 100.0
    Fraction(f32),    // Like CSS fr unit (1fr = Fraction(1.0))
    Auto,             // Size based on content
}
```

## Design Decisions

### 1. Container Base Approach

The existing `Container` struct is a concrete type. Instead of creating a trait, we'll follow the existing pattern of composition:

```rust
// Existing pattern (keep as-is):
pub struct Horizontal {
    inner: Container,
}

// New containers follow same pattern:
pub struct Center {
    inner: Container,
}

pub struct VerticalGroup {
    inner: Container,
}
```

**Rationale**: This matches the existing code style and avoids complex trait hierarchies.

### 2. Display Property for Visibility (NEW)

Add a `Display` enum to geometry.rs and extend LayoutHints. Display is returned directly from Widget::layout(), not from a separate Styles struct.

```rust
// In geometry.rs:
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum Display {
    /// Widget is visible and participates in layout.
    #[default]
    Block,
    /// Widget is hidden and skipped in layout (takes no space).
    None,
}

#[derive(Debug, Clone, Copy, Default)]
pub struct Alignment {
    pub horizontal: AlignH,
    pub vertical: AlignV,
}

#[derive(Debug, Clone, Copy, Default)]
pub enum AlignH {
    #[default]
    Left,
    Center,
    Right,
}

#[derive(Debug, Clone, Copy, Default)]
pub enum AlignV {
    #[default]
    Top,
    Middle,
    Bottom,
}

// Extend existing LayoutHints:
pub struct LayoutHints {
    pub width: Constraint,
    pub height: Constraint,
    pub min_size: Option<Size>,
    pub max_size: Option<Size>,
    pub dock: Option<Dock>,
    pub display: Display,       // NEW: Controls visibility
    pub alignment: Alignment,   // NEW: Child alignment
}

impl LayoutHints {
    /// Set display mode.
    pub fn with_display(mut self, display: Display) -> Self {
        self.display = display;
        self
    }

    /// Set alignment.
    pub fn with_alignment(mut self, alignment: Alignment) -> Self {
        self.alignment = alignment;
        self
    }

    /// Check if widget should be included in layout.
    pub fn is_visible(&self) -> bool {
        self.display != Display::None
    }
}
```

**Note on OverflowSettings**: OverflowSettings currently lives in scroll.rs. For LayoutHints to use it, either:
1. Move OverflowSettings to geometry.rs (preferred), or
2. Keep overflow handling in ScrollView/ScrollableContainer only

For this design, we'll keep overflow in scroll.rs and handle it within ScrollView.

### 2.1 Widget Trait Extensions (NEW)

Add new methods to the Widget trait in widget.rs:

```rust
pub trait Widget: Any + std::fmt::Debug {
    // ... existing methods ...

    /// Return the widget's string ID, if set.
    /// Used for ID-based lookups and display override coordination.
    fn id(&self) -> Option<&str> {
        None
    }

    /// Get minimum content width for Auto constraint sizing.
    fn get_content_width(&self) -> u16 {
        0
    }

    /// Get minimum content height for Auto constraint sizing.
    fn get_content_height(&self) -> u16 {
        0
    }

    /// Return grid configuration if this widget uses grid layout.
    fn grid_config(&self) -> Option<&GridConfig> {
        None
    }
}
```

### 2.2 ContentSwitcher Display Override

ContentSwitcher needs to override child display. Since children are in WidgetRegistry (not stored in widgets), we use a trait for widgets that can override child display:

```rust
/// Trait for widgets that can override child display (like ContentSwitcher).
pub trait ChildDisplayOverride {
    /// Return Display for a child widget by its ID.
    /// Returns None to use child's own display setting.
    fn child_display(&self, child_id: &str) -> Option<Display>;
}

impl ChildDisplayOverride for ContentSwitcher {
    fn child_display(&self, child_id: &str) -> Option<Display> {
        Some(if self.current.as_deref() == Some(child_id) {
            Display::Block
        } else {
            Display::None
        })
    }
}

// In layout.rs - get effective display for a child:
fn get_effective_display(
    registry: &WidgetRegistry,
    parent_id: WidgetId,
    child_id: WidgetId,
) -> Display {
    // Check if parent overrides child display
    if let Some(parent) = registry.get(parent_id) {
        if let Some(overrider) = parent.as_any().downcast_ref::<ContentSwitcher>() {
            if let Some(child) = registry.get(child_id) {
                if let Some(id) = child.id() {
                    if let Some(display) = overrider.child_display(id) {
                        return display;
                    }
                }
            }
        }
    }
    // Default: use child's own layout hints
    registry.get(child_id)
        .map(|w| w.layout().display)
        .unwrap_or(Display::Block)
}
```

### 3. Layout System Integration

#### 3.1 Layout Filtering for Display::None

Layout algorithms work with the WidgetRegistry to get children and filter by display. The layout system uses WidgetId, not widget references:

```rust
// In layout.rs - layout filtering with registry:

/// Get visible children for a parent widget.
fn get_visible_children(
    registry: &WidgetRegistry,
    parent_id: WidgetId,
) -> Vec<WidgetId> {
    registry.get_children(parent_id)
        .iter()
        .copied()
        .filter(|&child_id| {
            get_effective_display(registry, parent_id, child_id) != Display::None
        })
        .collect()
}

impl VerticalLayout {
    pub fn layout(
        &self,
        registry: &WidgetRegistry,
        parent_id: WidgetId,
        area: Rect,
    ) -> HashMap<WidgetId, Rect> {
        let visible_children = get_visible_children(registry, parent_id);
        self.layout_children(registry, parent_id, &visible_children, area)
    }
}
```

#### 3.2 Constraint Resolution with Fraction Support

The layout algorithm must properly handle all constraint types including Fraction. **Critical**: Fraction sizing requires a two-pass algorithm:

1. **Pass 1**: Calculate fixed sizes (Length, Percentage, Auto)
2. **Pass 2**: Distribute remaining space among Fraction constraints

```rust
/// Resolve constraints for a list of children on the main axis.
/// Returns sizes for each child.
fn resolve_main_axis_sizes(
    registry: &WidgetRegistry,
    children: &[WidgetId],
    available: u16,
    is_horizontal: bool,
) -> Vec<u16> {
    let mut sizes = vec![0u16; children.len()];
    let mut remaining = available;
    let mut total_fr = 0.0f32;

    // Pass 1: Resolve fixed constraints and tally fractions
    for (i, &child_id) in children.iter().enumerate() {
        let Some(child) = registry.get(child_id) else { continue };
        let hints = child.layout();
        let constraint = if is_horizontal { hints.width } else { hints.height };

        match constraint {
            Constraint::Length(n) => {
                let size = n.min(remaining);
                sizes[i] = size;
                remaining = remaining.saturating_sub(size);
            }
            Constraint::Percentage(p) => {
                let size = ((available as f32 * p / 100.0) as u16).min(remaining);
                sizes[i] = size;
                remaining = remaining.saturating_sub(size);
            }
            Constraint::Auto => {
                let content = if is_horizontal {
                    child.get_content_width()
                } else {
                    child.get_content_height()
                };
                let size = content.min(remaining);
                sizes[i] = size;
                remaining = remaining.saturating_sub(size);
            }
            Constraint::Fraction(fr) => {
                total_fr += fr;
                // Will be resolved in pass 2
            }
        }
    }

    // Pass 2: Distribute remaining space among fractions
    if total_fr > 0.0 && remaining > 0 {
        let per_fr = remaining as f32 / total_fr;

        for (i, &child_id) in children.iter().enumerate() {
            let Some(child) = registry.get(child_id) else { continue };
            let hints = child.layout();
            let constraint = if is_horizontal { hints.width } else { hints.height };

            if let Constraint::Fraction(fr) = constraint {
                sizes[i] = (fr * per_fr).round() as u16;
            }
        }
    }

    sizes
}
```

#### 3.3 Alignment with Proper Fraction Handling

Alignment must work after fractions are resolved. The layout calculates the total used size (which may be less than available if no fractions), then applies alignment:

```rust
impl VerticalLayout {
    fn layout_children(
        &self,
        registry: &WidgetRegistry,
        parent_id: WidgetId,
        children: &[WidgetId],
        area: Rect,
    ) -> HashMap<WidgetId, Rect> {
        let parent = registry.get(parent_id).unwrap();
        let parent_hints = parent.layout();

        // Resolve heights (main axis)
        let heights = resolve_main_axis_sizes(registry, children, area.height, false);
        let total_height: u16 = heights.iter().sum();

        // Apply VERTICAL alignment to the GROUP (main axis)
        // Only applies if total_height < area.height (no fractions filled it)
        let group_start_y = area.y + match parent_hints.alignment.vertical {
            AlignV::Top => 0,
            AlignV::Middle => (area.height.saturating_sub(total_height) / 2) as i16,
            AlignV::Bottom => (area.height.saturating_sub(total_height)) as i16,
        };

        let mut results = HashMap::new();
        let mut current_y = group_start_y;

        for (i, &child_id) in children.iter().enumerate() {
            let child = registry.get(child_id).unwrap();
            let child_hints = child.layout();
            let child_height = heights[i];

            // Resolve width (cross axis)
            let child_width = match child_hints.width {
                Constraint::Length(n) => n.min(area.width),
                Constraint::Percentage(p) => ((area.width as f32 * p / 100.0) as u16).min(area.width),
                Constraint::Auto => child.get_content_width().min(area.width),
                Constraint::Fraction(_) => area.width,  // Fractions fill cross axis
            };

            // Apply HORIZONTAL alignment to EACH CHILD (cross axis)
            let child_x = area.x + match parent_hints.alignment.horizontal {
                AlignH::Left => 0,
                AlignH::Center => (area.width.saturating_sub(child_width) / 2) as i16,
                AlignH::Right => (area.width.saturating_sub(child_width)) as i16,
            };

            results.insert(child_id, Rect::new(child_x, current_y, child_width, child_height));
            current_y += child_height as i16;
        }

        results
    }
}

impl HorizontalLayout {
    fn layout_children(
        &self,
        registry: &WidgetRegistry,
        parent_id: WidgetId,
        children: &[WidgetId],
        area: Rect,
    ) -> HashMap<WidgetId, Rect> {
        let parent = registry.get(parent_id).unwrap();
        let parent_hints = parent.layout();

        // Resolve widths (main axis)
        let widths = resolve_main_axis_sizes(registry, children, area.width, true);
        let total_width: u16 = widths.iter().sum();

        // Apply HORIZONTAL alignment to the GROUP (main axis)
        let group_start_x = area.x + match parent_hints.alignment.horizontal {
            AlignH::Left => 0,
            AlignH::Center => (area.width.saturating_sub(total_width) / 2) as i16,
            AlignH::Right => (area.width.saturating_sub(total_width)) as i16,
        };

        let mut results = HashMap::new();
        let mut current_x = group_start_x;

        for (i, &child_id) in children.iter().enumerate() {
            let child = registry.get(child_id).unwrap();
            let child_hints = child.layout();
            let child_width = widths[i];

            // Resolve height (cross axis)
            let child_height = match child_hints.height {
                Constraint::Length(n) => n.min(area.height),
                Constraint::Percentage(p) => ((area.height as f32 * p / 100.0) as u16).min(area.height),
                Constraint::Auto => child.get_content_height().min(area.height),
                Constraint::Fraction(_) => area.height,  // Fractions fill cross axis
            };

            // Apply VERTICAL alignment to EACH CHILD (cross axis)
            let child_y = area.y + match parent_hints.alignment.vertical {
                AlignV::Top => 0,
                AlignV::Middle => (area.height.saturating_sub(child_height) / 2) as i16,
                AlignV::Bottom => (area.height.saturating_sub(child_height)) as i16,
            };

            results.insert(child_id, Rect::new(current_x, child_y, child_width, child_height));
            current_x += child_width as i16;
        }

        results
    }
}
```

### 4. Container Implementations

#### 4.1 Alignment Containers

```rust
// src/widgets/container.rs (additions)

/// Horizontally centers children.
pub struct Center {
    inner: Container,
}

impl Center {
    pub fn new() -> Self {
        Self {
            inner: Container::new()
                .with_width(Constraint::Fraction(1.0))
                .with_height(Constraint::Auto),
        }
    }

    // Builder methods delegate to inner...
    pub fn with_id(mut self, id: impl Into<String>) -> Self {
        self.inner = self.inner.with_id(id);
        self
    }
}

impl Widget for Center {
    fn layout(&self) -> LayoutHints {
        self.inner.layout()
            .with_alignment(Alignment {
                horizontal: AlignH::Center,
                vertical: AlignV::Top,
            })
    }

    fn layout_type(&self) -> Option<LayoutType> {
        Some(LayoutType::Vertical)  // Stack children vertically, then center
    }
    // Other methods delegate to inner...
}

/// Vertically centers children.
pub struct Middle {
    inner: Container,
}

impl Middle {
    pub fn new() -> Self {
        Self {
            inner: Container::new()
                .with_width(Constraint::Auto)
                .with_height(Constraint::Fraction(1.0)),
        }
    }
}

impl Widget for Middle {
    fn layout(&self) -> LayoutHints {
        self.inner.layout()
            .with_alignment(Alignment {
                horizontal: AlignH::Left,
                vertical: AlignV::Middle,
            })
    }

    fn layout_type(&self) -> Option<LayoutType> {
        Some(LayoutType::Vertical)
    }
}

/// Centers children both horizontally and vertically.
pub struct CenterMiddle {
    inner: Container,
}

impl CenterMiddle {
    pub fn new() -> Self {
        Self {
            inner: Container::new()
                .with_width(Constraint::Fraction(1.0))
                .with_height(Constraint::Fraction(1.0)),
        }
    }
}

impl Widget for CenterMiddle {
    fn layout(&self) -> LayoutHints {
        self.inner.layout()
            .with_alignment(Alignment {
                horizontal: AlignH::Center,
                vertical: AlignV::Middle,
            })
    }

    fn layout_type(&self) -> Option<LayoutType> {
        Some(LayoutType::Vertical)
    }
}

/// Right-aligns children.
pub struct Right {
    inner: Container,
}

impl Right {
    pub fn new() -> Self {
        Self {
            inner: Container::new()
                .with_width(Constraint::Fraction(1.0))
                .with_height(Constraint::Auto),
        }
    }
}

impl Widget for Right {
    fn layout(&self) -> LayoutHints {
        self.inner.layout()
            .with_alignment(Alignment {
                horizontal: AlignH::Right,
                vertical: AlignV::Top,
            })
    }

    fn layout_type(&self) -> Option<LayoutType> {
        Some(LayoutType::Vertical)
    }
}
```

#### 4.2 Group Containers

```rust
/// Vertical layout that shrinks to content height.
pub struct VerticalGroup {
    inner: Container,
}

impl VerticalGroup {
    pub fn new() -> Self {
        Self {
            inner: Container::new()
                .with_width(Constraint::Fraction(1.0))
                .with_height(Constraint::Auto),
        }
    }
}

impl Widget for VerticalGroup {
    fn layout_type(&self) -> Option<LayoutType> {
        Some(LayoutType::Vertical)
    }

    fn layout(&self) -> LayoutHints {
        self.inner.layout()
            .with_overflow(OverflowSettings::both(Overflow::Hidden))
    }
}

/// Horizontal layout that shrinks to content height.
pub struct HorizontalGroup {
    inner: Container,
}

impl HorizontalGroup {
    pub fn new() -> Self {
        Self {
            inner: Container::new()
                .with_width(Constraint::Fraction(1.0))
                .with_height(Constraint::Auto),
        }
    }
}

impl Widget for HorizontalGroup {
    fn layout_type(&self) -> Option<LayoutType> {
        Some(LayoutType::Horizontal)
    }

    fn layout(&self) -> LayoutHints {
        self.inner.layout()
            .with_overflow(OverflowSettings::both(Overflow::Hidden))
    }
}
```

#### 4.3 ItemGrid

ItemGrid must override `layout_type()` to return Grid and expose its config. **Critical**: The cached config must be updated whenever reactive properties change, so `grid_config()` returns the up-to-date computed value:

```rust
/// Grid with dynamic column configuration.
pub struct ItemGrid {
    inner: Container,
    /// Cached computed config - updated when reactive properties change.
    cached_config: GridConfig,
    stretch_height: bool,
    min_column_width: Option<u16>,
    max_column_width: Option<u16>,
    regular: bool,
}

impl ItemGrid {
    pub fn new() -> Self {
        Self {
            inner: Container::new()
                .with_width(Constraint::Fraction(1.0))
                .with_height(Constraint::Auto),
            cached_config: GridConfig::default(),
            stretch_height: false,
            min_column_width: None,
            max_column_width: None,
            regular: false,
        }
    }

    /// Recompute cached_config from reactive properties.
    fn update_cached_config(&mut self) {
        let mut config = GridConfig::default();

        if let Some(min) = self.min_column_width {
            config = config.with_min_column_width(min);
        }
        if let Some(max) = self.max_column_width {
            config = config.with_max_column_width(max);
        }
        if self.regular {
            config = config.with_regular(true);
        }
        if self.stretch_height {
            config = config.with_stretch_height(true);
        }

        self.cached_config = config;
    }

    pub fn with_stretch_height(mut self, stretch: bool) -> Self {
        self.stretch_height = stretch;
        self.update_cached_config();
        self
    }

    pub fn with_min_column_width(mut self, width: u16) -> Self {
        self.min_column_width = Some(width);
        self.update_cached_config();
        self
    }

    pub fn with_max_column_width(mut self, width: u16) -> Self {
        self.max_column_width = Some(width);
        self.update_cached_config();
        self
    }

    pub fn with_regular(mut self, regular: bool) -> Self {
        self.regular = regular;
        self.update_cached_config();
        self
    }

    /// Setters for runtime property changes (reactive).
    pub fn set_stretch_height(&mut self, stretch: bool) {
        self.stretch_height = stretch;
        self.update_cached_config();
    }

    pub fn set_min_column_width(&mut self, width: Option<u16>) {
        self.min_column_width = width;
        self.update_cached_config();
    }

    pub fn set_max_column_width(&mut self, width: Option<u16>) {
        self.max_column_width = width;
        self.update_cached_config();
    }

    pub fn set_regular(&mut self, regular: bool) {
        self.regular = regular;
        self.update_cached_config();
    }
}

impl Widget for ItemGrid {
    fn layout_type(&self) -> Option<LayoutType> {
        Some(LayoutType::Grid)  // REQUIRED: Tells layout system to use GridLayout
    }

    fn layout(&self) -> LayoutHints {
        self.inner.layout()
    }

    /// Provide grid config to layout system.
    /// Returns the cached computed config (updated by reactive property setters).
    fn grid_config(&self) -> Option<&GridConfig> {
        Some(&self.cached_config)
    }

    fn widget_type_name(&self) -> &'static str {
        "ItemGrid"
    }
}
```

### 5. ScrollableContainer Design

ScrollableContainer wraps ScrollView and adds keyboard bindings:

```rust
// src/widgets/scroll.rs (additions)

/// Scrollable container with keyboard navigation.
pub struct ScrollableContainer {
    view: ScrollView,
    bindings: BindingManager,
}

impl ScrollableContainer {
    pub fn new() -> Self {
        let mut bindings = BindingManager::new();
        bindings.add_all([
            Binding::new("up", "scroll_up").with_show(false),
            Binding::new("down", "scroll_down").with_show(false),
            Binding::new("left", "scroll_left").with_show(false),
            Binding::new("right", "scroll_right").with_show(false),
            Binding::new("home", "scroll_home").with_show(false),
            Binding::new("end", "scroll_end").with_show(false),
            Binding::new("pageup", "page_up").with_show(false),
            Binding::new("pagedown", "page_down").with_show(false),
            Binding::new("ctrl+pageup", "page_left").with_show(false),
            Binding::new("ctrl+pagedown", "page_right").with_show(false),
        ]);

        Self {
            view: ScrollView::new(),
            bindings,
        }
    }
}

impl Widget for ScrollableContainer {
    fn focusable(&self) -> bool {
        true  // Can receive focus for keyboard input
    }

    // Delegate other methods to view...
}

impl ActionHandler for ScrollableContainer {
    fn run_action(&mut self, action: &ParsedAction) -> Result<bool, ActionError> {
        match action.name.as_str() {
            "scroll_up" => { self.view.scroll_mut().scroll_up(); Ok(true) }
            "scroll_down" => { self.view.scroll_mut().scroll_down(); Ok(true) }
            "scroll_left" => { self.view.scroll_mut().scroll_left(); Ok(true) }
            "scroll_right" => { self.view.scroll_mut().scroll_right(); Ok(true) }
            "scroll_home" => { self.view.scroll_home(); Ok(true) }
            "scroll_end" => { self.view.scroll_end(); Ok(true) }
            "page_up" => { self.view.scroll_mut().page_up(); Ok(true) }
            "page_down" => { self.view.scroll_mut().page_down(); Ok(true) }
            "page_left" => { self.view.scroll_mut().page_left(); Ok(true) }
            "page_right" => { self.view.scroll_mut().page_right(); Ok(true) }
            _ => Ok(false)
        }
    }

    fn bindings(&self) -> &[Binding] {
        self.bindings.bindings()
    }
}
```

#### 5.1 VerticalScroll / HorizontalScroll

```rust
/// Vertical-only scrolling container.
pub struct VerticalScroll {
    inner: ScrollableContainer,
}

impl VerticalScroll {
    pub fn new() -> Self {
        Self {
            inner: ScrollableContainer::new()
                .with_overflow(OverflowSettings::vertical()),
        }
    }
}

/// Horizontal-only scrolling container.
pub struct HorizontalScroll {
    inner: ScrollableContainer,
}

impl HorizontalScroll {
    pub fn new() -> Self {
        Self {
            inner: ScrollableContainer::new()
                .with_overflow(OverflowSettings::horizontal()),
        }
    }
}
```

### 6. ContentSwitcher Design

ContentSwitcher uses `Display::None` to hide non-current children:

```rust
// src/widgets/content_switcher.rs

use std::collections::HashMap;

/// Container that shows one child at a time.
/// Hidden children have display: none and are skipped in layout.
pub struct ContentSwitcher {
    inner: Container,
    /// Currently visible child ID (required, non-empty).
    current: Option<String>,
}

/// Message posted when current changes.
pub struct CurrentChanged {
    pub switcher_id: Option<String>,
}

impl ContentSwitcher {
    pub fn new() -> Self {
        Self {
            inner: Container::new().with_height(Constraint::Auto),
            current: None,
        }
    }

    pub fn with_initial(mut self, id: impl Into<String>) -> Self {
        let id = id.into();
        if !id.is_empty() {
            self.current = Some(id);
        }
        self
    }

    /// Get currently visible child ID.
    pub fn current(&self) -> Option<&str> {
        self.current.as_deref()
    }

    /// Set currently visible child.
    /// Panics if id is empty string.
    pub fn set_current(&mut self, id: Option<String>) -> CurrentChanged {
        if let Some(ref id) = id {
            assert!(!id.is_empty(), "ContentSwitcher child ID cannot be empty");
        }
        self.current = id;

        CurrentChanged {
            switcher_id: self.inner.id().map(|s| s.to_string()),
        }
    }

    /// Compute display property for a child widget by ID.
    /// Returns Display::Block for current child, Display::None for others.
    pub fn child_display(&self, child_id: &str) -> Display {
        if self.current.as_deref() == Some(child_id) {
            Display::Block
        } else {
            Display::None
        }
    }

    /// Add content with ID.
    /// Panics if id is empty.
    pub fn add_content(&mut self, id: impl Into<String>, set_current: bool) {
        let id = id.into();
        assert!(!id.is_empty(), "ContentSwitcher child ID cannot be empty");

        if set_current || self.current.is_none() {
            self.current = Some(id);
        }
    }
}

impl Widget for ContentSwitcher {
    fn layout(&self) -> LayoutHints {
        self.inner.layout()
    }

    fn layout_type(&self) -> Option<LayoutType> {
        Some(LayoutType::Vertical)
    }

    /// Render only the current (visible) child.
    /// Children with display: none are skipped.
    fn render(&self, area: Rect, frame: &mut Frame) {
        // Rendering is handled by the layout system which filters
        // display: none children. The ContentSwitcher delegates
        // rendering to its inner container which will only render
        // visible children.
        self.inner.render(area, frame);
    }

    /// Check if this widget overrides child display (for layout filtering).
    fn as_content_switcher(&self) -> Option<&ContentSwitcher> {
        Some(self)
    }

    fn widget_type_name(&self) -> &'static str {
        "ContentSwitcher"
    }
}
```

### 7. Tab Widgets Design

#### 7.1 ID Prefix System

TabbedContent uses a prefix system to coordinate Tab and TabPane IDs:

```rust
/// Prefix for content tab IDs (internal).
const CONTENT_TAB_PREFIX: &str = "--content-tab-";

/// Add prefix to create internal tab ID from pane ID.
fn add_tab_prefix(pane_id: &str) -> String {
    format!("{}{}", CONTENT_TAB_PREFIX, pane_id)
}

/// Remove prefix to get pane ID from internal tab ID.
fn remove_tab_prefix(tab_id: &str) -> Option<&str> {
    tab_id.strip_prefix(CONTENT_TAB_PREFIX)
}
```

#### 7.2 Tab Widget

```rust
// src/widgets/tabs.rs

/// A single tab in a tab bar.
pub struct Tab {
    id: String,
    label: String,
    disabled: bool,
    hidden: bool,
}

/// Tab clicked message.
pub struct TabClicked {
    pub tab_id: String,
}

/// Tab disabled state changed.
pub struct TabDisabled {
    pub tab_id: String,
}

pub struct TabEnabled {
    pub tab_id: String,
}

impl Tab {
    /// Create a new tab.
    /// Panics if id is empty.
    pub fn new(id: impl Into<String>, label: impl Into<String>) -> Self {
        let id = id.into();
        assert!(!id.is_empty(), "Tab ID cannot be empty");
        Self {
            id,
            label: label.into(),
            disabled: false,
            hidden: false,
        }
    }

    pub fn with_disabled(mut self, disabled: bool) -> Self {
        self.disabled = disabled;
        self
    }

    pub fn id(&self) -> &str {
        &self.id
    }

    pub fn label(&self) -> &str {
        &self.label
    }

    pub fn is_disabled(&self) -> bool {
        self.disabled
    }

    pub fn is_hidden(&self) -> bool {
        self.hidden
    }

    /// Compute display property based on hidden state.
    pub fn display(&self) -> Display {
        if self.hidden {
            Display::None
        } else {
            Display::Block
        }
    }
}

impl Widget for Tab {
    fn render(&self, area: Rect, frame: &mut Frame) {
        let style = if self.disabled {
            Style::default().fg(Color::DarkGray)  // 25% opacity equivalent
        } else {
            Style::default()  // Full opacity when active/hover
        };

        let span = Span::styled(&self.label, style);
        let paragraph = Paragraph::new(span)
            .alignment(ratatui::layout::Alignment::Center);
        frame.render_widget(paragraph, area);
    }

    fn layout(&self) -> LayoutHints {
        LayoutHints::default()
            .with_width(Constraint::Auto)
            .with_height(Constraint::Length(1))
            .with_display(self.display())
    }

    fn get_content_width(&self) -> u16 {
        self.label.len() as u16
    }

    fn get_content_height(&self) -> u16 {
        1
    }

    fn widget_type_name(&self) -> &'static str {
        "Tab"
    }
}
```

#### 7.3 Tabs Widget (Tab Bar)

```rust
/// Tab bar containing multiple tabs.
/// Composed of: Horizontal container of Tabs + Underline widget.
pub struct Tabs {
    tabs: Vec<Tab>,
    underline: Underline,
    active: Option<String>,
    bindings: BindingManager,
}

/// Tab activated message.
pub struct TabActivated {
    pub tab_id: String,
    pub previous: Option<String>,
}

/// All tabs cleared message.
pub struct Cleared;

impl Tabs {
    pub fn new() -> Self {
        let mut bindings = BindingManager::new();
        bindings.add_all([
            Binding::new("left", "previous_tab").with_show(false),
            Binding::new("right", "next_tab").with_show(false),
        ]);

        Self {
            tabs: Vec::new(),
            underline: Underline::new(),
            active: None,
            bindings,
        }
    }

    // Note: Tabs internally manages Tab widgets and renders them directly.
    // It does NOT use the Compose trait because tabs are dynamically added/removed
    // and stored within the Tabs struct itself.
    //
    // The layout system treats Tabs as a leaf widget that renders its own
    // internal structure (tabs row + underline) in its render() method.

    /// Add a tab.
    pub fn add_tab(&mut self, tab: Tab) {
        let id = tab.id().to_string();
        self.tabs.push(tab);

        // Auto-activate first tab
        if self.active.is_none() {
            self.active = Some(id);
        }
    }

    /// Remove a tab.
    pub fn remove_tab(&mut self, id: &str) -> Option<Tab> {
        if let Some(pos) = self.tabs.iter().position(|t| t.id() == id) {
            let tab = self.tabs.remove(pos);

            // If removed tab was active, activate next available
            if self.active.as_deref() == Some(id) {
                self.active = self.find_next_active_tab(pos);
            }

            Some(tab)
        } else {
            None
        }
    }

    /// Clear all tabs.
    pub fn clear(&mut self) -> Cleared {
        self.tabs.clear();
        self.active = None;
        Cleared
    }

    /// Get active tab ID.
    pub fn active(&self) -> Option<&str> {
        self.active.as_deref()
    }

    /// Set active tab.
    pub fn set_active(&mut self, id: impl Into<String>) -> Option<TabActivated> {
        let id = id.into();

        if self.tabs.iter().any(|t| t.id() == id && !t.is_disabled() && !t.is_hidden()) {
            let previous = self.active.take();
            self.active = Some(id.clone());

            // Update underline position to highlight active tab
            self.update_underline_position();

            Some(TabActivated {
                tab_id: id,
                previous,
            })
        } else {
            None
        }
    }

    /// Update underline highlight position based on active tab.
    fn update_underline_position(&mut self) {
        let Some(active_id) = &self.active else {
            self.underline.hide_highlight();
            return;
        };

        // Calculate position by summing widths of tabs before active
        let mut x = 0i16;
        for tab in &self.tabs {
            if tab.display() == Display::None {
                continue;
            }

            let tab_width = (tab.get_content_width() + 2) as i16;  // +2 for padding

            if tab.id() == active_id {
                self.underline.set_highlight(x, x + tab_width);
                return;
            }

            x += tab_width;
        }

        // Active tab not found
        self.underline.hide_highlight();
    }

    /// Activate previous tab (with wrap-around).
    pub fn previous_tab(&mut self) -> Option<TabActivated> {
        let current_idx = self.active_index()?;
        let next_idx = self.find_prev_enabled_tab(current_idx)?;
        let id = self.tabs[next_idx].id().to_string();
        self.set_active(id)
    }

    /// Activate next tab (with wrap-around).
    pub fn next_tab(&mut self) -> Option<TabActivated> {
        let current_idx = self.active_index()?;
        let next_idx = self.find_next_enabled_tab(current_idx)?;
        let id = self.tabs[next_idx].id().to_string();
        self.set_active(id)
    }

    fn active_index(&self) -> Option<usize> {
        self.active.as_ref().and_then(|id| {
            self.tabs.iter().position(|t| t.id() == id)
        })
    }

    fn find_next_enabled_tab(&self, from: usize) -> Option<usize> {
        let len = self.tabs.len();
        for i in 1..=len {
            let idx = (from + i) % len;
            let tab = &self.tabs[idx];
            if !tab.is_disabled() && !tab.is_hidden() {
                return Some(idx);
            }
        }
        None
    }

    fn find_prev_enabled_tab(&self, from: usize) -> Option<usize> {
        let len = self.tabs.len();
        for i in 1..=len {
            let idx = (from + len - i) % len;
            let tab = &self.tabs[idx];
            if !tab.is_disabled() && !tab.is_hidden() {
                return Some(idx);
            }
        }
        None
    }

    fn find_next_active_tab(&self, from: usize) -> Option<String> {
        self.find_next_enabled_tab(from)
            .map(|idx| self.tabs[idx].id().to_string())
    }

    /// Disable a tab.
    pub fn disable(&mut self, id: &str) -> Option<TabDisabled> {
        if let Some(tab) = self.tabs.iter_mut().find(|t| t.id() == id) {
            tab.disabled = true;
            Some(TabDisabled { tab_id: id.to_string() })
        } else {
            None
        }
    }

    /// Enable a tab.
    pub fn enable(&mut self, id: &str) -> Option<TabEnabled> {
        if let Some(tab) = self.tabs.iter_mut().find(|t| t.id() == id) {
            tab.disabled = false;
            Some(TabEnabled { tab_id: id.to_string() })
        } else {
            None
        }
    }

    /// Hide a tab (sets display: none).
    pub fn hide(&mut self, id: &str) {
        if let Some(tab) = self.tabs.iter_mut().find(|t| t.id() == id) {
            tab.hidden = true;
        }
    }

    /// Show a tab.
    pub fn show(&mut self, id: &str) {
        if let Some(tab) = self.tabs.iter_mut().find(|t| t.id() == id) {
            tab.hidden = false;
        }
    }
}

impl Widget for Tabs {
    fn focusable(&self) -> bool {
        true
    }

    fn layout(&self) -> LayoutHints {
        LayoutHints::default()
            .with_width(Constraint::Fraction(1.0))
            .with_height(Constraint::Length(2))  // 1 for tabs, 1 for underline
    }

    fn render(&self, area: Rect, frame: &mut Frame) {
        // Split area: tabs row (height 1) + underline (height 1)
        let tabs_area = Rect::new(area.x, area.y, area.width, 1);
        let underline_area = Rect::new(area.x, area.y + 1, area.width, 1);

        // Render tabs horizontally
        let mut x = area.x;
        for tab in &self.tabs {
            if tab.display() == Display::None {
                continue;
            }

            let tab_width = tab.get_content_width() + 2;  // +2 for padding
            let tab_area = Rect::new(x, tabs_area.y, tab_width, 1);

            // Apply active styling
            let is_active = self.active.as_deref() == Some(tab.id());
            if is_active {
                // Update underline position
                // (In practice, this would be done in set_active)
            }

            tab.render(tab_area, frame);
            x += tab_width;
        }

        // Render underline with highlight under active tab
        self.underline.render(underline_area, frame);
    }

    fn widget_type_name(&self) -> &'static str {
        "Tabs"
    }
}

impl ActionHandler for Tabs {
    fn run_action(&mut self, action: &ParsedAction) -> Result<bool, ActionError> {
        match action.name.as_str() {
            "previous_tab" => {
                self.previous_tab();
                Ok(true)
            }
            "next_tab" => {
                self.next_tab();
                Ok(true)
            }
            _ => Ok(false)
        }
    }

    fn bindings(&self) -> &[Binding] {
        self.bindings.bindings()
    }
}
```

#### 7.4 Underline Widget

```rust
/// Animated underline indicator for tabs.
pub struct Underline {
    highlight_start: i16,
    highlight_end: i16,
    show_highlight: bool,
}

/// Underline clicked message.
pub struct UnderlineClicked {
    pub offset: i16,
}

impl Underline {
    pub fn new() -> Self {
        Self {
            highlight_start: 0,
            highlight_end: 0,
            show_highlight: false,
        }
    }

    /// Update highlight position from active tab region.
    pub fn set_highlight(&mut self, start: i16, end: i16) {
        self.highlight_start = start;
        self.highlight_end = end;
        self.show_highlight = true;
    }

    /// Hide the highlight.
    pub fn hide_highlight(&mut self) {
        self.show_highlight = false;
    }
}

impl Widget for Underline {
    fn render(&self, area: Rect, frame: &mut Frame) {
        // Render bar with highlight
        for x in 0..area.width {
            let in_highlight = self.show_highlight
                && x as i16 >= self.highlight_start
                && x as i16 < self.highlight_end;

            let (char, style) = if in_highlight {
                ('━', Style::default().fg(Color::White))
            } else {
                ('─', Style::default().fg(Color::DarkGray))
            };

            let span = Span::styled(char.to_string(), style);
            frame.render_widget(
                Paragraph::new(span),
                Rect::new(area.x + x, area.y, 1, 1)
            );
        }
    }

    fn layout(&self) -> LayoutHints {
        LayoutHints::default()
            .with_width(Constraint::Fraction(1.0))
            .with_height(Constraint::Length(1))
    }

    fn widget_type_name(&self) -> &'static str {
        "Underline"
    }
}
```

#### 7.5 TabPane Widget

```rust
/// A pane of content associated with a tab.
pub struct TabPane {
    inner: Container,
    id: String,  // Required, non-empty
    title: String,
    disabled: bool,
}

/// TabPane disabled message.
pub struct TabPaneDisabled {
    pub pane_id: String,
}

/// TabPane enabled message.
pub struct TabPaneEnabled {
    pub pane_id: String,
}

/// TabPane focused (descendant received focus).
pub struct TabPaneFocused {
    pub pane_id: String,
}

impl TabPane {
    /// Create a new TabPane.
    /// Panics if id is empty.
    pub fn new(id: impl Into<String>, title: impl Into<String>) -> Self {
        let id = id.into();
        assert!(!id.is_empty(), "TabPane ID cannot be empty");
        Self {
            inner: Container::new()
                .with_id(&id)
                .with_height(Constraint::Auto),
            id,
            title: title.into(),
            disabled: false,
        }
    }

    pub fn id(&self) -> &str {
        &self.id
    }

    pub fn title(&self) -> &str {
        &self.title
    }

    pub fn set_title(&mut self, title: impl Into<String>) {
        self.title = title.into();
    }

    pub fn is_disabled(&self) -> bool {
        self.disabled
    }

    pub fn set_disabled(&mut self, disabled: bool) {
        self.disabled = disabled;
    }
}

impl Widget for TabPane {
    fn render(&self, area: Rect, frame: &mut Frame) {
        self.inner.render(area, frame);
    }

    fn layout(&self) -> LayoutHints {
        self.inner.layout()
    }

    fn widget_type_name(&self) -> &'static str {
        "TabPane"
    }
}
```

#### 7.6 TabbedContent Widget

```rust
/// Prefix for content tab IDs.
const CONTENT_TAB_PREFIX: &str = "--content-tab-";

/// Container with tabs and switchable content panes.
pub struct TabbedContent {
    id: Option<String>,
    tabs: Tabs,
    switcher: ContentSwitcher,
}

impl TabbedContent {
    pub fn new() -> Self {
        Self {
            id: None,
            tabs: Tabs::new(),
            switcher: ContentSwitcher::new(),
        }
    }

    pub fn with_id(mut self, id: impl Into<String>) -> Self {
        self.id = Some(id.into());
        self
    }

    /// Get active pane ID (without prefix).
    pub fn active(&self) -> Option<&str> {
        self.tabs.active().and_then(|tab_id| {
            tab_id.strip_prefix(CONTENT_TAB_PREFIX)
        })
    }

    /// Set active pane by pane ID (not tab ID).
    pub fn set_active(&mut self, pane_id: impl Into<String>) -> Option<TabActivated> {
        let pane_id = pane_id.into();
        let tab_id = format!("{}{}", CONTENT_TAB_PREFIX, pane_id);

        let result = self.tabs.set_active(&tab_id);
        if result.is_some() {
            self.switcher.set_current(Some(pane_id));
        }
        result
    }

    /// Add a pane.
    /// Panics if pane has empty ID.
    pub fn add_pane(&mut self, pane: TabPane) {
        let pane_id = pane.id().to_string();
        let tab_id = format!("{}{}", CONTENT_TAB_PREFIX, pane_id);

        let tab = Tab::new(&tab_id, pane.title())
            .with_disabled(pane.is_disabled());

        self.tabs.add_tab(tab);
        self.switcher.add_content(&pane_id, false);
    }

    /// Remove a pane by pane ID.
    pub fn remove_pane(&mut self, pane_id: &str) {
        let tab_id = format!("{}{}", CONTENT_TAB_PREFIX, pane_id);
        self.tabs.remove_tab(&tab_id);
    }

    /// Clear all panes.
    pub fn clear_panes(&mut self) {
        self.tabs.clear();
        self.switcher.set_current(None);
    }

    /// Get a tab by pane ID (uses prefix internally).
    pub fn get_tab(&self, pane_id: &str) -> Option<&Tab> {
        let tab_id = format!("{}{}", CONTENT_TAB_PREFIX, pane_id);
        self.tabs.tabs.iter().find(|t| t.id() == tab_id)
    }

    /// Disable a tab by pane ID.
    pub fn disable_tab(&mut self, pane_id: &str) {
        let tab_id = format!("{}{}", CONTENT_TAB_PREFIX, pane_id);
        self.tabs.disable(&tab_id);
    }

    /// Enable a tab by pane ID.
    pub fn enable_tab(&mut self, pane_id: &str) {
        let tab_id = format!("{}{}", CONTENT_TAB_PREFIX, pane_id);
        self.tabs.enable(&tab_id);
    }

    /// Hide a tab by pane ID.
    pub fn hide_tab(&mut self, pane_id: &str) {
        let tab_id = format!("{}{}", CONTENT_TAB_PREFIX, pane_id);
        self.tabs.hide(&tab_id);
    }

    /// Show a tab by pane ID.
    pub fn show_tab(&mut self, pane_id: &str) {
        let tab_id = format!("{}{}", CONTENT_TAB_PREFIX, pane_id);
        self.tabs.show(&tab_id);
    }
}

impl Widget for TabbedContent {
    fn layout(&self) -> LayoutHints {
        LayoutHints::default()
            .with_height(Constraint::Auto)
    }

    fn layout_type(&self) -> Option<LayoutType> {
        Some(LayoutType::Vertical)
    }

    fn widget_type_name(&self) -> &'static str {
        "TabbedContent"
    }
}

/// TabbedContent uses the Compose trait to build its child tree.
/// Structure: Tabs (docked top) + ContentSwitcher
impl Compose for TabbedContent {
    fn compose(&self, ctx: &mut ComposeContext) {
        // Add Tabs widget (with dock hint for top positioning)
        let tabs = self.tabs.clone();
        ctx.add(tabs);

        // Add ContentSwitcher that manages pane visibility
        // TabPanes are added as children of the switcher
        ctx.add(self.switcher.clone());
    }
}

// Note: TabbedContent uses layout_type() = Vertical to stack children.
// The Tabs widget renders at its natural height (2 rows).
// ContentSwitcher fills remaining space and shows only the active pane.
//
// Rendering is handled by the framework's layout/render pass, not manually.
// Each child widget's render() is called with its computed area.

// Message handling for tab activation
impl TabbedContent {
    /// Handle TabActivated message from Tabs.
    /// Syncs ContentSwitcher to show corresponding pane.
    pub fn on_tab_activated(&mut self, msg: &TabActivated) -> Option<TabActivated> {
        // Extract pane ID from tab ID (remove prefix)
        if let Some(pane_id) = msg.tab_id.strip_prefix(CONTENT_TAB_PREFIX) {
            self.switcher.set_current(Some(pane_id.to_string()));

            // Forward the message with the pane ID
            Some(TabActivated {
                tab_id: pane_id.to_string(),
                previous: msg.previous.as_ref().and_then(|p| {
                    p.strip_prefix(CONTENT_TAB_PREFIX).map(|s| s.to_string())
                }),
            })
        } else {
            None
        }
    }
}
```

### 8. Collapsible Widget Design

Based on research: Python Textual's Collapsible has **no animation** - it uses instant `display: none` toggle.

Collapsible is composed of two internal widgets:
- **CollapsibleTitle** - Focusable header with toggle keyboard support
- **Contents** - Container for child content (hidden when collapsed)

```rust
// src/widgets/collapsible.rs

/// Focusable collapsible header.
pub struct CollapsibleTitle {
    title: String,
    collapsed: bool,
    bindings: BindingManager,
}

impl CollapsibleTitle {
    pub fn new(title: impl Into<String>) -> Self {
        let mut bindings = BindingManager::new();
        bindings.add(Binding::new("enter", "toggle").with_show(false));

        Self {
            title: title.into(),
            collapsed: false,
            bindings,
        }
    }

    pub fn set_collapsed(&mut self, collapsed: bool) {
        self.collapsed = collapsed;
    }

    fn render_arrow(&self) -> &'static str {
        if self.collapsed { "▶" } else { "▼" }
    }
}

impl Widget for CollapsibleTitle {
    fn focusable(&self) -> bool {
        true
    }

    fn render(&self, area: Rect, frame: &mut Frame) {
        let text = format!("{} {}", self.render_arrow(), self.title);
        let paragraph = Paragraph::new(text);
        frame.render_widget(paragraph, area);
    }

    fn layout(&self) -> LayoutHints {
        LayoutHints::default()
            .with_width(Constraint::Auto)
            .with_height(Constraint::Auto)
    }

    fn get_content_width(&self) -> u16 {
        // Arrow (2 chars) + space + title
        (2 + 1 + self.title.len()) as u16
    }

    fn get_content_height(&self) -> u16 {
        1  // Single line
    }

    fn widget_type_name(&self) -> &'static str {
        "CollapsibleTitle"
    }
}

impl ActionHandler for CollapsibleTitle {
    fn run_action(&mut self, action: &ParsedAction) -> Result<bool, ActionError> {
        match action.name.as_str() {
            "toggle" => Ok(true),  // Signal to parent Collapsible
            _ => Ok(false)
        }
    }

    fn bindings(&self) -> &[Binding] {
        self.bindings.bindings()
    }
}

/// Container for collapsible content.
/// Has display: none when parent is collapsed.
pub struct Contents {
    inner: Container,
    display: Display,
}

impl Contents {
    pub fn new() -> Self {
        Self {
            inner: Container::new()
                .with_width(Constraint::Fraction(1.0))
                .with_height(Constraint::Auto),
            display: Display::Block,
        }
    }

    pub fn set_visible(&mut self, visible: bool) {
        self.display = if visible { Display::Block } else { Display::None };
    }
}

impl Widget for Contents {
    fn layout(&self) -> LayoutHints {
        self.inner.layout().with_display(self.display)
    }

    fn layout_type(&self) -> Option<LayoutType> {
        Some(LayoutType::Vertical)
    }

    fn get_content_width(&self) -> u16 {
        self.inner.get_content_width()
    }

    fn get_content_height(&self) -> u16 {
        if self.display == Display::None {
            0  // Hidden, takes no space
        } else {
            self.inner.get_content_height()
        }
    }

    fn widget_type_name(&self) -> &'static str {
        "Contents"
    }
}

/// Collapsible container with toggle header.
/// Composed of CollapsibleTitle + Contents.
pub struct Collapsible {
    id: Option<String>,
    title: CollapsibleTitle,
    contents: Contents,
    collapsed: bool,  // Default: true (starts collapsed)
}

/// Collapsible toggled message.
pub struct Toggled {
    pub collapsible_id: Option<String>,
    pub collapsed: bool,
}

impl Collapsible {
    pub fn new(title: impl Into<String>) -> Self {
        Self {
            id: None,
            title: CollapsibleTitle::new(title),
            contents: Contents::new(),
            collapsed: true,  // Default collapsed per Python Textual
        }
    }

    pub fn with_id(mut self, id: impl Into<String>) -> Self {
        self.id = Some(id.into());
        self
    }

    /// Set initial collapsed state (default: true).
    pub fn with_collapsed(mut self, collapsed: bool) -> Self {
        self.collapsed = collapsed;
        self.sync_state();
        self
    }

    fn sync_state(&mut self) {
        self.title.set_collapsed(self.collapsed);
        self.contents.set_visible(!self.collapsed);
    }

    pub fn title(&self) -> &str {
        &self.title.title
    }

    pub fn is_collapsed(&self) -> bool {
        self.collapsed
    }

    /// Expand the collapsible.
    pub fn expand(&mut self) -> Option<Toggled> {
        if self.collapsed {
            self.collapsed = false;
            self.sync_state();
            Some(Toggled {
                collapsible_id: self.id.clone(),
                collapsed: false,
            })
        } else {
            None
        }
    }

    /// Collapse the collapsible.
    pub fn collapse(&mut self) -> Option<Toggled> {
        if !self.collapsed {
            self.collapsed = true;
            self.sync_state();
            Some(Toggled {
                collapsible_id: self.id.clone(),
                collapsed: true,
            })
        } else {
            None
        }
    }

    /// Toggle collapsed state.
    pub fn toggle(&mut self) -> Toggled {
        self.collapsed = !self.collapsed;
        self.sync_state();
        Toggled {
            collapsible_id: self.id.clone(),
            collapsed: self.collapsed,
        }
    }
}

impl Widget for Collapsible {
    fn id(&self) -> Option<&str> {
        self.id.as_deref()
    }

    fn layout(&self) -> LayoutHints {
        LayoutHints::default()
            .with_width(Constraint::Fraction(1.0))
            .with_height(Constraint::Auto)
    }

    fn layout_type(&self) -> Option<LayoutType> {
        Some(LayoutType::Vertical)
    }

    fn get_content_height(&self) -> u16 {
        let title_height = self.title.get_content_height();
        if self.collapsed {
            title_height
        } else {
            title_height + self.contents.get_content_height()
        }
    }

    fn widget_type_name(&self) -> &'static str {
        "Collapsible"
    }
}

/// Collapsible uses the Compose trait to build its child tree.
/// Structure: CollapsibleTitle + Contents (vertical layout)
impl Compose for Collapsible {
    fn compose(&self, ctx: &mut ComposeContext) {
        // Add title widget (always visible, focusable)
        ctx.add(self.title.clone());

        // Add contents widget (has display: none when collapsed)
        // The Contents widget handles its own display state via layout()
        ctx.add(self.contents.clone());
    }
}

// Note: Collapsible uses layout_type() = Vertical to stack children.
// CollapsibleTitle renders the arrow and title text.
// Contents returns display: none in layout() when collapsed,
// so the layout system automatically skips it.

// Message flow for toggle action
impl Collapsible {
    /// Handle toggle action from CollapsibleTitle.
    /// Called when Enter is pressed on the focused title.
    pub fn on_title_toggle(&mut self) -> Toggled {
        self.toggle()
    }
}
```

### 9. Widget Trait Extension Summary

All Widget trait extensions are defined in **Section 2.1**. This section summarizes the new APIs:

| Method | Return Type | Purpose |
|--------|-------------|---------|
| `id()` | `Option<&str>` | Widget ID for lookups and ContentSwitcher coordination |
| `get_content_width()` | `u16` | Content width for `Constraint::Auto` sizing |
| `get_content_height()` | `u16` | Content height for `Constraint::Auto` sizing |
| `grid_config()` | `Option<&GridConfig>` | Grid configuration for ItemGrid |

All methods have default implementations returning `None` or `0`, making them opt-in.

**Implementation Requirements**:
- Widgets with IDs must implement `id()` (ContentSwitcher, TabbedContent panes)
- Widgets using `Constraint::Auto` should implement `get_content_*()` methods
- ItemGrid must implement `grid_config()` to return its computed configuration

### 10. File Organization

```
src/widgets/
├── mod.rs              # Widget exports
├── container.rs        # Container, Horizontal, Vertical, Grid (existing)
│                       # + Center, Middle, Right, CenterMiddle
│                       # + VerticalGroup, HorizontalGroup
│                       # + ItemGrid
├── scroll.rs           # ScrollState, ScrollBar, ScrollView (existing)
│                       # + ScrollableContainer, VerticalScroll, HorizontalScroll
├── content_switcher.rs # ContentSwitcher (NEW)
├── tabs.rs             # Tab, Tabs, Underline, TabPane, TabbedContent (NEW)
└── collapsible.rs      # CollapsibleTitle, Contents, Collapsible (NEW)
```

### 11. Dependencies Between Widgets

```
Container (base)
├── Horizontal, Vertical, Grid (existing)
├── Center, Middle, Right, CenterMiddle (new - alignment)
├── VerticalGroup, HorizontalGroup (new - collapsing)
└── ItemGrid (new - dynamic grid)

ScrollView (existing)
└── ScrollableContainer (new - adds keyboard bindings)
    ├── VerticalScroll (new - vertical only)
    └── HorizontalScroll (new - horizontal only)

ContentSwitcher (new)
└── TabbedContent uses ContentSwitcher for pane visibility

Tab, Tabs, Underline (new)
├── TabPane (new)
└── TabbedContent (new - composes Tabs + ContentSwitcher)

CollapsibleTitle, Contents (new)
└── Collapsible (new - composes Title + Contents)
```

### 12. Implementation Order

1. **Core Extensions** (Required First)
   - Add `Display` enum to geometry.rs
   - Add `Alignment` struct (AlignH, AlignV) to geometry.rs
   - Extend `LayoutHints` with display and alignment fields
   - Add Widget trait methods (see Section 2.1):
     - `id()` → `Option<&str>`
     - `get_content_width()` → `u16`
     - `get_content_height()` → `u16`
     - `grid_config()` → `Option<&GridConfig>`
   - Add `ChildDisplayOverride` trait to layout.rs
   - Update layout algorithms:
     - Filter `display: none` widgets via `get_effective_display()`
     - Two-pass constraint resolution for Fraction support
     - Apply alignment after fraction resolution

2. **Alignment Containers** (Simple)
   - Center, Middle, Right, CenterMiddle

3. **Group Containers** (Simple)
   - VerticalGroup, HorizontalGroup

4. **ScrollableContainer** (Medium)
   - Add keyboard bindings
   - Implement ActionHandler

5. **VerticalScroll, HorizontalScroll** (Simple)
   - Wrappers with overflow presets

6. **ItemGrid** (Medium)
   - Reactive properties
   - Override grid_config()

7. **ContentSwitcher** (Medium)
   - Display-based visibility management
   - CurrentChanged message

8. **Tab Widgets** (Complex)
   - Tab (with display for hidden)
   - Tabs, Underline
   - TabPane (required ID)
   - TabbedContent (ID prefix system)

9. **Collapsible** (Medium)
   - CollapsibleTitle, Contents
   - Collapsible (composed)
   - Default collapsed = true
