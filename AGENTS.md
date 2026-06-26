# AGENTS.md — Contributing to ast-guard

This document governs AI coding agents contributing to the ast-guard codebase. It is distinct from the `init`-shipped templates (which govern *users* of ast-guard in their own projects).

## Architecture

ast-guard is a **thin Rust orchestrator**, not a rewrite engine. Its core is a pipeline that:

1. Delegates all pattern matching and substitution to `ast-grep-core` (embedded as a library). No own regex, no own pattern engine.
2. Layers on three safety gates: **re-parse** (tree-sitter ERROR-node detection), **`--require`/`--forbid`** (delta-scoped post-conditions on matched sites), and **compiler validation** (structured JSON/machine-readable output).
3. Wraps external compilers via two trait seams: `Compiler` (Strategy — per-language command construction and output parsing) and `Runner` (Adapter — process spawning).

### Key modules (ordered by importance)

| Module | What it does | Why it's critical |
|---|---|---|
| `reparse.rs` | Feeds rewritten source through tree-sitter; detects ERROR nodes. | The syntactic safety gate — catches malformed rewrites before disk mutation. |
| `filter.rs` | Central `filter_diagnostics(raw, severity)` — uniform across languages. | The diagnostic quality gate — one `match` on severity, no per-compiler logic. |
| `dispatch.rs` | Wires `Compiler` + `Runner`, auto-discovers compile root. | The orchestration hub — the only place where the pipeline stages connect. |
| `runner.rs` | `RealRunner`/`StubRunner` — the only `std::process::Command` seam. | The I/O boundary — all real process spawning, all test mocking. |
| `init.rs` | Writes agent templates with marker blocks (non-destructive). | The teaching system — ships CRITICAL RULE + recovery playbook. |
| `cli.rs` | Clap subcommand definitions — thin, no logic. | Wire-up — all flags and their defaults. |
| `main.rs` | Thin dispatch from clap to subcommand handlers. | Entry point — no logic here. |
| `compiler/{rust,csharp,typescript,dart}.rs` | `Compiler` trait implementations — one per language. | Per-compiler command knowledge + JSON/structured parsers. |

### What ast-guard does NOT implement

- **Pattern matching** — delegated to `ast-grep-core`.
- **Rewrite substitution** — delegated to `ast-grep-core`.
- **Pattern validation** — delegated to `ast-grep-core` (rejects invalid syntax before file read).
- **Git rollback** — explicitly delegated to git and taught via `init`.
- **Own regex engine** — not needed, not wanted.
- **Stderr-regex parsing** — compilers are invoked with structured-output flags (`--message-format=json`), parsed as JSON/structured lines.

## Three-Phase Delivery Strategy

Understand this before writing code:

1. **Phase 1: Agent skill** (shipped — `docs/ast-guard-skills.md`). A Markdown prompt teaching agents to use `ast-grep` + `cargo check` directly. Validates demand.
2. **Phase 2: Lightweight CLI** (this repo — under development). The thin orchestrator. Adds hard `--forbid` gates, structured diagnostics, re-parse gate. Not a platform — a pipeline wrapper.
3. **Phase 3: Mature CLI** (conditional on Phase 2 demand). Rule-file support (`--rule-file <yaml>`), Python support, mutation-testing quality gates, CI-specific modes.

**Current phase: 2.** All code contributions target Phase 2 scope. Do not build Phase 3 features until Phase 2 proves demand.

## Refactoring within ast-guard

When you (the agent) refactor code in this repository, follow the same workflow that `init` teaches to users:

```bash
# 1. PREVIEW
ast-guard refactor --lang rust --pattern '...' --rewrite '...' --dir ./src --dry-run

# 2. Apply with gates (the CLI validates itself)
ast-guard refactor --lang rust --pattern '...' --rewrite '...' --dir ./src --require '<new>' --forbid '<old>'

# 3. Recovery if needed
git restore --source=HEAD -- <paths>
```

If `ast-guard` is not yet built (you're writing v1 code), use `ast-grep` directly — the Phase 1 skill workflow.

## Traits, Mocks, and Tests

- **`Compiler` trait:** Pure methods only (`command`, `structured_output_flag`, `parse_diagnostics`). `parse_diagnostics` takes `&[u8]` (not `&str`) — machine-readable output may not be valid UTF-8.
- **`Runner` trait:** The only I/O seam. `RealRunner` (production) handles `std::process::Command`. `StubRunner` (test) holds canned `RunOutput`.
- **Pipeline tests:** Inject `StubRunner` with canned compiler output. Drive the full pipeline (re-parse → require/forbid → compile → filter) without spawning real processes.
- **Golden tests:** Test `filter_diagnostics` with real JSON captures from `cargo check --message-format=json`, `tsc --noEmit --pretty false`, etc. Fixtures live in `tests/fixtures/`.
- **Real-compiler smoke tests:** `#[ignore]` — run in CI with real `cargo`/`dotnet`/`tsc`/`dart` installed.
- **No mutation-testing gate** in v1. The core modules are tested through integration tests with `StubRunner`. `cargo-mutants` is a Phase 3 quality tool.

## Design Decisions (the "why" behind the code)

### D1: No own rewrite engine
All pattern logic is delegated to `ast-grep-core`. ast-guard is an orchestrator.

**Consequence:** `pipeline.rs` does not implement matching or substitution. It calls `ast-grep-core` and then runs the gates.

### D2: Structured compiler output, not stderr-regex
All compilers are invoked with their machine-readable flag. `Compiler::parse_diagnostics` is a pure JSON/structured parser. No regex on stderr.

**Consequence:** When adding a new language, you write a JSON/structured-line parser for that compiler's output format. You do not write a regex for its stderr.

### D3: Compile root is separate from `--dir`
`--dir` (rewrite target) ≠ compile root (where `Cargo.toml`/`tsconfig.json` lives). The CLI auto-discovers by walking up from `--dir`.

**Consequence:** `dispatch.rs` contains the auto-discovery logic. `Runner` receives the compile root as `cwd`, not `--dir`.

### D4: `--require`/`--forbid` are delta-scoped
Post-conditions run on the files/sites that `--pattern` matched, not the entire `--dir`. `--scope dir` expands `--forbid` to the full directory.

**Consequence:** The pipeline needs two scopes: matched sites (default) and full directory (opt-in). Tests must verify both.

### D5: No git rollback
Recovery is taught via `init`, not implemented in the tool.

**Consequence:** The pipeline never touches git. It writes to disk sequentially and exits. If a compile fails, the files remain mutated — the agent recovers with git.

### D6: Init uses marker blocks
`init` inserts content between `<!-- ast-guard:start -->` / `<!-- ast-guard:end -->` markers, preserving existing file content outside those markers. `--force` overwrites entirely.

**Consequence:** `init.rs` implements marker-block insertion logic. Tests verify that existing content is preserved and that `--force` overwrites.

## Security

- **No shell:** `Command::arg()` only, never `sh -c`. Patterns and rewrites are data, not shell strings.
- **Absolute paths:** `which` resolves binaries to absolute paths before spawning.
- **No sandbox claim:** ast-guard runs the user's compilers in the user's repo with the user's environment. Defenses are about robustness, not isolation.

## Commit Conventions (soft)

- **Preferred format:** `type(scope): description` — e.g., `feat(pipeline): add delta-scoped --require/--forbid`.
- **No enforcement hook.** No regex hook, no CI check. It's a soft rule — respected, not enforced.
- **Accepted types:** `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `build`, `ci`, `perf`, `style`.

## CI

```bash
cargo fmt --check   # format check
cargo clippy -- -D warnings  # lint gate
cargo test          # all tests (unit + golden + StubRunner integration)
```

No cargo-husky, no cargo-mutants, no conventional-commit enforcement in v1.

## Code Style

- **Error handling:** `anyhow` for application errors, `thiserror` for library types.
- **Traits:** No async in v1. No `Box<dyn>`. Traits are constructor-injected, not dynamically dispatched.
- **Modules:** Flat structure — `compiler/` is a directory with `mod.rs` + one file per language. No deep nesting.
- **Tests:** `#[cfg(test)]` modules inside source files for unit tests. `tests/` directory for integration and golden tests.

## Dependencies

- `ast-grep-core` + `ast-grep-language` — embedded as libraries, not CLI tools. All pattern matching goes through these crates.
- `clap` (derive feature) — subcommand API.
- `which` — binary resolution for compiler absolute paths.
- `anyhow` / `thiserror` — error handling.
- `tempfile` (dev) — fixture directories for tests.
- `assert_cmd` (dev) — CLI end-to-end tests.