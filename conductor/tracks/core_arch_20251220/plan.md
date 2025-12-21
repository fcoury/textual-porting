# Plan: Core Architecture

## Phase 1: Widget Registry [checkpoint: 39a20ab]
- [x] Task: Add `slotmap` to `Cargo.toml`
- [x] Task: Define `WidgetId` and `Widget` trait foundation
- [x] Task: Implement `WidgetRegistry` with basic CRUD operations
- [x] Task: Conductor - User Manual Verification 'Widget Registry' (Protocol in workflow.md)

## Phase 2: The Tree Structure [checkpoint: e07c297]
- [x] Task: Implement `TreeNode` struct to manage parent/child ID relationships separate from widget data
- [x] Task: Integrate `TreeNode` into `WidgetRegistry` to form a graph
- [x] Task: Conductor - User Manual Verification 'Tree Structure' (Protocol in workflow.md)

## Phase 3: Message Bus [checkpoint: e16dd6c]
- [x] Task: Define `Message` envelope and `Callback` types
- [x] Task: Implement `post_message` with bubbling logic (traversing up the tree)
- [x] Task: Conductor - User Manual Verification 'Message Bus' (Protocol in workflow.md)
