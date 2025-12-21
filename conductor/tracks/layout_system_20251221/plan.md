# Plan: Layout System

## Phase 1: Analysis & Research
- [x] Task: Study Python Textual layouts/vertical.py and horizontal.py [4723bc9]
- [x] Task: Analyze layouts/grid.py for CSS Grid implementation [f25fd38]
- [x] Task: Study _arrange.py for the arrangement algorithm [c5d2c4c]
- [x] Task: Document scroll_view.py and scrollbar.py implementation [63fdbe2]
- [x] Task: Map CSS size properties (width, height, min-*, max-*, fr units) [08bb1f2]
- [x] Task: Conductor - User Manual Verification 'Analysis Complete' (Protocol in workflow.md) [9d55906]

## Phase 2: Design & Planning
- [x] Task: Design Layout trait with arrange() signature [9642187]
- [x] Task: Design constraint solving algorithm for flex distribution [84bacfb]
- [x] Task: Design ScrollView architecture with virtual content [b9dd104]
- [~] Task: Design viewport culling strategy
- [ ] Task: Write technical design document with diagrams
- [ ] Task: Conductor - User Manual Verification 'Design Approved' (Protocol in workflow.md)

## Phase 3: Implementation

### 3.1 Layout Trait Foundation
- [ ] Task: Define Layout trait with arrange() method
- [ ] Task: Define LayoutResult with widget placements
- [ ] Task: Implement layout selection based on widget type
- [ ] Task: Write tests for Layout trait

### 3.2 Vertical Layout
- [ ] Task: Implement VerticalLayout struct
- [ ] Task: Implement top-to-bottom stacking
- [ ] Task: Handle min/max height constraints
- [ ] Task: Implement flex space distribution
- [ ] Task: Write tests for vertical layout

### 3.3 Horizontal Layout
- [ ] Task: Implement HorizontalLayout struct
- [ ] Task: Implement left-to-right stacking
- [ ] Task: Handle min/max width constraints
- [ ] Task: Implement flex space distribution
- [ ] Task: Write tests for horizontal layout

### 3.4 Grid Layout
- [ ] Task: Implement GridLayout struct
- [ ] Task: Implement grid template parsing (columns, rows)
- [ ] Task: Implement explicit cell placement
- [ ] Task: Implement auto-placement algorithm
- [ ] Task: Implement gap between cells
- [ ] Task: Write tests for grid layout

### 3.5 Container Widgets
- [ ] Task: Implement Vertical container widget
- [ ] Task: Implement Horizontal container widget
- [ ] Task: Implement Grid container widget
- [ ] Task: Wire containers to use appropriate layouts
- [ ] Task: Write tests for container widgets

### 3.6 Scroll View
- [ ] Task: Implement ScrollView widget
- [ ] Task: Implement virtual content size tracking
- [ ] Task: Implement scroll offset management
- [ ] Task: Implement vertical ScrollBar widget
- [ ] Task: Implement horizontal ScrollBar widget
- [ ] Task: Implement scroll_to() and scroll_visible() methods
- [ ] Task: Implement keyboard scrolling (arrows, Page Up/Down)
- [ ] Task: Write tests for scroll behavior

### 3.7 Viewport Management
- [ ] Task: Implement visible region calculation
- [ ] Task: Implement widget culling for off-screen widgets
- [ ] Task: Implement dirty region tracking
- [ ] Task: Write performance tests

- [ ] Task: Conductor - User Manual Verification 'Implementation Complete' (Protocol in workflow.md)

## Phase 4: Testing & Verification
- [ ] Task: Create integration tests with nested layouts
- [ ] Task: Create example app with scrolling grid
- [ ] Task: Run all tests and fix failures
- [ ] Task: Performance test with 1000+ widgets
- [ ] Task: Conductor - User Manual Verification 'Track Complete' (Protocol in workflow.md)
