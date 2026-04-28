# Testing with Micronaut Test + Testcontainers

## Core Principle

Use Testcontainers or Micronaut Test Resources for integration tests involving infrastructure (databases, Redis, brokers). Keep domain tests framework-free.

---

## Test Types

1. **Unit tests** - pure domain logic, no Micronaut context
2. **Integration tests** - `@MicronautTest` with Testcontainers or Micronaut Test Resources
3. **Adapter tests** - verify jOOQ repositories and external clients against real infrastructure

---

## Integration Test Example

```java
@MicronautTest
class OrderUseCaseIntegrationTest {

    @Inject
    PlaceOrderUseCase placeOrderUseCase;

    @Test
    void shouldPlaceOrder() {
        var result = placeOrderUseCase.execute(validCommand());
        assertThat(result.status()).isEqualTo("SUBMITTED");
    }
}
```

Use a container lifecycle strategy appropriate for your build: per class, shared fixture, or Micronaut Test Resources.

---

## jOOQ Adapter Testing

- Run migrations before inserting test data so generated schema, migrations, and runtime database stay aligned.
- Verify observable behavior, not mocks of `DSLContext`.
- Cover optimistic-lock conflicts, joins/projections, pagination ordering, and vendor-specific SQL paths.

---

## Redis Testing

- Prefer Redis Testcontainers or Micronaut Test Resources over shared local Redis.
- Isolate keys by test namespace.
- Clear state between tests.

---

## Quick Checklist

- [ ] Domain tests do not boot Micronaut
- [ ] Integration tests use real containerized dependencies
- [ ] jOOQ adapters are tested against the target database dialect
- [ ] No hidden dependency on developer-local services
