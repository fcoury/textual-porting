# Layout System Technical Design

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Layout Trait](#layout-trait)
4. [Flex Distribution](#flex-distribution)
5. [ScrollView](#scrollview)
6. [Viewport Culling](#viewport-culling)
7. [Implementation Phases](#implementation-phases)

---

## Overview

The Layout System is responsible for sizing and positioning widgets within containers. It implements CSS-like layout algorithms including vertical stacking, horizontal stacking, and CSS Grid-like 2D layouts.

### Goals
- Implement Vertical, Horizontal, and Grid layouts
- Support fractional units (`fr`) for flexible sizing
- Enable scrolling for content larger than viewport
- Optimize rendering with viewport culling

### Non-Goals
- CSS parsing (handled by TCSS Styling track)
- Animation/transitions
- Individual widget implementations

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        App / Screen                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     arrange() Algorithm                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │   Layers    │→ │  Split/Dock │→ │   Layout.arrange()      │  │
│  └─────────────┘  └─────────────┘  │  (Vertical/Horizontal/  │  │
│                                     │   Grid)                  │  │
│                                     └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      PlacementCache                              │
│    HashMap<WidgetId, WidgetPlacement>  (positions + metadata)    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Rendering                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │   Culling   │→ │   Render    │→ │     RenderBuffer        │  │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **arrange()** orchestrates the layout:
   - Filters visible widgets
   - Groups by layer
   - Handles split/dock widgets
   - Delegates to Layout trait

2. **Layout.arrange()** computes placements:
   - Resolves widget sizes
   - Distributes flex space
   - Positions widgets
   - Returns `Vec<WidgetPlacement>`

3. **PlacementCache** stores results:
   - Maps WidgetId → WidgetPlacement (region + fixed flag + order)
   - Used for rendering, hit-testing, and culling decisions

4. **Rendering** uses cache:
   - Culls off-screen widgets
   - Renders visible widgets
   - Tracks dirty regions

---

## Layout Trait

### Core Types

```rust
/// Placement of a widget after layout.
pub struct WidgetPlacement {
    pub widget_id: WidgetId,
    pub region: Rect,           // Position and size
    pub offset: Offset,         // Additional offset (scroll, absolute)
    pub margin: Spacing,        // Margin (for reference)
    pub order: i32,             // Z-order
    pub fixed: bool,            // Doesn't scroll
}

/// Layout algorithm trait.
pub trait Layout: Debug + Send + Sync {
    fn arrange(
        &self,
        context: &LayoutContext,
        parent_id: WidgetId,
        children: &[WidgetId],
        size: Size,
        greedy: bool,
    ) -> Vec<WidgetPlacement>;

    /// Get content width for auto-sizing (default: 0).
    fn get_content_width(&self, context: &LayoutContext, children: &[WidgetId], container: Size) -> u16 { 0 }

    /// Get content height for auto-sizing (default: 0).
    fn get_content_height(&self, context: &LayoutContext, children: &[WidgetId], container: Size, width: u16) -> u16 { 0 }

    fn name(&self) -> &'static str;
}
```

### Layout Types

```
┌─────────────────────────────────────────────────────────────────┐
│                      LayoutType                                  │
├─────────────────────────────────────────────────────────────────┤
│  Vertical        Horizontal          Grid                       │
│  ┌─────┐         ┌───┬───┬───┐      ┌───┬───┬───┐              │
│  │  A  │         │ A │ B │ C │      │ A │ B │ C │              │
│  ├─────┤         └───┴───┴───┘      ├───┼───┼───┤              │
│  │  B  │                             │ D │ E │ F │              │
│  ├─────┤                             └───┴───┴───┘              │
│  │  C  │                                                         │
│  └─────┘                                                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Flex Distribution

### Algorithm Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                  Flex Distribution Algorithm                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Input: [1fr, 2fr, 1fr], available=100                          │
│                                                                  │
│  Step 1: Classify                                               │
│    └─ All fractional, no fixed sizes                            │
│                                                                  │
│  Step 2: Calculate fr_unit                                      │
│    └─ total_fr = 4, fr_unit = 100/4 = 25                        │
│                                                                  │
│  Step 3: Apply (no constraints hit)                             │
│    └─ [25, 50, 25]                                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### With Constraints

```
┌─────────────────────────────────────────────────────────────────┐
│                  With Min/Max Constraints                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Input: [1fr min:30, 2fr, 1fr max:20], available=100            │
│                                                                  │
│  Iteration 1:                                                   │
│    fr_unit = 100/4 = 25                                         │
│    A: 25 < 30 (min) → lock at 30                                │
│    B: 50 → OK                                                   │
│    C: 25 > 20 (max) → lock at 20                                │
│                                                                  │
│  Iteration 2:                                                   │
│    remaining = 100 - 30 - 20 = 50                               │
│    fr_unit = 50/2 = 25                                          │
│    B: 50 → OK, lock                                             │
│                                                                  │
│  Result: [30, 50, 20]                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## ScrollView

### Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                        ScrollView                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────┐ ┌───┐ │
│  │                                                      │ │ ▲ │ │
│  │              Viewport (visible area)                 │ │ █ │ │
│  │                                                      │ │ ║ │ │
│  │    ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐         │ │ ║ │ │
│  │            Virtual Content                           │ │ ║ │ │
│  │    │   (larger than viewport)            │          │ │ ▼ │ │
│  │    └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘         │ └───┘ │
│  └─────────────────────────────────────────────────────┘        │
│  ┌─────────────────────────────────────────────────────┐ ┌───┐ │
│  │  ◄ ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ ►   │ │   │ │
│  └─────────────────────────────────────────────────────┘ └───┘ │
│  └── Horizontal Scrollbar ──────────────────────────────┘ └──┘ │
│                                                          Corner │
└─────────────────────────────────────────────────────────────────┘
```

### Scroll State

```rust
pub struct ScrollState {
    pub offset: Offset,         // Current scroll position
    pub virtual_size: Size,     // Total content size
    pub viewport_size: Size,    // Visible area size
}
```

### Key Operations

| Operation | Description |
|-----------|-------------|
| `scroll_to(x, y)` | Scroll to absolute position |
| `scroll_by(dx, dy)` | Scroll by relative amount |
| `scroll_visible(rect)` | Scroll to make widget visible |
| `scroll_home()` | Scroll to top-left |
| `scroll_end()` | Scroll to bottom-right |
| `page_up/down()` | Scroll by viewport height |

---

## Viewport Culling

### Culling Process

```
┌─────────────────────────────────────────────────────────────────┐
│                      Viewport Culling                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│     Virtual Content                                              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  [Widget A]  ← Off-screen (SKIP)                          │  │
│  │  [Widget B]  ← Off-screen (SKIP)                          │  │
│  ├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │  [Widget C]  ← Visible (RENDER)                     │  │  │
│  │  │  [Widget D]  ← Visible (RENDER)                     │  │  │
│  │  │  [Widget E]  ← Partially visible (RENDER, clipped)  │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  ├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┤  │
│  │  [Widget F]  ← Off-screen (SKIP)                          │  │
│  │  [Widget G]  ← Off-screen (SKIP)                          │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Dirty Region Tracking

```rust
pub enum DirtyRegion {
    Full,              // Everything needs re-render
    Partial(Vec<Rect>), // Specific areas changed
    None,              // Nothing changed
}
```

### Optimization Levels

| Level | Technique | Use Case |
|-------|-----------|----------|
| Basic | Viewport culling | Always on |
| Medium | Dirty region tracking | Partial updates |
| Advanced | Virtual list | 10,000+ items |

---

## Implementation Phases

### Phase 3.1: Layout Trait Foundation
- Define `Layout` trait
- Define `WidgetPlacement`, `LayoutContext`
- Implement layout selection

### Phase 3.2: Vertical Layout
- Top-to-bottom stacking
- Min/max height constraints
- Flex space distribution

### Phase 3.3: Horizontal Layout
- Left-to-right stacking
- Min/max width constraints
- Flex space distribution

### Phase 3.4: Grid Layout
- Grid template (columns, rows)
- Auto-placement algorithm
- Cell spanning
- Gaps

### Phase 3.5: Container Widgets
- `Vertical` container
- `Horizontal` container
- `Grid` container

### Phase 3.6: Scroll View
- `ScrollView` widget
- `ScrollBar` widgets
- Scroll operations

### Phase 3.7: Viewport Management
- Culling implementation
- Dirty region tracking
- Performance optimization

---

## Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Widget references | WidgetId, not &Widget | Avoids lifetime complexity |
| Fraction calculation | f32 intermediate | Avoids rounding errors |
| Scroll offset | Negative content position | Matches Python Textual |
| Scrollbar widgets | Separate widgets | Enables independent styling |
| Culling | Bounding box intersection | Simple and fast |
| Dirty tracking | Rectangle-based | Good balance of complexity/benefit |

---

## File Structure

```
src/
├── layout/
│   ├── mod.rs           # Layout trait, WidgetPlacement
│   ├── vertical.rs      # VerticalLayout
│   ├── horizontal.rs    # HorizontalLayout
│   ├── grid.rs          # GridLayout
│   └── flex.rs          # Flex distribution algorithm
├── widgets/
│   ├── containers/
│   │   ├── vertical.rs  # Vertical container
│   │   ├── horizontal.rs # Horizontal container
│   │   ├── grid.rs      # Grid container
│   │   └── scroll_view.rs # ScrollView
│   └── scrollbar.rs     # ScrollBar widget
└── render/
    ├── culling.rs       # Viewport culling
    └── dirty.rs         # Dirty region tracking
```

---

## References

- [Layout Trait Design](./layout_trait.md)
- [Flex Distribution Algorithm](./flex_distribution.md)
- [ScrollView Architecture](./scrollview.md)
- [Viewport Culling Strategy](./viewport_culling.md)
- [Phase 1 Research](../research/)
