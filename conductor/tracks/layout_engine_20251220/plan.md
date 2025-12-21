# Plan: Layout Engine

## Phase 1: Geometry & Constraints
- [ ] Task: Define `Size` (width/height) and `Constraint` enum (Length, Percentage, Auto)
- [ ] Task: Implement `Rect` logic (intersection, union, containment)
- [ ] Task: Conductor - User Manual Verification 'Geometry' (Protocol in workflow.md)

## Phase 2: The Layout Solver
- [ ] Task: Implement the `Dock` layout algorithm
- [ ] Task: Write tests for complex nested docking scenarios
- [ ] Task: Conductor - User Manual Verification 'Layout Solver' (Protocol in workflow.md)

## Phase 3: Integration
- [ ] Task: Add `layout()` method to `Widget` trait
- [ ] Task: Integrate layout pass into `App::run` loop
- [ ] Task: Conductor - User Manual Verification 'Layout Integration' (Protocol in workflow.md)
