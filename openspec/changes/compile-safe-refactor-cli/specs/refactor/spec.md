## ADDED Requirements

### Requirement: Structural find-and-replace via ast-grep pattern and rewrite (delegated)

The `refactor` subcommand SHALL accept an `--lang`, a `--pattern` (ast-grep search pattern), a `--rewrite` (replacement template with `$$$META$$$` captures), and a `--dir` (target directory), and SHALL apply the structural substitution across all matching files for the target language. **All pattern-matching and substitution SHALL be delegated to `ast-grep-core` (embedded as a library). ast-guard does not implement its own pattern engine.**

Patterns that are not valid syntax of the target language SHALL be rejected before any file is read (ast-grep validates the pattern; ast-guard forwards the error).

#### Scenario: Valid pattern matches and rewrites sites
- **WHEN** `ast-guard refactor --lang rust --pattern 'fn old_name($$$A) { $$$B }' --rewrite 'fn new_name($$$A) { $$$B }' --dir ./src` is run and the pattern matches
- **THEN** every matching site is rewritten with the substitution applied and the rewrite proceeds to the safety pipeline

#### Scenario: Invalid pattern rejected before file access
- **WHEN** `ast-guard refactor --lang rust --pattern 'fn !!invalid{' --rewrite 'x' --dir ./src` is run
- **THEN** the command exits non-zero with a pattern-syntax error (from ast-grep) and no file under `--dir` is read or modified

#### Scenario: No matches is a clean no-op
- **WHEN** a valid pattern matches zero sites in `--dir`
- **THEN** the command exits 0 and reports "0 matches; no changes"

### Requirement: Syntactic re-parse gate before disk mutation

After substitution and before any file is written, the rewritten source SHALL be re-parsed with the target language's tree-sitter grammar. If the re-parsed CST contains ERROR or MISSING nodes, the command SHALL exit non-zero, name the syntax error location (line/column from the tree-sitter error node), and SHALL NOT write any file. This gate SHALL be enabled by default; `--unsafe-skip-reparse` SHALL opt out.

**Re-parse detection semantics:** tree-sitter is error-tolerant — it almost never returns a hard parse error. Instead, it produces ERROR/MISSING nodes in the CST. A "re-parse failure" means the resulting CST contains ERROR nodes (missing/MISSING nodes alone are not a failure — they are tree-sitter's recovery insertions). This is the correct, tree-sitter-aware definition of "the rewrite produced invalid syntax."

#### Scenario: Malformed rewrite caught before write
- **WHEN** a rewrite template produces unbalanced braces in the substituted output (tree-sitter reports an ERROR node)
- **THEN** the command exits non-zero with a message like "rewrite produced invalid syntax at line L, col C", and no file is written to disk

#### Scenario: Valid rewrite with MISSING nodes passes re-parse
- **WHEN** a rewrite produces valid syntax but tree-sitter's error recovery inserts MISSING nodes (e.g., missing semicolons in incomplete statements)
- **THEN** the re-parse gate passes (MISSING nodes alone are not a failure) and the pipeline proceeds

#### Scenario: Opt out of re-parse
- **WHEN** `--unsafe-skip-reparse` is passed
- **THEN** the re-parse step is skipped and the pipeline proceeds to subsequent stages without the syntactic check

### Requirement: Structural post-conditions via --require and --forbid (delta-scoped)

The `refactor` subcommand SHALL accept repeated `--require <pattern>` and `--forbid <pattern>` flags. After substitution and re-parse, each `--require` pattern SHALL match at least one site in the rewritten source, and each `--forbid` pattern SHALL match zero sites.

**Scope:** Both SHALL operate on the **files and sites that `--pattern` matched before the rewrite** (delta scope, the default). `--scope dir` SHALL expand `--forbid` to the entire `--dir` (not just the matched sites). `--require` is always delta-scoped — there is no use case for global `--require`.

If any post-condition fails, the command SHALL exit non-zero, name the offending pattern and the first offending site (file:line:col), and SHALL NOT write any file. Both flags SHALL accept the same ast-grep pattern grammar as `--pattern`.

#### Scenario: Require succeeds when new form present in rewritten files
- **WHEN** `--require 'fn new_name($$$A) { $$$B }'` is passed and the rewritten source contains at least one match in the files that were rewritten
- **THEN** the require check passes and the pipeline continues. A pre-existing `fn new_name` in `src/unrelated.rs` (a file not touched by the rewrite) does not satisfy the requirement.

#### Scenario: Forbid fails on a straggler in a matched file
- **WHEN** `--forbid 'console.log($$$A)'` is passed and the rewritten source still contains one `console.log(...)` site in `src/utils.ts:87` (a file that `--pattern` matched before the rewrite)
- **THEN** the command exits non-zero with a message naming the pattern and "src/utils.ts:87", and no file is written

#### Scenario: Forbid ignores pre-existing matches in untouched files
- **WHEN** `--forbid 'old_api($$$A)'` is passed and `--pattern` matched only in `src/models/`, but `src/legacy/old_impl.rs` also contains `old_api()` (a file never in scope of the rewrite)
- **THEN** `--forbid` does NOT fail — `src/legacy/old_impl.rs` was not touched by the rewrite and its pre-existing content is irrelevant. Only the delta (rewritten files) is checked.

#### Scenario: Multiple post-conditions all must hold
- **WHEN** both `--require A` and `--forbid B` are passed and `--require A` matches but `--forbid B` also matches a site in a rewritten file
- **THEN** the command exits non-zero naming the `--forbid B` violation, and no file is written

#### Scenario: --scope dir expands forbid to entire --dir
- **WHEN** `--scope dir --forbid 'unsafe'` is passed and `unsafe` appears in any file under `--dir` (even files not matched by `--pattern`)
- **THEN** the command exits non-zero naming the `--forbid 'unsafe'` violation. This mode is for CI/global validation, not refactor workflows.

### Requirement: Dry-run mode emits diff and does not write

When `--dry-run` is passed, the command SHALL produce a unified diff of the in-memory changes versus the filesystem, SHALL report the re-parse status, SHALL NOT write any file, and SHALL exit 0 regardless of whether matches were found (unless a safety gate fails — re-parse or require/forbid failures still exit non-zero on dry-run).

#### Scenario: Dry-run shows diff without writing
- **WHEN** `--dry-run` is passed and the rewrite would change two files
- **THEN** a unified diff for both files is printed to stdout, no file is written to disk, and the command exits 0

#### Scenario: Dry-run reports re-parse failure without writing
- **WHEN** `--dry-run` is passed and the substituted output fails re-parse (ERROR nodes in CST)
- **THEN** the syntax error location is printed, no file is written, and the command exits non-zero

### Requirement: Advisory hints on dry-run only

When `--dry-run` is passed and the rewrite structurally resembles a rename (changes an identifier) or a deletion (removes a node), the command SHALL emit a non-fatal advisory hint suggesting the appropriate `--require`/`--forbid` category. Hints SHALL advise the constraint *category* only and SHALL NOT synthesize pattern content. Hints SHALL name the `--quiet-hints` escape inline. Hints SHALL NOT be emitted when applying (non-dry-run).

#### Scenario: Rename-shaped rewrite triggers hint
- **WHEN** `--dry-run` is passed and the rewrite changes an identifier `$$N`
- **THEN** stdout includes a hint suggesting `--forbid '<pre-rewrite form of $$N>'` and `--require '<post-rewrite form of $$N>'`, and naming `--quiet-hints` as the suppressor

#### Scenario: Pure addition emits no hint
- **WHEN** `--dry-run` is passed and the rewrite only appends content without changing or deleting existing named nodes
- **THEN** no advisory hint is emitted

#### Scenario: --quiet-hints suppresses hints
- **WHEN** `--dry-run --quiet-hints` is passed
- **THEN** no advisory hints are emitted, while hard errors and `--require`/`--forbid` violations still report normally

### Requirement: Write, optional format, optional compile

When not in dry-run mode and all pre-write gates pass, the command SHALL write the rewritten files to disk. If `--fmt` is passed, the command SHALL invoke the target language's formatter after writing and before compiling. If `--no-build` is passed, the command SHALL exit 0 after writing (and optional formatting) without invoking the compiler. Otherwise the command SHALL invoke the compiler and report filtered diagnostics.

#### Scenario: Default applies and compiles
- **WHEN** `refactor` is run without `--dry-run`, `--no-build`, or `--fmt`
- **THEN** files are written, no formatter is invoked, the compiler is invoked, and filtered diagnostics are reported

#### Scenario: --no-build writes without compiling
- **WHEN** `--no-build` is passed
- **THEN** files are written, the compiler is not invoked, and the command exits 0

#### Scenario: --fmt formats after write
- **WHEN** `--fmt` is passed (and `--no-build` is not)
- **THEN** files are written, the target language's formatter is invoked, then the compiler is invoked, and filtered diagnostics are reported

### Requirement: Compile-root auto-discovery (separate from --dir)

The `refactor` subcommand SHALL auto-discover the compile root by walking up from `--dir` (the rewrite target directory) until finding a language-specific manifest file: `Cargo.toml` (Rust), `tsconfig.json` (TypeScript), `*.csproj` or `*.sln` (C#), `pubspec.yaml` (Dart). The discovered directory SHALL be used as the working directory for compiler invocation. If no manifest is found up to the filesystem root, the command SHALL exit non-zero with a message naming the expected manifest type and suggesting `--project` as an explicit override.

#### Scenario: Auto-discovery from subdirectory
- **WHEN** `refactor --lang rust --dir ./src/models` is run from a git repo whose root contains `Cargo.toml`
- **THEN** the compile root is automatically set to the repo root (where `Cargo.toml` lives), and `cargo check` runs there, not in `./src/models`

#### Scenario: Manual override with --project
- **WHEN** `--project /path/to/crate` is passed
- **THEN** auto-discovery is suppressed and the compiler runs in the given directory

#### Scenario: No manifest found
- **WHEN** `refactor --lang rust --dir ./src` is run and no `Cargo.toml` exists in any parent directory up to `/`
- **THEN** the command exits non-zero with a message like "No Cargo.toml found; specify --project to set the compile root manually"

### Requirement: Filtered compiler diagnostics with structured output

When the compiler is invoked, the command SHALL invoke the compiler with its **structured output flag** (e.g., `cargo check --message-format=json`, `tsc --noEmit --pretty false`, `dotnet build` with structured logging) and SHALL parse the machine-readable output into a `Vec<Diagnostic>` using the compiler's `parse_diagnostics` method — a pure JSON/structured-line parser, **not a regex-based stderr scraper**.

The parsed diagnostics SHALL be filtered centrally by `filter_diagnostics(raw, severity)` — a uniform, language-independent function. The default severity SHALL be `warnings` (errors and warnings, excluding notes). On compiler success (no matching diagnostics, exit 0), the command SHALL exit 0 with a success message. On compiler failure, the command SHALL print the filtered diagnostics and exit non-zero; the mutated files SHALL remain on disk.

#### Scenario: Successful compile
- **WHEN** the compiler exits 0 and produces no errors or warnings in the structured output
- **THEN** the command prints a success message and exits 0

#### Scenario: Compile failure returns filtered errors and warnings
- **WHEN** the compiler exits non-zero and `--severity` is the default `warnings`
- **THEN** the command prints errors and warnings (excluding notes) from the structured output and exits non-zero, and the mutated files remain on disk

#### Scenario: Severity errors drops warnings
- **WHEN** the compiler exits non-zero producing both errors and warnings and `--severity errors` is passed
- **THEN** only errors are printed, warnings are dropped, and the command exits non-zero

### Requirement: CLI flag surface

The `refactor` subcommand SHALL accept the following flags: `--lang` (required; one of rust, csharp, typescript, dart), `--pattern` (required), `--rewrite` (required), `--dir` (optional, default `.`), `--project` (optional, overrides auto-discovered compile root), `--dry-run` (default false), `--no-build` (default false), `--fmt` (default false), `--unsafe-skip-reparse` (default false), `--require` (repeatable), `--forbid` (repeatable), `--scope` (default `forbid`; one of `forbid`, `dir` — controls `--forbid` scope only), `--offline` (default false), `--timeout` (default 120 seconds), `--severity` (default `warnings`; one of `errors`, `warnings`, `all`), `--quiet-hints` (default false).

#### Scenario: Defaults are safe
- **WHEN** `refactor` is run with only required flags
- **THEN** re-parse is enabled, severity is `warnings`, hints are enabled, build is enabled, fmt is disabled, offline is disabled, timeout is 120s, `--forbid` scope is `forbid` (delta), `--dir` defaults to `.`, compile root is auto-discovered from `--dir`

#### Scenario: Unsupported language rejected
- **WHEN** `--lang python` is passed in v1
- **THEN** the command exits non-zero with a message listing the supported languages (rust, csharp, typescript, dart)