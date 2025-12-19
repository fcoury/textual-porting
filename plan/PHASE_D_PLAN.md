# Phase D Plan — CSS + Layout Parity (Textual → Rust)

## Current Rust Status (today)
- CSS core:
  - Tokenizer + selector parser: type/class/id/universal, descendent + child combinators, pseudo-class tokens.
  - Stylesheet apply with cascade order + !important.
  - Values: colors (named/hex/rgb/rgba), numbers/percent, lengths (cells/px, %, vw/vh, fr), edge sizes.
  - Properties implemented: color/background, text-style flags, margin/padding, width/height/min/max, box-sizing, display, overflow, opacity/text-opacity/background-opacity, tint/background-tint, z-index.
- Layout:
  - Basic vertical flow for widgets + DOM.
  - Box sizing (border-box/content-box), margin/padding, width/height/min/max.
  - Overflow clip for render tree + hit-testing.
- Rendering integration:
  - z-index ordering (compositor + hit-test ordering).
- Runtime/driver:
  - Input polling tolerates EINTR from SIGWINCH (resize) without crashing.

## Parity Target (Python Textual)
Python references for Phase D:
- CSS engine: `textual/src/textual/css/*`
- Layout + arrangement: `textual/src/textual/layout.py`, `_arrange.py`, `_layout_resolve.py`, `_spatial_map.py`, `layouts/*`
- Box model + geometry: `textual/src/textual/box_model.py`, `geometry.py`

Key feature families to reach parity (from `textual/css/styles.py` RulesMap and layout stack):
- CSS parsing: full TCSS grammar, selectors, specificity, variables (if present), imports, comments/strings/escapes.
- Style properties: visibility, layout selection, borders/outlines, alignment, dock/split, layers, overflow-x/y, position/offset/absolute, scrollbars, grid, text align/wrap/overflow, transitions.
- Layouts: vertical, horizontal, grid, stream, dock, layers, absolute/fixed/overlay placements.

## Work Breakdown (Detailed)

### D0 — Inventory + Parity Matrix
- Build a property matrix that maps Python RulesMap → Rust coverage.
  - Source of truth: `textual/src/textual/css/styles.py` + `_style_properties.py`.
  - Output: checklist of properties with parsing + computed + layout/render usage.
- Confirm TCSS syntax features used in docs/examples/tests (selectors, units, functions, variables).

### D1 — CSS Tokenizer/Parser Parity
- Tokenizer:
  - Preserve existing tokens; add full TCSS features used in Textual tests/docs.
  - Handle strings, escapes, quoted identifiers, and robust comments.
- Selectors:
  - Attribute selectors (if used), :nth-* or structural pseudos (if used), multi‑selectors + combinators.
  - Maintain specificity parity with Python (Specificity3/Specificity6 if needed).
- Values:
  - Extend `CssValue` to cover Textual scalars and units (%, fr, auto, content, min/max, etc.).
  - Add parsing for scalar lists (e.g., grid rows/columns) and composite values.

### D2 — Style System Parity
- Implement a `Styles`/`Rules` model that mirrors Python’s RulesMap semantics:
  - visibility, layout, display
  - offset/position/absolute
  - border/outline/keyline + titles/subtitles
  - alignment (align/content_align)
  - dock/split/layers/layer
  - overflow‑x/y (not just `overflow`)
  - scrollbars (visibility/gutter/size/colors)
  - text align/wrap/overflow
  - grid config (rows/columns, gutters, spans)
  - transitions (definition only; runtime in Phase F)
- Ensure inheritance + default values match Python where applicable.

### D3 — Layout Engine Parity
- Core layout resolve:
  - Implement `Layout` abstraction and arrange pipeline similar to `layout.py` + `_arrange.py`.
  - Introduce `WidgetPlacement` equivalent (region + margin + order + fixed + overlay + absolute).
  - Add spatial map for hit-testing, focus movement, and visibility culling.
- Layouts:
  - Vertical layout: refine to parity (spacing, margins, scroll spacing).
  - Horizontal layout.
  - Grid layout (rows/columns/gutters, spans).
  - Stream layout (flow + wrap if Textual uses).
- Docking + layering:
  - Dock (top/right/bottom/left/none) with remaining content area.
  - Layered rendering order (layer name + z-index).

### D4 — Box Model + Sizing
- Add border + outline to box model (spacing impact + rendering).
- Implement `offset`, `position` (relative/absolute), and constrain rules.
- Expand/greedy sizing and fractional sizing in layout.
- Ensure min/max constraints applied at correct stage.

### D5 — Rendering + Clip Integration
- Align render tree construction with layout placements (including margins/overlays).
- Clipping per overflow-x/y and scrollbars.
- Ensure compositor respects layer ordering (layers + z-index + overlay).

### D6 — Tests to Port for Parity
Prioritize these Python tests (or their distilled equivalents):
- CSS:
  - `textual/tests/css/*`
  - `test_style_parse.py`, `test_style_importance.py`, `test_style_inheritance.py`, `test_style_properties.py`, `test_rule.py`
- Layout:
  - `tests/layouts/*`, `test_layout_resolve.py`, `test_arrange.py`, `test_box_model.py`, `test_spatial_map.py`
- Rendering/layout integration:
  - `test_compositor.py`, `test_compositor_regions_to_spans.py`

## Suggested Sequence (Tests‑First)
1. D0: property matrix + test inventory.
2. D1: tokenizer/value extensions + selector parity (port CSS tests).
3. D2: expand Rules/Styles and apply cascade (tests from style_*).
4. D3: layouts + arrange pipeline (tests from layouts/ and arrange).
5. D4: box model + sizing edge cases (box_model tests).
6. D5: rendering/clip integration tests.

## Definition of Done for Phase D
- All CSS/layout tests ported (or equivalent) and green.
- Layouts (vertical/horizontal/grid/stream/dock/layers) match Textual behavior for core cases.
- Stylesheet supports major properties used by builtin widgets and examples.
