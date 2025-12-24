# Rust Examples Audit

This audit classifies current `textual-rs/examples` entries as keep/replace/archive, with notes about the intended Python source.

## Current Rust Examples

| Rust Example | Status | Intended Python Source | Notes |
| --- | --- | --- | --- |
| animation_demo.rs | Replace | docs/guide/animator/animation01.py | Keep until animator guide ports land |
| calculator.rs | Keep | textual/examples/calculator.py | Parity target (already ported) |
| container_widgets_demo.rs | Replace | textual/src/textual/demo/widgets.py | Redundant with widget gallery; to be replaced by demo port |
| hello_world.rs | Keep | docs/examples/widgets/hello*.py | Canonical starter example |
| interactive_widgets_demo.rs | Replace | docs/examples/widgets/* | Replace with widget gallery + specific widget examples |
| lifecycle_demo.rs | Replace | docs/examples/app/* | Replace with doc app examples |
| mouse_demo.rs | Replace | docs/examples/guide/input/mouse01.py | Replace with ported input/mouse example |
| scrolling_grid_demo.rs | Replace | docs/examples/guide/layout/grid_layout7_gutter.py | Replace with layout guide examples |
| static_widgets_demo.rs | Replace | docs/examples/widgets/static.py | Replace with widget gallery or docs example |
| styled_app.rs | Replace | docs/examples/app/widgets*.py | Replace with app docs examples |
| theme_demo.rs | Keep | textual/examples/theme_sandbox.py | Theme parity example |
| widget_gallery.rs | Keep | textual/src/textual/demo/widgets.py | Core widget showcase |
| demo.tcss | Keep | N/A (shared asset) | Used by demo-style examples |

## Initial Cleanup Policy

- **Keep**: examples that map directly to Python core or demo parity targets.
- **Replace**: examples that are useful but should be superseded by official Python ports.
- **Archive**: none yet; archive once a direct Python port replaces the Rust example.

## Action Items

- Replace items as their Python equivalents are ported.
- Archive old examples once replacements are validated.
- Maintain a short `examples/README.md` that maps Rust examples to Python sources.
