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
- [x] Task: Design Container base trait/struct
- [x] Task: Design scroll bar widget integration
- [x] Task: Design Tab/TabPane/TabbedContent relationship
- [x] Task: Plan collapsible animation approach (Result: No animation - instant toggle via display:none)
- [x] Task: Write technical design document
- [ ] Task: Conductor - User Manual Verification 'Design Approved' (Protocol in workflow.md)

## Phase 3: Implementation

### 3.1 Container Base
- [ ] Task: Implement Container struct
- [ ] Task: Implement compose() support for children
- [ ] Task: Implement default vertical layout
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Container

### 3.2 Vertical Container
- [ ] Task: Implement Vertical extending Container
- [ ] Task: Apply VerticalLayout
- [ ] Task: Support gap/spacing property
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Vertical

### 3.3 Horizontal Container
- [ ] Task: Implement Horizontal extending Container
- [ ] Task: Apply HorizontalLayout
- [ ] Task: Support gap/spacing property
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Horizontal

### 3.4 Grid Container
- [ ] Task: Implement Grid extending Container
- [ ] Task: Apply GridLayout
- [ ] Task: Support grid-template-* properties
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Grid

### 3.5 Center Container
- [ ] Task: Implement Center extending Container
- [ ] Task: Center child horizontally and vertically
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Center

### 3.6 Middle Container
- [ ] Task: Implement Middle extending Container
- [ ] Task: Center child vertically only
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for Middle

### 3.7 ScrollableContainer
- [ ] Task: Implement ScrollableContainer struct
- [ ] Task: Integrate vertical ScrollBar widget
- [ ] Task: Integrate horizontal ScrollBar widget
- [ ] Task: Implement scroll_to_widget() method
- [ ] Task: Implement scroll_home() method
- [ ] Task: Implement scroll_end() method
- [ ] Task: Implement scroll_page_up() method
- [ ] Task: Implement scroll_page_down() method
- [ ] Task: Add keyboard scrolling bindings
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for ScrollableContainer

### 3.8 VerticalScroll Container
- [ ] Task: Implement VerticalScroll extending ScrollableContainer
- [ ] Task: Show only vertical scroll bar
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for VerticalScroll

### 3.9 HorizontalScroll Container
- [ ] Task: Implement HorizontalScroll extending ScrollableContainer
- [ ] Task: Show only horizontal scroll bar
- [ ] Task: Add DEFAULT_CSS
- [ ] Task: Write tests for HorizontalScroll

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
- [ ] Task: Run all tests and fix failures
- [ ] Task: Conductor - User Manual Verification 'Track Complete' (Protocol in workflow.md)
