# Analysis: Python Textual theme.py

## Overview

The `theme.py` file implements the theming system that provides consistent color schemes across the application. Themes define color variables that are used throughout the CSS.

## Key Components

### 1. Theme Dataclass

```python
@dataclass
class Theme:
    name: str              # Theme identifier

    # Core semantic colors
    primary: str           # Primary brand color
    secondary: str | None = None
    warning: str | None = None
    error: str | None = None
    success: str | None = None
    accent: str | None = None

    # Base colors
    foreground: str | None = None
    background: str | None = None
    surface: str | None = None    # Elevated surface
    panel: str | None = None      # Panel backgrounds
    boost: str | None = None      # Highlight color

    # Theme settings
    dark: bool = True             # Dark or light theme
    luminosity_spread: float = 0.15  # Color variation range
    text_alpha: float = 0.95      # Default text opacity

    # Custom CSS variables
    variables: dict[str, str] = field(default_factory=dict)
```

### 2. Built-in Themes

```python
BUILTIN_THEMES: dict[str, Theme] = {
    "textual-dark": Theme(
        name="textual-dark",
        primary="#0178D4",
        secondary="#004578",
        accent="#ffa62b",
        warning="#ffa62b",
        error="#ba3c5b",
        success="#4EBF71",
        foreground="#e0e0e0",
    ),
    "textual-light": Theme(..., dark=False),
    "nord": Theme(...),
    "gruvbox": Theme(...),
    "catppuccin-mocha": Theme(...),
    "dracula": Theme(...),
    "tokyo-night": Theme(...),
    "monokai": Theme(...),
    "flexoki": Theme(...),
    "solarized-light": Theme(...),
    "solarized-dark": Theme(...),
    "rose-pine": Theme(...),
    # ... and more
}
```

### 3. ColorSystem Integration

Themes convert to ColorSystem for CSS variable generation:

```python
def to_color_system(self) -> ColorSystem:
    return ColorSystem(
        primary=self.primary,
        secondary=self.secondary,
        warning=self.warning,
        error=self.error,
        success=self.success,
        accent=self.accent,
        foreground=self.foreground,
        background=self.background,
        surface=self.surface,
        panel=self.panel,
        boost=self.boost,
        dark=self.dark,
        luminosity_spread=self.luminosity_spread,
        text_alpha=self.text_alpha,
        variables=self.variables,
    )
```

### 4. ThemeProvider

Command palette integration for theme switching:

```python
class ThemeProvider(Provider):
    @property
    def commands(self) -> list[tuple[str, Callable[[], None]]]:
        themes = self.app.available_themes
        # Note: "textual-ansi" is explicitly excluded from command palette
        return [
            (theme.name, partial(set_app_theme, theme.name))
            for theme in themes.values()
            if theme.name != "textual-ansi"
        ]

    async def discover(self) -> Hits:
        """Yields all themes for command palette discovery."""
        for command in self.commands:
            yield DiscoveryHit(*command)

    async def search(self, query: str) -> Hits:
        """Search themes by name."""
        matcher = self.matcher(query)
        for name, callback in self.commands:
            if (match := matcher.match(name)) > 0:
                yield Hit(match, matcher.highlight(name), callback)
```

## CSS Variable Generation

From `design.py` ColorSystem:

```python
class ColorSystem:
    def generate_variables(self) -> dict[str, str]:
        # Primary shades: $primary, $primary-lighten-1, $primary-darken-1, etc.
        # Secondary shades
        # Warning, error, success shades
        # Background, surface, panel variants
        # Text colors with opacity
        # Custom variables from theme
```

Example generated variables:
```css
$primary: #0178D4;
$primary-lighten-1: #1a89dd;
$primary-lighten-2: #349ae6;
$primary-darken-1: #016abc;
$primary-darken-2: #015da4;
$background: #1e1e1e;
$foreground: #e0e0e0;
$text: rgba(224, 224, 224, 0.95);  /* foreground with text_alpha */
```

## Theme Application Flow

1. **Set Theme**
   ```python
   app.theme = "nord"
   ```

2. **Generate Variables**
   ```python
   color_system = theme.to_color_system()
   variables = color_system.generate_variables()
   ```

3. **Update Stylesheet**
   ```python
   stylesheet.set_variables(variables)
   stylesheet.reparse()
   ```

4. **Refresh Styles**
   ```python
   app._refresh_styles()
   ```

## Rust Implementation Strategy

### Data Structures

```rust
#[derive(Clone)]
pub struct Theme {
    pub name: String,

    // Semantic colors
    pub primary: String,
    pub secondary: Option<String>,
    pub warning: Option<String>,
    pub error: Option<String>,
    pub success: Option<String>,
    pub accent: Option<String>,

    // Base colors
    pub foreground: Option<String>,
    pub background: Option<String>,
    pub surface: Option<String>,
    pub panel: Option<String>,
    pub boost: Option<String>,

    // Settings
    pub dark: bool,
    pub luminosity_spread: f32,
    pub text_alpha: f32,

    // Custom variables
    pub variables: HashMap<String, String>,
}

pub static BUILTIN_THEMES: Lazy<HashMap<&str, Theme>> = Lazy::new(|| {
    let mut m = HashMap::new();
    m.insert("textual-dark", Theme {
        name: "textual-dark".into(),
        primary: "#0178D4".into(),
        // ...
    });
    // ... more themes
    m
});
```

### ColorSystem

```rust
pub struct ColorSystem {
    // Same fields as Theme
}

impl ColorSystem {
    pub fn generate_variables(&self) -> HashMap<String, String> {
        let mut vars = HashMap::new();

        // Generate primary shades
        let primary = Color::parse(&self.primary)?;
        vars.insert("primary".into(), primary.to_css());
        vars.insert("primary-lighten-1".into(), primary.lighten(0.15).to_css());
        vars.insert("primary-darken-1".into(), primary.darken(0.15).to_css());
        // ... more shades

        // Add custom variables
        for (k, v) in &self.variables {
            vars.insert(k.clone(), v.clone());
        }

        vars
    }
}

impl From<Theme> for ColorSystem {
    fn from(theme: Theme) -> Self {
        ColorSystem {
            primary: theme.primary,
            secondary: theme.secondary,
            // ...
        }
    }
}
```

### Theme Registry

```rust
pub struct ThemeRegistry {
    themes: HashMap<String, Theme>,
}

impl ThemeRegistry {
    pub fn new() -> Self {
        let mut registry = Self { themes: HashMap::new() };
        // Register built-in themes
        for (name, theme) in BUILTIN_THEMES.iter() {
            registry.register(theme.clone());
        }
        registry
    }

    pub fn register(&mut self, theme: Theme) {
        self.themes.insert(theme.name.clone(), theme);
    }

    pub fn get(&self, name: &str) -> Option<&Theme> {
        self.themes.get(name)
    }
}
```

### Usage in App

```rust
impl App {
    pub fn set_theme(&mut self, name: &str) -> Result<(), ThemeError> {
        let theme = self.theme_registry.get(name)
            .ok_or(ThemeError::NotFound(name.into()))?;
        let color_system = ColorSystem::from(theme.clone());
        let variables = color_system.generate_variables();
        self.stylesheet.set_variables(variables);
        self.stylesheet.reparse()?;
        self.refresh_styles();
        Ok(())
    }
}
```
