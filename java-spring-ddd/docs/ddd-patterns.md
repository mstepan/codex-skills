# DDD Patterns for Spring

## Core Principle

**Business names at the top, technical names near the leaves.**

Structure the codebase around business capabilities and domain concepts, not around technical layers like `controller/service/repository`.

---

## Package Structure: Business Modules (Bounded Contexts)

### ✗ Avoid: Technical-Layer Slicing

```
com.acme.app/
├── controller/
│   ├── OrderController.java
│   ├── CustomerController.java
│   └── InvoiceController.java
├── service/
│   ├── OrderService.java
│   ├── CustomerService.java
│   └── InvoiceService.java
├── repository/
│   ├── OrderRepository.java
│   ├── CustomerRepository.java
│   └── InvoiceRepository.java
└── dto/
    └── ...
```

**Problems:** A change to "invoice cancellation rules" touches files across multiple packages. Business concepts are scattered. High coupling between unrelated domains.

### ✓ Prefer: Business-First Modules

```
com.acme.app/
├── ordering/                          # Bounded context: Order Management
│   ├── domain/
│   │   ├── model/
│   │   │   ├── Order.java            # Aggregate root
│   │   │   ├── OrderLine.java        # Entity within aggregate
│   │   │   ├── OrderId.java          # Value object
│   │   │   ├── OrderStatus.java      # Enum/sealed class
│   │   │   └── Money.java            # Value object
│   │   ├── policy/
│   │   │   └── PricingPolicy.java    # Domain service
│   │   ├── event/
│   │   │   └── OrderPlacedEvent.java
│   │   └── OrderRepository.java      # Repository interface (port)
│   ├── application/
│   │   ├── PlaceOrderUseCase.java
│   │   ├── CancelOrderUseCase.java
│   │   ├── command/
│   │   │   └── PlaceOrderCommand.java
│   │   └── mapper/
│   │       └── OrderMapper.java      # MapStruct mapper
│   ├── infrastructure/
│   │   ├── persistence/
│   │   │   └── JpaOrderRepository.java
│   │   └── messaging/
│   │       └── OrderEventPublisher.java
│   └── api/
│       ├── OrderController.java
│       ├── request/
│       │   ├── CreateOrderRequest.java
│       │   └── UpdateOrderRequest.java
│       └── response/
│           └── OrderResponse.java
│
├── billing/                           # Bounded context: Billing
│   ├── domain/
│   │   ├── model/
│   │   │   ├── Invoice.java
│   │   │   ├── Payment.java
│   │   │   └── BillingPolicy.java
│   │   └── InvoiceRepository.java
│   ├── application/
│   │   ├── IssueInvoiceUseCase.java
│   │   └── RecordPaymentUseCase.java
│   ├── infrastructure/
│   │   ├── JpaInvoiceRepository.java
│   │   └── PaymentGatewayClient.java
│   └── api/
│       └── BillingController.java
│
├── catalog/                           # Bounded context: Product Catalog
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── api/
│
└── identity/                          # Bounded context: Identity & Access
    ├── domain/
    ├── application/
    ├── infrastructure/
    └── api/
```

**Benefits:**
- Change to "invoice cancellation" touches only `billing/` package
- Business concepts are discoverable and cohesive
- Each module can evolve independently
- Clear ownership boundaries

---

## Alternative: Lightweight Structure

If `domain/application/infrastructure/api` feels heavy, use fewer folders while keeping business-first:

```
com.acme.app/
├── ordering/
│   ├── Order.java              # Aggregate root
│   ├── OrderLine.java          # Entity
│   ├── OrderId.java            # Value object
│   ├── OrderRepository.java    # Interface
│   ├── JpaOrderRepository.java # Implementation
│   ├── OrderService.java       # Use case / application service
│   ├── OrderController.java    # API
│   ├── CreateOrderRequest.java # DTO
│   └── OrderResponse.java      # DTO
├── billing/
│   └── ...
└── catalog/
    └── ...
```

Still business-first; just fewer "rings".

---

## Rules of Thumb

| Principle | Guideline |
|-----------|-----------|
| Package naming | Business names at top (`ordering/`), technical names at leaves (`infrastructure/`) |
| Change locality | A business change should mostly touch **one module** |
| Domain purity | Domain types don't depend on Spring (ideally no `@Entity`, `@Component` in pure domain) |
| Cross-module calls | Go through explicit APIs (use cases, ports, domain events), not internal classes |
| Dependency direction | `api` → `application` → `domain` ← `infrastructure` |

---

## Cross-Module Communication

Modules should not reach into each other's internals. Use:

### 1. Published Domain Events

```java
// In ordering module
public record OrderPlacedEvent(OrderId orderId, CustomerId customerId, Money total) {}

// In billing module - listens to ordering events
@Component
public class OrderEventListener {
    private final IssueInvoiceUseCase issueInvoice;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderPlaced(OrderPlacedEvent event) {
        issueInvoice.execute(new IssueInvoiceCommand(event.orderId(), event.total()));
    }
}
```

### 2. Explicit Ports (Interfaces)

```java
// In ordering module - defines what it needs from catalog
public interface ProductCatalog {
    ProductInfo getProduct(ProductId id);
    boolean isAvailable(ProductId id, int quantity);
}

// In catalog module - implements the port
@Component
public class CatalogProductAdapter implements ProductCatalog {
    private final ProductRepository productRepository;
    // ...
}
```

### 3. Shared Kernel (Sparingly)

```java
// Shared value objects used across modules
com.acme.app.shared/
├── Money.java
├── CustomerId.java
└── AuditInfo.java
```

---

## Aggregate Root (No Lombok)

```java
public class Order {
    private final OrderId id;
    private final CustomerId customerId;
    private final List<OrderLine> lines;
    private OrderStatus status;
    private Money totalAmount;
    private final Instant createdAt;

    // Private constructor - use factory method
    private Order(OrderId id, CustomerId customerId, List<OrderLine> lines) {
        this.id = Objects.requireNonNull(id);
        this.customerId = Objects.requireNonNull(customerId);
        this.lines = new ArrayList<>(lines);
        this.status = OrderStatus.DRAFT;
        this.totalAmount = calculateTotal();
        this.createdAt = Instant.now();
    }

    // Factory method with validation
    public static Order create(CustomerId customerId, List<OrderLine> lines) {
        if (lines == null || lines.isEmpty()) {
            throw new IllegalArgumentException("Order must have at least one line");
        }
        return new Order(OrderId.generate(), customerId, lines);
    }

    // Domain behavior
    public void submit() {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Only draft orders can be submitted");
        }
        this.status = OrderStatus.SUBMITTED;
    }

    public void addLine(OrderLine line) {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Cannot modify submitted order");
        }
        this.lines.add(line);
        this.totalAmount = calculateTotal();
    }

    private Money calculateTotal() {
        return lines.stream()
            .map(OrderLine::subtotal)
            .reduce(Money.ZERO, Money::add);
    }

    // Explicit getters
    public OrderId getId() { return id; }
    public CustomerId getCustomerId() { return customerId; }
    public List<OrderLine> getLines() { return List.copyOf(lines); }
    public OrderStatus getStatus() { return status; }
    public Money getTotalAmount() { return totalAmount; }
    public Instant getCreatedAt() { return createdAt; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order order)) return false;
        return Objects.equals(id, order.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}
```

## Value Objects as Records

```java
// Simple value object
public record Money(BigDecimal amount, Currency currency) {
    public static final Money ZERO = new Money(BigDecimal.ZERO, Currency.getInstance("USD"));

    public Money {
        Objects.requireNonNull(amount, "amount required");
        Objects.requireNonNull(currency, "currency required");
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money multiply(int quantity) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(quantity)), this.currency);
    }
}

// ID value object
public record OrderId(UUID value) {
    public OrderId {
        Objects.requireNonNull(value, "OrderId value required");
    }

    public static OrderId generate() {
        return new OrderId(UUID.randomUUID());
    }

    public static OrderId of(String value) {
        return new OrderId(UUID.fromString(value));
    }

    @Override
    public String toString() {
        return value.toString();
    }
}
```

## Repository Interface (Domain Layer)

```java
// In domain/repository - interface only, no Spring annotations
public interface OrderRepository {
    Order findById(OrderId id);
    Optional<Order> findByIdOptional(OrderId id);
    List<Order> findByCustomerId(CustomerId customerId);
    void save(Order order);
    void delete(OrderId id);
}
```

## Application Service

```java
@Service
@Transactional
public class OrderService {
    private final OrderRepository orderRepository;
    private final CustomerRepository customerRepository;
    private final ApplicationEventPublisher eventPublisher;

    public OrderService(OrderRepository orderRepository,
                        CustomerRepository customerRepository,
                        ApplicationEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.customerRepository = customerRepository;
        this.eventPublisher = eventPublisher;
    }

    public OrderDto placeOrder(PlaceOrderCommand command) {
        // Validate customer exists
        var customer = customerRepository.findById(command.customerId())
            .orElseThrow(() -> new CustomerNotFoundException(command.customerId()));

        // Create domain object
        var lines = command.items().stream()
            .map(item -> new OrderLine(item.productId(), item.quantity(), item.unitPrice()))
            .toList();

        var order = Order.create(customer.getId(), lines);
        order.submit();

        // Persist
        orderRepository.save(order);

        // Publish domain event
        eventPublisher.publishEvent(new OrderPlacedEvent(order.getId(), order.getTotalAmount()));

        return OrderDto.from(order);
    }
}
```

## Command and DTO Records

```java
// Command - input to application service
public record PlaceOrderCommand(
    CustomerId customerId,
    List<OrderItemCommand> items
) {
    public record OrderItemCommand(ProductId productId, int quantity, Money unitPrice) {}
}

// DTO - output from application service
public record OrderDto(
    String id,
    String customerId,
    String status,
    BigDecimal totalAmount,
    String currency,
    Instant createdAt
) {
    public static OrderDto from(Order order) {
        return new OrderDto(
            order.getId().toString(),
            order.getCustomerId().toString(),
            order.getStatus().name(),
            order.getTotalAmount().amount(),
            order.getTotalAmount().currency().getCurrencyCode(),
            order.getCreatedAt()
        );
    }
}
```

## Domain Events

```java
// Event record
public record OrderPlacedEvent(OrderId orderId, Money totalAmount) {}

// Event listener
@Component
public class OrderEventListener {
    private static final Logger log = LoggerFactory.getLogger(OrderEventListener.class);

    @EventListener
    public void handleOrderPlaced(OrderPlacedEvent event) {
        log.info("Order placed: {} with total {}", event.orderId(), event.totalAmount());
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void sendOrderConfirmation(OrderPlacedEvent event) {
        // Send email, trigger async processing, etc.
    }
}
```

## Controller (Thin Adapter)

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    public ResponseEntity<OrderResponse> placeOrder(@Valid @RequestBody OrderRequest request) {
        var command = request.toCommand();
        var dto = orderService.placeOrder(command);
        return ResponseEntity
            .created(URI.create("/api/orders/" + dto.id()))
            .body(OrderResponse.from(dto));
    }
}

// Request/Response records for API layer
public record OrderRequest(
    @NotNull String customerId,
    @NotEmpty List<OrderItemRequest> items
) {
    public PlaceOrderCommand toCommand() {
        return new PlaceOrderCommand(
            CustomerId.of(customerId),
            items.stream().map(OrderItemRequest::toCommand).toList()
        );
    }

    public record OrderItemRequest(
        @NotNull String productId,
        @Positive int quantity,
        @NotNull BigDecimal unitPrice
    ) {
        public PlaceOrderCommand.OrderItemCommand toCommand() {
            return new PlaceOrderCommand.OrderItemCommand(
                ProductId.of(productId),
                quantity,
                new Money(unitPrice, Currency.getInstance("USD"))
            );
        }
    }
}
```

## Builder Pattern (Without Lombok)

```java
public class Order {
    // ... fields and constructor ...

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private CustomerId customerId;
        private List<OrderLine> lines = new ArrayList<>();
        private OrderStatus status = OrderStatus.DRAFT;

        private Builder() {}

        public Builder customerId(CustomerId customerId) {
            this.customerId = customerId;
            return this;
        }

        public Builder addLine(OrderLine line) {
            this.lines.add(line);
            return this;
        }

        public Builder lines(List<OrderLine> lines) {
            this.lines = new ArrayList<>(lines);
            return this;
        }

        public Order build() {
            Objects.requireNonNull(customerId, "customerId required");
            if (lines.isEmpty()) {
                throw new IllegalStateException("At least one order line required");
            }
            return new Order(OrderId.generate(), customerId, lines);
        }
    }
}
```
