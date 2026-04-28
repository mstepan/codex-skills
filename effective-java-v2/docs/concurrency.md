# Effective Java (3rd Edition) — Concurrency

This chapter is about building correct concurrent code first, then improving throughput/latency safely.

## Key best practices

1. Synchronize access to shared mutable state.
2. Prefer immutable data and confinement to reduce synchronization needs.
3. Use higher-level concurrency utilities (`ExecutorService`, concurrent collections, synchronizers).
4. Keep synchronized regions small; avoid calling alien methods while holding locks.
5. Avoid excessive synchronization; profile before optimizing.
6. Document thread-safety guarantees clearly.

## Practical checklist for code reviews

- Is shared mutable state minimized and clearly protected?
- Are concurrency utilities preferred over low-level primitives where possible?
- Are thread-safety expectations documented?
- Could lock ordering/contention cause deadlocks or bottlenecks?
