# Plan: TCSS Styling

## Phase 1: Analysis & Research
- [x] Task: Study Python Textual css/styles.py for property definitions [163b1df]
- [x] Task: Analyze css/_style_properties.py for all property types [fae11e4]
- [x] Task: Document css/stylesheet.py for cascade and specificity
- [x] Task: Study css/query.py for selector matching algorithm
- [x] Task: Analyze _animator.py for animation system
- [x] Task: Document theme.py for theming approach
- [~] Task: Conductor - User Manual Verification 'Analysis Complete' (Protocol in workflow.md)

## Phase 2: Design & Planning
- [ ] Task: Design property type enum with all TCSS properties
- [ ] Task: Design selector AST for combinators
- [ ] Task: Design animation timeline architecture
- [ ] Task: Design theme variable resolution
- [ ] Task: Write technical design document
- [ ] Task: Conductor - User Manual Verification 'Design Approved' (Protocol in workflow.md)

## Phase 3: Implementation

### 3.1 Extended Property Types
- [ ] Task: Implement layout properties (width, height, min-*, max-*)
- [ ] Task: Implement box model properties (margin, padding, border)
- [ ] Task: Implement color properties (color, background, border-color)
- [ ] Task: Implement text properties (text-align, text-style)
- [ ] Task: Implement display properties (display, visibility, opacity)
- [ ] Task: Implement scroll properties (overflow, scrollbar-*)
- [ ] Task: Write tests for property parsing

### 3.2 Advanced Selectors
- [ ] Task: Implement descendant combinator (space)
- [ ] Task: Implement child combinator (>)
- [ ] Task: Implement adjacent sibling combinator (+)
- [ ] Task: Implement general sibling combinator (~)
- [ ] Task: Implement compound selectors (Type.class#id)
- [ ] Task: Implement selector specificity calculation
- [ ] Task: Write tests for selector matching

### 3.3 Animations
- [ ] Task: Implement @keyframes rule parsing
- [ ] Task: Implement animation property parsing
- [ ] Task: Implement animation timeline
- [ ] Task: Implement easing functions (linear, ease-in, ease-out, ease-in-out)
- [ ] Task: Implement cubic-bezier easing
- [ ] Task: Implement animation events (on_animation_end)
- [ ] Task: Write tests for animations

### 3.4 Transitions
- [ ] Task: Implement transition property parsing
- [ ] Task: Implement property value interpolation
- [ ] Task: Implement transition timing integration
- [ ] Task: Implement transition events
- [ ] Task: Write tests for transitions

### 3.5 Variables & Themes
- [ ] Task: Implement CSS variable declaration parsing
- [ ] Task: Implement var() function resolution
- [ ] Task: Implement Theme struct with variable maps
- [ ] Task: Implement dark/light mode detection
- [ ] Task: Create built-in themes (default, dark, light)
- [ ] Task: Implement theme switching API
- [ ] Task: Write tests for variables and themes

### 3.6 Hot Reloading
- [ ] Task: Implement file watcher for .tcss files
- [ ] Task: Implement stylesheet reloading
- [ ] Task: Implement change notification to widgets
- [ ] Task: Write tests for hot reloading

- [ ] Task: Conductor - User Manual Verification 'Implementation Complete' (Protocol in workflow.md)

## Phase 4: Testing & Verification
- [ ] Task: Create comprehensive property parsing tests
- [ ] Task: Create selector matching test suite
- [ ] Task: Create animation example app
- [ ] Task: Create theme switching example
- [ ] Task: Run all tests and fix failures
- [ ] Task: Conductor - User Manual Verification 'Track Complete' (Protocol in workflow.md)
