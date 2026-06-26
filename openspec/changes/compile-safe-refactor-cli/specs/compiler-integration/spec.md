## ADDED Requirements

### Requirement: Pluggable compiler via Compiler trait (Strategy) — structured output, not regex

The system SHALL define a `Compiler` trait that abstracts per-language compiler behavior. Each supported language (rust, csharp, typescript, dart) SHALL have a concrete implementation.

The trait SHALL expose:
- The language it handles.
- A method to build the compiler `CommandSpec` for a given compile context (including offline flag handling).
- A method `structured_output_flag(&self) -> &'static str` returning the flag that enables machine-readable output (e.g., `"--message-format=json"` for cargo, `"--pretty false"` for tsc).
- A **pure** method `parse_diagnostics(&self, raw: &[u8]) -> Result<Vec<RawDiagnostic>>` that parses the compiler's machine-readable output (JSON, structured lines, or structured text) into a `Vec<RawDiagnostic>`. **This method SHALL NOT use regex to parse stderr.** It parses structured, machine-readable output only.

The dispatcher SHALL select the compiler implementation by `Language` enum via exhaustive `match`.

**Severity filtering SHALL NOT happen in `parse_diagnostics`.** It happens in a central, language-independent `filter_diagnostics(raw: Vec<RawDiagnostic>, severity: Severity) -> Vec<Diagnostic>` function, called after parsing. This keeps the parsing logic clean (no severity logic per language) and the filtering logic uniform.

#### Scenario: Each language has a compiler implementation
- **WHEN** the dispatcher resolves a compiler for `Language::Rust`, `Language::CSharp`, `Language::TypeScript`, and `Language::Dart`
- **THEN** each returns a distinct `Compiler` implementation with the correct command, structured-output flag, and parse_diagnostics parser for that language

#### Scenario: Exhaustive match prevents missing language
- **WHEN** a new `Language` variant is added without a compiler implementation
- **THEN** the code fails to compile (exhaustive match), preventing a runtime "unsupported language" gap

#### Scenario: Structured output flag is always appended to compiler command
- **WHEN** a `Compiler` implementation provides a `structured_output_flag` (e.g., `"--message-format=json"`)
- **THEN** the `Runner` SHALL append this flag to every compiler invocation. The compiler is never called in human-readable mode.

### Requirement: Process spawning via Runner trait (Adapter) — captures structured output

The system SHALL define a `Runner` trait as the sole seam over `std::process::Command`. `RealRunner` SHALL spawn processes with:
- The binary resolved to an absolute path (via the `which` crate).
- An explicit canonicalized current directory (the **compile root**, not `--dir` — these are separate).
- A closed stdin.
- A timeout that kills the process on expiry.
- An output size cap that stops reading after N megabytes.
- **The compiler's structured-output flag appended to every command** (from `Compiler::structured_output_flag()`).
- **stdout captured as the primary output** (compilers write structured diagnostics to stdout, not stderr).

`Runner` SHALL be the only place `std::process::Command` is constructed. Tests SHALL inject a `StubRunner` with canned `RunOutput` values to drive the pipeline without spawning real processes.

#### Scenario: RealRunner resolves absolute binary path
- **WHEN** `RealRunner` runs a `CommandSpec` naming `cargo`
- **THEN** the spawned process uses the absolute path resolved by `which`, not a bare `cargo` lookup by the shell

#### Scenario: Timeout kills a wedged process
- **WHEN** a spawned process runs longer than the configured timeout
- **THEN** the process is killed and `Runner::run` returns an error indicating timeout

#### Scenario: Output size cap stops runaway output
- **WHEN** a spawned process produces more than the configured maximum output
- **THEN** reading stops at the cap and the returned `RunOutput` is marked truncated

#### Scenario: StubRunner injected in tests
- **WHEN** a pipeline test runs with `StubRunner` holding canned compiler output
- **THEN** no real process is spawned and the pipeline behaves as if the compiler produced the canned output

#### Scenario: Structured output flag appended to every command
- **WHEN** `RealRunner` runs a `CommandSpec` for `RustCompiler`
- **THEN** the command includes `--message-format=json` (from `Compiler::structured_output_flag()`) even if the user did not explicitly request it

### Requirement: No shell invocation

The system SHALL never invoke a shell (`sh -c`, `cmd /c`) when spawning compilers. All command arguments SHALL be passed via the process-spawning API's argument vector, treating patterns and rewrites as data, not as shell strings.

#### Scenario: Pattern with shell metacharacters is data
- **WHEN** a rewrite contains `;`, `|`, or `$()` characters
- **THEN** those characters are passed literally to ast-grep as rewrite data and are never interpreted by a shell

### Requirement: Diagnostic model and centralized severity filtering

The system SHALL define:
- `RawDiagnostic`: a type with fields `severity: Severity`, `code: Option<String>`, `file: Option<PathBuf>`, `line: Option<u32>`, `column: Option<u32>`, `message: String`, `span: Option<Span>` (start/end byte offsets), `related: Vec<RelatedDiagnostic>`. This is the raw, unfiltered diagnostic from the compiler's structured output.
- `Severity` enum: `Error`, `Warning`, `Note`.
- `filter_diagnostics(raw: Vec<RawDiagnostic>, severity: Severity) -> Vec<Diagnostic>`: a **central, language-independent** function that filters by severity level. This is NOT per-compiler logic — it is one function for all languages.

The `Diagnostic` type (final, filtered) SHALL contain the same fields as `RawDiagnostic` but with guaranteed non-optional values for `file`, `line`, `message` (compilers SHOULD always provide these, but if they don't, the diagnostic is still emitted with placeholder values).

#### Scenario: Parse is pure (JSON/structured parser)
- **WHEN** `Compiler::parse_diagnostics` is called twice with identical structured output
- **THEN** identical `Vec<RawDiagnostic>` is returned and no I/O or global state is touched

#### Scenario: Severity warnings excludes notes
- **WHEN** `filter_diagnostics` is called with `Severity::Warning` and a `Vec<RawDiagnostic>` containing an error, a warning, and a note
- **THEN** the returned diagnostics contain the error and the warning and exclude the note

#### Scenario: Diagnostic includes byte spans from structured output
- **WHEN** the compiler's structured output includes byte span information (e.g., cargo's `byte_start`/`byte_end`)
- **THEN** the `RawDiagnostic` SHALL include the span, and the filtered output SHALL report the exact byte range so the agent can locate the issue precisely

### Requirement: Offline flag handling per language

When `--offline` is passed, the `Compiler::command` implementation SHALL include the target compiler's offline/no-restore flag where one exists (`cargo --offline`, `dotnet build --no-restore`) and SHALL be a no-op with an emitted note where no such flag exists (tsc). Passing `--offline` to a language without an offline flag SHALL NOT be an error.

#### Scenario: Rust offline uses cargo --offline
- **WHEN** `--offline` is passed for `Language::Rust`
- **THEN** the `CommandSpec` includes `--offline` as a cargo argument

#### Scenario: TypeScript offline is a noted no-op
- **WHEN** `--offline` is passed for `Language::TypeScript`
- **THEN** the `CommandSpec` is unchanged and a note is emitted that `--offline` is not applicable to typescript

### Requirement: Binary resolution and absence reporting

When a target compiler binary cannot be resolved on PATH, the system SHALL exit non-zero with a clear message naming the missing binary and the language, and SHALL NOT spawn any process. The resolved absolute path of each invoked binary SHALL be included in the command's output so the agent can see which compiler ran.

#### Scenario: Missing compiler is a clear error
- **WHEN** `dotnet` is not on PATH and `check --lang csharp` is run
- **THEN** the command exits non-zero with a message like "dotnet not found on PATH; cannot compile C#", and no process is spawned

#### Scenario: Resolved path is reported
- **WHEN** `cargo` is resolved to `/home/user/.cargo/bin/cargo` and a compile is run
- **THEN** the output includes the resolved absolute path so the agent can verify which binary ran

### Requirement: Manifest-based compile root auto-discovery

The system SHALL auto-discover the compile root by walking up from `--dir` until finding a language-specific manifest file: `Cargo.toml` (Rust), `tsconfig.json` (TypeScript), `*.csproj` or `*.sln` (C#), `pubspec.yaml` (Dart). The discovered directory SHALL be used as the working directory for compiler invocation (`Command::current_dir(canonicalize(compile_root))`). If no manifest is found up to the filesystem root, the command SHALL exit non-zero with a message naming the expected manifest type and suggesting `--project` as an explicit override.

#### Scenario: Auto-discovery from subdirectory
- **WHEN** `refactor --dir ./src/models --lang rust` is run from a git repo whose root contains `Cargo.toml`
- **THEN** the compile root is automatically set to the repo root (where `Cargo.toml` lives), and `cargo check` runs there

#### Scenario: Manual override with --project
- **WHEN** `--project /path/to/crate` is passed
- **THEN** auto-discovery is suppressed and the compiler runs in the given directory

#### Scenario: No manifest found
- **WHEN** `refactor --lang rust --dir ./src` is run and no `Cargo.toml` exists in any parent directory up to `/`
- **THEN** the command exits non-zero with a message like "No Cargo.toml found; specify --project to set the compile root manually"