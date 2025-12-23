# ContentSwitcher Widget Research

## Overview

ContentSwitcher is a Container that manages visibility of multiple child widgets, showing only one at a time. All children must have unique IDs.

## Class Structure

```
ContentSwitcher (extends Container)
├── Child 1 (id="panel-1", display=True if current)
├── Child 2 (id="panel-2", display=False if not current)
└── Child N (id="panel-n", display=False if not current)
```

## Reactive Properties

### `current: str | None`
- Tracks ID of currently-displayed widget
- Setting to `None` hides all widgets
- Invalid IDs raise `NoMatches` exception
- Changes trigger `watch_current` to update visibility

## Properties

### `visible_content: Widget | None`
- Returns reference to currently-visible widget
- Returns `None` if no widget is visible

## Methods

### `__init__(self, *children, initial: str | None = None)`
- Accepts child widgets with unique IDs
- `initial` specifies which child to display first

### `add_content(widget: Widget, *, id: str, set_current: bool = False)`
- Dynamically adds widget with mandatory ID
- Optionally sets as current immediately

### `watch_current(old: str | None, new: str | None)`
- Reactive watcher for `current` property changes
- Hides old widget (sets display=False)
- Shows new widget (sets display=True)
- Batches UI updates for efficiency

### `_on_mount()`
- Handles initial setup when DOM is ready
- Displays initial widget
- Configures child visibility

## Messages

### `CurrentChanged`
- Posted when `current` property changes
- Contains reference to ContentSwitcher

## DEFAULT_CSS

```css
ContentSwitcher {
    height: auto;  /* Flexible based on content */
}
```

## Visibility Management

Children's visibility is controlled via their `display` CSS property:
- Current child: `display: block` (or default)
- Non-current children: `display: none`

Non-ID children remain permanently hidden.

## Usage Pattern

```python
# Create switcher with initial content
switcher = ContentSwitcher(
    Static("Page 1", id="page-1"),
    Static("Page 2", id="page-2"),
    Static("Page 3", id="page-3"),
    initial="page-1"
)

# Switch content
switcher.current = "page-2"

# Hide all content
switcher.current = None

# Add new content
switcher.add_content(
    Static("Page 4"),
    id="page-4",
    set_current=True
)
```

## Implementation Plan for textual-rs

### Phase 1: Core Structure
1. Create ContentSwitcher struct extending Container
2. Add `current: Option<String>` reactive property
3. Implement `visible_content()` → `Option<&Widget>`

### Phase 2: Visibility Management
1. Implement `watch_current()` reactive watcher
2. Toggle child widget display properties
3. Handle `None` case (hide all)
4. Handle invalid ID case (error or ignore)

### Phase 3: Dynamic Content
1. Implement `add_content(widget, id, set_current)`
2. Validate unique IDs
3. Set display property on mount

### Phase 4: Messages
1. Add `CurrentChanged` message
2. Post when `current` property changes

### Phase 5: DEFAULT_CSS
1. Set `height: auto`

## Integration with TabbedContent

ContentSwitcher is used internally by TabbedContent:

```
TabbedContent
├── ContentTabs (dock: top)
└── ContentSwitcher ← manages TabPane visibility
    ├── TabPane 1
    ├── TabPane 2
    └── TabPane N
```

When a tab is activated:
1. TabbedContent receives TabActivated message
2. Sets `switcher.current = pane_id`
3. ContentSwitcher shows corresponding TabPane

## Key Design Decisions

1. **ID Requirement**: All switchable children must have unique IDs
2. **None State**: `current = None` is valid (hides all)
3. **Error Handling**: Invalid IDs raise NoMatches in Python
4. **Height Auto**: Container sizes to visible content
5. **Batched Updates**: Visibility changes are batched for efficiency
