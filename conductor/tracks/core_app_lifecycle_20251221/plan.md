# Plan: Core App Lifecycle

## Phase 1: Analysis & Research
- [ ] Task: Study Python Textual app.py compose() and ComposeResult implementation
- [ ] Task: Study reactive.py for reactive property implementation (foundational)
- [ ] Task: Analyze message_pump.py for async dispatch patterns
- [ ] Task: Document screen.py screen stack and modal behavior
- [ ] Task: Map binding.py key binding to action dispatch flow
- [ ] Task: Study layout integration in widget lifecycle
- [ ] Task: Conductor - User Manual Verification 'Analysis Complete' (Protocol in workflow.md)

## Phase 2: Design & Planning
- [ ] Task: Design reactive property derive macro (needed by all systems)
- [ ] Task: Design Rust compose pattern (iterator approach vs builder pattern)
- [ ] Task: Design message pump architecture with tokio channels
- [ ] Task: Design screen stack with Rc/RefCell or arena allocation
- [ ] Task: Design DOM query API with CSS selector support
- [ ] Task: Design layout integration points
- [ ] Task: Write technical design document with API examples
- [ ] Task: Conductor - User Manual Verification 'Design Approved' (Protocol in workflow.md)

## Phase 3: Implementation

### 3.1 Compose Pattern
- [ ] Task: Implement ComposeResult iterator type
- [ ] Task: Add compose() method to App trait
- [ ] Task: Add compose() method to Widget trait
- [ ] Task: Implement automatic mounting of composed children
- [ ] Task: Create Container widget for grouping
- [ ] Task: Write tests for compose pattern

### 3.2 Reactive Properties (Foundation - many systems depend on this)
- [ ] Task: Implement reactive! macro or #[derive(Reactive)]
- [ ] Task: Implement watch_* method pattern
- [ ] Task: Trigger automatic refresh on property change
- [ ] Task: Support computed reactive values
- [ ] Task: Write tests for reactive updates

### 3.3 Mount/Unmount System
- [ ] Task: Implement mount() method on widgets
- [ ] Task: Implement unmount() method on widgets
- [ ] Task: Add on_mount() lifecycle callback
- [ ] Task: Add on_unmount() lifecycle callback
- [ ] Task: Track parent/child relationships in widget tree
- [ ] Task: Write tests for mount/unmount lifecycle

### 3.4 Layout Integration
- [ ] Task: Define layout trigger points in widget lifecycle
- [ ] Task: Implement layout invalidation on mount/unmount
- [ ] Task: Implement layout invalidation on reactive property change
- [ ] Task: Integrate with Layout System track's compute_layout
- [ ] Task: Write tests for layout integration

### 3.5 Screen Stack
- [ ] Task: Implement Screen base type
- [ ] Task: Implement push_screen() method
- [ ] Task: Implement pop_screen() method
- [ ] Task: Implement switch_screen() method
- [ ] Task: Add screen result callback mechanism
- [ ] Task: Write tests for screen stack operations

### 3.6 Message Pump
- [ ] Task: Define Message trait with metadata
- [ ] Task: Implement post_message() for async dispatch
- [ ] Task: Implement message handler lookup
- [ ] Task: Implement event bubbling through widget tree
- [ ] Task: Add message capture phase
- [ ] Task: Write tests for message dispatch

### 3.7 Bindings & Actions
- [ ] Task: Define Binding struct (key, action, description, priority)
- [ ] Task: Implement BINDINGS constant pattern
- [ ] Task: Implement action dispatch via action_* methods
- [ ] Task: Integrate bindings with Footer widget
- [ ] Task: Write tests for key binding dispatch

### 3.8 DOM Queries
- [ ] Task: Implement query() method returning iterator
- [ ] Task: Implement query_one() for single widget lookup
- [ ] Task: Add CSS selector parsing for queries
- [ ] Task: Implement type-safe widget casting
- [ ] Task: Write tests for DOM queries

- [ ] Task: Conductor - User Manual Verification 'Implementation Complete' (Protocol in workflow.md)

## Phase 4: Testing & Verification
- [ ] Task: Create integration test with full app lifecycle
- [ ] Task: Create example app demonstrating all features
- [ ] Task: Run all tests and fix any failures
- [ ] Task: Manual testing of screen transitions and message flow
- [ ] Task: Conductor - User Manual Verification 'Track Complete' (Protocol in workflow.md)
