# Product Guidelines: textual-rs

## Architectural Patterns
- **State Management:** Message-Based (The Elm Architecture). State transitions are managed through a centralized `update` loop processing owned messages, mirroring Textual's event handlers while strictly adhering to Rust's ownership model.
- **Component Tree:** ID-based Registry. Widgets are managed in a centralized registry (e.g., using `SlotMap`) and referenced by unique IDs to avoid lifetime complexity and reference cycles.
- **Rendering Foundation:** Build on top of `Ratatui`. Use Ratatui for low-level buffer management, double-buffering, and cross-platform rendering (via `crossterm`), while implementing Textual's retained-mode widget model (lifecycle, reactives, CSS) on top.

## Development Standards
- **Error Handling:** Result-Centric. Prefer `Result<T, E>` for all fallible operations. Library code must avoid `panic!` to ensure host application stability. Errors should bubble up to the caller or the main message loop.
- **Styling Strategy:** Hybrid TCSS & DSL. Prioritize parity with Textual's `.tcss` file format for selectors, cascading, and hot-reloading, while providing an optional Rust DSL for type-safe programmatic styling.
- **Performance Guidelines:** Prioritize rendering throughput and low-latency input processing. Leverage Rust's zero-cost abstractions; avoid unnecessary allocations in the hot path. Safety is non-negotiable—use safe Rust unless profiling proves otherwise.

## Technical Strategy
- **Async Runtime:** `tokio` (Industry standard, robust ecosystem).
- **MSRV:** Minimum Supported Rust Version 1.75+ (to leverage recent async traits and const generics improvements).
- **Feature Flags:** Modular architecture using Cargo features (e.g., `tcss`, `hot-reload`, `image-support`).
- **Testing:** Heavy reliance on Snapshot Testing (using `insta`) for rendering verification and unit tests for logic.
- **API Stability:** Pre-1.0 SemVer. Rapid iteration expected; breaking changes will be documented in changelogs.
