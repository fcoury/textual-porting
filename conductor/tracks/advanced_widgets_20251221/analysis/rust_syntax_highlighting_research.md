# Rust Syntax Highlighting Libraries (Research)

## Candidates

### syntect
- Grammar source: Sublime Text syntax definitions.
- Emphasis: mature, broad language coverage, fast highlighting, incremental parsing support.
- Notes: project is “mostly complete” and maintained but not under heavy development.
- Provides built‑in syntax dumps and themes; can output ANSI/HTML styles.

### tree-sitter + tree-sitter-highlight
- Grammar source: tree-sitter language grammars.
- Emphasis: incremental parsing with concrete syntax trees; highlight queries (captures) for styling.
- The tree-sitter project provides the `tree-sitter` Rust bindings and the `tree-sitter-highlight` crate.

### syntastica (tree-sitter based)
- Higher-level wrapper around tree-sitter highlighting.
- Bundles parsers + themes crates and exposes a more ergonomic API.
- Uses an internal fork of tree-sitter highlighting logic.

### tui-syntax-highlight (syntect adapter for ratatui)
- Provides a ratatui-oriented API on top of syntect.
- Useful if we want a smaller adapter layer rather than direct syntect integration.

## Initial Takeaways
- syntect is stable and widely used; good for immediate parity and breadth of language coverage.
- tree-sitter provides structural parsing and can support richer editor features (selection, matching, AST-driven highlighting).
- syntastica offers a tree-sitter-based high-level API but adds extra dependency surface.
- For TextArea parity, we likely want a tree-sitter path (to match Python Textual behavior), but syntect remains a strong fallback.

## Open Questions for Phase 2 Design
- Do we need tree-sitter’s incremental parse model for editor UX parity, or is syntect sufficient for initial porting?
- Do we want to bundle highlight queries in-repo (like Python Textual does) or rely on external crates?
- For the TUI pipeline, do we build a small adapter (syntect or tree-sitter) that emits `Text` / `Content` compatible spans?
