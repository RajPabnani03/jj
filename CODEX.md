# Codex README — `jj` (Jujutsu) development guide

This repository contains **Jujutsu (`jj`)**, an experimental, Git-compatible version control system written in Rust.

This file is intended as a **single, practical starting point** for contributors and tooling (including AI agents) to understand the repo layout and run the common dev workflows.

## What’s in this repo

- **`cli/`**: the `jj` command-line application (**crate:** `jj-cli`)
  - Entry point: `cli/src/main.rs` (initializes `CliRunner`)
  - Most CLI commands live under `cli/src/commands/`
  - CLI end-to-end tests live under `cli/tests/` (see “Testing”)
- **`lib/`**: core library (**crate:** `jj-lib`)
  - Designed to be UI-agnostic (CLI is the UI layer)
  - Architecture overview: `docs/technical/architecture.md`
- **`lib/gen-protos/`**: protobuf code generator (**crate:** `gen-protos`)
- **`lib/proc-macros/`**: proc-macro helpers (**crate:** `jj-lib-proc-macros`)
- **`lib/testutils/`**: shared test helpers (**crate:** `testutils`)
- **`docs/`**: MkDocs documentation site sources
- **`.github/`**: CI workflows and helper scripts
- **`flake.nix` / `flake.lock` / `.envrc.recommended`**: Nix + direnv dev environment support

## Rust toolchain + repo standards (important)

- **Workspace MSRV**: Rust **1.85** (see root `Cargo.toml`)
- **Edition**: **2024**
- **Formatting**: CI checks **nightly rustfmt** (`cargo +nightly fmt --all -- --check`)
- **Clippy**: CI runs `cargo +stable clippy --all-features --workspace --all-targets -- -D warnings`
- **Docs**: built with **MkDocs** via **uv** (CI uses uv **0.5.1** and Python 3.11)

## Cargo features and build variants

- **Default features**:
  - `jj-cli`: `git`, `watchman`
  - `jj-lib`: `git`
- **CI “no git” build**: ensures the CLI can build without default features:

```bash
cargo build -p jj-cli --no-default-features
```

- **CI config overrides**: CI passes `--config .cargo/config-ci.toml` (mold on Linux, rust-lld + static CRT on Windows). You can opt into the same locally:

```bash
cargo build --config .cargo/config-ci.toml --workspace --all-targets
```

## Quickstart (local dev)

### Build

```bash
cargo build --workspace --all-targets
```

### Run `jj` from source

```bash
cargo run -p jj-cli --bin jj -- --help
```

### Format (matches CI)

```bash
cargo +nightly fmt --all
```

### Lint (matches CI)

```bash
cargo +stable clippy --all-features --workspace --all-targets -- -D warnings
```

### Test (fast path + CI-like)

```bash
cargo nextest run --workspace --all-targets --profile ci
```

If you don’t have nextest installed:

```bash
cargo test --workspace
```

### Doctests + rustdoc warnings (CI checks these separately)

```bash
cargo test --workspace --doc
RUSTDOCFLAGS="--deny warnings" cargo doc --workspace --no-deps
```

## Where to start when making changes

- **CLI behavior / new subcommands**: `cli/src/commands/`
- **CLI wiring (arg parsing, global options, runner setup)**: `cli/src/cli_util/` and `cli/src/main.rs`
- **Core algorithms / data model / storage / revsets**: `lib/src/` (overview in `docs/technical/architecture.md`)
- **User-facing docs**: `docs/` (site config in `mkdocs.yml`)

## Testing notes (read this before adding tests)

### Snapshot testing (CLI)

Many CLI changes require updating **`insta`** snapshots.

Common commands:

```bash
cargo insta test --workspace
cargo insta review --workspace
```

To run insta snapshots with nextest:

```bash
cargo insta test --workspace --test-runner nextest
```

### `cli/tests` is “manual” (autotests disabled)

The `jj-cli` crate sets `autotests = false`, and explicitly defines the test runner (`cli/tests/runner.rs`).

Practical implication:

- When you add a new Rust file under `cli/tests/`, you typically must also add a `mod ...;` entry in `cli/tests/runner.rs`.
- There is a guard test (`test_no_forgotten_test_files`) that fails if you forget.

### Prefer lower-level tests when possible

End-to-end CLI tests are much slower than library-level tests. Prefer testing in `jj-lib` when you can, and add a small CLI test only to ensure wiring/flags/UX are correct (see `docs/style_guide.md`).

## Docs (MkDocs) development

The documentation site sources are in `docs/`. The Python toolchain is managed with **uv** (see `pyproject.toml`).

Preview locally:

```bash
uv run mkdocs serve
```

Build like CI (strict):

```bash
uv run mkdocs build --strict
```

Offline build (enables the offline plugin):

```bash
MKDOCS_OFFLINE=true uv run mkdocs build
```

Output goes to `rendered-docs/` (see `mkdocs.yml`).

## Spellcheck (codespell)

CI runs codespell via uv. Locally:

```bash
uv run -- codespell
```

Configuration lives in `pyproject.toml` (`[tool.codespell]`).

## Protobufs (`.proto`) and generated Rust code

The repository includes `.proto` files (under `lib/src/`) and checks in the generated Rust output.

When you change `.proto` files:

1. Ensure you have `protoc` installed (e.g. `protobuf-compiler` on Debian/Ubuntu).
2. Regenerate code:

```bash
cargo run -p gen-protos
```

CI verifies there are **no uncommitted diffs** after generation.

## Optional but recommended dev tooling

- **nextest**: faster test runner (`cargo nextest run ...`)
- **cargo-insta**: snapshot testing workflows
- **bacon**: background checking/linting
- **mold** (Linux): can speed up linking (CI config uses mold on Linux)

See `docs/contributing.md` for suggested installs and workflows.

## Nix + direnv (optional)

This repo ships a Nix flake with a dev shell (Rust toolchain + recommended tools).

- Enter dev shell:

```bash
nix develop
```

- With direnv, you can source the recommended setup:

```bash
echo "source_env .envrc.recommended" >> .envrc
direnv allow
```

See `.envrc.recommended` and `flake.nix` for details.

## Logging / debugging / profiling

From `docs/contributing.md`:

- **Logs**: `JJ_LOG` (EnvFilter-style directives) or `jj --debug` for targeted debug logging
- **Tracing**: `JJ_TRACE=/tmp/trace.json jj <command>` (Perfetto-compatible output)

## CI expectations (what must pass)

The main CI workflow is in `.github/workflows/ci.yml`. In addition to build/test, it checks:

- rustfmt (nightly)
- clippy (`-D warnings`)
- protobuf regeneration is up-to-date
- doctests + `cargo doc` warnings denied
- docs build (`uv run mkdocs build --strict`)
- codespell
- cargo-deny policy checks

If you want to mirror CI locally, the “Quickstart” commands above cover the main expectations.

## Contribution conventions (high level)

See `docs/contributing.md` for the full details. Highlights:

- Commits are reviewed **commit-by-commit**; avoid “fixup” stacks in PRs.
- Commit messages typically start with a **topic prefix** like `<topic>: ...` (not conventional commits).
- Keep refactors and features in separate commits when possible; keep tests/docs with the code they relate to.
