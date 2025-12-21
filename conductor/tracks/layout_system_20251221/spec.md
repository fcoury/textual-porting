# Specification: Layout System

## Overview
This track implements the complete layout system for arranging widgets within containers. It provides vertical, horizontal, and grid layouts, along with scrolling support and viewport management. The layout system determines how widgets are sized and positioned based on their constraints and container dimensions.

## Goals
- Implement Vertical layout (stacking widgets top-to-bottom)
- Implement Horizontal layout (stacking widgets left-to-right)
- Implement Grid layout (CSS Grid-like 2D placement)
- Add scrolling support with ScrollView widget
- Implement viewport management for efficient rendering
- Support fractional units (fr) for flexible sizing

## Reference: Python Textual Layout System

### Layout Classes
```python
# layouts/vertical.py
class VerticalLayout(Layout):
    """Arrange widgets vertically."""

    def arrange(self, parent: Widget, children: list[Widget], size: Size) -> DockArrangeResult:
        # Stack children top to bottom
        # Respect min/max sizes
        # Distribute remaining space

# layouts/horizontal.py
class HorizontalLayout(Layout):
    """Arrange widgets horizontally."""

# layouts/grid.py
class GridLayout(Layout):
    """CSS Grid-like 2D layout."""
    # grid-template-columns, grid-template-rows
    # grid-column, grid-row for placement
    # gap between cells
```

### Container Widgets
```python
from textual.containers import Vertical, Horizontal, Grid, ScrollableContainer

class MyApp(App):
    def compose(self):
        yield Vertical(
            Horizontal(
                Button("A"),
                Button("B"),
            ),
            Static("Below"),
        )
```

### Scrolling
```python
class ScrollableContainer(Widget):
    """Container with scroll bars."""

    def scroll_to(self, x: int, y: int) -> None: ...
    def scroll_visible(self, widget: Widget) -> None: ...

    # Virtual size vs viewport size
    # Scroll offset tracking
    # ScrollBar widgets
```

### Size Units
```python
# CSS-like sizing
width: 100%      # Percentage of parent
width: 50        # Fixed cells
width: 1fr       # Fractional unit
width: auto      # Content-based

min-width: 10
max-width: 50
```

## Deliverables

### Phase 1: Analysis & Research
- Study Python Textual layouts/ directory
- Analyze _arrange.py for arrangement algorithm
- Document ScrollView and ScrollBar implementation
- Map CSS sizing properties to layout decisions

### Phase 2: Design & Planning
- Design Layout trait in Rust
- Plan constraint solving algorithm
- Design ScrollView with virtual content
- Plan viewport culling for performance

### Phase 3: Implementation

#### 3.1 Layout Trait
- Define Layout trait with arrange() method
- LayoutResult with widget placements
- Support for nested layouts

#### 3.2 Vertical Layout
- Stack widgets top-to-bottom
- Respect min/max height constraints
- Distribute extra space based on flex

#### 3.3 Horizontal Layout
- Stack widgets left-to-right
- Respect min/max width constraints
- Distribute extra space based on flex

#### 3.4 Grid Layout
- Grid template definition (columns, rows)
- Cell placement with grid-column/grid-row
- Gap between cells
- Auto-placement algorithm

#### 3.5 Scroll View
- Virtual content size tracking
- Scroll offset management
- ScrollBar widgets (vertical, horizontal)
- Scroll-to-widget functionality
- Keyboard scrolling (Page Up/Down, arrows)

#### 3.6 Viewport Management
- Visible region calculation
- Widget culling (skip rendering off-screen)
- Efficient dirty region tracking

### Phase 4: Testing & Verification
- Unit tests for each layout type
- Nested layout tests
- Scroll behavior tests
- Performance tests with many widgets

## Success Criteria
- [ ] Vertical layout stacks widgets correctly
- [ ] Horizontal layout arranges widgets in a row
- [ ] Grid layout places widgets in 2D grid
- [ ] ScrollView enables scrolling large content
- [ ] Fractional units distribute space correctly
- [ ] Viewport culling improves performance

## Dependencies
- Core App Lifecycle track (compose, mount)
- Existing dock layout foundation

## Out of Scope
- CSS parsing (TCSS Styling track)
- Individual widgets (Widget tracks)
- Animations/transitions
