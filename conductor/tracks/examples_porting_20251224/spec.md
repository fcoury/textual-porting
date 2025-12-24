# Specification: Python Examples Porting and Cleanup

## Overview

This track ports all Python Textual examples into textual-rs and cleans up obsolete Rust examples. The goal is to ensure that every canonical Python example has a Rust counterpart that uses the widget tree, layout engine, and TCSS pipeline, and that our example set is coherent, up-to-date, and parity-focused.

## Scope

### In Scope

- Port examples from:
  - `textual/examples/*.py` (and associated `.tcss`, data files)
  - `textual/src/textual/demo/*.py`
  - `textual/docs/examples/**` (where examples are not duplicates of the above)
- Ensure each port:
  - Uses widget tree + TCSS + StyleManager
  - Matches Python behaviors and visuals as closely as practical
  - Includes integration tests or snapshot tests where appropriate
- Clean up obsolete or redundant Rust examples in `textual-rs/examples`.

### Out of Scope

- Rewriting core layout/rendering systems beyond what the examples require
- Adding new features not present in Python Textual examples
- Replacing ratatui or crossterm

## Functional Requirements

### FR-1: Example Inventory and Mapping
- Maintain an inventory of all Python examples and their Rust counterparts.
- Record assets (TCSS, JSON, images) required by each example.

### FR-2: Porting Standards
Each ported example must:
- Use widget tree composition and TCSS, not ad-hoc ratatui rendering.
- Register widget defaults and load example CSS into StyleManager.
- Use the render pipeline with RenderContext.
- Include input handling consistent with Python example behavior.

### FR-3: Tests and Parity Checks
- Add integration or snapshot tests for each example.
- Tests must validate:
  - Key widget presence (text and layout)
  - Style application for at least one key element (bg/fg/border)
  - Critical interaction flows (keyboard and mouse where applicable)

### FR-4: Example Cleanup
- Audit existing Rust examples and classify:
  - Keep (aligned with Python examples)
  - Replace (obsolete but still useful)
  - Archive (no longer needed)
- Remove or archive obsolete examples and document the decision.

## Acceptance Criteria

1. All examples in `textual/examples` have Rust ports or are explicitly documented as deferred.
2. All demo examples in `textual/src/textual/demo` have Rust ports or are explicitly documented as deferred.
3. Doc examples are either ported or mapped to existing Rust examples with explicit rationale.
4. Each ported example has tests with style assertions and renders via TCSS pipeline.
5. Rust examples directory is cleaned and documented to avoid redundant or obsolete samples.

## References

- Python examples: `textual/examples/`
- Python demo app: `textual/src/textual/demo/`
- Python docs examples: `textual/docs/examples/`
- Rust examples: `textual-rs/examples/`
