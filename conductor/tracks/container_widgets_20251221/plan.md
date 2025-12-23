# Plan: Container Widgets

## Phase 1: Analysis & Research [7b4e017]
- [x] Task: Study Python Textual containers.py implementation [b8379a3]
- [x] Task: Analyze ScrollableContainer and scroll methods [9ccbbdb]
- [x] Task: Document _tabbed_content.py and _tabs.py in detail [7b9ffa7]
- [x] Task: Study _collapsible.py animation and state management [9f57adc]
- [x] Task: Analyze _content_switcher.py patterns [a3bde9f]
- [x] Task: Map container DEFAULT_CSS patterns [aabc29f]
- [x] Task: Conductor - User Manual Verification 'Analysis Complete' (Protocol in workflow.md)

## Phase 2: Design & Planning
- [x] Task: Design Container base trait/struct [a89f914]
- [x] Task: Design scroll bar widget integration [a89f914]
- [x] Task: Design Tab/TabPane/TabbedContent relationship [a89f914]
- [x] Task: Plan collapsible animation approach (Result: No animation - instant toggle via display:none) [a89f914]
- [x] Task: Write technical design document [a89f914]
- [x] Task: Conductor - User Manual Verification 'Design Approved' (Protocol in workflow.md) [63eee40]

## Phase 3: Implementation

### 3.1 Container Base (Pre-existing from prior tracks)
- [x] Task: Implement Container struct [2cdcb9a]
- [x] Task: Implement compose() support for children [2cdcb9a]
- [x] Task: Implement default vertical layout [e59dc6f]
- [x] Task: Add DEFAULT_CSS [2cdcb9a]
- [x] Task: Write tests for Container [2cdcb9a]

### 3.2 Vertical Container (Pre-existing from prior tracks)
- [x] Task: Implement Vertical extending Container [e59dc6f]
- [x] Task: Apply VerticalLayout [e59dc6f]
- [x] Task: Support gap/spacing property [76f5a5a]
- [x] Task: Add DEFAULT_CSS [e59dc6f]
- [x] Task: Write tests for Vertical [e59dc6f]

### 3.3 Horizontal Container (Pre-existing from prior tracks)
- [x] Task: Implement Horizontal extending Container [e59dc6f]
- [x] Task: Apply HorizontalLayout [e59dc6f]
- [x] Task: Support gap/spacing property [76f5a5a]
- [x] Task: Add DEFAULT_CSS [e59dc6f]
- [x] Task: Write tests for Horizontal [e59dc6f]

### 3.4 Grid Container (Pre-existing from prior tracks)
- [x] Task: Implement Grid extending Container [76f5a5a]
- [x] Task: Apply GridLayout [76f5a5a]
- [x] Task: Support grid-template-* properties [76f5a5a]
- [x] Task: Add DEFAULT_CSS [76f5a5a]
- [x] Task: Write tests for Grid [76f5a5a]

### 3.5 Center Container
- [x] Task: Implement Center extending Container
- [x] Task: Center child horizontally (alignment: center, width: 1fr, height: auto)
- [x] Task: Add DEFAULT_CSS (via layout hints with alignment)
- [x] Task: Write tests for Center

### 3.6 Middle Container
- [x] Task: Implement Middle extending Container
- [x] Task: Center child vertically only (alignment: middle, width: auto, height: 1fr)
- [x] Task: Add DEFAULT_CSS (via layout hints with alignment)
- [x] Task: Write tests for Middle

### 3.7 ScrollableContainer
- [x] Task: Implement ScrollableContainer struct (wraps ScrollView, adds focusability)
- [x] Task: Integrate vertical ScrollBar widget (via ScrollView inheritance)
- [x] Task: Integrate horizontal ScrollBar widget (via ScrollView inheritance)
- [x] Task: Implement scroll_to_widget() method
- [x] Task: Implement scroll_home() method (action_scroll_home)
- [x] Task: Implement scroll_end() method (action_scroll_end)
- [x] Task: Implement scroll_page_up() method (action_page_up)
- [x] Task: Implement scroll_page_down() method (action_page_down)
- [x] Task: Add keyboard scrolling action methods (app dispatches via bindings)
- [x] Task: Add DEFAULT_CSS (via layout hints - inherits from ScrollView)
- [x] Task: Write tests for ScrollableContainer

### 3.8 VerticalScroll Container
- [x] Task: Implement VerticalScroll extending ScrollableContainer
- [x] Task: Show only vertical scroll bar (overflow-x: hidden, overflow-y: auto)
- [x] Task: Add DEFAULT_CSS (via overflow settings)
- [x] Task: Write tests for VerticalScroll

### 3.9 HorizontalScroll Container
- [x] Task: Implement HorizontalScroll extending ScrollableContainer
- [x] Task: Show only horizontal scroll bar (overflow-x: auto, overflow-y: hidden)
- [x] Task: Add DEFAULT_CSS (via overflow settings)
- [x] Task: Write tests for HorizontalScroll

### 3.10 TabPane Widget
- [x] Task: Implement TabPane struct
- [x] Task: Implement title property
- [x] Task: Implement disabled property
- [x] Task: Add DEFAULT_CSS (via layout hints - width: 1fr, height: auto)
- [x] Task: Write tests for TabPane (4 tests)

### 3.11 Tabs Widget
- [x] Task: Implement Tabs struct (tab bar)
- [x] Task: Implement active property
- [x] Task: Implement tab rendering
- [x] Task: Implement active tab styling
- [x] Task: Implement keyboard navigation (next_tab/previous_tab with wrap-around)
- [x] Task: Implement TabActivated message
- [x] Task: Implement Cleared message (TabsCleared)
- [x] Task: Add DEFAULT_CSS (via layout hints - width: 1fr, height: 2)
- [x] Task: Write tests for Tabs (17 tests)
- [x] Task: Implement Tab widget (label, disabled, hidden, active states)
- [x] Task: Implement Underline widget (highlight indicator)
- [x] Task: Write tests for Tab (6 tests) and Underline (3 tests)

### 3.12 TabbedContent Widget
- [x] Task: Implement TabbedContent struct
- [x] Task: Implement active property
- [x] Task: Implement with_pane_info() method for adding panes
- [x] Task: Implement disable_tab/enable_tab methods
- [x] Task: Implement hide_tab/show_tab methods
- [x] Task: Wire Tabs and ContentSwitcher together via compose()
- [x] Task: Implement TabbedContentActivated message
- [x] Task: Add DEFAULT_CSS (via layout hints - width: 1fr, height: auto)
- [x] Task: Write tests for TabbedContent (7 tests)

### 3.13 Collapsible Widget
- [x] Task: Implement Collapsible struct
- [x] Task: Implement collapsed property
- [x] Task: Implement title property
- [x] Task: Implement CollapsibleTitle header (symbol + label)
- [x] Task: Implement Contents container
- [x] Task: Implement expand() method
- [x] Task: Implement collapse() method
- [x] Task: Implement toggle() method (instant toggle, no animation per research)
- [x] Task: Implement CollapsibleToggled, CollapsibleExpanded, CollapsibleCollapsed messages
- [x] Task: Add keyboard support (CollapsibleTitle is focusable for Enter toggle)
- [x] Task: Add DEFAULT_CSS (via layout hints - width: 1fr, height: auto)
- [x] Task: Write tests for Collapsible (27 tests including Title and Contents)

### 3.14 ContentSwitcher Widget
- [x] Task: Implement ContentSwitcher struct
- [x] Task: Implement current property
- [x] Task: Implement visibility toggling for children (via ChildDisplayOverride trait)
- [x] Task: Implement CurrentChanged message
- [x] Task: Add DEFAULT_CSS (via layout hints - width: 1fr, height: auto)
- [x] Task: Write tests for ContentSwitcher (15 tests)

- [ ] Task: Conductor - User Manual Verification 'Implementation Complete' (Protocol in workflow.md)

## Phase 4: Testing & Verification
- [ ] Task: Create integration tests with nested containers
- [ ] Task: Create example app with tabs, collapsibles, scrolling
- [ ] Task: Test keyboard navigation
- [ ] Task: Test layout_recursive Display::None filtering (docked + content children, ContentSwitcher panes)
- [ ] Task: Test grid_config() via Widget trait (ItemGrid custom config, existing Grid.grid_config())
- [ ] Task: Test CloneableWidget + add_boxed() with downcasting/query paths and Auto sizing
- [ ] Task: Run all tests and fix failures
- [ ] Task: Conductor - User Manual Verification 'Track Complete' (Protocol in workflow.md)
