# Plan: TCSS Styling

## Phase 1: Analysis & Research
- [x] Task: Study Python Textual css/styles.py for property definitions [163b1df]
- [x] Task: Analyze css/_style_properties.py for all property types [fae11e4]
- [x] Task: Document css/stylesheet.py for cascade and specificity [22bbf6b]
- [x] Task: Study css/query.py for selector matching algorithm [22bbf6b]
- [x] Task: Analyze _animator.py for animation system [22bbf6b]
- [x] Task: Document theme.py for theming approach [22bbf6b]
- [x] Task: Conductor - User Manual Verification 'Analysis Complete' (Protocol in workflow.md) [22bbf6b]

## Phase 2: Design & Planning
- [x] Task: Design property type enum with all TCSS properties [22415e9]
- [x] Task: Design selector AST for combinators [22415e9]
- [x] Task: Design animation timeline architecture [22415e9]
- [x] Task: Design theme variable resolution [22415e9]
- [x] Task: Write technical design document [22415e9]
- [x] Task: Conductor - User Manual Verification 'Design Approved' (Protocol in workflow.md) [7a25ad6]

## Phase 3: Implementation

### 3.1 Extended Property Types
- [x] Task: Implement layout properties (width, height, min-*, max-*) [9a33880]
- [x] Task: Implement box model properties (margin, padding, border) [b7d0bfe]
- [x] Task: Implement color properties (color, background, border-color) [b7d0bfe]
- [x] Task: Implement text properties (text-align, text-style) [2fc5562]
- [x] Task: Implement display properties (display, visibility, opacity) [108ab64]
- [x] Task: Implement scroll properties (overflow, scrollbar-*) [5004aa5]
- [x] Task: Write tests for property parsing [5004aa5]

### 3.2 Advanced Selectors
- [x] Task: Implement descendant combinator (space) [8891c71]
- [x] Task: Implement child combinator (>) [344fc74]
- [x] Task: Implement compound selectors (Type.class#id) [b65d75b]
- [x] Task: Implement selector specificity calculation [99ab2da]
- [x] Task: Write tests for selector matching [166ae9c]

> **Note:** Sibling combinators (`+`, `~`) are NOT supported in TCSS per Python Textual analysis.

### 3.3 Animations
- [x] Task: Implement @keyframes rule parsing [6b704ba]
- [x] Task: Implement animation property parsing [6182fdc]
- [x] Task: Implement animation timeline [502cc65]
- [x] Task: Implement easing functions (linear, ease-in, ease-out, ease-in-out) [dd7931b]
- [x] Task: Implement cubic-bezier easing [28ba595]
- [x] Task: Implement animation events (on_animation_end) [ca6f188]
- [x] Task: Write tests for animations [5f2183c]

### 3.4 Transitions
- [x] Task: Implement transition property parsing [3ce236b]
- [x] Task: Implement property value interpolation [6a7afac]
- [x] Task: Implement transition timing integration [52003a1]
- [x] Task: Implement transition events [733ee8e]
- [x] Task: Write tests for transitions [7fa9daa]

### 3.5 Variables & Themes
- [x] Task: Implement CSS variable declaration parsing [e69d05d]
- [x] Task: Implement var() function resolution [e69d05d]
- [x] Task: Implement Theme struct with variable maps [0c01a4e]
- [x] Task: Implement dark/light mode detection [0c01a4e]
- [x] Task: Create built-in themes (default, dark, light) [0c01a4e]
- [x] Task: Implement theme switching API [0c01a4e]
- [x] Task: Write tests for variables and themes [0c01a4e]

### 3.6 Hot Reloading
- [x] Task: Implement file watcher for .tcss files [569312c]
- [x] Task: Implement stylesheet reloading [569312c]
- [x] Task: Implement change notification to widgets [569312c]
- [x] Task: Write tests for hot reloading [569312c]

- [ ] Task: Conductor - User Manual Verification 'Implementation Complete' (Protocol in workflow.md)

## Phase 4: Testing & Verification
- [x] Task: Create comprehensive property parsing tests [569312c]
- [x] Task: Create selector matching test suite [569312c]
- [x] Task: Create animation example app [569312c]
- [x] Task: Create theme switching example [569312c]
- [x] Task: Run all tests and fix failures [569312c]
- [ ] Task: Conductor - User Manual Verification 'Track Complete' (Protocol in workflow.md)

> **Note:** Phase 4 testing tasks were already completed during Phase 3 implementation.
> Total tests: 969 (508 style-related tests covering property parsing, selectors,
> animations, transitions, theming, and hot reloading).

## Phase 5: Bug Fixes for Python Parity [d374d5e]

### 5.1 High Priority Fixes (Completed)
- [x] Task: Fix CSS color parsing - add rgb/rgba/hsl/hsla, #RGBA, auto format [d374d5e]
- [x] Task: Wire advanced selectors into stylesheet parsing/matching [d374d5e]
- [x] Task: Connect @keyframes to StyleSheet::parse [d374d5e]
- [x] Task: Implement 6-tuple specificity (is_user_css, important, ids, classes, types, tie_breaker) [d374d5e]

### 5.2 Medium Priority Fixes (Deferred)
- [ ] Task: Variable substitution - only substitute $tokens, not $ in strings/comments
- [ ] Task: Duration parsing - require ms/s units like Python

> **Note:** High priority issues from code review have been addressed.
> Medium priority issues deferred for future work.
> Final tests: 986 passed (17 new tests added for color parsing, selectors, keyframes, specificity).
