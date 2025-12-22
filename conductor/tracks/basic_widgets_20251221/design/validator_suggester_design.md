# Validator and Suggester Traits Design

## Overview

This document defines the validation and suggestion systems for the Input widget, matching Python Textual's extensible design.

## Validation System

### ValidationResult

```rust
/// Result of validating an input value.
#[derive(Debug, Clone, Default)]
pub struct ValidationResult {
    /// List of validation failures (empty = valid).
    pub failures: Vec<Failure>,
}

impl ValidationResult {
    /// Create a successful validation result.
    pub fn success() -> Self {
        Self { failures: vec![] }
    }

    /// Create a failed validation result.
    pub fn failure(failures: Vec<Failure>) -> Self {
        Self { failures }
    }

    /// Check if the validation passed.
    pub fn is_valid(&self) -> bool {
        self.failures.is_empty()
    }

    /// Get descriptions of all failures.
    pub fn failure_descriptions(&self) -> Vec<&str> {
        self.failures
            .iter()
            .filter_map(|f| f.description.as_deref())
            .collect()
    }

    /// Merge multiple validation results into one.
    pub fn merge(results: impl IntoIterator<Item = ValidationResult>) -> Self {
        let failures: Vec<Failure> = results
            .into_iter()
            .flat_map(|r| r.failures)
            .collect();
        Self { failures }
    }
}
```

### Failure

```rust
/// A single validation failure.
///
/// Note: Python Textual's Failure stores a reference to the actual validator.
/// In Rust, we store the validator as a trait object to match this behavior.
#[derive(Debug, Clone)]
pub struct Failure {
    /// The validator that produced this failure (as trait object name for display).
    /// In Python, this is `validator: Validator | None`. We use the type name.
    pub validator_name: String,
    /// The value that failed validation.
    pub value: Option<String>,
    /// Human-readable description of the failure.
    pub description: Option<String>,
}

impl Failure {
    /// Create a failure from a validator instance.
    /// Extracts the validator's name for the failure record.
    pub fn from_validator<V: Validator + ?Sized>(validator: &V) -> Self {
        Self {
            validator_name: validator.name().to_string(),
            value: None,
            description: None,
        }
    }

    /// Create a failure with just a name (for cases where validator ref isn't available).
    pub fn new(validator_name: impl Into<String>) -> Self {
        Self {
            validator_name: validator_name.into(),
            value: None,
            description: None,
        }
    }

    pub fn with_value(mut self, value: impl Into<String>) -> Self {
        self.value = Some(value.into());
        self
    }

    pub fn with_description(mut self, description: impl Into<String>) -> Self {
        self.description = Some(description.into());
        self
    }
}
```

### Validator Trait

```rust
/// Trait for input validators.
///
/// Validators are used to check if input values meet certain criteria.
/// Multiple validators can be applied to a single Input widget.
pub trait Validator: Send + Sync + std::fmt::Debug {
    /// Validate the given value.
    fn validate(&self, value: &str) -> ValidationResult;

    /// Returns the name of this validator (for failure messages).
    fn name(&self) -> &str;

    /// Returns a description of what this validator checks.
    fn describe(&self) -> Option<String> {
        None
    }
}
```

### Built-in Validators

```rust
/// Validates that the value is a valid integer.
#[derive(Debug, Clone, Default)]
pub struct Integer {
    /// Minimum allowed value (inclusive).
    pub minimum: Option<i64>,
    /// Maximum allowed value (inclusive).
    pub maximum: Option<i64>,
}

impl Integer {
    pub fn new() -> Self {
        Self::default()
    }

    pub fn with_minimum(mut self, min: i64) -> Self {
        self.minimum = Some(min);
        self
    }

    pub fn with_maximum(mut self, max: i64) -> Self {
        self.maximum = Some(max);
        self
    }
}

impl Validator for Integer {
    fn validate(&self, value: &str) -> ValidationResult {
        match value.parse::<i64>() {
            Ok(n) => {
                if let Some(min) = self.minimum {
                    if n < min {
                        return ValidationResult::failure(vec![
                            Failure::from_validator(self)
                                .with_value(value)
                                .with_description(format!("Value must be at least {}", min))
                        ]);
                    }
                }
                if let Some(max) = self.maximum {
                    if n > max {
                        return ValidationResult::failure(vec![
                            Failure::from_validator(self)
                                .with_value(value)
                                .with_description(format!("Value must be at most {}", max))
                        ]);
                    }
                }
                ValidationResult::success()
            }
            Err(_) => ValidationResult::failure(vec![
                Failure::from_validator(self)
                    .with_value(value)
                    .with_description("Must be a valid integer")
            ]),
        }
    }

    fn name(&self) -> &str {
        "Integer"
    }
}

/// Validates that the value is a valid number (integer or float).
#[derive(Debug, Clone, Default)]
pub struct Number {
    pub minimum: Option<f64>,
    pub maximum: Option<f64>,
}

impl Validator for Number {
    fn validate(&self, value: &str) -> ValidationResult {
        match value.parse::<f64>() {
            Ok(n) => {
                if let Some(min) = self.minimum {
                    if n < min {
                        return ValidationResult::failure(vec![
                            Failure::from_validator(self)
                                .with_value(value)
                                .with_description(format!("Value must be at least {}", min))
                        ]);
                    }
                }
                if let Some(max) = self.maximum {
                    if n > max {
                        return ValidationResult::failure(vec![
                            Failure::from_validator(self)
                                .with_value(value)
                                .with_description(format!("Value must be at most {}", max))
                        ]);
                    }
                }
                ValidationResult::success()
            }
            Err(_) => ValidationResult::failure(vec![
                Failure::from_validator(self)
                    .with_value(value)
                    .with_description("Must be a valid number")
            ]),
        }
    }

    fn name(&self) -> &str {
        "Number"
    }
}

/// Validates string length.
#[derive(Debug, Clone)]
pub struct Length {
    /// Minimum length (inclusive).
    pub minimum: Option<usize>,
    /// Maximum length (inclusive).
    pub maximum: Option<usize>,
}

impl Validator for Length {
    fn validate(&self, value: &str) -> ValidationResult {
        let len = value.chars().count();
        let mut failures = Vec::new();

        if let Some(min) = self.minimum {
            if len < min {
                failures.push(
                    Failure::from_validator(self)
                        .with_value(value)
                        .with_description(format!("Must be at least {} characters", min))
                );
            }
        }

        if let Some(max) = self.maximum {
            if len > max {
                failures.push(
                    Failure::from_validator(self)
                        .with_value(value)
                        .with_description(format!("Must be at most {} characters", max))
                );
            }
        }

        if failures.is_empty() {
            ValidationResult::success()
        } else {
            ValidationResult::failure(failures)
        }
    }

    fn name(&self) -> &str {
        "Length"
    }
}

/// Validates against a regex pattern.
#[derive(Debug, Clone)]
pub struct Regex {
    pattern: regex::Regex,
    description: String,
}

impl Regex {
    pub fn new(pattern: &str, description: impl Into<String>) -> Result<Self, regex::Error> {
        Ok(Self {
            pattern: regex::Regex::new(pattern)?,
            description: description.into(),
        })
    }
}

impl Validator for Regex {
    fn validate(&self, value: &str) -> ValidationResult {
        if self.pattern.is_match(value) {
            ValidationResult::success()
        } else {
            ValidationResult::failure(vec![
                Failure::from_validator(self)
                    .with_value(value)
                    .with_description(&self.description)
            ])
        }
    }

    fn name(&self) -> &str {
        "Regex"
    }
}

/// Validates URL format.
#[derive(Debug, Clone, Default)]
pub struct Url;

impl Validator for Url {
    fn validate(&self, value: &str) -> ValidationResult {
        // Use url crate for validation
        match url::Url::parse(value) {
            Ok(_) => ValidationResult::success(),
            Err(_) => ValidationResult::failure(vec![
                Failure::from_validator(self)
                    .with_value(value)
                    .with_description("Must be a valid URL")
            ]),
        }
    }

    fn name(&self) -> &str {
        "Url"
    }
}
```

### Auto-Validator Behavior

```rust
impl Input {
    /// Apply auto-validators based on input type.
    /// Called during construction if no validators are provided.
    fn apply_auto_validators(&mut self) {
        if self.validators.is_empty() {
            match self.input_type {
                InputType::Integer => {
                    self.validators.push(Box::new(Integer::new()));
                }
                InputType::Number => {
                    self.validators.push(Box::new(Number::new()));
                }
                InputType::Text => {
                    // No auto-validator for text
                }
            }
        }
    }
}
```

### Validation Triggers

```rust
/// When validation should be triggered.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum ValidationTrigger {
    /// Validate when input loses focus.
    Blur,
    /// Validate on every change.
    Changed,
    /// Validate on submit (Enter).
    Submitted,
}

impl Input {
    /// Default: validate on all triggers.
    pub const DEFAULT_VALIDATE_ON: &'static [ValidationTrigger] = &[
        ValidationTrigger::Blur,
        ValidationTrigger::Changed,
        ValidationTrigger::Submitted,
    ];
}
```

## Suggestion System

### Suggester Trait

```rust
use async_trait::async_trait;

/// Trait for input suggesters (autocomplete providers).
///
/// Suggesters provide completion suggestions based on the current input value.
/// They are async to support network-based suggestions.
#[async_trait]
pub trait Suggester: Send + Sync + std::fmt::Debug {
    /// Get a suggestion for the given value.
    ///
    /// Returns `None` if no suggestion is available.
    async fn get_suggestion(&self, value: &str) -> Option<String>;

    /// Whether the suggester is case-sensitive.
    fn case_sensitive(&self) -> bool {
        false
    }
}
```

### SuggestFromList

```rust
/// A suggester that provides suggestions from a fixed list.
///
/// Note: Python Textual defaults `case_sensitive=True`.
#[derive(Debug, Clone)]
pub struct SuggestFromList {
    /// The list of possible suggestions.
    suggestions: Vec<String>,
    /// Whether matching is case-sensitive (default: true, matches Python).
    case_sensitive: bool,
}

impl SuggestFromList {
    pub fn new(suggestions: impl IntoIterator<Item = impl Into<String>>) -> Self {
        Self {
            suggestions: suggestions.into_iter().map(Into::into).collect(),
            case_sensitive: true,  // Python default is True
        }
    }

    pub fn case_sensitive(mut self, case_sensitive: bool) -> Self {
        self.case_sensitive = case_sensitive;
        self
    }
}

#[async_trait]
impl Suggester for SuggestFromList {
    async fn get_suggestion(&self, value: &str) -> Option<String> {
        if value.is_empty() {
            return None;
        }

        let compare = |s: &str, prefix: &str| {
            if self.case_sensitive {
                s.starts_with(prefix)
            } else {
                s.to_lowercase().starts_with(&prefix.to_lowercase())
            }
        };

        self.suggestions
            .iter()
            .find(|s| compare(s, value))
            .cloned()
    }

    fn case_sensitive(&self) -> bool {
        self.case_sensitive
    }
}
```

### Suggestion Caching

```rust
use lru::LruCache;
use std::num::NonZeroUsize;

/// A cached suggester wrapper.
#[derive(Debug)]
pub struct CachedSuggester<S: Suggester> {
    inner: S,
    cache: std::sync::Mutex<LruCache<String, Option<String>>>,
}

impl<S: Suggester> CachedSuggester<S> {
    pub fn new(suggester: S, cache_size: usize) -> Self {
        Self {
            inner: suggester,
            cache: std::sync::Mutex::new(
                LruCache::new(NonZeroUsize::new(cache_size).unwrap())
            ),
        }
    }
}

#[async_trait]
impl<S: Suggester> Suggester for CachedSuggester<S> {
    async fn get_suggestion(&self, value: &str) -> Option<String> {
        // Check cache first
        {
            let mut cache = self.cache.lock().unwrap();
            if let Some(cached) = cache.get(value) {
                return cached.clone();
            }
        }

        // Get from inner suggester
        let result = self.inner.get_suggestion(value).await;

        // Cache the result
        {
            let mut cache = self.cache.lock().unwrap();
            cache.put(value.to_string(), result.clone());
        }

        result
    }

    fn case_sensitive(&self) -> bool {
        self.inner.case_sensitive()
    }
}
```

### SuggestionReady Message

```rust
/// Message sent when a suggestion is ready for display.
#[derive(Debug, Clone)]
pub struct SuggestionReady {
    /// The input value the suggestion is for.
    pub value: String,
    /// The suggested completion.
    pub suggestion: String,
}

impl Message for SuggestionReady {}
```

## Input Integration

```rust
impl Input {
    /// Run validation on the current value.
    pub fn validate(&self) -> ValidationResult {
        if self.value.is_empty() && self.valid_empty {
            return ValidationResult::success();
        }

        let results: Vec<_> = self.validators
            .iter()
            .map(|v| v.validate(&self.value))
            .collect();

        ValidationResult::merge(results)
    }

    /// Check if current value passes validation.
    pub fn is_valid(&self) -> bool {
        self.validate().is_valid()
    }

    /// Request a suggestion for the current value.
    pub async fn request_suggestion(&self) {
        if let Some(ref suggester) = self.suggester {
            if let Some(suggestion) = suggester.get_suggestion(&self.value).await {
                self.post_message(SuggestionReady {
                    value: self.value.clone(),
                    suggestion,
                });
            }
        }
    }
}
```

## Testing Strategy

### Validator Tests
1. Integer validator with valid/invalid inputs
2. Number validator with decimals, scientific notation
3. Length validator with min/max bounds
4. Regex validator with custom patterns
5. URL validator with various URL formats
6. ValidationResult merging

### Suggester Tests
1. SuggestFromList basic matching
2. Case-sensitive vs insensitive matching
3. Cached suggester hit/miss
4. Async suggestion retrieval
5. Empty value handling

### Integration Tests
1. Auto-validator application based on InputType
2. Validation on blur/changed/submitted triggers
3. Suggestion display and acceptance
