# Data Access with Spring Data JPA

## Entity Mapping (No Lombok)

```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "customer_id", nullable = false)
    private String customerId;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status;

    @Column(name = "total_amount", nullable = false)
    private BigDecimal totalAmount;

    @Version
    private Long version;  // Optimistic locking

    @Embedded
    private Address shippingAddress;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Set<OrderLine> lines = new HashSet<>();

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    protected Order() {}  // JPA requires no-arg constructor

    public Order(String customerId, Set<OrderLine> lines) {
        this.customerId = customerId;
        this.lines = new HashSet<>(lines);
        this.status = OrderStatus.DRAFT;
        this.createdAt = Instant.now();
        recalculateTotal();
    }

    // Domain behavior
    public void addLine(OrderLine line) {
        this.lines.add(line);
        recalculateTotal();
    }

    public void submit() {
        if (this.status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Only draft orders can be submitted");
        }
        this.status = OrderStatus.SUBMITTED;
    }

    private void recalculateTotal() {
        this.totalAmount = lines.stream()
            .map(l -> l.getUnitPrice().multiply(BigDecimal.valueOf(l.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    // Explicit getters
    public Long getId() { return id; }
    public String getCustomerId() { return customerId; }
    public OrderStatus getStatus() { return status; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public Set<OrderLine> getLines() { return Set.copyOf(lines); }
    public Instant getCreatedAt() { return createdAt; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order order)) return false;
        return id != null && Objects.equals(id, order.id);
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();
    }
}
```

---

## Embedded Value Objects

```java
@Embeddable
public class Address {

    @Column(name = "shipping_street")
    private String street;

    @Column(name = "shipping_city")
    private String city;

    @Column(name = "shipping_postal_code")
    private String postalCode;

    @Column(name = "shipping_country")
    private String country;

    protected Address() {}  // JPA

    public Address(String street, String city, String postalCode, String country) {
        this.street = street;
        this.city = city;
        this.postalCode = postalCode;
        this.country = country;
    }

    // Getters only (immutable)
    public String getStreet() { return street; }
    public String getCity() { return city; }
    public String getPostalCode() { return postalCode; }
    public String getCountry() { return country; }
}
```

---

## Repository

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    List<Order> findByCustomerId(String customerId);

    List<Order> findByStatus(OrderStatus status);

    // Avoid N+1 with JOIN FETCH
    @Query("SELECT o FROM Order o JOIN FETCH o.lines WHERE o.id = :id")
    Optional<Order> findByIdWithLines(@Param("id") Long id);

    // EntityGraph alternative
    @EntityGraph(attributePaths = {"lines", "shippingAddress"})
    Optional<Order> findWithDetailsById(Long id);

    // Custom query
    @Query("SELECT o FROM Order o WHERE o.createdAt > :since ORDER BY o.createdAt DESC")
    List<Order> findRecentOrders(@Param("since") Instant since);

    // Modifying query
    @Modifying
    @Query("UPDATE Order o SET o.status = :status WHERE o.id = :id")
    void updateStatus(@Param("id") Long id, @Param("status") OrderStatus status);

    // DTO projection
    @Query("SELECT new com.acme.ordering.api.response.OrderSummary(o.id, o.status, o.totalAmount) " +
           "FROM Order o WHERE o.customerId = :customerId")
    List<OrderSummary> findSummariesByCustomerId(@Param("customerId") String customerId);
}
```

---

## Fetching Strategies

### Problem: N+1 Queries

```java
// ✗ N+1: loads orders, then 1 query per order for lines
List<Order> orders = orderRepository.findByCustomerId(customerId);
orders.forEach(o -> o.getLines().size());  // Triggers N queries!
```

### Solution 1: JOIN FETCH

```java
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.lines WHERE o.customerId = :customerId")
List<Order> findByCustomerIdWithLines(@Param("customerId") String customerId);
```

### Solution 2: @EntityGraph

```java
@EntityGraph(attributePaths = {"lines"})
List<Order> findByCustomerId(String customerId);
```

### Solution 3: DTO Projection (Best for Read-Only)

```java
// Interface projection
public interface OrderSummary {
    Long getId();
    OrderStatus getStatus();
    BigDecimal getTotalAmount();
}

List<OrderSummary> findSummaryByCustomerId(String customerId);
```

---

## Optimistic Locking

```java
@Entity
public class Order {

    @Version
    private Long version;
}
```

Handle conflicts in service:

```java
@Transactional
public OrderResponse updateOrder(Long id, UpdateOrderRequest request) {
    try {
        Order order = orderRepository.findById(id).orElseThrow(NotFoundException::new);
        mapper.apply(request, order);
        return mapper.toResponse(order);
    } catch (OptimisticLockingFailureException e) {
        throw new ConcurrentModificationException("Order was modified by another user");
    }
}
```

---

## Pessimistic Locking (Use Sparingly)

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT o FROM Order o WHERE o.id = :id")
    Optional<Order> findByIdForUpdate(@Param("id") Long id);
}
```

**Caution:** Risk of deadlocks and reduced throughput.

---

## Auditing

```java
@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass
public abstract class AuditableEntity {

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private Instant updatedAt;

    @CreatedBy
    @Column(name = "created_by", updatable = false)
    private String createdBy;

    @LastModifiedBy
    @Column(name = "updated_by")
    private String updatedBy;

    // Getters
}

// Enable in configuration
@Configuration
@EnableJpaAuditing
public class JpaConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .map(Authentication::getName);
    }
}
```

---

## Schema Migration (Flyway)

```sql
-- V1__create_orders.sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id VARCHAR(36) NOT NULL,
    status VARCHAR(20) NOT NULL,
    total_amount DECIMAL(19,2) NOT NULL,
    version BIGINT NOT NULL DEFAULT 0,
    shipping_street VARCHAR(255),
    shipping_city VARCHAR(100),
    shipping_postal_code VARCHAR(20),
    shipping_country VARCHAR(2),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE
);

CREATE TABLE order_lines (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id VARCHAR(36) NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(19,2) NOT NULL
);

CREATE INDEX idx_orders_customer_id ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
```

---

## Quick Reference

| Need | Solution |
|------|----------|
| Avoid N+1 | `JOIN FETCH`, `@EntityGraph`, or DTO projection |
| Read-only query | `@Transactional(readOnly = true)` + projection |
| Concurrent updates | `@Version` for optimistic locking |
| Audit fields | `@EnableJpaAuditing` + `@CreatedDate`, `@LastModifiedDate` |
| Custom query | `@Query` with JPQL or native SQL |
| Bulk update | `@Modifying` + `@Query` |
| Lock row | `@Lock(PESSIMISTIC_WRITE)` — use sparingly |
