# CLAUDE.md - AI Assistant Guide for Jujutsu Development

This document provides a comprehensive guide for AI assistants working on the Jujutsu (jj) version control system codebase. It outlines the project structure, development conventions, workflows, and key patterns to follow.

## Project Overview

**Jujutsu** (command: `jj`) is an experimental yet production-ready version control system written in Rust that combines innovative ideas from Git, Mercurial, Sapling, and Darcs. It abstracts the user interface and version control algorithms from the storage systems, allowing it to serve as a VCS with multiple possible physical backends. Currently, it uses Git repositories as a storage layer, making it **fully compatible with Git** and all Git-based tools and forges.

All core developers use Jujutsu to develop Jujutsu itself on GitHub. The project was started by Martin von Zweigbergk (Google) as a hobby project in late 2019 and has evolved into his full-time project at Google with several other Googlers assisting, though **this is not a Google product**.

### Project Information

- **Version**: 0.32.0
- **License**: Apache-2.0 (see [`LICENSE`](./LICENSE))
  - Does not transfer copyright ownership
  - Allows free use, modification, and redistribution
  - Includes patent grant protection
- **Copyright**: The Jujutsu Authors
- **Language**: Rust
- **Edition**: 2024
- **MSRV**: 1.85 (current stable Rust minus one version)
- **Repository**: https://github.com/jj-vcs/jj
- **Documentation**: https://jj-vcs.github.io/jj/
- **Categories**: version-control, development-tools

### Maturity Status

While labeled "experimental," Jujutsu is:
- Used daily by all core developers for all their version control needs
- Feature-complete for most workflows
- Stable Git compatibility
- Production-ready for individual and team use

However, note that:
- Some features (e.g., Git submodules) are incomplete
- Performance optimizations are ongoing
- Backward-incompatible changes may occur before 1.0.0
- Email-based workflows are not natively supported

### Key Features & Innovations

**Core Design Principles:**
- **Working-copy-as-a-commit**: Changes are automatically recorded as commits and amended on each change, eliminating the need for staging areas or stashes
- **Operation log with undo**: Every operation is recorded with repo snapshots, allowing undo/redo of any operation
- **First-class conflicts**: Conflicts are stored in commits and can be resolved later; conflict resolutions propagate through descendants automatically
- **Automatic rebase**: Descendants are automatically rebased when you modify a commit
- **Backend abstraction**: Storage-independent design (currently uses Git repositories for wide compatibility)

**Inspired Features:**
- **Revset language** (from Mercurial/Sapling): Powerful query language for selecting commits
- **Template language**: Configurable output formatting
- **No staging area** (like Mercurial): Simpler mental model
- **Anonymous branches**: No need to name every small change
- **Performance focus** (like Git): Efficient algorithms and data structures

**Advanced Capabilities:**
- Comprehensive history rewriting tools (`describe`, `diffedit`, `split`, `squash`)
- Concurrent-safe replication (experimental): Safe with Dropbox, rsync, etc.
- Co-located repositories: Use both `jj` and `git` commands on the same repo
- Git interoperability: Commits are regular Git commits, fully compatible with Git remotes

## Repository Structure

```
/home/user/jj/
├── lib/                          # Core library (jj-lib)
│   ├── src/                     # Library source code
│   ├── tests/                   # Library tests
│   ├── benches/                 # Benchmarks
│   ├── proc-macros/             # Procedural macros
│   ├── testutils/               # Test utilities
│   └── gen-protos/              # Protocol buffer generation
├── cli/                          # CLI application (jj-cli)
│   ├── src/                     # CLI source code
│   │   ├── commands/            # Command implementations
│   │   └── *.rs                 # Core CLI infrastructure
│   └── tests/                   # CLI integration tests
├── docs/                         # Documentation (Markdown)
│   ├── *.md                     # User documentation
│   ├── technical/               # Technical documentation
│   └── design/                  # Design proposals
├── .github/                      # GitHub workflows and CI/CD
│   ├── workflows/               # CI workflows
│   └── scripts/                 # Helper scripts
├── .cargo/                       # Cargo configuration
├── Cargo.toml                    # Workspace manifest
└── [config files]               # Various config files
```

### Workspace Structure

The repository is a Rust workspace with 5 members:
- `cli` - User-facing CLI application
- `lib` - Core VCS library
- `lib/gen-protos` - Protocol buffer compilation
- `lib/proc-macros` - Procedural macros
- `lib/testutils` - Test utilities

## Key Architectural Components

### Core Library (`lib/src/`)

**Data Model:**
- `repo.rs` - Repository interface (read-only and mutable)
- `commit.rs`, `commit_builder.rs` - Commit objects
- `tree.rs`, `tree_builder.rs` - Tree objects
- `backend.rs` - Abstract storage backend
- `store.rs` - Object storage wrapper

**Storage Backends:**
- `git_backend.rs` - Git repository backend (uses gix/gitoxide)
- `simple_backend.rs` - Proof-of-concept file-based backend

**Core Operations:**
- `revset.rs`, `revset_parser.rs` - Revision set query language
- `operation.rs`, `op_store.rs`, `op_walk.rs` - Operation log system
- `transaction.rs` - ACID transaction management
- `working_copy.rs`, `local_working_copy.rs` - Working directory management
- `rewrite.rs` - Rebase and history rewriting

**Advanced Features:**
- `merge.rs`, `merged_tree.rs` - Three-way merge algorithms
- `conflicts.rs` - First-class conflict representation
- `diff.rs` - Diff computation
- `annotate.rs` - Blame/annotate functionality
- `fileset.rs`, `fileset_parser.rs` - File selection language
- `signing.rs`, `gpg_signing.rs`, `ssh_signing.rs` - Cryptographic signing

**Indexing:**
- `default_index/` - Default indexing system with 10+ submodules
  - `revset_engine.rs` - Revset query evaluation
  - `composite.rs`, `mutable.rs`, `readonly.rs` - Index variants
  - Bitmap-based data structures for performance

### CLI Application (`cli/src/`)

**Command Structure:**
- `commands/` - 46+ command implementations
  - Individual commands: `new.rs`, `commit.rs`, `log.rs`, `diff.rs`, etc.
  - Git-specific: `git/` subdirectory
  - Hierarchical: `bookmark/`, `tag/`, `operation/`, `workspace/`, etc.

**Core Infrastructure:**
- `cli_util.rs` - Main CLI runner and utilities (157K)
- `ui.rs` - User interface abstractions
- `formatter.rs` - Output formatting
- `command_error.rs` - Error handling

**Templating System:**
- `template_builder.rs` - Template AST building (153K)
- `template_parser.rs` - Template language parsing (50K)
- `templater.rs` - Template engine
- `commit_templater.rs` - Commit-specific templates (121K)
- `operation_templater.rs` - Operation templates

**Utilities:**
- `diff_util.rs` - Diff presentation (86K)
- `text_util.rs` - Text processing (57K)
- `config.rs` - CLI configuration (64K)
- `merge_tools/` - External merge tool integration

## Development Setup

### Prerequisites

1. **Install Rust** (1.85 or newer):
   ```bash
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
   ```

2. **Install toolchains:**
   ```bash
   rustup toolchain add nightly  # For rustfmt
   rustup toolchain add 1.85     # MSRV
   ```

3. **Install development tools:**
   ```bash
   cargo install --locked bacon          # Continuous build/test
   cargo install --locked cargo-insta    # Snapshot testing
   cargo install --locked cargo-nextest  # Faster test runner
   ```

### Common Development Commands

```bash
# Format code (MUST use nightly)
cargo +nightly fmt

# Check for lints
cargo clippy --workspace --all-targets

# Build
cargo build --workspace

# Run tests
cargo nextest run --workspace

# Run tests with snapshot updates
cargo insta test --workspace --test-runner nextest
cargo insta review --workspace

# Continuous development
bacon clippy-all
```

### Configure `jj fix` for auto-formatting

```bash
jj config set --repo fix.tools.rustfmt '{ command = ["rustfmt", "+nightly"], patterns = ["glob:**/*.rs"] }'
```

## Coding Conventions and Style Guide

### General Principles

1. **No Panics**: Panics are not allowed, especially in library code
   - `.unwrap()` is acceptable only if safety is guaranteed by prior checks
   - Document invariants that make unwraps safe

2. **Separation of Concerns**:
   - `jj-lib`: Pure logic, no UI/terminal I/O
   - `jj-cli`: User interface and terminal interaction

3. **Error Handling**:
   - Use `thiserror` for error types
   - Return `Result` types, don't panic
   - Provide helpful error messages

### Rust Style

- **Edition**: Use Rust 2024 edition features
- **Formatting**: Use `rustfmt` with nightly toolchain
- **Lints**: Follow workspace-level clippy lints (see `Cargo.toml`)
  - `explicit_iter_loop`
  - `flat_map_option`
  - `implicit_clone`
  - `needless_for_each`
  - `semicolon_if_nothing_returned`
  - `uninlined_format_args`
  - `unused_trait_names`
  - `useless_conversion`
  - `use_self`

### Code Organization

- Keep files focused on a single responsibility
- Use modules to organize related functionality
- Prefer trait-based abstractions for extensibility
- Use `#[cfg(test)]` for unit tests within modules

## Testing

### Test Organization

**Library Tests** (`lib/tests/`):
- 20+ test modules using a runner pattern
- `runner.rs` loads all test modules
- Organized by feature: `test_revset.rs`, `test_conflicts.rs`, etc.

**CLI Tests** (`cli/tests/`):
- Integration tests for CLI commands
- End-to-end testing with the `jj` binary
- Use snapshot testing with `insta`

### Testing Best Practices

1. **Prefer Lower-Level Tests**:
   - Library tests are ~100x faster than CLI tests
   - Use `jj-lib` directly in tests when possible
   - Reserve CLI tests for testing command integration

2. **Snapshot Testing**:
   - Use `insta::assert_snapshot!()` for output verification
   - Review snapshots with `cargo insta review --workspace`
   - Snapshots are in `.snap` files (excluded from git via patterns)

3. **Property Testing**:
   - Use `proptest` for property-based tests
   - See `lib/testutils/proptest.rs` for strategies

4. **Test Utilities**:
   - `testutils` crate provides test fixtures
   - `test_backend.rs` for mock backends
   - `git.rs` for Git test fixtures

### Running Tests

```bash
# Run all tests
cargo nextest run --workspace

# Run specific test file
cargo nextest run -p jj-lib test_revset

# Run tests with snapshot updates
cargo insta test --workspace --test-runner nextest

# Review snapshot changes
cargo insta review --workspace
```

## Commit and PR Workflow

### Commit Guidelines

**IMPORTANT**: Unlike typical GitHub projects, we care about individual commits, not just PRs.

1. **Each commit should do one thing**:
   - Separate refactoring from new features
   - Include tests and docs in the same commit as the code
   - Use `jj split` to split commits if needed

2. **Commit Message Format**:
   ```
   <topic>: <description>

   Optional longer explanation of why (not what) the change was made.
   ```
   - Use topic from affected component (e.g., `revset:`, `next/prev:`, `git:`)
   - NOT using conventional commits format
   - Focus on "why" rather than "what"
   - See: https://cbea.ms/git-commit/

3. **Examples**:
   ```
   revset: add support for ~ operator in ancestry expressions

   This makes it easier to refer to ancestors without using the
   verbose ancestors() function.
   ```

### Pull Request Workflow

1. **Submit PR** with clean commit history
   - Don't squash commits unless they're fixups
   - Each commit is reviewed separately

2. **Address Review Comments**:
   - Don't add commits on top (like typical GitHub workflow)
   - Instead, modify the appropriate commit:
     ```bash
     jj new <commit-to-fix>
     # Make changes
     jj squash
     jj git push  # Auto force-pushes
     ```

3. **After First Approval**:
   - Contributors typically get access to merge their own PRs
   - Address minor comments yourself
   - Request re-review for non-trivial changes

4. **Conflict of Interest**:
   - PRs need approval from someone outside your organization
   - See `docs/contributing.md` for details

### Code Review Checklist

- [ ] Each commit does one logical thing
- [ ] Commit messages are clear and explain "why"
- [ ] Tests are included with code changes
- [ ] Documentation is updated if needed
- [ ] No panics in library code
- [ ] Error handling is appropriate
- [ ] Code follows style guide
- [ ] Snapshots are reviewed and committed

## Building and CI/CD

### Build Configuration

**Workspace Dependencies** (`Cargo.toml`):
- 130+ shared dependencies defined at workspace level
- Key dependencies:
  - `gix` (gitoxide) - Git backend
  - `clap` - CLI argument parsing
  - `serde` - Serialization
  - `prost` - Protocol buffers
  - `pest` - Parser generator (for revset/fileset)
  - `tracing` - Instrumentation

**Feature Flags**:
- `jj-lib`:
  - `git` (default) - Git backend support
  - `watchman` - Watchman filesystem monitoring
  - `testing` - Test utilities
- `jj-cli`:
  - `git` (default) - Git integration
  - `watchman` (default) - Fast working copy
  - `test-fakes` - Fake editor/diff-editor binaries

### CI/CD Pipeline

**Main CI** (`.github/workflows/ci.yml`):
- Matrix testing across:
  - Linux (x86_64, aarch64)
  - macOS (x86_64, aarch64)
  - Windows (x86_64, aarch64)
- Uses `cargo nextest` for faster testing
- Timeout: 20 minutes per job
- Features tested: `--all-features` on Linux, default on others

**Other Workflows**:
- `release.yml` - Release automation
- `binaries.yml` - Binary building
- `docs.yml` - Documentation deployment
- `pr.yml` - PR automation

**Build Optimizations**:
- `mold` linker on Linux for faster linking
- Optimized compilation of test dependencies (see `Cargo.toml` profiles)
- `cargo nextest` for parallel test execution

## Documentation

### Structure

**User Documentation** (`docs/`):
- `tutorial.md` - Getting started
- `config.md` - Configuration guide (62K)
- `revsets.md` - Revset language (23K)
- `templates.md` - Template language (23K)
- `git-compatibility.md` - Git integration
- And many more...

**Developer Documentation**:
- `docs/contributing.md` - Contribution guide
- `docs/style_guide.md` - Code style
- `docs/technical/` - Technical deep-dives
- `docs/design/` - Design proposals

### Building Documentation Locally

```bash
# Install uv (Python project manager)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Serve docs locally (auto-reloads on changes)
uv run mkdocs serve

# View at http://127.0.0.1:8000
```

### Documentation Standards

- Wrap Markdown at 80 columns (no auto-formatter yet)
- Update docs in same commit as code changes
- Use GitHub-flavored Markdown
- Include code examples where helpful

## Key Patterns and Idioms

### Repository Operations

```rust
// Read-only repository access
fn read_repo(repo: &dyn Repo) {
    let commit = repo.store().get_commit(&commit_id)?;
}

// Mutable operations via transactions
fn modify_repo(mut tx: Transaction) -> Result<()> {
    let mut_repo = tx.repo_mut();
    // Make changes
    tx.commit("operation description");
    Ok(())
}
```

### Error Handling

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum MyError {
    #[error("commit not found: {0}")]
    CommitNotFound(CommitId),

    #[error("backend error")]
    Backend(#[from] BackendError),
}
```

### Testing Patterns

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use testutils::TestRepo;

    #[test]
    fn test_something() {
        let test_repo = TestRepo::init();
        let repo = &test_repo.repo;
        // Test code
    }
}
```

### CLI Command Structure

```rust
/// Command documentation
#[derive(clap::Args, Clone, Debug)]
pub struct MyCommand {
    /// Argument documentation
    #[arg(long, short)]
    flag: bool,
}

pub fn cmd_my_command(
    ui: &mut Ui,
    command: &CommandHelper,
    args: &MyCommand,
) -> Result<(), CommandError> {
    // Implementation
    Ok(())
}
```

## Important Files to Know

### Configuration Files

- `Cargo.toml` - Workspace manifest with all dependencies
- `rustfmt.toml` - Code formatting rules
- `deny.toml` - Dependency auditing configuration
- `.cargo/config-ci.toml` - CI-specific Cargo settings
- `mkdocs.yml` - Documentation site configuration

### Build Scripts

- `cli/build.rs` - Generates version string at build time
- `lib/gen-protos/src/main.rs` - Compiles .proto files

### CI Scripts

- `.github/scripts/docs-build-deploy` - Documentation deployment
- `.github/workflows/ci.yml` - Main test CI

### Development Environment

- `flake.nix` / `flake.lock` - Nix development environment
- `.envrc.recommended` - Direnv configuration
- `pyproject.toml` / `uv.lock` - Python tools for docs

## Logging and Debugging

### Logging

```bash
# Enable debug logs for jj-lib and jj-cli
jj --debug <command>

# Enable logs for specific modules
JJ_LOG=jj_lib::revset=debug jj log

# Enable all debug logs
JJ_LOG=debug jj <command>
```

### Profiling

```bash
# Sampling profiler (recommended)
cargo install samply
samply record jj diff

# Tracing instrumentation
JJ_TRACE=/tmp/trace.json jj diff
# Then open https://ui.perfetto.dev/ and load /tmp/trace.json
```

## Common Development Tasks

### Adding a New Command

1. Create `cli/src/commands/my_command.rs`
2. Add command struct with `clap` attributes
3. Implement `cmd_my_command()` function
4. Register in `cli/src/commands/mod.rs`
5. Add tests in `cli/tests/test_my_command.rs`
6. Update documentation

### Modifying Protocol Buffers

1. Install `protoc` compiler
2. Edit `.proto` files in `lib/protos/`
3. Run `cargo run -p gen-protos` to regenerate Rust code
4. Update `lib/gen-protos/src/main.rs` if adding new files

### Adding a New Revset Function

1. Add function to revset parser in `lib/src/revset_parser.rs`
2. Implement evaluation in `lib/src/default_index/revset_engine.rs`
3. Add tests in `lib/tests/test_revset.rs`
4. Document in `docs/revsets.md`

### Fixing a Bug

1. Write a failing test that reproduces the bug
2. Fix the bug in a single commit
3. Ensure the test passes
4. Update any affected snapshots
5. Write clear commit message explaining the fix

## Performance Considerations

- **Lazy evaluation**: Many operations use iterators for efficiency
- **Indexing**: Default index provides O(1) lookups for graph queries
- **Parallel operations**: Use `rayon` for parallelizable work
- **Avoid clones**: Use references and `Cow` where appropriate
- **Benchmark**: Use `cargo bench` for performance-critical code

## Special Considerations for AI Assistants

### When Modifying Code

1. **Always read files before editing** - Don't make assumptions
2. **Respect the architecture** - Keep library and CLI separate
3. **Follow existing patterns** - Look at similar code first
4. **Test thoroughly** - Add or update tests with changes
5. **Document why, not what** - Code should be self-explanatory

### When Adding Features

1. **Check if it already exists** - Search codebase first
2. **Consider design docs** - Large changes may need design review
3. **Think about Git compatibility** - Backend changes affect Git storage
4. **Consider the user experience** - CLI should be intuitive

### When Fixing Bugs

1. **Reproduce the bug** - Write a failing test first
2. **Find root cause** - Don't just patch symptoms
3. **Check for similar bugs** - Fix the pattern, not just the instance
4. **Update documentation** - If behavior changes were documented

### Common Pitfalls to Avoid

- **Don't use `.unwrap()` without justification**
- **Don't add terminal I/O to jj-lib**
- **Don't skip writing tests**
- **Don't make commits that do multiple things**
- **Don't forget to run `cargo +nightly fmt`**
- **Don't assume Git-specific behavior** - Backend is abstract

## Getting Help

- **Discord**: https://discord.gg/dkmfj3aGQN
- **GitHub Discussions**: https://github.com/jj-vcs/jj/discussions
- **IRC**: #jujutsu on Libera Chat
- **Documentation**: https://jj-vcs.github.io/jj/

## Additional Resources

- **Contributing Guide**: `docs/contributing.md`
- **Style Guide**: `docs/style_guide.md`
- **Architecture**: `docs/technical/architecture.md`
- **Changelog**: `CHANGELOG.md`
- **Design Docs**: `docs/design/`

## Version Information

This guide is current as of:
- **Date**: 2026-01-21
- **Version**: 0.32.0
- **Rust MSRV**: 1.85
- **Last Updated**: Initial creation

---

Remember: Jujutsu is an experimental project with high standards for code quality and user experience. Take time to understand the architecture before making changes, and don't hesitate to ask for clarification when needed.
