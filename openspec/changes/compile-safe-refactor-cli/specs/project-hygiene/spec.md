## ADDED Requirements

### Requirement: Standard CI checks (no cargo-husky, no mutation-testing gate)

The project SHALL use standard Rust tooling for CI: `cargo fmt --check`, `cargo clippy -- -D warnings`, and `cargo test`. No cargo-husky pre-commit/pre-push hooks SHALL be installed in v1. No cargo-mutants merge-gate SHALL be enforced in v1. Mutation testing is a Phase 3 quality tool.

#### Scenario: CI blocks on formatting
- **WHEN** CI runs `cargo fmt --check` and code is unformatted
- **THEN** the CI step fails and the contributor is told to run `cargo fmt`

#### Scenario: CI blocks on clippy warnings
- **WHEN** CI runs `cargo clippy -- -D warnings` and clippy produces a warning
- **THEN** the CI step fails with the clippy diagnostic

#### Scenario: CI runs tests
- **WHEN** CI runs `cargo test`
- **THEN** the CI step fails if any test fails

### Requirement: No conventional-commit enforcement hook

The project SHALL NOT enforce conventional commit messages via git hooks or CI in v1. Conventional commits are a **soft convention** documented in `AGENTS.md` (agent-respected). No regex hook SHALL be installed. No CI commit-message check SHALL be enforced. This is a deliberate Phase 3 deferral — the tool is too young to justify commit-message ceremony.

#### Scenario: Non-conventional commit is NOT rejected
- **WHEN** a commit message `fixed the bug` is created
- **THEN** no git hook rejects it. The commit proceeds normally. The AGENTS.md soft rule may advise but does not enforce.

### Requirement: Project-specific AGENTS.md for hacking on ast-guard

The project SHALL include an `AGENTS.md` at the repository root that is the agent-facing guide to contributing to ast-guard itself. It SHALL cover: the architecture (trait seams, staged pipeline, delegation to `ast-grep-core`), the three-phase delivery strategy (skill → lightweight CLI → mature CLI), the dry-run-first workflow for refactors *within* ast-guard, the security/non-sandbox posture, and the lightweight hygiene (no mutation-testing gate, no commit hooks).

It SHALL NOT be the canonical home of the CRITICAL RULE shipped to *users* via `init` — that lives in the `src/agents/` templates. The repository-root `AGENTS.md` and the `init`-shipped templates are distinct documents.

#### Scenario: AGENTS.md orients a new agent contributor
- **WHEN** an agent opens the repo to contribute to ast-guard
- **THEN** `AGENTS.md` explains the architecture, the delegation to `ast-grep-core`, the three-phase strategy, and the dry-run-first refactor workflow for changes to ast-guard itself

#### Scenario: AGENTS.md is distinct from init-shipped templates
- **WHEN** `AGENTS.md` and the `src/agents/agents.md` template are compared
- **THEN** they are distinct documents: the repo-root `AGENTS.md` governs contributions to ast-guard; the template governs users of ast-guard in their own projects

### Requirement: No mutation-testing merge-gate

The project SHALL NOT enforce `cargo-mutants` as a merge-gate in v1. Mutation testing is a future-phase quality tool for mature codebases. The core modules (`reparse`, `filter`, `dispatch`, `runner`, `rewrite`, `init`) are tested through integration tests with `StubRunner` and real-compiler smoke tests. `mutants.toml` is not required in v1.

#### Scenario: Surviving mutant is informational
- **WHEN** `cargo-mutants` is optionally run and reports a surviving mutation
- **THEN** the mutant is informational and does not block merge. The contributor may optionally add a test to kill it.

### Requirement: .gitignore includes ast-guard and mutants artifacts

The project `.gitignore` SHALL include `target/`, `**/mutants.out*/` (if mutation testing is ever run), and `.ast-guard/` (reserved for future use even though unused in v1). These are standard Rust project exclusions.

#### Scenario: Build artifacts not committed
- **WHEN** `cargo build` runs
- **THEN** `target/` is ignored and not committed