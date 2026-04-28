# Effective Java (3rd Edition) — Generics

This chapter is about maximizing compile-time type safety while keeping APIs expressive and practical.

## Key best practices

1. Don’t use raw types in new code.
2. Eliminate unchecked warnings rather than suppressing blindly.
3. Prefer lists to arrays for generic APIs.
4. Favor generic types and generic methods when type-parameterized behavior is needed.
5. Use bounded wildcards to increase API flexibility (`? extends T`, `? super T`).
6. Document and centralize any unavoidable unchecked casts.

## Practical checklist for code reviews

- Are raw types avoided?
- Are wildcard bounds chosen to support producers/consumers correctly?
- Are suppressions narrowly scoped and justified with comments?
