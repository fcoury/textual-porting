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
- [ ] Task: Implement TabPane struct
- [ ] Task: Implement title property
- [ ] Task: Implement disabled property
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for TabPane

### 3.11 Tabs Widget
- [ ] Task: Implement Tabs struct (tab bar)
- [ ] Task: Implement active property
- [ ] Task: Implement tab rendering
- [ ] Task: Implement active tab styling
- [ ] Task: Implement keyboard navigation (arrows)
- [ ] Task: Implement TabActivated message
- [ ] Task: Implement Cleared message
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Tabs

### 3.12 TabbedContent Widget
- [ ] Task: Implement TabbedContent struct
- [ ] Task: Implement active property
- [ ] Task: Implement add_pane() method
- [ ] Task: Implement remove_pane() method
- [ ] Task: Implement clear_panes() method
- [ ] Task: Implement get_tab() method
- [ ] Task: Implement get_pane() method
- [ ] Task: Wire Tabs and TabPanes together
- [ ] Task: Implement TabActivated message
- [ ] Task: Implement Cleared message
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for TabbedContent

### 3.13 Collapsible Widget
- [ ] Task: Implement Collapsible struct
- [ ] Task: Implement collapsed reactive property
- [ ] Task: Implement title property
- [ ] Task: Implement collapsible header rendering
- [ ] Task: Implement expand() method
- [ ] Task: Implement collapse() method
- [ ] Task: Implement toggle() method (instant toggle, no animation per research)
- [ ] Task: Implement Toggled message
- [ ] Task: Add keyboard support (Enter to toggle)
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Collapsible

### 3.14 ContentSwitcher Widget
- [ ] Task: Implement ContentSwitcher struct
- [ ] Task: Implement current property
- [ ] Task: Implement visibility toggling for children
- [ ] Task: Implement CurrentChanged message
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for ContentSwitcher

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
