# Specification: Project Initialization

## Goal
Establish the foundational Rust project structure for `textual-rs` and verify the rendering pipeline with a minimal "Hello World" application.

## Requirements
- New Cargo crate named `textual-rs`.
- Integration of `ratatui`, `crossterm`, and `tokio`.
- Safe terminal initialization (alt screen, raw mode) and cleanup (panic handling).
- A functional event loop that renders a "Hello World" message and exits on 'q'.

## Success Criteria
- `cargo build` and `cargo test` pass.
- Running the application shows "Hello World" in the center of the terminal.
- Pressing 'q' restores the terminal state perfectly.
