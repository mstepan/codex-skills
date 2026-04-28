# Data Access with jOOQ

## Core Principle

Use **jOOQ and generated schema types** for relational persistence. Keep jOOQ generated classes, `DSLContext`, SQL details, and database records in infrastructure adapters; keep domain contracts and business invariants in domain/application layers.

jOOQ is SQL-first, not ORM-first:

- Model persistence from the database schema and migrations, not from annotated entities.
- Generate jOOQ tables and records from the schema so column and type changes fail at compile time.
- Write explicit SQL-shaped queries with `DSLContext`; do not hide important joins, filters, ordering, or locking behind implicit ORM behavior.
- Map records to domain objects or query DTOs at the adapter boundary.

---

## Setup Expectations

- Add `io.micronaut.sql:micronaut-jooq` and configure a `DataSource`; Micronaut will provide a `DSLContext` bean.
- Generate jOOQ sources from the same schema managed by Flyway/Liquibase or checked-in DDL.
- Put generated classes under an infrastructure package such as `billing.infrastructure.jooq.generated`.
- Do not expose generated jOOQ `Record`, `Table`, or `Field` types from domain, application, or API packages.
- Decide deliberately whether generated sources are committed or regenerated during every build; keep the decision consistent in CI.

---

## Repository Adapter Pattern

```java
// Domain port
public interface OrderRepository {
    Optional<Order> findById(OrderId id);
    void save(Order order);
}

// Infrastructure adapter
@Singleton
final class JooqOrderRepository implements OrderRepository {
    private final DSLContext dsl;
    private final OrderPersistenceMapper mapper;

    JooqOrderRepository(DSLContext dsl, OrderPersistenceMapper mapper) {
        this.dsl = dsl;
        this.mapper = mapper;
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return dsl.selectFrom(ORDERS)
            .where(ORDERS.ID.eq(id.value()))
            .fetchOptional(mapper::toDomain);
    }

    @Override
    public void save(Order order) {
        OrderRecord record = mapper.toRecord(order);

        if (order.isNew()) {
            dsl.insertInto(ORDERS)
                .set(record)
                .execute();
            return;
        }

        int updated = dsl.update(ORDERS)
            .set(ORDERS.CUSTOMER_ID, record.getCustomerId())
            .set(ORDERS.STATUS, record.getStatus())
            .set(ORDERS.TOTAL_AMOUNT, record.getTotalAmount())
            .set(ORDERS.VERSION, ORDERS.VERSION.plus(1L))
            .where(ORDERS.ID.eq(record.getId()))
            .and(ORDERS.VERSION.eq(record.getVersion()))
            .execute();

        if (updated == 0) {
            throw new ConcurrentOrderModificationException(order.id());
        }
    }
}
```

Rules:

- Repository interfaces are domain ports; jOOQ implementations live in infrastructure.
- Inject `DSLContext`; do not create ad hoc contexts in repositories.
- Prefer explicit `insertInto`, `update`, `deleteFrom`, and `select` statements over generated DAOs for business persistence.
- Use generated table constants with static imports inside infrastructure only.
- Return domain objects or query DTOs, never jOOQ records.

---

## Mapping

Use small, explicit mappers between jOOQ records and domain objects. MapStruct can help for structural mappings, but persistence decisions stay in the adapter.

```java
@Singleton
final class OrderPersistenceMapper {

    Order toDomain(OrderRecord record) {
        return Order.rehydrate(
            new OrderId(record.getId()),
            new CustomerId(record.getCustomerId()),
            OrderStatus.valueOf(record.getStatus()),
            Money.of(record.getTotalAmount(), record.getCurrency()),
            record.getVersion()
        );
    }

    OrderRecord toRecord(Order order) {
        OrderRecord record = new OrderRecord();
        record.setId(order.id().value());
        record.setCustomerId(order.customerId().value());
        record.setStatus(order.status().name());
        record.setTotalAmount(order.total().amount());
        record.setCurrency(order.total().currency());
        record.setVersion(order.version());
        return record;
    }
}
```

Guidance:

- Use domain factory methods such as `rehydrate` when loading persisted aggregate state.
- Keep conversion of enums, value objects, money, JSON, and timestamps explicit.
- Prefer jOOQ converters or forced types for repeated low-level column conversions.
- Do not put domain behavior on generated records.

---

## Query Models and Projections

For read use cases, project directly into API/application query DTOs instead of loading aggregates that are not needed.

```java
public record OrderSummary(
    Long id,
    String customerId,
    String status,
    BigDecimal totalAmount,
    Instant createdAt
) {}

@Singleton
final class JooqOrderQueries implements OrderQueries {
    private final DSLContext dsl;

    JooqOrderQueries(DSLContext dsl) {
        this.dsl = dsl;
    }

    @Override
    public List<OrderSummary> findRecentForCustomer(CustomerId customerId, int limit) {
        return dsl.select(
                ORDERS.ID,
                ORDERS.CUSTOMER_ID,
                ORDERS.STATUS,
                ORDERS.TOTAL_AMOUNT,
                ORDERS.CREATED_AT
            )
            .from(ORDERS)
            .where(ORDERS.CUSTOMER_ID.eq(customerId.value()))
            .orderBy(ORDERS.CREATED_AT.desc(), ORDERS.ID.desc())
            .limit(limit)
            .fetch(record -> new OrderSummary(
                record.get(ORDERS.ID),
                record.get(ORDERS.CUSTOMER_ID),
                record.get(ORDERS.STATUS),
                record.get(ORDERS.TOTAL_AMOUNT),
                record.get(ORDERS.CREATED_AT)
            ));
    }
}
```

Guidance:

- Select only columns needed by the use case.
- Use joins, `exists`, aggregation, window functions, and CTEs directly when they express the query clearly.
- Keep pagination deterministic with stable `orderBy` clauses.
- Use separate query ports for read models instead of stretching aggregate repositories into reporting APIs.

---

## Transactions and Locking

- Put `@Transactional` on application service/use-case methods, not on every repository method.
- Let all repository calls in the use case share the Micronaut-managed transaction and injected `DSLContext`.
- For jOOQ's programmatic transaction API, use the transaction-scoped configuration/DSL from the callback, not an outer `DSLContext`.
- Use optimistic locking explicitly with a `version` column in `where` clauses, and translate zero updated rows into a domain concurrency error.
- Use `forUpdate()` only for short critical sections where pessimistic locking is required.

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

## Plain SQL and Safety

- Prefer the type-safe DSL over plain SQL strings.
- When plain SQL is necessary, use bind values or jOOQ `QueryPart` placeholders; never concatenate user input.
- Keep vendor-specific SQL isolated in infrastructure adapters.
- Treat rendered SQL as an implementation detail unless a test is explicitly verifying SQL generation.

```java
dsl.selectFrom(ORDERS)
    .where(condition("jsonb_exists({0}, {1})", ORDERS.METADATA, val("priority")))
    .fetch(mapper::toDomain);
```

---

## Testing

- Test jOOQ adapters against a real database with Testcontainers.
- Run migrations before generating test data so generated schema, migrations, and runtime database stay aligned.
- Verify important SQL behavior with observable results, not with mocks of `DSLContext`.
- Cover optimistic-lock conflicts, joins/projections, pagination ordering, and vendor-specific SQL paths.

---

## Quick Reference

| Need | jOOQ approach |
|------|---------------|
| Database access | Inject Micronaut-managed `DSLContext` |
| Schema types | Generate jOOQ code from migrations or DDL |
| Domain repository | Domain port plus infrastructure jOOQ adapter |
| Save aggregate | Explicit `insertInto` / `update` / `deleteFrom` |
| Read model | Dedicated query port with explicit projection |
| Mapping | Record-to-domain mapper at adapter boundary |
| Pagination | `limit` / `offset` or keyset pagination with stable `orderBy` |
| Optimistic lock | `version` column in `where`, handle zero updated rows |
| Pessimistic lock | `forUpdate()` in a short transaction |
| Plain SQL | Bind values or `QueryPart` placeholders only |
| Generated DAOs | Avoid for business persistence; prefer explicit SQL |
| Domain layer needs jOOQ | No; keep jOOQ in infrastructure |
