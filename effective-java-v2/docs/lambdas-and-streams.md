# Effective Java (3rd Edition) — Lambdas and Streams

This chapter focuses on expressing behavior concisely while keeping code readable, testable, and free of hidden side effects.

## Key best practices

1. Prefer lambdas to anonymous classes for function objects.
2. Use method references when they improve readability.
3. Prefer standard functional interfaces (`Function`, `Predicate`, `Supplier`, etc.).
4. Use streams for bulk operations when they make intent clearer than loops.
5. Keep stream pipelines simple, side-effect free, and easy to debug.
6. Be careful with parallel streams; use only when correctness and performance are validated.

## Practical checklist for code reviews

- Is stream/lambda code clearer than the equivalent loop?
- Are side effects avoided inside pipeline stages?
- Is parallelization justified with evidence?
