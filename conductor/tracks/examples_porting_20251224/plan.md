# Plan: Python Examples Porting and Cleanup

## Phase 1: Inventory and Standards [checkpoint: b7ffb5d]
- [x] Task: Create `design/example_inventory.md` listing all Python examples (examples, demo, docs) and required assets (03e83b3)
- [x] Task: Create `design/rust_examples_audit.md` to classify Rust examples as keep/replace/archive (1b06e30)
- [x] Task: Define porting template and conventions in `design/example_template.md` (StyleManager + TCSS + RenderContext) (c014a27)
- [x] Task: Define test harness conventions for example parity tests (snapshot + style assertions) (4d70db0)
- [x] Task: Conductor - User Manual Verification 'Inventory Complete' (Protocol in workflow.md) (b7ffb5d)

## Phase 2: Port `textual/examples` (Core Examples)
- [~] Task: Port layout/interaction examples (breakpoints, clock, five_by_five)
- [ ] Task: Port data/text examples (dictionary, markdown, json_tree, code_browser)
- [ ] Task: Port showcase examples (merlin, mother, pride, sidebar, splash, theme_sandbox, color_command)
- [ ] Task: Add integration/snapshot tests for each ported example with style assertions
- [ ] Task: Conductor - User Manual Verification 'Core Examples Ported' (Protocol in workflow.md)

## Phase 3: Port `textual/src/textual/demo` (Demo App)
- [ ] Task: Port `demo_app.py` shell with navigation and page routing
- [ ] Task: Port demo pages (home, projects, page, widgets)
- [ ] Task: Port demo game example (game.py)
- [ ] Task: Port demo data utilities if required by pages
- [ ] Task: Add demo integration tests and snapshot coverage
- [ ] Task: Conductor - User Manual Verification 'Demo Ported' (Protocol in workflow.md)

## Phase 4: Port `textual/docs/examples` (Docs Examples)
- [ ] Task: Inventory docs examples and map to existing Rust examples (skip duplicates)
- [ ] Task: Port unique docs examples into `textual-rs/examples` or `textual-rs/tests/docs_examples`
- [ ] Task: Add minimal snapshot tests for each docs example
- [ ] Task: Conductor - User Manual Verification 'Docs Examples Ported' (Protocol in workflow.md)

## Phase 5: Cleanup and Verification
- [ ] Task: Archive or remove obsolete Rust examples based on audit
- [ ] Task: Update `textual-rs/examples/README.md` with current example list and purpose
- [ ] Task: Update visual parity verification docs with new example list
- [ ] Task: Run full test suite and resolve failures
- [ ] Task: Conductor - User Manual Verification 'Track Complete' (Protocol in workflow.md)
