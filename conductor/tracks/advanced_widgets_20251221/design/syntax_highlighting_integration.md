# Syntax Highlighting Integration (Design)

## Goals
- Match Python Textual behavior: optional tree-sitter highlighting with query-based spans.
- Provide safe fallback to plain-text when parser is missing.
- Keep highlight spans per line to simplify rendering integration.

## Proposed Approach

### Primary Path: tree-sitter + highlight queries
- Use `tree-sitter` Rust crates with language-specific parsers (via feature flags).
- Load highlight queries from `assets/tree-sitter/highlights/<lang>.scm` (mirrors Python).
- Compile queries per language; cache in `HighlightRegistry`.

### Fallback Path: syntect (optional)
- If tree-sitter not enabled, allow syntect (or no highlighting) as fallback.
- This can be behind a feature flag (e.g., `syntax-syntect`).

## Types

```rust
pub struct HighlightSpan {
    pub start: usize, // column in codepoints
    pub end: Option<usize>,
    pub name: String,
}

pub struct HighlightMap {
    pub per_line: HashMap<usize, Vec<HighlightSpan>>,
}

pub trait SyntaxHighlighter {
    fn highlight(&mut self, text: &str) -> HighlightMap;
}
```

## Integration with TextArea
- `TextArea` owns a `DocumentBase` and an optional `SyntaxHighlighter`.
- On `set_document` or content edit:
  1) refresh syntax tree
  2) rebuild highlight map
- Render path:
  - For each visible line, apply spans from highlight map before selection/cursor styling.

## Theme Mapping
- `TextAreaTheme.syntax_styles: HashMap<String, Style>` map highlight name -> Style.
- Apply style if present; otherwise fallback to base style.
- Theme overrides component class styles if explicitly defined.

## Incremental Updates
- Initial implementation can rebuild highlights for the full document.
- Later optimization: only rebuild changed ranges and update line spans.

## Failure Handling
- If a language parser/query is missing:
  - log warning
  - fall back to plain text (no highlights)
- If highlight query fails to parse: disable highlighting for that language.

## Open Questions
- Should we cache parse trees by document version to avoid full rebuilds?
- Do we allow user-registered languages and queries (like Python)?
