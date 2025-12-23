# Container DEFAULT_CSS Patterns

## Summary

This document consolidates all DEFAULT_CSS patterns for container widgets from Python Textual.

## Layout Containers

### Container (Base)
```css
Container {
    width: 1fr;
    height: 1fr;
    layout: vertical;
}
```

### Vertical
```css
Vertical {
    width: 1fr;
    height: 1fr;
    layout: vertical;
}
```

### VerticalGroup (Non-expanding)
```css
VerticalGroup {
    width: 1fr;
    height: auto;
    layout: vertical;
    overflow: hidden hidden;
}
```

### Horizontal
```css
Horizontal {
    width: 1fr;
    height: 1fr;
    layout: horizontal;
}
```

### HorizontalGroup (Non-expanding)
```css
HorizontalGroup {
    width: 1fr;
    height: auto;
    layout: horizontal;
    overflow: hidden hidden;
}
```

### Grid
```css
Grid {
    width: 1fr;
    height: 1fr;
    layout: grid;
}
```

### ItemGrid (Dynamic Grid)
```css
ItemGrid {
    width: 1fr;
    height: auto;
    layout: grid;
}
```

**Note**: ItemGrid also has reactive properties that affect grid layout:
- `stretch_height`: Expands widgets to fill row height
- `min_column_width`: Minimum column width constraint
- `max_column_width`: Maximum column width constraint
- `regular`: Forces equal items per row

## Alignment Containers

### Center
```css
Center {
    align-horizontal: center;
    width: 1fr;
    height: auto;
}
```

### Middle
```css
Middle {
    align-vertical: middle;
    width: auto;
    height: 1fr;
}
```

### CenterMiddle
```css
CenterMiddle {
    align: center middle;
    width: 1fr;
    height: 1fr;
}
```

### Right
```css
Right {
    align-horizontal: right;
    width: 1fr;
    height: auto;
}
```

## Scrolling Containers

### ScrollableContainer
```css
ScrollableContainer {
    /* Inherits from Widget - no special CSS */
}
```

### VerticalScroll
```css
VerticalScroll {
    overflow-x: hidden;
    overflow-y: auto;
}
```

### HorizontalScroll
```css
HorizontalScroll {
    overflow-y: hidden;
    overflow-x: auto;
}
```

## Tabbed Widgets

### Tab
```css
Tab {
    width: auto;
    height: 1;
    text-align: center;
    padding: 0 1;
    opacity: 0.5;

    &:hover {
        opacity: 1.0;
    }

    &.-active {
        opacity: 1.0;
    }

    &:disabled {
        opacity: 0.25;
    }

    &.-hidden {
        display: none;
    }
}
```

### Tabs
```css
Tabs {
    width: 1fr;
    height: 2;

    Underline {
        background: $foreground 10%;
        color: $block-cursor-background;
    }

    Tab.-active {
        color: $block-cursor-foreground;
    }
}
```

### TabPane
```css
TabPane {
    height: auto;
}
```

### TabbedContent
```css
TabbedContent {
    height: auto;

    > ContentTabs {
        dock: top;
    }
}
```

**Note**: `ContentTabs` is an internal variant of `Tabs` used within TabbedContent for coordination. It inherits the same CSS as `Tabs`.

### Underline (Internal Widget)
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

**Note**: `Underline` is an internal widget used by `Tabs` to render the animated indicator bar beneath the active tab. It has reactive properties:
- `highlight_start`: Starting cell position for the highlight
- `highlight_end`: Ending cell position for the highlight
- `show_highlight`: Boolean controlling visibility

## Collapsible Widgets

### CollapsibleTitle
```css
CollapsibleTitle {
    width: auto;
    height: auto;
    padding: 0 1;

    &:hover {
        background: $surface-lighten-1;
        color: $text;
    }

    &:focus {
        background: $block-cursor-background;
        color: $block-cursor-foreground;
    }
}
```

### Collapsible
```css
Collapsible {
    width: 1fr;
    height: auto;

    &.-collapsed > Contents {
        display: none;
    }

    &:focus-within {
        background: $surface-lighten-1;
    }
}

Contents {
    width: 1fr;
    height: auto;
    padding-left: 1;
}
```

## ContentSwitcher
```css
ContentSwitcher {
    height: auto;
}
```

## Pattern Summary Table

| Widget | Width | Height | Layout | Special |
|--------|-------|--------|--------|---------|
| Container | 1fr | 1fr | vertical | - |
| Vertical | 1fr | 1fr | vertical | - |
| VerticalGroup | 1fr | auto | vertical | overflow: hidden hidden |
| Horizontal | 1fr | 1fr | horizontal | - |
| HorizontalGroup | 1fr | auto | horizontal | overflow: hidden hidden |
| Grid | 1fr | 1fr | grid | - |
| ItemGrid | 1fr | auto | grid | Reactive column properties |
| Center | 1fr | auto | - | align-horizontal: center |
| Middle | auto | 1fr | - | align-vertical: middle |
| CenterMiddle | 1fr | 1fr | - | align: center middle |
| Right | 1fr | auto | - | align-horizontal: right |
| VerticalScroll | - | - | - | overflow-x: hidden, overflow-y: auto |
| HorizontalScroll | - | - | - | overflow-y: hidden, overflow-x: auto |
| Tab | auto | 1 | - | Opacity states |
| Tabs | 1fr | 2 | - | Contains Underline |
| Underline | 1fr | 1 | - | Internal: animated highlight |
| TabPane | - | auto | - | - |
| TabbedContent | - | auto | - | ContentTabs dock: top |
| CollapsibleTitle | auto | auto | - | Hover/focus states |
| Collapsible | 1fr | auto | - | .-collapsed hides Contents |
| ContentSwitcher | - | auto | - | - |

## Key Observations

1. **Expanding vs Collapsing**:
   - `1fr` = Expand to fill available space
   - `auto` = Shrink to content size

2. **Layout Types**:
   - `vertical` for stacking top-to-bottom
   - `horizontal` for stacking left-to-right
   - `grid` for 2D layouts

3. **Scroll Containers**:
   - Control overflow-x and overflow-y independently
   - Use `auto` for show-when-needed, `hidden` for never

4. **State Classes**:
   - `.-active` for selected items
   - `.-collapsed` for hidden content
   - `.-hidden` for completely hidden elements
   - `:disabled` pseudo-class for inactive items

5. **Opacity Pattern**:
   - 0.5 = inactive
   - 1.0 = active/hover
   - 0.25 = disabled

6. **Variable References**:
   - `$foreground`, `$background` = theme colors
   - `$surface-lighten-1` = hover state
   - `$block-cursor-background/foreground` = focus state
   - `$text` = text color
