# Effective Java (3rd Edition) — Exceptions

This chapter focuses on modeling failures cleanly and making error handling actionable and maintainable.

## Key best practices

1. Use exceptions only for exceptional conditions, not normal control flow.
2. Prefer checked exceptions for recoverable conditions, runtime exceptions for programming errors.
3. Throw exceptions appropriate to the abstraction level.
4. Include useful failure-capture details in exception messages.
5. Preserve causes when wrapping lower-level exceptions.
6. Keep methods exception-safe; avoid partial state updates when failures occur.

## Practical checklist for code reviews

- Is exception type selection aligned with recoverability?
- Do messages include enough context for diagnosis?
- Are causes preserved when translating exceptions?
- Is object state left consistent after failures?
