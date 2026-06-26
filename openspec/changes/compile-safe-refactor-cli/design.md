## Context

The ast-guard repository is greenfield: a README, an empty OpenSpec scaffold, and a Rust-tuned `.gitignore`. No code yet. This design governs the first implementation.

The tool's job: let an AI coding agent express a structural code rewrite as an `ast-grep` pattern + rewrite template, apply it across a directory, and validate the result against the real compiler — returning only the signal the agent needs (filtered diagnostics, not raw stderr). The agent never writes `sed`/`awk`/throwaway scripts for typed-language refactors; it writes `ast-guard` commands.

The agent is the primary user. Human ergonomics matter second. This shapes everything: flat CLI flags, just-in-time hints, failure messages that point at offending sites, and a deliberate refusal to do things (git rollback, auto-format) that would silently invalidate the agent's model of the working tree.

**Important:** ast-guard is not a reimplementation of ast-grep. It is a thin orchestrator around `ast-grep-core` (embedded as a library) + `cargo check` / `dotnet build` / `tsc --noEmit`. All pattern-matching, substitution, and rewrite logic is delegated to ast-grep. ast-guard adds: the re-parse gate, the `--require`/`--forbid` post-conditions, the structured compiler diagnostics pipeline, and the agent teaching (`init`). This keeps the scope lean and the value proposition clear.

Current state of agent-driven refactoring in the target workflows: already happening, already breaking. The failure mode is silent — a refactor that *looks* complete but leaves stragglers the compiler won't catch (e.g., 47 of 48 `console.log` calls replaced). That class of bug is what `--require`/`--forbid` exists to kill.

## Goals / Non-Goals

**Goals:**

- A staged safety pipeline that runs the cheapest validation first: syntactic (tree-sitter re-parse) → structural invariants (`--require`/`--forbid`) → semantic (compiler). A failure at the syntactic or structural tier aborts *before* disk mutation.
- Pluggable, testable compiler integration: two trait seams (`Compiler` Strategy, `Runner` Adapter), pure modules for core logic, no DI framework.
- A secure posture for spawning external compilers: no shell, absolute-path resolution, timeout + kill, output cap, opt-in offline flags.
- LLM-ergonomic CLI: flags reuse `--pattern`'s grammar; advisory hints fire on `--dry-run`; failure messages locate the offending site.
- A git-based recovery story taught to the agent via `init`, replacing any in-tool restore/rollback subsystem.
- **Structured compiler diagnostics (JSON/machine-readable), not stderr-regex.** All compilers are invoked with their structured-output flag; `Compiler::parse` is a pure JSON/line parser, not a regex scraper. This eliminates the "regex rots as compilers evolve" risk class.
- **Compile-root auto-discovery.** `--dir` (rewrite target) and the compile root (where `Cargo.toml`/`tsconfig.json`/`*.csproj` lives) are separate concepts. The CLI auto-discovers the compile root; `--project` overrides.
- **`--require`/`--forbid` delta scope.** Post-conditions run on the files/sites `--pattern` matched, not the entire `--dir`. This prevents false positives from pre-existing matches in untouched files.
- A lightweight project hygiene: `cargo test` + `cargo fmt --check` + `cargo clippy`. No mutation-testing merge-gate, no conventional-commit hooks in v1.

**Non-Goals:**

- No restore/rollback/snapshot subsystem. Recovery is delegated to git and taught, not implemented.
- No MCP/JSON-RPC server. The CLI *is* the interface.
- No type-aware matching. ast-grep is syntactic; the compiler is the semantic layer.
- No auto-format by default. `--fmt` is opt-in.
- No Python in v1 (rust/csharp/typescript/dart only).
- No sandbox claim.
- No own rewrite/regex engine. All matching and substitution is delegated to `ast-grep-core`.
- No regex-based stderr parsing. Compiler output is consumed via structured formats.
- No mutation-testing merge-gate. Mutants are a future-phase quality tool, not a v1 gate.

## Decisions

### D1: Staged pipeline with three safety tiers, cheapest first

```
  load pattern ─▶ ast-grep validates pattern syntax ─▶ ERR? exit ≠0
       │
       ▼
  walk --dir, parse each file (CST) ─▶ match --pattern
       │
       ├─ 0 matches → report, exit 0 (no-op)
       │
       ▼
  substitute --rewrite (template + $$$META$$$ captures)
       │
       ▼
  ┌─ RE-PARSE (tree-sitter, per language) ────────────┐
  │  SYNTACTIC gate, always-on (--unsafe-skip-reparse)│
  └───┬───────────────────────────────────────────────┘
      │ fail → "invalid syntax at L/C" → exit ≠0, NO write
      ▼
  ┌─ --require / --forbid (post-conditions) ──────────┐
  │  STRUCTURAL invariants on the rewritten source      │
  │  Scope: files/sites that --pattern matched         │
  └───┬───────────────────────────────────────────────┘
      │ fail → name the offending site → exit ≠0, NO write
      ▼
  --dry-run? ──▶ print unified diff + parse-status + hints ─▶ exit 0
      │
      ▼
  write files (std::fs)
      │
  --fmt? ──▶ cargo fmt / dotnet format / prettier   (opt-in)
      │
      ▼
  --no-build? ──▶ exit 0  (agent runs `check` later)
      │
      ▼
  dispatch: pick Compiler(lang) + Runner
      │
      ├─ Runner: spawn (absolute path, cwd=compile_root, timeout, output cap, structured flag)
      │
      ▼
  Compiler.parse_diagnostics(structured_output) → Vec<RawDiagnostic>  (pure, JSON/line parser)
      │
      ├─ filter_diagnostics(raw, severity) → Vec<Diagnostic>           (central, uniform)
      │
      ├─ empty + exit 0 ─▶ "ok"
      └─ non-empty       ─▶ print filtered diagnostics, exit ≠0
                            (files stay mutated; agent uses git recovery)
```

**Why three tiers, cheapest first.** A malformed rewrite template (unbalanced braces, bad capture splice) is caught by re-parse in milliseconds, locally, with no external process — instead of round-tripping through `cargo check` just to report `error: expected }`. `--require`/`--forbid` catches the *silent-straggler* class (parses + compiles but semantically wrong: 47 of 48 sites replaced) that no later tier catches. The compiler is the semantic backstop, run last because it's the most expensive.

**Why re-parse is always-on with an opt-out.** A safety default you can retreat from is better than an opt-in you forget to enable. `--unsafe-skip-reparse` exists for the rare agent doing intentional mid-refactor breakage.

**Why `--require`/`--forbid` run *before* write.** They are invariants on the rewritten *source*, independent of disk and compiler. Failing them aborts before mutation, which is the whole point.

**Why the compiler runs *after* write.** Compiling in-memory would require ast-guard to model each compiler's build graph, which is out of scope and fragile. The compiler owns its own incremental build state; ast-guard writes and lets the compiler do what it's good at. The trade-off: a compile failure leaves mutated files on disk. That's accepted and *taught* — the agent recovers with git (see D5).

### D2: Two trait seams — `Compiler` (Strategy) and `Runner` (Adapter)

Poor-man's DI in Rust: traits as seams, injected via constructor args, no container, no lifecycle.

```
  VARIES per language (Strategy: Compiler trait)
  ─────────────────────────────────────────────
   • how to build the command (cargo check vs dotnet build vs tsc)
   • the structured-output flag for that compiler
   • how to parse that compiler's machine-readable output (JSON, structured lines)
   • which offline flag flavor exists (--offline / --no-restore / n/a)

  UNIFORM across languages (Adapter: Runner trait)
  ────────────────────────────────────────────────
   • spawn process, set cwd/env, timeout, kill on expiry
   • capture stdout/stderr with size cap
   • always append the structured-output flag to the command
   • report exit code
   • the ONLY place std::process::Command is touched

  DISPATCH (Factory + wiring)
  ──────────────────────────
   • Language::Rust → RustCompiler, etc.
   • wires (Compiler, Runner) together, returns Vec<Diagnostic>
   • auto-discovers compile root from --dir
```

Shape:

```
  trait Runner {
      fn run(&self, spec: CommandSpec) -> Result<RunOutput>;
  }
  struct RealRunner { timeout: Duration, max_output: usize }
  #[cfg(test)] struct StubRunner { canned: Vec<RunOutput> }

  trait Compiler {
      fn language(&self) -> Language;
      fn command(&self, ctx: &CompileCtx) -> CommandSpec;             // knows the flags
      fn structured_output_flag(&self) -> &'static str;                // e.g. "--message-format=json"
      fn parse_diagnostics(&self, raw: &[u8]) -> Result<Vec<RawDiagnostic>>;  // pure JSON/line parser
  }
```

**Why two seams, not one.** Splitting "what varies per language" (Compiler) from "what's uniform I/O" (Runner) makes the pure logic testable without spawning anything. `Compiler::parse_diagnostics` is a pure parser — golden-testable with real JSON captures. `Runner` is the only I/O seam; integration tests inject `StubRunner` and drive the pipeline with canned compiler output.

**Why `structured_output_flag()` is a separate method.** The runner appends this flag to every command invocation — the compiler is never called in human-readable mode. This keeps the parsing logic clean (no regex), and the flag is a stable, documented part of each compiler's interface.

**Why `parse_diagnostics` takes `&[u8]`, not `&str`.** Machine-readable output is often UTF-8 JSON or structured lines — not guaranteed to be valid UTF-8 in all edge cases (e.g., BOM, non-standard encodings from non-English locales). Taking bytes is robust and deferential to the actual output format.

### D3: `--require` / `--forbid` — structural post-conditions, delta-scoped

Semantics: after substitution + re-parse, match each `--require` pattern against the rewritten source (must match ≥1 site); match each `--forbid` pattern (must match 0 sites). **Both run on the files/sites `--pattern` matched before the rewrite, not the entire `--dir`.**

This is the delta scope: if `--pattern` matched in `src/a.rs` and `src/b.rs`, `--forbid` only checks those two files — a pre-existing `old_api()` call in `src/unrelated.rs` (never touched by the rewrite) does not cause a failure.

```
  --require <pattern>   : "this pattern MUST exist in the rewritten files"  (positivity)
  --forbid  <pattern>   : "this pattern MUST be absent from the rewritten files" (completeness)
  --scope   forbid|dir   : forbid scope: forbid=only matched sites (default), dir=entire --dir
```

**Why delta scope is the default.** An agent refactoring `src/models/` does not care about `src/legacy/` — that file was never in scope. Making `--forbid` scan the entire `--dir` would produce false positives from pre-existing, unrelated code. The delta scope is the *only* correct semantics for a batch-refactor workflow.

**Why `--require`/`--forbid` over `assert`/`assert-not`.** `require`/`forbid` are genuine antonyms (RFC 2119 MUST/MUST NOT vocabulary) describing constraints, not xUnit boolean assertions. The positive/negative pair encodes the refactoring's invariant: "the new form exists AND the old form is gone." `--forbid` is the high-value one — it catches the silent-straggler class (1 of 48 sites un-rewritten) that parses and compiles fine.

**Why the flags reuse `--pattern`'s grammar.** Zero new syntax for the LLM to learn. If it can write `--pattern 'console.log($$$A)'`, it can write `--forbid 'console.log($$$A)'`. Skill transfers directly.

**Why flat repeated flags, not compound syntax.** Rejected `--assert '!pattern'` (overloads the pattern grammar) and `--constraint '+a,-b'` (forces comma-delimiting inside a language that already uses `$` and `{}`). Repeated flat flags are the lowest-cognitive-load form for an LLM to emit correctly.

**Why `--scope` exists.** The delta scope is correct for refactor workflows. But CI/global validation ("absolutely no `unsafe` in this crate") needs the full `--dir` scan. `--scope dir` gives that power explicitly, not as a default.

### D4: Advisory hints on `--dry-run` only — the warm contract

The system prompt is a *cold contract* that degrades as the context window fills. Hints are the *warm*, just-in-time contract that fires at the moment of decision, when the LLM has actually drifted from the playbook. They are not redundant with `init`; they are the redundancy that survives context pressure.

**Placement: dry-run only, never on apply.** Dry-run is the agent's receptive "thinking" step. On apply the agent has committed; hints there are noise burying the actual result.

**Trigger: structural signal in the rewrite, not intent heuristics.** ast-grep can tell what the rewrite *does to named nodes*; that's the reliable signal.

| Rewrite shape | Hint |
|---|---|
| changes an identifier (rename) | suggest `--forbid <old>` + `--require <new>` |
| deletes a node | suggest `--forbid <old>` (verify gone) |
| wraps a node | suggest `--require <new-wrapper>` (no `--forbid` — old form may coexist) |
| pure addition | no hint (no straggler risk) |

**Hard rule: hints advise *categories*, never synthesize *content*.** Auto-synthesized "helpful" patterns that are wrong often enough train the LLM to distrust the tool.

**`--quiet-hints` opt-out.** Default on; CI or expert agents suppress.

### D5: No restore — git-based recovery taught via `init`

A scoped restore is, mechanically, a partial reimplementation of git with worse battle-testing. The failure modes stack: crash mid-write, concurrent edit, on-disk state persistence, generation stacks, git-index coupling. A flaky restore is *worse than no restore* because it manufactures false confidence.

The agent already has git. `init` ships a recovery playbook teaching the surgical tool — `git restore --source=<ref> -- <paths>` — which is the *scoped restore we wanted*, implemented by git, battle-tested, with zero ast-guard test surface.

**Two teaching points agents get wrong, captured in the playbook:**
- Prefer a commit over a stash when the tree is clean-ish.
- `git restore --source=<ref> -- <paths>` is the surgical tool — only specific files from a ref. `git checkout -- .` is nuclear; never use it.

### D6: Security posture — robustness, not sandbox

Threat model: *the agent passes weird patterns, or runs in a misconfigured/stale environment* — not "untrusted attacker controls inputs."

| Defense | Why |
|---|---|
| No shell, ever | `Command::arg()`, never `sh -c`. |
| Resolve binary to absolute path, log it | `which` crate → spawn absolute path. Kills PATH-shadowing surprises. |
| Explicit cwd, canonicalized | `Command::current_dir(canonicalize(compile_root))`. Not `--dir` — that is the rewrite target, not the compile root. |
| Timeout + kill on expiry | A wedged `dotnet build` must not hang the agent. |
| Output size cap | Stop reading after N MB. Real errors are small. |
| Offline flags opt-in | `--offline` → `cargo check --offline` / `dotnet build --no-restore`. |
| Closed stdin | Don't inherit stdin; compilers don't need it. |

### D7: `--no-build` + standalone `check` — compile is decoupled, not removed

Without compile, ast-guard is just ast-grep with a wrapper and the "compile-safe" branding is hollow — the compiler is the semantic layer. So compile stays in the tool. The question is *coupling*:

- `refactor` **defaults to compiling** (safe-by-default).
- `--no-build` skips compile (agent batching N refactors pays compile once).
- `check` is a standalone compile + filter (the batching partner, reusable in CI).
- The diagnostics logic lives in one place (`filter_diagnostics`) and serves both.

**`--no-build` skips the expensive external semantic check; the cheap local syntactic check (re-parse) and structural checks (`--require`/`--forbid`) still run.**

### D8: Compile root auto-discovery — `--dir` is not the compile root

**The problem:** `--dir` specifies which files to rewrite. But `cargo check` runs at the crate root (where `Cargo.toml` lives). If the agent runs `ast-guard refactor --dir ./src/models`, cargo check must run in the parent directory that contains `Cargo.toml`, not in `./src/models`.

**The solution:**
- `--dir` (default `.`): the directory of files to rewrite.
- `--project` (optional override): the compile root. If not set, auto-discover by walking up from `--dir`.
- Auto-discovery: walk parent directories from `--dir` until finding a language-specific manifest (`Cargo.toml`, `tsconfig.json`, `*.csproj`, `pubspec.yaml`). The first parent containing one is the compile root.
- If no manifest is found up to `/`: exit with error, suggest `--project`.

**Why this is a correctness requirement, not a nice-to-have.** Without this, the default Rust path (`--dir ./src` → compile in `./src` → no `Cargo.toml`) is **guaranteed broken** for the most common agent workflow. This is the single biggest correctness bug in the original design.

### D9: `init` writes two things, to agent-specific paths

`init` writes (1) the CRITICAL RULE (use ast-guard, not sed) and (2) the git recovery playbook. Target files:

| `--target` | Path |
|---|---|
| `agents` (default) | `AGENTS.md` |
| `copilot` | `.github/copilot-instructions.md` |
| `claude` | `CLAUDE.md` |
| `pi` | `.pi/prompts/ast-guard.md` |
| `all` | all of the above |

Templates are embedded via `include_str!` and live under `src/agents/`. The "CRITICAL RULE" text is *not* hardcoded in binary logic — it's read from a template the user can edit post-init.

**Important:** `init` writes using **marker blocks**, not full-file-overwrite. If a target file already exists (e.g., a pre-existing `AGENTS.md`), `init` inserts its content between `<!-- ast-guard:start -->` and `<!-- ast-guard:end -->` markers, preserving existing content outside those markers. If the file does not exist, it creates it with the markers and content. This prevents clobbering user-customized files on repeated `init` runs.

### D10: Project hygiene — lightweight, no cargo-husky/mutants gates

- **CI:** `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test`.
- **No cargo-husky** in v1. Git hooks are a future nice-to-have.
- **No cargo-mutants merge-gate** in v1. Mutation testing is a quality tool for mature codebases, not a v1 requirement. The core modules (`reparse`, `filter`) are tested through integration tests with `StubRunner`.
- **Conventional commits:** soft rule in `AGENTS.md` only. No regex hook, no CI enforcement.
- **`AGENTS.md`** at the repository root: agent-facing guide to contributing to ast-guard. Covers architecture, trait seams, the dry-run-first workflow for refactors *within* ast-guard, and the security/non-sandbox posture. Distinct from the `init`-shipped templates (which govern *users* of ast-guard in their own projects).

## Risks / Trade-offs

- **[Re-parse false-negatives from tree-sitter grammar gaps]** → Mitigation: `--unsafe-skip-reparse` escape hatch. If re-parse proves unstable, the default flips without rearchitecting.
- **[Compiler runs after write, leaving mutated files on compile failure]** → Accepted by design. Mitigation: `init` playbook teaches `git restore`.
- **[`--forbid` patterns too narrow, miss formatting variants]** → Same skill as writing `--pattern`. Failure messages name the offending site.
- **[Hints add noise on expert/CI runs]** → Mitigation: `--quiet-hints`; default-on but escapable.
- **[Structured output format changes across compiler versions]** → Mitigation: the structured format is a documented, stable interface (e.g., cargo's JSON is versioned). Golden tests with real captures per supported version. This risk is *dramatically lower* than "regex rots".
- **[Output size cap hides a real cascade of errors]** → Mitigation: emit a clear "output truncated" notice. Real errors are small.
- **[No restore means an agent that panics mid-batch leaves the tree dirty]** → Accepted. Mitigation: workflow taught in `init`.
- **[Offline flag mismatch across languages]** → `--offline` is a no-op with a note for languages that don't support it.
- **[Compile root auto-discovery fails in monorepos with multiple manifests]** → Mitigation: `--project` override. The agent specifies the exact crate/project root.
- **[Init marker-block insertion fails on malformed existing files]** → Mitigation: if the file exists but marker insert fails, `init` warns and skips that file. `--force` overwrites.

## Open Questions

- **Supported compiler version matrix.** Which exact `cargo`/`dotnet`/`tsc`/`dart` versions are v1-supported? Affects golden-test fixtures. Decide at task-start by checking what's installed in CI.
- **`--no-incremental` for C#?** Defer: default to incremental dotnet in v1, revisit if false-positives observed.
- **`--warn-as-error` in v1 or later?** Leaning later.
- **Hint trigger for "wraps a node" — is the structural signal reliable enough?** May ship rename+delete hints first and add wrap later.
- **Phase 3 scope: rule-file support (`--rule-file <yaml>`)?** Defer to Phase 3. In Phase 2, the single `--pattern` + `--rewrite` covers the 80% case. Complex relational rules (ast-grep's `inside`/`has`/`not`) need YAML and are a Phase 3 feature if demand proves it.
- **Phase 3 scope: mutation-testing gate (`cargo-mutants`)?** Defer to Phase 3. The core modules (`reparse`, `filter`) are tested through integration tests with `StubRunner`. Mutation testing adds value when the codebase matures.
- **Skill-only Phase 1: does the `ast-guard-skills.md` prompt eliminate the need for the CLI?** Possibly — but the hard `--forbid` gate (not advisory) and the structured diagnostics (not agent-written `jq` filters) are the CLI's unique value. The skill validates the workflow; the CLI enforces it. Phase 1 (skill) ships first to validate demand; Phase 2 (CLI) builds if agents consistently skip validation steps or produce broken `jq` filters.