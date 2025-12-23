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

### 2. Layout Type Extension

The `Widget` trait already has `layout_type()` returning `Option<LayoutType>`:

```rust
// Existing:
fn layout_type(&self) -> Option<LayoutType> {
    None // Default
}

// LayoutType enum (extend if needed):
pub enum LayoutType {
    Vertical,
    Horizontal,
    Grid,
}
```

For alignment containers, they don't need a special layout type - they use CSS-like alignment which is handled differently.

### 3. Alignment Container Design

Alignment is handled via `LayoutHints`. Add alignment fields:

```rust
// In geometry.rs or layout.rs:
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

// Extend LayoutHints:
pub struct LayoutHints {
    pub width: Constraint,
    pub height: Constraint,
    pub dock: Option<Dock>,
    pub alignment: Alignment,  // NEW
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
                .with_width(Constraint::Ratio(1.0))
                .with_height(Constraint::Auto),
        }
    }
    // Builder methods delegate to inner...
}

impl Widget for Center {
    fn layout(&self) -> LayoutHints {
        self.inner.layout().with_align_horizontal(AlignH::Center)
    }
    // Other methods delegate to inner...
}
```

Same pattern for:
- **Middle** - `align_vertical: AlignV::Middle`, `width: auto`, `height: 1fr`
- **CenterMiddle** - Both alignments, `width: 1fr`, `height: 1fr`
- **Right** - `align_horizontal: AlignH::Right`, `width: 1fr`, `height: auto`

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
                .with_width(Constraint::Ratio(1.0))
                .with_height(Constraint::Auto),
        }
    }
}

impl Widget for VerticalGroup {
    fn layout_type(&self) -> Option<LayoutType> {
        Some(LayoutType::Vertical)
    }

    fn layout(&self) -> LayoutHints {
        // Set overflow: hidden hidden in layout hints
        self.inner.layout().with_overflow(OverflowSettings::both(Overflow::Hidden))
    }
}
```

Same pattern for **HorizontalGroup**.

#### 4.3 ItemGrid

```rust
/// Grid with dynamic column configuration.
pub struct ItemGrid {
    inner: Container,
    config: GridConfig,
    stretch_height: bool,
    min_column_width: Option<u16>,
    max_column_width: Option<u16>,
    regular: bool,
}

impl ItemGrid {
    pub fn new() -> Self {
        Self {
            inner: Container::new()
                .with_width(Constraint::Ratio(1.0))
                .with_height(Constraint::Auto),
            config: GridConfig::default(),
            stretch_height: false,
            min_column_width: None,
            max_column_width: None,
            regular: false,
        }
    }

    pub fn with_stretch_height(mut self, stretch: bool) -> Self {
        self.stretch_height = stretch;
        self
    }

    pub fn with_min_column_width(mut self, width: u16) -> Self {
        self.min_column_width = Some(width);
        self
    }

    // ... more builders
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

```rust
// src/widgets/content_switcher.rs

use std::collections::HashMap;

/// Container that shows one child at a time.
pub struct ContentSwitcher {
    inner: Container,
    /// Currently visible child ID.
    current: Option<String>,
    /// Child visibility states.
    visibility: HashMap<String, bool>,
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
            visibility: HashMap::new(),
        }
    }

    pub fn with_initial(mut self, id: impl Into<String>) -> Self {
        self.current = Some(id.into());
        self
    }

    /// Get currently visible child ID.
    pub fn current(&self) -> Option<&str> {
        self.current.as_deref()
    }

    /// Set currently visible child.
    /// Returns CurrentChanged message to post.
    pub fn set_current(&mut self, id: Option<String>) -> CurrentChanged {
        // Hide old current
        if let Some(old) = &self.current {
            self.visibility.insert(old.clone(), false);
        }

        // Show new current
        if let Some(new) = &id {
            self.visibility.insert(new.clone(), true);
        }

        self.current = id;

        CurrentChanged {
            switcher_id: self.inner.id().map(|s| s.to_string()),
        }
    }

    /// Check if child should be displayed.
    pub fn is_visible(&self, id: &str) -> bool {
        self.visibility.get(id).copied().unwrap_or(false)
    }

    /// Add content with ID.
    pub fn add_content(&mut self, id: impl Into<String>, set_current: bool) {
        let id = id.into();
        let visible = set_current || self.current.is_none();
        self.visibility.insert(id.clone(), visible);

        if set_current || self.current.is_none() {
            self.current = Some(id);
        }
    }
}
```

### 7. Tab Widgets Design

#### 7.1 Tab Widget

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
    pub fn new(id: impl Into<String>, label: impl Into<String>) -> Self {
        Self {
            id: id.into(),
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
    }

    fn widget_type_name(&self) -> &'static str {
        "Tab"
    }
}
```

#### 7.2 Tabs Widget (Tab Bar)

```rust
/// Tab bar containing multiple tabs.
pub struct Tabs {
    tabs: Vec<Tab>,
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
            active: None,
            bindings,
        }
    }

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

            Some(TabActivated {
                tab_id: id,
                previous,
            })
        } else {
            None
        }
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

    /// Hide a tab.
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
            .with_width(Constraint::Ratio(1.0))
            .with_height(Constraint::Length(2))  // 1 for tabs, 1 for underline
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

#### 7.3 Underline Widget

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
            .with_width(Constraint::Ratio(1.0))
            .with_height(Constraint::Length(1))
    }

    fn widget_type_name(&self) -> &'static str {
        "Underline"
    }
}
```

#### 7.4 TabPane Widget

```rust
/// A pane of content associated with a tab.
pub struct TabPane {
    inner: Container,
    title: String,
    disabled: bool,
}

/// TabPane disabled message.
pub struct TabPaneDisabled {
    pub pane_id: Option<String>,
}

/// TabPane enabled message.
pub struct TabPaneEnabled {
    pub pane_id: Option<String>,
}

/// TabPane focused (descendant received focus).
pub struct TabPaneFocused {
    pub pane_id: Option<String>,
}

impl TabPane {
    pub fn new(title: impl Into<String>) -> Self {
        Self {
            inner: Container::new().with_height(Constraint::Auto),
            title: title.into(),
            disabled: false,
        }
    }

    pub fn with_id(mut self, id: impl Into<String>) -> Self {
        self.inner = self.inner.with_id(id);
        self
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

#### 7.5 TabbedContent Widget

```rust
/// Container with tabs and switchable content panes.
pub struct TabbedContent {
    inner: Container,
    tabs: Tabs,
    switcher: ContentSwitcher,
}

impl TabbedContent {
    pub fn new() -> Self {
        Self {
            inner: Container::new().with_height(Constraint::Auto),
            tabs: Tabs::new(),
            switcher: ContentSwitcher::new(),
        }
    }

    /// Get active pane ID.
    pub fn active(&self) -> Option<&str> {
        self.tabs.active()
    }

    /// Set active pane.
    pub fn set_active(&mut self, id: impl Into<String>) -> Option<TabActivated> {
        let id = id.into();
        let result = self.tabs.set_active(&id);
        if result.is_some() {
            self.switcher.set_current(Some(id));
        }
        result
    }

    /// Add a pane.
    pub fn add_pane(&mut self, pane: TabPane) {
        let id = pane.inner.id().unwrap_or("").to_string();
        let tab = Tab::new(&id, pane.title())
            .with_disabled(pane.is_disabled());

        self.tabs.add_tab(tab);
        self.switcher.add_content(&id, false);
    }

    /// Remove a pane.
    pub fn remove_pane(&mut self, id: &str) {
        self.tabs.remove_tab(id);
        // Note: actual pane removal requires compose tree manipulation
    }

    /// Clear all panes.
    pub fn clear_panes(&mut self) {
        self.tabs.clear();
        self.switcher.set_current(None);
    }

    /// Get a tab by pane ID.
    pub fn get_tab(&self, pane_id: &str) -> Option<&Tab> {
        self.tabs.tabs.iter().find(|t| t.id() == pane_id)
    }

    /// Disable a tab.
    pub fn disable_tab(&mut self, id: &str) {
        self.tabs.disable(id);
    }

    /// Enable a tab.
    pub fn enable_tab(&mut self, id: &str) {
        self.tabs.enable(id);
    }

    /// Hide a tab.
    pub fn hide_tab(&mut self, id: &str) {
        self.tabs.hide(id);
    }

    /// Show a tab.
    pub fn show_tab(&mut self, id: &str) {
        self.tabs.show(id);
    }
}
```

### 8. Collapsible Widget Design

Based on research: Python Textual's Collapsible has **no animation** - it uses instant `display: none` toggle.

```rust
// src/widgets/collapsible.rs

/// Collapsible container with toggle header.
pub struct Collapsible {
    inner: Container,
    title: String,
    collapsed: bool,
    bindings: BindingManager,
}

/// Collapsible toggled message.
pub struct Toggled {
    pub collapsible_id: Option<String>,
    pub collapsed: bool,
}

impl Collapsible {
    pub fn new(title: impl Into<String>) -> Self {
        let mut bindings = BindingManager::new();
        bindings.add(Binding::new("enter", "toggle").with_show(false));

        Self {
            inner: Container::new()
                .with_width(Constraint::Ratio(1.0))
                .with_height(Constraint::Auto),
            title: title.into(),
            collapsed: false,
            bindings,
        }
    }

    pub fn with_id(mut self, id: impl Into<String>) -> Self {
        self.inner = self.inner.with_id(id);
        self
    }

    pub fn with_collapsed(mut self, collapsed: bool) -> Self {
        self.collapsed = collapsed;
        self
    }

    pub fn title(&self) -> &str {
        &self.title
    }

    pub fn is_collapsed(&self) -> bool {
        self.collapsed
    }

    /// Expand the collapsible.
    pub fn expand(&mut self) -> Option<Toggled> {
        if self.collapsed {
            self.collapsed = false;
            Some(Toggled {
                collapsible_id: self.inner.id().map(|s| s.to_string()),
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
            Some(Toggled {
                collapsible_id: self.inner.id().map(|s| s.to_string()),
                collapsed: true,
            })
        } else {
            None
        }
    }

    /// Toggle collapsed state.
    pub fn toggle(&mut self) -> Toggled {
        self.collapsed = !self.collapsed;
        Toggled {
            collapsible_id: self.inner.id().map(|s| s.to_string()),
            collapsed: self.collapsed,
        }
    }

    /// Check if content should be displayed.
    pub fn content_visible(&self) -> bool {
        !self.collapsed
    }
}

impl Widget for Collapsible {
    fn focusable(&self) -> bool {
        true
    }

    fn layout(&self) -> LayoutHints {
        self.inner.layout()
    }

    fn widget_type_name(&self) -> &'static str {
        "Collapsible"
    }
}

impl ActionHandler for Collapsible {
    fn run_action(&mut self, action: &ParsedAction) -> Result<bool, ActionError> {
        match action.name.as_str() {
            "toggle" => {
                self.toggle();
                Ok(true)
            }
            "expand" => {
                self.expand();
                Ok(true)
            }
            "collapse" => {
                self.collapse();
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

### 9. File Organization

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
└── collapsible.rs      # Collapsible (NEW)
```

### 10. Dependencies Between Widgets

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

Collapsible (new - standalone)
```

### 11. Implementation Order

1. **Alignment Extensions** (LayoutHints)
   - Add Alignment struct to geometry/layout
   - Extend LayoutHints with alignment field

2. **Alignment Containers** (Trivial)
   - Center, Middle, Right, CenterMiddle

3. **Group Containers** (Trivial)
   - VerticalGroup, HorizontalGroup

4. **ScrollableContainer** (Medium)
   - Add keyboard bindings
   - Implement ActionHandler

5. **VerticalScroll, HorizontalScroll** (Trivial)
   - Wrappers with overflow presets

6. **ItemGrid** (Medium)
   - Reactive properties
   - GridLayout integration

7. **ContentSwitcher** (Medium)
   - Visibility management
   - CurrentChanged message

8. **Tab Widgets** (Complex)
   - Tab, Tabs, Underline
   - TabPane, TabbedContent
   - Message flow coordination

9. **Collapsible** (Medium)
   - Toggle state
   - Keyboard binding (Enter)
   - No animation (instant toggle)
