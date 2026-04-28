# Effective Java (3rd Edition) — Methods Common to All Objects

This chapter ensures foundational object behavior is consistent, predictable, and safe for collections, logging, and APIs.

## Key best practices

1. **Obey the `equals` contract**
   - Implement `equals` only when logical equality differs from identity.
   - Ensure reflexive, symmetric, transitive, consistent behavior and non-null comparison.

2. **Always override `hashCode` when overriding `equals`**
   - Equal objects must produce equal hash codes.
   - Use stable, high-quality field combinations.

3. **Override `toString` for diagnostics and observability**
   - Return concise, useful, human-readable state.
   - Keep format stable enough for debugging, but document if consumed externally.

4. **Use `clone` very cautiously**
   - Prefer copy constructors or static factory copy methods.
   - If cloning, perform deep-copying of mutable internal state.

5. **Implement `Comparable` thoughtfully**
   - Provide natural ordering consistent with `equals` when possible.
   - Use type-safe comparisons and avoid subtraction overflow in comparisons.

## Practical checklist for code reviews

- Does `equals` compare the right fields and respect the full contract?
- Is `hashCode` implemented and aligned with `equals`?
- Is `toString` useful for troubleshooting?
- Is cloning avoided unless absolutely required?
- Is natural ordering well-defined and stable?
