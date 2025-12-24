# Plan: TCSS Style Integration

## Phase 1: Analysis & Research
- [x] Task: Study existing Widget trait and identify all methods that need extension [9ef03ba]
- [x] Task: Analyze current widget render pipeline in run_managed() and ManagedWidgetApp [4fd1396]
- [x] Task: Document how WidgetRefreshState.refresh_styles_required is currently used (if at all) [bff7d15]
- [x] Task: Review Python Textual's style integration for reference patterns [11a1054]
- [x] Task: Map existing ComputedStyle fields to ratatui::Style capabilities [3fb53f3]
- [x] Task: Conductor - User Manual Verification 'Analysis Complete' (Protocol in workflow.md)

## Phase 2: Design & Planning
- [x] Task: Design StyleManager struct with all owned components [db341a4]
- [x] Task: Design RenderContext struct and access pattern [e2f0744]
- [x] Task: Design style cache data structures and invalidation strategy [dbc4034]
- [x] Task: Design ancestor chain construction during render traversal [c91a191]
- [x] Task: Design animated value overlay mechanism [3a0b8bf]
- [x] Task: Write technical design document [2f5a0cf]
- [ ] Task: Conductor - User Manual Verification 'Design Approved' (Protocol in workflow.md)

## Phase 3: Implementation

### 3.1 Core StyleManager
- [ ] Task: Write tests for StyleManager construction and basic API
- [ ] Task: Implement StyleManager struct with StyleSheet, ThemeRegistry, Animator, HotReloadManager
- [ ] Task: Write tests for register_widget_defaults() parsing and merging
- [ ] Task: Implement register_widget_defaults::<W>() with DEFAULT_CSS parsing
- [ ] Task: Write tests for register_builtin_widgets() convenience function
- [ ] Task: Implement register_builtin_widgets() for all existing widgets
- [ ] Task: Write tests for user stylesheet loading and merging with defaults
- [ ] Task: Implement StyleManager::load_user_stylesheet() with proper specificity (is_user_css = 1)
- [ ] Task: Write tests for stylesheet merge order (defaults first, then user CSS)
- [ ] Task: Implement merged stylesheet construction in StyleManager::new() or builder
- [ ] Task: Write tests for theme_version tracking
- [ ] Task: Implement theme_version() and version increment on changes

### 3.2 Widget Trait Extensions
- [ ] Task: Write tests for Widget::DEFAULT_CSS default implementation
- [ ] Task: Add DEFAULT_CSS constant to Widget trait with empty default
- [ ] Task: Write tests for Widget::pseudo_classes() default implementation
- [ ] Task: Add pseudo_classes() method to Widget trait with empty default
- [ ] Task: Write tests for Widget::widget_meta() construction
- [ ] Task: Add widget_meta() method to Widget trait with default implementation

### 3.3 ComputedStyle Adapter
- [ ] Task: Write tests for ComputedStyle::to_ratatui_style() color conversion
- [ ] Task: Implement color → fg/bg conversion in to_ratatui_style()
- [ ] Task: Write tests for text_style → Modifier conversion
- [ ] Task: Implement text_style (bold, italic, underline) → Modifier conversion
- [ ] Task: Write tests for opacity handling (dim modifier)
- [ ] Task: Implement opacity → dim modifier conversion
- [ ] Task: Write comprehensive to_ratatui_style() integration tests

### 3.4 Style Computation Pipeline
- [ ] Task: Write tests for get_style() using compute_style_resolved()
- [ ] Task: Implement StyleManager::get_style() with proper resolution order
- [ ] Task: Write tests for ancestor chain construction (root→parent order)
- [ ] Task: Implement ancestor chain building during render traversal
- [ ] Task: Write tests for animated value overlay application
- [ ] Task: Implement animated value overlay in get_style()

### 3.5 Style Caching
- [ ] Task: Define ancestor hash inputs (widget_type, id, classes, pseudo_classes)
- [ ] Task: Write tests for ancestor hash determinism with same inputs
- [ ] Task: Implement ancestor_chain_hash() function with stable ordering
- [ ] Task: Write tests for style cache hit/miss behavior
- [ ] Task: Implement StyleCacheEntry with ancestor_hash and theme_version
- [ ] Task: Write tests for cache invalidation on class change
- [ ] Task: Implement invalidate_widget() for single widget invalidation
- [ ] Task: Write tests for cache invalidation on theme switch
- [ ] Task: Implement invalidate_all() for global invalidation
- [ ] Task: Write tests for ancestor change invalidation
- [ ] Task: Implement ancestor-aware cache invalidation

### 3.6 RenderContext
- [ ] Task: Write tests for RenderContext construction
- [ ] Task: Implement RenderContext struct with style_manager and ancestors
- [ ] Task: Write tests for RenderContext::style_for() widget lookup
- [ ] Task: Implement style_for() using StyleManager::get_style()
- [ ] Task: Write tests for parent_background propagation
- [ ] Task: Implement parent_background tracking through render tree

### 3.7 Style Invalidation Triggers
- [ ] Task: Write tests for class change triggering invalidation
- [ ] Task: Wire add_class()/remove_class() to invalidate_widget()
- [ ] Task: Write tests for pseudo-class change invalidation
- [ ] Task: Wire focus/hover/active state changes to invalidate_widget()
- [ ] Task: Write tests for mount/unmount style handling
- [ ] Task: Implement style computation on mount, cache removal on unmount
- [ ] Task: Write tests for refresh_styles_required flag integration
- [ ] Task: Wire invalidation to WidgetRefreshState.refresh_styles_required

### 3.8 Animator Integration
- [ ] Task: Write tests for StyleManager::tick() advancing animations
- [ ] Task: Implement tick() delegating to Animator
- [ ] Task: Write tests for StyleManager::animate() starting animations
- [ ] Task: Implement animate() API on StyleManager
- [ ] Task: Write tests for animated values appearing in get_style() result
- [ ] Task: Integrate Animator value lookup into get_style() pipeline

### 3.9 Hot Reload Integration
- [ ] Task: Write tests for enable_hot_reload() configuration
- [ ] Task: Implement enable_hot_reload() with path registration
- [ ] Task: Write tests for poll_hot_reload() detecting changes
- [ ] Task: Implement poll_hot_reload() with stylesheet reload
- [ ] Task: Write tests for hot reload triggering invalidate_all()
- [ ] Task: Wire poll_hot_reload() to invalidate_all() on changes

### 3.10 ManagedWidgetApp Integration
- [ ] Task: Write tests for ManagedWidgetApp::style_manager() access
- [ ] Task: Add style_manager() and style_manager_mut() to ManagedWidgetApp trait
- [ ] Task: Write tests for register_builtin_widgets() called during app init
- [ ] Task: Document app initialization sequence (register defaults → load user CSS → start)
- [ ] Task: Update run_managed() or App setup to call register_builtin_widgets()
- [ ] Task: Write tests for run_managed() ticking animator each frame
- [ ] Task: Update run_managed() to call style_manager.tick(dt)
- [ ] Task: Write tests for run_managed() polling hot reload
- [ ] Task: Update run_managed() to call poll_hot_reload() and handle changes
- [ ] Task: Write tests for RenderContext construction in render loop
- [ ] Task: Update render pipeline to construct and pass RenderContext

- [ ] Task: Conductor - User Manual Verification 'Implementation Complete' (Protocol in workflow.md)

## Phase 4: Widget Migration

### 4.1 Button Widget
- [ ] Task: Write tests for Button DEFAULT_CSS styling
- [ ] Task: Add DEFAULT_CSS to Button with variant styles
- [ ] Task: Write tests for Button pseudo-classes (focus, active, disabled)
- [ ] Task: Implement pseudo_classes() for Button
- [ ] Task: Update Button::render() to use RenderContext styles
- [ ] Task: Verify Button renders correctly with TCSS styles

### 4.2 Label Widget
- [ ] Task: Write tests for Label DEFAULT_CSS styling
- [ ] Task: Add DEFAULT_CSS to Label
- [ ] Task: Update Label::render() to use RenderContext styles
- [ ] Task: Verify Label renders correctly with TCSS styles

### 4.3 Input Widget
- [ ] Task: Write tests for Input DEFAULT_CSS styling
- [ ] Task: Add DEFAULT_CSS to Input with focus/placeholder styles
- [ ] Task: Write tests for Input pseudo-classes (focus, disabled)
- [ ] Task: Implement pseudo_classes() for Input
- [ ] Task: Update Input::render() to use RenderContext styles
- [ ] Task: Verify Input renders correctly with TCSS styles

- [ ] Task: Conductor - User Manual Verification 'Widget Migration Complete' (Protocol in workflow.md)

## Phase 5: Integration Testing & Verification

### 5.1 Integration Tests
- [ ] Task: Create integration test for full style computation pipeline
- [ ] Task: Create integration test for theme switching at runtime
- [ ] Task: Create integration test for animation flow (start → tick → render)
- [ ] Task: Create integration test for hot reload (file change → style update)
- [ ] Task: Create integration test for style invalidation cascade

### 5.2 Example Applications
- [ ] Task: Create styled_app example demonstrating TCSS integration
- [ ] Task: Create theme_switcher example with light/dark themes
- [ ] Task: Create animated_widgets example with CSS transitions
- [ ] Task: Update existing examples to use StyleManager where appropriate

### 5.3 Documentation
- [ ] Task: Document StyleManager API in module docs
- [ ] Task: Document RenderContext usage pattern
- [ ] Task: Document widget migration guide (hardcoded → TCSS)
- [ ] Task: Add TCSS integration section to crate-level docs

### 5.4 Final Verification
- [ ] Task: Run all tests and fix any failures
- [ ] Task: Verify backwards compatibility with unmodified widgets
- [ ] Task: Performance test style computation with 100+ widgets
- [ ] Task: Conductor - User Manual Verification 'Track Complete' (Protocol in workflow.md)
