# Effective Java (3rd Edition) — Enums and Annotations

This chapter covers type-safe constants, extensible enum patterns, and metadata-driven design with annotations.

## Key best practices

1. Use enums instead of `int`/`String` constants.
2. Prefer instance fields and methods on enums over ordinal-based logic.
3. Use `EnumSet`/`EnumMap` for high-performance enum collections.
4. Use annotations over naming conventions for declarative behavior.
5. Prefer standard annotations where they fit; define custom ones sparingly and clearly.

## Practical checklist for code reviews

- Are constants modeled as enums where applicable?
- Is enum behavior encoded in methods rather than switches on ordinals?
- Are annotation contracts documented and retained with correct targets?
