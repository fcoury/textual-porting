# TextArea Syntax Highlighting (Python Textual)

Sources:
- `textual/src/textual/widgets/_text_area.py`
- `textual/src/textual/_tree_sitter.py`
- `textual/src/textual/_text_area_theme.py`
- `textual/src/textual/tree-sitter/highlights/*.scm`

## Tree-sitter Integration
- Tree-sitter is optional and gated by `TREE_SITTER` (import guard in `_tree_sitter.py`).
- `get_language(name)` loads `tree_sitter_<name>` modules dynamically, with caching.
  - Special case for `xml` using `language_xml()`.
- If tree-sitter is unavailable or language missing, TextArea falls back to plain text and logs a warning.

## Language Registration
- `TextArea.register_language(name, language, highlight_query)`:
  - Stores custom `TextAreaLanguage` in `_languages` (per-instance).
  - Overrides builtin highlight query if the language already exists.
- `update_highlight_query(name, query)` updates only the query and reloads if active.
- Builtin highlight queries live in `textual/tree-sitter/highlights/<language>.scm`.

## Document Construction & Highlight Query
- `_set_document(text, language)` determines document type:
  - If tree-sitter + language:
    - Resolve `Language` (user-registered first, then builtin).
    - Build `SyntaxAwareDocument`; fallback to `Document` on parser error.
    - Prepare query (`document.prepare_query`) and store in `_highlight_query`.
  - If tree-sitter missing or language None: `Document`.
- On successful syntax-aware document, `_build_highlight_map()` is called.

## Highlight Map Construction
- `_build_highlight_map()`:
  - Clears `_line_cache` and `_highlights`.
  - Calls `document.query_syntax_tree(query)` -> map of highlight name -> nodes.
  - Converts node ranges into per-line highlight tuples:
    - Single-line: `(start_col, end_col, name)`
    - Multi-line: first line `(start_col, None, name)`, middle lines `(0, None, name)`, last `(0, end_col, name)`

## Theme Mapping
- `TextAreaTheme.syntax_styles: dict[str, Style]` maps highlight names to Rich styles.
- Builtin themes (e.g., `monokai`) define many mappings (`keyword`, `string`, `comment`, etc.).
- `TextAreaTheme.apply_css(text_area)` fills any missing theme styles from CSS component classes.

## Render-time Application
- Rendering pulls `line_highlights = _highlights[line_index]` and applies styles if theme present.
- Highlight byte offsets are converted to codepoint offsets using `document.get_utf8_offsets`.
- Selection, cursor, cursor-line, and matching-bracket styling overlay on top of syntax styles.

## Behavior on Changes
- Highlights rebuilt on:
  - language changes
  - theme changes that affect highlight query
  - content edits (after `edit()`)
  - wrap/indent changes (affects mapping)

## Implications for Rust Port
- Need optional tree-sitter integration with fallback to plain text.
- Highlight query sources should mirror `.scm` files per language.
- Theme mapping must support per-token style names.
- Per-line highlight spans should be stored, and conversion from byte offsets to display offsets is required.
- Highlight recomputation must be triggered on edits and language changes.
