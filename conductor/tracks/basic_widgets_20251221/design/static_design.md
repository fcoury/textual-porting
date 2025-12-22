# Static Widget Technical Design

## Overview

Static is the foundational widget for displaying static content. It serves as the base class for Label and other simple display widgets. This design ensures API parity with Python Textual while leveraging Rust idioms.

## Rust Implementation

### Content Type System

```rust
/// Represents content that can be displayed by Static.
/// Mirrors Python Textual's VisualType.
#[derive(Debug, Clone)]
pub enum ContentType {
    /// Plain text (may contain markup if markup=true)
    Text(String),
    /// Pre-styled content (markup already parsed)
    Styled(StyledContent),
    /// Empty content
    Empty,
}

impl Default for ContentType {
    fn default() -> Self {
        Self::Empty
    }
}

impl From<&str> for ContentType {
    fn from(s: &str) -> Self {
        Self::Text(s.to_string())
    }
}

impl From<String> for ContentType {
    fn from(s: String) -> Self {
        Self::Text(s)
    }
}

/// Pre-styled content with spans.
#[derive(Debug, Clone)]
pub struct StyledContent {
    spans: Vec<StyledSpan>,
}

#[derive(Debug, Clone)]
pub struct StyledSpan {
    text: String,
    style: Style,
}
```

### Static Widget Structure

```rust
use crate::reactive::{Reactive, ReactiveFlags, WidgetRefreshState};

/// A widget for displaying static content with optional markup parsing.
///
/// Static is the base for Label and other simple display widgets.
/// It supports Rich-style markup (when `markup=true`) and can expand
/// or shrink to fit its container.
///
/// # Example
/// ```rust,ignore
/// let widget = Static::new("Hello [bold]World[/bold]");
/// let no_markup = Static::new("Plain text").with_markup(false);
/// ```
#[derive(Debug)]
pub struct Static {
    // === Reactive Properties ===
    /// The content to display.
    content: ContentType,

    /// Whether to expand content to fill available space.
    expand: bool,

    /// Whether to shrink content to fit container.
    shrink: bool,

    /// Whether to parse markup in text content.
    markup: bool,

    // === Internal State ===
    /// Cached visual representation (lazy-evaluated).
    visual_cache: Option<StyledContent>,

    /// Refresh state for reactive updates.
    refresh_state: WidgetRefreshState,
}

impl Default for Static {
    fn default() -> Self {
        Self {
            content: ContentType::Empty,
            expand: false,
            shrink: false,
            markup: true,  // Default: markup enabled (matches Python)
            visual_cache: None,
            refresh_state: WidgetRefreshState::new(),
        }
    }
}
```

### Constructor and Builder Pattern

```rust
impl Static {
    /// Create a new Static widget with the given content.
    pub fn new(content: impl Into<ContentType>) -> Self {
        Self {
            content: content.into(),
            ..Default::default()
        }
    }

    /// Set whether to parse markup in text content.
    pub fn with_markup(mut self, markup: bool) -> Self {
        self.markup = markup;
        self
    }

    /// Set whether to expand content to fill container.
    pub fn with_expand(mut self, expand: bool) -> Self {
        self.expand = expand;
        self
    }

    /// Set whether to shrink content to fit container.
    pub fn with_shrink(mut self, shrink: bool) -> Self {
        self.shrink = shrink;
        self
    }
}
```

### Content Update Method

```rust
impl Static {
    /// Update the content and trigger a refresh.
    ///
    /// This matches Python Textual's `update()` method.
    ///
    /// # Arguments
    /// * `content` - The new content to display
    /// * `layout` - If true, also triggers layout recalculation (default: true)
    pub fn update(&mut self, content: impl Into<ContentType>, layout: bool) {
        self.content = content.into();
        self.visual_cache = None;  // Invalidate cache

        if layout {
            self.refresh_state.mark_refresh(ReactiveFlags::layout());
        } else {
            self.refresh_state.mark_refresh(ReactiveFlags::REPAINT);
        }
    }

    /// Update content with default layout=true.
    pub fn update_content(&mut self, content: impl Into<ContentType>) {
        self.update(content, true);
    }
}
```

### Visual Rendering

```rust
impl Static {
    /// Get or create the visual representation for rendering.
    fn visual(&mut self) -> &StyledContent {
        if self.visual_cache.is_none() {
            self.visual_cache = Some(self.visualize());
        }
        self.visual_cache.as_ref().unwrap()
    }

    /// Convert content to styled content for rendering.
    fn visualize(&self) -> StyledContent {
        match &self.content {
            ContentType::Empty => StyledContent { spans: vec![] },
            ContentType::Styled(styled) => styled.clone(),
            ContentType::Text(text) => {
                if self.markup {
                    parse_markup(text)
                } else {
                    StyledContent {
                        spans: vec![StyledSpan {
                            text: text.clone(),
                            style: Style::default(),
                        }],
                    }
                }
            }
        }
    }
}
```

### Widget Trait Implementation

```rust
impl Widget for Static {
    fn render(&self, area: Rect, frame: &mut Frame) {
        // Use cached visual or create if needed
        let styled = match &self.visual_cache {
            Some(v) => v,
            None => {
                // Note: We need interior mutability here, or render takes &mut self
                // For now, recompute on each render if not cached
                &self.visualize()
            }
        };

        // Convert to ratatui spans
        let spans: Vec<Span> = styled.spans.iter()
            .map(|s| Span::styled(&s.text, s.style.into()))
            .collect();

        let paragraph = Paragraph::new(Line::from(spans));
        frame.render_widget(paragraph, area);
    }

    fn layout(&self) -> LayoutHints {
        LayoutHints::new()
            .with_height(Constraint::Auto)  // height: auto (from DEFAULT_CSS)
    }

    fn focusable(&self) -> bool {
        false  // Static is not focusable by default
    }

    fn as_any(&self) -> &dyn Any { self }
    fn as_any_mut(&mut self) -> &mut dyn Any { self }

    fn widget_type_name(&self) -> &'static str {
        "Static"
    }
}
```

### Reactive Implementation

```rust
impl Reactive for Static {
    fn on_reactive_change(&self, property: &str, flags: ReactiveFlags) {
        self.refresh_state.mark_refresh(flags);
    }

    fn refresh_state(&self) -> &WidgetRefreshState {
        &self.refresh_state
    }
}
```

## Markup Parsing

The markup parser converts Rich-style markup to styled spans:

```rust
/// Parse Rich-style markup into styled content.
///
/// Supported tags:
/// - [bold], [b] - Bold text
/// - [italic], [i] - Italic text
/// - [underline], [u] - Underlined text
/// - [red], [green], [blue], etc. - Foreground colors
/// - [on red], [on green], etc. - Background colors
/// - [/] - Close most recent tag
/// - [/bold], [/italic], etc. - Close specific tag
///
/// # Example
/// ```rust,ignore
/// let styled = parse_markup("Hello [bold]World[/bold]!");
/// // Results in: "Hello " (normal) + "World" (bold) + "!" (normal)
/// ```
pub fn parse_markup(text: &str) -> StyledContent {
    // Implementation uses a state machine to track open tags
    // and applies styles accordingly
    todo!("Implement markup parser")
}
```

## DEFAULT_CSS Equivalent

Static's default styling is minimal:

```rust
impl Static {
    /// Returns the default layout hints (equivalent to DEFAULT_CSS).
    pub fn default_layout() -> LayoutHints {
        LayoutHints::new()
            .with_height(Constraint::Auto)
    }
}
```

## Integration Notes

### Inheritance Pattern

Rust doesn't have class inheritance, so Label and other widgets that extend Static use composition:

```rust
pub struct Label {
    inner: Static,
    variant: Option<LabelVariant>,
}

impl Label {
    pub fn new(content: impl Into<ContentType>) -> Self {
        Self {
            inner: Static::new(content),
            variant: None,
        }
    }
}

// Delegate Widget methods to inner Static
impl Widget for Label {
    fn render(&self, area: Rect, frame: &mut Frame) {
        // Apply variant styling, then delegate
        self.inner.render(area, frame);
    }
    // ... other methods
}
```

### Link Widget Override

Link extends Static with `markup=false`:

```rust
impl Link {
    pub fn new(text: impl Into<String>) -> Self {
        Self {
            inner: Static::new(text).with_markup(false),
            url: None,
        }
    }
}
```

## Testing Strategy

1. **Unit Tests**:
   - Content type conversions
   - Markup parsing (various tags, nested, malformed)
   - Update method (with/without layout)
   - Visual caching

2. **Integration Tests**:
   - Static rendering in a Frame
   - Layout calculations
   - Reactive property changes

## Open Questions

1. **Interior Mutability**: Should `visual_cache` use `RefCell` for lazy evaluation in render()?
2. **Markup Parser**: Implement full Rich markup or subset for MVP?
3. **expand/shrink**: How do these interact with the layout system?
