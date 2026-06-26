## Why

The `compile-safe-refactor-cli` change (Phase 2) contains a `specs/skill/spec.md` that describes a full standalone Phase 1 deliverable — the `ast-guard-skills.md` agent prompt. This is a structural mismatch:

- The skill spec defines **requirements** (CRITICAL RULE, three-step core loop, per-language patterns, recovery playbook, anti-patterns, test scenarios) that go far beyond a single documentation task.
- The CLI change's tasks.md only has **task 11.5** ("Write `ast-guard-skills.md`") buried under "Documentation" — it cannot properly cover the full scope of the skill spec's requirements.
- The three-phase delivery strategy (skill → lightweight CLI → mature CLI) explicitly treats the skill as **Phase 1**, a separate milestone that ships independently and validates demand before Phase 2 investment.

The skill spec needs its own change with its own tasks that properly address all requirements. This change extracts the skill from the CLI change and establishes it as a first-class Phase 1 deliverable.

## What Changes

- **Extract** the `specs/skill/spec.md` from the `compile-safe-refactor-cli` change into this new `extract-skill-phase1` change as `specs/skill-prompt/spec.md` (renamed for clarity).
- **Create proper tasks** in this change that fully cover all requirements from the skill spec: CRITICAL RULE, three-step core loop, per-language patterns, recovery playbook, anti-patterns, self-contained design, test scenarios, and supported language scope.
- **Preserve the skill spec as-is** from the CLI change (no content change), just relocate it to its own change.
- **Remove `specs/skill/spec.md`** from the `compile-safe-refactor-cli` change — that change's task 11.5 may reference it externally or be revised to a cross-reference.
- **No code changes.** This change is purely structural: splitting one change into two independent deliverables.

## Capabilities

### New Capabilities
- `skill-prompt`: The standalone `ast-guard-skills.md` agent prompt that teaches AI coding agents to use `ast-grep` + compiler workflow instead of `sed`/`regex`/scripts for structural refactoring in typed languages. Self-contained, works without the `ast-guard` CLI.

### Modified Capabilities
- *(None. No main specs exist in `openspec/specs/` yet.)*

## Impact

- **`compile-safe-refactor-cli` change**: loses `specs/skill/spec.md` and task 11.5 becomes a cross-reference or is removed. The CLI change becomes purely Phase 2 (the Rust tool).
- **New change**: `extract-skill-phase1` owns Phase 1 (the skill prompt). Ships independently, validates demand before Phase 2.
- **Deliverable**: `ast-guard-skills.md` — a single Markdown file in the repo root (or `docs/`). No code, no build step. Ships immediately.
- **Dependencies**: None. The skill prompt is self-contained, uses only `sg` (ast-grep CLI) and `cargo check`/`dotnet build`/`tsc`/`dart analyze`.
