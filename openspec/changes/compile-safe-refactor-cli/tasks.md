## 1. Project scaffold and dependencies

- [ ] 1.1 `cargo init` and set up `Cargo.toml` with deps: `clap` (derive, subcommand API), `ast-grep-core`, `ast-grep-language`, `which`, `anyhow`/`thiserror`; dev-deps: `tempfile`, `assert_cmd`. No cargo-husky, no cargo-mutants in v1.
- [ ] 1.2 Create the module tree: `main.rs`, `cli.rs`, `pipeline.rs`, `reparse.rs`, `dispatch.rs`, `runner.rs`, `compiler/{mod,rust,csharp,typescript,dart}.rs`, `filter.rs`, `init.rs`, `agents/` (templates via `include_str!`). No `pattern.rs`, no `rewrite.rs` — pattern and rewrite are delegated to `ast-grep-core`.
- [ ] 1.3 Confirm `.gitignore` has `target/`, `**/mutants.out*/` (if ever run), `*.rs.bk` (already present), `.ast-guard/` (reserved).
- [ ] 1.4 `cargo build` green; `cargo fmt --check` and `cargo clippy -- -D warnings` green.

## 2. Compiler-integration trait seams (Compiler + Runner) — structured output

- [ ] 2.1 Define `Severity` enum (`Error`, `Warning`, `Note`), `RawDiagnostic` type (severity, code, file, line, column, message, span, related), `Diagnostic` type (filtered), `Language` enum (rust, csharp, typescript, dart), `CommandSpec`, `RunOutput`, `CompileCtx`.
- [ ] 2.2 Define `Runner` trait and `RealRunner` (absolute-path resolve via `which`, canonicalized cwd as **compile root** not `--dir`, closed stdin, timeout+kill, output size cap, **always appends `structured_output_flag`** to command, captures stdout not stderr).
- [ ] 2.3 Define `StubRunner` (canned `RunOutput`) behind `#[cfg(test)]`.
- [ ] 2.4 Define `Compiler` trait: `language()`, `command(context) -> CommandSpec`, `structured_output_flag() -> &'static str`, `parse_diagnostics(raw: &[u8]) -> Result<Vec<RawDiagnostic>>` (pure JSON/structured-line parser, **no regex**).
- [ ] 2.5 Implement `RustCompiler` (`cargo check --message-format=json`, offline `--offline`), `CSharpCompiler` (`dotnet build` with structured logging, offline `--no-restore`), `TypeScriptCompiler` (`tsc --noEmit --pretty false`, offline = noted no-op), `DartCompiler` (`dart analyze --format=json`, offline = noted no-op).
- [ ] 2.6 Implement **central** `filter_diagnostics(raw: Vec<RawDiagnostic>, severity: Severity) -> Vec<Diagnostic>` — one function for all languages, not per-compiler logic. This is `filter.rs`.
- [ ] 2.7 Implement `dispatch.rs`: `pick_compiler(lang)` via exhaustive `match`; wire `Compiler` + `Runner` → `Vec<Diagnostic>`; auto-discover compile root by walking up from `--dir` to find manifest (`Cargo.toml`/`tsconfig.json`/`*.csproj`/`pubspec.yaml`); if not found, error with `--project` suggestion.
- [ ] 2.8 Binary-resolution error path: missing compiler → clear non-zero exit naming the binary + language; resolved absolute path included in output.
- [ ] 2.9 Unit tests for `dispatch` with `StubRunner`; `#[ignore]` smoke tests invoking real `cargo`/`dotnet`/`tsc`/`dart`.

## 3. Filter and diagnostics (central, uniform, no regex)

- [ ] 3.1 Implement `filter.rs`: `fn filter_diagnostics(raw: Vec<RawDiagnostic>, severity: Severity) -> Vec<Diagnostic>` — a simple `match` on severity level with `.filter()`. Uniform across languages. No per-compiler severity logic.
- [ ] 3.2 Capture structured JSON fixtures from `cargo check --message-format=json`, `tsc --noEmit --pretty false`, `dotnet build` (structured), `dart analyze --format=json` under `tests/fixtures/`.
- [ ] 3.3 Golden tests: each JSON fixture → expected `Vec<Diagnostic>`; assert severity filtering at `errors`/`warnings`/`all` using the central `filter_diagnostics`.
- [ ] 3.4 `filter.rs` is a trivial `match` + `.filter()` — **no mutation testing required** on this module.

## 4. Re-parse gate (tree-sitter ERROR-node detection)

- [ ] 4.1 `reparse.rs`: feed rewritten source through the target language's tree-sitter grammar; on failure = **ERROR nodes present in CST** (not MISSING nodes). Return error with line/column from the ERROR node.
- [ ] 4.2 Wire re-parse into pipeline after substitution, before write; on failure exit non-zero with location and DO NOT write.
- [ ] 4.3 Implement `--unsafe-skip-reparse` opt-out.
- [ ] 4.4 Tests: balanced/unbalanced braces (ERROR node detected), bad capture splice (ERROR node), valid rewrite passes, MISSING nodes alone pass (not a failure), opt-out skips.
- [ ] 4.5 Integration tests with `StubRunner` over fixture crates; `#[ignore]` real-compiler smoke tests.

## 5. --require / --forbid post-conditions (delta-scoped)

- [ ] 5.1 Implement `--require <pattern>` (repeatable): after re-parse, match against rewritten source; ≥1 match required. **Scope: only files/sites that `--pattern` matched before the rewrite.**
- [ ] 5.2 Implement `--forbid <pattern>` (repeatable): after re-parse, match against rewritten source; 0 matches required. **Scope: only files/sites that `--pattern` matched before the rewrite.** `--scope dir` expands to entire `--dir`.
- [ ] 5.3 `--scope` flag: `forbid` (default, delta) | `dir` (entire directory). Controls `--forbid` scope only; `--require` is always delta-scoped.
- [ ] 5.4 On failure: name offending pattern + first site (file:line:col), exit non-zero, DO NOT write.
- [ ] 5.5 Tests: require-present in rewritten files (passes), require-present in rewritten files but match is in untouched file (fails — requires delta scope), forbid-straggler in matched file (fails, names site), forbid-ignores-pre-existing-match-in-untouched-file (passes), multiple post-conditions all-must-hold, `--scope dir` catches forbid in untouched file.

## 6. Advisory hints (dry-run only)

- [ ] 6.1 Detect rewrite shape from ast-grep named-node diff: rename (changes identifier), delete (removes node), wrap (adds enclosing syntax), pure-addition.
- [ ] 6.2 Emit category-only hints on `--dry-run`: rename → suggest `--forbid <old>` + `--require <new>`; delete → suggest `--forbid <old>`; wrap → suggest `--require <new>` (no forbid); addition → no hint.
- [ ] 6.3 Hint format: parseable, names the trigger, gives flag *shape* (not content), names `--quiet-hints` inline.
- [ ] 6.4 Implement `--quiet-hints` opt-out; ensure hints NEVER fire on apply (non-dry-run).
- [ ] 6.5 Tests: rename hint, addition no-hint, `--quiet-hints` suppresses, no hints on apply.

## 7. refactor pipeline orchestration

- [ ] 7.1 `pipeline.rs`: sequence: load pattern (delegated to ast-grep-core) → match (delegated) → substitute (delegated) → re-parse (tree-sitter ERROR check) → require/forbid (delta-scoped) → dry-run? → write → fmt? → no-build? → compile (structured output) → filter → report.
- [ ] 7.2 `--dry-run`: print diff + re-parse status + hints, exit 0, no write. Re-parse/require/forbid failures still exit non-zero on dry-run.
- [ ] 7.3 `--fmt`: invoke language formatter after write, before compile (`cargo fmt`, `dotnet format`, `prettier`).
- [ ] 7.4 `--no-build`: exit 0 after write (and optional fmt), no compile.
- [ ] 7.5 Default: write → compile (with structured output flag) → filtered diagnostics; success → exit 0; failure → print diagnostics, exit non-zero, files remain.
- [ ] 7.6 Auto-discover compile root from `--dir` (walk up to `Cargo.toml`/`tsconfig.json`/`*.csproj`/`pubspec.yaml`); `--project` overrides.
- [ ] 7.7 `--timeout` (default 120s) and `--offline` plumbed through to `Runner`/`Compiler::command`.
- [ ] 7.8 Integration tests with `StubRunner` over fixture crates per language; `#[ignore]` real-compiler smoke tests.

## 8. check subcommand (structured output, auto-discovery)

- [ ] 8.1 `check --lang --dir`: invoke compiler with structured-output flag via `Compiler` + `Runner`, parse structured output, filter by `--severity` (default `warnings`), report; NO rewriting.
- [ ] 8.2 Auto-discover compile root from `--dir` (same walk-up logic as `refactor`); `--project` overrides.
- [ ] 8.3 `--offline`, `--timeout`, `--severity errors|warnings|all` flags.
- [ ] 8.4 Tests: clean compile, errors+warnings filtered, severity variations, offline no-op for tsc, timeout kills wedged process, auto-discovery from subdirectory.

## 9. init subcommand and agent templates (marker blocks)

- [ ] 9.1 Write `src/agents/agents.md`, `copilot.md`, `claude.md`, `pi.md` templates containing CRITICAL RULE + git recovery playbook.
- [ ] 9.2 CRITICAL RULE: prohibit sed/awk/Python for typed-language refactors; mandate ast-guard; include usage + dry-run-first workflow.
- [ ] 9.3 Git recovery playbook: `git status --porcelain` pre-batch; prefer commit over stash; `git restore --source=<ref> -- <paths>` surgical restore; prohibit `git checkout -- .`; state ast-guard does NOT rollback.
- [ ] 9.4 `init --target agents|copilot|claude|pi|all` (default `agents`); write to correct paths.
- [ ] 9.5 Implement marker-block insertion: if target file exists, insert between `<!-- ast-guard:start -->` / `<!-- ast-guard:end -->` markers, preserving existing content. If file doesn't exist, create with markers. `--force` overwrites entirely.
- [ ] 9.6 Tests: idempotent init preserves existing content, force overwrites, invalid target rejected, marker conflict emits warning and skips.

## 10. CLI wiring

- [ ] 10.1 `cli.rs`: `Commands` enum with `Refactor`, `Check`, `Init` subcommands; all flags per spec with defaults.
- [ ] 10.2 Validate `--lang` against supported set (rust, csharp, typescript, dart); reject others with supported list.
- [ ] 10.3 `main.rs`: thin dispatch from clap to subcommand handlers.
- [ ] 10.4 End-to-end CLI tests via `assert_cmd` for each subcommand's happy path and key error paths.

## 11. Documentation

- [ ] 11.1 Update README: remove "automatic git rollbacks" claim; replace with no-rollback + git-recovery-playbook stance; describe three-tier safety model, subcommands, and **three-phase delivery strategy** (skill → lightweight CLI → mature CLI). Remove "atomic" claim; clarify that ast-guard writes files sequentially and delegates recovery to git.
- [ ] 11.2 Document the **structured diagnostics approach** (JSON/machine-readable, not stderr-regex) in README.
- [ ] 11.3 Document auto-discovery of compile root (walk-up from `--dir` to `Cargo.toml`/`tsconfig.json`/`*.csproj`/`pubspec.yaml`).
- [ ] 11.4 Add per-language `--require`/`--forbid` examples (Rust: rename with straggler check, C#: method migration, TypeScript: `console.log` → logger, Dart: widget tree refactor) to README.
- [ ] 11.5 Write `ast-guard-skills.md` (Phase 1 agent skill) — a standalone Markdown prompt documenting the ast-grep + cargo-check workflow for agents that don't have the CLI installed. This is the Phase 1 deliverable, shipped alongside the README.

## 12. CI

- [ ] 12.1 GitHub Actions workflow: `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo test`.
- [ ] 12.2 Real-compiler smoke tests (`#[ignore]` set) run in CI with cargo/dotnet/tsc/dart installed.
- [ ] 12.3 No conventional-commit check, no cargo-mutants check, no cargo-husky hooks in v1 CI.

## 13. Validation gate (lightweight)

- [ ] 13.1 Full `cargo test` green (unit + golden + integration with `StubRunner`).
- [ ] 13.2 Manual end-to-end: rename refactor with `--require`/`--forbid` on a real fixture crate, dry-run then apply, confirm compile and filtered output.
- [ ] 13.3 Manual end-to-end: simulate a straggler (1 of N sites un-rewritten) and confirm `--forbid` (delta-scoped) catches it before write.
- [ ] 13.4 Manual end-to-end: confirm `--forbid` (delta-scoped) ignores pre-existing match in untouched file.
- [ ] 13.5 Manual end-to-end: `init` with marker blocks on existing `AGENTS.md` — confirm existing content preserved.
- [ ] 13.6 Manual test of `ast-guard-skills.md` prompt with an actual AI agent — does the agent follow the skill and use ast-grep + cargo-check correctly? Validate Phase 1 before Phase 2.