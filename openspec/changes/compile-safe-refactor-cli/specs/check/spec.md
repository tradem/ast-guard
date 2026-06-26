## ADDED Requirements

### Requirement: Standalone compiler invocation with structured diagnostics

The `check` subcommand SHALL accept `--lang` and `--dir` and SHALL invoke the target language's compiler against the directory. **The compiler SHALL be invoked with its structured-output flag** (e.g., `cargo check --message-format=json`, `tsc --noEmit --pretty false`), and the output SHALL be parsed using the compiler's `parse_diagnostics` method — a pure JSON/structured-line parser, **not a regex-based stderr scraper.**

`check` SHALL NOT perform any rewriting. `check` exists as the batching partner to `refactor --no-build` and as a reusable validation step in CI and agent loops.

#### Scenario: Clean compile
- **WHEN** `ast-guard check --lang rust --dir ./src` is run and the compiler exits 0 with no diagnostics in the structured output
- **THEN** the command prints a success message and exits 0

#### Scenario: Errors returned filtered
- **WHEN** the compiler exits non-zero with diagnostics in the structured output and `--severity` is the default `warnings`
- **THEN** errors and warnings (excluding notes) are printed and the command exits non-zero

#### Scenario: No rewrite performed
- **WHEN** `check` is run against a directory
- **THEN** no file in the directory is modified by `check`

### Requirement: Severity control (central, uniform)

`check` SHALL accept `--severity` with values `errors`, `warnings` (default), and `all`. The severity filtering SHALL be performed by a **central, language-independent** `filter_diagnostics(raw, severity)` function — not per-compiler parsing logic. `errors` returns only errors; `warnings` returns errors and warnings excluding notes; `all` returns errors, warnings, and notes.

#### Scenario: Default severity is warnings
- **WHEN** `check` is run without `--severity` and the compiler produces errors, warnings, and notes in its structured output
- **THEN** errors and warnings are printed and notes are excluded

#### Scenario: Severity all includes notes
- **WHEN** `--severity all` is passed and the compiler produces notes
- **THEN** notes are included in the output

### Requirement: Offline and timeout flags

`check` SHALL accept `--offline` (maps to the target compiler's offline/no-restore flag where applicable; a no-op with a note where not applicable) and `--timeout` (default 120 seconds; the compiler process SHALL be killed if it exceeds the timeout and the command SHALL exit non-zero with a timeout error).

#### Scenario: Offline flag passed to cargo
- **WHEN** `check --lang rust --offline` is run
- **THEN** `cargo check` is invoked with `--offline`

#### Scenario: Offline flag inapplicable to tsc
- **WHEN** `check --lang typescript --offline` is run
- **THEN** the command runs `tsc --noEmit` and emits a note that `--offline` is not applicable to typescript, and proceeds normally

#### Scenario: Timeout kills a wedged compiler
- **WHEN** the compiler process runs longer than `--timeout` seconds
- **THEN** the process is killed, the command exits non-zero with a timeout error message, and no further output is read

### Requirement: Compile-root auto-discovery for check

`check` SHALL auto-discover the compile root by walking up from `--dir` until finding a language-specific manifest file (`Cargo.toml`, `tsconfig.json`, `*.csproj`, `pubspec.yaml`). The discovered directory SHALL be used as the compiler's working directory. If no manifest is found, the command SHALL exit non-zero with a message naming the expected manifest and suggesting `--project`.

#### Scenario: Check auto-discovers from subdirectory
- **WHEN** `check --lang rust --dir ./src/models` is run and the repo root contains `Cargo.toml`
- **THEN** the compile root is auto-discovered and `cargo check` runs at the repo root

#### Scenario: Manual --project override
- **WHEN** `--project /path/to/crate` is passed
- **THEN** auto-discovery is suppressed and the compiler runs in the given directory

### Requirement: CLI flag surface

The `check` subcommand SHALL accept: `--lang` (required; one of rust, csharp, typescript, dart), `--dir` (optional, default `.` — the directory to check; auto-discovers compile root from this path), `--project` (optional, overrides auto-discovered compile root), `--severity` (default `warnings`), `--offline` (default false), `--timeout` (default 120 seconds).

#### Scenario: Minimal invocation uses defaults
- **WHEN** `check --lang rust` is run
- **THEN** `--dir` is `.`, `--severity` is `warnings`, `--offline` is false, `--timeout` is 120s, compile root is auto-discovered from `--dir`