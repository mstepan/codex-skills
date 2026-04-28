# DTO and Domain Mapping with MapStruct (Micronaut)

## Core Principle

DTO mapping lives in dedicated MapStruct mappers, invoked by application services/use cases. Persistence mapping between jOOQ records and domain objects stays in infrastructure adapters and is usually explicit Java.

---

## Layer Responsibilities

| Layer | Does | Does NOT |
|-------|------|----------|
| Controller | HTTP routing, validation trigger, response codes | Touch domain objects or jOOQ records directly |
| Application service/use case | Transaction boundary, orchestration, relationship resolution | Expose domain internals or jOOQ records to API |
| Domain | Invariants and behavior | Depend on DTO, SQL, or jOOQ types |
| DTO Mapper | Pure structural request/command/response mapping | Access repositories or implement business logic |
| Persistence Mapper | Convert jOOQ records to/from domain objects | Leak generated jOOQ types outside infrastructure |

---

## Rules

1. Request DTO -> command/use-case input; avoid direct DTO -> aggregate mutation.
2. Never map client-controlled values into server-owned fields (`id`, `version`, audit fields).
3. Use `@MappingTarget` + null-ignore strategy for partial DTO updates when useful.
4. Resolve references by ID in services, not in mappers.
5. Map jOOQ records to domain objects in infrastructure; do not expose jOOQ records to controllers or use cases.

---

## Mapper Example

```java
@Mapper(
    componentModel = MappingConstants.ComponentModel.JAKARTA,
    nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE
)
public interface OrderMapper {

    PlaceOrderCommand toCommand(CreateOrderRequest request);

    UpdateOrderCommand toCommand(UpdateOrderRequest request);

    OrderResponse toResponse(Order order);
}
```

For Micronaut 4, prefer the Jakarta component model so generated mappers use `jakarta.inject`.

---

## Checklist

- [ ] Controllers exchange DTOs only
- [ ] Services own orchestration and relationship resolution
- [ ] DTO mappers remain pure structural converters
- [ ] Domain remains independent from API DTOs, SQL, and jOOQ
- [ ] jOOQ generated types remain inside infrastructure
