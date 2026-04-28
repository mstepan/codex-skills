# Effective Java (3rd Edition) — Serialization

This chapter treats serialization as a long-term API commitment with security and compatibility implications.

## Key best practices

1. Prefer alternatives to Java serialization for long-term protocols and persistence formats.
2. If serialization is required, design it deliberately and document compatibility guarantees.
3. Implement `readObject` defensively; validate invariants and perform defensive copies.
4. Consider serialization proxies for robust, maintainable serialized forms.
5. Use `readResolve`/`writeReplace` where needed for singleton or invariant preservation.
6. Keep serialized forms as simple and stable as possible.

## Practical checklist for code reviews

- Is Java serialization truly necessary for this use case?
- Are deserialization paths validated and safe?
- Is serialized form stability/versioning considered?
- Are invariants preserved across serialization boundaries?
