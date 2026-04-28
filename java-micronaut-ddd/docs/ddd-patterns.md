# DDD Patterns for Micronaut

## Core Principle

**Business names at the top, technical names near the leaves.**

Structure the codebase around business capabilities and domain concepts, not around technical layers like `controller/service/repository`.

---

## Package Structure: Business Modules (Bounded Contexts)

### Avoid: Technical-Layer Slicing

```text
com.acme.app/
+-- controller/
+-- service/
+-- repository/
`-- dto/
```

### Prefer: Business-First Modules

```text
com.acme.app/
+-- ordering/
|   +-- domain/
|   |   +-- model/
|   |   |   +-- Order.java
|   |   |   +-- OrderLine.java
|   |   |   +-- OrderId.java
|   |   |   `-- Money.java
|   |   +-- policy/
|   |   +-- event/
|   |   `-- OrderRepository.java      # port
|   +-- application/
|   |   +-- PlaceOrderUseCase.java
|   |   +-- command/
|   |   `-- mapper/
|   +-- infrastructure/
|   |   +-- jooq/
|   |   |   +-- JooqOrderRepository.java
|   |   |   `-- generated/
|   |   +-- messaging/
|   |   `-- redis/
|   `-- api/
|       +-- OrderController.java
|       +-- request/
|       `-- response/
+-- billing/
`-- catalog/
```

**Benefits:**

- Business changes stay localized in one module
- Domain concepts are cohesive and discoverable
- Clear ownership boundaries between modules

---

## Rules of Thumb

| Principle | Guideline |
|-----------|-----------|
| Package naming | Business names at top (`ordering/`) |
| Change locality | One business change should mostly touch one module |
| Domain purity | Domain should not depend on Micronaut framework types |
| Persistence isolation | jOOQ generated types stay in infrastructure |
| Dependency direction | `api` -> `application` -> `domain`; `infrastructure` implements ports |
| Cross-module calls | Go through explicit APIs (ports/use-cases/events) |

---

## Cross-Module Communication

Use explicit contracts instead of internal class access:

1. **Domain events** published after successful transaction
2. **Ports/interfaces** in domain/application, adapters in infrastructure
3. **Shared kernel** only for stable cross-cutting value objects (`Money`, IDs)

---

## Aggregate Root Example (No Lombok)

```java
public class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderLine> lines;
    private OrderStatus status;
    private Money totalAmount;

    private Order(OrderId id, CustomerId customerId, List<OrderLine> lines) {
        this.id = Objects.requireNonNull(id);
        this.customerId = Objects.requireNonNull(customerId);
        this.lines = new ArrayList<>(lines);
        this.status = OrderStatus.DRAFT;
        this.totalAmount = calculateTotal();
    }

    public static Order create(CustomerId customerId, List<OrderLine> lines) {
        if (lines == null || lines.isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one line");
        }
        return new Order(OrderId.generate(), customerId, lines);
    }

    public void submit() {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Only draft orders can be submitted");
        }
        this.status = OrderStatus.SUBMITTED;
    }

    private Money calculateTotal() {
        return lines.stream().map(OrderLine::subtotal).reduce(Money.ZERO, Money::add);
    }

    public OrderId getId() { return id; }
    public CustomerId getCustomerId() { return customerId; }
    public List<OrderLine> getLines() { return List.copyOf(lines); }
    public OrderStatus getStatus() { return status; }
    public Money getTotalAmount() { return totalAmount; }
}
```

## Repository Port (Domain)

```java
public interface OrderRepository {
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomerId(CustomerId customerId);
    void save(Order order);
}
```
