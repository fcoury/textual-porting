# Technology Stack

## Core Technologies
- **Language:** Rust
- **Source/Reference:** Python (Textual)

## Core Libraries & Frameworks
- **TUI Rendering Engine:** `ratatui` (backend-agnostic rendering foundation)
- **Terminal Backend:** `crossterm` (cross-platform terminal handling)
- **Async Runtime:** `tokio` (asynchronous I/O and task scheduling)
- **Data Structures:** `slotmap` (for the ID-based widget registry)
- **Parsing:** `nom` or `pest` (for TCSS implementation)

## Utilities & Observability
- **Error Handling:** `thiserror` (ergonomic library error types)
- **Logging/Diagnostics:** `tracing` (structured diagnostics)
- **Testing:** `insta` (snapshot testing), `criterion` (benchmarking)

## Text Processing
- **Unicode Width:** `unicode-width` (correct cell-width measurement for box-drawing characters, emoji, CJK)
  - *Transitive dependency via ratatui* - no additional download required
  - Required for content-align, line-pad, and text layout parity with Python Textual
  - Access via direct dependency or ratatui re-export when implementing render-time alignment

## Build & Package Management
- **Build System:** Cargo
- **Package Manager:** Cargo (crates.io)

## Project Structure
- **Monorepo:** Contains `textual` (Python reference) and `textual-rs` (Rust port)
