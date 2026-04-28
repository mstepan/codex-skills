---
name: java-spring-ddd
description: Use when working on Java Spring Boot or Spring Framework applications, Spring MVC, Spring WebFlux, Spring Data JPA, DDD module boundaries, Testcontainers integration tests, MapStruct DTO mapping, transaction management, caching, HTTP clients, or Lombok-free backend code.
---

# Java/Spring Development Standards

## Core Principles

### 1. Domain-Driven Design (DDD)
Structure code **around business capabilities**, not technical layers. See [docs/ddd-patterns.md](docs/ddd-patterns.md) for patterns and examples.

**Avoid: technical-layer slicing**
```
com.acme.app.controller
com.acme.app.service
com.acme.app.repository
```

**Prefer: business modules (bounded contexts)**
```
com.acme.app/
├── billing/
│   ├── domain/        # Invoice, Payment, Money, BillingPolicy
│   ├── application/   # IssueInvoiceUseCase, RecordPaymentUseCase
│   ├── infrastructure/# JpaInvoiceRepository, PaymentGatewayClient
│   └── api/           # BillingController, request/response DTOs
├── catalog/
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── api/
└── identity/
    └── ...
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
2. Check if Spring Boot starters already provide it
3. Check if Java standard library covers the use case

**Analyze classpath first:**
```bash
./mvnw dependency:tree
# or
./gradlew dependencies
```

### 4. Spring Data JPA
Use Spring Data JPA with Hibernate for persistence. Key practices:

- **Entities:** Explicit getters, `@Version` for optimistic locking, protected no-arg constructor
- **Fetching:** Avoid N+1 with `JOIN FETCH`, `@EntityGraph`, or DTO projections
- **Transactions:** `@Transactional` on service methods, `readOnly = true` for queries

See [docs/data-access.md](docs/data-access.md) for entity mapping and repository patterns.
See [docs/transactions.md](docs/transactions.md) for transaction management rules.

### 5. Testcontainers for Integration Testing
Use Testcontainers for any test requiring external dependencies (databases, message brokers, etc.).

See [docs/testing.md](docs/testing.md) for setup and patterns.

### 6. MapStruct for Entity/DTO Mapping
Use MapStruct for all structural conversions between layers. Enforce strict boundaries:

- **Controllers**: Accept/return DTOs only, never touch entities
- **Services**: Invoke mappers, resolve relationships by ID, call domain behavior
- **Mappers**: Pure structural conversion only, no repository access, no business logic
- **Domain**: Independent of DTOs and web layer

See [docs/mapping.md](docs/mapping.md) for patterns and examples.

---

## Workflow

### Before Writing Code

1. **Analyze existing dependencies:**
   ```bash
   ./mvnw dependency:tree | grep -E "(spring-data|lombok|mapstruct|test)"
   ```

2. **Verify Spring Data JPA:** Ensure `spring-boot-starter-data-jpa` is present.

3. **Check for MapStruct:** If not present and DTOs needed, add `mapstruct` + `mapstruct-processor`.

4. **Check for Lombok:** If `lombok` in dependencies, discuss removal strategy with user before proceeding.

### When Implementing Features

1. **Start with domain model** — entities and value objects
2. **Define repository interfaces** — in domain layer
3. **Create DTOs** — request/response records per use case
4. **Create MapStruct mappers** — pure structural conversion
5. **Implement application services** — orchestrate domain operations, invoke mappers
6. **Add API layer last** — controllers are thin adapters, DTOs only

### When Writing Tests

1. **Unit tests** — domain logic, no Spring context
2. **Integration tests** — use `@SpringBootTest` + Testcontainers
3. **Slice tests** — `@DataJpaTest` or `@WebMvcTest` for focused testing

---

## Quick Reference

| Scenario | Action |
|----------|--------|
| Need a DTO | Use Java record |
| Need entity | Write class with explicit getters, equals/hashCode |
| Need builder | Write static inner Builder class |
| Need JSON mapping | Use Jackson annotations on records |
| Need validation | Use Jakarta Bean Validation (`@NotNull`, etc.) |
| Need database | Use Spring Data JPA repository |
| Need caching | Check if `spring-boot-starter-cache` already present |
| Need HTTP client | Check for WebClient or RestClient before adding new lib |
| Need DTO→Entity | Use MapStruct mapper, resolve relationships in service |
| Need Entity→DTO | Use MapStruct mapper, ensure graph is fetched first |
| Need partial update | Use `@MappingTarget` with `IGNORE` null strategy |
| Controller needs entity | NO — return DTO from service, never expose entities |
| Service modifies data | Add `@Transactional` on public method |
| Read-only query | Add `@Transactional(readOnly = true)` |
| Concurrent modifications | Use `@Version` for optimistic locking |
| External call in transaction | NO — use transactional outbox pattern |

---

## Reference Files

- [DDD Patterns](docs/ddd-patterns.md) — Aggregates, entities, value objects, domain events
- [Data Access](docs/data-access.md) — Spring Data JPA entities, repositories, fetching strategies
- [Transactions](docs/transactions.md) — JPA transaction management, propagation, locking
- [Testing](docs/testing.md) — Testcontainers setup and integration test patterns
- [Mapping](docs/mapping.md) — MapStruct patterns, layer boundaries, DTO design
