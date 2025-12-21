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
            foreground: None,
            background: None,
            surface: None,
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
                    Color::parse("#e0e0e0")?
                } else {
                    Color::parse("#1e1e1e")?
                }),
            background: theme.background.as_ref()
                .map(|s| Color::parse(s))
                .transpose()?
                .unwrap_or(if theme.dark {
                    Color::parse("#1e1e1e")?
                } else {
                    Color::parse("#f5f5f5")?
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

    /// Generate all CSS variables
    pub fn generate_variables(&self) -> HashMap<String, String> {
        let mut vars = HashMap::new();

        // Primary color shades
        self.add_color_shades(&mut vars, "primary", &self.primary);

        // Optional semantic colors
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

        // Base colors
        vars.insert("foreground".into(), self.foreground.to_css());
        vars.insert("background".into(), self.background.to_css());

        if let Some(ref c) = self.surface {
            vars.insert("surface".into(), c.to_css());
        }
        if let Some(ref c) = self.panel {
            vars.insert("panel".into(), c.to_css());
        }
        if let Some(ref c) = self.boost {
            vars.insert("boost".into(), c.to_css());
        }

        // Text color with alpha
        vars.insert("text".into(), self.foreground.with_alpha(self.text_alpha).to_css());
        vars.insert("text-muted".into(), self.foreground.with_alpha(0.6).to_css());
        vars.insert("text-disabled".into(), self.foreground.with_alpha(0.4).to_css());

        // Add custom variables
        for (name, value) in &self.custom_variables {
            vars.insert(name.clone(), value.clone());
        }

        vars
    }

    /// Add color with lighten/darken shades
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

        // Background variant (for use as background color)
        let bg = if self.dark {
            color.darken(0.5)
        } else {
            color.lighten(0.5)
        };
        vars.insert(format!("{}-background", name), bg.to_css());

        // Muted variant
        let muted = color.with_alpha(0.5);
        vars.insert(format!("{}-muted", name), muted.to_css());
    }
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

### Variable Reference Types

```rust
/// A reference to a CSS variable
#[derive(Debug, Clone, PartialEq)]
pub enum VariableRef {
    /// Simple variable reference: $primary
    Simple(String),
    /// Var function with optional fallback: var(--primary, #fff)
    Var {
        name: String,
        fallback: Option<Box<CssValue>>,
    },
}

impl VariableRef {
    /// Parse variable reference from string
    pub fn parse(s: &str) -> Option<Self> {
        let s = s.trim();

        // Simple $variable syntax
        if s.starts_with('$') {
            let name = s[1..].to_string();
            if !name.is_empty() && name.chars().all(|c| c.is_alphanumeric() || c == '-' || c == '_') {
                return Some(VariableRef::Simple(name));
            }
        }

        // var(--name) or var(--name, fallback) syntax
        if s.starts_with("var(") && s.ends_with(')') {
            let inner = &s[4..s.len()-1].trim();
            if let Some(comma_pos) = inner.find(',') {
                let name = inner[..comma_pos].trim();
                let fallback = inner[comma_pos+1..].trim();
                if name.starts_with("--") {
                    return Some(VariableRef::Var {
                        name: name[2..].to_string(),
                        fallback: Some(Box::new(CssValue::Raw(fallback.to_string()))),
                    });
                }
            } else if inner.starts_with("--") {
                return Some(VariableRef::Var {
                    name: inner[2..].to_string(),
                    fallback: None,
                });
            }
        }

        None
    }
}
```

### Variable Context

```rust
/// Context for resolving CSS variables
pub struct VariableContext {
    /// Variables from theme
    theme_variables: HashMap<String, String>,
    /// Variables from :root or global scope
    root_variables: HashMap<String, String>,
    /// Variables from ancestor scopes (for inheritance)
    inherited_variables: HashMap<String, String>,
    /// Local variables (inline styles)
    local_variables: HashMap<String, String>,
}

impl VariableContext {
    pub fn new() -> Self {
        VariableContext {
            theme_variables: HashMap::new(),
            root_variables: HashMap::new(),
            inherited_variables: HashMap::new(),
            local_variables: HashMap::new(),
        }
    }

    /// Create context from theme
    pub fn from_theme(theme: &Theme) -> Result<Self, ColorParseError> {
        let color_system = ColorSystem::from_theme(theme)?;
        Ok(VariableContext {
            theme_variables: color_system.generate_variables(),
            root_variables: HashMap::new(),
            inherited_variables: HashMap::new(),
            local_variables: HashMap::new(),
        })
    }

    /// Resolve a variable reference
    pub fn resolve(&self, var_ref: &VariableRef) -> Option<String> {
        match var_ref {
            VariableRef::Simple(name) | VariableRef::Var { name, fallback: None } => {
                self.lookup(name)
            }
            VariableRef::Var { name, fallback: Some(fb) } => {
                self.lookup(name).or_else(|| {
                    // Recursively resolve fallback if it contains variables
                    Some(fb.resolve(self))
                })
            }
        }
    }

    /// Look up variable by name (checks all scopes)
    fn lookup(&self, name: &str) -> Option<String> {
        // Priority: local > inherited > root > theme
        self.local_variables.get(name)
            .or_else(|| self.inherited_variables.get(name))
            .or_else(|| self.root_variables.get(name))
            .or_else(|| self.theme_variables.get(name))
            .cloned()
    }

    /// Set a local variable
    pub fn set_local(&mut self, name: String, value: String) {
        self.local_variables.insert(name, value);
    }

    /// Create child context with inherited variables
    pub fn child(&self) -> VariableContext {
        let mut inherited = self.inherited_variables.clone();
        inherited.extend(self.local_variables.clone());

        VariableContext {
            theme_variables: self.theme_variables.clone(),
            root_variables: self.root_variables.clone(),
            inherited_variables: inherited,
            local_variables: HashMap::new(),
        }
    }
}
```

### Value Resolution

```rust
impl CssValue {
    /// Resolve all variables in this value
    pub fn resolve(&self, ctx: &VariableContext) -> String {
        match self {
            CssValue::Raw(s) => {
                // Check if entire value is a variable
                if let Some(var_ref) = VariableRef::parse(s) {
                    if let Some(resolved) = ctx.resolve(&var_ref) {
                        return resolved;
                    }
                }

                // Check for embedded variables
                self.resolve_embedded_vars(s, ctx)
            }
            CssValue::Resolved(s) => s.clone(),
        }
    }

    /// Resolve variables embedded in a string
    fn resolve_embedded_vars(&self, s: &str, ctx: &VariableContext) -> String {
        let mut result = String::new();
        let mut chars = s.chars().peekable();

        while let Some(c) = chars.next() {
            if c == '$' {
                // Parse variable name
                let mut name = String::new();
                while let Some(&c) = chars.peek() {
                    if c.is_alphanumeric() || c == '-' || c == '_' {
                        name.push(chars.next().unwrap());
                    } else {
                        break;
                    }
                }

                if !name.is_empty() {
                    let var_ref = VariableRef::Simple(name);
                    if let Some(resolved) = ctx.resolve(&var_ref) {
                        result.push_str(&resolved);
                        continue;
                    }
                }

                // Variable not found, keep original
                result.push('$');
                result.push_str(&name);
            } else {
                result.push(c);
            }
        }

        result
    }
}

#[derive(Debug, Clone)]
pub enum CssValue {
    /// Raw value that may contain variables
    Raw(String),
    /// Fully resolved value
    Resolved(String),
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

        let variables = color_system.generate_variables();
        self.set_variables(variables);
        self.reparse()?;

        Ok(())
    }

    /// Set CSS variables
    pub fn set_variables(&mut self, variables: HashMap<String, String>) {
        self.variable_context = VariableContext {
            theme_variables: variables,
            ..VariableContext::new()
        };
    }

    /// Reparse all rules with current variables
    pub fn reparse(&mut self) -> Result<(), ParseError> {
        for (location, source) in &self.source {
            let resolved_css = self.resolve_variables(&source.content);
            let rules = self.parse_css(&resolved_css, source.scope.clone())?;
            // Update rules...
        }
        Ok(())
    }

    /// Resolve all variables in CSS text
    fn resolve_variables(&self, css: &str) -> String {
        // Simple implementation - replace $var references
        let mut result = css.to_string();

        for (name, value) in &self.variable_context.theme_variables {
            let var_pattern = format!("${}", name);
            result = result.replace(&var_pattern, value);
        }

        result
    }
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
