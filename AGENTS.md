# AGENTS.md

Project-specific instructions for agents working in this repo.

## Git and submodules

- This is the migration project for Textual's Python codebase to Rust.
- It uses two git submodules:
  - `textual`: The original Python Textual codebase for reference and test porting -- our source. It is pinned to a specific commit for stability.
  - `textual-rs`: The Rust implementation of Textual -- our target. This is where all Rust code changes will occur. When committing features, do so in this submodule.
- When commiting code changes, ensure you are in the `textual-rs` submodule directory.
- When commiting planning or documentation changes, do so in the main repository.

## TUI testing via tmux

- Start a session: `tmux new -d -s codex-shell -n shell`
- Run commands safely: `tmux send-keys -t codex-shell:0.0 -- 'cargo run --example demo' Enter`
- Attach to watch: `tmux attach -t codex-shell`
- Capture output (last 200 lines): `tmux capture-pane -p -J -t codex-shell:0.0 -S -200`
- If a run exceeds ~10 minutes, treat it as potentially hung and inspect via tmux.
- Cleanup: `tmux kill-session -t codex-shell`

## Workflow for porting tests

- Migrate the relevant Python tests into Rust tests first (except the first time).
- Run the tests and observe expected failures.
- Implement or adjust code to make the tests pass.
- Keep tests in the standard `textual-rs/tests` folder.

## Commit workflow (submodule + outer repo)

- Commit Rust changes in the `textual-rs` submodule first, then push that submodule.
- If a phase is completed, update the outer repo to point at the new `textual-rs` commit:
  - From the root repo, stage the submodule pointer update and commit it.
- Use Angular-style commit messages (e.g., `feat: ...`, `chore: ...`) and include a short detail summary in the body.

## Code hygiene

- After each major feature implementation, run `cargo fmt --all` and `cargo clippy`.

## Examples workflow

- Demos live in `textual-rs/examples` and should be runnable via `cargo run --example <name>`.
- Prefer smaller focused examples for new features instead of growing a single monolithic demo.
- At the end of each major new feature, check if we should update the main demo to showcase it.
