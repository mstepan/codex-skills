# Spring Data JPA Transaction Management

## Core Principle

**Define transaction boundaries at the service/use-case layer.** Repositories are called *inside* that transaction.

---

## Transaction Basics

### 1. Place `@Transactional` on Public Service Methods

```java
@Service
public class PlaceOrderUseCase {

    @Transactional  // ✓ Correct: on public service method
    public OrderResponse execute(CreateOrderRequest request) {
        // All repository calls happen inside this transaction
    }
}
```

### 2. Proxy Limitations

Spring applies transactions via proxies:

```java
@Service
public class OrderService {

    @Transactional
    public void methodA() { /* transactional */ }

    public void methodB() {
        methodA();  // ✗ Self-invocation bypasses proxy — NO transaction!
    }
}
```

**Fix:** Move transactional method to a separate bean, or call through the proxy.

### 3. Propagation Modes

| Mode | Behavior |
|------|----------|
| `REQUIRED` (default) | Join existing or start new |
| `REQUIRES_NEW` | Always start new (suspends current) |
| `MANDATORY` | Fail if no transaction exists |
| `NESTED` | Use savepoint (DB support required) |
| `NOT_SUPPORTED` | Run without transaction |
| `SUPPORTS` | Join if present, otherwise non-transactional |

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void auditLog(String action) {
    // Runs in separate transaction — commits even if caller rolls back
}
```

---

## Commit and Rollback

### Default Behavior

- **Rollback:** Unchecked exceptions (`RuntimeException`, `Error`)
- **Commit:** Checked exceptions (unless configured)

### Control Rollback Explicitly

```java
// Rollback on checked exception
@Transactional(rollbackFor = PaymentFailedException.class)
public void processPayment(PaymentRequest request) throws PaymentFailedException {
    // ...
}

// Don't rollback on specific runtime exception
@Transactional(noRollbackFor = ItemOutOfStockException.class)
public void placeOrder(OrderRequest request) {
    // ...
}
```

### Don't Swallow Exceptions

```java
@Transactional
public void process() {
    try {
        riskyOperation();
    } catch (Exception e) {
        log.error("Failed", e);
        // ✗ Swallowed — transaction will COMMIT partial work!
    }
}

// ✓ Correct: rethrow or mark for rollback
@Transactional
public void process() {
    try {
        riskyOperation();
    } catch (Exception e) {
        log.error("Failed", e);
        throw e;  // Triggers rollback
    }
}
```

---

## JPA Persistence Context

### Managed Entities (Inside Transaction)

```java
@Transactional
public void changeEmail(Long userId, String email) {
    User user = userRepository.findById(userId)
        .orElseThrow(NotFoundException::new);

    user.setEmail(email);  // Dirty checking — no save() needed!
    // Commit flushes changes automatically
}
```

### Detached Entities (Outside Transaction)

After transaction ends, entities are **detached**:
- Changes are NOT tracked
- Lazy associations throw `LazyInitializationException`

---

## Flush vs Commit

| Operation | Effect |
|-----------|--------|
| **Flush** | Sends SQL to DB, but transaction continues |
| **Commit** | Ends transaction, makes changes permanent |

Hibernate auto-flushes:
- Before commit
- Before queries (flush mode `AUTO`)
- On explicit `flush()` call

**Implication:** Constraint violations may appear during auto-flush, not at commit.

### `save()` vs `saveAndFlush()`

```java
// save() — SQL may be delayed until flush/commit
orderRepository.save(order);

// saveAndFlush() — SQL executes immediately (reduces batching)
orderRepository.saveAndFlush(order);  // Use sparingly
```

---

## Lazy Loading

### Problem

```java
@Transactional
public Order getOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
}

// Later, outside transaction:
order.getLines().size();  // ✗ LazyInitializationException!
```

### Solutions

**1. Fetch inside transaction:**
```java
@Transactional(readOnly = true)
public OrderResponse getOrder(Long id) {
    Order order = orderRepository.findByIdWithLines(id)  // JOIN FETCH
        .orElseThrow();
    return mapper.toResponse(order);  // Map while session open
}
```

**2. Use `@EntityGraph`:**
```java
@EntityGraph(attributePaths = {"lines", "customer"})
Optional<Order> findById(Long id);
```

**3. Use DTO projections:**
```java
@Query("SELECT new com.acme.ordering.api.response.OrderSummary(o.id, o.status, o.total) FROM Order o WHERE o.id = :id")
Optional<OrderSummary> findSummaryById(Long id);
```

**Avoid:** Relying on "Open Session in View" as a fix.

---

## Read-Only Transactions

```java
@Transactional(readOnly = true)
public List<OrderResponse> findByCustomer(CustomerId customerId) {
    return orderRepository.findByCustomerId(customerId)
        .stream()
        .map(mapper::toResponse)
        .toList();
}
```

**Benefits:**
- Hibernate skips dirty checking
- Some DBs route to read replicas
- Performance optimization hint

**Note:** Does not *guarantee* no writes — it's a hint.

---

## Thread Boundaries

Transactions are **thread-bound**:

```java
@Transactional
public void process(List<Item> items) {
    items.parallelStream()
        .forEach(this::processItem);  // ✗ Each thread needs own transaction!
}

@Async
public void asyncProcess(Long id) {
    // ✗ Runs in different thread — no transaction from caller!
}
```

**Fix:** Each thread/async method needs its own `@Transactional`.

---

## Keep Transactions Short

While transaction is open, you hold DB resources/locks.

**Avoid inside transactions:**
- HTTP calls to external services
- Long computations
- File I/O
- User interaction / waiting

```java
// ✗ Bad: HTTP call inside transaction
@Transactional
public void processOrder(Order order) {
    orderRepository.save(order);
    paymentGateway.charge(order.getTotal());  // External HTTP call!
}

// ✓ Better: transactional outbox pattern
@Transactional
public void processOrder(Order order) {
    orderRepository.save(order);
    outboxRepository.save(new PaymentOutboxEvent(order));  // Local write only
}
// Separate process handles outbox → payment gateway
```

---

## Concurrency Control

### Optimistic Locking (Preferred)

```java
@Entity
public class Order {
    @Id
    private Long id;

    @Version
    private Long version;  // Automatic optimistic locking
}
```

Handle conflicts:
```java
try {
    orderRepository.save(order);
} catch (OptimisticLockingFailureException e) {
    throw new ConcurrentModificationException("Order was modified by another user");
}
```

### Pessimistic Locking (Use Sparingly)

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT o FROM Order o WHERE o.id = :id")
Optional<Order> findByIdForUpdate(Long id);
```

**Caution:** Risk of deadlocks and reduced throughput.

---

## Multiple Datasources

With multiple transaction managers, be explicit:

```java
@Transactional(transactionManager = "orderTransactionManager")
public void processOrder(Order order) { }

@Transactional(transactionManager = "inventoryTransactionManager")
public void updateInventory(InventoryUpdate update) { }
```

**Note:** True atomic transactions across multiple DBs require XA/JTA or saga/compensation patterns.

---

## Quick Reference

| Scenario | Action |
|----------|--------|
| Service method modifies data | Add `@Transactional` |
| Read-only query | Add `@Transactional(readOnly = true)` |
| Need rollback on checked exception | Add `rollbackFor = YourException.class` |
| Calling transactional method from same class | Move to separate bean |
| Need entity with associations | Fetch eagerly or use `@EntityGraph` |
| Concurrent modifications | Use `@Version` for optimistic locking |
| External service call needed | Use transactional outbox pattern |
| Multiple databases | Specify `transactionManager` explicitly |
