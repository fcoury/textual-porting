# Style Cache Design

## Overview

The style cache stores computed styles to avoid recomputation on every frame. It uses a multi-factor invalidation strategy to ensure correctness while maximizing cache hits.

## Cache Structure

```rust
/// Style cache for computed widget styles.
pub struct StyleCache {
    /// Cached styles by widget ID.
    entries: HashMap<WidgetId, StyleCacheEntry>,

    /// Cache statistics for debugging/optimization.
    stats: CacheStats,
}

/// A single cache entry.
pub struct StyleCacheEntry {
    /// The computed style.
    computed: ComputedStyle,

    /// Hash of the ancestor chain at computation time.
    /// Used to detect when ancestors have changed.
    ancestor_hash: u64,

    /// Theme version at computation time.
    /// Used to invalidate on theme switch or hot reload.
    theme_version: u64,

    /// Hash of the widget's own styling inputs.
    /// (type_name, id, classes, pseudo_classes)
    widget_hash: u64,
}

/// Cache statistics for monitoring.
#[derive(Default)]
pub struct CacheStats {
    /// Number of cache hits.
    pub hits: u64,
    /// Number of cache misses.
    pub misses: u64,
    /// Number of invalidations.
    pub invalidations: u64,
}
```

## Cache Key

The primary cache key is `WidgetId`. This assumes:
1. Each widget has a unique, stable ID
2. The same widget instance always has the same ID
3. IDs are not reused for different widgets

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq, Hash)]
pub struct WidgetId(pub usize);
```

## Cache Validation

A cached entry is valid if ALL of the following match:
1. **Theme version** matches current `style_manager.theme_version()`
2. **Ancestor hash** matches current ancestor chain
3. **Widget hash** matches current widget state

### Widget Hash Computation

```rust
fn compute_widget_hash(widget: &WidgetMeta) -> u64 {
    use std::hash::{Hash, Hasher};
    use std::collections::hash_map::DefaultHasher;

    let mut hasher = DefaultHasher::new();

    // Type name
    widget.type_name.hash(&mut hasher);

    // ID
    widget.id.hash(&mut hasher);

    // Classes (sorted for determinism)
    let mut classes: Vec<_> = widget.classes.iter().collect();
    classes.sort();
    for class in classes {
        class.hash(&mut hasher);
    }

    // Pseudo-classes (sorted for determinism)
    let mut pseudo: Vec<_> = widget.pseudo_classes.iter().collect();
    pseudo.sort();
    for pc in pseudo {
        pc.hash(&mut hasher);
    }

    hasher.finish()
}
```

### Ancestor Hash Computation

```rust
fn compute_ancestor_hash(ancestors: &[&WidgetMeta]) -> u64 {
    use std::hash::{Hash, Hasher};
    use std::collections::hash_map::DefaultHasher;

    let mut hasher = DefaultHasher::new();

    // Order matters: root → parent
    for ancestor in ancestors {
        // Hash each ancestor's styling-relevant fields
        ancestor.type_name.hash(&mut hasher);
        ancestor.id.hash(&mut hasher);

        // Sort for determinism
        let mut classes: Vec<_> = ancestor.classes.iter().collect();
        classes.sort();
        for class in classes {
            class.hash(&mut hasher);
        }

        let mut pseudo: Vec<_> = ancestor.pseudo_classes.iter().collect();
        pseudo.sort();
        for pc in pseudo {
            pc.hash(&mut hasher);
        }
    }

    hasher.finish()
}
```

## Cache Operations

### Get (with validation)

```rust
impl StyleCache {
    /// Get a cached style if valid, otherwise return None.
    pub fn get(
        &mut self,
        widget_id: WidgetId,
        widget: &WidgetMeta,
        ancestors: &[&WidgetMeta],
        theme_version: u64,
    ) -> Option<&ComputedStyle> {
        let entry = self.entries.get(&widget_id)?;

        // Validate entry
        let valid = entry.theme_version == theme_version
            && entry.widget_hash == compute_widget_hash(widget)
            && entry.ancestor_hash == compute_ancestor_hash(ancestors);

        if valid {
            self.stats.hits += 1;
            Some(&entry.computed)
        } else {
            // Entry exists but is stale
            self.stats.misses += 1;
            None
        }
    }
}
```

### Insert

```rust
impl StyleCache {
    /// Insert a computed style into the cache.
    pub fn insert(
        &mut self,
        widget_id: WidgetId,
        widget: &WidgetMeta,
        ancestors: &[&WidgetMeta],
        theme_version: u64,
        computed: ComputedStyle,
    ) {
        let entry = StyleCacheEntry {
            computed,
            ancestor_hash: compute_ancestor_hash(ancestors),
            theme_version,
            widget_hash: compute_widget_hash(widget),
        };

        self.entries.insert(widget_id, entry);
    }
}
```

### Invalidation

```rust
impl StyleCache {
    /// Invalidate a single widget's cached style.
    pub fn invalidate(&mut self, widget_id: WidgetId) {
        if self.entries.remove(&widget_id).is_some() {
            self.stats.invalidations += 1;
        }
    }

    /// Invalidate all cached styles.
    ///
    /// Called on:
    /// - Theme switch
    /// - Hot reload
    /// - Global state change
    pub fn invalidate_all(&mut self) {
        let count = self.entries.len();
        self.entries.clear();
        self.stats.invalidations += count as u64;
    }

    /// Remove entries for unmounted widgets.
    pub fn remove(&mut self, widget_id: WidgetId) {
        self.entries.remove(&widget_id);
    }
}
```

## Invalidation Triggers

### Trigger Matrix

| Event | Scope | Method | Notes |
|-------|-------|--------|-------|
| Class added/removed | Single widget | `invalidate(widget_id)` | Widget hash changes |
| ID changed | Single widget | `invalidate(widget_id)` | Widget hash changes |
| Pseudo-class change | Single widget | `invalidate(widget_id)` | Widget hash changes |
| Focus change | Single widget | `invalidate(widget_id)` | :focus pseudo-class |
| Hover change | Single widget | `invalidate(widget_id)` | :hover pseudo-class |
| Disabled change | Single widget | `invalidate(widget_id)` | :disabled pseudo-class |
| Theme switch | All widgets | `invalidate_all()` | Theme version incremented |
| Hot reload | All widgets | `invalidate_all()` | Theme version incremented |
| Widget mount | N/A | Compute on first access | No cache entry exists |
| Widget unmount | Single widget | `remove(widget_id)` | Cleanup |
| Ancestor class change | Descendants | *See below* | Ancestor hash changes |

### Ancestor Change Handling

When an ancestor's styling-relevant state changes, descendants' cached styles become invalid because their `ancestor_hash` no longer matches.

**Lazy invalidation approach** (recommended):
- Don't explicitly invalidate descendants
- Let `get()` detect stale entries via `ancestor_hash` mismatch
- Stale entries are replaced on next access

This is simpler and more efficient than walking the widget tree to find descendants.

**Explicit invalidation approach** (alternative):
```rust
impl StyleCache {
    /// Invalidate all descendants of a widget.
    /// Requires widget tree traversal.
    pub fn invalidate_descendants(
        &mut self,
        widget_id: WidgetId,
        registry: &WidgetRegistry,
    ) {
        // Walk tree and invalidate each descendant
        for descendant_id in registry.descendants(widget_id) {
            self.invalidate(descendant_id);
        }
    }
}
```

## Integration with StyleManager

```rust
impl StyleManager {
    pub fn get_style(
        &mut self,
        widget_id: WidgetId,
        widget: &WidgetMeta,
        ancestors: &[&WidgetMeta],
        parent_background: &Color,
    ) -> ComputedStyle {
        // Try cache first
        if let Some(cached) = self.cache.get(
            widget_id,
            widget,
            ancestors,
            self.theme_version,
        ) {
            // Apply animated values on top of cached style
            return self.apply_animations(widget_id, cached.clone());
        }

        // Cache miss: compute fresh
        let style = compute_style_resolved(
            widget,
            ancestors,
            &self.stylesheet,
            parent_background,
        );

        // Store in cache
        self.cache.insert(
            widget_id,
            widget,
            ancestors,
            self.theme_version,
            style.clone(),
        );

        // Apply animations
        self.apply_animations(widget_id, style)
    }
}
```

## Memory Considerations

### Cache Size Limits

For applications with many widgets, consider a size limit:

```rust
impl StyleCache {
    /// Maximum number of cached entries.
    const MAX_ENTRIES: usize = 10_000;

    pub fn insert(&mut self, /* ... */) {
        // Evict if at capacity
        if self.entries.len() >= Self::MAX_ENTRIES {
            self.evict_oldest();
        }

        // ... insert logic ...
    }

    fn evict_oldest(&mut self) {
        // Simple strategy: clear half the cache
        // More sophisticated: LRU eviction
        let to_remove: Vec<_> = self.entries.keys()
            .take(self.entries.len() / 2)
            .cloned()
            .collect();

        for id in to_remove {
            self.entries.remove(&id);
        }
    }
}
```

### Entry Size

Each `StyleCacheEntry` contains:
- `ComputedStyle`: ~200-400 bytes (many Option<T> fields)
- `ancestor_hash`: 8 bytes
- `theme_version`: 8 bytes
- `widget_hash`: 8 bytes

Total: ~250-450 bytes per widget.

For 1,000 widgets: ~250-450 KB cache overhead.

## Performance Analysis

### Cache Hit Path

1. HashMap lookup: O(1)
2. Hash comparisons: 3 × O(1)
3. Return reference: O(1)

**Total: O(1)** - very fast

### Cache Miss Path

1. HashMap lookup: O(1)
2. `compute_style_resolved()`: O(R × S) where R = rules, S = selectors
3. HashMap insert: O(1)
4. Animation overlay: O(A) where A = active animations

**Total: O(R × S)** - dominated by style computation

### Optimization: Selector Matching Cache

For very large stylesheets, cache selector matching results:

```rust
pub struct SelectorMatchCache {
    /// (widget_hash, selector_hash) → matches
    matches: HashMap<(u64, u64), bool>,
}
```

This is a future optimization if profiling shows selector matching as a bottleneck.

## Testing Strategy

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn test_cache_hit() {
        let mut cache = StyleCache::new();
        let meta = WidgetMeta::new("Button");
        let ancestors: &[&WidgetMeta] = &[];

        cache.insert(WidgetId(1), &meta, ancestors, 1, ComputedStyle::default());

        assert!(cache.get(WidgetId(1), &meta, ancestors, 1).is_some());
        assert_eq!(cache.stats.hits, 1);
    }

    #[test]
    fn test_cache_miss_wrong_theme_version() {
        let mut cache = StyleCache::new();
        let meta = WidgetMeta::new("Button");
        let ancestors: &[&WidgetMeta] = &[];

        cache.insert(WidgetId(1), &meta, ancestors, 1, ComputedStyle::default());

        // Different theme version
        assert!(cache.get(WidgetId(1), &meta, ancestors, 2).is_none());
        assert_eq!(cache.stats.misses, 1);
    }

    #[test]
    fn test_cache_miss_widget_changed() {
        let mut cache = StyleCache::new();
        let meta1 = WidgetMeta::new("Button");
        let meta2 = WidgetMeta::new("Button").with_class("primary");
        let ancestors: &[&WidgetMeta] = &[];

        cache.insert(WidgetId(1), &meta1, ancestors, 1, ComputedStyle::default());

        // Widget has new class
        assert!(cache.get(WidgetId(1), &meta2, ancestors, 1).is_none());
    }

    #[test]
    fn test_cache_miss_ancestor_changed() {
        let mut cache = StyleCache::new();
        let meta = WidgetMeta::new("Button");
        let parent1 = WidgetMeta::new("Container");
        let parent2 = WidgetMeta::new("Container").with_class("dark");

        cache.insert(WidgetId(1), &meta, &[&parent1], 1, ComputedStyle::default());

        // Ancestor has new class
        assert!(cache.get(WidgetId(1), &meta, &[&parent2], 1).is_none());
    }

    #[test]
    fn test_invalidate_all() {
        let mut cache = StyleCache::new();
        let meta = WidgetMeta::new("Button");
        let ancestors: &[&WidgetMeta] = &[];

        cache.insert(WidgetId(1), &meta, ancestors, 1, ComputedStyle::default());
        cache.insert(WidgetId(2), &meta, ancestors, 1, ComputedStyle::default());

        cache.invalidate_all();

        assert!(cache.get(WidgetId(1), &meta, ancestors, 1).is_none());
        assert!(cache.get(WidgetId(2), &meta, ancestors, 1).is_none());
    }
}
```
