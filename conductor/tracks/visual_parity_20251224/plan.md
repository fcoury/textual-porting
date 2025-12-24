# Plan: Visual Parity - Rendering and Themes

## Phase 1: Baseline and Audit
- [x] Task: Inventory Python examples and their TCSS files (calculator, demo apps) and record reference paths
- [ ] Task: Capture baseline visual outputs for Python examples (manual steps + tmux capture checklist)
- [x] Task: Diff Python DEFAULT_CSS vs Rust DEFAULT_CSS for built-in widgets; record gaps
- [x] Task: Enumerate CSS properties used by Python defaults/examples and mark status in Rust (parsed, computed, rendered)
- [x] Task: Compare Python theme palettes and derived variables vs Rust ThemeRegistry; record mismatches
- [ ] Task: Conductor - User Manual Verification "Analysis Complete" (Protocol in workflow.md)

**Audit Summary:** See [design/css_parity_audit.md](./design/css_parity_audit.md)

**Python Examples with TCSS:**
- calculator.py / calculator.tcss
- five_by_five.py / five_by_five.tcss
- dictionary.py / dictionary.tcss
- code_browser.py / code_browser.tcss

**Critical Gaps Identified:**
- P0: `content-align`, `line-pad`, Digits glyph patterns (rounded corners)
- P1: `border: block`, `tint`, pseudo-class selectors (`:hover`, `:focus`), SCSS nesting
- P2: Grid CSS properties, `text-opacity`, `background-tint`, theme derivation

## Phase 2: Design and Parity Targets
- [x] Task: Design style-to-layout mapping (width/height/min/max, margin/padding, grid-gutter, column-span, row-span)
- [x] Task: Design border rendering pipeline with per-edge styles and tall/double borders
- [x] Task: Design content alignment and line-pad rendering behavior (vertical + horizontal)
- [x] Task: Design tint/background-tint and text-opacity application order
- [x] Task: Design pseudo-class selector support and widget state mapping
- [x] Task: Design Digits parity approach (glyph map, spacing, bold variants)
- [x] Task: Write technical design document for visual parity integration
- [x] Task: Conductor - User Manual Verification "Design Approved" (Protocol in workflow.md)

**Design Document:** See [design/visual_parity_design.md](./design/visual_parity_design.md)

**Key Design Decisions:**
1. `LayoutHints::from_computed_style()` bridges CSS dimensions to layout system
2. Per-edge borders stored in ComputedStyle, rendered independently
3. `content-align` applied at render time via `align_content()` helper
4. `line-pad` applied to text content, not layout
5. Pseudo-classes added to Selector AST and Widget trait
6. Digits use Python's rounded corner patterns (`╭─╮`)
7. `tint`/`text-opacity` applied via color blending functions

## Phase 3: Core Rendering and Theme Parity
- [x] Task: Write snapshot tests for border rendering (per-edge colors, tall borders)
- [x] Task: Implement per-edge border rendering in the paint layer
- [x] Task: Write snapshot tests for line-pad/content-align vertical alignment
- [x] Task: Implement line-pad and vertical content alignment support in render pipeline
- [x] Task: Write tests for tint/background-tint/text-opacity application order
- [x] Task: Implement tint/background-tint/text-opacity in style resolution or render pipeline
- [x] Task: Write tests for style-driven layout hints
- [x] Task: Integrate computed style into layout (margin/padding/size/display/grid config)
- [x] Task: Write tests for pseudo-class selector matching
- [x] Task: Implement pseudo-class parsing and matching in TCSS (focus/hover/active/disabled/inline)
- [x] Task: Write tests for theme parity variables (textual-dark/light)
- [x] Task: Port Python theme palettes and derived variable rules to Rust
- [x] Task: Write tests for Digits glyph parity
- [x] Task: Update Digits glyph patterns and rendering to match Python
- [x] Task: Conductor - User Manual Verification "Core Rendering Parity Complete" (Protocol in workflow.md)

**Completed in this session:**
- Added `AlignHorizontal`, `AlignVertical` enums with `Left/Center/Right` and `Top/Middle/Bottom` values
- Added `ContentAlign`, `ContentAlignHorizontal`, `ContentAlignVertical`, `LinePad` Declaration variants
- Added `content_align_horizontal`, `content_align_vertical`, `line_pad` fields to ComputedStyle
- Added parsing for `content-align`, `content-align-horizontal`, `content-align-vertical`, `line-pad` properties
- Added 17 new tests for content-align and line-pad parsing, computation, and cascading
- Updated Digits widget glyph patterns to match Python Textual's DIGITS3X3 (rounded corners)
- Added `+` pattern support to Digits widget

**Review Fixes Applied:**
- Fixed content-align defaults: `AlignHorizontal::Left` and `AlignVertical::Top` (was Center/Middle)
- Fixed Digits spacing: Removed extra space between glyphs, all patterns are fixed 3-cell width
- Added Digits bold glyph variant: `DIGIT_PATTERNS_BOLD` with heavy box-drawing characters
- Added `with_bold()`, `bold()`, `set_bold()` methods to Digits widget
- Fixed line-pad documentation: Correctly describes as horizontal padding (left/right per line)
- Fixed Digits period/unknown char width: `.` and unknown chars are 1 cell (matching Python), not 3
  - Removed `.` from `get_pattern`, now falls through to 1-cell fallback
  - Added `glyph_width()` and `normalize_char()` functions
  - Fallback renders normalized char on bottom row only (`.` → `•`)
  - "3.14" now correctly = 10 cells (3+1+3+3), not 12

**Border Rendering Infrastructure:**
- Added `BorderChars` struct to style.rs with character mappings for all BorderKind variants
- Added `Corner` enum (TopLeft, TopRight, BottomLeft, BottomRight)
- Added `BorderChars::for_kind()` returning character sets for each border style
- Added `BorderChars::corner_for()` for corner resolution with mixed edge styles
- Added 7 tests for BorderChars in style.rs
- Created new `border_render` module with:
  - `render_border()` - Render per-edge styled borders into Buffer
  - `inner_area()` - Calculate content area inside border
  - `border_thickness()` - Get thickness per edge
  - `render_uniform_border()` - Convenience for uniform borders
  - 17 tests covering all border kinds, colors, offsets, and edge cases
- Updated Button widget:
  - Added `render_button_with_style()` method using full ComputedStyle
  - `render_with_context()` now uses per-edge border rendering
  - Fills background, renders border with `render_border()`, then renders content

**Review Fixes Applied (Session 2):**
- Added `BorderKind::Block` variant with full block character (`█`) mappings
  - Parser now recognizes "block" as a valid border style
  - Added tests for block border parsing and character mapping
- Fixed per-edge corner color resolution:
  - Added `corner_style()` helper that prefers horizontal edge color, falls back to vertical
  - Top-left uses top color → left color; bottom-left uses bottom color → left color
  - Added 2 tests for asymmetric corner color scenarios
- Fixed Button CSS-driven rendering:
  - Changed default from `Border::all(Solid)` to `Border::none()`
  - Now strictly CSS-driven: no border unless specified in computed style

**tint/background-tint/text-opacity Implementation:**
- Added `Declaration` variants: `TextOpacity(f64)`, `Tint(Color)`, `BackgroundTint(Color)`
- Added `ComputedStyle` fields: `text_opacity`, `tint`, `background_tint`
- Added parsing for `text-opacity`, `tint`, `background-tint` CSS properties
- Added computation in `ComputedStyle::apply()`
- Added 7 tests for parsing and computation of all three properties

**Style-to-Layout Integration:**
- Added `ResolutionContext` struct to hold parent and viewport sizes for unit resolution
- Added `resolve_scalar()` function for resolving CSS scalars to absolute cell values
- Added `scalar_to_constraint()` function for converting CSS scalars to layout constraints
- Added `LayoutHints::from_computed_style()` method bridging CSS dimensions to layout system
- Supports all CSS units: cells, %, w, h, vw, vh, auto, fr
- Handles min/max width/height with proper fallback values
- Added 22 tests covering all resolution scenarios

**Rendering Integration (Session 3):**
- Wired tint/background-tint/text-opacity into `ComputedStyle::to_ratatui_style()`:
  - `background_tint`: Blends with background color using tint's alpha
  - `tint`: Blends with both foreground and background colors using tint's alpha
  - `text_opacity`: Blends foreground toward background based on opacity (0.0-1.0) [fixed in Session 4]
  - Application order: background_tint → tint → text_opacity
- Added 5 tests for rendering integration:
  - `test_to_ratatui_style_text_opacity_reduces_foreground`
  - `test_to_ratatui_style_text_opacity_full_no_change`
  - `test_to_ratatui_style_tint_blends_colors`
  - `test_to_ratatui_style_background_tint_only_affects_bg`
  - `test_to_ratatui_style_tint_without_base_colors`

**Layout Pipeline Integration (Session 3):**
- Updated `LayoutContext` to optionally hold `StyleManager` reference
- Added `LayoutContext::with_styles()` constructor for CSS-aware contexts
- Updated `layout_hints()` to merge CSS-driven hints when StyleManager available:
  - Gets widget meta via `widget_meta()` method
  - Builds ancestor chain via `get_ancestors()` helper
  - Queries StyleManager for computed style
  - Merges CSS width/height/min/max constraints over widget defaults
- Added `compute_layout_with_styles()` function for CSS-driven layout
- Added `LayoutManager::run_layout_with_styles()` method for API completeness
- Updated `app.rs` to use `compute_layout_with_styles()` when StyleManager available:
  - Lines 355-358: Checks for StyleManager, calls appropriate layout function
  - CSS properties now affect actual widget positions in the layout

**Pseudo-Class Selector Support (Session 3):**
- Added `PseudoClass` enum: `Focus`, `Hover`, `Active`, `Disabled`, `Inline`
- Added `PseudoClass::from_name()` for parsing and `specificity()` (same as class: 0,1,0)
- Added `parse_pseudo_class()` function parsing `:focus`, `:hover`, etc.
- Updated `CompoundSelector` to include `pseudo_classes: Vec<PseudoClass>` field
- Added `CompoundSelector::with_pseudo()` constructor
- Updated `parse_compound_selector()` to parse pseudo-classes after base selectors
- Updated `WidgetMeta` to include `pseudo_classes` field
- Added `WidgetMeta::with_pseudo()`, `with_pseudos()`, `has_pseudo()` methods
- Updated `matches_compound()` to require all selector pseudo-classes match widget
- Added 12 tests covering:
  - Pseudo-class parsing (`:focus`, `:hover`, `:disabled`, `:active`, `:inline`)
  - Compound selector with pseudo-classes (`Button:focus`, `Button.primary:focus:hover`)
  - Specificity calculation with pseudo-classes
  - Style resolution with pseudo-class selectors (`Button:focus { color: blue; }`)
- Exported `PseudoClass` from lib.rs

**Theme Variables Parity (Session 3):**
- Added text contrast variables for semantic colors:
  - `text-primary` - Contrast text for primary background
  - `text-secondary` - Contrast text for secondary background
  - `text-warning` - Contrast text for warning background
  - `text-error` - Contrast text for error background
  - `text-success` - Contrast text for success background
  - `text-accent` - Contrast text for accent background
- Uses `RgbaColor::contrast_text()` for automatic contrast calculation
- Added `test_color_system_text_contrast_variables` test

**Review Fixes Applied (Session 4):**
- Fixed CSS display visibility in layout:
  - Updated `get_effective_display()` to accept optional style_context
  - Now checks CSS computed style display when StyleManager available
  - Updated `is_child_visible()` to pass style_context through
  - CSS `display: none` now correctly hides widgets during layout
- Fixed pseudo-class wiring to WidgetMeta:
  - Added `PseudoClass` import to widget.rs
  - Updated `Widget::widget_meta()` to convert `HashSet<String>` from `pseudo_classes()` to `Vec<PseudoClass>`
  - Pseudo-class selectors (`:focus`, `:hover`, etc.) now match at runtime
- Fixed ancestor order for CSS matching:
  - Added `ancestors.reverse()` to `LayoutContext::get_ancestors()`
  - Made `get_ancestors()` method public
  - Now returns root-to-parent order as CSS matching expects
- Fixed text-opacity blending:
  - Changed to blend foreground toward background color (not black)
  - Falls back to black only if no background color available
  - Matches Python Textual behavior for transparent text
- Fixed text-contrast variable derivation:
  - Added `add_text_contrast_var()` helper method to Theme
  - Tints contrast text (white/black) with semantic color at 66% alpha
  - Matches Python's approach for `text-primary`, `text-secondary`, etc.
  - Updated test to verify blue tint applied to contrast text

## Phase 4: Widget Parity Updates
- [x] Task: Write snapshot tests for Button default and variants (default/primary/warning)
- [x] Task: Update Button rendering to use new border/tint/align features
- [x] Task: Write snapshot tests for Label/Input/Digits using DEFAULT_CSS
- [x] Task: Update Label/Input/Digits to use content-align, padding, text-opacity
- [x] Task: Update DEFAULT_CSS selectors to match Python (including pseudo-classes)
- [ ] Task: Conductor - User Manual Verification "Widget Parity Complete" (Protocol in workflow.md)

**Completed in Phase 4:**
- Created `tests/widget_snapshots.rs` with 29 snapshot tests for Button, Label, Input, Digits
- Tests cover: rendering, variants, pseudo-classes (:focus, :hover, :active, :disabled), custom CSS, borders
- Updated Button DEFAULT_CSS to use pseudo-class selectors instead of class selectors (.-focused → :focus)
- Button DEFAULT_CSS now matches Python Textual patterns:
  - Base: `width: auto; min-width: 16; height: auto; line-pad: 1; text-align: center; content-align: center middle;`
  - 3D style (.-style-default): tall top/bottom borders for depth effect
  - Flat style (.-style-flat): block borders
  - Pseudo-class rules for :focus, :hover, :active, :disabled
  - Variant rules for -primary, -success, -warning, -error
- Updated Button `render_button_with_style()` to apply CSS features:
  - Uses `content_align_horizontal` for horizontal text alignment
  - Uses `content_align_vertical` for vertical text positioning
  - Uses `line_pad` for horizontal padding on label

**Review Fixes (Phase 4):**
- Added Button hover state tracking:
  - `hovered` field in Button struct
  - `is_hovered()`/`set_hovered()` methods for mouse handling
  - `pseudo_classes()` now includes "hover" when hovered
  - `:hover` rules in CSS will now activate at runtime
- Fixed Button tint application:
  - Background fill now uses `ratatui_style.bg` (tint already applied)
  - Previously used raw `computed_style.background` (no tint)
  - `:active` tint effect now properly dims the button background

**Label/Input/Digits CSS Updates (Phase 4):**
- Label:
  - Added `render_label_with_computed()` method for CSS-driven rendering
  - `render_with_context()` now applies content-align (horizontal and vertical)
  - Applies line-pad for horizontal padding around text
  - Uses `to_ratatui_style()` for text-opacity/tint application
  - Fills background area when specified in CSS
- Input:
  - Updated DEFAULT_CSS to use pseudo-class selectors (`:focus`, `:disabled`)
  - Added `content-align: left middle` and `line-pad: 1` to DEFAULT_CSS
  - Uses `text-opacity: 0.6` for disabled state (instead of `opacity`)
  - Added `render_input_with_computed()` for CSS-driven rendering
  - Applies line-pad to all text states (placeholder, value, cursor)
- Digits:
  - Added `render_with_context()` method for CSS-aware rendering
  - Added `render_digits_with_computed()` with content-align support
  - Added `default_css()` with `content-align: center middle`
  - Added `classes()` method to Widget trait implementation
  - Applies horizontal and vertical alignment to position content

**Widget Rendering Fixes (Phase 4 continued):**
- Input:
  - Replaced `Block::default().borders(Borders::ALL)` with CSS border rendering pipeline
  - Now uses `render_border()` + `inner_area()` like Button
  - Vertical alignment applied (v_offset was computed but unused)
  - Horizontal alignment applied via `Paragraph::alignment()`
- Label:
  - Fixed line-pad to apply per line, not per span (avoids duplicates with markup)
  - Fixed vertical alignment for multiline content (uses actual line count)
  - Properly splits spans on newlines and applies padding once per line
- Digits:
  - Added background fill for entire widget area when CSS specifies background

## Phase 5: Example Parity
- [ ] Task: Convert Rust calculator example to widget tree + TCSS (reuse calculator.tcss)
- [ ] Task: Add integration test that renders calculator to buffer and snapshots output
- [ ] Task: Update other examples to use widgets + TCSS where needed
- [ ] Task: Add manual screenshot capture checklist for each example
- [ ] Task: Conductor - User Manual Verification "Example Parity Complete" (Protocol in workflow.md)

## Phase 6: Verification and Regression Coverage
- [ ] Task: Run full test suite and address failures
- [ ] Task: Add visual regression snapshot set for core widgets and examples
- [ ] Task: Document parity verification process in track notes
- [ ] Task: Conductor - User Manual Verification "Track Complete" (Protocol in workflow.md)
