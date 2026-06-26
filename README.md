# ast-guard

A compile-safe AST refactoring tool for AI coding agents. Safely execute structural rewrites with automatic compiler validation — and a git-based recovery playbook when things go wrong.

## What it does

When AI coding agents attempt large-scale refactorings in strongly-typed languages (Rust, C#, TypeScript, Dart), they default to fragile regex or script-based text replacements that break syntax, misalign brackets, and destroy type safety. The failure mode is silent: a refactor that *looks* complete but leaves stragglers the compiler won't catch (e.g., 47 of 48 sites replaced — the 48th survives silently).

**ast-guard** solves this by providing a thin, test-driven refactoring pipeline. It delegates pattern-matching and substitution to `ast-grep-core`, then layers on:

1. **A re-parse gate** (tree-sitter, always-on) — catches malformed rewrites before they touch the disk.
2. **Hard `--require`/`--forbid` post-conditions** — guarantees that the new form exists AND the old form is gone. Not advisory — enforced.
3. **Structured compiler validation** — invokes `cargo check` / `dotnet build` / `tsc --noEmit` with machine-readable output (JSON, not stderr-regex), filters diagnostics centrally, and returns only the signal the agent needs.
4. **An agent teaching system** (`init`) — ships the CRITICAL RULE (use ast-guard, not sed) and a git-based recovery playbook so the agent never loses work.

**ast-guard does not perform git rollbacks.** Recovery is taught, not automated — `init` equips agents with `git restore --source=<ref> -- <paths>`, the battle-tested surgical tool.

## The three-phase delivery strategy

ast-guard is built in three phases, each validating demand before the next:

### Phase 1: Agent skill (ship today)

A standalone [Markdown prompt](docs/ast-guard-skills.md) that teaches AI coding agents to use `ast-grep` (`sg scan`/`sg run`) + `cargo check --message-format=json` + manual straggler checks. No binary needed — just a 200-line document the agent reads.

**Deliverable:** `docs/ast-guard-skills.md` — a prompt that says "don't use sed for typed-language refactors; here's the ast-grep workflow."

### Phase 2: Lightweight CLI (this repo)

A thin Rust orchestrator (this project) that automates the skill's manual steps:

- **`refactor`**: Delegates to `ast-grep-core` for pattern matching + substitution. Adds the re-parse gate, delta-scoped `--require`/`--forbid` post-conditions, optional dry-run with hints, and optional compile with **structured** (JSON) diagnostics.
- **`check`**: Standalone compiler invocation with structured outputs — the batching partner to `refactor --no-build`.
- **`init`**: Writes agent-integration prompt files (AGENTS.md / Copilot / Claude / pi) with the CRITICAL RULE and git recovery playbook. Uses marker blocks to preserve existing content.

The CLI is **not** a reimplementation of ast-grep. It is a pipeline orchestrator — ~500 lines of Rust wrapping `ast-grep-core` + `cargo check`.

### Phase 3: Mature CLI (if demand proves)

Expanded language support (Python, where `--require`/`--forbid` is the primary safety layer), rule-file support for complex ast-grep YAML rules (`inside`/`has`/`not`), CI-specific modes, and optional mutation-testing quality gates.

## Why not just use ast-grep directly?

ast-grep (`sg scan`/`sg run`) handles pattern matching and rewriting. An agent *can* use it directly — and Phase 1 teaches exactly that. The CLI adds:

- **Hard safety gates, not advisory.** `--forbid` rejects the write if old patterns remain. A prompt says "please check" — the CLI enforces.
- **Structured diagnostics.** `cargo check --message-format=json` parsed centrally — the agent sees file:line:col:span, not raw stderr. No `jq` filter to write, no regex to break on compiler updates.
- **Re-parse before disk mutation.** Catch malformed rewrites before they touch the filesystem.
- **Standardized recovery.** `init` teaches the same git recovery playbook in every repo.

For a single refactor, ast-grep + cargo-check is fine. For **hundreds of refactors per session** (the AI agent workflow), the token efficiency and enforced correctness of a single `ast-guard refactor` command is the difference between a clean migration and a silent straggler in production.

## Safety model (three tiers, cheapest first)

```
  AST PATTERN (ast-grep-core)
       ▼
  RE-PARSE (tree-sitter, always-on)     ← SYNTACTIC: catches malformed rewrites
       ▼
  --require / --forbid                    ← STRUCTURAL: catches stragglers
       ▼
  --dry-run? (diff + hints, no write)
       ▼
  WRITE (std::fs, sequential)
       ▼
  --no-build? → exit 0
       ▼
  COMPILER (structured JSON output)      ← SEMANTIC: catches type/safety errors
       ▼
  FILTERED DIAGNOSTICS (file:line:col)
```

## Key design decisions

- **No git rollback.** Recovery is delegated to git and taught via `init`. A restore subsystem would be a partial reimplementation of git with worse battle-testing and subtle concurrency bugs.
- **Structured compiler output, not stderr-regex.** Every compiler is invoked with its machine-readable flag (`--message-format=json`, `--pretty false`). `Compiler::parse` is a pure JSON/structured parser, not a regex scraper. This eliminates the "regex rots as compilers evolve" risk.
- **Compile root auto-discovery.** `--dir` (rewrite target) and the compile root are separate concepts. The CLI walks up from `--dir` to find `Cargo.toml`/`tsconfig.json`/`*.csproj`/`pubspec.yaml`. Without this, `refactor --dir ./src/models` would try to compile in `./src/models` (no Cargo.toml) — a correctness bug.
- **Delta-scoped `--require`/`--forbid`.** Post-conditions run only on the files/sites that `--pattern` matched, not the entire `--dir`. A pre-existing `old_api()` in `src/legacy/` (never touched by the rewrite) does not cause a false positive.
- **No own pattern engine.** All matching and substitution is delegated to `ast-grep-core`. ast-guard orchestrates the pipeline.
- **Hints on `--dry-run` only.** Just-in-time warnings fire when the agent is receptive (thinking step), not when they've committed (apply).
- **`--fmt` is opt-in.** Auto-formatting silently invalidates the agent's mental model of the file — it's opt-in, not default.
- **No Python in v1.** Supported languages: Rust, C#, TypeScript, Dart. Python is a Phase 3 candidate (there, `--require`/`--forbid` + re-parse is the primary safety layer — no compiler backstop).

## Usage

### Phase 2: The CLI

```bash
# Install
cargo install ast-guard

# Teach the agent (run once per repo)
ast-guard init                     # writes AGENTS.md
ast-guard init --target all        # writes AGENTS.md + .github/copilot-instructions.md + CLAUDE.md + .pi/prompts/ast-guard.md

# Refactor with safety pipeline
ast-guard refactor \
  --lang rust \
  --pattern 'fn old_name($$$A) { $$$B }' \
  --rewrite 'fn new_name($$$A) { $$$B }' \
  --dir ./src \
  --require 'fn new_name' \
  --forbid 'fn old_name' \
  --dry-run                        # preview first

ast-guard refactor \
  --lang rust \
  --pattern 'fn old_name($$$A) { $$$B }' \
  --rewrite 'fn new_name($$$A) { $$$B }' \
  --dir ./src \
  --require 'fn new_name' \
  --forbid 'fn old_name'           # apply: re-parse → require/forbid → write → compile → filtered diagnostics

# Batching workflow
ast-guard refactor \
  --lang rust --pattern '...' --rewrite '...' --dir ./src --no-build
ast-guard refactor \
  --lang rust --pattern '...' --rewrite '...' --dir ./src --no-build
ast-guard check --lang rust        # one compile for all refactors

# Check only (no refactoring)
ast-guard check --lang rust --dir ./src --severity errors
ast-guard check --lang typescript --dir ./packages --offline  # tsc --noEmit, no network
```

### Phase 1: The skill (no binary needed)

If the CLI isn't installed yet, use the [agent skill prompt](docs/ast-guard-skills.md):

```bash
sg scan -p 'pattern' -l rust --json   # find matches
sg run -p 'pattern' -r 'rewrite' -l rust  # apply
cargo check --message-format=json       # validate
sg scan -p '<old form>' -l rust --json | jq '.count'  # straggler check
```

The skill documents the full workflow. The CLI automates it.

## How it works (internal architecture)

```
  cli.rs (clap subcommands)
       │
  pipeline.rs (orchestrator)
       │
       ├─ ast-grep-core (pattern match + substitution)  ← DELEGATED
       ├─ reparse.rs (tree-sitter ERROR-node detection)  ← SYNTACTIC gate
       ├─ require/forbid (delta-scoped post-conditions)   ← STRUCTURAL gate
       │
       ▼ write
       ▼ fmt (opt-in)
       ▼ --no-build? → exit
       ▼
  dispatch.rs → pick_compiler(lang) → Compiler(command, structured_output_flag) + Runner
       │
       ├─ Compiler::parse_diagnostics(structured_output) → Vec<RawDiagnostic>  ← PURE JSON/line parser
       ├─ filter_diagnostics(raw, severity)                → Vec<Diagnostic>     ← CENTRAL, uniform
       │
       ▼ report filtered diagnostics, exit ≠0 on failure
```

- **`Compiler` trait (Strategy):** Per-language: how to build the command, which structured-output flag to use, how to parse machine-readable output. `parse_diagnostics` is pure — **no regex**.
- **`Runner` trait (Adapter):** Spawns processes (absolute path, compile root cwd, timeout, output cap, closed stdin). Always appends `structured_output_flag()`. Captures **stdout** (compilers write structured diagnostics there). The only place `std::process::Command` is touched.
- **`filter_diagnostics` (central):** One function for all languages. Parsing is per-compiler (JSON dialect); filtering is uniform (match on severity level).
- **No DI framework, no lifecycle management.** Traits as constructor args. Mockable via `StubRunner` (canned `RunOutput`) in unit tests.

## Security posture

ast-guard spawns the user's compilers in the user's repo with the user's environment. It is **not** a sandbox — it is **robust against misconfiguration**:

- **No shell, ever** — `Command::arg()` only, no `sh -c`.
- **Absolute binary paths** (via `which`) — logs which `cargo`/`dotnet`/`tsc` ran.
- **Canonicalized compile root** — not `--dir` (the rewrite target, which may not contain `Cargo.toml`).
- **Timeout + kill** — a wedged `dotnet build` won't hang the agent.
- **Output size cap** — stops reading after N MB. Real errors are small.
- **Offline flags** — opt-in `--offline` → `cargo check --offline`, `dotnet build --no-restore`.
- **Closed stdin** — compilers don't need it, avoids rare hangs.

## Non-goals

- **No git rollback** — recovery is taught, not automated.
- **No MCP/JSON-RPC server** — the CLI is the interface.
- **No type-aware matching** — ast-grep is syntactic; the compiler is the semantic layer.
- **No own rewrite engine** — all matching is delegated to `ast-grep-core`.
- **No auto-formatting by default** — `--fmt` is opt-in.
- **No Python in v1** — Rust, C#, TypeScript, Dart only.
- **No mutation-testing merge-gate** (Phase 3).

## Contributing

### For AI agents contributing to ast-guard

Read `AGENTS.md` at the repo root — it covers the architecture (trait seams, delegation to `ast-grep-core`), the three-phase delivery strategy, and the dry-run-first workflow for changes to ast-guard itself.

### For humans

```bash
cargo fmt --check
cargo clippy -- -D warnings
cargo test
```

No pre-commit hooks, no mutation-testing gates, no conventional-commit enforcement in v1. Just standard Rust tooling.

## License

MIT — see [LICENSE](LICENSE).

## Status

Phase 1 (agent skill) — shipping now.
Phase 2 (lightweight CLI) — this repo, under development.
Phase 3 (mature CLI) — conditional on Phase 2 demand.