# Plan: Extended Color Parsing and Variable Resolution

## Overview

Three tasks to complete TCSS parity with Python Textual:

1. **Extend parse_color()** - Add support for #RRGGBBAA, #RGB, rgb(), hsl() formats
2. **Token-level variable substitution** - Move from line-based to token-based resolution
3. **Support `auto 87%` format** - Enable $text / $text-muted usage

**Decision:** All three tasks will be implemented.

---

## Task 1: Replace Color Enum with RgbaColor

### Problem

`parse_color()` (line 828) returns simple `Color` enum. `RgbaColor::parse()` (lines 5218-5370) already supports all formats.

### Solution: Replace Color with RgbaColor

**Migration approach:**

```rust
// Add type alias for backward compatibility
pub type Color = RgbaColor;
// Then remove old Color enum (lines 797-802)
```

### Files & Locations to Modify

**`textual-rs/src/style.rs`:**

| Component                         | Line       | Change                                       |
| --------------------------------- | ---------- | -------------------------------------------- |
| `Color` enum                      | 797-802    | Replace with `pub type Color = RgbaColor;`   |
| `parse_color()`                   | 828-830    | Return `RgbaColor`, use `RgbaColor::parse()` |
| `parse_hex_color()`               | 814-820    | Remove (absorbed into RgbaColor::parse)      |
| `parse_named_color()`             | 823-825    | Remove (absorbed into RgbaColor::parse)      |
| `Declaration::Color`              | ~2135      | Change to `Color(RgbaColor)`                 |
| `Declaration::Background`         | ~2136      | Change to `Background(RgbaColor)`            |
| `Declaration::Scrollbar*`         | ~2175-2185 | All scrollbar color variants → RgbaColor     |
| `ComputedStyle.color`             | 4358       | Change to `Option<RgbaColor>`                |
| `ComputedStyle.background`        | 4360       | Change to `Option<RgbaColor>`                |
| `StyleValue::Color`               | 2924       | Change to `Color(RgbaColor)`                 |
| `impl Animatable for Color`       | 2851-2871  | Update to work with RgbaColor                |
| `impl From<Color> for StyleValue` | 3036-3039  | Update for RgbaColor                         |
| Border parsing                    | ~554-571   | Update `parse_border_edge()`                 |

### Key Discovery

`RgbaColor::parse()` (line 5225) **already supports all formats**:

- auto/auto 87% (lines 5234-5236 → `parse_auto()` at 5266-5298)
- rgb()/rgba() (lines 5249-5251 → `parse_rgb_function()`)
- hsl()/hsla() (lines 5253-5256 → `parse_hsl_function()`)
- **#RGB, #RGBA, #RRGGBB, #RRGGBBAA** via `parse_hex()` (lines 5463-5510)
- Named colors (lines 5258-5261)
- `contrast_text()` exists at line 5709

**Verified:** `parse_hex()` at lines 5476-5487 handles 4-digit #RGBA:

```rust
4 => {
    // #RGBA -> #RRGGBBAA
    let r = u8::from_str_radix(&hex[0..1].repeat(2), 16)...
    let a = u8::from_str_radix(&hex[3..4].repeat(2), 16)...
    Ok(RgbaColor::rgba(r, g, b, a as f32 / 255.0))
}
```

**Task 1 is primarily wiring**, not implementing new parsers.

### Implementation Steps

1. **Add type alias** (backward compat):

   ```rust
   pub type Color = RgbaColor;
   ```

2. **Update `parse_color()`** - CRITICAL: must handle multi-token colors like "auto 87%":

   ```rust
   pub fn parse_color(input: &str) -> IResult<&str, RgbaColor> {
       let input = input.trim_start();

       // Multi-token colors: "auto 87%", "rgb(1,2,3)"
       // Find end: ; or } or /* (comment start) but NOT whitespace (auto 87% has space)
       // For function colors, find matching )
       let end = if input.starts_with("rgb(") || input.starts_with("rgba(")
                 || input.starts_with("hsl(") || input.starts_with("hsla(") {
           // Find closing paren
           input.find(')').map(|i| i + 1).unwrap_or(input.len())
       } else if input.starts_with("auto") {
           // CRITICAL: "auto" or "auto NN%" - DO NOT split on whitespace!
           // Must find ; or } or /* to capture the full "auto 87%" token
           // Handle: `color: auto 87% /* note */;`
           find_color_end(input)
       } else {
           // Simple tokens: #hex, named color - CAN split on whitespace
           input.find(|c: char| c.is_whitespace() || c == ';' || c == '}')
               .unwrap_or(input.len())
       };

       let color_str = input[..end].trim();
       match RgbaColor::parse(color_str) {
           Ok(color) => Ok((&input[end..], color)),
           Err(_) => Err(nom::Err::Error(nom::error::Error::new(
               input, nom::error::ErrorKind::Tag
           ))),
       }
   }

   /// Find end of color value, stopping at ; } or /* (comment start)
   fn find_color_end(input: &str) -> usize {
       let mut i = 0;
       let bytes = input.as_bytes();
       while i < bytes.len() {
           match bytes[i] {
               b';' | b'}' => return i,
               b'/' if i + 1 < bytes.len() && bytes[i + 1] == b'*' => return i,
               _ => i += 1,
           }
       }
       input.len()
   }
   ```

   **Warning:** If the `auto` branch splits on whitespace, `auto 87%` will regress to just `auto`.
   **Warning:** Must stop at `/*` to handle `color: auto 87% /* note */;` correctly.

   **Known limitation:** Comments after function colors (e.g., `rgb(1,2,3) /* note */`) will still fail because `parse_declaration` doesn't strip comments mid-value. This is pre-existing behavior. Task 2's tokenizer can address this by stripping comments before parsing, but it's out of scope for Task 1.

3. **Update Declaration enum** color variants to use RgbaColor

4. **Update ComputedStyle** color fields

5. **Update StyleValue::Color and Animatable impl**:

   ```rust
   impl Animatable for RgbaColor {
       fn blend(&self, other: &Self, factor: f64) -> Self {
           // ANSI or auto colors: snap at 50% (no smooth interpolation)
           if self.is_ansi() || self.auto || other.is_ansi() || other.auto {
               return if factor < 0.5 { self.clone() } else { other.clone() };
           }
           // RGB interpolation with alpha
           RgbaColor::rgba(
               (self.r as f64 + (other.r as f64 - self.r as f64) * factor) as u8,
               (self.g as f64 + (other.g as f64 - self.g as f64) * factor) as u8,
               (self.b as f64 + (other.b as f64 - self.b as f64) * factor) as u8,
               (self.a + (other.a - self.a) * factor as f32),
           )
       }
   }
   ```

6. **Update border parsing** to use RgbaColor

7. **Remove old Color enum** and helper functions

8. **Update existing tests** that reference `Color::Named` / `Color::Rgb`:

   **~70+ test assertions** use `Color::Named` or `Color::Rgb` patterns. Key areas:

   | Line Range  | Test Area                        | Count |
   | ----------- | -------------------------------- | ----- |
   | 6890-6970   | parse_color, Declaration parsing | ~15   |
   | 7337-7425   | ComputedStyle.apply()            | ~15   |
   | 7682-7890   | Border parsing                   | ~10   |
   | 8312-8430   | Scrollbar color parsing          | ~12   |
   | 10147-10165 | Color blending                   | ~6    |
   | 10877-10905 | Animation interpolation          | ~8    |
   | 11401-11870 | StyleValue with Color            | ~20   |

   **Migration pattern:**

   ```rust
   // OLD: Color::Named("red".to_string())
   // NEW: Use RgbaColor::parse() (from_named is private)
   assert_eq!(color, RgbaColor::parse("red").unwrap());
   // OR compare RGB values directly (red = 255,0,0)
   assert_eq!(color.r, 255);
   assert_eq!(color.g, 0);
   assert_eq!(color.b, 0);

   // OLD: Color::Rgb(255, 0, 0)
   // NEW: Direct replacement
   assert_eq!(color, RgbaColor::rgb(255, 0, 0));

   // OLD: match Color::Rgb(r, g, b) => ...
   // NEW: Direct field access
   let RgbaColor { r, g, b, .. } = color;
   ```

   **Strategy:** After removing `Color` enum, run `cargo test` - compiler errors will identify each assertion needing update.

   **IMPORTANT:** All ~70+ test updates MUST be done in the same PR as the Color→RgbaColor migration. Otherwise the test suite will fail and block CI.

9. **Add new tests** for all color formats

---

## Task 2: Token-Level Variable Substitution

### Problem

Current `resolve_source_variables()` (line 4911) is line-based and substitutes `$` everywhere:

- In selectors: `Button.$class` → broken
- In comments: `/* $5 price */` → undefined variable error
- In strings: `content: "$5";` → undefined variable error

### Solution

Create a context-aware tokenizer that only resolves variables in property value positions.

### Files & Locations to Modify

**`textual-rs/src/style.rs`:**

| Component                    | Line      | Change                                |
| ---------------------------- | --------- | ------------------------------------- |
| `resolve_source_variables()` | 4911-4938 | Replace with tokenizer-based approach |
| `parse_with_variables()`     | 4058-4116 | Use new tokenizer                     |
| New: `CssToken` enum         | -         | Add token types                       |
| New: `tokenize_css()`        | -         | Add context-aware tokenizer           |

### Implementation Steps

1. **Add span-based CssToken enum** (critical for lossless reconstruction):

   ```rust
   /// A token with its exact source slice for lossless reconstruction
   struct CssToken<'a> {
       kind: CssTokenKind,
       /// Exact source slice including whitespace/punctuation.
       /// CRITICAL: This is a direct slice of input, NOT processed.
       source: &'a str,
       /// Line number where this token starts (1-based)
       line: usize,
       /// Column number where this token starts (1-based, char count)
       column: usize,
   }

   enum CssTokenKind {
       Comment,              // /* ... */ - NO substitution
       VariableDef,          // $name: value; - extract but remove from output
       Selector,             // Button, .class, #id - NO substitution
       PropertyName,         // background, color - NO substitution
       PropertyValue,        // ← resolve $ here (strings skipped internally)
       AtRule,               // @keyframes name { ... }
       Punctuation,          // {, }, :, ; - preserve exactly
       Whitespace,           // spaces, newlines - preserve exactly
   }
   ```

   **Source slice population rules** (ensures lossless reconstruction):
   - Each token's `source` is `&input[start..end]` - a direct slice, never copied/modified
   - Whitespace between tokens is captured as separate `Whitespace` tokens
   - Punctuation (`{`, `}`, `:`, `;`) captured as separate `Punctuation` tokens
   - Composite selectors (e.g., `Button > .class:hover`) are a SINGLE `Selector` token
     - `source` = `"Button > .class:hover"` (entire selector text)
   - Property values with spaces (e.g., `1 2 3 4`) are a SINGLE `PropertyValue` token
     - `source` = `"1 2 3 4"` (entire value text)
   - Concatenating all `token.source` in order reproduces input exactly (minus VariableDef)

   **CRITICAL: Comments inside property values must be split out:**
   - Input: `color: red /* note */ blue;`
   - Tokens: `[PropertyName("color"), Punctuation(":"), Whitespace(" "), PropertyValue("red "), Comment("/* note */"), PropertyValue(" blue"), Punctuation(";")]`
   - This ensures `resolve_variables_string_aware()` never sees `$` inside comments
   - Without this split, `color: $var /* $5 */;` would fail on `$5`

2. **Create tokenizer with state machine and brace-depth tracking:**

   ```rust
   fn tokenize_css(input: &str) -> Vec<CssToken> {
       enum State {
           TopLevel,        // Outside any rule
           Selector,        // Reading selector (until {)
           PropertyName,    // After { or ;, reading property name (until :)
           PropertyValue,   // After :, reading value (until ; or })
           Comment,         // Inside /* ... */
           AtRuleHeader,    // After @, reading @keyframes name (until {)
           AtRuleBody,      // Inside @keyframes { ... }
       }

       let mut state = State::TopLevel;
       let mut brace_depth = 0;  // Track nested braces
       let mut at_rule_depth = 0; // Track @keyframes nesting specifically
       let mut line = 1;          // Track line number (1-based)
       let mut column = 1;        // Track column number (1-based)

       // State transitions with brace depth:
       // '{' → brace_depth += 1
       // '}' → brace_depth -= 1; if depth == 0, back to TopLevel
       //
       // @keyframes special handling:
       // @keyframes name { → at_rule_depth = 1, enter AtRuleBody
       // Inside AtRuleBody: { → at_rule_depth += 1 (for 0%, from {, etc.)
       // Inside AtRuleBody: } → at_rule_depth -= 1; if 0, exit AtRuleBody
       //
       // Key insight: @keyframes contains nested { } for keyframe blocks
       // e.g., @keyframes fade { from { opacity: 0; } to { opacity: 1; } }
       //       ^--- at_rule_depth=1  ^--- depth=2      ^--- depth=2
   }
   ```

   **Variable definition detection rule:**
   - When in `TopLevel` or `PropertyName` state and token starts with `$`:
     - If followed by `identifier:`, emit `VariableDef` token
     - This handles both root-level and nested scope variable definitions
   - Example: `$primary: #ff0000;` → `VariableDef("$primary: #ff0000;")`
   - Example inside rule: `Button { $local: red; color: $local; }` → variable def extracted

   **CRITICAL: VariableDef must include trailing semicolon:**
   - `VariableDef.source` spans `$name: value;` INCLUDING the `;`
   - If not, removing VariableDef leaves a bare `;` → creates empty declaration → breaks parsing
   - Optionally include trailing whitespace/newline for cleaner output
   - Example: `$primary: #ff0000;\n` → entire span is one VariableDef token

   **Line/column tracking:**
   - Tokenizer tracks `line` and `column` as it scans
   - On `\n`: `line += 1; column = 1`
   - Otherwise: `column += 1` per char (not byte)
   - Each token stores `(line, column)` at its start position
   - This enables accurate error reporting in `UndefinedVariableError`

3. **Update `resolve_source_variables()` with lossless reconstruction:**

   ```rust
   pub fn resolve_source_variables(
       source: &str,
       base_vars: &StylesheetVariables,
   ) -> Result<(String, StylesheetVariables), UndefinedVariableError> {
       let tokens = tokenize_css(source);
       let mut vars = base_vars.clone();

       // First pass: extract variable definitions (with chaining support)
       for token in &tokens {
           if let CssTokenKind::VariableDef = token.kind {
               // Parse "$name: value;" from token.source
               // e.g., "$primary: #ff0000;" → name="primary", value="#ff0000"
               // e.g., "$alias: $primary;" → name="alias", value="$primary"
               let (name, raw_value) = parse_variable_def(token.source);

               // CRITICAL: Resolve any $references in the value (enables chaining)
               // e.g., "$alias: $primary;" → resolve $primary → actual color
               let resolved_value = resolve_variables_string_aware(
                   &raw_value, &vars, token.line, token.column
               )?;

               vars.define(name, resolved_value);
           }
       }

       // Second pass: resolve only in PropertyValue tokens, preserve others exactly
       let mut output = String::with_capacity(source.len());
       for token in &tokens {
           match token.kind {
               CssTokenKind::PropertyValue => {
                   // CRITICAL: resolve_variables_string_aware() skips quoted strings
                   // Pass token position for accurate error reporting
                   output.push_str(&resolve_variables_string_aware(
                       token.source, &vars, token.line, token.column
                   )?);
               }
               CssTokenKind::VariableDef => {
                   // Remove variable definitions from output (already extracted)
               }
               _ => {
                   // Preserve exact source slice (whitespace, punctuation, etc.)
                   output.push_str(token.source);
               }
           }
       }

       Ok((output, vars))
   }
   ```

4. **Add string-aware variable resolution** (prevents $ substitution inside quotes):

   ```rust
   /// Resolves $variables in a property value, skipping quoted strings.
   /// token_line/token_column are the token's start position for accurate error reporting.
   fn resolve_variables_string_aware(
       value: &str,
       vars: &StylesheetVariables,
       token_line: usize,
       token_column: usize,
   ) -> Result<String, UndefinedVariableError> {
       let mut result = String::with_capacity(value.len());
       let mut chars = value.char_indices().peekable();
       let mut char_offset = 0; // Track character offset within value (for column calc)

       while let Some((byte_idx, c)) = chars.next() {
           match c {
               '"' | '\'' => {
                   // Skip to end of string literal, preserving exactly
                   let quote = c;
                   result.push(c);
                   char_offset += 1;
                   while let Some((_, sc)) = chars.next() {
                       result.push(sc);
                       char_offset += 1;
                       if sc == quote { break; }
                       if sc == '\\' {
                           // Skip escaped char
                           if let Some((_, ec)) = chars.next() {
                               result.push(ec);
                               char_offset += 1;
                           }
                       }
                   }
               }
               '$' => {
                   // Resolve variable reference
                   // Count the $ character for accurate column reporting
                   let dollar_column = token_column + char_offset;
                   char_offset += 1; // Count the $ itself

                   let var_start = byte_idx + 1;
                   let var_end = value[var_start..].find(|c: char| !c.is_alphanumeric() && c != '_' && c != '-')
                       .map(|e| var_start + e).unwrap_or(value.len());

                   let var_name = &value[var_start..var_end];

                   // Python parity: variable names can start with digits ($5 is valid)
                   // Regex: \$[a-zA-Z0-9_-]+
                   // Only preserve literal $ if there's NO valid identifier chars following
                   if var_name.is_empty() {
                       // Lone $ at end of value or followed by non-identifier char
                       result.push('$');
                       continue;
                   }

                   let resolved = vars.get(var_name)
                       .ok_or_else(|| UndefinedVariableError {
                           name: var_name.to_string(),
                           line: token_line,
                           column: dollar_column, // Column of the $ character
                       })?;
                   result.push_str(&resolved);
                   // Advance chars iterator past the variable name (by byte index, not count)
                   // Use while loop to handle non-ASCII safely
                   while let Some(&(next_idx, _)) = chars.peek() {
                       if next_idx >= var_end { break; }
                       chars.next();
                       char_offset += 1;
                   }
               }
               _ => {
                   result.push(c);
                   char_offset += 1;
               }
           }
       }
       Ok(result)
   }
   ```

### Edge Cases to Handle

| Case                       | Expected Behavior                                              |
| -------------------------- | -------------------------------------------------------------- |
| `/* $5 */`                 | `$5` is comment, NOT resolved (Comment token)                  |
| `@keyframes $name { }`     | `$name` is NOT resolved (AtRule tokens are pass-through)       |
| Nested `{ }` in @keyframes | Track brace depth properly                                     |
| `color: red /* $5 */;`     | Comment split out as separate token before resolver sees value |

**Note:** Some edge cases from Python Textual don't apply:

- `Button.$class` is invalid CSS syntax (`$` not allowed in identifiers)
- `content: "..."` property is not supported by this parser

**Design decision:** Variables in at-rule names (`@keyframes $name`) are NOT resolved. This matches Python Textual behavior where animation names are static identifiers. If needed in future, add AtRule token resolution.

---

## Task 3: Support `auto 87%` Format

### Problem

Python Textual's `auto` triggers runtime contrast calculation against the actual background.

- `ColorSystem.generate_variables()` outputs: `$text: "auto 87%"`, `$text-muted: "auto 60%"`
- These currently fail because `auto` isn't resolved to an actual color

### Python Behavior

1. Parse `auto 87%` → sets `auto_color = true`, `color.alpha = 0.87`
2. At render time: `background.get_contrast_text(0.87)` → returns white or black with 87% opacity

### What Already Works

- `RgbaColor` struct has `pub auto: bool` field (line 5200)
- `RgbaColor::parse()` handles "auto 87%" (lines 5234-5236 → `parse_auto()` at 5266-5298)
- `RgbaColor::contrast_text()` exists at line 5709 (computes white/black based on brightness)

### What Needs to Be Added

**`textual-rs/src/style.rs`:**

| Component                    | Line  | Change                                                |
| ---------------------------- | ----- | ----------------------------------------------------- |
| `ComputedStyle`              | 4355+ | Add `auto_color: bool`, `auto_background: bool` flags |
| `ComputedStyle::apply()`     | 4462+ | Set auto flags when applying auto colors              |
| New: `resolve_auto_colors()` | -     | Runtime contrast calculation                          |

**Rendering integration** (wherever styles are applied to widgets):

| Location           | Change                                              |
| ------------------ | --------------------------------------------------- |
| Widget render code | Call `resolve_auto_colors()` with actual background |

### Implementation Steps

1. **Add auto flags to ComputedStyle:**

   ```rust
   pub struct ComputedStyle {
       pub color: Option<RgbaColor>,
       pub auto_color: bool,  // NEW
       pub background: Option<RgbaColor>,
       pub auto_background: bool,  // NEW (rarely used)
       // ...
   }
   ```

2. **Update ComputedStyle::apply():**

   ```rust
   Declaration::Color(c) => {
       self.color = Some(c.clone());
       self.auto_color = c.auto;
   }
   ```

3. **Add resolve_auto_colors():**

   ```rust
   impl ComputedStyle {
       pub fn resolve_auto_colors(&self, background: &RgbaColor) -> ComputedStyle {
           let mut resolved = self.clone();
           if self.auto_color {
               if let Some(ref color) = self.color {
                   resolved.color = Some(background.contrast_text().with_alpha(color.a));
               }
           }
           resolved
       }
   }
   ```

4. **Integrate at render time** (in widget code):
   ```rust
   let effective_style = computed_style.resolve_auto_colors(&actual_background);
   ```

### Note on $text Variables

Once Task 1 is complete, `$text: "auto 87%"` will:

1. Parse via `RgbaColor::parse()` → `RgbaColor { auto: true, a: 0.87, ... }`
2. Store in ComputedStyle with `auto_color = true`
3. Resolve at render time against actual background

---

## Execution Order

| Order | Task                            | Effort | Dependencies |
| ----- | ------------------------------- | ------ | ------------ |
| 1     | Replace Color with RgbaColor    | Medium | None         |
| 2     | Auto color support              | Medium | Task 1       |
| 3     | Token-level variable resolution | High   | None         |

**Rationale:**

- Task 1 enables all color formats and is foundation for Task 3
- Task 3 enables $text/$text-muted, completing theme variable support
- Task 2 is correctness improvement, can run in parallel or after

---

## Testing Strategy

### Task 1 Tests (`textual-rs/tests/color_formats.rs`)

```rust
#[test]
fn test_parse_short_hex_rgb() {
    let css = "Button { color: #f00; }";  // #RGB → #ff0000
    let stylesheet = StyleSheet::parse(css).unwrap();
}

#[test]
fn test_parse_short_hex_rgba() {
    let css = "Button { color: #f008; }";  // #RGBA → #ff000088 (53% alpha)
    let stylesheet = StyleSheet::parse(css).unwrap();
    // Verify alpha ≈ 0.53 (0x88 / 255)
}

#[test]
fn test_parse_rgba_hex_8digit() {
    let css = "Button { color: #ff000080; }";  // #RRGGBBAA
    let stylesheet = StyleSheet::parse(css).unwrap();
    // Verify alpha = 0.5 (0x80 / 255)
}

#[test]
fn test_parse_rgb_function() {
    let css = "Button { background: rgb(255, 128, 0); }";
    let stylesheet = StyleSheet::parse(css).unwrap();
}

#[test]
fn test_parse_rgba_function() {
    let css = "Button { background: rgba(255, 128, 0, 0.5); }";
    let stylesheet = StyleSheet::parse(css).unwrap();
}

#[test]
fn test_parse_hsl_function() {
    let css = "Button { color: hsl(120, 100%, 50%); }";
    let stylesheet = StyleSheet::parse(css).unwrap();
}

#[test]
fn test_parse_hsla_function() {
    let css = "Button { color: hsla(120, 100%, 50%, 0.5); }";
    let stylesheet = StyleSheet::parse(css).unwrap();
}
```

### Task 2 Tests (`textual-rs/tests/style_variables.rs` - extend existing)

```rust
#[test]
fn test_variable_not_resolved_in_comment() {
    // $5 in comment should NOT trigger undefined variable error
    let css = "/* Price: $5 */ Button { color: red; }";
    let result = StyleSheet::parse(css);
    assert!(result.is_ok()); // $5 in comment ignored
}

#[test]
fn test_variable_resolved_in_unquoted_value() {
    let css = "$primary: #ff0000; Button { color: $primary; }";
    let stylesheet = StyleSheet::parse(css).unwrap();
    // $primary in unquoted value SHOULD be resolved
}

#[test]
fn test_dollar_in_comment_not_error() {
    // Various $ patterns in comments should be ignored
    let css = r#"
        /* This costs $5.00 */
        /* $undefined_var */
        Button { background: #ff0000; }
    "#;
    let result = StyleSheet::parse(css);
    assert!(result.is_ok()); // Comments ignored entirely
}
```

**Note on removed tests:**

- `Button.$primary` - Invalid CSS selector syntax ($ not allowed in identifiers). Removed.
- `content: "$primary"` - `content` property not supported by parser. Removed.

The tokenizer should handle these edge cases, but we can't write tests for invalid syntax.

### Task 3 Tests (`textual-rs/tests/auto_colors.rs`)

```rust
#[test]
fn test_parse_auto_color() {
    let css = "Label { color: auto 87%; }";
    let stylesheet = StyleSheet::parse(css).unwrap();
    // Extract color from rule and verify auto flag
}

#[test]
fn test_auto_contrast_light_background() {
    let bg = RgbaColor::rgb(255, 255, 255); // white
    let text = RgbaColor::parse("auto 87%").unwrap();
    let resolved = bg.contrast_text().with_alpha(text.a);
    // Should be near-black (dark text on light bg)
    assert!(resolved.r < 50 && resolved.g < 50 && resolved.b < 50);
}

#[test]
fn test_auto_contrast_dark_background() {
    let bg = RgbaColor::rgb(0, 0, 0); // black
    let text = RgbaColor::parse("auto 87%").unwrap();
    let resolved = bg.contrast_text().with_alpha(text.a);
    // Should be near-white (light text on dark bg)
    assert!(resolved.r > 200 && resolved.g > 200 && resolved.b > 200);
}
```

---

## Summary

All three tasks will be implemented to achieve full Python Textual parity:

1. **Task 1: Color Migration** - Replace `Color` enum with `RgbaColor`, enabling all color formats
2. **Task 2: Token-Level Variables** - Context-aware tokenizer to resolve `$` only in values
3. **Task 3: Auto Colors** - Runtime contrast calculation for `auto 87%` format

**Total estimated changes:** ~800-1000 lines in `textual-rs/src/style.rs`
