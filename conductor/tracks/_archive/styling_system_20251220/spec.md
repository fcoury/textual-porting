# Specification: Styling System

## Goal
Implement a subset of CSS (TCSS) to separate visual design from application logic.

## Requirements
- **Parser:** A parser for `.tcss` files supporting selectors (ID, class, type) and basic rules (color, background, borders).
- **Style Struct:** A Rust representation of computed styles.
- **Cascading:** Logic to merge styles based on Specificity (ID > Class > Type) and Inheritance (parent -> child).
- **Application:** The Renderer must look up the computed style for each widget before drawing.

## Success Criteria
- Tests verify that a `.tcss` file is correctly parsed into a Style rule tree.
- Tests verify that "cascading" correctly overrides generic styles with specific ones.
