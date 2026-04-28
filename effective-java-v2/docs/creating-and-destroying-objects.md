# Effective Java (3rd Edition) — Creating and Destroying Objects

This chapter focuses on safe object construction, clear lifecycle management, and avoiding unnecessary allocation.

## Key best practices

1. **Consider static factory methods instead of constructors**
   - Give methods meaningful names (`of`, `from`, `valueOf`, `newInstance`, etc.).
   - Reuse cached instances when appropriate.
   - Return subtypes or hidden implementations to keep APIs flexible.

2. **Use builders when constructors have many parameters**
   - Prefer builders for readability and maintainability.
   - Keep object creation fluent while preserving validation before `build()`.

3. **Enforce singleton semantics correctly**
   - Prefer enum singleton for serialization/reflection safety.
   - If not possible, use a private constructor + static factory + defensive `readResolve` strategy.

4. **Enforce non-instantiability for utility classes**
   - Use a private constructor and optionally throw assertion error in it.

5. **Prefer dependency injection over hard-wiring resources**
   - Inject required collaborators via constructor/factory.
   - Improves testability and modularity.

6. **Avoid creating unnecessary objects**
   - Reuse immutable values and expensive objects where reasonable.
   - Prefer primitives over boxed types in performance-sensitive paths.
   - Avoid accidental autoboxing in loops/critical code.

7. **Eliminate obsolete object references**
   - Clear references in custom memory-managing structures.
   - Watch for memory leaks via caches, listeners, callbacks, and thread locals.

8. **Avoid finalizers and cleaners for normal resource management**
   - Use `try-with-resources` and explicit close semantics.
   - Finalization is unpredictable and can hurt performance/correctness.

9. **Use try-with-resources whenever possible**
   - Prefer `AutoCloseable` resources with deterministic cleanup.
   - Keep resource scopes small and explicit.

## Practical checklist for code reviews

- Is object creation API clear (constructor vs static factory vs builder)?
- Are invariants validated at construction time?
- Are expensive/immutable objects reused appropriately?
- Is resource cleanup deterministic (`try-with-resources`)?
- Are there risks of leaks from lingering references/caches/listeners?
