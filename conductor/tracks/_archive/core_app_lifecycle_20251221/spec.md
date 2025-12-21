# Specification: Core App Lifecycle

## Overview
This track implements the foundational application lifecycle that all Textual apps depend on. It establishes the patterns for widget composition, mounting, screen management, and message passing that enable reactive, event-driven TUI applications.

## Goals
- Implement the `compose()` pattern for declarative widget tree construction
- Enable dynamic widget mounting and unmounting at runtime
- Create a screen stack system for modal dialogs and navigation
- Build an async message pump for event dispatch
- Implement key bindings and action system
- Add reactive properties that trigger updates on change
- Provide DOM query system for widget lookup

## Reference: Python Textual Architecture

### App Class Core Methods
```python
class App:
    def compose(self) -> ComposeResult:
        """Yield child widgets to compose the app."""
        yield Header()
        yield Container(
            Static("Hello"),
            Button("Click me"),
        )
        yield Footer()

    async def on_mount(self) -> None:
        """Called when app is mounted."""
        pass

    async def action_quit(self) -> None:
        """Action to quit the app."""
        self.exit()
```

### Screen Stack
```python
# Push a new screen
self.push_screen(SettingsScreen())

# Pop current screen
self.pop_screen()

# Switch to named screen
self.switch_screen("main")
```

### Message Passing
```python
class Button:
    class Pressed(Message):
        """Button was pressed."""
        pass

    def press(self) -> None:
        self.post_message(self.Pressed())

class App:
    def on_button_pressed(self, event: Button.Pressed) -> None:
        """Handle button press."""
        pass
```

### Bindings
```python
class MyApp(App):
    BINDINGS = [
        Binding("q", "quit", "Quit"),
        Binding("d", "toggle_dark", "Toggle dark mode"),
    ]
```

### Reactive Properties
```python
class Counter(Widget):
    count: reactive[int] = reactive(0)

    def watch_count(self, new_value: int) -> None:
        """Called when count changes."""
        self.refresh()
```

## Deliverables

### Phase 1: Analysis & Research
- Study Python Textual app.py, compose.py, message_pump.py
- Document the compose() yield pattern
- Understand mount lifecycle and timing
- Map out message dispatch flow

### Phase 2: Design & Planning
- Design Rust compose pattern (iterators vs builders)
- Plan async message pump architecture
- Design screen stack with lifetime management
- Plan reactive property macro/derive approach

### Phase 3: Implementation

#### 3.1 Compose Pattern
- `ComposeResult` type (iterator of widgets)
- `compose()` method on App and Widget
- Automatic mounting of composed widgets
- Container widget for grouping

#### 3.2 Mount/Unmount System
- `mount()` method for dynamic widget addition
- `unmount()` method for widget removal
- `on_mount()` / `on_unmount()` lifecycle callbacks
- Widget tree parent/child tracking

#### 3.3 Screen Stack
- `Screen` base type
- `push_screen()` / `pop_screen()` / `switch_screen()`
- Screen result callbacks
- Modal screen support

#### 3.4 Message Pump
- `Message` trait with bubble/capture
- `post_message()` for async dispatch
- Handler lookup by message type
- Event bubbling through widget tree

#### 3.5 Bindings & Actions
- `Binding` struct (key, action, description)
- `BINDINGS` constant on App/Widget
- `action_*` method convention
- Binding display in Footer

#### 3.6 Reactive Properties
- `reactive!` or `#[derive(Reactive)]` macro
- `watch_*` method pattern
- Automatic refresh triggering
- Computed reactive values

#### 3.7 DOM Queries
- `query()` method returning iterator
- `query_one()` for single widget
- CSS selector support
- Type-safe widget casting

### Phase 4: Testing & Verification
- Unit tests for each component
- Integration test with full app lifecycle
- Example app demonstrating all features
- Manual verification of screen transitions

## Success Criteria
- [ ] Apps can use compose() to declare widget structure
- [ ] Widgets can be mounted/unmounted dynamically
- [ ] Screen stack enables modal dialogs
- [ ] Messages bubble through widget tree
- [ ] Key bindings trigger actions
- [ ] Reactive properties auto-update UI
- [ ] DOM queries find widgets by selector

## Dependencies
- Existing textual-rs foundation (widget, layout, focus)
- No external track dependencies (this is Tier 1)

## Out of Scope
- CSS styling (separate track)
- Layout algorithms (separate track)
- Individual widgets (separate tracks)
- Animations and transitions
