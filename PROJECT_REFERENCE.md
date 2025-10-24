# Jujutsu (jj) - Project Reference Documentation

> **Last Updated**: 2025-10-24
> **Version**: 0.32.0
> **Purpose**: Comprehensive reference for future development and maintenance

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Core Architecture](#core-architecture)
3. [Project Structure](#project-structure)
4. [Technology Stack](#technology-stack)
5. [Key Components](#key-components)
6. [Development Workflow](#development-workflow)
7. [Testing Strategy](#testing-strategy)
8. [Build System](#build-system)
9. [Documentation](#documentation)
10. [CI/CD Pipeline](#cicd-pipeline)
11. [Contributing Guidelines](#contributing-guidelines)

---

## Project Overview

### What is Jujutsu?

Jujutsu is an **experimental distributed version control system** designed as a modern alternative to Git. It abstracts the user interface and VCS algorithms from storage systems, allowing for multiple backends while maintaining Git compatibility as the production-ready option.

**Key Information**:
- **Project Name**: Jujutsu (command: `jj`)
- **Language**: Rust (Edition 2024, MSRV 1.85)
- **License**: Apache 2.0
- **Repository**: https://github.com/jj-vcs/jj
- **Documentation**: https://jj-vcs.github.io/jj/
- **Status**: Experimental but stable for Git compatibility; used daily by all core developers

### Design Philosophy

Jujutsu draws inspiration from multiple VCS systems:

1. **Git**: Fast performance, efficient algorithms, Git repository storage for interoperability
2. **Mercurial & Sapling**: Revset language, no staging area, anonymous branches, template language
3. **Darcs**: First-class conflict objects that can be resolved and propagated automatically

### Innovative Features

1. **Working-copy-as-a-commit**: Changes recorded automatically as commits, amended on each change
   - Eliminates need for stashes or staging area
   - Simplifies user model (commits are the only visible object)

2. **Operation log & undo**: Every operation recorded with repository snapshots
   - Easy debugging ("what just happened?")
   - Full undo/redo capabilities
   - Helps debug coworker repositories

3. **Automatic rebase and conflict resolution**: Descendants automatically rebased when modifying commits
   - Transparent version of `git rebase --update-refs` + `git rerere`
   - Conflict resolutions propagated through descendants

4. **Safe concurrent replication** (experimental): Designed to work safely with distributed filesystems
   - Safe with Dropbox, rsync, S3 backups
   - Concurrent operations never corrupt repository state

---

## Core Architecture

### Layered Architecture (Bottom to Top)

```
┌─────────────────────────────────────────┐
│      CLI Layer (User Interface)         │
│  - Command parsing (clap)                │
│  - User interaction (crossterm)          │
│  - Output formatting, error handling     │
└─────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────┐
│      Workspace Layer                     │
│  - Working copy management               │
│  - File change tracking                  │
│  - Configuration resolution              │
└─────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────┐
│   Query & Language Layer                 │
│  - Revset (commit selection)             │
│  - Fileset (file selection)              │
│  - Template (output formatting)          │
└─────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────┐
│      Domain Layer (Core VCS Logic)       │
│  - Commit graph algorithms               │
│  - Conflict resolution                   │
│  - History rewriting                     │
│  - Evolution tracking, DAG walking       │
└─────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────┐
│   Persistence Layer (Store & Index)      │
│  - Transaction management                │
│  - Operation logging                     │
│  - Index maintenance                     │
└─────────────────────────────────────────┘
                    ↕
┌─────────────────────────────────────────┐
│      Storage Layer (Backend)             │
│  - Git/Simple/Secret backends            │
│  - Protocol Buffer serialization         │
│  - Object storage (commits, trees)       │
└─────────────────────────────────────────┘
```

### Key Design Principles

1. **Storage Independence**: Abstract backends allow different storage models (enables future cloud backends)

2. **Library/CLI Separation**:
   - `jj-lib`: No terminal I/O, config only from repo (designed for future GUI/TUI/server use)
   - `jj-cli`: All user interaction and configuration

3. **Operation Log**: Every operation recorded with snapshot (enables undo, redo, recovery, debugging)

4. **First-Class Conflicts**: Conflicts stored in commits, resolution propagated through descendants

5. **Working Copy as Commit**: Working directory is a real commit (eliminates staging area concept)

---

## Project Structure

```
/home/user/jj/
├── cli/                          # CLI crate (jj-cli) - User-facing interface
│   ├── src/
│   │   ├── main.rs              # Entry point for jj binary
│   │   ├── lib.rs               # CLI library root
│   │   ├── commands/            # 37 command modules
│   │   │   ├── abandon.rs
│   │   │   ├── absorb.rs
│   │   │   ├── commit.rs
│   │   │   ├── log.rs
│   │   │   ├── rebase.rs
│   │   │   └── ... (32 more)
│   │   ├── ui.rs                # Terminal UI and user interaction
│   │   ├── templater.rs         # Template system for output formatting
│   │   ├── config.rs            # CLI configuration handling
│   │   ├── cli_util.rs          # CLI utilities and helpers
│   │   ├── git_util.rs          # Git-specific utilities
│   │   └── formatter.rs         # Output formatting
│   ├── Cargo.toml               # CLI crate manifest
│   ├── build.rs                 # Build script for CLI
│   ├── testing/                 # Test helper binaries
│   └── tests/                   # Integration and CLI tests
│
├── lib/                          # jj-lib crate - Core VCS library
│   ├── src/
│   │   ├── lib.rs               # Library root
│   │   │
│   │   ├── backend.rs           # Backend trait definitions (core abstraction)
│   │   ├── git_backend.rs       # Git storage backend (production)
│   │   ├── simple_backend.rs    # Simple file-based backend (PoC)
│   │   ├── secret_backend.rs    # Secret storage backend
│   │   │
│   │   ├── repo.rs              # Repository core structure
│   │   ├── workspace.rs         # Workspace management
│   │   ├── store.rs             # Store abstraction layer
│   │   ├── transaction.rs       # Transaction handling
│   │   │
│   │   ├── commit.rs            # Commit data structures
│   │   ├── commit_builder.rs    # Commit construction utilities
│   │   ├── tree.rs              # Tree data structures
│   │   ├── files.rs             # File operations and paths
│   │   │
│   │   ├── index.rs             # Index interfaces
│   │   ├── default_index/       # Default index implementation
│   │   │
│   │   ├── op_store.rs          # Operation store interfaces
│   │   ├── op_heads_store.rs    # Operation heads storage
│   │   ├── simple_op_store.rs   # Simple op store implementation
│   │   │
│   │   ├── local_working_copy.rs # Working copy implementation
│   │   ├── working_copy.rs      # Working copy traits
│   │   │
│   │   ├── diff.rs              # Diff algorithms and structures
│   │   ├── conflicts.rs         # Conflict handling and resolution
│   │   ├── evolution.rs         # Commit evolution/history tracking
│   │   │
│   │   ├── revset.rs            # Revset language evaluation (largest module)
│   │   ├── revset_parser.rs     # Revset parser
│   │   ├── revset.pest          # Revset grammar (PEG format)
│   │   │
│   │   ├── fileset.rs           # Fileset language
│   │   ├── fileset_parser.rs    # Fileset parser
│   │   ├── fileset.pest         # Fileset grammar
│   │   │
│   │   ├── config.rs            # Configuration system
│   │   ├── config_resolver.rs   # Config resolution logic
│   │   │
│   │   ├── rewrite.rs           # History rewriting utilities
│   │   ├── dag_walk.rs          # DAG walking algorithms
│   │   ├── graph.rs             # Graph algorithms
│   │   │
│   │   ├── git.rs               # Git integration (conditional feature)
│   │   ├── git_subprocess.rs    # Git subprocess management
│   │   │
│   │   ├── absorb.rs            # Absorb command logic
│   │   ├── annotate.rs          # Blame/annotate functionality
│   │   ├── fix.rs               # Fix/linter integration
│   │   │
│   │   ├── signing.rs           # Commit signing abstractions
│   │   ├── gpg_signing.rs       # GPG signing implementation
│   │   ├── ssh_signing.rs       # SSH signing implementation
│   │   │
│   │   ├── fsmonitor.rs         # Filesystem monitoring
│   │   ├── gitignore.rs         # Gitignore handling
│   │   │
│   │   └── protos/              # Protocol buffer definitions
│   │       ├── git_store.proto
│   │       ├── simple_store.proto
│   │       ├── local_working_copy.proto
│   │       ├── default_index.proto
│   │       └── simple_op_store.proto
│   │
│   ├── Cargo.toml               # Library crate manifest
│   ├── tests/                   # Library tests
│   ├── benches/                 # Performance benchmarks
│   ├── proc-macros/             # Procedural macro definitions
│   └── testutils/               # Testing utilities
│
├── docs/                         # MkDocs documentation source
│   ├── index.md                 # Home page
│   ├── tutorial.md              # Getting started tutorial
│   ├── FAQ.md                   # Frequently asked questions
│   ├── config.md                # Configuration reference
│   ├── revsets.md               # Revset language documentation
│   ├── templates.md             # Template language documentation
│   ├── filesets.md              # Fileset language documentation
│   ├── git-comparison.md        # Git vs Jujutsu comparison
│   ├── git-compatibility.md     # Git compatibility details
│   ├── conflicts.md             # Conflict handling guide
│   ├── bookmarks.md             # Bookmark concepts
│   ├── operation-log.md         # Operation log documentation
│   ├── working-copy.md          # Working copy concepts
│   ├── glossary.md              # Terminology glossary
│   ├── contributing.md          # Contributor guidelines
│   ├── technical/               # Technical documentation
│   │   ├── architecture.md      # System architecture
│   │   ├── concurrency.md       # Concurrency model
│   │   └── conflicts.md         # Conflict handling details
│   └── design/                  # Design proposals
│
├── .github/
│   └── workflows/               # CI/CD workflows
│       ├── ci.yml               # Main CI pipeline
│       ├── pr.yml               # PR checks
│       ├── release.yml          # Release automation
│       ├── binaries.yml         # Binary distribution
│       └── docs.yml             # Documentation building
│
├── Cargo.toml                   # Root workspace manifest
├── Cargo.lock                   # Dependency lock file
├── mkdocs.yml                   # MkDocs configuration
├── pyproject.toml               # Python documentation tooling
├── flake.nix                    # Nix development environment
├── rustfmt.toml                 # Rust code formatting rules
├── deny.toml                    # Dependency security scanning
├── README.md                    # Project README
├── CHANGELOG.md                 # Version history
├── GOVERNANCE.md                # Project governance
└── LICENSE                      # Apache 2.0 license
```

---

## Technology Stack

### Primary Language

**Rust**
- Edition: 2024
- MSRV (Minimum Supported Rust Version): 1.85
- Update locations when changing MSRV: CI, contributing.md, changelog.md, install-and-setup.md

### Core Dependencies

#### VCS & Storage
- `gix` (v0.73.0) - Gitoxide: Rust implementation of Git
- Git backend for production storage

#### Parsing & DSLs
- `pest` (v2.8.1) - PEG parser generator
- `pest_derive` - Procedural macros for pest grammars
- Used for: Revset, Fileset, and Template languages

#### Serialization
- `prost` (v0.13.5) - Protocol Buffers
- `serde` (v1.0) - Serialization framework
- `serde_json` (v1.0.143)
- `toml` / `toml_edit` - Configuration parsing

#### Cryptography & Hashing
- `blake2` (v0.10.6) - Content hashing
- `sha2` - SHA2 hashing
- GPG and SSH signing support

#### Async & Concurrency
- `tokio` (v1.47.1) - Async runtime
- `async-trait` - Async trait support
- `rayon` (v1.10.0) - Data parallelism
- `futures` (v0.3.31) - Async combinators
- `pollster` - Async block runtime

#### Filesystem
- `watchman_client` (optional) - File system monitoring
- `ignore` (v0.4.23) - Gitignore-style patterns
- `globset` (v0.4.16) - Glob pattern support
- `nix` - Unix system calls

#### UI & Output
- `crossterm` (v0.28) - Terminal UI control
- `sapling-renderdag` (v0.1.0) - DAG rendering
- `sapling-streampager` (v0.11.0) - Paging output
- `scm-record` (v0.8.0) - Interactive record UI
- `clap` (v4.5.43) - CLI argument parsing

#### Testing
- `proptest` - Property-based testing
- `insta` (v1.43.1) - Snapshot testing
- `criterion` (v0.5.1) - Benchmarking
- `pretty_assertions` - Better test assertions
- `datatest-stable` - Data-driven testing

#### Documentation
- MkDocs with Material theme
- `mkdocs-include-markdown-plugin`
- `mkdocs-table-reader-plugin`
- `mike` - Multiple version hosting

#### Other Important Libraries
- `regex` (v1.11.2) - Regular expressions
- `chrono` (v0.4.41) - Date/time handling
- `tempfile` (v3.21.0) - Temporary files
- `thiserror` - Error derive macros
- `tracing` (v0.1.41) - Structured logging
- `indexmap` (v2.11.0) - Ordered hash maps

---

## Key Components

### 1. Backend Layer (Storage Abstraction)

**Location**: `lib/src/backend.rs`, `lib/src/git_backend.rs`, `lib/src/simple_backend.rs`

**Purpose**: Abstract interface for commit storage

**Implementations**:
- **GitBackend**: Production storage using gitoxide, stores commits in Git repositories
- **SimpleBackend**: Proof-of-concept file-based storage
- **SecretBackend**: Wrapper for sensitive data

**Why it matters**: Enables swapping storage without changing UI; future cloud backends possible

### 2. Store Layer

**Location**: `lib/src/store.rs`

**Purpose**: Wraps Backend for easier usage, provides commit reading/writing with automatic parent lookup

### 3. Index Layer

**Location**: `lib/src/index.rs`, `lib/src/default_index/`

**Purpose**: Efficient queries on commit graph

**Implementation**: DefaultIndex (production)

### 4. Operation Store Layer

**Location**: `lib/src/op_store.rs`, `lib/src/simple_op_store.rs`

**Purpose**: Tracks repository operations

**Features**:
- Records every operation with repo state snapshots
- Enables operation log browsing and undoing

### 5. Working Copy Layer

**Location**: `lib/src/local_working_copy.rs`, `lib/src/working_copy.rs`

**Purpose**: File system working directory representation

**Features**:
- Snapshots working directory state as commits
- Tracks file metadata, modifications, removals

### 6. Revision Selection (Revset)

**Location**: `lib/src/revset.rs` (largest module ~230K lines), `lib/src/revset_parser.rs`, `lib/src/revset.pest`

**Purpose**: Query language for selecting commits

**Features**:
- Predicates, operators, functions, DAG operations
- Examples: `main..my-branch`, `ancestors(commit)`, `commits(file:pattern)`

### 7. File Selection (Fileset)

**Location**: `lib/src/fileset.rs`, `lib/src/fileset_parser.rs`, `lib/src/fileset.pest`

**Purpose**: Language for selecting files

**Usage**: Various commands for file matching

### 8. Template Language

**Location**: `cli/src/templater.rs`

**Purpose**: Customizing output

**Features**:
- Conditional rendering
- Variable expansion
- Used in commit log, status, other output

### 9. Conflict Handling

**Location**: `lib/src/conflicts.rs`

**Purpose**: First-class conflict objects

**Features**:
- Stores conflict information in commits
- Enables conflict resolution propagation
- Different from Git's textual conflict markers

### 10. Command System

**Location**: `cli/src/commands/`

**37 command modules organized by category**:

- **History**: log, annotate, evolog
- **Modification**: commit, describe, edit, amend, rebase, squash, split
- **Movement**: checkout, restore, duplicate
- **Integration**: git, bookmark
- **Inspection**: diff, show, interdiff
- **Administrative**: config, debug, help
- **Advanced**: absorb, backout, fix, diffedit

### 11. Configuration System

**Location**: `lib/src/config.rs`, `lib/src/config_resolver.rs`, `cli/src/config.rs`

**Multi-level configuration**:
1. System-wide defaults
2. User configuration (`~/.config/jj/config.toml`)
3. Repository configuration (`.jj/config.toml`)
4. Per-command overrides

**Features**:
- Schema validation with JSON schema
- Supports templates, revsets, custom settings

### 12. Git Integration

**Location**: `lib/src/git.rs`, `lib/src/git_backend.rs`, `lib/src/git_subprocess.rs`

**Features**:
- Read/write Git repositories
- Fetch/push to remotes
- Maintain `refs/jj/keep/` for GC safety
- Store extra metadata (change IDs) in StackedTable
- Full interoperability with standard Git tools

**Conditional compilation**: Feature flag "git" (enabled by default)

---

## Development Workflow

### Initial Setup

1. **Clone repository**: `git clone https://github.com/jj-vcs/jj.git`
2. **Nix environment** (optional): `nix develop` (uses `flake.nix`)
3. **Build**: `cargo build`
4. **Test**: `cargo test`

### Development Process

1. **Dogfooding**: All core developers use `jj` for `jj` development
2. **Branch-less development**: Use revsets instead of traditional branches
3. **Code review**: Mandatory for all commits to main
4. **Format code**: `cargo fmt` (configured via `rustfmt.toml`)
5. **Lint**: `cargo clippy` (workspace lints enforced)

### Code Quality Standards

**Enforced via workspace lints** (`Cargo.toml`):
```toml
[workspace.lints.clippy]
explicit_iter_loop = "warn"
flat_map_option = "warn"
implicit_clone = "warn"
needless_for_each = "warn"
semicolon_if_nothing_returned = "warn"
uninlined_format_args = "warn"
unused_trait_names = "warn"
useless_conversion = "warn"
use_self = "warn"
```

### Common Development Tasks

**Run specific test**:
```bash
cargo test test_name
```

**Run benchmarks**:
```bash
cargo bench
```

**Generate documentation**:
```bash
# Rust docs
cargo doc --open

# User docs (requires Python)
mkdocs serve
```

**Update snapshots**:
```bash
cargo insta review
```

### Working with Protocol Buffers

When modifying `.proto` files in `lib/src/protos/`:
1. Edit the `.proto` file
2. Build process automatically regenerates Rust code via `prost-build`
3. Located in `lib/gen-protos/`

---

## Testing Strategy

### Test Locations

- `lib/tests/` - Library integration tests
- `cli/tests/` - CLI integration tests
- Module-level tests via `#[cfg(test)]`

### Testing Frameworks

#### 1. Snapshot Testing (`insta`)

**Purpose**: Capture command output as snapshots, verify against expected results

**Great for**: CLI regression testing

**Usage**:
```rust
insta::assert_snapshot!(output);
```

**Review changes**:
```bash
cargo insta review
```

#### 2. Property-Based Testing (`proptest`)

**Purpose**: Generate random test data, verify properties hold universally

**Usage**: State machine testing, fuzzing

**Optimized**: Compiled in release mode for faster tests (see `Cargo.toml` profile)

#### 3. Data-Driven Testing (`datatest-stable`)

**Purpose**: Test data in files, multiple inputs from single test function

#### 4. Benchmark Testing (`criterion`)

**Purpose**: Performance regression detection

**Location**: `lib/benches/`

**Critical areas**: Diff algorithm, revset evaluation

#### 5. Test Helpers

**Location**: `lib/testutils/`

**Includes**:
- Fake backend for testing
- Test data builders
- Common assertions

### Running Tests

**All tests**:
```bash
cargo test
```

**Specific test**:
```bash
cargo test test_name
```

**With logging**:
```bash
RUST_LOG=debug cargo test
```

**Benchmarks**:
```bash
cargo bench
```

---

## Build System

### Cargo Workspace Structure

**Root Workspace** (`Cargo.toml`):
- Resolver: "3" (workspace dependency resolution)
- Members:
  - `cli` - Command-line interface (jj binary)
  - `lib` - Core library (jj-lib)
  - `lib/gen-protos` - Protobuf code generation
  - `lib/proc-macros` - Procedural macros
  - `lib/testutils` - Testing utilities

**Shared Configuration**:
```toml
[workspace.package]
version = "0.32.0"
license = "Apache-2.0"
rust-version = "1.85"
edition = "2024"
```

### Build Scripts

#### 1. CLI Build Script (`cli/build.rs`)

Generates:
- CLI documentation
- Man pages
- Shell completions (bash, zsh, fish, nushell)
- Processes `clap_mangen` and `clap_complete`

#### 2. Protobuf Compilation

- `prost-build` compiles `.proto` files in `lib/src/protos/`
- Generates Rust serialization code for persistent storage

### Features

**Library Features** (`lib/Cargo.toml`):
- `git` (default) - Git backend support
- `watchman` - Filesystem monitoring integration
- `testing` - Internal testing features

**CLI Features** (`cli/Cargo.toml`):
- `watchman` (default) - Filesystem monitoring
- `git` (default) - Git support
- `test-fakes` - Testing infrastructure

### Profile Optimization

**Development** (`profile.dev.package`):
- `insta`, `proptest` compiled in release mode for faster tests
- Improves test iteration speed

**Release** (`profile.release`):
- `strip = "debuginfo"` - Remove debug info
- `codegen-units = 1` - Single codegen unit for max optimization

### Distribution

- **Cargo**: `cargo install jj-cli`
- **Cargo binstall**: Pre-built binaries
- **Debian**: `cargo deb` configuration
- **Platform-specific**: Unix/Windows conditional compilation

---

## Documentation

### Documentation System

**Platform**: MkDocs with Material theme

**Source**: `/docs/` directory (Markdown)

**Build**: Python-based with `uv` package manager

**Hosting**: GitHub Pages (https://jj-vcs.github.io/jj/)

**Versioning**: `mike` plugin for multiple version support

### Documentation Structure

#### 1. User Documentation
- Installation and setup
- Tutorial
- FAQ
- CLI reference (auto-generated from code)
- Configuration guide
- Query languages (revset, fileset, templates)

#### 2. Concept Documentation
- Working copy behavior
- Bookmarks (branches)
- Conflict handling
- Operation log
- Glossary

#### 3. Comparison Guides
- Git comparison
- Git command mapping table
- Git compatibility details
- Sapling comparison
- Related tools

#### 4. Technical Documentation
- Architecture overview (`docs/technical/architecture.md`)
- Concurrency model
- Conflict resolution algorithm
- Design proposals

#### 5. Contributor Documentation
- Contributing guidelines
- Code of conduct
- Style guide
- Design document process
- Governance

### Building Documentation

**Local preview**:
```bash
mkdocs serve
```

**Build static site**:
```bash
mkdocs build
```

**Deploy to GitHub Pages** (automated via CI):
```bash
mike deploy --push --update-aliases VERSION_TAG latest
```

### Documentation Dependencies

```toml
[dependency-groups]
dev = [
    "mkdocs<1.7,>=1.6",
    "mkdocs-material==9.6.14",
    "mike<3,>=2.1.3",
    "mkdocs-include-markdown-plugin>=7.1.4",
    "mkdocs-table-reader-plugin>=3.1.0",
    "codespell[toml]>=2.4.0",
]
```

---

## CI/CD Pipeline

### GitHub Actions Workflows

**Location**: `.github/workflows/`

#### 1. CI (`ci.yml`)

**Runs on**: Every push, pull request

**Jobs**:
- Lint checks (`cargo clippy`, `rustfmt`)
- Test suite (all platforms: Linux, macOS, Windows)
- Coverage reporting
- MIRI (undefined behavior detection)

#### 2. PR (`pr.yml`)

**Runs on**: Pull requests

**Jobs**:
- Pre-release checks
- Code review gates

#### 3. Release (`release.yml`)

**Runs on**: Release tags

**Jobs**:
- Version bumping
- Changelog generation
- Release notes
- Crate publishing to crates.io

#### 4. Binaries (`binaries.yml`)

**Runs on**: Release tags

**Jobs**:
- Multi-platform binary builds (macOS, Linux, Windows)
- Asset uploads to GitHub releases

#### 5. Documentation (`docs.yml`)

**Runs on**: Push to main, release tags

**Jobs**:
- MkDocs build
- Version management with `mike`
- Deploy to GitHub Pages

#### 6. Security

**Automated**:
- Dependabot for dependency updates
- SLSA provenance (scorecards)
- `deny.toml` for dependency security scanning

### CI Configuration Tips

**Running CI locally**:
```bash
# Install act (https://github.com/nektos/act)
act -j test
```

**Key files**:
- `.github/workflows/*.yml` - Workflow definitions
- `deny.toml` - Dependency security policy
- `rustfmt.toml` - Code formatting rules

---

## Contributing Guidelines

### Before Contributing

1. **Read contributing guide**: `docs/contributing.md`
2. **Join community**:
   - Discord: https://discord.gg/dkmfj3aGQN
   - IRC: #jujutsu on Libera Chat
   - GitHub Discussions

3. **Sign CLA**: Mandatory Contributor License Agreement
   - Does NOT transfer copyright ownership
   - Gives project right to redistribute and use changes

### Development Process

#### 1. Code Review

**Mandatory** for all commits to main:
- All changes require review
- Use GitHub pull requests
- Respond to reviewer feedback

#### 2. Code Style

**Enforced**:
- `rustfmt` for formatting
- Clippy lints (workspace-level configuration)
- Snapshot tests for regression prevention

#### 3. Testing Requirements

**All changes must**:
- Include tests (unit, integration, or snapshot)
- Pass existing test suite
- Pass clippy without warnings

#### 4. Commit Messages

**Format**:
```
component: Brief summary (50 chars or less)

More detailed explanation if needed. Explain what and why,
not how (code shows the how).

Fixes #123
```

### Finding Work

**Good first issues**: https://github.com/jj-vcs/jj/labels/good%20first%20issue

**Areas needing help**:
- Documentation improvements
- Bug fixes
- Performance improvements
- New features (discuss design first)

### Submitting Changes

1. **Fork repository**
2. **Create feature branch** (or use jj's branch-less workflow)
3. **Make changes**
4. **Write tests**
5. **Run test suite**: `cargo test`
6. **Format code**: `cargo fmt`
7. **Lint**: `cargo clippy`
8. **Update snapshots if needed**: `cargo insta review`
9. **Commit changes**
10. **Push to fork**
11. **Open pull request**

### Community Guidelines

**Be respectful**:
- Follow Code of Conduct
- Be patient with new contributors
- Provide constructive feedback

**Communication**:
- Ask questions if unclear
- Discuss design before implementing large features
- Keep PRs focused and reasonably sized

---

## Notable Code Patterns

### 1. Builder Pattern

**Example**: `CommitBuilder` for safe commit construction

**Location**: `lib/src/commit_builder.rs`

**Usage**:
```rust
CommitBuilder::new()
    .author(author)
    .message(message)
    .tree(tree_id)
    .build()
```

### 2. Trait-Based Abstraction

**Core traits**:
- `Backend` - Storage abstraction
- `Store` - Store operations
- `Index` - Commit graph indexing
- `OpStore` - Operation storage

**Benefits**: Enables storage independence

### 3. Error Handling

**Pattern**: `thiserror` for error definitions

**Usage**: Result types throughout, structured error reporting

### 4. Procedural Macros

**Location**: `lib/proc-macros/`

**Example**: `content_hash` macro for content addressing

### 5. PEG Parsing

**Pattern**: Grammar-based parsing for DSLs

**Three languages**:
- Revset (`.pest` grammar)
- Fileset (`.pest` grammar)
- Template (`.pest` grammar)

### 6. Protocol Buffers

**Purpose**: Storage format for persistent data

**Benefits**: Forward compatibility, efficient serialization

---

## Performance Considerations

### Optimizations

1. **Index for fast commit graph queries**
   - Location: `lib/src/default_index/`
   - SQLite-like stacking

2. **Caching strategies**
   - `clru` for LRU caches
   - Memoization in revset evaluation

3. **Parallelization**
   - `rayon` for data parallelism
   - Async operations with `tokio`

4. **Watchman integration**
   - Optional filesystem monitoring
   - Fast change detection in large repositories

### Benchmarks

**Location**: `lib/benches/`

**Critical benchmarks**:
- Diff algorithm (`benches/diff_bench.rs`)
- Revset evaluation
- Index operations

**Run benchmarks**:
```bash
cargo bench
```

**Compare with baseline**:
```bash
cargo bench --bench diff_bench -- --save-baseline before
# Make changes
cargo bench --bench diff_bench -- --baseline before
```

### Large Repository Support

**Design considerations**:
- Efficient revset evaluation
- Incremental index updates
- Lazy loading of commit data

---

## Security Features

### 1. Signing

**Location**: `lib/src/signing.rs`, `lib/src/gpg_signing.rs`, `lib/src/ssh_signing.rs`

**Supported methods**:
- GPG commit signing
- SSH signing
- Test signing backend (for testing)

### 2. Hashing

**Algorithms**:
- BLAKE2 for content hashing (default)
- SHA2 support

### 3. Dependency Security

**Tools**:
- `deny.toml` for scanning
- Automated Dependabot updates
- GitHub security advisories

**Policy**: `SECURITY.md`

**Report vulnerabilities**: See `SECURITY.md` for contact info

---

## Extensibility Points

### 1. Backend Implementations

**How**: Implement `Backend` trait

**Current**: Git, Simple (PoC), Secret

**Future possibilities**: Cloud backends, custom storage

### 2. Signing Backends

**How**: Implement `Signing` trait

**Current**: GPG, SSH, Test

**Future possibilities**: Custom signing methods

### 3. Index Implementations

**How**: Implement `Index` trait

**Current**: DefaultIndex (SQLite-like stacking)

**Future possibilities**: Custom index strategies

### 4. Commands

**How**: Add module in `cli/src/commands/`

**Process**:
1. Create `commands/new_command.rs`
2. Define command struct with `clap` derive
3. Implement command logic
4. Register in `commands/mod.rs`
5. Add tests in `cli/tests/`

### 5. Working Copy Backends

**How**: Implement `WorkingCopy` trait

**Current**: `LocalWorkingCopy`

**Future possibilities**: Remote working copies, virtual filesystems

---

## Important File References

### Core Documentation

- `README.md` - Project overview
- `docs/technical/architecture.md` - Architecture details
- `docs/contributing.md` - Developer guide
- `CHANGELOG.md` - Version history
- `GOVERNANCE.md` - Project governance

### Build & Configuration

- `Cargo.toml` - Workspace structure
- `lib/Cargo.toml` - Library dependencies
- `cli/Cargo.toml` - CLI dependencies
- `rustfmt.toml` - Formatting rules
- `deny.toml` - Security policy

### Code Organization

- `lib/src/lib.rs` - Library module organization
- `cli/src/lib.rs` - CLI layer organization
- `cli/src/commands/mod.rs` - Command registry

### CI/CD

- `.github/workflows/ci.yml` - Main CI pipeline
- `.github/workflows/release.yml` - Release process
- `.github/workflows/docs.yml` - Documentation deployment

---

## Quick Reference Commands

### Development

```bash
# Build
cargo build

# Test
cargo test

# Test specific
cargo test test_name

# Format
cargo fmt

# Lint
cargo clippy

# Benchmarks
cargo bench

# Generate docs
cargo doc --open

# Review snapshots
cargo insta review
```

### Documentation

```bash
# Serve docs locally
mkdocs serve

# Build docs
mkdocs build

# Check spelling
codespell
```

### Git Operations

```bash
# Current branch
git status

# Fetch latest
git fetch origin

# Update dependencies
cargo update
```

---

## Key Takeaways for Future Development

1. **Separation of Concerns**: Library (jj-lib) contains VCS logic with no UI; CLI (jj-cli) handles all user interaction

2. **Storage Independence**: Backend trait allows different storage implementations; Git is production-ready

3. **Testing is Critical**: Every change needs tests (snapshot, property-based, or integration)

4. **Revset is Central**: Understanding the revset language and evaluation is key to understanding jj

5. **Operation Log**: Every operation is recorded; this is fundamental to jj's design

6. **First-Class Conflicts**: Conflicts are stored in commits, not just textual diffs

7. **Working Copy is a Commit**: This simplifies the model and eliminates staging area

8. **Dogfooding**: Developers use jj for jj development; practical experience drives design

9. **Community Matters**: Active Discord, IRC, and GitHub Discussions; ask questions!

10. **Experimental Status**: While stable for Git compatibility, breaking changes may occur before 1.0

---

## Version Information

**Document Version**: 1.0
**Project Version**: 0.32.0
**Last Updated**: 2025-10-24
**Rust Version**: 1.85 (Edition 2024)

---

## Additional Resources

### Official Links

- **Homepage**: https://jj-vcs.github.io/jj
- **Repository**: https://github.com/jj-vcs/jj
- **Documentation**: https://jj-vcs.github.io/jj/latest/
- **Discord**: https://discord.gg/dkmfj3aGQN
- **IRC**: #jujutsu on Libera Chat

### Community Resources

- **GitHub Discussions**: https://github.com/jj-vcs/jj/discussions
- **Wiki**: https://github.com/jj-vcs/jj/wiki
- **Media References**: https://github.com/jj-vcs/jj/wiki/Media

### Learning Resources

- Tutorial: https://jj-vcs.github.io/jj/latest/tutorial
- Git Comparison: https://jj-vcs.github.io/jj/latest/git-comparison
- FAQ: https://jj-vcs.github.io/jj/latest/FAQ
- Glossary: https://jj-vcs.github.io/jj/latest/glossary

---

*This document serves as a comprehensive reference for the Jujutsu project. Keep it updated as the project evolves.*
