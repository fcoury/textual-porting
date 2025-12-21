# Specification: Standard Widgets

## Goal
Provide a "batteries-included" set of basic widgets so developers don't have to build everything from scratch.

## Requirements
- **Label:** Simple text display with styling support.
- **Button:** Clickable text block with hover effects and `OnPress` message dispatch.
- **Input:** Single-line text entry with cursor management.
- **Header:** Standard application header showing title and clock.
- **Footer:** Standard application footer showing key bindings.

## Success Criteria
- Tests verify each widget renders its state correctly into the buffer.
- `Input` widget correctly updates its value on KeyPress.
- `Button` correctly fires a message on Click.
