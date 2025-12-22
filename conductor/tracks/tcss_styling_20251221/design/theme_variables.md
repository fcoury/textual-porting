# Design: Theme Variable Resolution

## Overview

This document defines the theming system architecture, including theme definitions, CSS variable generation, and variable resolution during stylesheet parsing.

## Theme Structure

### Theme Definition

```rust
/// A complete theme definition
#[derive(Debug, Clone)]
pub struct Theme {
    /// Theme identifier
    pub name: String,

    // Core semantic colors (required)
    pub primary: String,

    // Optional semantic colors
    pub secondary: Option<String>,
    pub warning: Option<String>,
    pub error: Option<String>,
    pub success: Option<String>,
    pub accent: Option<String>,

    // Base colors
    pub foreground: Option<String>,
    pub background: Option<String>,
    pub surface: Option<String>,    // Elevated surface
    pub panel: Option<String>,      // Panel backgrounds
    pub boost: Option<String>,      // Highlight color

    // Theme settings
    pub dark: bool,                 // Dark or light theme
    pub luminosity_spread: f32,     // Color variation range (default: 0.15)
    pub text_alpha: f32,            // Default text opacity (default: 0.95)

    // Custom CSS variables
    pub variables: HashMap<String, String>,
}

impl Default for Theme {
    fn default() -> Self {
        Theme {
            name: "default".into(),
            primary: "#0178D4".into(),
            secondary: None,
            warning: None,
            error: None,
            success: None,
            accent: None,
            foreground: None,      // Defaults to #e0e0e0 (dark) or #1e1e1e (light) in ColorSystem
            background: None,      // Defaults to #121212 (dark) or #ffffff (light) in ColorSystem
            surface: None,         // Defaults to #1e1e1e (dark) or #f5f5f5 (light) in ColorSystem
            panel: None,
            boost: None,
            dark: true,
            luminosity_spread: 0.15,
            text_alpha: 0.95,
            variables: HashMap::new(),
        }
    }
}
```

### Built-in Themes

```rust
use once_cell::sync::Lazy;

pub static BUILTIN_THEMES: Lazy<HashMap<&'static str, Theme>> = Lazy::new(|| {
    let mut m = HashMap::new();

    m.insert("textual-dark", Theme {
        name: "textual-dark".into(),
        primary: "#0178D4".into(),
        secondary: Some("#004578".into()),
        accent: Some("#ffa62b".into()),
        warning: Some("#ffa62b".into()),
        error: Some("#ba3c5b".into()),
        success: Some("#4EBF71".into()),
        foreground: Some("#e0e0e0".into()),
        dark: true,
        ..Default::default()
    });

    m.insert("textual-light", Theme {
        name: "textual-light".into(),
        primary: "#0178D4".into(),
        secondary: Some("#004578".into()),
        accent: Some("#ffa62b".into()),
        warning: Some("#ffa62b".into()),
        error: Some("#ba3c5b".into()),
        success: Some("#4EBF71".into()),
        foreground: Some("#1e1e1e".into()),
        background: Some("#f5f5f5".into()),
        dark: false,
        ..Default::default()
    });

    m.insert("nord", Theme {
        name: "nord".into(),
        primary: "#88c0d0".into(),
        secondary: Some("#81a1c1".into()),
        accent: Some("#ebcb8b".into()),
        warning: Some("#ebcb8b".into()),
        error: Some("#bf616a".into()),
        success: Some("#a3be8c".into()),
        foreground: Some("#eceff4".into()),
        background: Some("#2e3440".into()),
        surface: Some("#3b4252".into()),
        dark: true,
        ..Default::default()
    });

    m.insert("dracula", Theme {
        name: "dracula".into(),
        primary: "#bd93f9".into(),
        secondary: Some("#6272a4".into()),
        accent: Some("#ffb86c".into()),
        warning: Some("#ffb86c".into()),
        error: Some("#ff5555".into()),
        success: Some("#50fa7b".into()),
        foreground: Some("#f8f8f2".into()),
        background: Some("#282a36".into()),
        surface: Some("#44475a".into()),
        dark: true,
        ..Default::default()
    });

    // Additional themes: gruvbox, catppuccin-mocha, tokyo-night, monokai,
    // flexoki, solarized-light, solarized-dark, rose-pine, etc.

    m
});
```

## Color System

Converts theme to CSS variables with color shades:

```rust
/// Generates CSS variables from theme colors
pub struct ColorSystem {
    pub primary: Color,
    pub secondary: Option<Color>,
    pub warning: Option<Color>,
    pub error: Option<Color>,
    pub success: Option<Color>,
    pub accent: Option<Color>,

    pub foreground: Color,
    pub background: Color,
    pub surface: Option<Color>,
    pub panel: Option<Color>,
    pub boost: Option<Color>,

    pub dark: bool,
    pub luminosity_spread: f32,
    pub text_alpha: f32,

    pub custom_variables: HashMap<String, String>,
}

impl ColorSystem {
    /// Create from theme
    /// Defaults match Python's DEFAULT_COLORS in theme.py
    pub fn from_theme(theme: &Theme) -> Result<Self, ColorParseError> {
        Ok(ColorSystem {
            primary: Color::parse(&theme.primary)?,
            secondary: theme.secondary.as_ref().map(|s| Color::parse(s)).transpose()?,
            warning: theme.warning.as_ref().map(|s| Color::parse(s)).transpose()?,
            error: theme.error.as_ref().map(|s| Color::parse(s)).transpose()?,
            success: theme.success.as_ref().map(|s| Color::parse(s)).transpose()?,
            accent: theme.accent.as_ref().map(|s| Color::parse(s)).transpose()?,
            foreground: theme.foreground.as_ref()
                .map(|s| Color::parse(s))
                .transpose()?
                .unwrap_or(if theme.dark {
                    Color::parse("#e0e0e0")?   // DEFAULT_COLORS["dark"]["foreground"]
                } else {
                    Color::parse("#1e1e1e")?   // DEFAULT_COLORS["light"]["foreground"]
                }),
            background: theme.background.as_ref()
                .map(|s| Color::parse(s))
                .transpose()?
                .unwrap_or(if theme.dark {
                    Color::parse("#121212")?   // DEFAULT_COLORS["dark"]["background"]
                } else {
                    Color::parse("#ffffff")?   // DEFAULT_COLORS["light"]["background"]
                }),
            surface: theme.surface.as_ref().map(|s| Color::parse(s)).transpose()?,
            panel: theme.panel.as_ref().map(|s| Color::parse(s)).transpose()?,
            boost: theme.boost.as_ref().map(|s| Color::parse(s)).transpose()?,
            dark: theme.dark,
            luminosity_spread: theme.luminosity_spread,
            text_alpha: theme.text_alpha,
            custom_variables: theme.variables.clone(),
        })
    }

    /// Generate all CSS variables (matches Python ColorSystem.generate())
    pub fn generate_variables(&self) -> HashMap<String, String> {
        let mut vars = HashMap::new();

        // Primary color shades (with background and muted variants)
        self.add_color_shades(&mut vars, "primary", &self.primary);

        // Optional semantic colors with shades
        if let Some(ref c) = self.secondary {
            self.add_color_shades(&mut vars, "secondary", c);
        }
        if let Some(ref c) = self.warning {
            self.add_color_shades(&mut vars, "warning", c);
        }
        if let Some(ref c) = self.error {
            self.add_color_shades(&mut vars, "error", c);
        }
        if let Some(ref c) = self.success {
            self.add_color_shades(&mut vars, "success", c);
        }
        if let Some(ref c) = self.accent {
            self.add_color_shades(&mut vars, "accent", c);
        }

        // Base colors using Textual constants
        vars.insert("foreground".into(), self.foreground.to_css());
        vars.insert("background".into(), self.background.to_css());

        // Surface uses Textual's default constants if not specified
        let surface = self.surface.clone().unwrap_or_else(|| {
            if self.dark {
                DEFAULT_DARK_SURFACE.clone()
            } else {
                DEFAULT_LIGHT_SURFACE.clone()
            }
        });
        vars.insert("surface".into(), surface.to_css());

        // Panel defaults to surface
        let panel = self.panel.clone().unwrap_or(surface.clone());
        vars.insert("panel".into(), panel.to_css());

        if let Some(ref c) = self.boost {
            vars.insert("boost".into(), c.to_css());
        }

        // Text colors with alpha
        vars.insert("text".into(), self.foreground.with_alpha(self.text_alpha).to_css());
        vars.insert("text-muted".into(), self.foreground.with_alpha(0.6).to_css());
        vars.insert("text-disabled".into(), self.foreground.with_alpha(0.4).to_css());

        // Contrast text for colored backgrounds (uses Textual's get_contrast_text)
        // Python uses a tinted variant based on background, not pure black/white
        let text_on_primary = self.contrast_text(&self.primary);
        vars.insert("text-primary".into(), text_on_primary.to_css());

        if let Some(ref secondary) = self.secondary {
            let text_on_secondary = self.contrast_text(secondary);
            vars.insert("text-secondary".into(), text_on_secondary.to_css());
        }

        // Contrast text for warning/error/success/accent backgrounds
        if let Some(ref warning) = self.warning {
            let text_on_warning = self.contrast_text(warning);
            vars.insert("text-warning".into(), text_on_warning.to_css());
        }
        if let Some(ref error) = self.error {
            let text_on_error = self.contrast_text(error);
            vars.insert("text-error".into(), text_on_error.to_css());
        }
        if let Some(ref success) = self.success {
            let text_on_success = self.contrast_text(success);
            vars.insert("text-success".into(), text_on_success.to_css());
        }
        if let Some(ref accent) = self.accent {
            let text_on_accent = self.contrast_text(accent);
            vars.insert("text-accent".into(), text_on_accent.to_css());
        }

        // Add custom variables
        for (name, value) in &self.custom_variables {
            vars.insert(name.clone(), value.clone());
        }

        vars
    }

    /// Add color with lighten/darken shades, background, and muted variants
    fn add_color_shades(&self, vars: &mut HashMap<String, String>, name: &str, color: &Color) {
        // Base color
        vars.insert(name.into(), color.to_css());

        // Lighten shades (1-3)
        for i in 1..=3 {
            let factor = self.luminosity_spread * i as f32;
            let shade = color.lighten(factor);
            vars.insert(format!("{}-lighten-{}", name, i), shade.to_css());
        }

        // Darken shades (1-3)
        for i in 1..=3 {
            let factor = self.luminosity_spread * i as f32;
            let shade = color.darken(factor);
            vars.insert(format!("{}-darken-{}", name, i), shade.to_css());
        }

        // Background variant (blended with theme background)
        let bg = color.blend(&self.background, 0.85);
        vars.insert(format!("{}-background", name), bg.to_css());

        // Muted variant (reduced saturation/opacity for subtle use)
        let muted = color.with_alpha(0.3);
        vars.insert(format!("{}-muted", name), muted.to_css());
    }

    /// Get contrasting text color for a background (matches Python's get_contrast_text)
    /// Uses a tinted variant based on the color being contrasted against,
    /// not pure black/white.
    fn contrast_text(&self, background: &Color) -> Color {
        // Calculate relative luminance
        let luminance = background.luminance();

        // Use WCAG contrast threshold (0.179 is the threshold for 4.5:1 contrast)
        if luminance > 0.179 {
            // Light background: use dark text tinted toward the background color
            // This creates a more cohesive look than pure black
            let base = Color::parse("#000000").unwrap();
            base.blend(background, 0.15).with_alpha(self.text_alpha)
        } else {
            // Dark background: use light text tinted toward the background color
            // This creates a more cohesive look than pure white
            let base = Color::parse("#ffffff").unwrap();
            base.blend(background, 0.15).with_alpha(self.text_alpha)
        }
    }
}

/// Default dark theme background (matches Python DEFAULT_COLORS["dark"]["background"])
pub static DEFAULT_DARK_BACKGROUND: Color = Color { r: 18, g: 18, b: 18, a: 1.0 };  // #121212

/// Default dark theme surface (matches Python DEFAULT_COLORS["dark"]["surface"])
pub static DEFAULT_DARK_SURFACE: Color = Color { r: 30, g: 30, b: 30, a: 1.0 };  // #1e1e1e

/// Default light theme background (matches Python DEFAULT_COLORS["light"]["background"])
pub static DEFAULT_LIGHT_BACKGROUND: Color = Color { r: 255, g: 255, b: 255, a: 1.0 };  // #ffffff

/// Default light theme surface (matches Python DEFAULT_COLORS["light"]["surface"])
pub static DEFAULT_LIGHT_SURFACE: Color = Color { r: 245, g: 245, b: 245, a: 1.0 };  // #f5f5f5

impl Color {
    /// Calculate relative luminance (for WCAG contrast calculations)
    pub fn luminance(&self) -> f32 {
        // Convert to linear RGB
        fn to_linear(c: u8) -> f32 {
            let c = c as f32 / 255.0;
            if c <= 0.03928 {
                c / 12.92
            } else {
                ((c + 0.055) / 1.055).powf(2.4)
            }
        }
        let r = to_linear(self.r);
        let g = to_linear(self.g);
        let b = to_linear(self.b);
        0.2126 * r + 0.7152 * g + 0.0722 * b
    }

    /// Blend this color with another by factor (0.0 = self, 1.0 = other)
    pub fn blend(&self, other: &Color, factor: f32) -> Color {
        Color {
            r: lerp_u8(self.r, other.r, factor),
            g: lerp_u8(self.g, other.g, factor),
            b: lerp_u8(self.b, other.b, factor),
            a: lerp_f32(self.a, other.a, factor),
        }
    }
}

fn lerp_u8(a: u8, b: u8, t: f32) -> u8 {
    (a as f32 + (b as f32 - a as f32) * t).round() as u8
}

fn lerp_f32(a: f32, b: f32, t: f32) -> f32 {
    a + (b - a) * t
}

impl Color {
    /// Lighten the color by factor (0.0 to 1.0)
    pub fn lighten(&self, factor: f32) -> Color {
        let (h, s, l, a) = self.to_hsla();
        let new_l = (l + factor).clamp(0.0, 1.0);
        Color::from_hsla(h, s, new_l, a)
    }

    /// Darken the color by factor (0.0 to 1.0)
    pub fn darken(&self, factor: f32) -> Color {
        let (h, s, l, a) = self.to_hsla();
        let new_l = (l - factor).clamp(0.0, 1.0);
        Color::from_hsla(h, s, new_l, a)
    }

    /// Return color with new alpha
    pub fn with_alpha(&self, alpha: f32) -> Color {
        Color {
            r: self.r,
            g: self.g,
            b: self.b,
            a: alpha,
        }
    }
}
```

## Variable Resolution

TCSS uses a simple `$variable` syntax for CSS variables. Variables are defined and resolved
at the stylesheet level only (no per-node scoping or CSS `var()` function).

### Variable Definition Syntax

Variables can be defined in CSS using the `$name: value;` syntax:

```css
$my-color: #ff6b6b;
$spacing: 2;

Button {
    background: $my-color;
    padding: $spacing;
}
```

### Variable Storage

```rust
/// Stylesheet-level variable storage
///
/// Variables are resolved at parse time, not runtime.
/// There is no per-node scoping - all variables are global to the stylesheet.
pub struct StylesheetVariables {
    /// Theme-generated variables ($primary, $secondary, etc.)
    theme_variables: HashMap<String, String>,
    /// User-defined variables from $name: value declarations
    user_variables: HashMap<String, String>,
}

impl StylesheetVariables {
    pub fn new() -> Self {
        StylesheetVariables {
            theme_variables: HashMap::new(),
            user_variables: HashMap::new(),
        }
    }

    /// Create with theme variables
    pub fn from_theme(theme: &Theme) -> Result<Self, ColorParseError> {
        let color_system = ColorSystem::from_theme(theme)?;
        Ok(StylesheetVariables {
            theme_variables: color_system.generate_variables(),
            user_variables: HashMap::new(),
        })
    }

    /// Define a user variable
    pub fn define(&mut self, name: String, value: String) {
        self.user_variables.insert(name, value);
    }

    /// Look up a variable by name
    /// User variables take precedence over theme variables
    pub fn get(&self, name: &str) -> Option<&String> {
        self.user_variables.get(name)
            .or_else(|| self.theme_variables.get(name))
    }

    /// Resolve a $variable reference
    pub fn resolve(&self, name: &str) -> Option<String> {
        self.get(name).cloned()
    }
}
```

### Value Resolution

```rust
/// Error for undefined variables
#[derive(Debug, Clone)]
pub struct UndefinedVariableError {
    pub name: String,
    pub line: usize,
    pub column: usize,
}

/// Resolve all $variable references in a CSS value string
/// Returns error if any variable is undefined (matches Python behavior)
pub fn resolve_variables(
    value: &str,
    vars: &StylesheetVariables,
) -> Result<String, UndefinedVariableError> {
    let mut result = String::new();
    let mut chars = value.chars().peekable();
    let mut column = 0;

    while let Some(c) = chars.next() {
        column += 1;
        if c == '$' {
            let var_start = column;
            // Parse variable name (alphanumeric, -, _)
            let mut name = String::new();
            while let Some(&c) = chars.peek() {
                if c.is_alphanumeric() || c == '-' || c == '_' {
                    name.push(chars.next().unwrap());
                    column += 1;
                } else {
                    break;
                }
            }

            if !name.is_empty() {
                if let Some(resolved) = vars.resolve(&name) {
                    result.push_str(&resolved);
                    continue;
                } else {
                    // Python raises parse error for undefined variables
                    return Err(UndefinedVariableError {
                        name,
                        line: 0,  // Would be set by caller
                        column: var_start,
                    });
                }
            }

            // Lone $ with no valid name - keep as-is
            result.push('$');
        } else {
            result.push(c);
        }
    }

    Ok(result)
}
```

### Variable Definition Parsing

```rust
/// Parse variable definitions from CSS
/// Returns (variable_name, value) for lines like "$name: value;"
pub fn parse_variable_definition(line: &str) -> Option<(String, String)> {
    let line = line.trim();

    if !line.starts_with('$') {
        return None;
    }

    let colon_pos = line.find(':')?;
    let name = line[1..colon_pos].trim().to_string();

    let mut value = line[colon_pos + 1..].trim();
    // Remove trailing semicolon if present
    if value.ends_with(';') {
        value = &value[..value.len() - 1];
    }

    Some((name, value.trim().to_string()))
}
```

## Theme Registry

```rust
/// Manages available themes
pub struct ThemeRegistry {
    themes: HashMap<String, Theme>,
    current: String,
}

impl ThemeRegistry {
    pub fn new() -> Self {
        let mut registry = ThemeRegistry {
            themes: HashMap::new(),
            current: "textual-dark".into(),
        };

        // Register built-in themes
        for (name, theme) in BUILTIN_THEMES.iter() {
            registry.themes.insert(name.to_string(), theme.clone());
        }

        registry
    }

    /// Register a custom theme
    pub fn register(&mut self, theme: Theme) {
        self.themes.insert(theme.name.clone(), theme);
    }

    /// Get a theme by name
    pub fn get(&self, name: &str) -> Option<&Theme> {
        self.themes.get(name)
    }

    /// Get the current theme
    pub fn current(&self) -> &Theme {
        self.themes.get(&self.current).expect("Current theme must exist")
    }

    /// Set the current theme
    pub fn set_current(&mut self, name: &str) -> Result<(), ThemeError> {
        if self.themes.contains_key(name) {
            self.current = name.to_string();
            Ok(())
        } else {
            Err(ThemeError::NotFound(name.into()))
        }
    }

    /// List all available theme names
    pub fn available(&self) -> Vec<&str> {
        self.themes.keys().map(|s| s.as_str()).collect()
    }

    /// Check if running in dark mode
    pub fn is_dark(&self) -> bool {
        self.current().dark
    }
}

#[derive(Debug, Clone)]
pub enum ThemeError {
    NotFound(String),
    InvalidColor(String),
}
```

## Integration with Stylesheet

```rust
impl Stylesheet {
    /// Set theme and regenerate variables
    pub fn set_theme(&mut self, theme: &Theme) -> Result<(), ThemeError> {
        let color_system = ColorSystem::from_theme(theme)
            .map_err(|e| ThemeError::InvalidColor(e.to_string()))?;

        let theme_vars = color_system.generate_variables();
        self.variables = StylesheetVariables {
            theme_variables: theme_vars,
            user_variables: self.variables.user_variables.clone(),
        };
        self.reparse()?;

        Ok(())
    }

    /// Reparse all rules with current variables
    pub fn reparse(&mut self) -> Result<(), StylesheetError> {
        for (location, source) in &self.source {
            // First pass: extract $name: value definitions
            for line in source.content.lines() {
                if let Some((name, value)) = parse_variable_definition(line) {
                    self.variables.define(name, value);
                }
            }

            // Second pass: resolve $var references and parse rules
            // Note: undefined variables cause parse error (matches Python)
            let resolved_css = resolve_variables(&source.content, &self.variables)
                .map_err(|e| StylesheetError::UndefinedVariable {
                    name: e.name,
                    location: location.clone(),
                })?;
            let rules = self.parse_css(&resolved_css, source.scope.clone())?;
            // Update rules...
        }
        Ok(())
    }
}

#[derive(Debug)]
pub enum StylesheetError {
    Parse(ParseError),
    UndefinedVariable { name: String, location: CssLocation },
}
```

## Theme Provider (Command Palette Integration)

```rust
/// Provides theme commands for command palette
pub struct ThemeProvider {
    registry: ThemeRegistry,
}

impl ThemeProvider {
    pub fn new(registry: ThemeRegistry) -> Self {
        ThemeProvider { registry }
    }

    /// Get available theme commands
    /// Note: "textual-ansi" is excluded from command palette
    pub fn commands(&self) -> Vec<ThemeCommand> {
        self.registry.available()
            .into_iter()
            .filter(|name| *name != "textual-ansi")
            .map(|name| ThemeCommand {
                name: name.to_string(),
                display_name: self.format_display_name(name),
            })
            .collect()
    }

    /// Search themes by query
    pub fn search(&self, query: &str) -> Vec<ThemeCommand> {
        let query_lower = query.to_lowercase();
        self.commands()
            .into_iter()
            .filter(|cmd| cmd.name.to_lowercase().contains(&query_lower))
            .collect()
    }

    fn format_display_name(&self, name: &str) -> String {
        // Convert "textual-dark" to "Textual Dark"
        name.split('-')
            .map(|word| {
                let mut chars = word.chars();
                match chars.next() {
                    Some(c) => c.to_uppercase().chain(chars).collect(),
                    None => String::new(),
                }
            })
            .collect::<Vec<_>>()
            .join(" ")
    }
}

pub struct ThemeCommand {
    pub name: String,
    pub display_name: String,
}
```

## Usage Example

```rust
// Create theme registry
let mut registry = ThemeRegistry::new();

// Register custom theme
registry.register(Theme {
    name: "my-theme".into(),
    primary: "#ff6b6b".into(),
    secondary: Some("#4ecdc4".into()),
    dark: true,
    ..Default::default()
});

// Apply theme to stylesheet
let theme = registry.get("nord").unwrap();
stylesheet.set_theme(theme)?;

// In CSS, use variables:
// Button {
//     background: $primary;
//     color: $foreground;
// }
// Button:hover {
//     background: $primary-lighten-1;
// }
```
