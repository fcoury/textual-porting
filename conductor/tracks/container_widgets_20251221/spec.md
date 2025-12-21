# Specification: Container Widgets

## Overview
This track implements container widgets that organize and group other widgets. These containers provide layout structure, scrolling, tabbed navigation, and collapsible sections for building complex UIs.

## Goals
- Implement layout containers (Container, Vertical, Horizontal, Grid, Center, Middle)
- Implement scrolling containers (VerticalScroll, HorizontalScroll, ScrollableContainer)
- Implement TabbedContent with Tab navigation
- Implement Collapsible sections
- Implement ContentSwitcher for dynamic content

## Reference: Python Textual Containers

### Base Container
```python
class Container(Widget):
    """Base container widget."""
    # Provides compose() for children
    # Default vertical layout
```

### Layout Containers
```python
from textual.containers import (
    Vertical,      # Stack children vertically
    Horizontal,    # Stack children horizontally
    Grid,          # CSS Grid layout
    Center,        # Center child horizontally and vertically
    Middle,        # Center child vertically only
)

class MyApp(App):
    def compose(self):
        yield Vertical(
            Horizontal(
                Button("A"),
                Button("B"),
            ),
            Static("Content"),
        )
```

### Scrolling Containers
```python
class VerticalScroll(ScrollableContainer):
    """Vertical scrolling container."""
    # Only vertical scroll bar

class HorizontalScroll(ScrollableContainer):
    """Horizontal scrolling container."""
    # Only horizontal scroll bar

class ScrollableContainer(Widget):
    """Container with both scroll bars."""
    can_focus = True

    def scroll_to_widget(self, widget: Widget) -> None: ...
    def scroll_home(self) -> None: ...
    def scroll_end(self) -> None: ...
    def scroll_page_up(self) -> None: ...
    def scroll_page_down(self) -> None: ...
```

### TabbedContent
```python
class TabbedContent(Widget):
    """Container with tab bar navigation."""
    active: str  # ID of active tab

    class TabActivated(Message):
        tabbed_content: TabbedContent
        tab: Tab

    class Cleared(Message): ...

    def add_pane(self, pane: TabPane) -> AwaitComplete: ...
    def remove_pane(self, pane_id: str) -> AwaitComplete: ...
    def clear_panes(self) -> AwaitComplete: ...
    def get_tab(self, tab_id: str) -> Tab: ...
    def get_pane(self, pane_id: str) -> TabPane: ...

class TabPane(Widget):
    """A single pane within TabbedContent."""
    title: str
    disabled: bool

class Tabs(Widget):
    """Tab bar widget."""
    active: str

    class TabActivated(Message): ...
    class Cleared(Message): ...
```

### Collapsible
```python
class Collapsible(Widget):
    """A collapsible section with header."""
    collapsed: reactive[bool] = reactive(True)
    title: str

    class Toggled(Message):
        collapsible: Collapsible

    def expand(self) -> None: ...
    def collapse(self) -> None: ...
    def toggle(self) -> None: ...
```

### ContentSwitcher
```python
class ContentSwitcher(Widget):
    """Switch between multiple content widgets."""
    current: str | None  # ID of visible content

    class CurrentChanged(Message): ...
```

## Deliverables

### Phase 1: Analysis & Research
- Study Python Textual containers.py implementation
- Analyze ScrollableContainer scrolling behavior
- Document TabbedContent and Tabs implementation
- Study Collapsible animation and state

### Phase 2: Design & Planning
- Design Container base with compose support
- Plan scroll bar integration
- Design tab/pane relationship
- Plan collapsible animation

### Phase 3: Implementation

#### 3.1 Container Base
- Base Container widget
- Default layout behavior
- Children management via compose

#### 3.2 Layout Containers
- Vertical (stacks top-to-bottom)
- Horizontal (stacks left-to-right)
- Grid (CSS Grid layout)
- Center (center both axes)
- Middle (center vertical only)

#### 3.3 Scrolling Containers
- ScrollableContainer with both scroll bars
- VerticalScroll (vertical only)
- HorizontalScroll (horizontal only)
- Scroll methods (scroll_to_widget, scroll_home, etc.)
- Keyboard scrolling support

#### 3.4 TabbedContent
- Tabs widget (tab bar)
- TabPane widget (content pane)
- TabbedContent (combines Tabs + TabPanes)
- Tab switching and activation
- Dynamic add/remove panes
- TabActivated message

#### 3.5 Collapsible
- Collapsible header rendering
- Collapsed/expanded state
- Toggle animation
- Toggled message
- expand/collapse/toggle methods

#### 3.6 ContentSwitcher
- Multiple content widgets
- current property for visible content
- CurrentChanged message

### Phase 4: Testing & Verification
- Unit tests for each container
- Nested container tests
- Scroll behavior tests
- Tab navigation tests

## Success Criteria
- [ ] Container provides base for all containers
- [ ] Vertical/Horizontal stack correctly
- [ ] Grid arranges in 2D
- [ ] ScrollableContainer scrolls with bars
- [ ] TabbedContent switches between tabs
- [ ] Collapsible toggles open/closed
- [ ] ContentSwitcher shows one content at a time

## Dependencies
- Core App Lifecycle (compose, mount)
- Layout System (layouts, ScrollView)
- TCSS Styling (container styles)

## Out of Scope
- ListView (virtualized - Tier 4)
- Complex data containers
