# Manual Screenshot Capture Checklist

This document provides a checklist for capturing screenshots of textual-rs examples
to verify visual parity with Python Textual.

## Prerequisites

1. Terminal with 24-bit color support (iTerm2, Alacritty, Kitty, etc.)
2. Terminal size: 80x24 minimum (larger recommended for galleries)
3. Dark theme for terminal background (matches default textual-dark theme)
4. Screenshot tool: `tmux capture-pane -p` or native OS screenshot

## Examples to Capture

### 1. Calculator (`cargo run --example calculator`)

**Terminal size:** 40x25 (or larger)

**Verification points:**
- [ ] Grid layout shows 4 columns x 6 rows
- [ ] Digits display at top with rounded corners ("0" on startup)
- [ ] Button borders render with tall border style (▔/▁)
- [ ] Primary buttons (AC, +/-, %) have distinct color
- [ ] Warning buttons (÷, ×, -, +, =) have warning color
- [ ] Number buttons (0-9) have default styling
- [ ] "0" button spans 2 columns (wider than others)
- [ ] Row/column gaps create visual separation

**Interactive checks:**
- [ ] Press numbers 1-9, verify digits update
- [ ] Press operators (+, -, *, /), verify calculation works
- [ ] Press = to evaluate
- [ ] Arrow keys move focus (focus ring visible)
- [ ] Space activates focused button

---

### 2. Styled App (`cargo run --example styled_app`)

**Terminal size:** 60x20

**Verification points:**
- [ ] Title label uses accent color
- [ ] Subtitle label uses muted text color
- [ ] Input field has surface background
- [ ] Buttons show primary background
- [ ] Focus state shows lighter background + bold text

**Interactive checks:**
- [ ] Tab cycles through Input → Submit → Cancel
- [ ] Focused widget has distinct styling
- [ ] Enter on Submit updates info label

---

### 3. Theme Demo (`cargo run --example theme_demo`)

**Terminal size:** 80x30

**Verification points:**
- [ ] Theme list on left side
- [ ] Color swatches show theme palette
- [ ] CSS variables display correctly
- [ ] Theme preview area updates when switching

**Interactive checks:**
- [ ] Arrow keys navigate theme list
- [ ] Each theme shows distinct color palette
- [ ] Both dark and light themes render correctly

---

### 4. Widget Gallery (`cargo run --example widget_gallery`)

**Terminal size:** 100x40

**Verification points:**
- [ ] All widget types render (Button, Input, Label, Checkbox, etc.)
- [ ] Widget variants display correctly
- [ ] Disabled widgets have reduced opacity

---

### 5. Hello World (`cargo run --example hello_world`)

**Terminal size:** 40x15

**Verification points:**
- [ ] "Textual-RS" block with borders
- [ ] "Hello World!" centered text
- [ ] Help text at bottom

---

## Capture Process

1. Run the example:
   ```bash
   cargo run --example <name>
   ```

2. Resize terminal to recommended size

3. Capture screenshot:
   - macOS: Cmd+Shift+4, then select terminal window
   - Linux: Use `scrot` or `gnome-screenshot`
   - tmux: `tmux capture-pane -ep > screenshot.txt`

4. Save to `screenshots/<example_name>_<date>.png`

5. Compare with Python Textual reference (if available)

## Reference Capture (Python Textual)

For visual parity comparison, also capture Python equivalents:

```bash
cd python-textual/examples
python -m textual run calculator.py
```

Save Python screenshots alongside Rust screenshots for comparison.

## Known Differences

Document any intentional differences from Python Textual:
- Border character sets may vary slightly
- Font rendering depends on terminal
- Some advanced widgets not yet implemented
