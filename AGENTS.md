# AGENTS.md

## Cursor Cloud specific instructions

### Project overview

Jujutsu (`jj`) is a Rust CLI version control system. It is a single-product workspace with 5 crates: `jj-cli`, `jj-lib`, `gen-protos`, `jj-lib-proc-macros`, and `testutils`. No external services (databases, Docker, etc.) are needed. See `docs/contributing.md` for standard dev setup and commands.

### Key gotcha: `NO_COLOR` environment variable

The Cloud VM environment sets `NO_COLOR=1` by default. This causes crossterm (used by jj for terminal colors) to disable ANSI color output, which breaks ~32 snapshot tests in `jj-cli` (formatter, template_builder, text_util modules). **You must unset `NO_COLOR` when running tests:**

```sh
NO_COLOR= cargo nextest run --workspace
```

### Commands quick reference

- **Build:** `cargo build --workspace`
- **Lint:** `cargo clippy --workspace --all-targets`
- **Format check:** `cargo +nightly fmt --check`
- **Format fix:** `cargo +nightly fmt`
- **Run tests:** `NO_COLOR= cargo nextest run --workspace`
- **Run tests with snapshot review:** `NO_COLOR= cargo insta test --workspace --test-runner nextest`
- **Review snapshots:** `cargo insta review --workspace`
- **Run the CLI:** `cargo run -p jj-cli -- <args>`

### Rust toolchain

- MSRV is **1.85** (set in `Cargo.toml` `rust-version`). The default toolchain must be >= 1.85.
- Nightly toolchain is required for `cargo +nightly fmt` (edition 2024 formatting support).
