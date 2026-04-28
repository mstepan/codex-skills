# Effective Java (3rd Edition) — General Programming

This chapter highlights language and library habits that improve correctness, portability, and maintainability.

## Key best practices

1. Minimize scope of local variables and initialize near first use.
2. Prefer enhanced `for` loops where indexing is unnecessary.
3. Know and use libraries (`java.util`, `java.time`, `java.nio`, etc.) instead of reinventing.
4. Prefer primitive types over boxed types in performance-sensitive code.
5. Avoid floating-point for exact decimal calculations; use `BigDecimal` when precision matters.
6. Write clear, deterministic code around strings, localization, and formatting.

## Practical checklist for code reviews

- Is library functionality reused instead of custom ad-hoc implementations?
- Are numeric types chosen correctly for domain precision/performance?
- Is variable scope tight and code easy to read?
