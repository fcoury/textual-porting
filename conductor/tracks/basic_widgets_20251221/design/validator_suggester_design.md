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
use std::sync::Arc;

/// A single validation failure.
///
/// Matches Python Textual's Failure dataclass exactly:
/// - Stores a reference to the actual validator (via Arc)
/// - Uses validator.describe_failure() if no explicit description
/// - Uses validator.failure_description as fallback
#[derive(Debug, Clone)]
pub struct Failure {
    /// The validator that produced this failure.
    /// In Python: `validator: Validator | None`
    pub validator: Arc<dyn Validator>,
    /// The value that failed validation.
    pub value: Option<String>,
    /// Human-readable description of the failure (explicit override).
    description_override: Option<String>,
}

impl Failure {
    /// Create a failure from a validator instance.
    pub fn new(validator: Arc<dyn Validator>) -> Self {
        Self {
            validator,
            value: None,
            description_override: None,
        }
    }

    pub fn with_value(mut self, value: impl Into<String>) -> Self {
        self.value = Some(value.into());
        self
    }

    pub fn with_description(mut self, description: impl Into<String>) -> Self {
        self.description_override = Some(description.into());
        self
    }

    /// Get the failure description.
    /// Priority: explicit override > validator.failure_description > validator.describe_failure
    pub fn description(&self) -> Option<String> {
        // 1. Explicit override takes precedence
        if let Some(ref desc) = self.description_override {
            return Some(desc.clone());
        }

        // 2. Check validator's static failure_description
        if let Some(desc) = self.validator.failure_description() {
            return Some(desc);
        }

        // 3. Fall back to dynamic describe_failure
        self.validator.describe_failure(self)
    }

    /// Get the validator's name.
    pub fn validator_name(&self) -> &str {
        self.validator.name()
    }
}
```

### Validator Trait

```rust
/// Trait for input validators.
///
/// Validators are used to check if input values meet certain criteria.
/// Multiple validators can be applied to a single Input widget.
///
/// Note: Validators are stored as Arc<dyn Validator> so Failure can reference them.
pub trait Validator: Send + Sync + std::fmt::Debug {
    /// Validate the given value.
    /// Implementations should use `self.make_failure()` to create failures.
    fn validate(self: Arc<Self>, value: &str) -> ValidationResult;

    /// Returns the name of this validator (for failure messages).
    fn name(&self) -> &str;

    /// Static failure description (used if describe_failure returns None).
    fn failure_description(&self) -> Option<String> {
        None
    }

    /// Dynamic failure description based on the specific failure.
    /// Override this for context-aware error messages.
    fn describe_failure(&self, failure: &Failure) -> Option<String> {
        None
    }
}

/// Extension trait for creating failures with self-reference.
pub trait ValidatorExt: Validator + Sized + 'static {
    /// Create a failure that references this validator.
    fn make_failure(self: &Arc<Self>) -> Failure {
        Failure::new(self.clone())
    }
}

impl<T: Validator + 'static> ValidatorExt for T {}
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
    fn validate(self: Arc<Self>, value: &str) -> ValidationResult {
        match value.parse::<i64>() {
            Ok(n) => {
                if let Some(min) = self.minimum {
                    if n < min {
                        return ValidationResult::failure(vec![
                            self.make_failure()
                                .with_value(value)
                                .with_description(format!("Value must be at least {}", min))
                        ]);
                    }
                }
                if let Some(max) = self.maximum {
                    if n > max {
                        return ValidationResult::failure(vec![
                            self.make_failure()
                                .with_value(value)
                                .with_description(format!("Value must be at most {}", max))
                        ]);
                    }
                }
                ValidationResult::success()
            }
            Err(_) => ValidationResult::failure(vec![
                self.make_failure()
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
    fn validate(self: Arc<Self>, value: &str) -> ValidationResult {
        match value.parse::<f64>() {
            Ok(n) => {
                if let Some(min) = self.minimum {
                    if n < min {
                        return ValidationResult::failure(vec![
                            self.make_failure()
                                .with_value(value)
                                .with_description(format!("Value must be at least {}", min))
                        ]);
                    }
                }
                if let Some(max) = self.maximum {
                    if n > max {
                        return ValidationResult::failure(vec![
                            self.make_failure()
                                .with_value(value)
                                .with_description(format!("Value must be at most {}", max))
                        ]);
                    }
                }
                ValidationResult::success()
            }
            Err(_) => ValidationResult::failure(vec![
                self.make_failure()
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
    fn validate(self: Arc<Self>, value: &str) -> ValidationResult {
        let len = value.chars().count();
        let mut failures = Vec::new();

        if let Some(min) = self.minimum {
            if len < min {
                failures.push(
                    self.make_failure()
                        .with_value(value)
                        .with_description(format!("Must be at least {} characters", min))
                );
            }
        }

        if let Some(max) = self.maximum {
            if len > max {
                failures.push(
                    self.make_failure()
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
    fn validate(self: Arc<Self>, value: &str) -> ValidationResult {
        if self.pattern.is_match(value) {
            ValidationResult::success()
        } else {
            ValidationResult::failure(vec![
                self.make_failure()
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
    fn validate(self: Arc<Self>, value: &str) -> ValidationResult {
        // Use url crate for validation
        match url::Url::parse(value) {
            Ok(_) => ValidationResult::success(),
            Err(_) => ValidationResult::failure(vec![
                self.make_failure()
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

### Suggester Base (Python Parity Design)

Python's `Suggester` is an ABC with built-in caching and case normalization in `_get_suggestion`.
In Rust, we model this with a base struct that wraps the custom logic.

```rust
use lru::LruCache;
use std::num::NonZeroUsize;
use std::sync::Mutex;
use async_trait::async_trait;
use unicase::UniCase;  // For Unicode case-insensitive comparison

/// Unicode casefold equivalent for Python parity.
/// Uses the `unicase` crate which implements Unicode case folding.
/// Example: "ß" → "ss", "İ" → "i̇"
fn casefold(s: &str) -> String {
    // UniCase uses Unicode case folding internally
    UniCase::new(s).to_folded_case()
}

/// Trait for custom suggestion logic.
/// Implementors only need to provide the core suggestion logic.
/// Normalization and caching are handled by `Suggester`.
#[async_trait]
pub trait SuggesterImpl: Send + Sync + std::fmt::Debug {
    /// Get a suggestion for the given value.
    ///
    /// Args:
    ///   value: The input value (already normalized if case_sensitive=false)
    ///   case_sensitive: Whether matching should be case-sensitive
    ///
    /// Note: When case_sensitive=false, the value is already lowercased.
    /// Implementations should also lowercase their comparison targets.
    async fn get_suggestion(&self, value: &str, case_sensitive: bool) -> Option<String>;
}

/// Base suggester with built-in caching and normalization.
/// Matches Python Textual's Suggester architecture.
#[derive(Debug)]
pub struct Suggester<T: SuggesterImpl> {
    inner: T,
    /// LRU cache for suggestions (None = caching disabled).
    cache: Option<Mutex<LruCache<String, Option<String>>>>,
    /// Whether matching is case-sensitive.
    /// If false, values are casefolded before calling get_suggestion.
    case_sensitive: bool,
}

impl<T: SuggesterImpl> Suggester<T> {
    /// Create a new Suggester.
    ///
    /// Args:
    ///   impl_: The custom suggestion implementation
    ///   use_cache: Whether to cache suggestions (default: true in Python)
    ///   case_sensitive: If false, normalize values before lookup (default: false in Python)
    pub fn new(impl_: T, use_cache: bool, case_sensitive: bool) -> Self {
        Self {
            inner: impl_,
            cache: if use_cache {
                Some(Mutex::new(LruCache::new(NonZeroUsize::new(1024).unwrap())))
            } else {
                None
            },
            case_sensitive,
        }
    }

    /// Internal method called by widgets to get suggestions.
    /// Handles normalization and caching (matches Python's _get_suggestion).
    pub async fn request_suggestion(&self, requester: WidgetId, value: &str) -> Option<SuggestionReady> {
        // Normalize value if not case sensitive
        // Uses casefold() for Python parity (more aggressive than lowercase for Unicode)
        let normalized_value = if self.case_sensitive {
            value.to_string()
        } else {
            casefold(value)
        };

        // Check cache first
        if let Some(ref cache_mutex) = self.cache {
            let mut cache = cache_mutex.lock().unwrap();
            if let Some(cached) = cache.get(&normalized_value) {
                return cached.as_ref().map(|s| SuggestionReady {
                    value: value.to_string(),
                    suggestion: s.clone(),
                });
            }
        }

        // Get from implementation, passing case_sensitive for comparison logic
        let suggestion = self.inner.get_suggestion(&normalized_value, self.case_sensitive).await;

        // Cache the result
        if let Some(ref cache_mutex) = self.cache {
            let mut cache = cache_mutex.lock().unwrap();
            cache.put(normalized_value, suggestion.clone());
        }

        suggestion.map(|s| SuggestionReady {
            value: value.to_string(),
            suggestion: s,
        })
    }
}
```

### SuggestFromList

```rust
/// A suggester that provides suggestions from a fixed list.
/// Implements only the core logic; caching/normalization handled by Suggester.
#[derive(Debug, Clone)]
pub struct SuggestFromListImpl {
    /// The list of possible suggestions.
    suggestions: Vec<String>,
}

impl SuggestFromListImpl {
    pub fn new(suggestions: impl IntoIterator<Item = impl Into<String>>) -> Self {
        Self {
            suggestions: suggestions.into_iter().map(Into::into).collect(),
        }
    }
}

#[async_trait]
impl SuggesterImpl for SuggestFromListImpl {
    async fn get_suggestion(&self, value: &str, case_sensitive: bool) -> Option<String> {
        if value.is_empty() {
            return None;
        }

        // Value is already normalized by Suggester (casefolded if case_sensitive=false).
        // We need to normalize suggestions the same way for comparison,
        // but return the original (non-normalized) suggestion.
        if case_sensitive {
            // Case-sensitive: direct comparison
            self.suggestions
                .iter()
                .find(|s| s.starts_with(value))
                .cloned()
        } else {
            // Case-insensitive: casefold suggestion for comparison (Python parity)
            self.suggestions
                .iter()
                .find(|s| casefold(s).starts_with(value))
                .cloned()
        }
    }
}

/// Convenience type alias matching Python's SuggestFromList.
pub type SuggestFromList = Suggester<SuggestFromListImpl>;

impl SuggestFromList {
    /// Create a SuggestFromList with Python-compatible defaults.
    /// use_cache=true, case_sensitive=true (Python defaults).
    pub fn from_list(suggestions: impl IntoIterator<Item = impl Into<String>>) -> Self {
        Suggester::new(
            SuggestFromListImpl::new(suggestions),
            true,  // use_cache (Python default)
            true,  // case_sensitive (Python default)
        )
    }

    /// Create with explicit case sensitivity.
    pub fn from_list_with_options(
        suggestions: impl IntoIterator<Item = impl Into<String>>,
        use_cache: bool,
        case_sensitive: bool,
    ) -> Self {
        Suggester::new(
            SuggestFromListImpl::new(suggestions),
            use_cache,
            case_sensitive,
        )
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
    /// Called internally when the input value changes.
    pub async fn request_suggestion(&self) {
        if let Some(ref suggester) = self.suggester {
            // Suggester handles normalization and caching internally
            if let Some(ready) = suggester.request_suggestion(self.id(), &self.value).await {
                self.post_message(ready);
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
