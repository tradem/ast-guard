## ADDED Requirements

### Requirement: Skill Prompt — CRITICAL RULE for Typed-Language Refactors

The Phase 1 skill prompt SHALL teach AI coding agents to **never** use `sed`, `awk`, Python scripts, or manual file manipulation (`replace`, `edit`) for structural code refactoring in Rust, C#, TypeScript, or Dart. It SHALL mandate `ast-grep` (`sg`) and the target language's compiler as the **only** acceptable tools for structural changes in these languages.

The CRITICAL RULE SHALL be a self-contained, actionable instruction — it must function without the `ast-guard` CLI being installed. It SHALL be the first section of the skill prompt.

#### Scenario: Agent reads the skill and stops using sed
- **WHEN** an AI coding agent reads the Phase 1 skill prompt before attempting a structural refactor in Rust
- **THEN** the agent abandons its default `sed`/regex approach and switches to `ast-grep` + `cargo check` workflow, because the CRITICAL RULE explicitly prohibits text-based manipulation for typed languages and names the acceptable tools

#### Scenario: Skill works without ast-guard CLI
- **WHEN** an agent does not have `ast-guard` installed on PATH
- **THEN** the skill prompt still functions: it teaches the `sg scan` → `sg run` → `cargo check --message-format=json` workflow with no dependency on the `ast-guard` binary. The prompt is self-contained.

### Requirement: Skill Workflow — Three-Step Core Loop

The skill prompt SHALL teach a three-step core loop:

1. **DISCOVER** (`sg scan`, JSON, context lines) — find all matching sites and count them.
2. **APPLY** (`sg run`, dry-run first, then apply) — execute the rewrite.
3. **VALIDATE** (compiler invocation with machine-readable output, straggler check via `sg scan` on the old pattern) — verify correctness and completeness.

All three steps SHALL be presented as concrete shell commands (not abstract descriptions). The skill SHALL include a **straggler check** step (step 3b: `sg scan -p '<old form>' -l <lang> --json | jq '.count'` must return 0) — this is the guarantee that the rewrite is complete, not just "looks complete."

#### Scenario: Agent follows the three-step loop for a function rename
- **WHEN** an agent follows the skill workflow for renaming `old_calculate` → `new_calculate` in a Rust crate
- **THEN** the agent runs: (1) `sg scan -p 'fn old_calculate($$$ARGS) { $$$BODY }' -l rust --json` to discover sites, (2) `sg run --dry-run` to preview, (3) `sg run` to apply, (4) `sg scan -p 'fn old_calculate' -l rust --json | jq '.count'` to confirm 0 stragglers, (5) `cargo check --message-format=json` to validate. All three steps are executed and the agent reports 0 stragglers.

#### Scenario: Agent catches a straggler via the check
- **WHEN** 3 of 4 files are successfully rewritten but the 4th file still contains `old_calculate` (a straggler)
- **THEN** the agent's straggler check (`sg scan -p 'fn old_calculate' -l rust --json | jq '.count'`) returns 1, and the agent does NOT proceed to `sg run` — it either re-runs the pattern on the missed file or reports the straggler as a failure

### Requirement: Skill Patterns — Per-Language Examples

The skill prompt SHALL include concrete `ast-grep` pattern examples for each supported language (Rust, C#, TypeScript, Dart), covering at minimum:
- **Function rename** (the most common agent refactor — "rename X to Y").
- **Method migration** (replace deprecated API with new form, including generic type parameter handling).
- **Straggler check** (re-scanning for the old form after a rewrite to verify 0 matches).

Each example SHALL use valid `ast-grep` pattern syntax (`$$$`, `$`, `$T` for generics). Examples SHALL be copy-pasteable — the agent must be able to run them verbatim.

#### Scenario: Agent copies a Rust function rename example
- **WHEN** an agent reads the Rust function rename example in the skill
- **THEN** the agent can copy-paste `sg run -p 'fn old_api($$$ARGS) -> $RET { $$$BODY }' -r 'fn new_api($$$ARGS) -> $RET { $$$BODY }' -l rust --dry-run` and execute it without modifying the pattern

#### Scenario: Agent uses the C# generic method migration example
- **WHEN** an agent reads the C# example for migrating `JsonConvert.DeserializeObject<T>` to `JsonSerializer.Deserialize<T>`
- **THEN** the agent can copy-paste the `sg scan` and `sg run` commands, run them, and verify the straggler count with `sg scan -p 'JsonConvert.' -l csharp --json | jq '.count'`

### Requirement: Recovery Playbook (git restore, not git checkout)

The skill prompt SHALL include a **git recovery playbook** that teaches the agent:

- **Before every refactor batch:** create a restore point — `git status --porcelain` to check tree state, then `git commit` (preferred) or `git stash`. Commits are preferred over stashes because `stash pop` can conflict if the agent makes intermediate changes.
- **On failure (compilation error, straggler, anything):** surgically restore only the files that were touched — `git restore --source=<ref> -- <paths>`. This restores specific files from a commit, not the entire tree.
- **Explicitly prohibited:** `git checkout -- .` — this is nuclear, destroying all changes including unrelated work. The skill SHALL state this prohibition explicitly and name it "never use."

The skill SHALL state that `ast-grep`/`ast-guard` do NOT perform rollbacks — recovery is the agent's responsibility. The playbook SHALL be a self-contained section, not dependent on `ast-guard init`.

#### Scenario: Agent creates a restore point before a refactor batch
- **WHEN** an agent follows the skill's recovery playbook before a refactor batch
- **THEN** the agent runs `git status --porcelain`, and if the tree is clean, runs `git commit --allow-empty -m "refactor: pre-rewrite checkpoint"`. If the tree has uncommitted changes, the agent runs `git add -A && git commit -m "refactor: checkpoint before structural rewrite"`. The agent never uses `git stash` unless the tree has unstaged changes that cannot be committed.

#### Scenario: Agent recovers surgically after a failed compile
- **WHEN** a compiler run fails after a structural rewrite of 3 files
- **THEN** the agent runs `git restore --source=HEAD -- src/models/user.rs src/models/order.rs src/services/api.rs` — restoring only the 3 files that were touched. The agent does NOT run `git checkout -- .` (the nuclear option) and does NOT lose unrelated work.

### Requirement: Anti-Patterns Section

The skill prompt SHALL include an **anti-patterns section** that shows what agents currently do (the problem) and contrasts it with the skill-prescribed approach (the solution). The anti-patterns SHALL document:

- `sed`-based renames (the classic `sed -i 's/old/new/g'` across files — breaks syntax, no AST awareness).
- Python throwaway scripts (`os.listdir` + `str.replace` — string-level, not syntax-level).
- Manual file manipulation (the agent's default — "replace X with Y in file Z" — produces silent stragglers because only 1 file is touched while 47 others remain unchanged).

Each anti-pattern SHALL be paired with the corresponding skill-prescribed command, visually showing the contrast.

#### Scenario: Agent recognizes a sed anti-pattern
- **WHEN** an agent is about to write `find src/ -name '*.rs' -exec sed -i 's/old_name/new_name/g' {} \;` and has read the skill's anti-patterns section
- **THEN** the agent recognizes this as the classic "sed rename" anti-pattern and switches to `sg run -p 'fn old_name($$$ARGS) { $$$BODY }' -r 'fn new_name($$$ARGS) { $$$BODY }' -l rust` instead

### Requirement: Self-Contained (No CLI Dependency)

The skill prompt SHALL be **self-contained** — it SHALL NOT require `ast-guard` to be installed. All commands reference `sg` (the `ast-grep` CLI) and `cargo check`/`dotnet build`/`tsc`/`dart analyze`. The skill SHALL mention `ast-guard` as the Phase 2 automation ("when available, use `ast-guard refactor` instead of the manual `sg` + `cargo check` loop"), but the primary workflow SHALL be `sg`-based.

#### Scenario: Agent uses the skill without ast-guard installed
- **WHEN** an agent does not have `ast-guard` installed and reads the skill
- **THEN** the agent follows the `sg`-based workflow: `sg scan`, `sg run --dry-run`, `sg run`, `cargo check --message-format=json`, straggler check. The agent never attempts to run `ast-guard refactor` because the skill explicitly says "when available" and provides the `sg` fallback.

#### Scenario: Skill mentions ast-guard as optional upgrade
- **WHEN** an agent reads the skill's "When to use the CLI" section
- **THEN** the agent understands that `ast-guard` automates the manual `sg` + `cargo check` + straggler check loop, but that the `sg` workflow is the **primary** workflow and works without `ast-guard`. The agent can choose to install `ast-guard` for token efficiency in large refactor batches.

### Requirement: Testing the Skill — Three Validation Scenarios

The skill prompt SHALL include **three test scenarios** that the agent can execute to verify the skill works:

1. **Test 1: Rename a function in a Rust crate** — create a fixture with `old_calculate` in 3 files, run `sg scan`, `sg run`, `sg scan` (straggler), `cargo check`. Verify 0 stragglers, compile passes.
2. **Test 2: Simulate a straggler** — same fixture, but manually leave one `old_calculate` in a 4th file. Run `sg scan` — verify count = 1. Do not proceed to `sg run`.
3. **Test 3: Malformed rewrite template** — write a `--rewrite` that produces unbalanced braces. Run `sg run --dry-run` — verify `sg` reports a syntax error. Do not apply.

These tests SHALL be concrete: state the exact commands, the expected output, and the pass/fail condition. They SHALL be self-contained — the agent can run them without external dependencies beyond `sg` and `cargo`.

#### Scenario: Agent runs Test 1 and verifies the skill
- **WHEN** an agent executes the three test scenarios against a fixture crate
- **THEN** Test 1 passes (0 stragglers, compile succeeds), Test 2 passes (1 straggler detected, rewrite not applied), Test 3 passes (syntax error detected, rewrite not applied). The agent confirms that the skill is correct and that its workflow catches the three failure modes: incomplete rewrite (straggler), malformed syntax (rewrite error), and missing compile validation.

### Requirement: Supported Languages (Scope)

The skill prompt SHALL support **Rust, C#, TypeScript, and Dart** in Phase 1. Python SHALL NOT be included — the skill SHALL state that Python is a Phase 3 candidate because Python refactors lack a compiler backstop (the primary safety layer is `--require`/`--forbid` + re-parse, which requires the CLI).

#### Scenario: Agent asks about Python refactors
- **WHEN** an agent reads the skill and wants to refactor Python
- **THEN** the skill states that Python is not supported in Phase 1 and that the agent should use a separate workflow for Python. The skill does not provide Python examples or patterns.