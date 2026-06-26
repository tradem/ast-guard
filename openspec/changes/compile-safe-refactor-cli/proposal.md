## Why

AI coding agents performing large-scale refactorings in strongly-typed languages (Rust, C#, TypeScript, Dart) default to regex/script-based text replacement (`sed`, `awk`, throwaway Python). This routinely breaks ASTs, destroys async contexts, misaligns brackets, and produces compile errors the agent then has to untangle.

There is no tool that lets an agent express a structural rewrite, validate it against the real compiler, and get back only the signal it needs — so agents keep reaching for `sed`. We need that tool because agent-driven refactoring is already happening in this codebase's target workflows, and the failure mode is silent: a refactor that *looks* complete but leaves stragglers the compiler won't catch.

**However:** a full Rust CLI is over-engineered for the initial need. The pragmatic approach is a three-phase delivery:

1. **Phase 1: Agent skill** — a Markdown prompt the agent reads and follows, teaching it to use `ast-grep` (`sg`) directly + `cargo check` + structured validation steps. Ships today as a 200-line markdown document.
2. **Phase 2: Lightweight CLI** (this change) — a thin Rust orchestrator that automates the skill's manual steps, adds hard `--require`/`--forbid` gates (not advisory), and delivers filtered, structured compiler diagnostics. The CLI is *not* a platform — it's a 300-line wrapper around `ast-grep-core` + `cargo check`.
3. **Phase 3: Mature CLI** — expanded language support, rule-file support for complex ast-grep YAML rules, auto-discovered compile roots, and CI integration. Built only if Phase 2 proves the need in real agent workflows.

## What Changes (Phase 2: Lightweight CLI)

- Introduce the `ast-guard` Rust CLI with three subcommands: `refactor`, `check`, `init`.
- `refactor`: thin pipeline orchestrator around `ast-grep-core`:
  - `sg scan` for match discovery (not reimplemented — delegating to ast-grep)
  - `sg run` for substitution + rewrite (not reimplemented)
  - **re-parse gate** (tree-sitter, always-on) — catches malformed rewrites before disk mutation
  - **`--require`/`--forbid` post-conditions** (delta-scope on matched sites) — the unique, hard safety gate
  - `--dry-run` with diff preview + advisory hints
  - Optional `--fmt` (cargo fmt / dotnet format / prettier, opt-in)
  - Optional `--no-build` + `check` as batching partner
- `check`: standalone compiler invocation with **structured diagnostics** (JSON/machine-readable, NOT stderr-regex):
  - Invokes compiler with language-specific structured-output flag (`--message-format=json`, `--pretty false`, etc.)
  - Parses structured output into `Diagnostic` model (file, span, severity, code, message)
  - Central `filter_diagnostics(raw, severity)` — uniform across languages, no per-compiler regex
  - Reusable in CI and agent loops; the batching partner to `refactor --no-build`
- `init`: writes agent-integration prompt files (AGENTS.md / Copilot / Claude / pi / all) containing:
  - The CRITICAL RULE (use ast-guard, not sed) for languages where the binary is available
  - The git-based recovery playbook (surgical `git restore`, not nuclear `git checkout -- .`)
  - The dry-run-first, check-before-commit workflow
- **No restore/rollback feature.** ast-guard deliberately does not touch git or snapshot state. Recovery is delegated to git and *taught to the agent* via `init`. This removes a whole subsystem of subtle correctness bugs and avoids clobbering the agent's/user's own git strategy.
- Three-tier safety model: syntactic (re-parse, cheap, local, always-on) → structural invariants (`--require`/`--forbid`) → semantic (compiler). Cheapest tier runs first; a failure at any tier aborts before disk mutation (except the compiler tier, which runs after write by design).
- **No own rewrite engine.** `refactor` delegates all pattern-matching and substitution to `ast-grep-core` (embedded as a library). ast-guard is an orchestrator, not a reimplementation of ast-grep.
- **Structured compiler output, not stderr-regex.** All compiler invocations use the target compiler's machine-readable format (JSON, structured lines). `Compiler::parse` is a pure JSON/structured parser, not a regex scraper. This eliminates the entire "regex rots as compilers evolve" risk.
- **Compile-root auto-discovery.** `--dir` (rewrite target) is separate from the compile root. The CLI walks up from `--dir` to find `Cargo.toml`/`tsconfig.json`/`*.csproj`/`pubspec.yaml`. If not found, `--project` overrides. This prevents the primary correctness bug (cargo check in `./src` instead of crate root).
- **`--require`/`--forbid` delta scope.** Post-conditions run only on the sites/files that `--pattern` matched before the rewrite — not the entire `--dir`. This prevents false positives from pre-existing, unrelated matches in untouched files.
- LLM ergonomics: flat flags reusing `--pattern`'s grammar, just-in-time advisory hints on `--dry-run`, and failure messages that point at the offending site.
- Pluggable compiler integration via two trait seams (`Compiler` Strategy per language, `Runner` Adapter over `std::process::Command`) — poor-man's DI, no framework, mockable for tests.
- Security posture for external-tool calls: no shell, absolute-path resolution, timeout + kill, output size cap, opt-in offline flags.
- **Project hygiene is lightweight.** No cargo-husky, no cargo-mutants merge-gate, no conventional-commit hooks in v1. The project uses standard `cargo test` + `cargo fmt --check` + `cargo clippy`. Mutation testing and commit conventions are Phase 3 additions.

### Non-goals (explicit)

- No automatic git rollback or snapshot/restore subsystem. Recovery is the agent's job, taught via `init`.
- No MCP/JSON-RPC server. The CLI is the interface; token efficiency comes from a flat CLI surface, not a wire protocol.
- No type-aware matching. ast-grep is syntactic; the compiler is the semantic layer. ast-guard does not duplicate semantic analysis.
- No auto-formatting by default. `--fmt` is opt-in because formatting silently invalidates the agent's mental model of the file.
- No Python support in v1. Supported languages: rust, csharp, typescript, dart.
- No own rewrite engine. All pattern-matching and substitution is delegated to `ast-grep-core`.
- No stderr-regex-based diagnostics. Compiler output is consumed via structured, machine-readable formats.
- No mutation-testing merge-gate. `cargo-mutants` is a nice-to-have for future phases, not a v1 requirement.
- No conventional-commit enforcement hooks. Conventional commits are a soft convention, not a git-hook gate.

## Capabilities

### New Capabilities (Phase 2)

- `refactor`: structural find-and-replace with a staged safety pipeline (re-parse gate, delta-scoped `--require`/`--forbid` post-conditions, dry-run, optional fmt, optional compile, **structured** diagnostics).
- `check`: standalone compiler invocation with **structured diagnostics** (JSON/machine-readable parsing, not stderr-regex), decoupled from rewriting so refactors can be batched under `--no-build` and validated once.
- `init`: write agent-integration prompt files (CRITICAL RULE + git recovery playbook) for supported agent targets.
- `compiler-integration`: the pluggable, testable, secure wrapper around external compilers (`Compiler`/`Runner` trait seams, dispatch by language, structured output parsing into `Diagnostic`s).
- `project-hygiene`: the ast-guard codebase's own **lightweight** tooling — standard cargo test/fmt/clippy, a project-specific AGENTS.md that governs agent contributions to ast-guard, and a minimal CI pipeline.

### Modified Capabilities

(None — greenfield repository, no existing specs.)

## Impact

- **New code**: minimal `src/` tree (cli, pipeline, reparse, dispatch, runner, compiler/{rust,csharp,typescript,dart}, filter, init, agents templates), `tests/` (integration + filter-golden with JSON fixtures), `fixtures/` (sample crates/projects per language), `Cargo.toml` with `clap`, `ast-grep-core`, `ast-grep-language`, `which`.
- **New project files**: `AGENTS.md` (agent-facing guide to hacking on ast-guard), `.gitignore` additions (`.ast-guard/` reserved).
- **Dependencies**: `ast-grep-core` + `ast-grep-language` (embedded as libraries), `clap` (subcommand API), `which` (binary resolution). No dev-deps beyond `tempfile`, `assert_cmd` for integration tests. No cargo-husky, no cargo-mutants in v1.
- **External tools orchestrated** (not bundled): `cargo`/`rustc`, `dotnet`, `tsc`/`npx`, language formatters (`cargo fmt`, `dotnet format`, `prettier`). ast-guard resolves these from PATH at runtime; absence is a clear error, not a crash.
- **README**: must be updated to match the no-rollback stance and the three-phase delivery strategy (currently advertises "automatic git rollbacks" — that claim is removed and replaced with the git-recovery-playbook delegation). Must also reflect the structured-diagnostics approach and the lean scope.
- **No breaking changes** (greenfield).