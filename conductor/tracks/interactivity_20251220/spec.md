# Specification: Interactivity

## Goal
Enable users to interact with widgets via Keyboard (Tab navigation, Shortcuts) and Mouse (Clicks, Hover).

## Requirements
- **Focus Manager:** Track which widget is currently active (receiving keyboard input).
- **Tab Order:** Mechanism to cycle focus via `Tab` / `Shift+Tab`.
- **Mouse Handling:** Hit-testing (mapping screen coordinates to Widgets) and dispatching Click/Hover events.
- **Key Bindings:** System to map key combinations to abstract Actions (e.g., `Ctrl+Q` -> `Action::Quit`).

## Success Criteria
- Tests verify `set_focus` changes the active widget ID.
- Tests verify a Mouse Click at (X, Y) triggers the `OnClick` handler of the widget at that location.
