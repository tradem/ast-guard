## Context

The ast-guard project follows a three-phase delivery strategy:

1. **Phase 1 (Skill)**: A Markdown prompt that teaches AI coding agents to use `ast-grep` + compiler workflow — ships immediately, no code.
2. **Phase 2 (CLI)**: The lightweight Rust orchestrator (`ast-guard refactor`/`check`/`init`) — under development.
3. **Phase 3 (Mature CLI)**: Expanded language support, rule-file, mutation testing — conditional on Phase 2 demand.

Currently, the Phase 1 skill spec (`specs/skill/spec.md`) is nested inside the Phase 2 change (`compile-safe-refactor-cli`), where its requirements are addressed by a single documentation task (11.5). This change splits it out into a standalone Phase 1 deliverable.

The deliverable is a single Markdown file: `ast-guard-skills.md`. No code, no build step, no dependencies. It teaches agents the `sg scan` → `sg run` → compiler validation → straggler check workflow.

### Current State

- `openspec/changes/compile-safe-refactor-cli/specs/skill/spec.md` exists with comprehensive requirements (CRITICAL RULE, three-step loop, per-language patterns, recovery playbook, anti-patterns, test scenarios, supported languages)
- `openspec/changes/compile-safe-refactor-cli/tasks.md` has task 11.5 ("Write `ast-guard-skills.md`") — insufficient scope
- `ast-guard-skills.md` does not exist yet anywhere in the repo
- No code changes are needed — this is purely a documentation + project-structure change

### Constraints

- The skill prompt must be **self-contained**: it must work without the `ast-guard` CLI installed
- All commands reference `sg` (ast-grep CLI) and standard compiler tools (`cargo check`, `dotnet build`, `tsc`, `dart analyze`)
- Python is explicitly excluded from Phase 1 scope
- The skill prompt lives in the repo root as `ast-guard-skills.md` (or `docs/ast-guard-skills.md` if a docs directory exists)
- The existing CLI change `compile-safe-refactor-cli` must have `specs/skill/` removed and task 11.5 revised or removed

## Goals / Non-Goals

**Goals:**

- Create a standalone `ast-guard-skills.md` that teaches agents the `ast-grep` + compiler workflow
- Cover all requirements from the skill spec: CRITICAL RULE, three-step loop, per-language examples, recovery playbook, anti-patterns, test scenarios, language scope
- Remove `specs/skill/` from the CLI change (it no longer belongs there)
- Ensure the CLI change tasks no longer claim responsibility for the skill prompt
- All requirements spec'd in this change's `specs/skill-prompt/spec.md`

**Non-Goals:**

- No code changes (no Rust, no CLI, no binary)
- No change to the content of the skill spec itself (it's correct as-is)
- No changes to `compile-safe-refactor-cli`'s other specs or tasks beyond removing `specs/skill/` and adjusting task 11.5
- No creation of a main spec in `openspec/specs/` (this change delivers a new capability, not a modification of existing main specs)
- No deployment or distribution mechanism beyond the Markdown file itself

## Decisions

### D1: The skill spec content is reused as-is, not rewritten

The existing `specs/skill/spec.md` in the CLI change is well-written with proper requirements and scenarios. It was authored with the three-phase strategy in mind and its content is correct.

**Decision:** Copy the skill spec content into this change's `specs/skill-prompt/spec.md` without substantive changes. The only change is the capability name from `skill` to `skill-prompt` for clarity.

**Alternatives considered:** Rewriting from scratch (unnecessary — content is correct), or keeping it in the CLI change (defeats the purpose of the split).

### D2: The deliverable file is `ast-guard-skills.md` in the repo root

The original Phase 1 design (proposal.md of CLI change) specifies the Phase 1 deliverable as a "200-line markdown document" called `ast-guard-skills.md`. Placing it in the repo root makes it immediately visible to agents and users.

**Decision:** `ast-guard-skills.md` at the repository root.

**Alternatives considered:** `docs/ast-guard-skills.md` (reasonable if a docs directory exists, but root is more visible for a Phase 1 standalone artifact).

### D3: The CLI change's `specs/skill/` is removed, not moved

**Decision:** Delete `openspec/changes/compile-safe-refactor-cli/specs/skill/` (the directory and its `spec.md`). The content lives here now. Task 11.5 in the CLI change is revised to cross-reference this change or removed entirely.

**Alternatives considered:** Leaving a symlink/cross-reference stub (adds maintenance complexity with no benefit).

### D4: No main-spec merge in this change

The `openspec/specs/` directory is empty (no main specs exist yet). The skill-prompt is a new capability, so it stays as a delta spec within this change. If main specs are established later, this can be promoted.

**Decision:** `specs/skill-prompt/spec.md` stays as a delta spec in this change only. No `openspec/specs/skill-prompt/spec.md` is created.

## Risks / Trade-offs

- **[CLI change tasks remain out of sync after removal]** → Mitigation: After archive of this change, the CLI change's task 11.5 must be revised to remove the `ast-guard-skills.md` deliverable. Tracked as a post-archive step.
- **[Skill prompt becomes stale without CLI integration]** → Mitigation: The skill is designed to work solely with `sg` + compiler. It references `ast-guard` only as an optional Phase 2 upgrade. Staleness risk is low because `sg` and compiler interfaces are stable.
- **[Agent confusion from two separate changes]** → Mitigation: Phase 1 (this change) ships and archives before Phase 2 implementation completes. The README documents the three-phase strategy so readers understand the relationship.
