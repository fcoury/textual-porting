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

### VerticalGroup (Collapsing)
```css
VerticalGroup {
    width: auto;
    height: auto;
    layout: vertical;
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

### HorizontalGroup (Collapsing)
```css
HorizontalGroup {
    width: auto;
    height: auto;
    layout: horizontal;
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
| VerticalGroup | auto | auto | vertical | Shrink-to-fit |
| Horizontal | 1fr | 1fr | horizontal | - |
| HorizontalGroup | auto | auto | horizontal | Shrink-to-fit |
| Grid | 1fr | 1fr | grid | - |
| Center | 1fr | auto | - | align-horizontal: center |
| Middle | auto | 1fr | - | align-vertical: middle |
| CenterMiddle | 1fr | 1fr | - | align: center middle |
| Right | 1fr | auto | - | align-horizontal: right |
| VerticalScroll | - | - | - | overflow-x: hidden, overflow-y: auto |
| HorizontalScroll | - | - | - | overflow-y: hidden, overflow-x: auto |
| Tab | auto | 1 | - | Opacity states |
| Tabs | 1fr | 2 | - | Contains Underline |
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
