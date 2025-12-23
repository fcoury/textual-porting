# TabbedContent and Tabs Research

## Architecture Overview

```
TabbedContent
├── ContentTabs (dock: top)
│   ├── Container (id="tabs-scroll")
│   │   └── Vertical (id="tabs-list-bar")
│   │       ├── Horizontal (id="tabs-list")
│   │       │   ├── Tab 1
│   │       │   ├── Tab 2
│   │       │   └── Tab N
│   │       └── Underline (animated indicator)
└── ContentSwitcher
    ├── TabPane 1
    ├── TabPane 2
    └── TabPane N
```

## Tab Class

### Properties
- `label: Content` - Display text (supports ContentText input)
- `label_text: str` - Plain text of the label
- `disabled: bool` - Whether tab is interactive

### Messages
| Message | Trigger |
|---------|---------|
| `Tab.Clicked` | Tab receives click event |
| `Tab.Disabled` | disabled property changes to true |
| `Tab.Enabled` | disabled property changes to false |
| `Tab.Relabelled` | Label content updates |

### DEFAULT_CSS
```css
Tab {
    width: auto;
    height: 1;
    text-align: center;
    padding: 0 1;
    /* Opacity states */
    opacity: 0.5;  /* inactive */
    &:hover { opacity: 1.0; }
    &.-active { opacity: 1.0; }
    &:disabled { opacity: 0.25; }
    &.-hidden { display: none; }
}
```

## Tabs Class (Tab Bar)

### Reactive Properties
- `active: str` - ID of currently active tab (empty string if none)

### Messages
| Message | Trigger |
|---------|---------|
| `TabActivated` | New tab becomes active |
| `Cleared` | No active tabs remain |
| `TabDisabled` | Tab is disabled |
| `TabEnabled` | Tab is enabled |
| `TabHidden` | Tab is hidden |
| `TabShown` | Tab is shown |

### Keyboard Bindings
```
Left Arrow  → action_previous_tab (wraps around)
Right Arrow → action_next_tab (wraps around)
```

Navigation skips disabled/hidden tabs.

### Methods
- `add_tab(tab, before=None, after=None)` - Insert tab
- `remove_tab(tab_id)` - Delete tab
- `clear()` - Remove all tabs
- `disable(tab_id)` / `enable(tab_id)` - Control interactivity
- `hide(tab_id)` / `show(tab_id)` - Toggle visibility

### DEFAULT_CSS
```css
Tabs {
    width: 1fr;
    height: 2;  /* 1 for tabs, 1 for underline */

    Underline {
        background: $foreground 10%;
        color: $block-cursor-background;
    }

    Tab.-active {
        color: $block-cursor-foreground;
    }
}
```

## Underline Class (Internal Widget)

The Underline widget is an internal component used by Tabs to render an animated indicator bar beneath the active tab.

### Reactive Properties
| Property | Type | Description |
|----------|------|-------------|
| `highlight_start` | int | Starting cell position for the highlight |
| `highlight_end` | int | Ending cell position (inclusive) for the highlight |
| `show_highlight` | bool | Controls visibility of the highlight bar |

### Messages
| Message | Trigger |
|---------|---------|
| `Underline.Clicked` | Underline receives click event (reports offset coordinates) |

### Methods
- `render()` - Returns a `Bar` renderable with highlight range and styled appearance

### Behavior
- Calculates highlight position from active tab's virtual region dimensions
- Renders using the `Bar` renderable for smooth visual appearance
- Click events report relative offset coordinates for tab detection

### DEFAULT_CSS
```css
Underline {
    width: 1fr;
    height: 1;

    > .underline--bar {
        color: $block-cursor-background;
        background: $foreground 10%;
    }

    &:ansi {
        text-style: dim;
    }
}
```

### Component Classes
- `underline--bar` - Styling hook for bar color customization

## TabPane Class

### Properties
- `title: str` - Tab title (becomes Tab label)
- `disabled: bool` - Whether pane is disabled

### Messages
| Message | Trigger |
|---------|---------|
| `Disabled` | disabled property changes to true |
| `Enabled` | disabled property changes to false |
| `Focused` | Descendant widget receives focus |

### DEFAULT_CSS
```css
TabPane {
    height: auto;
}
```

## TabbedContent Class

### Reactive Properties
- `active: str` - ID of currently active tab

### Messages
| Message | Content |
|---------|---------|
| `TabActivated` | tabbed_content, tab, pane |
| `Cleared` | tabbed_content |

### Content Management Methods
- `add_pane(pane, before=None, after=None)` → `AwaitComplete`
- `remove_pane(pane_id)` → `AwaitComplete`
- `clear_panes()` → `AwaitComplete`

### Retrieval Methods
- `get_tab(pane_id: str | TabPane)` → `Tab`
- `get_pane(pane_id: str | ContentTab)` → `TabPane`
- `active_pane` property → `TabPane | None`

### Tab Control Methods
- `disable_tab(tab_id)`
- `enable_tab(tab_id)`
- `hide_tab(tab_id)`
- `show_tab(tab_id)`

### DEFAULT_CSS
```css
TabbedContent {
    height: auto;

    > ContentTabs {
        dock: top;
    }
}
```

## ID Prefix System

TabbedContent uses an ID prefix system to coordinate tabs and panes:

- `ContentTab.add_prefix(id)` → `"--content-tab-" + id`
- `ContentTab.sans_prefix(id)` → removes prefix

This ensures:
1. Tab IDs don't conflict with pane IDs
2. Internal wiring remains coordinated
3. API accepts unprefixed IDs for simplicity

## Event Flow

1. **Tab Click**:
   ```
   Tab.Clicked → Tabs._on_click → TabActivated → TabbedContent._on_tabs_tab_activated
   → Updates switcher.current → Updates active property
   ```

2. **Active Property Change**:
   ```
   active.watch → _watch_active → Syncs tabs.active and switcher.current
   ```

3. **Pane Focus**:
   ```
   TabPane.Focused → TabbedContent._on_tab_pane_focused → Activates containing pane
   ```

## Implementation Plan for textual-rs

### Phase 1: Tab Widget
1. Create Tab struct with label, disabled properties
2. Implement Tab.Clicked message
3. Implement rendering with opacity states
4. Add DEFAULT_CSS

### Phase 2: Tabs Widget
1. Create Tabs struct with active reactive property
2. Implement keyboard navigation (arrow keys)
3. Implement add_tab, remove_tab, clear methods
4. Implement disable/enable, hide/show methods
5. Add TabActivated, Cleared messages
6. Add DEFAULT_CSS with underline

### Phase 3: TabPane Widget
1. Create TabPane struct with title, disabled properties
2. Implement as container for child widgets
3. Add Disabled/Enabled/Focused messages
4. Add DEFAULT_CSS

### Phase 4: TabbedContent Widget
1. Create TabbedContent struct with active property
2. Implement compose with ContentTabs + ContentSwitcher
3. Wire tab selection to content visibility
4. Implement add_pane, remove_pane, clear_panes
5. Implement get_tab, get_pane, active_pane
6. Add TabActivated, Cleared messages
7. Add DEFAULT_CSS

### Phase 5: Underline Widget
1. Create Underline struct with reactive properties:
   - `highlight_start: i16`
   - `highlight_end: i16`
   - `show_highlight: bool`
2. Implement render() with Bar renderable
3. Implement Underline.Clicked message with offset coordinates
4. Calculate highlight position from active tab's region
5. Add DEFAULT_CSS with component class `.underline--bar`
6. Write tests for highlight positioning
