# Redis Access with Micronaut Redis

Reference: https://micronaut-projects.github.io/micronaut-redis/6.9.0/guide/

## Core Principle

Treat Redis as an infrastructure concern: expose domain-oriented ports in domain/application layers and implement Redis adapters in infrastructure.

---

## Recommended Approach

1. Use Micronaut Redis integration and configuration first.
2. Keep key construction and serialization explicit and versionable.
3. Encapsulate Redis operations behind focused adapter classes.
4. Avoid leaking Redis-specific APIs into domain model/use-case contracts.

---

## Adapter Sketch

```java
public interface OrderCache {
    Optional<CachedOrderView> findById(OrderId id);
    void put(CachedOrderView view);
    void evict(OrderId id);
}

@Singleton
class RedisOrderCache implements OrderCache {
    // depends on Micronaut Redis client bean
}
```

---

## Operational Notes

- Define TTL per use case (do not rely on defaults).
- Namespace keys by bounded context (`ordering:order:{id}`).
- Validate behavior under cache misses and stale reads.
- Use Testcontainers Redis for integration tests.
