---
name: effective-java-v2
description: Use when implementing, refactoring, or reviewing Java code and needing concise Effective Java 3rd edition guidance on object creation, immutability, APIs, generics, enums, lambdas, exceptions, concurrency, serialization, equals, hashCode, toString, Comparable, or defensive programming.
---

# Java Skill — Effective Java (3rd Edition)

Use this skill to apply **Effective Java (3rd edition)** best practices while implementing or reviewing Java code.

## How this skill is organized

```
java/
├── SKILL.md
└── docs/
    ├── creating-and-destroying-objects.md
    ├── methods-common-to-all-objects.md
    ├── classes-and-interfaces.md
    ├── generics.md
    ├── enums-and-annotations.md
    ├── lambdas-and-streams.md
    ├── methods.md
    ├── general-programming.md
    ├── exceptions.md
    ├── concurrency.md
    └── serialization.md
```

- `SKILL.md` contains core guidance and a chapter map.
- `docs/` contains chapter-specific summaries (one chapter per document).

## Core principles (always apply)

1. Prefer clear, readable APIs over cleverness.
2. Design for correctness first, then optimize with evidence.
3. Favor immutability and defensive copying for safety.
4. Make illegal states unrepresentable through type/API design.
5. Document contracts, nullability, and thread-safety expectations.
6. Keep scope of change minimal and aligned to the task.

## Effective Java chapter map (quick summary)

- **Creating and Destroying Objects**: prefer static factories when useful, enforce invariants in construction, and manage object lifecycle explicitly.
- **Methods Common to All Objects**: implement `equals`, `hashCode`, `toString`, `clone`, and `Comparable` consistently.
- **Classes and Interfaces**: minimize mutability, design stable abstractions, and prefer interfaces to implementation inheritance.
- **Generics**: use generics to maximize type safety and minimize unchecked operations.
- **Enums and Annotations**: prefer enums over int constants and annotations over naming conventions.
- **Lambdas and Streams**: use lambdas/streams when they improve clarity; avoid overuse and side effects.
- **Methods**: design signatures for usability, validate parameters, and return empty collections/optionals rather than null.
- **General Programming**: write robust, clear code with careful attention to numerics, strings, and APIs.
- **Exceptions**: use exceptions for exceptional conditions only; provide actionable messages and preserve causes.
- **Concurrency**: synchronize shared mutable state correctly and prefer high-level concurrency utilities.
- **Serialization**: treat serialization as an explicit, risky API; prefer alternatives when possible.

## Repository-specific execution notes (hold-my-nats)

- Java version: **25 LTS**
- Build tool: **Maven Wrapper** (`./mvnw`, `mvnw.cmd`)
- Concurrency model used in project: **Virtual Threads** + **Structured Concurrency (preview)**
- Run tests before finalizing non-trivial changes:
  - Unix-like: `./mvnw -q test`
  - Windows: `mvnw.cmd -q test`
