# ast-guard: Agent Skill for Safe Structural Refactoring

**Phase 1 deliverable** — a prompt for AI coding agents that teaches the `ast-grep` + `cargo check` workflow. No binary needed. Read this, follow it, and never write `sed`/`awk`/throwaway Python for typed-language refactors again.

---

## CRITICAL RULE

**You MUST NOT use `sed`, `awk`, Python scripts, or manual file manipulation (`replace`, `edit`) for structural code refactoring in Rust, C#, TypeScript, or Dart.**

Why:
- Regex-based text replacement does not understand syntax — it breaks brackets, destroys async contexts, and can't distinguish `old_name` the function from `old_name` the string literal.
- Manual file manipulation produces silent stragglers — "47 of 48 sites replaced" looks complete but leaves one surviving instance the compiler won't catch.
- The resulting compile errors force a multi-round repair loop that wastes tokens and introduces new bugs.

**You MUST use `ast-grep` (`sg`), `ast-guard` (when available), or `cargo check` / `dotnet build` / `tsc --noEmit` for all structural changes in these languages.**

---

## The Workflow

### When `ast-guard` is not installed (this skill)

Use `ast-grep` directly. It's the same pattern engine, just without the safety gates:

```bash
# 1. DISCOVER: what matches?
sg scan -p '<pattern>' -l <lang> --json -C 3  # show context around matches

# 2. PREVIEW: what will change? (dry-run equivalent)
sg run -p '<pattern>' -r '<rewrite>' -l <lang> --dry-run

# 3. CHECK COUNT: how many sites?
sg scan -p '<old form after rewrite>' -l <lang> --json | jq '.count'  # should be 0
sg scan -p '<new form>' -l <lang> --json | jq '.count'  # should be >0

# 4. APPLY: rewrite
sg run -p '<pattern>' -r '<rewrite>' -l <lang>

# 5. VALIDATE: compile
cargo check --message-format=json | jq '.messages[] | select(.level == "error" or .level == "warning")'
# or: dotnet build; tsc --noEmit --pretty false; dart analyze --format=json

# 6. RECOVER on failure:
git restore --source=HEAD -- <paths>   # surgical - only the files you touched
# NEVER: git checkout -- .  (nuclear - destroys unrelated work)
```

**Critical: Steps 3-4-5 are non-negotiable.** The straggler check (step 3) is the difference between "looks complete" and "is complete." The compile (step 5) catches the semantic errors.

### When `ast-guard` is installed (Phase 2)

The skill workflow is automated into a single command:

```bash
# Preview
ast-guard refactor --lang rust --pattern 'fn old_name($$$) { $$$ }' --rewrite 'fn new_name($$$) { $$$ }' --dir ./src --require 'fn new_name' --forbid 'fn old_name' --dry-run

# Apply (all gates enforced)
ast-guard refactor --lang rust --pattern 'fn old_name($$$) { $$$ }' --rewrite 'fn new_name($$$) { $$$ }' --dir ./src --require 'fn new_name' --forbid 'fn old_name'

# Batching
ast-guard refactor --lang rust --pattern '...' --rewrite '...' --dir ./src --no-build
ast-guard refactor --lang rust --pattern '...' --rewrite '...' --dir ./src --no-build
ast-guard check --lang rust  # one compile, all refactors validated

# Recovery
git restore --source=<ref> -- <paths>  # taught by `ast-guard init`
```

---

## Patterns: How to write them

### ast-grep pattern grammar (same for `sg` and `ast-guard`)

- `$$$A` — match any subtree (zero or more nodes). For function bodies, argument lists, blocks.
- `$$A` — match a single node (identifier, expression). For function names, variables.
- `$A` — match a single AST node (narrower than `$$A`).

### Examples per language

#### Rust

```bash
# Rename a function
sg scan -p 'fn old_api($$$ARGS) -> $RET { $$$BODY }' -l rust --json

# Change error handling (unwrap → ?)
sg run -p 'x.unwrap()' -r 'x?' -l rust --dry-run
sg run -p 'x.unwrap()' -r 'x?' -l rust
# Straggler check:
sg scan -p '.unwrap()' -l rust --json | jq '.count'  # should be 0

# Remove deprecated function calls
sg run -p 'deprecated_fn($$$ARGS)' -r 'new_fn($$$ARGS)' -l rust
sg scan -p 'deprecated_fn($$$ARGS)' -l rust --json | jq '.count'  # verify 0
```

#### C#

```bash
# Migrate from Newtonsoft.Json to System.Text.Json
sg scan -p 'JsonConvert.DeserializeObject<$T>($$$ARGS)' -l csharp --json
sg run -p 'JsonConvert.DeserializeObject<$T>($$$ARGS)' -r 'JsonSerializer.Deserialize<$T>($$$ARGS)' -l csharp
sg scan -p 'JsonConvert.' -l csharp --json | jq '.count'  # verify no stragglers

# Replace synchronous API with async
sg run -p 'client.Get($$$ARGS)' -r 'await client.GetAsync($$$ARGS)' -l csharp
```

#### TypeScript

```bash
# Migrate console.log to structured logger
sg scan -p 'console.log($$$ARGS)' -l typescript --json
sg run -p 'console.log($$$ARGS)' -r 'logger.info($$$ARGS)' -l typescript
# Straggler check:
sg scan -p 'console.log($$$ARGS)' -l typescript --json | jq '.count'  # must be 0

# Replace deprecated API
sg run -p 'deprecatedFunction($$$ARGS)' -r 'newFunction($$$ARGS)' -l typescript
tsc --noEmit --pretty false
```

#### Dart

```bash
# Widget tree refactor: replace deprecated widget
sg scan -p 'FlatButton($$$PROPS)' -l dart --json
sg run -p 'FlatButton($$$PROPS)' -r 'TextButton($$$PROPS)' -l dart
dart analyze --format=json
```

---

## Recovery Playbook

### Before every refactor batch

```bash
# 1. Check state
git status --porcelain

# 2. Create restore point — prefer a commit over a stash
# Commits are hard restore points. `stash pop` can conflict if you make other changes mid-refactor.
git add -A && git commit -m "refactor: checkpoint before structural rewrite"

# 3. If the tree is already clean, a commit is still better than a stash
git commit --allow-empty -m "refactor: pre-rewrite checkpoint"
```

### On failure (compilation, straggler, anything)

```bash
# SURGICAL: restore only the files ast-guard/sg touched
# This is the tool you want. It restores specific files from a commit, not the entire tree.
git restore --source=<ref> -- <paths>

# Example: restore only the three files you just rewrote
git restore --source=HEAD -- src/models/user.rs src/models/order.rs src/services/api.rs

# NUCLEAR: NEVER use this
# git checkout -- .   # This destroys ALL your changes, including unrelated work.
#                      # The agent must never use this.
```

**Important:** `ast-guard` itself does NOT perform rollbacks. It writes to disk and delegates recovery to you. You are responsible for creating restore points (`git commit`) before refactoring and recovering surgically (`git restore --source=<ref> -- <paths>`) on failure.

---

## Anti-Patterns (what this skill replaces)

### ❌ What agents currently do (the problem)

```bash
# sed-based rename — breaks syntax, no compiler validation
find src/ -name '*.rs' -exec sed -i 's/old_name/new_name/g' {} \;
# Result: partial matches, broken syntax, no straggler check, no compile.

# Python throwaway script — no AST awareness
python3 -c "
import os
for f in os.listdir('src/'):
    with open(f) as fp:
        content = fp.read().replace('old_name', 'new_name')
    with open(f, 'w') as fp:
        fp.write(content)
"
# Result: string literals accidentally renamed, macro calls missed, imports not updated.

# Manual file manipulation — the agent's default
# Agent writes: "replace 'old_name' with 'new_name' in src/utils.rs"
# Result: 1 file changed, 47 files untouched. Silent straggler.
```

### ✅ What this skill mandates

```bash
# AST-aware pattern matching — understands code structure
sg run -p 'fn old_name($$$ARGS) { $$$BODY }' -r 'fn new_name($$$ARGS) { $$$BODY }' -l rust

# Compiler validation with structured output — no stderr regex parsing
cargo check --message-format=json | jq '.messages[] | select(.level == "error")'

# Straggler check — the guarantee that "looks complete" is actually complete
sg scan -p 'fn old_name' -l rust --json | jq '.count'  # must be 0
```

---

## Testing this skill

### Test 1: Rename a function in a Rust crate

1. Create a fixture crate with `fn old_calculate(x: i32) -> i32 { x * 2 }` in 3 files.
2. Run the skill workflow: `sg scan`, `sg run`, `sg scan` (straggler), `cargo check`.
3. Verify: 0 stragglers, compile passes, all 3 files updated.

### Test 2: Simulate a straggler

1. Same fixture crate, but manually leave one `old_calculate` call untouched in a 4th file.
2. Run `sg scan -p 'fn old_calculate' -l rust --json | jq '.count'`.
3. Verify: count = 1 — the straggler is caught. Do not proceed to `sg run`.

### Test 3: Malformed rewrite template

1. Write a `--rewrite` that produces unbalanced braces (e.g., `'fn new($$$) { '` — missing `}`).
2. Run `sg run --dry-run` and observe that `sg` reports a syntax error.
3. Verify: the agent does not apply the rewrite. No files are mutated.

---

## When to use the CLI (Phase 2) instead of this skill

This skill works today with `ast-grep` (`sg`) + `cargo check`. The CLI (`ast-guard`) adds:

- **Hard `--require`/`--forbid` gates** — the CLI rejects the write if conditions fail. This skill says "please check yourself" — it's advisory.
- **Structured compiler diagnostics** — the CLI parses `cargo check --message-format=json` centrally. This skill delegates to you (the agent) to write `jq` filters. The CLI gives file:line:col:span with one command.
- **Re-parse gate** — the CLI catches malformed rewrites before disk mutation, automatically.
- **Token efficiency** — one `ast-guard refactor` vs. 5-6 `sg` + `jq` + `cargo check` commands per refactor.

For a single refactor, the skill workflow is fine. For **hundreds of refactors per session**, the CLI's enforced gates and token efficiency are the difference between a clean migration and a silent straggler in production.

---

## Supported Languages (Phase 1)

- **Rust:** `sg` + `cargo check --message-format=json`
- **C#:** `sg` + `dotnet build`
- **TypeScript:** `sg` + `tsc --noEmit --pretty false`
- **Dart:** `sg` + `dart analyze --format=json`

**Python is not supported in Phase 1.** Python refactors need `--require`/`--forbid` + re-parse as the primary safety layer (no compiler backstop). This is a Phase 3 candidate.

---

## The Contract

1. **You (the agent) write `ast-guard` commands, not `sed`/`awk`/Python scripts.**
2. **You always run `--dry-run` first — before any file is written.**
3. **You always check for stragglers (`--forbid`) — before committing.**
4. **You always compile after rewriting — before opening a PR.**
5. **You always create a git restore point before a refactor batch — `git commit` over `git stash`.**
6. **You never use `git checkout -- .` — that's nuclear. Use `git restore --source=<ref> -- <paths>`.**

When `ast-guard` is installed, run `ast-guard init` once per repo — it writes these rules to your `AGENTS.md`.