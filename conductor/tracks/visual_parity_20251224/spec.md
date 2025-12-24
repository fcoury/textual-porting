# Specification: Visual Parity - Rendering and Themes

## Overview

This track brings textual-rs to visual parity with Python Textual across all built-in widgets and example apps. The goal is not only feature parity but also the "beautiful UI" that is a hallmark of Textual: colors, borders, typography, spacing, and alignment should match the Python reference.

## Problem Statement and Root Cause Analysis

The calculator example highlights several gaps between Python Textual and textual-rs. The differences are not cosmetic-only; they originate from structural gaps in the rendering and layout pipeline.

Root causes identified from code and assets:

1. **Examples bypass the widget + TCSS pipeline**
   - Python calculator uses widgets (Digits, Button, Container) plus TCSS (`textual/examples/calculator.py`, `textual/examples/calculator.tcss`).
   - Rust calculator renders directly with ratatui blocks (`textual-rs/examples/calculator.rs`), so TCSS, themes, and widget defaults are never used.

2. **Computed styles are not fully applied at render time**
   - `ComputedStyle::to_ratatui_style()` only maps fg/bg + modifiers.
   - Properties used by Python defaults are ignored at paint time: `border-top`, `border-bottom`, `tint`, `background-tint`, `text-opacity`, `line-pad`, `content-align`, `text-align`.

3. **Border model mismatch**
   - Python Button defaults use tall per-edge borders and tinting to create an embossed effect (`textual/src/textual/widgets/_button.py`).
   - Rust buttons use ratatui `Block` borders, which are uniform, thin, and do not support per-edge styles or thickness.

4. **Theme and variable mismatches**
   - Python uses `Theme` + `ColorSystem` (`textual/src/textual/theme.py`, `textual/src/textual/design.py`) with derived variables and tuned palettes.
   - Rust theme registry is similar but not guaranteed to match Python values or derived variable formulas, leading to dull or shifted colors.

5. **Digits renderable mismatch**
   - Python `Digits` uses a custom 3x3 glyph font (`textual/src/textual/renderables/digits.py`).
   - Rust `Digits` uses different patterns (`textual-rs/src/widgets/digits.rs`) and the calculator example does not use the widget at all.

6. **Vertical alignment and padding not implemented**
   - Python uses `content-align: center middle` and `line-pad: 1` for button and digits alignment.
   - ratatui `Paragraph` has no vertical alignment, so Rust content snaps to the top.

7. **Pseudo-class semantics not equivalent**
   - Python uses `:focus`, `:hover`, `:active`, `:disabled` in DEFAULT_CSS.
   - Rust uses class toggles (.-focused, .-disabled) as a workaround, which does not match Python selectors or cascade behavior.

8. **Layout and CSS integration gaps**
   - CSS grid settings and margins/padding in `calculator.tcss` (grid-gutter, grid-rows, column-span, etc.) are not driving layout in Rust.
   - Layout currently relies on `LayoutHints` rather than computed CSS properties.

## Goals

1. Match Python Textual visuals (colors, borders, digits, alignment) for all built-in widgets.
2. Ensure examples are built from widgets + TCSS, not ad-hoc ratatui rendering.
3. Make rendering respect the full computed style surface used by Python DEFAULT_CSS.
4. Establish repeatable visual regression tests and manual verification steps.

## Non-Goals

- Replacing ratatui or crossterm.
- Implementing terminal font rendering or true alpha blending.
- Adding new widgets beyond parity with Python.

## Functional Requirements

### FR-1: Example parity pipeline
All examples must use the widget tree, layout engine, and TCSS. Direct ratatui rendering is not allowed for parity-sensitive examples.

### FR-2: Style-to-layout integration
Computed CSS properties must drive layout hints and container configuration:
- width/height/min/max
- margin/padding
- grid configuration (grid-rows, grid-columns, grid-gutter, column-span, row-span)
- display/visibility (if used in Python)

### FR-3: Render pipeline supports full style surface
Rendering must apply:
- per-edge borders (including tall/double variants)
- border colors per edge
- background tint and tint
- text-opacity and opacity
- content-align (horizontal + vertical)
- line-pad (left/right padding for each line of text)

### FR-4: Theme parity
Built-in themes and derived variables must match Python Textual, including:
- palette values
- derived variables such as surface-lighten-1, primary-darken-2, etc.
- auto colors resolved against parent background

### FR-5: Digits parity
Digits rendering must match Python's glyphs and alignment, including:
- 3x3 digits font (regular + bold if applicable)
- punctuation mapping (e.g., dot to bullet)
- consistent spacing and alignment

### FR-6: Pseudo-class support
TCSS parser and matcher must support pseudo-classes used in Python defaults:
`:focus`, `:hover`, `:active`, `:disabled`, `:inline`.

### FR-7: Widget default CSS parity
DEFAULT_CSS for built-in widgets should match Python rules and selectors as closely as possible, including variant-specific rules for Button and Input.

### FR-8: Visual regression tests
Create tests and tooling to compare visuals across widgets/examples:
- snapshot tests for widget buffers (insta)
- manual screenshot capture procedures for major examples

## Acceptance Criteria

1. Calculator example renders visually comparable to Python Textual:
   - colors match theme palette
   - buttons show embossed border effect
   - digits match Python glyphs and alignment
   - vertical and horizontal alignment matches
2. Built-in widgets (Button, Label, Input, Digits, SelectionList, etc.) match Python defaults in snapshots.
3. Theme variables from Python resolve to the same RGB values in Rust.
4. TCSS properties listed in FR-3 are used in rendering, not ignored.
5. All parity-related tests pass and manual verification steps are documented.

## References

- Python calculator: `textual/examples/calculator.py`
- Python calculator CSS: `textual/examples/calculator.tcss`
- Python Button defaults: `textual/src/textual/widgets/_button.py`
- Python Digits renderable: `textual/src/textual/renderables/digits.py`
- Python themes: `textual/src/textual/theme.py`, `textual/src/textual/design.py`
- Rust calculator: `textual-rs/examples/calculator.rs`
- Rust widgets: `textual-rs/src/widgets/*.rs`
