# Markdown Rendering Pipeline (Python Textual)

Source: `textual/src/textual/widgets/_markdown.py`

## Core Architecture
- Markdown uses `markdown-it-py` (`MarkdownIt("gfm-like")`) to parse source into tokens.
- Tokens are converted into a hierarchy of `MarkdownBlock` widgets.
- Blocks render their own content using `Content` + `Span` styling (Textual's rich text).
- Layout is vertical; each block is a widget with its own CSS.

## Inline Parsing (`MarkdownBlock._token_to_content`)
- Converts inline tokens to `Content` with spans:
  - `text` -> normalized whitespace
  - `softbreak` -> space
  - `hardbreak` -> newline
  - `em`, `strong`, `s` -> span with component class style
  - `code_inline` -> span with `.code_inline`
  - `link_open` / `image` -> span with `@click` action meta for link handling
- Spans are built via a stack: `add_style` pushes, `close_tag` emits spans.

## Block Mapping
`BLOCKS` maps token types to widgets:
- Headers: `h1`..`h6`
- Paragraph, blockquote, list (ordered/unordered), list items
- Table blocks (`MarkdownTable`, `MarkdownTR`, `MarkdownTH`, `MarkdownTD`)
- Code fences (`MarkdownFence`)

## Update Flow
- `update(markdown)`:
  - Parses tokens asynchronously (thread executor).
  - Builds blocks in batches (BATCH_SIZE=200) and mounts them.
  - Clears existing MarkdownBlock children on first batch.
  - Recomputes table of contents and posts `TableOfContentsUpdated`.
- `append(markdown_fragment)`:
  - Only parses delta after `_last_parsed_line`.
  - Attempts to update last block in place, then mounts new blocks.
  - Adjusts TOC if new headers appear.

## Table of Contents
- TOC is computed from mounted `MarkdownHeader` children: `(level, text, id)`.
- Each header receives a unique `id` derived from slug + `id(self)`.
- `MarkdownTableOfContents` renders TOC as a `Tree` widget.
- Selecting a TOC node posts `TableOfContentsSelected` with target block id.

## Links & Navigation
- Link clicks post `Markdown.LinkClicked` message.
- If `open_links=True`, default handler opens URL.
- `MarkdownViewer` manages navigation history via `Navigator`:
  - `go(path)` loads new document or anchor
  - `back()` / `forward()` re-load previous documents

## Major Blocks & Rendering Notes
- Headers: styled via CSS variables (`$markdown-h1-*`, etc.).
- Blockquotes: styled with border + background.
- Lists: use `Horizontal` layout with `MarkdownBullet` widget.
- Tables: `MarkdownTableContent` renders a grid, uses keyline and header styling.
- Code fences: use `highlight()` to render code, allow horizontal scroll, theme-dependent background.

## Implications for Rust Port
- Requires a Markdown parser (markdown-it or equivalent).
- Must build a widget tree rather than single text render.
- Inline parsing requires span-based styling + link click metadata.
- Table of contents needs stable header IDs and Tree integration.
- Append/update workflow should be incremental for large documents.
