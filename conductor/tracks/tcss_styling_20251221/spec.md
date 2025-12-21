# Specification: TCSS Styling

## Overview
This track extends the existing TCSS foundation to support the full range of CSS properties, advanced selectors, animations, transitions, CSS variables, and theming. TCSS (Textual CSS) is a subset of CSS tailored for terminal UI styling.

## Goals
- Implement all TCSS property types
- Add advanced selector combinators (descendant, child, sibling)
- Implement CSS animations with keyframes
- Implement CSS transitions for smooth property changes
- Add CSS variables (custom properties)
- Build theme system with dark/light mode support

## Reference: Python Textual CSS System

### CSS Properties
```css
/* Layout */
width: 100%;
height: auto;
min-width: 10;
max-height: 50%;

/* Box Model */
margin: 1 2;
padding: 1;
border: solid green;

/* Colors */
color: red;
background: $surface;
border-color: $primary;

/* Text */
text-align: center;
text-style: bold italic;

/* Display */
display: block | none;
visibility: visible | hidden;
opacity: 0.5;

/* Scrolling */
overflow: auto | hidden | scroll;
scrollbar-color: $accent;
```

### Selectors
```css
/* Type selector */
Button { }

/* Class selector */
.danger { }

/* ID selector */
#submit-button { }

/* Pseudo-classes */
Button:hover { }
Input:focus { }
ListItem:even { }

/* Combinators */
Container Button { }      /* descendant */
Container > Button { }    /* child */
Button + Label { }        /* adjacent sibling */
Button ~ Label { }        /* general sibling */

/* Attribute selectors (limited) */
Button.-active { }        /* class check */
```

### Animations
```css
@keyframes fade-in {
    0% { opacity: 0; }
    100% { opacity: 1; }
}

Widget {
    animation: fade-in 0.5s ease-in-out;
}
```

### Transitions
```css
Button {
    background: blue;
    transition: background 0.3s ease;
}

Button:hover {
    background: lightblue;
}
```

### Variables & Themes
```css
:root {
    --primary: #007acc;
    --surface: #1e1e1e;
}

Button {
    background: var(--primary);
}
```

## Deliverables

### Phase 1: Analysis & Research
- Study Python Textual css/ directory thoroughly
- Document all supported CSS properties
- Analyze selector matching algorithm
- Study animation and transition systems

### Phase 2: Design & Planning
- Design property type system in Rust
- Plan selector combinator implementation
- Design animation timeline system
- Plan theme variable resolution

### Phase 3: Implementation

#### 3.1 Extended Properties
- Layout properties (width, height, min-*, max-*)
- Box model (margin, padding, border)
- Color properties (color, background, border-color)
- Text properties (text-align, text-style)
- Display properties (display, visibility, opacity)
- Scroll properties (overflow, scrollbar-*)

#### 3.2 Advanced Selectors
- Descendant combinator (space)
- Child combinator (>)
- Adjacent sibling (+)
- General sibling (~)
- Compound selectors (Button.danger)
- Selector specificity calculation

#### 3.3 Animations
- @keyframes rule parsing
- Animation property parsing
- Timeline management
- Easing functions (linear, ease-in, ease-out, etc.)
- Animation events (start, end, iteration)

#### 3.4 Transitions
- Transition property parsing
- Property interpolation
- Transition timing
- Transition events

#### 3.5 Variables & Themes
- CSS variable declaration (--name: value)
- var() function resolution
- Theme struct with variable maps
- Dark/light mode switching
- Built-in themes

#### 3.6 Hot Reloading
- File watcher for .tcss files
- Stylesheet reloading without restart
- Change notification to widgets

### Phase 4: Testing & Verification
- Property parsing tests
- Selector matching tests
- Animation timeline tests
- Theme switching tests

## Success Criteria
- [ ] All TCSS properties parsed and applied
- [ ] Complex selectors match correctly
- [ ] Animations play with keyframes
- [ ] Transitions smooth property changes
- [ ] Variables resolve in styles
- [ ] Themes switch dark/light mode

## Dependencies
- Core App Lifecycle track (for widget tree)
- Existing textual-rs style foundation

## Out of Scope
- Rich text markup (Content system)
- Custom property types beyond TCSS spec
