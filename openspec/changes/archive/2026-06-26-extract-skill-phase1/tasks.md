## 1. Skill Prompt — CRITICAL RULE and Self-Contained Structure

- [x] 1.1 Write the CRITICAL RULE section: prohibit `sed`/`awk`/Python/`edit` for structural refactors in Rust, C#, TypeScript, Dart; mandate `ast-grep` (`sg`) + compiler as the only acceptable tools.
- [x] 1.2 Add the self-contained statement: the skill works without `ast-guard` CLI installed; all commands use `sg` and standard compiler tools.
- [x] 1.3 Add "When to use the CLI" subsection: mentions `ast-guard refactor` as Phase 2 automation (optional upgrade), with the `sg`-based workflow as primary.

## 2. Skill Workflow — Three-Step Core Loop

- [x] 2.1 Write the **DISCOVER** step: `sg scan -p '<pattern>' -l <lang> --json` with context lines, counting matching sites.
- [x] 2.2 Write the **APPLY** step: `sg run --dry-run` for preview, then `sg run` to apply the rewrite.
- [x] 2.3 Write the **VALIDATE** step: compiler invocation with machine-readable output (`cargo check --message-format=json`, `dotnet build`, `tsc --noEmit --pretty false`, `dart analyze --format=json`) and straggler check (`sg scan -p '<old form>' -l <lang> --json | jq '.count'` must return 0).
- [x] 2.4 Add concrete shell commands for each step (not abstract descriptions) with inline comments explaining each flag.

## 3. Skill Patterns — Per-Language Examples

- [x] 3.1 Write Rust example: function rename (`old_api` → `new_api`) with straggler check; method migration with generics (`$T`).
- [x] 3.2 Write C# example: `JsonConvert.DeserializeObject<T>` → `JsonSerializer.Deserialize<T>` with straggler check on `JsonConvert.`.
- [x] 3.3 Write TypeScript example: `console.log(...)` → logger function with straggler check on `console.log`.
- [x] 3.4 Write Dart example: widget tree refactor (e.g., `OldWidget` → `NewWidget`) with straggler check.
- [x] 3.5 Ensure all examples are copy-pasteable verbatim (valid `ast-grep` syntax with `$$$`, `$`, `$T`).

## 4. Recovery Playbook (git restore, not git checkout)

- [x] 4.1 Write pre-refactor restore point section: `git status --porcelain` check, prefer `git commit` over `git stash`, commit message convention (`refactor: pre-rewrite checkpoint`).
- [x] 4.2 Write surgical recovery section: `git restore --source=<ref> -- <paths>` with example restoring only touched files.
- [x] 4.3 Write explicit prohibition: `git checkout -- .` is nuclear, never use; the skill names this prohibition and explains why.
- [x] 4.4 State that `ast-grep`/`ast-guard` do NOT perform rollbacks — recovery is the agent's responsibility.

## 5. Anti-Patterns Section

- [x] 5.1 Document `sed`-based rename anti-pattern with the contrast: the `sed` command vs. the `sg run` equivalent.
- [x] 5.2 Document Python throwaway script anti-pattern (`os.listdir` + `str.replace`) with `sg` equivalent.
- [x] 5.3 Document manual file manipulation anti-pattern (agent's default `edit`/`replace`) — produces silent stragglers — with `sg` equivalent.
- [x] 5.4 Ensure each anti-pattern is visually paired with the skill-prescribed command.

## 6. Testing the Skill — Three Validation Scenarios

- [x] 6.1 Write **Test 1**: Create Rust fixture with `old_calculate` in 3 files, run full DISCOVER → APPLY → VALIDATE loop, verify 0 stragglers and compile passes.
- [x] 6.2 Write **Test 2**: Same fixture, manually leave one straggler in 4th file, run `sg scan` — verify count = 1, do NOT proceed to `sg run`.
- [x] 6.3 Write **Test 3**: Write `--rewrite` that produces unbalanced braces, run `sg run --dry-run` — verify `sg` reports syntax error, do NOT apply.
- [x] 6.4 Ensure all three tests include exact commands, expected output, and pass/fail condition.

## 7. Language Scope and Final Polish

- [x] 7.1 Add supported languages declaration (Rust, C#, TypeScript, Dart) with explicit Python exclusion note (Phase 3 candidate).
- [x] 7.2 Add a "How to read this skill" summary at the top explaining the structure (CRITICAL RULE → Workflow → Examples → Recovery → Anti-Patterns → Tests).
- [x] 7.3 Review the entire `ast-guard-skills.md` for consistency, copy-paste correctness of all commands, and self-containment (no `ast-guard` CLI required).
- [x] 7.4 Write `ast-guard-skills.md` at the repository root.

## 8. Structural Cleanup — Remove Skill from CLI Change

- [x] 8.1 Remove `openspec/changes/compile-safe-refactor-cli/specs/skill/` directory (the spec now lives in this change).
- [x] 8.2 Revise task 11.5 in `compile-safe-refactor-cli/tasks.md`: either remove the task or change it to cross-reference this change (`extract-skill-phase1`) with a note that the skill prompt is now owned by the Phase 1 change.
- [x] 8.3 Verify the CLI change's proposal.md and design.md still make sense without the skill spec (no dangling references).

## 9. Validation

- [x] 9.1 Run `openspec status --change "extract-skill-phase1"` and confirm all artifacts are complete.
- [x] 9.2 Read `ast-guard-skills.md` end-to-end and verify it is self-contained, actionable, and covers all requirements from the skill-prompt spec.
- [x] 9.3 Verify the CLI change (`compile-safe-refactor-cli`) has no remaining references to `specs/skill/` content.
