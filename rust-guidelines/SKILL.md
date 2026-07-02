---
name: rust-guidelines
description: Use when designing, implementing, refactoring, reviewing, or documenting Rust code and you want project-wide guidance on idiomatic APIs, correctness, documentation, performance, safety, library design, application setup, and AI-friendly Rust patterns.
---

# Rust Guidelines

Use this skill when working in Rust codebases that should follow the rules collected in `rust-all-rules.md`. The skill keeps the trigger text small and uses the bundled reference file for the full guideline set.

## Reference Map

Load the full rulebook when broad guidance is needed:

- [rust-all-rules.md](references/rust-all-rules.md): complete Pragmatic Rust rule set.

Use these section anchors or text search to jump to the relevant area inside the reference:

- `# AI Guidelines`: AI-friendly API and documentation guidance.
- `# Application Guidelines`: application allocator, error handling, target CPU.
- `# Correctness Guidelines`: panic behavior, tests, semantics, invariants.
- `# Documentation`: crate, module, and public API docs rules.
- `# FFI Guidelines`: interop rules and foreign boundary constraints.
- `# Library Guidelines`: public API and library-facing design rules.
- `# Macros Guidelines`: macro structure and helper visibility.
- `# Performance Guidelines`: allocation, cloning, async, and efficiency guidance.
- `# Project Guidelines`: workspace and crate layout expectations.
- `# Safety`: unsafe code expectations and containment.
- `# Universal Guidelines`: naming, signatures, visibility, linting, and broad Rust conventions.
- `# Libraries / Building Guidelines`: build organization and dependency guidance.
- `# Libraries / Interoperability Guidelines`: trait/object/FFI and ecosystem interoperability rules.
- `# Libraries / Resilience Guidelines`: resilience, statics, concurrency, and failure handling.
- `# Libraries / UX Guidelines`: API ergonomics, errors, builders, and user-facing patterns.

## Core Workflow

1. Inspect the existing crate layout, public API surface, tests, docs, and workspace configuration before editing.
2. Choose the narrowest guideline area that matches the task instead of treating the rulebook as a single undifferentiated checklist.
3. Prefer idiomatic Rust over patterns copied from Java, C#, Python, or C++ codebases.
4. Keep public APIs strongly typed, well documented, and reachable through a single clear path.
5. When changing behavior, add or update meaningful tests that verify observable outcomes rather than tautologies or implementation mirrors.
6. Re-check docs, exports, error types, ownership, async boundaries, and visibility after refactors because those are common regression points in Rust code.

## Default Expectations

- Favor canonical Rust APIs, naming, module structure, and ownership semantics.
- Use strong domain types instead of raw primitives where semantics matter.
- Keep user-facing documentation focused on the stable end state, not the design journey.
- Design libraries and applications differently where the rules say to do so.
- Prefer explicit, testable behavior over clever abstractions or cross-language translations.

## Common Mistakes

- Porting OO or GC-language patterns directly into Rust.
- Re-exporting the same public item from multiple paths.
- Adding tests that only restate constants, branches, or implementation details.
- Mixing application-style error crates into shared libraries.
- Letting module docs drift into design journals or meta commentary.
- Using the rulebook as justification for broad rewrites instead of targeting the relevant constraints for the task at hand.
