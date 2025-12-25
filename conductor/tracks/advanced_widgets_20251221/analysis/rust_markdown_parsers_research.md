# Rust Markdown Parsers (Research)

## Candidates

### pulldown-cmark
- Commonly used, fast, low allocation parser for CommonMark.
- Exposes an iterator of events; easy to map to widget blocks.
- Supports GFM extensions via options (tables, task lists, strikethrough, etc.).

### comrak
- Full CommonMark/GFM parser with an AST.
- More flexible but heavier; produces an explicit node tree.
- Good for features like TOC generation, heading IDs, and more complex transformations.

### markdown-it (Rust ports)
- There are Rust ports of markdown-it, but they’re less mature and not as widely used.
- Python Textual uses markdown-it for token stream; pulldown-cmark provides a similar event stream.

## Initial Takeaways
- pulldown-cmark is a good fit for streaming / event-driven conversion into widgets.
- comrak is a good fit for TOC generation and more complex transformations (but higher overhead).
- For parity with Python Textual’s token approach, pulldown-cmark with GFM options is closest in spirit.

## Open Questions for Phase 2 Design
- Do we need a full AST (comrak) to support TOC and incremental updates, or is event-stream (pulldown) enough?
- Do we plan to support markdown-it specific features (if any) from Python examples, or just GFM?
