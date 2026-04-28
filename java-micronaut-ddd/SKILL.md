---
name: java-micronaut-ddd
description: Use when working on Java Micronaut applications, Micronaut HTTP APIs, jOOQ persistence, Micronaut Validation, Micronaut Security, Micronaut Messaging, DDD module boundaries, Testcontainers integration tests, MapStruct DTO mapping, transaction management, Redis adapters, or Lombok-free backend code.
---

# Java/Micronaut Development Standards

## Core Principles

### 1. Domain-Driven Design (DDD)

Structure code **around business capabilities**, not technical layers. See [docs/ddd-patterns.md](docs/ddd-patterns.md) for patterns and examples.

**Avoid: technical-layer slicing**

```text
com.acme.app.controller
com.acme.app.service
com.acme.app.repository
```

**Prefer: business modules (bounded contexts)**

```text
com.acme.app/
+-- billing/
|   +-- domain/         # Invoice, Payment, Money, BillingPolicy
|   +-- application/    # IssueInvoiceUseCase, RecordPaymentUseCase
|   +-- infrastructure/ # JooqInvoiceRepository, PaymentGatewayClient
|   `-- api/            # BillingController, request/response DTOs
+-- catalog/
|   +-- domain/
|   +-- application/
|   +-- infrastructure/
|   `-- api/
`-- identity/
    `-- ...
```

**Key rules:**

- Top-level packages are **business modules** (bounded contexts)
- Technical layers (`domain/application/infrastructure/api`) are **inside** each module
- A change like "adjust invoice rules" should touch **one module** (`billing/`), not hop across layers
- Cross-module calls go through **explicit APIs** (use cases, ports, domain events)

### 2. No Lombok

Write explicit Java code. Do not use Lombok annotations (`@Data`, `@Getter`, `@Builder`, etc.).

**Why:** Explicit code is debuggable, IDE-friendly, and avoids compile-time magic issues.

**Instead of Lombok:**

- Use Java records for immutable data carriers
- Generate getters/setters via IDE
- Write explicit builders when needed
- Use `Objects.equals()` and `Objects.hash()` for equals/hashCode

### 3. Conservative Dependency Management

Before adding a new library:

1. Check if functionality exists in current dependencies
2. Check if Micronaut modules already provide it
3. Check if Java standard library covers the use case

**Analyze classpath first:**

```bash
./mvnw dependency:tree
# or
./gradlew dependencies
```

### 4. jOOQ Persistence

Use jOOQ with Micronaut SQL for relational persistence. Do not introduce Micronaut Data, JPA, or Hibernate for persistence in this skill.

- **Schema:** Model persistence from migrations/DDL and generated jOOQ schema types
- **Queries:** Use explicit `DSLContext` queries, joins, projections, and locking
- **Repositories:** Keep domain repository interfaces as ports; implement them with jOOQ in infrastructure
- **Mapping:** Convert between jOOQ records and domain objects at the adapter boundary
- **Transactions:** Use `@Transactional` on application service methods; use explicit version columns for optimistic locking

See [docs/data-access.md](docs/data-access.md) for jOOQ repository and mapping patterns.
See [docs/transactions.md](docs/transactions.md) for transaction management rules.

### 5. Testcontainers for Integration Testing

Use Testcontainers or Micronaut Test Resources for tests requiring external dependencies (databases, message brokers, Redis, etc.).

See [docs/testing.md](docs/testing.md) for setup and patterns.

### 6. MapStruct for DTO Mapping

Use MapStruct for structural conversions between API DTOs, use-case commands, and responses. Enforce strict boundaries:

- **Controllers:** Accept/return DTOs only, never touch domain objects or jOOQ records directly
- **Services/Use cases:** Invoke mappers, resolve relationships by ID, call domain behavior
- **Mappers:** Pure structural conversion only, no repository access, no business logic
- **Domain:** Independent of DTOs, web layer, jOOQ records, and SQL details

See [docs/mapping.md](docs/mapping.md) for patterns and examples.

### 7. Micronaut Redis Access

Use Micronaut Redis module according to official docs: https://micronaut-projects.github.io/micronaut-redis/6.9.0/guide/

- Prefer Micronaut-provided Redis clients/configuration before introducing custom wrappers
- Keep Redis usage behind ports/adapters in the infrastructure layer
- Use serialization and key naming conventions explicitly
- For tests that depend on Redis, use Testcontainers instead of shared local instances

See [docs/redis-access.md](docs/redis-access.md) for usage guidance.

---

## Workflow

### Before Writing Code

1. **Analyze existing dependencies:**

   ```bash
   ./mvnw dependency:tree | grep -E "(micronaut-jooq|jooq|micronaut-data|hibernate|jakarta.persistence|micronaut-validation|micronaut-redis|lombok|mapstruct|test)"
   ```

2. **Verify jOOQ stack:** Ensure `micronaut-jooq`, jOOQ code generation, a `DataSource`, and migrations are configured.

3. **Check for forbidden persistence stack:** If Micronaut Data, JPA, or Hibernate are present, discuss removal or isolation before adding persistence code.

4. **Check for MapStruct:** If not present and DTOs are needed, add `mapstruct` + `mapstruct-processor`.

5. **Check for Lombok:** If `lombok` is in dependencies, discuss removal strategy with user before proceeding.

### When Implementing Features

1. **Start with domain model** - domain entities and value objects
2. **Define repository interfaces (ports)** - in domain/application layer
3. **Create DTOs** - request/response records per use case
4. **Create MapStruct mappers** - pure DTO/command/response conversion
5. **Implement application services/use cases** - orchestrate domain operations and transactions
6. **Implement infrastructure adapters** - jOOQ repositories, Redis adapters, external clients
7. **Add API layer last** - controllers are thin adapters, DTOs only

### When Writing Tests

1. **Unit tests** - domain logic, no Micronaut context
2. **Integration tests** - use `@MicronautTest` + Testcontainers or Micronaut Test Resources
3. **Focused tests** - test jOOQ repository/client behavior with minimal required context

---

## Quick Reference

| Scenario | Action |
|----------|--------|
| Need a DTO | Use Java record |
| Need domain entity | Write explicit domain class with invariants and behavior |
| Need builder | Write static inner Builder class |
| Need JSON mapping | Use Jackson annotations on records |
| Need validation | Use Jakarta Bean Validation (`@NotNull`, etc.) via Micronaut Validation |
| Need database | Use jOOQ through a Micronaut-managed `DSLContext` |
| Need caching | Check Micronaut Cache support before adding new lib |
| Need HTTP client | Prefer Micronaut HTTP client before adding external client libraries |
| Need DTO to command | Use MapStruct mapper; resolve relationships in service |
| Need domain to DTO | Use MapStruct mapper or explicit response assembler |
| Need jOOQ record to domain | Use an explicit persistence mapper in infrastructure |
| Need partial update | Use `@MappingTarget` with `IGNORE` null strategy |
| Controller needs domain object or jOOQ record | No - return DTO from service, never expose internals |
| Service modifies data | Add `@Transactional` on public use-case/service method |
| Read-only query | Use read-only transactional semantics for query use cases |
| Concurrent modifications | Use explicit `version` column predicates in jOOQ updates |
| External call in transaction | No - use outbox/event-driven patterns |
| Need Redis access | Use Micronaut Redis integration behind infrastructure port |

---

## Reference Files

- [DDD Patterns](docs/ddd-patterns.md) - Aggregates, entities, value objects, domain events
- [Data Access](docs/data-access.md) - jOOQ repositories, generated schema types, explicit SQL
- [Transactions](docs/transactions.md) - transaction boundaries, propagation, locking
- [Testing](docs/testing.md) - Testcontainers setup and integration test patterns
- [Mapping](docs/mapping.md) - MapStruct patterns, layer boundaries, DTO design
- [Redis Access](docs/redis-access.md) - Micronaut Redis integration and adapter patterns
- Micronaut guide - https://docs.micronaut.io/4.10.21/guide/
- Micronaut SQL/jOOQ guide - https://micronaut-projects.github.io/micronaut-sql/latest/guide/
- Micronaut Redis guide - https://micronaut-projects.github.io/micronaut-redis/6.9.0/guide/
