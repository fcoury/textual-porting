# Plan: Core App Lifecycle

## Phase 1: Analysis & Research
- [x] Task: Study Python Textual app.py compose() and ComposeResult implementation
- [x] Task: Study reactive.py for reactive property implementation (foundational)
- [x] Task: Analyze message_pump.py for async dispatch patterns
- [x] Task: Document screen.py screen stack and modal behavior
- [x] Task: Map binding.py key binding to action dispatch flow
- [x] Task: Study layout integration in widget lifecycle
- [x] Task: Conductor - User Manual Verification 'Analysis Complete' (Protocol in workflow.md)

## Phase 2: Design & Planning
- [x] Task: Design reactive property derive macro (needed by all systems)
- [x] Task: Design Rust compose pattern (iterator approach vs builder pattern)
- [x] Task: Design message pump architecture with tokio channels
- [x] Task: Design screen stack with Rc/RefCell or arena allocation
- [x] Task: Design DOM query API with CSS selector support
- [x] Task: Design layout integration points
- [x] Task: Write technical design document with API examples
- [x] Task: Conductor - User Manual Verification 'Design Approved' (Protocol in workflow.md)

## Phase 3: Implementation

### 3.1 Compose Pattern
- [x] Task: Implement ComposeContext builder type
- [x] Task: Add Compose trait with compose() method
- [x] Task: Implement add(), add_with(), add_if(), add_iter() methods
- [x] Task: Add walk_descendants() and remove_subtree() to WidgetRegistry
- [x] Task: Create Container widget for grouping
- [x] Task: Write tests for compose pattern

### 3.2 Reactive Properties (Foundation - many systems depend on this)
- [x] Task: Implement ReactiveFlags bitflags
- [x] Task: Implement WidgetRefreshState with Cell-based flags
- [x] Task: Implement Reactive trait
- [x] Task: Implement #[derive(Reactive)] proc macro (separate crate)
- [x] Task: Implement watch_* method pattern in macro
- [x] Task: Write tests for reactive updates

### 3.3 Mount/Unmount System
- [x] Task: Implement mount() method on widgets
- [x] Task: Implement unmount() method on widgets
- [x] Task: Add on_mount() lifecycle callback
- [x] Task: Add on_unmount() lifecycle callback
- [x] Task: Track parent/child relationships in widget tree
- [x] Task: Write tests for mount/unmount lifecycle

### 3.4 Layout Integration
- [x] Task: Define layout trigger points in widget lifecycle
- [x] Task: Implement layout invalidation on mount/unmount
- [x] Task: Implement layout invalidation on reactive property change
- [x] Task: Integrate with Layout System track's compute_layout
- [x] Task: Write tests for layout integration

### 3.5 Screen Stack
- [x] Task: Implement Screen base type
- [x] Task: Implement push_screen() method
- [x] Task: Implement pop_screen() method
- [x] Task: Implement switch_screen() method
- [x] Task: Add screen result callback mechanism
- [x] Task: Write tests for screen stack operations

### 3.6 Message Pump
- [x] Task: Define Message trait with metadata
- [x] Task: Implement post_message() for async dispatch
- [x] Task: Implement message handler lookup
- [x] Task: Implement event bubbling through widget tree
- [x] Task: Add message capture phase
- [x] Task: Write tests for message dispatch

### 3.7 Bindings & Actions
- [x] Task: Define Binding struct (key, action, description, priority)
- [x] Task: Implement BINDINGS constant pattern
- [x] Task: Implement action dispatch via action_* methods
- [x] Task: Integrate bindings with Footer widget
- [x] Task: Write tests for key binding dispatch

### 3.8 DOM Queries
- [x] Task: Implement query() method returning iterator
- [x] Task: Implement query_one() for single widget lookup
- [x] Task: Add CSS selector parsing for queries
- [x] Task: Implement type-safe widget casting
- [x] Task: Write tests for DOM queries

- [x] Task: Conductor - User Manual Verification 'Implementation Complete' (Protocol in workflow.md)

## Phase 4: Testing & Verification
- [x] Task: Create integration test with full app lifecycle
- [x] Task: Create example app demonstrating all features
- [x] Task: Run all tests and fix any failures
- [x] Task: Manual testing of screen transitions and message flow
- [x] Task: Conductor - User Manual Verification 'Track Complete' (Protocol in workflow.md)

## Verification Report

**Track**: Core App Lifecycle (Tier 1)
**Date**: 2025-12-21
**Status**: COMPLETE

### Automated Tests
```
cargo test --all -- --test-threads=1
403 tests passed (387 unit + 9 app_lifecycle + 7 reactive_derive)
```

### Manual Verification
- [x] Built and ran `lifecycle_demo` example
- [x] Verified counter increment/decrement with `+`/`=` and `-`/`_` keys
- [x] Verified footer toggle with `h` key (mount/unmount lifecycle)
- [x] Verified quit with `q` key
- [x] Verified reactive property updates trigger layout recalculation
- [x] User confirmed verification meets expectations

### Key Bindings Design Note
The `+,=` binding pattern allows both keys to work, following Python Textual's UX pattern where `=` is provided as an easier alternative since `+` requires Shift on most keyboards.

[checkpoint: 6ccd16f]
