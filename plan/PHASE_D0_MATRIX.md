# Phase D0 — CSS Property Parity Matrix

Status legend: Supported | Partial | Missing

| Python property | Rust status | Notes |
|---|---|---|
| `display` | Supported | Supported (block/none). |
| `visibility` | Supported | Implemented in DOM + layout render trees (hidden keeps layout space). |
| `layout` | Supported | Added `layout: vertical|horizontal` with horizontal stacking in DOM. |
| `auto_color` | Missing | Not implemented in Rust yet. |
| `color` | Supported | CSS `color` implemented. |
| `background` | Supported | CSS `background` implemented. |
| `text_style` | Supported | CSS `text-style` implemented. |
| `background_tint` | Supported | Supported as `background-tint`. |
| `opacity` | Supported | Supported (applies to fg/bg through effective style). |
| `text_opacity` | Supported | Supported as `text-opacity`. |
| `padding` | Supported | CSS `padding` implemented. |
| `margin` | Supported | CSS `margin` implemented. |
| `offset` | Supported | Implemented for DOM render tree + hit-testing (relative/absolute offsets). |
| `position` | Supported | Relative/absolute positioning in DOM render tree. |
| `border_top` | Missing | Not implemented in Rust yet. |
| `border_right` | Missing | Not implemented in Rust yet. |
| `border_bottom` | Missing | Not implemented in Rust yet. |
| `border_left` | Missing | Not implemented in Rust yet. |
| `border_title_align` | Missing | Not implemented in Rust yet. |
| `border_subtitle_align` | Missing | Not implemented in Rust yet. |
| `outline_top` | Missing | Not implemented in Rust yet. |
| `outline_right` | Missing | Not implemented in Rust yet. |
| `outline_bottom` | Missing | Not implemented in Rust yet. |
| `outline_left` | Missing | Not implemented in Rust yet. |
| `keyline` | Missing | Not implemented in Rust yet. |
| `box_sizing` | Supported | CSS `box-sizing` implemented. |
| `width` | Supported | CSS `width` implemented. |
| `height` | Supported | CSS `height` implemented. |
| `min_width` | Supported | CSS `min-width` implemented. |
| `min_height` | Supported | CSS `min-height` implemented. |
| `max_width` | Supported | CSS `max-width` implemented. |
| `max_height` | Supported | CSS `max-height` implemented. |
| `dock` | Missing | Not implemented in Rust yet. |
| `split` | Missing | Not implemented in Rust yet. |
| `overflow_x` | Supported | Axis-specific overflow now supported. |
| `overflow_y` | Supported | Axis-specific overflow now supported. |
| `layers` | Missing | Missing (Rust uses `z-index` but no named layers). |
| `layer` | Missing | Missing (Rust uses `z-index` but no named layers). |
| `transitions` | Missing | Missing (planned for Phase F). |
| `tint` | Supported | CSS `tint` implemented. |
| `scrollbar_color` | Missing | Not implemented in Rust yet. |
| `scrollbar_color_hover` | Missing | Not implemented in Rust yet. |
| `scrollbar_color_active` | Missing | Not implemented in Rust yet. |
| `scrollbar_corner_color` | Missing | Not implemented in Rust yet. |
| `scrollbar_background` | Missing | Not implemented in Rust yet. |
| `scrollbar_background_hover` | Missing | Not implemented in Rust yet. |
| `scrollbar_background_active` | Missing | Not implemented in Rust yet. |
| `scrollbar_gutter` | Missing | Not implemented in Rust yet. |
| `scrollbar_size_vertical` | Missing | Not implemented in Rust yet. |
| `scrollbar_size_horizontal` | Missing | Not implemented in Rust yet. |
| `scrollbar_visibility` | Missing | Not implemented in Rust yet. |
| `align_horizontal` | Missing | Not implemented in Rust yet. |
| `align_vertical` | Missing | Not implemented in Rust yet. |
| `content_align_horizontal` | Missing | Not implemented in Rust yet. |
| `content_align_vertical` | Missing | Not implemented in Rust yet. |
| `grid_size_rows` | Missing | Not implemented in Rust yet. |
| `grid_size_columns` | Missing | Not implemented in Rust yet. |
| `grid_gutter_horizontal` | Missing | Not implemented in Rust yet. |
| `grid_gutter_vertical` | Missing | Not implemented in Rust yet. |
| `grid_rows` | Missing | Not implemented in Rust yet. |
| `grid_columns` | Missing | Not implemented in Rust yet. |
| `row_span` | Missing | Not implemented in Rust yet. |
| `column_span` | Missing | Not implemented in Rust yet. |
| `text_align` | Missing | Not implemented in Rust yet. |
| `link_color` | Missing | Not implemented in Rust yet. |
| `auto_link_color` | Missing | Not implemented in Rust yet. |
| `link_background` | Missing | Not implemented in Rust yet. |
| `link_style` | Missing | Not implemented in Rust yet. |
| `link_color_hover` | Missing | Not implemented in Rust yet. |
| `auto_link_color_hover` | Missing | Not implemented in Rust yet. |
| `link_background_hover` | Missing | Not implemented in Rust yet. |
| `link_style_hover` | Missing | Not implemented in Rust yet. |
| `auto_border_title_color` | Missing | Not implemented in Rust yet. |
| `border_title_color` | Missing | Not implemented in Rust yet. |
| `border_title_background` | Missing | Not implemented in Rust yet. |
| `border_title_style` | Missing | Not implemented in Rust yet. |
| `auto_border_subtitle_color` | Missing | Not implemented in Rust yet. |
| `border_subtitle_color` | Missing | Not implemented in Rust yet. |
| `border_subtitle_background` | Missing | Not implemented in Rust yet. |
| `border_subtitle_style` | Missing | Not implemented in Rust yet. |
| `hatch` | Missing | Not implemented in Rust yet. |
| `overlay` | Missing | Not implemented in Rust yet. |
| `constrain_x` | Missing | Not implemented in Rust yet. |
| `constrain_y` | Missing | Not implemented in Rust yet. |
| `text_wrap` | Missing | Not implemented in Rust yet. |
| `text_overflow` | Missing | Not implemented in Rust yet. |
| `expand` | Missing | Not implemented in Rust yet. |
| `line_pad` | Missing | Not implemented in Rust yet. |

## Rust-only CSS properties (not in Python RulesMap)

- `z-index` (layering order)
- `background-opacity` (separate from `opacity`/`text-opacity`)

## Parser/selector coverage (notes)

- Nested TCSS blocks with `&` replacement: Supported (basic parent + list expansion).
- Nested selector lists (comma-separated) inside blocks: Supported.
- Line (`# ...`) and block (`/* ... */`) comments: Supported in stylesheet parsing.

## TCSS syntax inventory (docs/examples/tests)

- Selectors: type/class/id/universal, compound selectors, descendant + child combinators, comma-separated selector lists.
- Pseudo-classes seen in `.tcss`: `:focus`, `:hover`, `:enabled`, `:inline`.
- `::` usage appears in `test_mega_stylesheet.tcss` (currently parsed as a type token in Rust; pseudo-element handling is missing).
- Variables: `$var:` declarations + `$var` references (in mega stylesheet + tokenizer tests).
- Values/functions: `rgb(...)`, `hsl(...)`, hex + named colors, `!important`.
- Units/lengths: plain numbers, `%`, `fr`, `vw`, `vh`, `w`, `h`, durations `ms`/`s`, plus keyword `auto`.

## Recent Rust coverage updates

- Color parsing: `hsl(...)`/`hsla(...)` support plus `transparent`, `coral`, `aqua`, `deepskyblue`, `rebeccapurple`.
- Pseudo-class matching: added `:enabled` (still missing `:inline` handling).
