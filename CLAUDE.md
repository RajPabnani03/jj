# CLAUDE.md — Jujutsu (jj) Codebase Guide

This file provides guidance for AI assistants working in the Jujutsu version
control system repository.

## Project Overview

Jujutsu (`jj`) is a powerful, Git-compatible version control system written in
Rust. It abstracts the user interface and version control algorithms from the
underlying storage backend (currently Git). Key distinguishing features:

- **Working-copy-as-a-commit**: changes are automatically snapshotted
- **First-class conflicts**: conflicts are stored in the commit graph
- **Operation log & undo**: every repo mutation is recorded and reversible
- **Automatic rebase**: descendants rebase automatically on ancestor changes
- **Revset language**: powerful expression language for selecting commits

Current version: `0.32.0` — Apache 2.0 licensed.

---

## Repository Layout

```
jj/
├── Cargo.toml          # Workspace root — lists all crate members
├── Cargo.lock
├── deny.toml           # cargo-deny configuration (licenses, advisories)
├── rustfmt.toml        # rustfmt configuration (nightly, max_width=100)
├── cli/                # jj-cli crate: binary, commands, UI layer
│   ├── src/
│   │   ├── main.rs
│   │   ├── commands/   # One file per command (or subdir for command groups)
│   │   ├── cli_util.rs # WorkspaceCommandHelper and shared CLI logic
│   │   ├── ui.rs       # Terminal output abstraction
│   │   ├── formatter.rs
│   │   ├── templater.rs / template_builder.rs / template_parser.rs
│   │   ├── revset_util.rs
│   │   ├── diff_util.rs
│   │   └── config/     # Config schema and loading
│   └── tests/          # End-to-end tests (one file per command)
│       ├── common/     # TestEnvironment and test helpers
│       └── test_*.rs
├── lib/                # jj-lib crate: core VCS logic (no CLI dependency)
│   ├── src/
│   │   ├── backend.rs       # Backend trait: CommitId, ChangeId, TreeId, etc.
│   │   ├── repo.rs          # Repo, MutableRepo, ReadonlyRepo
│   │   ├── commit.rs        # Commit type
│   │   ├── commit_builder.rs
│   │   ├── revset.rs        # Revset language evaluation
│   │   ├── fileset.rs       # File pattern matching
│   │   ├── rewrite.rs       # Rebase, squash, history rewriting
│   │   ├── git_backend.rs   # Git storage backend (via gix)
│   │   ├── index.rs / default_index/ # Commit graph index
│   │   ├── transaction.rs   # Mutation transactions
│   │   ├── view.rs          # Repo view (bookmarks, heads)
│   │   ├── working_copy.rs  # Working copy abstraction
│   │   └── config.rs        # Config types
│   ├── tests/               # Unit/integration tests for lib
│   ├── benches/             # Criterion benchmarks
│   ├── gen-protos/          # Build tool: generates .rs from .proto files
│   ├── proc-macros/         # Procedural macros (jj-lib-proc-macros)
│   └── testutils/           # Shared test helpers (TestRepo, TestWorkspace)
├── docs/               # MkDocs documentation source (Markdown)
│   ├── contributing.md # Development setup and policies
│   ├── style_guide.md  # Code style rules
│   └── ...
├── .github/
│   └── workflows/
│       ├── ci.yml      # Main CI: test, clippy, rustfmt, cargo-deny, docs
│       └── release.yml # Release workflow
└── flake.nix           # Nix development environment
```

---

## Workspace Crates

| Crate | Path | Purpose |
|---|---|---|
| `jj-cli` | `cli/` | CLI binary and command implementations |
| `jj-lib` | `lib/` | Core VCS library (no CLI dependency) |
| `jj-lib-proc-macros` | `lib/proc-macros/` | Derive macros for jj-lib |
| `testutils` | `lib/testutils/` | Shared test infrastructure |
| `gen-protos` | `lib/gen-protos/` | Generates Rust code from `.proto` files |

Minimum Supported Rust Version (MSRV): **1.85** (stable − 1).

---

## Development Commands

### Building

```sh
cargo build --workspace
cargo build --workspace --all-targets   # includes tests, benches, examples
```

### Testing

```sh
# Standard
cargo test --workspace

# Faster (recommended)
cargo nextest run --workspace

# With snapshot tests
cargo insta test --workspace
cargo insta test --workspace --test-runner nextest

# Review updated snapshots
cargo insta review --workspace
```

### Formatting (nightly required)

```sh
cargo +nightly fmt            # format all code
cargo +nightly fmt --check    # CI check (fails if changes needed)
```

The `rustfmt.toml` settings: edition=2024, max_width=100, format_strings=true,
group_imports=StdExternalCrate.

### Linting

```sh
cargo clippy --workspace --all-targets -- -D warnings
cargo +nightly clippy          # stricter lints (optional)
```

### Continuous background checking (recommended during development)

```sh
cargo install --locked bacon
bacon clippy-all               # auto-rebuild + clippy on every save
```

### Documentation (MkDocs)

```sh
uv run mkdocs serve            # live preview at http://127.0.0.1:8000
MKDOCS_OFFLINE=true uv run mkdocs build  # offline build
```

### Protobuf generation

Only needed when modifying `.proto` files:

```sh
sudo apt-get install protobuf-compiler   # or equivalent
cargo run -p gen-protos
```

Generated `.rs` files are checked in. CI will fail if they drift from `.proto`.

---

## Key Abstractions

### Backend trait (`lib/src/backend.rs`)

The `Backend` trait is the storage abstraction. The primary implementation is
`GitBackend` (via the `gix` crate). Key IDs:

- `CommitId` — content-addressed identifier; changes on rewrite
- `ChangeId` — stable identifier that follows a commit through rewrites
- `TreeId`, `FileId`, `SymlinkId`, `ConflictId` — other content types

### Repo & Transaction (`lib/src/repo.rs`, `lib/src/transaction.rs`)

- `ReadonlyRepo` — immutable snapshot of the repository state
- `MutableRepo` — wraps `ReadonlyRepo` for in-progress mutations
- `Transaction` — commits a `MutableRepo` mutation, creating a new operation

### View (`lib/src/view.rs`)

Tracks the visible state of the repo: bookmarks (branches), remote-tracking
refs, and commit heads.

### WorkspaceCommandHelper (`cli/src/cli_util.rs`)

The central CLI helper. Commands get a `WorkspaceCommandHelper` via
`CommandHelper::workspace_helper()`. It provides access to the workspace, repo,
settings, and helpers for resolving revsets, loading commits, etc.

### Revset language (`lib/src/revset.rs`, `lib/src/revset.pest`)

A Mercurial-inspired expression language for querying commit graphs. Parsed with
`pest`. Used extensively in CLI `-r` / `--revision` flags.

### Fileset language (`lib/src/fileset.rs`, `lib/src/fileset.pest`)

File pattern language similar to revsets but for file paths.

### Template language (`cli/src/templater.rs`, `cli/src/template.pest`)

A typed template language for formatting output. Templates accept a context
type (e.g., `Commit`, `CommitEvolutionEntry`, `Operation`). Configured via
`templates.*` in user/repo config.

---

## Adding a New Command

1. Create `cli/src/commands/<name>.rs`.
2. Define the CLI struct with `#[derive(clap::Args)]` and doc comments (these
   become the `jj help` text).
3. Implement `fn cmd_<name>(ui: &mut Ui, command: &CommandHelper, args: &<Args>)
   -> Result<(), CommandError>`.
4. Register in `cli/src/commands/mod.rs` (add `mod <name>` and wire into the
   dispatch match).
5. Add end-to-end tests in `cli/tests/test_<name>_command.rs` using
   `TestEnvironment` from `cli/tests/common/`.
6. Update `docs/cli-reference.md` if the command is user-facing.

---

## Testing Conventions

### End-to-end tests (CLI layer)

Located in `cli/tests/test_*.rs`. Use `TestEnvironment` from
`cli/tests/common/`:

```rust
let test_env = TestEnvironment::default();
test_env.run_jj_in(".", ["git", "init", "repo"]).success();
let work_dir = test_env.work_dir("repo");
let output = work_dir.run_jj(["log"]);
insta::assert_snapshot!(output, @r"...");
```

Snapshot tests use the [`insta`](https://insta.rs/) crate. After changing CLI
output, run `cargo insta review --workspace` to update snapshots.

### Library tests (preferred)

Located in `lib/tests/` and `lib/src/*.rs` (`#[cfg(test)]`). Use
`testutils::{TestRepo, TestWorkspace}`. ~100× faster than end-to-end tests.

**Rule**: Prefer lib-level tests. Only add CLI end-to-end tests to verify
integration (e.g., that a flag is wired up correctly).

### Snapshot tests note

`cargo +nightly fmt` is required before tests pass CI. Snapshot files live
under `cli/tests/*.snap` / `cli/tests/cli-reference@.md.snap`.

---

## Commit Message Conventions

Format: `<topic>: <description>` (no conventional-commits prefixes like `feat:`
or `fix:`).

- Topic is typically the command name or subsystem: `log:`, `revset:`,
  `git_backend:`, `config:`, `templater:`, etc.
- No period at end of subject line.
- Body explains *why*, not just *what*.
- Each commit should do **one logical thing**.
- Include tests and docs in the same commit as the feature.

Examples from recent history:
```
revset: allow consecutive '-' and '+' in bookmark names
config-schema: add --when and --scope keys
templater: relax whitespace requirement by leveraging implicit whitespace rule
```

---

## Code Style

- **No panics** in library code unless invariant is documented and checked.
  `.unwrap()` is acceptable when the invariant is guaranteed by prior logic.
- **No `unsafe` code** — `lib/src/lib.rs` has `#![forbid(unsafe_code)]`.
- **Missing docs warning** is enabled for `jj-lib` (`#![warn(missing_docs)]`).
- Clippy lints enforced workspace-wide (see `[workspace.lints.clippy]` in
  `Cargo.toml`): `uninlined_format_args`, `use_self`, `implicit_clone`, etc.
- Imports: grouped as std / external / workspace, one item per `use` line
  (`imports_granularity = "Item"`).
- Line width: 100 characters maximum.
- Rust edition: **2024**.

---

## CI Checks

All of the following must pass for a PR to merge:

| Check | Command |
|---|---|
| Tests | `cargo nextest run --workspace` |
| Rustfmt | `cargo +nightly fmt --all -- --check` |
| Clippy | `cargo +stable clippy --all-features --workspace --all-targets -- -D warnings` |
| cargo-deny | `cargo deny check advisories / bans / licenses / sources` |
| Codespell | spell-check on source files |
| Doctests | `cargo doc --workspace --no-deps` (RUSTDOCFLAGS="--deny warnings") |
| MkDocs | `uv run mkdocs build` |
| Proto check | `cargo run -p gen-protos` + `git diff --exit-code` |
| Nix flake | `nix flake check` |

---

## Environment Variables

| Variable | Purpose |
|---|---|
| `JJ_LOG` | Tracing filter (like `RUST_LOG`); enable debug with `JJ_LOG=debug` |
| `JJ_TRACE` | Path for Perfetto trace JSON output |
| `RUST_BACKTRACE` | Enable backtraces on panic |

---

## Configuration System

Config is layered (TOML): built-in defaults → system → user → repo → command-
line. The schema is in `cli/src/config-schema.json`. The `--config=KEY=VALUE`
and `--config-file=PATH` flags override at the command level.

Key config namespaces: `ui.*`, `diff.*`, `merge-tools.*`, `git.*`, `fix.*`,
`templates.*`, `revset-aliases.*`, `aliases.*`.

---

## Protobuf Data Format

On-disk commit and operation data uses protobuf. `.proto` files live in
`lib/src/protos/`. Generated Rust code is checked in. Run
`cargo run -p gen-protos` after any `.proto` change and commit the generated
files alongside.

---

## Useful Links

- Documentation: <https://jj-vcs.github.io/jj/>
- Contributing guide: `docs/contributing.md`
- Style guide: `docs/style_guide.md`
- CLI reference: `docs/cli-reference.md`
- Revsets: `docs/revsets.md`
- Templates: `docs/templates.md`
- Discord: <https://discord.gg/dkmfj3aGQN>
- Issue tracker: <https://github.com/jj-vcs/jj/issues>
