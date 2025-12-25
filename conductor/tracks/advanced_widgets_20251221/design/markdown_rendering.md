# Markdown AST → Terminal Rendering (Design)

## Goals
- Match Python Textual behavior: widget-per-block, inline spans for emphasis/links.
- Use a Markdown parser with GFM features (tables, strikethrough, task lists).
- Support incremental updates (append) and table of contents.

## Parser Choice
- Use `pulldown-cmark` for event stream parsing (close to markdown-it token flow).
- Enable GFM options: tables, strikethrough, task lists.

## Rendering Model

### Block Widgets
Map markdown blocks into widget tree:
- `Heading` → `MarkdownH1..H6` widgets
- `Paragraph` → `MarkdownParagraph`
- `BlockQuote` → `MarkdownBlockQuote`
- `List` / `ListItem` → `MarkdownList`, `MarkdownListItem`
- `CodeBlock` → `MarkdownFence` (with syntax highlight if language specified)
- `Table` → `MarkdownTable` + `MarkdownTableContent`
- `HorizontalRule` → `MarkdownHorizontalRule`

### Inline Rendering
- Inline events are converted to `Content` spans:
  - emphasis, strong, strikethrough
  - inline code
  - links with `@click` action metadata
  - soft break → space, hard break → newline

### TOC
- Extract headings during parse and build TOC data: `(level, label, id)`.
- Use slugified IDs for anchors, `TrackedSlugs` to ensure uniqueness.
- `MarkdownTableOfContents` renders TOC via `Tree` widget.

### Links & Navigation
- `LinkClicked` message posted from inline spans.
- `MarkdownViewer` manages navigation history and `goto_anchor`.

## Incremental Updates
- Track `_last_parsed_line` to parse only new fragments on append.
- For append, attempt to update last block in place; mount new blocks if required.

## Styling
- Use component classes for inline styles: `.em`, `.strong`, `.code_inline`, `.s`.
- Block widgets define DEFAULT_CSS (match Python tokens and variables).

## Open Questions
- Should we preserve exact markdown-it token semantics or adapt to pulldown-cmark events?
- How to map nested lists and ordered list numbering in the widget tree cleanly?
