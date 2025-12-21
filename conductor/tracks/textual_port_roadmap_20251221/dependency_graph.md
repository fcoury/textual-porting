# Textual Port: Dependency Graph & Execution Order

## Track Summary

| Track | Tier | Status | Dependencies |
|-------|------|--------|--------------|
| Core App Lifecycle | 1 | New | None |
| Layout System | 2 | New | Core App Lifecycle |
| TCSS Styling | 2 | New | Core App Lifecycle |
| Basic Widgets | 3 | New | Core App Lifecycle, Layout System, TCSS Styling |
| Container Widgets | 3 | New | Core App Lifecycle, Layout System, TCSS Styling |
| Advanced Widgets | 4 | New | All Tier 1-3 tracks |

## Dependency Graph

```
                    ┌─────────────────────┐
                    │  Core App Lifecycle │  ◄── Tier 1 (Foundation)
                    │      (Tier 1)       │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                │
    ┌─────────────────┐ ┌─────────────────┐    │
    │  Layout System  │ │  TCSS Styling   │    │  ◄── Tier 2 (Parallel)
    │    (Tier 2)     │ │    (Tier 2)     │    │
    └────────┬────────┘ └────────┬────────┘    │
             │                   │              │
             └─────────┬─────────┘              │
                       │                        │
              ┌────────┴────────┐               │
              │                 │               │
              ▼                 ▼               │
    ┌─────────────────┐ ┌─────────────────┐    │
    │  Basic Widgets  │ │Container Widgets│    │  ◄── Tier 3 (Parallel)
    │    (Tier 3)     │ │    (Tier 3)     │    │
    └────────┬────────┘ └────────┬────────┘    │
             │                   │              │
             └─────────┬─────────┘              │
                       │                        │
                       ▼                        │
            ┌─────────────────────┐             │
            │  Advanced Widgets   │ ◄───────────┘  ◄── Tier 4 (Final)
            │      (Tier 4)       │
            └─────────────────────┘
```

## Recommended Execution Order

### Sequential Dependencies
1. **Core App Lifecycle** (Tier 1) - Must complete first
2. **Layout System + TCSS Styling** (Tier 2) - Can run in parallel
3. **Basic Widgets + Container Widgets** (Tier 3) - Can run in parallel
4. **Advanced Widgets** (Tier 4) - Requires all above

### Parallel Execution Opportunities

| Phase | Tracks | Can Parallelize? |
|-------|--------|------------------|
| 1 | Core App Lifecycle | No (single track) |
| 2 | Layout System, TCSS Styling | **Yes** |
| 3 | Basic Widgets, Container Widgets | **Yes** |
| 4 | Advanced Widgets | No (single track) |

### Estimated Effort Distribution

```
Tier 1: Core App Lifecycle     ████████░░░░░░░░░░░░  ~20%
Tier 2: Layout + TCSS          ████████████░░░░░░░░  ~30%
Tier 3: Basic + Container      ████████████░░░░░░░░  ~30%
Tier 4: Advanced Widgets       ████████░░░░░░░░░░░░  ~20%
```

## Task Counts by Track

| Track | Phase 1 | Phase 2 | Phase 3 | Phase 4 | Total |
|-------|---------|---------|---------|---------|-------|
| Core App Lifecycle | 7 | 8 | ~35 | 5 | ~55 |
| Layout System | 6 | 6 | ~40 | 5 | ~57 |
| TCSS Styling | 7 | 6 | ~45 | 6 | ~64 |
| Basic Widgets | 10 | 7 | ~85 | 5 | ~107 |
| Container Widgets | 7 | 6 | ~55 | 5 | ~73 |
| Advanced Widgets | 11 | 8 | ~95 | 8 | ~122 |
| **Total** | | | | | **~478** |

## Critical Path

The critical path (longest sequential dependency chain) is:

```
Core App Lifecycle → Layout System → Basic Widgets → Advanced Widgets
        OR
Core App Lifecycle → TCSS Styling → Basic Widgets → Advanced Widgets
```

To minimize total time, prioritize:
1. Complete Core App Lifecycle quickly
2. Parallelize Layout System and TCSS Styling
3. Parallelize Basic Widgets and Container Widgets
4. Advanced Widgets can begin once Tier 3 completes

## Rebuild Notes

The following existing textual-rs widgets must be rebuilt in the **Basic Widgets** track:
- Button (add variants, action support, DEFAULT_CSS)
- Input (extend ScrollView, add validation, selection, suggester)
- Label (extend Static, add variants)
- Header (add clock, screen title integration)
- Footer (add binding collection, compact mode)

## Success Metrics

When all tracks are complete:
- [ ] All Python Textual widgets have Rust equivalents
- [ ] API signatures match Python Textual
- [ ] TCSS styling works with all widgets
- [ ] compose() pattern works as expected
- [ ] Message passing matches Python Textual semantics
- [ ] Performance meets or exceeds Python version
