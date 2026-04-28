# Effective Java (3rd Edition) — Classes and Interfaces

This chapter focuses on robust type design, clear abstraction boundaries, and minimizing accidental complexity.

## Key best practices

1. Minimize accessibility of classes and members.
2. In public classes, use accessors rather than exposing mutable fields.
3. Minimize mutability; make fields `final` where possible.
4. Favor composition over inheritance for code reuse.
5. Design and document inheritance explicitly, or prohibit it.
6. Prefer interfaces to abstract classes for extensibility.
7. Use interface types for references, parameters, and return values.
8. Prefer class hierarchies for tagged data over ad-hoc tag fields.

## Practical checklist for code reviews

- Are visibility and encapsulation as restrictive as possible?
- Is mutability justified and controlled?
- Is inheritance used only with a documented extension contract?
- Are interfaces used to decouple API from implementation?
