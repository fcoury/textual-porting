# Visual Parity Verification Process

This document describes how to verify visual parity between textual-rs and Python Textual.

## Automated Verification

### 1. Run Test Suite

```bash
cargo test
```

Expected: All 1938+ tests pass

### 2. Widget Snapshot Tests

Location: `tests/widget_snapshots.rs`

Tests cover:
- Button rendering (default, primary, warning, error, success variants)
- Button pseudo-classes (:focus, :hover, :active, :disabled)
- Button borders (CSS-driven, per-edge styles)
- Label rendering (default, custom CSS)
- Input rendering (default, placeholder, custom CSS)
- Digits rendering (numbers, time format, decimal, bold variant)

### 3. Calculator Integration Tests

Location: `tests/calculator_snapshot.rs`

Tests cover:
- All button labels render correctly (AC, operators, digits 0-9)
- Digits display updates when value changes
- Grid layout computes proper dimensions
- Overall calculator renders with expected structure

## Manual Verification

### 1. Visual Comparison Checklist

See `design/screenshot_checklist.md` for detailed per-example verification points.

### 2. Theme Verification

Run theme demo to verify color palettes:
```bash
cargo run --example theme_demo
```

Compare colors with Python Textual's themes.

### 3. Interactive Testing

For each example:
1. Verify focus states change visual appearance
2. Verify hover states (if applicable)
3. Verify disabled states reduce opacity
4. Check keyboard navigation works correctly

## CSS Parity Checklist

### Implemented CSS Properties

| Property | Parsing | Computation | Rendering |
|----------|---------|-------------|-----------|
| color | ✅ | ✅ | ✅ |
| background | ✅ | ✅ | ✅ |
| border (all variants) | ✅ | ✅ | ✅ |
| padding | ✅ | ✅ | ✅ |
| margin | ✅ | ✅ | ✅ |
| width/height | ✅ | ✅ | ✅ |
| min/max width/height | ✅ | ✅ | ✅ |
| content-align | ✅ | ✅ | ✅ |
| text-align | ✅ | ✅ | ✅ |
| line-pad | ✅ | ✅ | ✅ |
| tint | ✅ | ✅ | ✅ |
| background-tint | ✅ | ✅ | ✅ |
| text-opacity | ✅ | ✅ | ✅ |
| display | ✅ | ✅ | ✅ |
| text-style | ✅ | ✅ | ✅ |

### Pseudo-class Support

| Pseudo-class | Parsing | Matching | Widget Support |
|--------------|---------|----------|----------------|
| :focus | ✅ | ✅ | Button, Input |
| :hover | ✅ | ✅ | Button |
| :active | ✅ | ✅ | Button |
| :disabled | ✅ | ✅ | Button, Input |
| :inline | ✅ | ✅ | Widget trait |

### Border Styles

| Style | Character Set | Tests |
|-------|---------------|-------|
| solid | ─│┌┐└┘ | ✅ |
| dashed | ┄┆┌┐└┘ | ✅ |
| heavy | ━┃┏┓┗┛ | ✅ |
| double | ═║╔╗╚╝ | ✅ |
| round | ─│╭╮╰╯ | ✅ |
| tall | ▔▁█ | ✅ |
| wide | ▏▕ | ✅ |
| block | █ | ✅ |
| ascii | -|++++ | ✅ |
| inner | ▄▀ | ✅ |
| outer | ▀▄ | ✅ |
| hidden | spaces | ✅ |

## Known Gaps

No known parity gaps remain at this stage. Any intentional differences or
deferred items should be listed in plan.md "Remaining Parity Gaps".
