# Technical Design: TCSS Styling System

## Executive Summary

This document describes the complete TCSS (Textual CSS) styling system for the Rust implementation of Textual. The system provides a CSS-like styling language with 70+ properties, selector matching, animations, transitions, and theming support.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         Application                              │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    ThemeRegistry                             ││
│  │  - Built-in themes (textual-dark, nord, dracula, etc.)      ││
│  │  - Custom theme registration                                 ││
│  │  - CSS variable generation                                   ││
│  └──────────────────────────┬──────────────────────────────────┘│
│                             │                                    │
│  ┌──────────────────────────▼──────────────────────────────────┐│
│  │                     Stylesheet                               ││
│  │  - CSS parsing with variable resolution                     ││
│  │  - RuleSet storage with specificity                         ││
│  │  - Rules map for O(1) selector lookup                       ││
│  │  - Cascade resolution                                        ││
│  └──────────────────────────┬──────────────────────────────────┘│
│                             │                                    │
│  ┌──────────────────────────▼──────────────────────────────────┐│
│  │                      Animator                                ││
│  │  - Active animation tracking                                 ││
│  │  - 60fps timer loop                                          ││
│  │  - Easing function library                                   ││
│  │  - Transition integration                                    ││
│  └──────────────────────────┬──────────────────────────────────┘│
│                             │                                    │
│  ┌──────────────────────────▼──────────────────────────────────┐│
│  │                   Widget Styles                              ││
│  │  - RenderStyles (computed values)                           ││
│  │  - Inline styles                                             ││
│  │  - Property refresh semantics                                ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

## Component Designs

### 1. Property Types

**Detailed Design:** [property_types.md](./property_types.md)

The property system provides:
- **StyleValue enum**: All possible CSS values (Scalar, Color, Spacing, etc.)
- **PropertyName enum**: 70+ CSS property identifiers
- **Animatable trait**: Interpolation for animations
- **Refresh semantics**: Layout vs repaint triggering

Key types:
```rust
pub enum StyleValue { Scalar(Scalar), Color(Color), Spacing(Spacing), ... }
pub enum PropertyName { Width, Height, Color, Background, ... }
pub trait Animatable { fn blend(&self, dest: &Self, factor: f64) -> Self; }
```

### 2. Selector AST

**Detailed Design:** [selector_ast.md](./selector_ast.md)

The selector system provides:
- **SelectorSet**: Comma-separated selector groups (OR logic)
- **Selector**: Compound selectors with combinators
- **CompoundSelector**: Multiple simple selectors (AND logic)
- **SimpleSelector**: Type, class, ID, pseudo-class matchers
- **Specificity**: 6-tuple for cascade ordering

**Important:** Only descendant (space) and child (`>`) combinators are supported. Sibling combinators (`~`, `+`) are NOT supported.

Key structures:
```rust
pub struct SelectorSet { pub selectors: Vec<Selector> }
pub struct Selector { pub parts: Vec<SelectorPart>, pub specificity: Specificity }
pub enum SimpleSelector { Type(String), Class(String), Id(String), PseudoClass(PseudoClass) }
```

### 3. Animation Timeline

**Detailed Design:** [animation_timeline.md](./animation_timeline.md)

The animation system provides:
- **Animator**: Central controller with 60fps timer
- **Animation trait**: Interface for all animation types
- **SimpleAnimation**: Standard property interpolation
- **EasingFunction**: 30+ easing functions
- **Transition**: CSS transition definitions

Key constraint: Duration and speed are mutually exclusive.

Key structures:
```rust
pub struct Animator { animations: HashMap<AnimationKey, Box<dyn Animation>>, fps: u32 }
pub trait Animation { fn tick(&mut self, time: Instant, level: AnimationLevel) -> bool; }
pub enum EasingFunction { Linear, InQuad, OutQuad, InOutQuad, ... }
```

### 4. Theme Variables

**Detailed Design:** [theme_variables.md](./theme_variables.md)

The theming system provides:
- **Theme**: Complete theme definition with colors and settings
- **ColorSystem**: CSS variable generation with shades
- **VariableContext**: Variable resolution during parsing
- **ThemeRegistry**: Theme management and switching

Key structures:
```rust
pub struct Theme { name: String, primary: String, dark: bool, ... }
pub struct ColorSystem { fn generate_variables(&self) -> HashMap<String, String> }
pub struct ThemeRegistry { themes: HashMap<String, Theme>, current: String }
```

## Data Flow

### Style Application Flow

```
1. CSS Source                    2. Parse & Store
   ┌──────────────┐                ┌─────────────────┐
   │ Button {     │  ──────────>   │ RuleSet {       │
   │   color: red;│                │   selectors,    │
   │ }            │                │   styles,       │
   └──────────────┘                │   specificity   │
                                   └────────┬────────┘
                                            │
3. Selector Match                  4. Cascade Resolution
   ┌─────────────────┐               ┌─────────────────┐
   │ For each node:  │  <────────    │ Collect rules   │
   │ - Get candidates│               │ Sort by         │
   │ - Match selector│               │   specificity   │
   │ - Collect rules │               │ Select highest  │
   └────────┬────────┘               └────────┬────────┘
            │                                 │
5. Apply Styles                    6. Trigger Refresh
   ┌─────────────────┐               ┌─────────────────┐
   │ For each prop:  │  ──────────>  │ if layout: true │
   │ - Check animate │               │   recalc layout │
   │ - Set value     │               │ repaint widget  │
   └─────────────────┘               └─────────────────┘
```

### Animation Flow

```
1. Property Change               2. Check Transition
   ┌─────────────────┐              ┌─────────────────┐
   │ color: red →    │  ─────────>  │ transition:     │
   │        blue     │              │   color 0.3s    │
   └─────────────────┘              └────────┬────────┘
                                             │
3. Create Animation              4. Timer Tick (60fps)
   ┌─────────────────┐              ┌─────────────────┐
   │ SimpleAnimation │  <────────   │ For each anim:  │
   │   start: red    │              │   calc factor   │
   │   end: blue     │              │   apply easing  │
   │   easing: ease  │              │   blend values  │
   └────────┬────────┘              │   set property  │
            │                       └────────┬────────┘
            ▼                                │
5. Completion                               ▼
   ┌─────────────────┐              6. Remove Animation
   │ factor >= 1.0   │                 ┌─────────────────┐
   │ set final value │                 │ Run callback    │
   │ call on_complete│                 │ Remove from map │
   └─────────────────┘                 └─────────────────┘
```

### Theme Switch Flow

```
1. Set Theme                     2. Generate Variables
   ┌─────────────────┐              ┌─────────────────┐
   │ app.set_theme(  │  ─────────>  │ ColorSystem::   │
   │   "nord"        │              │   from_theme()  │
   │ )               │              │   .generate()   │
   └─────────────────┘              └────────┬────────┘
                                             │
3. Update Stylesheet             4. Reparse CSS
   ┌─────────────────┐              ┌─────────────────┐
   │ stylesheet      │  <────────   │ Resolve all     │
   │   .set_variables│              │   $variables    │
   │   (vars)        │              │ Rebuild rules   │
   └────────┬────────┘              └────────┬────────┘
            │                                │
5. Refresh Styles                          ▼
   ┌─────────────────┐              6. Repaint All
   │ apply(node)     │                 ┌─────────────────┐
   │ for all widgets │                 │ Trigger full    │
   └─────────────────┘                 │   repaint       │
                                       └─────────────────┘
```

## Specificity Resolution

CSS cascade priority (6-tuple, higher wins - use `max()` to select winning rule):

| Position | Field | Description |
|----------|-------|-------------|
| 0 | `is_user_css` | 0 for DEFAULT_CSS (widget defaults), 1 for user CSS (user wins) |
| 1 | `important` | 1 if !important, 0 otherwise |
| 2 | `ids` | Count of ID selectors (#id) |
| 3 | `classes` | Count of class/pseudo-class selectors (.class, :hover) |
| 4 | `types` | Count of type selectors (Button, Label) |
| 5 | `tie_breaker` | Source order (later wins) |

Example:
```
User CSS:    #main .btn:hover  → (1, 0, 1, 2, 0, n)  // user CSS, 1 ID, 2 classes
User CSS:    Button.primary    → (1, 0, 0, 1, 1, n)  // user CSS, 1 class, 1 type
DEFAULT_CSS: Button            → (0, 0, 0, 0, 1, n)  // widget default, 1 type
```

## Implementation Plan Corrections

Based on Phase 1 analysis, the following plan items need correction:

### Phase 3.2: Advanced Selectors
**Remove:**
- ~~Implement adjacent sibling combinator (+)~~
- ~~Implement general sibling combinator (~)~~

**Reason:** Textual CSS does not support sibling selectors. Only descendant (space), child (`>`), and compound (same element) combinators are supported.

**Keep:**
- Implement descendant combinator (space)
- Implement child combinator (>)
- Implement compound selectors (Type.class#id)
- Implement selector specificity calculation

## File Structure

```
textual-rs/src/
├── css/
│   ├── mod.rs              # Module exports
│   ├── styles.rs           # Styles struct, RenderStyles
│   ├── stylesheet.rs       # Stylesheet, cascade, rules map
│   ├── parser.rs           # CSS parser
│   ├── selector.rs         # Selector AST, parsing, matching
│   ├── specificity.rs      # Specificity calculation
│   ├── properties.rs       # Property names, value types
│   ├── values.rs           # StyleValue, Scalar, Spacing, etc.
│   ├── color.rs            # Color parsing and manipulation
│   ├── animation.rs        # Animator, Animation trait
│   ├── easing.rs           # Easing functions
│   ├── transition.rs       # Transition parsing and storage
│   ├── theme.rs            # Theme, ColorSystem
│   ├── variables.rs        # Variable resolution
│   └── query.rs            # DOMQuery implementation
└── ...
```

## Testing Strategy

### Unit Tests
- Property parsing and validation
- Selector parsing and matching
- Specificity calculation
- Easing function correctness
- Color manipulation (lighten, darken, blend)
- Variable resolution

### Integration Tests
- Full cascade resolution
- Animation lifecycle
- Theme switching
- Stylesheet reloading

### Property-Based Tests
- Selector parsing roundtrip
- Color blending commutes with factor
- Easing function bounds (output in 0.0-1.0)

## Performance Considerations

1. **Rules Map**: O(1) candidate rule lookup by selector name
2. **Style Cache**: Cache computed styles by (parent, classes, pseudo-classes)
3. **Animation Batching**: Single timer for all animations
4. **Lazy Evaluation**: DOMQuery nodes evaluated only when accessed
5. **Variable Pre-resolution**: Resolve variables once at parse time

## Open Questions

1. **Hot Reloading**: Should file watching be in the CSS module or application layer?
2. **Custom Properties**: Support for custom CSS properties beyond theme variables?
3. **Media Queries**: Support for responsive breakpoints in terminal context?
4. **Inheritance**: Which properties should inherit from parent widgets?

## References

- Phase 1 Analysis Documents: `../analysis/`
- Python Textual Source: `src/textual/css/`
- CSS Specification: https://www.w3.org/Style/CSS/
