# Effective Java (3rd Edition) — Methods

This chapter focuses on API ergonomics, strong contracts, and defensive method design.

## Key best practices

1. Choose method names that clearly express intent.
2. Keep parameter lists short; use helper types/builders for many optional parameters.
3. Validate parameters early and fail fast with informative exceptions.
4. Return empty collections/arrays instead of `null`.
5. Use `Optional` judiciously for absent return values (not for fields/parameters in most cases).
6. Document behavior, side effects, nullability, and thread-safety expectations.

## Practical checklist for code reviews

- Are method contracts clear and documented?
- Are arguments validated consistently?
- Are `null` returns avoided where practical?
- Is `Optional` used appropriately (not overused)?
