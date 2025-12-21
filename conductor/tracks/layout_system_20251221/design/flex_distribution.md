# Design: Constraint Solving for Flex Distribution

## Overview

This document describes the algorithm for distributing space among widgets that use fractional (`fr`) units, while respecting min/max constraints. This is the core of CSS Grid/Flexbox-like flexible sizing.

## Problem Statement

Given:
- A container with available space `total_space`
- A list of widgets, each with:
  - `size`: Can be fixed (cells), fractional (fr), percentage (%), or auto
  - `min_size`: Optional minimum constraint
  - `max_size`: Optional maximum constraint

Compute the final size for each widget such that:
1. Fixed sizes are honored exactly
2. Fractional widgets share remaining space proportionally
3. Min/max constraints are never violated
4. Total used space equals available space (if possible)

## Algorithm: Iterative Constraint Resolution

The key insight is that min/max constraints can change the distribution. When a widget hits its min or max, it's "locked" and remaining space is redistributed among unlocked widgets.

### Phase 1: Classify Widgets

```rust
#[derive(Debug, Clone)]
struct FlexItem {
    id: WidgetId,
    /// The size specification from styles
    size: SizeSpec,
    /// Resolved fixed size (if not fractional)
    resolved: Option<u16>,
    /// Fraction value (if fractional)
    fraction: Option<f32>,
    /// Minimum size constraint
    min: u16,
    /// Maximum size constraint (u16::MAX if none)
    max: u16,
    /// Whether this item is locked (hit min/max)
    locked: bool,
}

enum SizeSpec {
    Fixed(u16),           // width: 50
    Fraction(f32),        // width: 2fr
    Percent(f32),         // width: 50%
    Auto,                 // width: auto
}
```

### Phase 2: Resolve Non-Fraction Sizes

```rust
/// Resolve non-fraction sizes.
///
/// IMPORTANT: `container_size` is the original container size (before gutters).
/// Percent sizes are relative to container, not available space after gutters.
/// This matches CSS/Textual behavior.
fn resolve_non_fractions(
    items: &mut [FlexItem],
    container_size: u16,  // Original container size (for percent calculation)
    context: &LayoutContext,
) -> u16 {
    let mut consumed = 0u16;

    for item in items.iter_mut() {
        match item.size {
            SizeSpec::Fixed(size) => {
                let clamped = size.clamp(item.min, item.max);
                item.resolved = Some(clamped);
                item.locked = true;
                consumed += clamped;
            }
            SizeSpec::Percent(pct) => {
                // IMPORTANT: Percent is relative to CONTAINER size, not available space
                // This matches CSS/Textual behavior where 50% always means 50% of container
                let size = ((pct / 100.0) * container_size as f32) as u16;
                let clamped = size.clamp(item.min, item.max);
                item.resolved = Some(clamped);
                item.locked = true;
                consumed += clamped;
            }
            SizeSpec::Auto => {
                // Auto-sized items get their content size
                let content = context.content_size(item.id);
                let clamped = content.clamp(item.min, item.max);
                item.resolved = Some(clamped);
                item.locked = true;
                consumed += clamped;
            }
            SizeSpec::Fraction(fr) => {
                item.fraction = Some(fr);
                // Will be resolved in next phase
            }
        }
    }

    consumed
}
```

### Phase 3: Distribute Remaining Space

This is the iterative constraint resolution loop:

```rust
/// Distribute remaining space among fractional items.
///
/// IMPORTANT: This handles the case where min constraints exceed available space.
/// In that case, items get their min size and we accept overflow.
fn distribute_fractions(
    items: &mut [FlexItem],
    remaining_space: i32,  // Can be negative if fixed items exceed available
) {
    let mut space_left = remaining_space as f32;

    // OVERFLOW HANDLING: If space is already negative or zero,
    // fractional items get their minimum size (or 0 if no min).
    // This can result in overflow which the container handles via scrolling/clipping.
    if space_left <= 0.0 {
        for item in items.iter_mut() {
            if item.locked || item.fraction.is_none() {
                continue;
            }
            // Give each fractional item its minimum size (accepting overflow)
            item.resolved = Some(item.min);
            item.locked = true;
        }
        return;
    }

    loop {
        // Calculate total fraction units among unlocked items
        let total_fr: f32 = items.iter()
            .filter(|i| !i.locked && i.fraction.is_some())
            .map(|i| i.fraction.unwrap())
            .sum();

        // No fractional items left
        if total_fr <= 0.0 {
            break;
        }

        // Calculate size of 1fr (guaranteed positive since we checked space_left > 0)
        let fr_unit = (space_left / total_fr).max(0.0);
        let mut changed = false;

        for item in items.iter_mut() {
            if item.locked || item.fraction.is_none() {
                continue;
            }

            let fr = item.fraction.unwrap();
            let ideal_size = (fr * fr_unit) as u16;

            // Check min constraint
            if ideal_size < item.min {
                item.resolved = Some(item.min);
                item.locked = true;
                space_left -= item.min as f32;
                // If this drives space negative, subsequent items will get 0 from fr_unit.max(0)
                changed = true;
            }
            // Check max constraint
            else if ideal_size > item.max {
                item.resolved = Some(item.max);
                item.locked = true;
                space_left -= item.max as f32;
                changed = true;
            }
        }

        // If no constraints were hit, we can resolve all remaining
        if !changed {
            for item in items.iter_mut() {
                if item.locked || item.fraction.is_none() {
                    continue;
                }
                let fr = item.fraction.unwrap();
                let size = (fr * fr_unit) as u16;
                item.resolved = Some(size);
                item.locked = true;
            }
            break;
        }
    }

    // Handle edge case: all items locked by constraints
    // Distribute any remaining space to the last unlocked item (if any)
    // or leave it as unused space
}
```

## Complete Algorithm

```rust
pub fn resolve_flex_sizes(
    context: &LayoutContext,
    items: &[WidgetId],
    dimension: Dimension,  // Width or Height
    total_space: u16,
    gutter: u16,  // Gap between items
) -> Vec<(WidgetId, u16)> {
    // Calculate total gutter space
    let total_gutter = if items.len() > 1 {
        gutter * (items.len() as u16 - 1)
    } else {
        0
    };
    let available = total_space.saturating_sub(total_gutter);

    // Build flex items from widget styles
    let mut flex_items: Vec<FlexItem> = items.iter()
        .map(|&id| {
            let styles = context.styles(id);
            let (size, min, max) = match dimension {
                Dimension::Width => (
                    styles.width.clone(),
                    styles.min_width.unwrap_or(0),
                    styles.max_width.unwrap_or(u16::MAX),
                ),
                Dimension::Height => (
                    styles.height.clone(),
                    styles.min_height.unwrap_or(0),
                    styles.max_height.unwrap_or(u16::MAX),
                ),
            };
            FlexItem {
                id,
                size,
                resolved: None,
                fraction: None,
                min,
                max,
                locked: false,
            }
        })
        .collect();

    // Phase 1: Resolve non-fraction sizes
    // IMPORTANT: Pass total_space (container size) for percent calculation,
    // not available (which has gutters subtracted)
    let consumed = resolve_non_fractions(&mut flex_items, total_space, context);

    // Phase 2: Distribute remaining space to fractions
    // Use i32 to handle overflow case where consumed > available
    let remaining = available as i32 - consumed as i32;
    distribute_fractions(&mut flex_items, remaining);

    // Return results
    flex_items.iter()
        .map(|item| (item.id, item.resolved.unwrap_or(0)))
        .collect()
}
```

## Example Walkthrough

**Scenario**: Container width = 100, 3 widgets:
- Widget A: `width: 1fr`, `min-width: 30`
- Widget B: `width: 2fr`
- Widget C: `width: 1fr`, `max-width: 20`

**Iteration 1**:
- Total fr = 4 (1 + 2 + 1)
- fr_unit = 100 / 4 = 25
- Widget A: 1 * 25 = 25 (but min is 30, so lock at 30)
- Widget B: 2 * 25 = 50 (OK)
- Widget C: 1 * 25 = 25 (but max is 20, so lock at 20)
- Remaining = 100 - 30 - 20 = 50

**Iteration 2**:
- Total fr = 2 (only Widget B unlocked)
- fr_unit = 50 / 2 = 25
- Widget B: 2 * 25 = 50 (OK, lock)

**Result**: A=30, B=50, C=20 (total=100)

## Edge Cases

### All Fixed Sizes Exceed Available Space
- Widgets are sized to their min/max constraints
- May result in overflow (total > available)
- Parent handles overflow via scrolling or clipping

### No Fractional Units
- All widgets are fixed/percent/auto
- Remaining space is unused (or distributed based on alignment)

### Zero Fraction Total
- Guard against division by zero
- fr_unit = 0 when no fr items

### Negative Remaining Space
- When fixed items exceed available space
- Fractional items get 0 size (or their min)

## Integration with Layouts

### VerticalLayout
```rust
impl Layout for VerticalLayout {
    fn arrange(&self, context: &LayoutContext, ...) -> Vec<WidgetPlacement> {
        // Resolve heights using flex distribution
        let heights = resolve_flex_sizes(
            context, children, Dimension::Height, size.height, 0
        );

        // Position widgets top-to-bottom
        let mut y = 0;
        heights.iter().map(|(id, height)| {
            let placement = WidgetPlacement {
                widget_id: *id,
                region: Rect::new(0, y, size.width, *height),
                ...
            };
            y += *height as i16;
            placement
        }).collect()
    }
}
```

### HorizontalLayout
```rust
impl Layout for HorizontalLayout {
    fn arrange(&self, context: &LayoutContext, ...) -> Vec<WidgetPlacement> {
        // Resolve widths using flex distribution
        let widths = resolve_flex_sizes(
            context, children, Dimension::Width, size.width, 0
        );

        // Position widgets left-to-right
        let mut x = 0;
        widths.iter().map(|(id, width)| {
            let placement = WidgetPlacement {
                widget_id: *id,
                region: Rect::new(x, 0, *width, size.height),
                ...
            };
            x += *width as i16;
            placement
        }).collect()
    }
}
```

### GridLayout
Grid uses flex distribution twice:
1. Resolve column widths
2. Resolve row heights

```rust
// Resolve columns
let columns = resolve_flex_sizes(
    context, &column_scalars, Dimension::Width, size.width, gutter_vertical
);

// Resolve rows (after columns, because auto-height may depend on width)
let rows = resolve_flex_sizes(
    context, &row_scalars, Dimension::Height, size.height, gutter_horizontal
);
```

## Design Decisions

### Why Iterative?
A single-pass approach can't handle cases where hitting a constraint changes the distribution for other items. The iterative approach converges quickly (usually 1-2 iterations).

### Why Lock on Constraint Hit?
Once a widget is locked at min/max, it shouldn't participate in further distribution. This ensures constraints are respected and the algorithm terminates.

### Integer vs Float
We use `f32` for intermediate calculations to avoid accumulating rounding errors, then convert to `u16` for final sizes.
