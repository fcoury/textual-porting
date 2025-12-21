# Plan: Layout Engine

## Phase 1: Geometry & Constraints [checkpoint: 22d0538]
- [x] Task: Define `Size` (width/height) and `Constraint` enum (Length, Percentage, Auto) [78c464c]
- [x] Task: Implement `Rect` logic (intersection, union, containment) [9889ca1]
- [x] Task: Conductor - User Manual Verification 'Geometry' (Protocol in workflow.md)

## Phase 2: The Layout Solver
- [x] Task: Implement the `Dock` layout algorithm [8f2b353]
- [x] Task: Write tests for complex nested docking scenarios [e82ae53]
- [ ] Task: Conductor - User Manual Verification 'Layout Solver' (Protocol in workflow.md)

## Phase 3: Integration
- [ ] Task: Add `layout()` method to `Widget` trait
- [ ] Task: Integrate layout pass into `App::run` loop
- [ ] Task: Conductor - User Manual Verification 'Layout Integration' (Protocol in workflow.md)
