# Plan: Styling System

## Phase 1: TCSS Parser
- [x] Task: Add `nom` or `pest` to Cargo.toml [6a38ef5]
- [x] Task: Implement parser for CSS selectors (Class, ID, Type) [c1b7446]
- [x] Task: Implement parser for basic declarations (color, background) [c67b2eb]
- [ ] Task: Conductor - User Manual Verification 'TCSS Parser' (Protocol in workflow.md)

## Phase 2: Cascading Logic
- [ ] Task: Implement `Specificity` calculation for selectors
- [ ] Task: Implement `StyleSheet` struct to hold rules
- [ ] Task: Implement `compute_style(widget_id, registry, stylesheet)`
- [ ] Task: Conductor - User Manual Verification 'Cascading' (Protocol in workflow.md)
