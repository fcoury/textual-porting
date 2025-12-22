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
            foreground: None,      // Defaults to background.inverse() in ColorSystem
            background: None,      // Defaults to #121212 (dark) or #efefef (light) in ColorSystem
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
        primary: "#004578".into(),
        secondary: Some("#0178D4".into()),
        accent: Some("#ffa62b".into()),
        warning: Some("#ffa62b".into()),
        error: Some("#ba3c5b".into()),
        success: Some("#4EBF71".into()),
        foreground: None,
        background: Some("#E0E0E0".into()),
        surface: Some("#D8D8D8".into()),
        panel: Some("#D0D0D0".into()),
        dark: false,
        variables: hashmap! {
            "footer-key-foreground" => "#0178D4",
        },
        ..Default::default()
    });

    m.insert("nord", Theme {
        name: "nord".into(),
        primary: "#88C0D0".into(),
        secondary: Some("#81A1C1".into()),
        accent: Some("#B48EAD".into()),
        warning: Some("#EBCB8B".into()),
        error: Some("#BF616A".into()),
        success: Some("#A3BE8C".into()),
        foreground: Some("#D8DEE9".into()),
        background: Some("#2E3440".into()),
        surface: Some("#3B4252".into()),
        panel: Some("#434C5E".into()),
        dark: true,
        variables: hashmap! {
            "block-cursor-background" => "#88C0D0",
            "block-cursor-foreground" => "#2E3440",
            "block-cursor-text-style" => "none",
            "footer-key-foreground" => "#88C0D0",
            "input-selection-background" => "#81a1c1 35%",
            "button-color-foreground" => "#2E3440",
            "button-focus-text-style" => "reverse",
        },
        ..Default::default()
    });

    m.insert("gruvbox", Theme {
        name: "gruvbox".into(),
        primary: "#85A598".into(),
        secondary: Some("#A89A85".into()),
        warning: Some("#fe8019".into()),
        error: Some("#fb4934".into()),
        success: Some("#b8bb26".into()),
        accent: Some("#fabd2f".into()),
        foreground: Some("#fbf1c7".into()),
        background: Some("#282828".into()),
        surface: Some("#3c3836".into()),
        panel: Some("#504945".into()),
        dark: true,
        variables: hashmap! {
            "block-cursor-foreground" => "#fbf1c7",
            "input-selection-background" => "#689d6a40",
            "button-color-foreground" => "#282828",
        },
        ..Default::default()
    });

    m.insert("catppuccin-mocha", Theme {
        name: "catppuccin-mocha".into(),
        primary: "#F5C2E7".into(),
        secondary: Some("#cba6f7".into()),
        warning: Some("#FAE3B0".into()),
        error: Some("#F28FAD".into()),
        success: Some("#ABE9B3".into()),
        accent: Some("#fab387".into()),
        foreground: Some("#cdd6f4".into()),
        background: Some("#181825".into()),
        surface: Some("#313244".into()),
        panel: Some("#45475a".into()),
        dark: true,
        variables: hashmap! {
            "input-cursor-foreground" => "#11111b",
            "input-cursor-background" => "#f5e0dc",
            "input-selection-background" => "#9399b2 30%",
            "border" => "#b4befe",
            "border-blurred" => "#585b70",
            "footer-background" => "#45475a",
            "block-cursor-foreground" => "#1e1e2e",
            "block-cursor-text-style" => "none",
            "button-color-foreground" => "#181825",
        },
        ..Default::default()
    });

    m.insert("textual-ansi", Theme {
        name: "textual-ansi".into(),
        primary: "ansi_blue".into(),
        secondary: Some("ansi_cyan".into()),
        warning: Some("ansi_yellow".into()),
        error: Some("ansi_red".into()),
        success: Some("ansi_green".into()),
        accent: Some("ansi_bright_blue".into()),
        foreground: Some("ansi_default".into()),
        background: Some("ansi_default".into()),
        surface: Some("ansi_default".into()),
        panel: Some("ansi_default".into()),
        boost: Some("ansi_default".into()),
        dark: false,
        variables: hashmap! {
            "block-cursor-text-style" => "b",
            "block-cursor-blurred-text-style" => "i",
            "input-selection-background" => "ansi_blue",
            "input-cursor-text-style" => "reverse",
            "scrollbar" => "ansi_blue",
            "border-blurred" => "ansi_blue",
            "border" => "ansi_bright_blue",
        },
        ..Default::default()
    });

    m.insert("dracula", Theme {
        name: "dracula".into(),
        primary: "#BD93F9".into(),
        secondary: Some("#6272A4".into()),
        warning: Some("#FFB86C".into()),
        error: Some("#FF5555".into()),
        success: Some("#50FA7B".into()),
        accent: Some("#FF79C6".into()),
        foreground: Some("#F8F8F2".into()),
        background: Some("#282A36".into()),
        surface: Some("#2B2E3B".into()),
        panel: Some("#313442".into()),
        dark: true,
        variables: hashmap! {
            "button-color-foreground" => "#282A36",
        },
        ..Default::default()
    });

    m.insert("tokyo-night", Theme {
        name: "tokyo-night".into(),
        primary: "#BB9AF7".into(),
        secondary: Some("#7AA2F7".into()),
        warning: Some("#E0AF68".into()),
        error: Some("#F7768E".into()),
        success: Some("#9ECE6A".into()),
        accent: Some("#FF9E64".into()),
        foreground: Some("#a9b1d6".into()),
        background: Some("#1A1B26".into()),
        surface: Some("#24283B".into()),
        panel: Some("#414868".into()),
        dark: true,
        variables: hashmap! {
            "button-color-foreground" => "#24283B",
        },
        ..Default::default()
    });

    m.insert("monokai", Theme {
        name: "monokai".into(),
        primary: "#AE81FF".into(),
        secondary: Some("#F92672".into()),
        accent: Some("#66D9EF".into()),
        warning: Some("#FD971F".into()),
        error: Some("#F92672".into()),
        success: Some("#A6E22E".into()),
        foreground: Some("#d6d6d6".into()),
        background: Some("#272822".into()),
        surface: Some("#2e2e2e".into()),
        panel: Some("#3E3D32".into()),
        dark: true,
        variables: hashmap! {
            "foreground-muted" => "#797979",
            "input-selection-background" => "#575b6190",
            "button-color-foreground" => "#272822",
        },
        ..Default::default()
    });

    m.insert("flexoki", Theme {
        name: "flexoki".into(),
        primary: "#205EA6".into(),
        secondary: Some("#24837B".into()),
        warning: Some("#AD8301".into()),
        error: Some("#AF3029".into()),
        success: Some("#66800B".into()),
        accent: Some("#9B76C8".into()),
        foreground: Some("#FFFCF0".into()),
        background: Some("#100F0F".into()),
        surface: Some("#1C1B1A".into()),
        panel: Some("#282726".into()),
        dark: true,
        variables: hashmap! {
            "input-cursor-foreground" => "#5E409D",
            "input-cursor-background" => "#FFFCF0",
            "input-selection-background" => "#6F6E69 35%",
            "button-color-foreground" => "#FFFCF0",
        },
        ..Default::default()
    });

    m.insert("catppuccin-latte", Theme {
        name: "catppuccin-latte".into(),
        primary: "#8839EF".into(),
        secondary: Some("#DC8A78".into()),
        warning: Some("#DF8E1D".into()),
        error: Some("#D20F39".into()),
        success: Some("#40A02B".into()),
        accent: Some("#FE640B".into()),
        foreground: Some("#4C4F69".into()),
        background: Some("#EFF1F5".into()),
        surface: Some("#E6E9EF".into()),
        panel: Some("#CCD0DA".into()),
        dark: false,
        variables: hashmap! {
            "button-color-foreground" => "#EFF1F5",
        },
        ..Default::default()
    });

    m.insert("solarized-light", Theme {
        name: "solarized-light".into(),
        primary: "#268bd2".into(),
        secondary: Some("#2aa198".into()),
        warning: Some("#cb4b16".into()),
        error: Some("#dc322f".into()),
        success: Some("#859900".into()),
        accent: Some("#6c71c4".into()),
        foreground: Some("#586e75".into()),
        background: Some("#fdf6e3".into()),
        surface: Some("#eee8d5".into()),
        panel: Some("#eee8d5".into()),
        dark: false,
        variables: hashmap! {
            "button-color-foreground" => "#fdf6e3",
            "footer-background" => "#268bd2",
            "footer-key-foreground" => "#fdf6e3",
            "footer-description-foreground" => "#fdf6e3",
        },
        ..Default::default()
    });

    m.insert("solarized-dark", Theme {
        name: "solarized-dark".into(),
        primary: "#268bd2".into(),
        secondary: Some("#2aa198".into()),
        warning: Some("#cb4b16".into()),
        error: Some("#dc322f".into()),
        success: Some("#859900".into()),
        accent: Some("#6c71c4".into()),
        foreground: Some("#839496".into()),
        background: Some("#002b36".into()),
        surface: Some("#073642".into()),
        panel: Some("#073642".into()),
        dark: true,
        variables: hashmap! {
            "button-color-foreground" => "#fdf6e3",
            "footer-background" => "#268bd2",
            "footer-key-foreground" => "#fdf6e3",
            "footer-description-foreground" => "#fdf6e3",
            "input-selection-background" => "#073642",
        },
        ..Default::default()
    });

    m.insert("rose-pine", Theme {
        name: "rose-pine".into(),
        primary: "#c4a7e7".into(),
        secondary: Some("#31748f".into()),
        warning: Some("#f6c177".into()),
        error: Some("#eb6f92".into()),
        success: Some("#9ccfd8".into()),
        accent: Some("#ebbcba".into()),
        foreground: Some("#e0def4".into()),
        background: Some("#191724".into()),
        surface: Some("#1f1d2e".into()),
        panel: Some("#26233a".into()),
        dark: true,
        variables: hashmap! {
            "input-cursor-background" => "#f4ede8",
            "input-selection-background" => "#403d52",
            "border" => "#524f67",
            "border-blurred" => "#6e6a86",
            "footer-background" => "#26233a",
            "block-cursor-foreground" => "#191724",
            "block-cursor-text-style" => "none",
            "block-cursor-background" => "#c4a7e7",
        },
        ..Default::default()
    });

    m.insert("rose-pine-moon", Theme {
        name: "rose-pine-moon".into(),
        primary: "#c4a7e7".into(),
        secondary: Some("#3e8fb0".into()),
        warning: Some("#f6c177".into()),
        error: Some("#eb6f92".into()),
        success: Some("#9ccfd8".into()),
        accent: Some("#ea9a97".into()),
        foreground: Some("#e0def4".into()),
        background: Some("#232136".into()),
        surface: Some("#2a273f".into()),
        panel: Some("#393552".into()),
        dark: true,
        variables: hashmap! {
            "input-cursor-background" => "#f4ede8",
            "input-selection-background" => "#44415a",
            "border" => "#56526e",
            "border-blurred" => "#6e6a86",
            "footer-background" => "#393552",
            "block-cursor-foreground" => "#232136",
            "block-cursor-text-style" => "none",
            "block-cursor-background" => "#c4a7e7",
        },
        ..Default::default()
    });

    m.insert("rose-pine-dawn", Theme {
        name: "rose-pine-dawn".into(),
        primary: "#907aa9".into(),
        secondary: Some("#286983".into()),
        warning: Some("#ea9d34".into()),
        error: Some("#b4637a".into()),
        success: Some("#56949f".into()),
        accent: Some("#d7827e".into()),
        foreground: Some("#575279".into()),
        background: Some("#faf4ed".into()),
        surface: Some("#fffaf3".into()),
        panel: Some("#f2e9e1".into()),
        dark: false,
        variables: hashmap! {
            "input-cursor-background" => "#575279",
            "input-selection-background" => "#dfdad9",
            "border" => "#cecacd",
            "border-blurred" => "#9893a5",
            "footer-background" => "#f2e9e1",
            "block-cursor-foreground" => "#faf4ed",
            "block-cursor-text-style" => "none",
            "block-cursor-background" => "#575279",
        },
        ..Default::default()
    });

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
        // Parse background first (needed for foreground default)
        let background = theme.background.as_ref()
            .map(|s| Color::parse(s))
            .transpose()?
            .unwrap_or(if theme.dark {
                Color::parse("#121212")?   // DEFAULT_COLORS["dark"]["background"]
            } else {
                Color::parse("#efefef")?   // DEFAULT_COLORS["light"]["background"]
            });

        // Foreground defaults to background.inverse (Python behavior)
        let foreground = theme.foreground.as_ref()
            .map(|s| Color::parse(s))
            .transpose()?
            .unwrap_or_else(|| background.inverse());

        // Parse primary (required)
        let primary = Color::parse(&theme.primary)?;

        // Optional colors with fallback chain (matches Python):
        // secondary/warning/accent fall back to primary
        // error/success fall back to secondary (or primary if no secondary)
        let secondary = theme.secondary.as_ref()
            .map(|s| Color::parse(s))
            .transpose()?;
        let fallback_secondary = secondary.clone().unwrap_or(primary.clone());

        Ok(ColorSystem {
            primary,
            secondary,
            warning: theme.warning.as_ref()
                .map(|s| Color::parse(s))
                .transpose()?
                .or_else(|| Some(primary.clone())),  // Falls back to primary
            error: theme.error.as_ref()
                .map(|s| Color::parse(s))
                .transpose()?
                .or_else(|| Some(fallback_secondary.clone())),  // Falls back to secondary
            success: theme.success.as_ref()
                .map(|s| Color::parse(s))
                .transpose()?
                .or_else(|| Some(fallback_secondary.clone())),  // Falls back to secondary
            accent: theme.accent.as_ref()
                .map(|s| Color::parse(s))
                .transpose()?
                .or_else(|| Some(primary.clone())),  // Falls back to primary
            foreground,
            background,
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
    ///
    /// Generated variables (organized by category):
    ///
    /// **Semantic Colors (primary, secondary, warning, error, success, accent):**
    /// - `{color}`: Base color
    /// - `{color}-lighten-{1,2,3}`: Lightened shades (steps = spread/2)
    /// - `{color}-darken-{1,2,3}`: Darkened shades (steps = spread/2)
    /// - `{color}-muted`: Blended 70% toward background (all semantic colors)
    /// - `{color}-background`: Background variant (primary/secondary only)
    /// - `{color}-background-{lighten,darken}-{1,2,3}`: Background shade family (primary/secondary only)
    ///
    /// **Base Colors:**
    /// - `foreground`, `background`, `surface`, `panel`, `boost`: Base values
    /// - `{base}-lighten-{1,2,3}`, `{base}-darken-{1,2,3}`: Shades for each
    /// - `surface-active`: surface.lighten(luminosity_spread / 2.5)
    /// - `foreground-muted`: foreground.with_alpha(0.6)
    /// - `foreground-disabled`: foreground.with_alpha(0.38)
    ///
    /// **Text Colors:**
    /// - `text`, `text-muted`, `text-disabled`: "auto X%" or "ansi_default"
    /// - `text-{color}`: contrast_text(background).tint(color.with_alpha(0.66))
    ///
    /// **UI Component Colors:**
    /// - `block-cursor-*`: foreground=$text, blurred-foreground=foreground, blurred-background=primary.with_alpha(0.3)
    /// - `block-hover-background`: boost.with_alpha(0.1)
    /// - `border`: primary, `border-blurred`: surface.darken(0.025)
    /// - `scrollbar-*`: background-darken-1 blended with primary (factors 0.4/0.5)
    /// - `link-*`: color=$text, background-hover=primary, style-hover="bold not underline"
    /// - `footer-*`: description-foreground=foreground (not blended)
    /// - `input-*`: cursor-background=foreground, cursor-foreground=background
    /// - `button-*`: foreground=foreground, color-foreground=$text, focus-text-style="b reverse"
    /// - `markdown-h{1-6}-{color,background,text-style}`: Full heading styles
    ///
    /// **Error Handling:**
    /// When parsing color overrides from custom_variables (e.g., background-darken-1,
    /// primary-lighten-1), invalid values cause ColorParseError to be returned,
    /// matching Python's behavior. Callers must handle the Result.
    pub fn generate_variables(&self) -> Result<HashMap<String, String>, ColorParseError> {
        let mut vars = HashMap::new();

        // Primary color shades - generates:
        //   primary, primary-lighten-{1,2,3}, primary-darken-{1,2,3},
        //   primary-background, primary-muted
        self.add_color_shades_full(&mut vars, "primary", &self.primary);

        // Secondary color (uses fallback if None, so always has a value) - generates:
        //   secondary, secondary-lighten-{1,2,3}, secondary-darken-{1,2,3},
        //   secondary-background, secondary-muted
        let secondary = self.secondary.clone().unwrap_or(self.primary.clone());
        self.add_color_shades_full(&mut vars, "secondary", &secondary);

        // Warning, error, success, accent - get base + lighten/darken + muted
        // (All semantic colors get -muted variant, per Python behavior)
        let warning = self.warning.clone().unwrap_or(self.primary.clone());
        self.add_color_shades_with_muted(&mut vars, "warning", &warning);

        let error = self.error.clone().unwrap_or(secondary.clone());
        self.add_color_shades_with_muted(&mut vars, "error", &error);

        let success = self.success.clone().unwrap_or(secondary.clone());
        self.add_color_shades_with_muted(&mut vars, "success", &success);

        let accent = self.accent.clone().unwrap_or(self.primary.clone());
        self.add_color_shades_with_muted(&mut vars, "accent", &accent);

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

        // surface-active: Python sets surface.lighten(luminosity_spread / 2.5)
        let surface_active = surface.lighten(self.luminosity_spread / 2.5);
        vars.insert("surface-active".into(), surface_active.to_css());

        // Boost: contrast_text(background).with_alpha(0.04)
        let boost = self.boost.clone().unwrap_or_else(|| {
            self.contrast_text(&self.background).with_alpha(0.04)
        });
        vars.insert("boost".into(), boost.to_css());

        // Panel: Python ALWAYS uses surface.blend(primary, 0.1, alpha=1.0) when unset
        // This forces alpha to 1.0 in both light and dark modes to prevent translucent leaks
        // Dark mode also adds boost overlay using panel += boost (additive)
        let panel = self.panel.clone().unwrap_or_else(|| {
            let base = surface.blend_with_alpha(&self.primary, 0.1, Some(1.0));
            if self.dark {
                // Dark mode: Python uses panel += boost (additive blend at alpha=1.0)
                // This uses blend_with_alpha to force result alpha to 1.0
                base.blend_with_alpha(&boost, boost.a, Some(1.0))
            } else {
                base
            }
        });
        vars.insert("panel".into(), panel.to_css());

        // Text colors: Python emits literal "auto X%" strings, not concrete colors
        // These get resolved at runtime based on background
        // For ANSI color mode, Python emits "ansi_default" instead
        if self.is_ansi_foreground() {
            vars.insert("text".into(), "ansi_default".into());
            vars.insert("text-muted".into(), "ansi_default".into());
            vars.insert("text-disabled".into(), "ansi_default".into());
        } else {
            vars.insert("text".into(), "auto 87%".into());
            vars.insert("text-muted".into(), "auto 60%".into());
            vars.insert("text-disabled".into(), "auto 38%".into());
        }

        // Contrast text for colored backgrounds
        // Python: contrast_text is computed from BACKGROUND once, then tinted toward each color
        // Python: text-{color} = contrast_text(background).tint(color.with_alpha(0.66))
        let base_contrast = self.contrast_text(&self.background);
        vars.insert("text-primary".into(), base_contrast.tint(&self.primary.with_alpha(0.66)).to_css());
        vars.insert("text-secondary".into(), base_contrast.tint(&secondary.with_alpha(0.66)).to_css());
        vars.insert("text-warning".into(), base_contrast.tint(&warning.with_alpha(0.66)).to_css());
        vars.insert("text-error".into(), base_contrast.tint(&error.with_alpha(0.66)).to_css());
        vars.insert("text-success".into(), base_contrast.tint(&success.with_alpha(0.66)).to_css());
        vars.insert("text-accent".into(), base_contrast.tint(&accent.with_alpha(0.66)).to_css());

        // Foreground muted/disabled variants
        // Python uses foreground.with_alpha(0.6) and foreground.with_alpha(0.38)
        vars.insert("foreground-muted".into(), self.foreground.with_alpha(0.6).to_css());
        vars.insert("foreground-disabled".into(), self.foreground.with_alpha(0.38).to_css());

        // Base color shades (background, foreground, surface, panel, boost)
        // These get lighten/darken variants like semantic colors
        // Note: Base values already set above, only add shades
        self.add_lighten_darken_shades(&mut vars, "background", &self.background);
        self.add_lighten_darken_shades(&mut vars, "foreground", &self.foreground);
        self.add_lighten_darken_shades(&mut vars, "surface", &surface);
        self.add_lighten_darken_shades(&mut vars, "panel", &panel);
        self.add_lighten_darken_shades(&mut vars, "boost", &boost);

        // ========================================================================
        // PHASE 2: Apply custom_variables BEFORE derived UI components
        // ========================================================================
        // Python's colors.get() returns overridden values because custom_variables
        // are merged before derived values are computed. We apply them here so that
        // overrides to background-darken-1, primary-lighten-1, foreground-muted,
        // scrollbar-background, etc. propagate to derived defaults.
        //
        // EXCEPTION 1: In ANSI foreground mode, text/text-muted/text-disabled are
        // forced to "ansi_default" and cannot be overridden.
        //
        // EXCEPTION 2: When a base color is ANSI, its shade keys are protected.
        // Python's shade loop ignores self.variables when color.ansi is set,
        // always using color.hex. So overrides for shade keys are ignored.
        // Protected patterns: {base}, {base}-lighten-{1,2,3}, {base}-darken-{1,2,3},
        // {base}-background, {base}-background-lighten-{1,2,3}, {base}-background-darken-{1,2,3}
        // Note: {base}-muted is NOT protected - Python uses get() allowing overrides
        let mut ansi_protected_keys: Vec<String> = Vec::new();

        // Text keys protected when foreground is ANSI
        if self.is_ansi_foreground() {
            ansi_protected_keys.extend(["text", "text-muted", "text-disabled"].iter().map(|s| s.to_string()));
        }

        // Helper to add shade keys for an ANSI base color
        // Note: *-muted keys are NOT protected - Python uses get() for them, allowing overrides
        let add_shade_keys = |keys: &mut Vec<String>, base: &str, include_background: bool| {
            keys.push(base.to_string());
            for i in 1..=3 {
                keys.push(format!("{}-lighten-{}", base, i));
                keys.push(format!("{}-darken-{}", base, i));
            }
            // Don't protect {base}-muted - Python allows overrides via get()
            if include_background {
                keys.push(format!("{}-background", base));
                for i in 1..=3 {
                    keys.push(format!("{}-background-lighten-{}", base, i));
                    keys.push(format!("{}-background-darken-{}", base, i));
                }
            }
        };

        // Check each effective color and protect its shades if ANSI
        // Use computed locals (secondary, warning, etc.) not raw Options,
        // so fallbacks inherit ANSI status correctly
        if self.primary.is_ansi() {
            add_shade_keys(&mut ansi_protected_keys, "primary", true);
        }
        if secondary.is_ansi() {
            add_shade_keys(&mut ansi_protected_keys, "secondary", true);
        }
        if warning.is_ansi() {
            add_shade_keys(&mut ansi_protected_keys, "warning", false);
        }
        if error.is_ansi() {
            add_shade_keys(&mut ansi_protected_keys, "error", false);
        }
        if success.is_ansi() {
            add_shade_keys(&mut ansi_protected_keys, "success", false);
        }
        if accent.is_ansi() {
            add_shade_keys(&mut ansi_protected_keys, "accent", false);
        }
        if self.background.is_ansi() {
            add_shade_keys(&mut ansi_protected_keys, "background", false);
        }
        if self.foreground.is_ansi() {
            add_shade_keys(&mut ansi_protected_keys, "foreground", false);
        }
        if surface.is_ansi() {
            add_shade_keys(&mut ansi_protected_keys, "surface", false);
        }
        if panel.is_ansi() {
            add_shade_keys(&mut ansi_protected_keys, "panel", false);
        }
        if boost.is_ansi() {
            add_shade_keys(&mut ansi_protected_keys, "boost", false);
        }

        // text-{color} keys are always computed (no get() in Python), never overridable
        let computed_only_keys: HashSet<String> = [
            "text-primary", "text-secondary", "text-warning",
            "text-error", "text-success", "text-accent",
        ].iter().map(|s| s.to_string()).collect();

        for (name, value) in &self.custom_variables {
            // Skip protected keys for ANSI colors
            if ansi_protected_keys.contains(name) {
                continue;
            }
            // Skip computed-only keys (Python computes directly, no get())
            if computed_only_keys.contains(name) {
                continue;
            }
            vars.insert(name.clone(), value.clone());
        }

        // ========================================================================
        // PHASE 3: Generate derived UI component defaults
        // ========================================================================
        // These use vars.get() to pick up any custom_variables overrides.
        // Use entry().or_insert() for values that can be individually overridden.

        // Block cursor colors (per Python design.py)
        // Python uses setdefault() so custom_variables can override these
        // Derived defaults use vars.get() to pick up overrides (e.g., text override)
        let text_value = vars.get("text").cloned().unwrap_or("auto 87%".into());
        vars.entry("block-cursor-foreground".into()).or_insert(text_value.clone());
        vars.entry("block-cursor-background".into()).or_insert(self.primary.to_css());
        vars.entry("block-cursor-text-style".into()).or_insert("bold".into());
        vars.entry("block-cursor-blurred-foreground".into()).or_insert(self.foreground.to_css());
        vars.entry("block-cursor-blurred-background".into()).or_insert(self.primary.with_alpha(0.3).to_css());
        vars.entry("block-cursor-blurred-text-style".into()).or_insert("none".into());
        vars.entry("block-hover-background".into()).or_insert(boost.with_alpha(0.1).to_css());

        // Border colors (per Python design.py)
        // Python uses setdefault() so custom_variables can override
        vars.entry("border".into()).or_insert(self.primary.to_css());
        vars.entry("border-blurred".into()).or_insert(surface.darken(0.025).to_css());

        // Scrollbar colors (per Python design.py)
        // Python pulls background-darken-1 from generated shades (respects custom_variables overrides)
        // Python uses + semantics: blend(primary, factor, alpha=1.0) - forces alpha to 1.0
        // Python raises ColorParseError if override isn't parseable; we propagate the error
        let bg_darken_1 = match vars.get("background-darken-1") {
            Some(s) => Color::parse(s)?,
            None => self.background.darken(self.luminosity_spread / 2.0),
        };
        // Python + semantics: blend_with_alpha forces result alpha to 1.0
        // Python uses primary.with_alpha(factor) as destination, so interpolation uses that alpha
        let scrollbar_color = bg_darken_1.blend_with_alpha(&self.primary.with_alpha(0.4), 0.4, Some(1.0));
        let scrollbar_hover_color = bg_darken_1.blend_with_alpha(&self.primary.with_alpha(0.5), 0.5, Some(1.0));
        vars.entry("scrollbar".into()).or_insert(scrollbar_color.to_css());
        vars.entry("scrollbar-hover".into()).or_insert(scrollbar_hover_color.to_css());
        vars.entry("scrollbar-active".into()).or_insert(self.primary.to_css());

        // Scrollbar background: set default, then cascade to dependent vars
        // Use entry().or_insert() so custom_variables overrides are preserved
        let scrollbar_bg_default = bg_darken_1.to_css();
        vars.entry("scrollbar-background".into()).or_insert(scrollbar_bg_default);
        // Get the (possibly overridden) scrollbar-background for cascading
        let scrollbar_background = vars.get("scrollbar-background").cloned().unwrap();
        // corner-color and background-hover/active cascade from scrollbar-background
        // but can be individually overridden via custom_variables
        vars.entry("scrollbar-corner-color".into()).or_insert(scrollbar_background.clone());
        vars.entry("scrollbar-background-hover".into()).or_insert(scrollbar_background.clone());
        vars.entry("scrollbar-background-active".into()).or_insert(scrollbar_background);

        // Link colors (per Python design.py)
        // Derived defaults use vars.get() to pick up overrides (e.g., text override → link-color)
        // Note: text_value was computed above for block-cursor-foreground
        vars.entry("link-color".into()).or_insert(text_value.clone());
        vars.entry("link-color-hover".into()).or_insert(text_value);
        vars.entry("link-background".into()).or_insert("initial".into());
        vars.entry("link-background-hover".into()).or_insert(self.primary.to_css());
        vars.entry("link-style".into()).or_insert("underline".into());
        vars.entry("link-style-hover".into()).or_insert("bold not underline".into());

        // Footer colors (per Python design.py)
        vars.entry("footer-foreground".into()).or_insert(self.foreground.to_css());
        vars.entry("footer-background".into()).or_insert(panel.to_css());
        vars.entry("footer-key-foreground".into()).or_insert(accent.to_css());
        vars.entry("footer-key-background".into()).or_insert("transparent".into());
        vars.entry("footer-description-foreground".into()).or_insert(self.foreground.to_css());
        vars.entry("footer-description-background".into()).or_insert("transparent".into());
        vars.entry("footer-item-background".into()).or_insert("transparent".into());

        // Input widget colors (per Python design.py)
        // Derived defaults use vars.get() to pick up overrides (e.g., primary-lighten-1 override)
        vars.entry("input-cursor-foreground".into()).or_insert(self.background.to_css());
        vars.entry("input-cursor-background".into()).or_insert(self.foreground.to_css());
        vars.entry("input-cursor-text-style".into()).or_insert("none".into());
        // Use generated primary-lighten-1 for ANSI/override safety
        // Python raises ColorParseError if override isn't parseable; we propagate the error
        let primary_lighten_1 = match vars.get("primary-lighten-1") {
            Some(s) => Color::parse(s)?,
            None => self.primary.lighten(self.luminosity_spread / 2.0),
        };
        vars.entry("input-selection-background".into()).or_insert(primary_lighten_1.with_alpha(0.4).to_css());

        // Markdown heading colors (per Python design.py)
        // Each heading has -color, -background, and -text-style
        // h1-h3: color=primary, h4-h5: color=foreground, h6: color=foreground-muted
        vars.entry("markdown-h1-color".into()).or_insert(self.primary.to_css());
        vars.entry("markdown-h1-background".into()).or_insert("transparent".into());
        vars.entry("markdown-h1-text-style".into()).or_insert("bold".into());

        vars.entry("markdown-h2-color".into()).or_insert(self.primary.to_css());
        vars.entry("markdown-h2-background".into()).or_insert("transparent".into());
        vars.entry("markdown-h2-text-style".into()).or_insert("underline".into());

        vars.entry("markdown-h3-color".into()).or_insert(self.primary.to_css());
        vars.entry("markdown-h3-background".into()).or_insert("transparent".into());
        vars.entry("markdown-h3-text-style".into()).or_insert("bold".into());

        vars.entry("markdown-h4-color".into()).or_insert(self.foreground.to_css());
        vars.entry("markdown-h4-background".into()).or_insert("transparent".into());
        vars.entry("markdown-h4-text-style".into()).or_insert("bold underline".into());

        vars.entry("markdown-h5-color".into()).or_insert(self.foreground.to_css());
        vars.entry("markdown-h5-background".into()).or_insert("transparent".into());
        vars.entry("markdown-h5-text-style".into()).or_insert("bold".into());

        // h6 uses foreground-muted (which we already generated above)
        // Derived default uses vars.get() to pick up overrides (e.g., foreground-muted override)
        let foreground_muted = vars.get("foreground-muted")
            .cloned()
            .unwrap_or_else(|| self.foreground.with_alpha(0.6).to_css());
        vars.entry("markdown-h6-color".into()).or_insert(foreground_muted);
        vars.entry("markdown-h6-background".into()).or_insert("transparent".into());
        vars.entry("markdown-h6-text-style".into()).or_insert("bold".into());

        // Button colors (per Python design.py)
        // These provide defaults for button widgets; themes can override via custom_variables
        // button-color-foreground uses colors["text"] (auto/ansi), not background
        vars.entry("button-foreground".into()).or_insert(self.foreground.to_css());
        vars.entry("button-color-foreground".into()).or_insert(text_value.clone());
        vars.entry("button-focus-text-style".into()).or_insert("b reverse".into());

        // Note: custom_variables were applied in Phase 2, before derived UI components.
        // This ensures overrides propagate correctly (e.g., text → link-color).

        Ok(vars)
    }

    /// Add only lighten/darken shades (no base color)
    /// Used when base color is already set elsewhere
    /// If color is ANSI, all shades get the same color (Python behavior)
    fn add_lighten_darken_shades(&self, vars: &mut HashMap<String, String>, name: &str, color: &Color) {
        // Python uses luminosity_step = luminosity_spread / 2
        // Default spread 0.15 yields steps of ±0.075, ±0.15, ±0.225
        let luminosity_step = self.luminosity_spread / 2.0;

        // ANSI handling: if color is ANSI, all shades get same color.hex
        let is_ansi = color.is_ansi();

        // Lighten shades (1-3)
        for i in 1..=3 {
            let shade = if is_ansi {
                color.clone()
            } else {
                let factor = luminosity_step * i as f32;
                color.lighten(factor)
            };
            vars.insert(format!("{}-lighten-{}", name, i), shade.to_css());
        }

        // Darken shades (1-3)
        for i in 1..=3 {
            let shade = if is_ansi {
                color.clone()
            } else {
                let factor = luminosity_step * i as f32;
                color.darken(factor)
            };
            vars.insert(format!("{}-darken-{}", name, i), shade.to_css());
        }
    }

    /// Add basic color shades (base + lighten/darken only)
    /// Used internally - generates base color and lighten/darken shades
    fn add_color_shades_basic(&self, vars: &mut HashMap<String, String>, name: &str, color: &Color) {
        // Base color
        vars.insert(name.into(), color.to_css());

        // Add lighten/darken shades
        self.add_lighten_darken_shades(vars, name, color);
    }

    /// Add full color shades including -background (with shade family) and -muted variants
    /// Used for primary and secondary (per Python behavior)
    fn add_color_shades_full(&self, vars: &mut HashMap<String, String>, name: &str, color: &Color) {
        // Add basic shades and -muted first
        self.add_color_shades_with_muted(vars, name, color);

        // ANSI handling: if color is ANSI, all -background shades get same color
        // (Python behavior: ANSI colors don't blend/lighten/darken)
        let is_ansi = color.is_ansi();
        if is_ansi {
            let color_css = color.to_css();
            vars.insert(format!("{}-background", name), color_css.clone());
            for i in 1..=3 {
                vars.insert(format!("{}-background-lighten-{}", name, i), color_css.clone());
                vars.insert(format!("{}-background-darken-{}", name, i), color_css.clone());
            }
            return;
        }

        // Background shade family: Python generates *-background and *-background-{lighten,darken}-{1..3}
        // using the same algorithm for each shade level, NOT HSL lighten/darken
        let luminosity_step = self.luminosity_spread / 2.0;
        let white = Color::parse("#ffffff").unwrap();

        // For each shade level (0 = base, 1-3 = lighten/darken)
        // Python computes: background.blend(color, 0.15).blend(WHITE, spread + delta, alpha=1.0)
        // where delta = luminosity_step * level for lighten, 0 for base, negative concept for darken
        //
        // Python uses alpha=1.0 in these blends to ensure opaque results regardless of
        // input color alpha. We use blend_with_alpha(..., Some(1.0)) to match this behavior.

        if self.dark {
            // Dark theme: background.blend(color, 0.15), then blend toward white with alpha=1.0
            let blended_base = self.background.blend_with_alpha(color, 0.15, Some(1.0));

            // Base *-background (delta = 0)
            let bg_base = blended_base.blend_with_alpha(&white, self.luminosity_spread, Some(1.0));
            vars.insert(format!("{}-background", name), bg_base.to_css());

            // Lighten shades: increasing blend toward white
            for i in 1..=3 {
                let delta = luminosity_step * i as f32;
                let shade = blended_base.blend_with_alpha(&white, self.luminosity_spread + delta, Some(1.0));
                vars.insert(format!("{}-background-lighten-{}", name, i), shade.to_css());
            }

            // Darken shades: decreasing blend toward white (but still blending, not HSL)
            for i in 1..=3 {
                let delta = luminosity_step * i as f32;
                let blend_factor = (self.luminosity_spread - delta).max(0.0);
                let shade = blended_base.blend_with_alpha(&white, blend_factor, Some(1.0));
                vars.insert(format!("{}-background-darken-{}", name, i), shade.to_css());
            }
        } else {
            // Light theme: color.lighten(delta) for each shade
            // Base *-background (delta = 0, i.e., the color itself lightened by 0)
            vars.insert(format!("{}-background", name), color.to_css());

            // Lighten shades
            for i in 1..=3 {
                let delta = luminosity_step * i as f32;
                let shade = color.lighten(delta);
                vars.insert(format!("{}-background-lighten-{}", name, i), shade.to_css());
            }

            // Darken shades
            for i in 1..=3 {
                let delta = luminosity_step * i as f32;
                let shade = color.darken(delta);
                vars.insert(format!("{}-background-darken-{}", name, i), shade.to_css());
            }
        }
    }

    /// Add color shades with -muted variant (no -background)
    /// Used for warning, error, success, accent
    fn add_color_shades_with_muted(&self, vars: &mut HashMap<String, String>, name: &str, color: &Color) {
        // Add basic shades first
        self.add_color_shades_basic(vars, name, color);

        // Muted variant: Python uses blend(background, 0.7), not alpha
        // All semantic colors get this (primary, secondary, warning, error, success, accent)
        let muted = color.blend(&self.background, 0.7);
        vars.insert(format!("{}-muted", name), muted.to_css());
    }

    /// Check if foreground is an ANSI color (for text variable generation)
    /// Python checks if foreground color was parsed from an ANSI color name
    fn is_ansi_foreground(&self) -> bool {
        // ANSI colors are terminal palette colors (0-15) or named ANSI colors
        // Python's Color class has an .ansi property that tracks this
        // When foreground is ANSI (e.g., "ansi_default"), text vars emit "ansi_default"
        // The Color struct tracks this via its `ansi` field
        self.foreground.is_ansi()
    }

    /// Get base contrasting text color for a background (matches Python's get_contrast_text)
    /// Uses brightness < 0.5 threshold (Textual's Color.get_contrast_text)
    fn contrast_text(&self, background: &Color) -> Color {
        // Calculate brightness (simple RGB average)
        let brightness = background.brightness();

        // Python uses brightness < 0.5 threshold
        if brightness < 0.5 {
            // Dark background: use white text
            Color::parse("#ffffff").unwrap()
        } else {
            // Light background: use black text
            Color::parse("#000000").unwrap()
        }
    }

}

/// Error type for color parsing failures
/// Matches Python's ColorParseError for strict parity
#[derive(Debug, Clone, PartialEq)]
pub enum ColorParseError {
    /// Invalid color format (not hex, rgb, hsl, or named)
    InvalidFormat(String),
    /// Invalid hex color (wrong length, invalid characters)
    InvalidHex(String),
    /// Invalid RGB values (out of range)
    InvalidRgb(String),
    /// Invalid HSL values (out of range)
    InvalidHsl(String),
    /// Unknown color name
    UnknownName(String),
}

impl std::fmt::Display for ColorParseError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::InvalidFormat(s) => write!(f, "invalid color format: {}", s),
            Self::InvalidHex(s) => write!(f, "invalid hex color: {}", s),
            Self::InvalidRgb(s) => write!(f, "invalid RGB color: {}", s),
            Self::InvalidHsl(s) => write!(f, "invalid HSL color: {}", s),
            Self::UnknownName(s) => write!(f, "unknown color name: {}", s),
        }
    }
}

impl std::error::Error for ColorParseError {}

/// ANSI color identity for round-trip serialization
/// Stores the exact token string (ansi_default, ansi_black, ansi_red, etc.)
/// to match Python's output exactly
#[derive(Clone, Debug, PartialEq)]
pub struct AnsiColor(pub String);

impl AnsiColor {
    /// Create from token string (e.g., "ansi_default", "ansi_blue")
    pub fn new(token: impl Into<String>) -> Self {
        AnsiColor(token.into())
    }

    /// Convert to CSS string for serialization
    /// Returns the exact token for round-trip fidelity
    pub fn to_css(&self) -> String {
        self.0.clone()
    }
}

/// Color representation with RGBA values and origin tracking
///
/// The `ansi` and `auto` fields track how the color was specified:
/// - `ansi`: Some(AnsiColor) for terminal palette colors, preserving identity for round-trip
/// - `auto`: True for dynamic text colors ("auto 87%", "auto 60%", etc.)
///
/// These flags affect blending behavior and shade generation:
/// - ANSI colors don't blend/lighten/darken; all shades use the same color
/// - Auto colors resolve to contrast_text(background) during blend operations
#[derive(Clone, Debug, PartialEq)]
pub struct Color {
    pub r: u8,
    pub g: u8,
    pub b: u8,
    pub a: f32,
    pub ansi: Option<AnsiColor>,  // Preserves ANSI identity for round-trip serialization
    pub auto: bool,               // True if from "auto X%" specification
}

/// Default dark theme background (matches Python DEFAULT_COLORS["dark"]["background"])
pub static DEFAULT_DARK_BACKGROUND: Color = Color { r: 18, g: 18, b: 18, a: 1.0, ansi: None, auto: false };  // #121212

/// Default dark theme surface (matches Python DEFAULT_COLORS["dark"]["surface"])
pub static DEFAULT_DARK_SURFACE: Color = Color { r: 30, g: 30, b: 30, a: 1.0, ansi: None, auto: false };  // #1e1e1e

/// Default light theme background (matches Python DEFAULT_COLORS["light"]["background"])
pub static DEFAULT_LIGHT_BACKGROUND: Color = Color { r: 239, g: 239, b: 239, a: 1.0, ansi: None, auto: false };  // #efefef

/// Default light theme surface (matches Python DEFAULT_COLORS["light"]["surface"])
pub static DEFAULT_LIGHT_SURFACE: Color = Color { r: 245, g: 245, b: 245, a: 1.0, ansi: None, auto: false };  // #f5f5f5

impl Color {
    /// Calculate brightness (0.0 to 1.0)
    /// Used by get_contrast_text - simple perceived brightness formula
    pub fn brightness(&self) -> f32 {
        // Perceived brightness formula (same as Python Textual)
        (0.299 * self.r as f32 + 0.587 * self.g as f32 + 0.114 * self.b as f32) / 255.0
    }

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
    ///
    /// Python semantics (from rich/color.py blend()):
    /// 1. Factor is clamped to 0.0-1.0
    /// 2. Destination (other) handling:
    ///    - If other is ANSI: return other (destination wins for ANSI)
    ///    - If other is "auto": convert to contrast_text(self), then blend
    /// 3. Self being ANSI does NOT cause short-circuit
    /// 4. Optional alpha override forces result alpha (used for panel, scrollbar)
    /// 5. Otherwise, linear interpolation in RGBA space
    ///
    /// This matters for ANSI themes where blending toward ANSI returns ANSI.
    pub fn blend(&self, other: &Color, factor: f32) -> Color {
        self.blend_with_alpha(other, factor, None)
    }

    /// Blend with optional alpha override
    ///
    /// When alpha is Some(value), the result alpha is forced to that value.
    /// This matches Python's blend(alpha=...) parameter used in:
    /// - Panel dark mode: base.blend(&boost.with_alpha(1.0), boost.a)
    /// - Scrollbar colors: blend(primary, factor).with_alpha(1.0)
    pub fn blend_with_alpha(&self, other: &Color, factor: f32, alpha: Option<f32>) -> Color {
        // Destination ANSI handling: if other is ANSI, return other unchanged
        // Python: "if destination is ANSI, return destination"
        // This allows blending toward an ANSI color to preserve the ANSI value
        if other.is_ansi() {
            return other.clone();
        }

        // Destination auto handling: if other is "auto", resolve to contrast_text(self)
        // Python converts auto colors to contrast_text based on the source color,
        // using the destination alpha (other.a) for the resolved color.
        let effective_other = if other.is_auto() {
            self.contrast_text_with_alpha(other.a)
        } else {
            other.clone()
        };

        // Factor bounds handling (Python semantics)
        // factor <= 0: return self unchanged
        // factor >= 1: return effective_other directly (Python ignores alpha in this case)
        if factor <= 0.0 {
            return self.clone();
        }
        if factor >= 1.0 {
            return effective_other;
        }

        // Standard RGBA interpolation (factor is in (0, 1) range)
        let result = Color {
            r: lerp_u8(self.r, effective_other.r, factor),
            g: lerp_u8(self.g, effective_other.g, factor),
            b: lerp_u8(self.b, effective_other.b, factor),
            a: lerp_f32(self.a, effective_other.a, factor),
            ansi: None,
            auto: false,
        };

        // Apply alpha override if specified
        match alpha {
            Some(a) => result.with_alpha(a),
            None => result,
        }
    }

    /// Check if this color is an ANSI color
    /// ANSI colors are terminal palette colors (0-15) or named ANSI colors
    /// When true, shade generation should emit the same color for all shades
    pub fn is_ansi(&self) -> bool {
        self.ansi.is_some()
    }

    /// Convert to CSS string for serialization
    /// ANSI colors emit their original token (ansi_default, ansi_blue, etc.)
    /// for correct round-trip serialization
    /// Python uses uppercase hex (#RRGGBB or #RRGGBBAA), not rgba()
    pub fn to_css(&self) -> String {
        // If this is an ANSI color, emit the ANSI token for round-trip fidelity
        if let Some(ref ansi) = self.ansi {
            return ansi.to_css();
        }

        // Python's .hex uses uppercase #RRGGBB or #RRGGBBAA (no rgba format)
        if self.a >= 1.0 {
            format!("#{:02X}{:02X}{:02X}", self.r, self.g, self.b)
        } else {
            // Python's int(alpha * 255) truncates; clamp to [0,1] first for safety
            let alpha_clamped = self.a.clamp(0.0, 1.0);
            let alpha_byte = (alpha_clamped * 255.0) as u8;  // truncate, not round
            format!("#{:02X}{:02X}{:02X}{:02X}", self.r, self.g, self.b, alpha_byte)
        }
    }

    /// Check if this color is an "auto" color
    /// Auto colors are specified as "auto X%" and resolve at runtime based on
    /// contrast with the background. They represent dynamic text colors.
    pub fn is_auto(&self) -> bool {
        // This requires the Color struct to track its origin:
        // - Colors parsed from "auto 87%", "auto 60%", etc.
        //
        // For implementation, add an `auto: bool` field to Color struct
        // and set true when parsing "auto X%" values
        self.auto
    }

    /// Compute contrast text color for this color as background with specified alpha
    /// Returns white or black depending on this color's brightness (not luminance)
    /// Used when resolving "auto" colors during blend operations
    /// Python: Color.get_contrast_text(alpha) uses brightness < 0.5 threshold
    fn contrast_text_with_alpha(&self, alpha: f32) -> Color {
        // Python uses brightness < 0.5, not luminance
        if self.brightness() < 0.5 {
            Color { r: 255, g: 255, b: 255, a: alpha, ansi: None, auto: false }
        } else {
            Color { r: 0, g: 0, b: 0, a: alpha, ansi: None, auto: false }
        }
    }

    /// Compute contrast text color with default alpha (0.95)
    /// Wrapper for contrast_text_with_alpha with Python's default alpha
    fn contrast_text(&self) -> Color {
        self.contrast_text_with_alpha(0.95)
    }
}

fn lerp_u8(a: u8, b: u8, t: f32) -> u8 {
    // Python uses int() which truncates; don't round to match exactly
    (a as f32 + (b as f32 - a as f32) * t) as u8
}

fn lerp_f32(a: f32, b: f32, t: f32) -> f32 {
    a + (b - a) * t
}

impl Color {
    /// Lighten the color by factor (0.0 to 1.0)
    /// Python uses LAB color space (rgb_to_lab / lab_to_rgb) for perceptual uniformity
    /// Note: Result is not ANSI/auto (LAB transformation produces concrete color)
    pub fn lighten(&self, factor: f32) -> Color {
        let (l, a_lab, b_lab) = self.to_lab();
        // L ranges 0-100 in LAB; factor is 0.0-1.0, so scale by 100
        let new_l = (l + factor * 100.0).clamp(0.0, 100.0);
        Color::from_lab(new_l, a_lab, b_lab, self.a)  // from_lab sets ansi=false, auto=false
    }

    /// Darken the color by factor (0.0 to 1.0)
    /// Python uses LAB color space (rgb_to_lab / lab_to_rgb) for perceptual uniformity
    /// Note: Result is not ANSI/auto (LAB transformation produces concrete color)
    pub fn darken(&self, factor: f32) -> Color {
        let (l, a_lab, b_lab) = self.to_lab();
        // L ranges 0-100 in LAB; factor is 0.0-1.0, so scale by 100
        let new_l = (l - factor * 100.0).clamp(0.0, 100.0);
        Color::from_lab(new_l, a_lab, b_lab, self.a)  // from_lab sets ansi=false, auto=false
    }

    /// Convert RGB to LAB color space
    /// Python's rgb_to_lab implementation for perceptual lightness adjustments
    /// Uses Python's exact coefficients for parity
    fn to_lab(&self) -> (f32, f32, f32) {
        // RGB to XYZ (sRGB with D65 illuminant)
        // Python uses coefficients: 41.24, 35.76, 18.05, etc. (scaled by 100)
        let r = Self::srgb_to_linear(self.r as f32 / 255.0);
        let g = Self::srgb_to_linear(self.g as f32 / 255.0);
        let b = Self::srgb_to_linear(self.b as f32 / 255.0);

        // Python's exact RGB→XYZ matrix (scaled for 0-100 range)
        let x = r * 0.4124 + g * 0.3576 + b * 0.1805;
        let y = r * 0.2126 + g * 0.7152 + b * 0.0722;
        let z = r * 0.0193 + g * 0.1192 + b * 0.9505;

        // XYZ to LAB (D65 reference white)
        let xn = 0.95047;
        let yn = 1.0;
        let zn = 1.08883;

        let fx = Self::lab_f(x / xn);
        let fy = Self::lab_f(y / yn);
        let fz = Self::lab_f(z / zn);

        let l = 116.0 * fy - 16.0;
        let a = 500.0 * (fx - fy);
        let b = 200.0 * (fy - fz);

        (l, a, b)
    }

    /// Convert LAB to RGB color space
    /// Uses Python's exact coefficients for parity
    fn from_lab(l: f32, a_lab: f32, b_lab: f32, alpha: f32) -> Color {
        // LAB to XYZ
        let xn = 0.95047;
        let yn = 1.0;
        let zn = 1.08883;

        let fy = (l + 16.0) / 116.0;
        let fx = a_lab / 500.0 + fy;
        let fz = fy - b_lab / 200.0;

        let x = xn * Self::lab_f_inv(fx);
        let y = yn * Self::lab_f_inv(fy);
        let z = zn * Self::lab_f_inv(fz);

        // XYZ to RGB (Python's exact coefficients)
        let r = x *  3.2406 + y * -1.5372 + z * -0.4986;
        let g = x * -0.9689 + y *  1.8758 + z *  0.0415;
        let b = x *  0.0557 + y * -0.2040 + z *  1.0570;

        Color {
            r: (Self::linear_to_srgb(r) * 255.0).clamp(0.0, 255.0) as u8,
            g: (Self::linear_to_srgb(g) * 255.0).clamp(0.0, 255.0) as u8,
            b: (Self::linear_to_srgb(b) * 255.0).clamp(0.0, 255.0) as u8,
            a: alpha,
            ansi: None,
            auto: false,
        }
    }

    fn srgb_to_linear(c: f32) -> f32 {
        if c <= 0.04045 { c / 12.92 } else { ((c + 0.055) / 1.055).powf(2.4) }
    }

    fn linear_to_srgb(c: f32) -> f32 {
        if c <= 0.0031308 { c * 12.92 } else { 1.055 * c.powf(1.0 / 2.4) - 0.055 }
    }

    fn lab_f(t: f32) -> f32 {
        // Python's exact threshold: 0.008856 (6/29)^3
        if t > 0.008856 { t.cbrt() } else { 7.787 * t + 16.0 / 116.0 }
    }

    fn lab_f_inv(t: f32) -> f32 {
        // Python's exact threshold: 0.2068930344 (6/29)
        let delta = 0.2068930344;
        if t > delta { t.powi(3) } else { 3.0 * delta * delta * (t - 4.0 / 29.0) }
    }

    /// Return color with new alpha
    /// Python's with_alpha returns a concrete color (ansi/auto cleared)
    pub fn with_alpha(&self, alpha: f32) -> Color {
        Color {
            r: self.r,
            g: self.g,
            b: self.b,
            a: alpha,
            ansi: None,
            auto: false,
        }
    }

    /// Get inverse color (complement in RGB space)
    /// Used for foreground default when not specified
    pub fn inverse(&self) -> Color {
        Color {
            r: 255 - self.r,
            g: 255 - self.g,
            b: 255 - self.b,
            a: self.a,
            ansi: None,  // Inverse of ANSI is not ANSI
            auto: false,  // Inverse of auto is not auto
        }
    }

    /// Tint this color toward another color
    /// Used for text-{color} variables: contrast_text.tint(color.with_alpha(0.66))
    /// Python's tint: returns self if either is ANSI, keeps self.a, explicit RGB lerp
    pub fn tint(&self, tint_color: &Color) -> Color {
        // ANSI guard: if either color is ANSI, return self unchanged
        if self.is_ansi() || tint_color.is_ansi() {
            return self.clone();
        }

        // Explicit RGB interpolation using tint_color's alpha as factor
        // Unlike blend(), this preserves self.a and doesn't handle auto
        let factor = tint_color.a;
        Color {
            r: lerp_u8(self.r, tint_color.r, factor),
            g: lerp_u8(self.g, tint_color.g, factor),
            b: lerp_u8(self.b, tint_color.b, factor),
            a: self.a,  // Preserve original alpha (unlike blend)
            ansi: None,
            auto: false,
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
            theme_variables: color_system.generate_variables()?,
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

        let theme_vars = color_system.generate_variables()
            .map_err(|e| ThemeError::InvalidColor(e.to_string()))?;
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
