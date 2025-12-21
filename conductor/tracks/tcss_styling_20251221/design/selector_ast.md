# Design: Selector AST

## Overview

This document defines the Abstract Syntax Tree (AST) for TCSS selectors. The selector system enables matching DOM nodes using type names, classes, IDs, and pseudo-classes with descendant and child combinators.

**Important:** TCSS does NOT support sibling combinators (`~`, `+`). Only descendant (space) and child (`>`) combinators are supported.

## Selector Grammar

```
selector_list   = selector ("," selector)*
selector        = compound_selector (combinator compound_selector)*
compound_selector = simple_selector+
simple_selector = type_selector | class_selector | id_selector | pseudo_class | universal
combinator      = SPACE | ">"

type_selector   = IDENTIFIER
class_selector  = "." IDENTIFIER
id_selector     = "#" IDENTIFIER
pseudo_class    = ":" IDENTIFIER
universal       = "*"
```

## AST Types

### Core Structures

```rust
/// A list of selectors separated by commas (OR logic)
#[derive(Debug, Clone, PartialEq)]
pub struct SelectorSet {
    pub selectors: Vec<Selector>,
}

/// A single selector chain with combinators
#[derive(Debug, Clone, PartialEq)]
pub struct Selector {
    /// Sequence of compound selectors with combinators
    pub parts: Vec<SelectorPart>,
    /// Precomputed specificity for cascade ordering
    pub specificity: Specificity,
}

/// A compound selector with its combinator to the next part
#[derive(Debug, Clone, PartialEq)]
pub struct SelectorPart {
    pub compound: CompoundSelector,
    pub combinator: Combinator,
}

/// Multiple simple selectors on the same element (AND logic)
#[derive(Debug, Clone, PartialEq)]
pub struct CompoundSelector {
    pub simple_selectors: Vec<SimpleSelector>,
}

/// A single simple selector
#[derive(Debug, Clone, PartialEq)]
pub enum SimpleSelector {
    /// Matches widget type name (e.g., `Button`)
    Type(String),

    /// Matches class (e.g., `.primary`)
    Class(String),

    /// Matches ID (e.g., `#submit`)
    Id(String),

    /// Matches pseudo-class (e.g., `:hover`)
    PseudoClass(PseudoClass),

    /// Matches any element (e.g., `*`)
    Universal,
}

/// Relationship between selector parts
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub enum Combinator {
    /// No combinator - this is the final/rightmost part
    #[default]
    None,

    /// Descendant combinator (space) - matches any descendant
    Descendant,

    /// Child combinator (>) - matches direct child only
    Child,
}
```

### Pseudo-Classes

```rust
/// Supported pseudo-classes
#[derive(Debug, Clone, PartialEq)]
pub enum PseudoClass {
    // Focus states
    Focus,
    FocusWithin,
    Blur,

    // Interaction states
    Hover,
    Enabled,
    Disabled,

    // Structural (cache-invalidating)
    FirstOfType,
    LastOfType,
    FirstChild,
    LastChild,
    Odd,
    Even,
    Empty,

    // Theme
    Dark,
    Light,
    Ansi,     // Terminal in ANSI-only mode
    Nocolor,  // No color support

    // Inline styles
    Inline,

    // Custom/unknown pseudo-class (for extensibility)
    Custom(String),
}

impl PseudoClass {
    /// Parse from string (without leading colon)
    pub fn parse(name: &str) -> Self {
        match name {
            "focus" => PseudoClass::Focus,
            "focus-within" => PseudoClass::FocusWithin,
            "blur" => PseudoClass::Blur,
            "hover" => PseudoClass::Hover,
            "enabled" => PseudoClass::Enabled,
            "disabled" => PseudoClass::Disabled,
            "first-of-type" => PseudoClass::FirstOfType,
            "last-of-type" => PseudoClass::LastOfType,
            "first-child" => PseudoClass::FirstChild,
            "last-child" => PseudoClass::LastChild,
            "odd" => PseudoClass::Odd,
            "even" => PseudoClass::Even,
            "empty" => PseudoClass::Empty,
            "dark" => PseudoClass::Dark,
            "light" => PseudoClass::Light,
            "ansi" => PseudoClass::Ansi,
            "nocolor" | "no-color" => PseudoClass::Nocolor,
            "inline" => PseudoClass::Inline,
            _ => PseudoClass::Custom(name.to_string()),
        }
    }

    /// Does this pseudo-class invalidate style caching?
    /// These depend on DOM structure, not just the node itself
    pub fn invalidates_cache(&self) -> bool {
        matches!(self,
            PseudoClass::FirstOfType | PseudoClass::LastOfType |
            PseudoClass::FirstChild | PseudoClass::LastChild |
            PseudoClass::Odd | PseudoClass::Even |
            PseudoClass::FocusWithin | PseudoClass::Empty
        )
    }
}
```

## Specificity Calculation

```rust
/// CSS specificity for cascade ordering
/// Uses 6-tuple for full cascade priority
///
/// Ordering: Higher tuple value wins (use max() to select winning rule)
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Default)]
pub struct Specificity {
    /// 0 for DEFAULT_CSS (widget defaults), 1 for user CSS (user CSS wins)
    pub is_user_css: u8,
    /// 1 if !important, 0 otherwise (higher wins)
    pub important: u8,
    /// Count of ID selectors
    pub ids: u16,
    /// Count of class, pseudo-class, and attribute selectors
    pub classes: u16,
    /// Count of type selectors
    pub types: u16,
    /// Source order tie-breaker (higher wins)
    pub tie_breaker: u32,
}

impl Specificity {
    pub const ZERO: Specificity = Specificity {
        is_user_css: 0,
        important: 0,
        ids: 0,
        classes: 0,
        types: 0,
        tie_breaker: 0,
    };

    /// Create specificity for DEFAULT_CSS (widget defaults)
    pub fn default_css(ids: u16, classes: u16, types: u16, tie_breaker: u32) -> Self {
        Specificity { is_user_css: 0, important: 0, ids, classes, types, tie_breaker }
    }

    /// Create specificity for user CSS
    pub fn user_css(ids: u16, classes: u16, types: u16, tie_breaker: u32) -> Self {
        Specificity { is_user_css: 1, important: 0, ids, classes, types, tie_breaker }
    }

    /// Create specificity for inline styles (highest priority)
    pub fn inline() -> Self {
        Specificity {
            is_user_css: 1,
            important: 0,
            ids: u16::MAX,
            classes: 0,
            types: 0,
            tie_breaker: 0,
        }
    }

    /// Convert to comparable tuple for ordering (higher wins)
    pub fn as_tuple(&self) -> (u8, u8, u16, u16, u16, u32) {
        (
            self.is_user_css,  // 1 (user) > 0 (default)
            self.important,
            self.ids,
            self.classes,
            self.types,
            self.tie_breaker,
        )
    }
}

impl Selector {
    /// Calculate specificity from selector parts
    pub fn calculate_specificity(&self) -> Specificity {
        let mut spec = Specificity::ZERO;

        for part in &self.parts {
            for simple in &part.compound.simple_selectors {
                match simple {
                    SimpleSelector::Id(_) => spec.ids += 1,
                    SimpleSelector::Class(_) => spec.classes += 1,
                    SimpleSelector::PseudoClass(_) => spec.classes += 1,
                    SimpleSelector::Type(_) => spec.types += 1,
                    SimpleSelector::Universal => {} // No specificity
                }
            }
        }

        spec
    }
}
```

## Selector Parsing

```rust
pub struct SelectorParser<'a> {
    input: &'a str,
    pos: usize,
}

impl<'a> SelectorParser<'a> {
    pub fn new(input: &'a str) -> Self {
        SelectorParser { input, pos: 0 }
    }

    /// Parse a complete selector list
    pub fn parse(&mut self) -> Result<SelectorSet, SelectorParseError> {
        let mut selectors = Vec::new();

        loop {
            self.skip_whitespace();
            let selector = self.parse_selector()?;
            selectors.push(selector);

            self.skip_whitespace();
            if !self.consume(',') {
                break;
            }
        }

        Ok(SelectorSet { selectors })
    }

    /// Parse a single selector chain
    fn parse_selector(&mut self) -> Result<Selector, SelectorParseError> {
        let mut parts = Vec::new();

        loop {
            let compound = self.parse_compound_selector()?;

            // Check for combinator
            let had_whitespace = self.skip_whitespace();
            let combinator = if self.consume('>') {
                self.skip_whitespace();
                Combinator::Child
            } else if had_whitespace && !self.is_at_end() && !self.peek_char_is(',') {
                Combinator::Descendant
            } else {
                Combinator::None
            };

            parts.push(SelectorPart { compound, combinator });

            if combinator == Combinator::None {
                break;
            }
        }

        let mut selector = Selector {
            parts,
            specificity: Specificity::ZERO,
        };
        selector.specificity = selector.calculate_specificity();

        Ok(selector)
    }

    /// Parse a compound selector (multiple simple selectors)
    fn parse_compound_selector(&mut self) -> Result<CompoundSelector, SelectorParseError> {
        let mut simple_selectors = Vec::new();

        loop {
            if let Some(simple) = self.try_parse_simple_selector()? {
                simple_selectors.push(simple);
            } else {
                break;
            }
        }

        if simple_selectors.is_empty() {
            return Err(SelectorParseError::EmptySelector);
        }

        Ok(CompoundSelector { simple_selectors })
    }

    /// Try to parse a simple selector
    fn try_parse_simple_selector(&mut self) -> Result<Option<SimpleSelector>, SelectorParseError> {
        match self.peek_char() {
            Some('.') => {
                self.advance();
                let name = self.parse_identifier()?;
                Ok(Some(SimpleSelector::Class(name)))
            }
            Some('#') => {
                self.advance();
                let name = self.parse_identifier()?;
                Ok(Some(SimpleSelector::Id(name)))
            }
            Some(':') => {
                self.advance();
                let name = self.parse_identifier()?;
                let pseudo = PseudoClass::parse(&name);
                Ok(Some(SimpleSelector::PseudoClass(pseudo)))
            }
            Some('*') => {
                self.advance();
                Ok(Some(SimpleSelector::Universal))
            }
            Some(c) if c.is_alphabetic() || c == '_' => {
                let name = self.parse_identifier()?;
                Ok(Some(SimpleSelector::Type(name)))
            }
            _ => Ok(None),
        }
    }

    fn parse_identifier(&mut self) -> Result<String, SelectorParseError> {
        let start = self.pos;
        while let Some(c) = self.peek_char() {
            if c.is_alphanumeric() || c == '_' || c == '-' {
                self.advance();
            } else {
                break;
            }
        }
        if start == self.pos {
            return Err(SelectorParseError::ExpectedIdentifier);
        }
        Ok(self.input[start..self.pos].to_string())
    }

    // Helper methods...
    fn peek_char(&self) -> Option<char> {
        self.input[self.pos..].chars().next()
    }

    fn peek_char_is(&self, c: char) -> bool {
        self.peek_char() == Some(c)
    }

    fn advance(&mut self) {
        if let Some(c) = self.peek_char() {
            self.pos += c.len_utf8();
        }
    }

    fn consume(&mut self, c: char) -> bool {
        if self.peek_char_is(c) {
            self.advance();
            true
        } else {
            false
        }
    }

    fn skip_whitespace(&mut self) -> bool {
        let start = self.pos;
        while let Some(c) = self.peek_char() {
            if c.is_whitespace() {
                self.advance();
            } else {
                break;
            }
        }
        self.pos > start
    }

    fn is_at_end(&self) -> bool {
        self.pos >= self.input.len()
    }
}

#[derive(Debug, Clone)]
pub enum SelectorParseError {
    EmptySelector,
    ExpectedIdentifier,
    UnexpectedCharacter(char),
    UnexpectedEnd,
}
```

## Selector Matching

```rust
/// Context for selector matching
pub struct MatchContext<'a> {
    /// Path from root to current node
    pub path: &'a [&'a DOMNode],
}

impl Selector {
    /// Check if this selector matches the node in context
    pub fn matches(&self, ctx: &MatchContext) -> bool {
        if ctx.path.is_empty() || self.parts.is_empty() {
            return false;
        }

        // Start matching from rightmost part (the subject)
        let mut part_idx = self.parts.len() - 1;
        let mut node_idx = ctx.path.len() - 1;

        // The rightmost part must match the target node
        if !self.parts[part_idx].compound.matches(ctx.path[node_idx]) {
            return false;
        }

        // If only one part, we're done
        if part_idx == 0 {
            return true;
        }

        // Match remaining parts going up the tree
        part_idx -= 1;
        loop {
            let part = &self.parts[part_idx];
            let combinator = self.parts[part_idx + 1].combinator;

            match combinator {
                Combinator::Child => {
                    // Must match immediate parent
                    if node_idx == 0 {
                        return false;
                    }
                    node_idx -= 1;
                    if !part.compound.matches(ctx.path[node_idx]) {
                        return false;
                    }
                }
                Combinator::Descendant => {
                    // Search up the tree for a match
                    let mut found = false;
                    while node_idx > 0 {
                        node_idx -= 1;
                        if part.compound.matches(ctx.path[node_idx]) {
                            found = true;
                            break;
                        }
                    }
                    if !found {
                        return false;
                    }
                }
                Combinator::None => {
                    // Shouldn't happen for non-rightmost parts
                    unreachable!("Invalid combinator in selector chain")
                }
            }

            if part_idx == 0 {
                return true;
            }
            part_idx -= 1;
        }
    }
}

impl CompoundSelector {
    /// Check if all simple selectors match the node
    pub fn matches(&self, node: &DOMNode) -> bool {
        self.simple_selectors.iter().all(|s| s.matches(node))
    }
}

impl SimpleSelector {
    /// Check if this simple selector matches the node
    pub fn matches(&self, node: &DOMNode) -> bool {
        match self {
            SimpleSelector::Type(name) => {
                node.type_name() == name || node.inherits_from(name)
            }
            SimpleSelector::Class(name) => {
                node.has_class(name)
            }
            SimpleSelector::Id(name) => {
                node.id() == Some(name.as_str())
            }
            SimpleSelector::PseudoClass(pseudo) => {
                node.matches_pseudo_class(pseudo)
            }
            SimpleSelector::Universal => true,
        }
    }
}
```

## DOMNode Interface for Matching

```rust
/// Required interface for selector matching
pub trait SelectorMatchable {
    /// Get the widget type name (e.g., "Button", "Label")
    fn type_name(&self) -> &str;

    /// Check if this node inherits from a type
    fn inherits_from(&self, type_name: &str) -> bool;

    /// Get the node's ID (from id="..." attribute)
    fn id(&self) -> Option<&str>;

    /// Check if node has a specific class
    fn has_class(&self, class: &str) -> bool;

    /// Get all classes on this node
    fn classes(&self) -> &HashSet<String>;

    /// Check if node matches a pseudo-class
    fn matches_pseudo_class(&self, pseudo: &PseudoClass) -> bool;

    /// Get selector names for quick rule filtering
    /// Returns: type name, all classes prefixed with ".", id prefixed with "#"
    fn selector_names(&self) -> HashSet<String>;
}
```

## Rules Map for Fast Lookup

```rust
/// Maps selector names to rules for O(1) rule filtering
pub struct RulesMap {
    /// Maps selector name to rule indices
    map: HashMap<String, Vec<usize>>,
}

impl RulesMap {
    pub fn new() -> Self {
        RulesMap { map: HashMap::new() }
    }

    /// Index a rule by its selector names
    pub fn add_rule(&mut self, index: usize, selector_set: &SelectorSet) {
        for selector in &selector_set.selectors {
            for part in &selector.parts {
                for simple in &part.compound.simple_selectors {
                    let name = match simple {
                        SimpleSelector::Type(n) => n.clone(),
                        SimpleSelector::Class(n) => format!(".{}", n),
                        SimpleSelector::Id(n) => format!("#{}", n),
                        _ => continue,
                    };
                    self.map.entry(name).or_default().push(index);
                }
            }
        }
    }

    /// Get potentially matching rule indices for a node
    pub fn get_candidates(&self, node: &impl SelectorMatchable) -> HashSet<usize> {
        let mut candidates = HashSet::new();
        for name in node.selector_names() {
            if let Some(indices) = self.map.get(&name) {
                candidates.extend(indices.iter().copied());
            }
        }
        candidates
    }
}
```

## Example Usage

```rust
// Parse selector
let mut parser = SelectorParser::new("Button.primary:hover > Label");
let selector_set = parser.parse().unwrap();

// Build path from root to target
let path: Vec<&DOMNode> = vec![&root, &container, &button, &label];
let ctx = MatchContext { path: &path };

// Check if selector matches
for selector in &selector_set.selectors {
    if selector.matches(&ctx) {
        println!("Matched with specificity {:?}", selector.specificity);
    }
}
```
