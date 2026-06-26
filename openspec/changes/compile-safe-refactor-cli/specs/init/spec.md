## ADDED Requirements

### Requirement: Write agent-integration prompt files (marker blocks, non-destructive)

The `init` subcommand SHALL write agent-integration prompt files containing (1) the CRITICAL RULE prohibiting sed/awk/throwaway-script refactors in typed languages and mandating ast-guard, and (2) a git-based recovery playbook for failed refactors.

The `--target` flag SHALL select which agent's prompt file is written: `agents` (default, writes `AGENTS.md`), `copilot` (writes `.github/copilot-instructions.md`), `claude` (writes `CLAUDE.md`), `pi` (writes `.pi/prompts/ast-guard.md`), or `all` (writes all of the above).

**Init SHALL use marker blocks**, not full-file overwrite. If a target file already exists, `init` SHALL insert its content between `<!-- ast-guard:start -->` and `<!-- ast-guard:end -->` markers, preserving existing content outside those markers. If the file does not exist, `init` SHALL create it with the markers and content. This prevents clobbering user-customized files on repeated `init` runs.

If a target file already exists and marker insertion fails (file has malformed structure), `init` SHALL emit a warning and skip that file. `--force` SHALL overwrite the file entirely (without marker blocks).

#### Scenario: Default writes AGENTS.md with markers
- **WHEN** `ast-guard init` is run in a directory without an `AGENTS.md`
- **THEN** an `AGENTS.md` file is created in the project root containing `<!-- ast-guard:start -->`, the CRITICAL RULE, the git recovery playbook, and `<!-- ast-guard:end -->`

#### Scenario: Init inserts into existing AGENTS.md without clobbering
- **WHEN** `ast-guard init` is run and an `AGENTS.md` already exists with user content
- **THEN** `init` inserts the CRITICAL RULE and recovery playbook between `<!-- ast-guard:start -->` and `<!-- ast-guard:end -->` markers, and the existing user content outside those markers is preserved

#### Scenario: All targets writes every supported file
- **WHEN** `ast-guard init --target all` is run
- **THEN** `AGENTS.md`, `.github/copilot-instructions.md`, `CLAUDE.md`, and `.pi/prompts/ast-guard.md` are all created or updated with marker blocks

#### Scenario: Force overwrites without markers
- **WHEN** `ast-guard init --force` is run and an `AGENTS.md` already exists
- **THEN** the entire file is overwritten with the template content (no markers, no preservation)

#### Scenario: Existing file with marker conflict emits warning and skips
- **WHEN** `ast-guard init` is run and a target file exists but marker insertion fails (e.g., file cannot be parsed for insertion)
- **THEN** `init` emits a warning for that file and skips it; other target files proceed normally

### Requirement: CRITICAL RULE content

The CRITICAL RULE section of every init template SHALL state that sed, awk, Python scripts, and native file manipulation MUST NOT be used for refactoring Rust, C#, TypeScript, or Dart, and that ast-guard MUST be used for all structural changes. It SHALL include the `ast-guard refactor` usage line and the workflow: dry-run first, then apply, then (if compilation fails) analyze filtered output and either fix-forward with a new ast-guard command or recover with git.

**Note:** The CRITICAL RULE references ast-guard as the tool — not ast-grep directly. The skill (Phase 1) teaches ast-grep usage; the CLI (Phase 2) is the enforced wrapper. `init` ships the CLI rule because it assumes the CLI is installed.

#### Scenario: Critical rule present in every target
- **WHEN** any init target file is written
- **THEN** the file contains a CRITICAL RULE section naming sed, awk, and Python scripts as prohibited for typed-language refactors and naming ast-guard as the required tool

### Requirement: Git recovery playbook content

The git recovery playbook section SHALL teach the agent to inspect state before a refactor batch (`git status --porcelain`), to prefer a commit over a stash when the tree is clean-ish, and on failure to recover surgically with `git restore --source=<ref> -- <paths>` (only the files ast-guard touched). It SHALL explicitly warn that `git checkout -- .` is nuclear and MUST NOT be used. It SHALL state that ast-guard does NOT perform rollbacks and that recovery is the agent's responsibility.

#### Scenario: Playbook teaches surgical restore
- **WHEN** any init target file is written
- **THEN** the file contains a recovery playbook that includes `git restore --source=<ref> -- <paths>` and explicitly prohibits `git checkout -- .`

#### Scenario: Playbook states no-rollback stance
- **WHEN** any init target file is written
- **THEN** the file states that ast-guard does not perform rollbacks and that the agent is responsible for recovery using git

### Requirement: CLI flag surface

The `init` subcommand SHALL accept `--target` (default `agents`; one of `agents`, `copilot`, `claude`, `pi`, `all`), and `--force` (default false; overwrites existing files entirely without marker blocks).

#### Scenario: Invalid target rejected
- **WHEN** `init --target unknown` is run
- **THEN** the command exits non-zero listing the valid targets