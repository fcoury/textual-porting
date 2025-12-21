# Specification: Core Architecture

## Goal
Implement the fundamental "Retained Mode" architecture of `textual-rs`, enabling a persistent tree of widgets that can communicate via messages.

## Requirements
- **WidgetRegistry:** A centralized storage for all widgets using `SlotMap` to manage ownership and ID generation.
- **Widget Trait:** A Rust trait defining the lifecycle (`mount`, `unmount`) and behavior (`render`, `update`) of a widget.
- **Message Bus:** A system to propagate messages:
    - **Bubbling:** From child to parent (e.g., Button click).
    - **Cascading:** From parent to children (e.g., Theme change).

## Success Criteria
- Tests verify widgets can be added/removed from the registry via IDs.
- Tests verify a message sent from a child reaches the parent (bubbling).
- Tests verify a message broadcast from the root reaches all children (cascading).
