# ast-guard
A compile-safe AST refactoring tool for AI coding agents. Safely execute structural rewrites with automatic compiler validation and git rollbacks

When AI coding agents attempt large-scale refactorings, they often rely on fragile regex or script-based text replacements that break syntax, misalign brackets, and destroy type safety.

[ast-guard](https://github.com/tradem/ast-guard/e) solves this by providing an atomic, test-driven refactoring pipeline. Powered by ast-grep-core and written in Rust, it allows AI agents to apply structural AST mutations in-memory, instantly validates them against the target compiler (cargo, dotnet, tsc), and automatically rolls back all changes if compilation fails.
