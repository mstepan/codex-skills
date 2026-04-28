# Transaction Management with Micronaut SQL and jOOQ

## Core Principle

Define transaction boundaries in **application services/use cases**, not in controllers or jOOQ repository adapters.

---

## Basic Rule

```java
@Singleton
public class PlaceOrderUseCase {

    @Transactional
    public OrderResponse execute(CreateOrderCommand command) {
        // jOOQ repository calls happen inside one Micronaut-managed transaction.
    }
}
```

---

## Guidance

- Use `@Transactional` for write use cases.
- Prefer read-only transaction semantics for query-focused use cases where supported.
- Keep transactions short; avoid external HTTP/network calls inside open DB transactions.
- Let repositories use the injected Micronaut-managed `DSLContext`; do not create ad hoc contexts.
- Use optimistic locking with explicit `version` columns in jOOQ `where` clauses.
- If using jOOQ's programmatic transaction API, use the transaction-scoped `DSLContext` from the callback.

```java
int updated = dsl.update(ORDERS)
    .set(ORDERS.STATUS, nextStatus.name())
    .set(ORDERS.VERSION, ORDERS.VERSION.plus(1L))
    .where(ORDERS.ID.eq(orderId.value()))
    .and(ORDERS.VERSION.eq(expectedVersion))
    .execute();

if (updated == 0) {
    throw new ConcurrentOrderModificationException(orderId);
}
```

---

## Failure Handling

- Let exceptions propagate to trigger rollback.
- Do not swallow exceptions inside transactional methods.
- For external side effects, prefer outbox/event-driven delivery after commit.

---

## Quick Reference

| Scenario | Action |
|----------|--------|
| Write use case | Add `@Transactional` |
| Query use case | Prefer read-only transaction semantics |
| Concurrent updates | Add `version` column predicate and handle zero updated rows |
| External integration | Use outbox/eventing instead of in-transaction HTTP |
