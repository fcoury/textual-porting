# Specification: Layout Engine

## Goal
Enable the framework to automatically calculate the size and position of widgets based on constraints and parent dimensions, replacing hardcoded layouts.

## Requirements
- **Geometry:** `Rect`, `Size`, `Point`, and `Region` structs (likely wrapping or extending Ratatui's primitives).
- **Constraints:** Support for Fixed, Percentage, and Auto sizing.
- **Docking Layout:** A layout strategy where widgets dock to edges (Top, Bottom, Left, Right) filling remaining space.
- **Integration:** The `App` loop must run a "Layout" pass before the "Render" pass.

## Success Criteria
- Tests verify a "Dock Top" widget correctly reduces the available space for subsequent widgets.
- The `App` loop correctly reflows the layout when the terminal is resized.
