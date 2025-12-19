# Phase D0 — Test Inventory (Python → Rust)

## CSS Tests (Python)
Ported/covered (partial coverage):
- `textual/tests/css/test_tokenize.py` → `textual-rs/tests/css_tokenizer_tests.rs` (basic tokens only; no variables/scalars/advanced TCSS yet)
- `textual/tests/css/test_parse.py` → `textual-rs/tests/css_stylesheet_parse_tests.rs`, `textual-rs/tests/css_selector_parser_tests.rs`
- `textual/tests/css/test_stylesheet.py` → `textual-rs/tests/css_stylesheet_apply_tests.rs` (specificity/important/cascade basics)
- `textual/tests/css/test_styles.py` → `textual-rs/tests/effective_style_tests.rs` (tint/opacity blend)
- `textual/tests/css/test_scalar.py` → `textual-rs/tests/css_spacing_parse_tests.rs` (length + edge parsing subset)

Not yet ported (requires missing features):
- `test_inheritance.py`, `test_initial.py`, `test_nested_css.py`, `test_screen_css.py`, `test_styles.py` (full), `test_stylesheet.py` (full), `test_programmatic_style_changes.py`
- `test_grid_rows_columns_relative_units.py` (grid), `test_mega_stylesheet.py` (full TCSS), `test_css_reloading.py`, `test_help_text.py`

## Layout/Arrange Tests (Python)
Ported/covered (partial coverage):
- `textual/tests/layouts/test_common_layout_features.py` → `textual-rs/tests/layout_style_tests.rs` (display + basic sizing)
- `textual/tests/layouts/test_content_dimensions.py` → `textual-rs/tests/layout_style_tests.rs` (width/height/min/max)

Not yet ported (missing layouts/arrange pipeline):
- `textual/tests/layouts/test_horizontal.py`
- `textual/tests/layouts/test_grid.py`
- `textual/tests/layouts/test_factory.py`
- `textual/tests/test_layout_resolve.py`
- `textual/tests/test_arrange.py`
- `textual/tests/test_box_model.py`
- `textual/tests/test_spatial_map.py`

## Notes
- Porting strategy: start with tests for parser/tokenizer + simple stylesheet apply; then add layout engine tests as layouts land (vertical → horizontal → grid → dock/layers).
- For tests requiring features not yet implemented, add them only when the corresponding feature is scheduled to land (tests-first).
