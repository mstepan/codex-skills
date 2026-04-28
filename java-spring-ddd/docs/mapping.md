# Entity ⇄ DTO Mapping with MapStruct

## Core Principle

**Mapping lives in dedicated MapStruct mappers, invoked by application services; controllers stay thin, and relationship resolution plus business rules live in services/domain.**

---

## Layer Responsibilities

| Layer | Does | Does NOT |
|-------|------|----------|
| **Controller** | Routing, validation trigger, status codes, exception translation | Touch entities, contain mapping logic, resolve relationships |
| **Application Service** | Transactions, authorization, relationship resolution, invoke mappers, call domain behavior | Expose entities to API |
| **Domain** | Enforce invariants, express behavior via methods | Know about DTOs |
| **Mapper** | Structural conversion (DTO↔Entity/Command) | Repository access, business logic, relationship resolution |

---

## Mapping Rules

### 1. Request DTO → Command, not directly to Entity

```java
// ✓ Correct: Request → Command → Domain behavior
CreateOrderCommand toCommand(CreateOrderRequest dto);

// ✗ Wrong: Request → Entity directly
Order toEntity(CreateOrderRequest dto);
```

### 2. Never let clients set server-owned fields

Always ignore in MapStruct:
```java
@Mapping(target = "id", ignore = true)
@Mapping(target = "version", ignore = true)
@Mapping(target = "createdAt", ignore = true)
@Mapping(target = "createdBy", ignore = true)
@Mapping(target = "updatedAt", ignore = true)
```

### 3. Updates: Load entity, apply with @MappingTarget

```java
@Mapper(
    componentModel = "spring",
    nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE
)
public interface CustomerMapper {

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "version", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    void apply(UpdateCustomerRequest dto, @MappingTarget Customer customer);

    CustomerResponse toResponse(Customer customer);
}
```

### 4. Associations: IDs in DTO, resolve in service

```java
// DTO carries IDs
public record UpdateOrderRequest(
    String description,
    UUID customerId,      // ID only
    UUID accountManagerId // ID only
) {}

// Service resolves references
@Transactional
public OrderResponse updateOrder(UUID id, UpdateOrderRequest req) {
    Order order = orderRepository.findById(id).orElseThrow(NotFoundException::new);

    mapper.apply(req, order); // scalar fields only

    if (req.customerId() != null) {
        Customer customer = customerRepository.getReferenceById(req.customerId());
        order.assignCustomer(customer); // domain method
    }

    if (req.accountManagerId() != null) {
        Employee mgr = employeeRepository.getReferenceById(req.accountManagerId());
        order.assignAccountManager(mgr); // domain method
    }

    return mapper.toResponse(order);
}
```

### 5. Fetch required graph before response mapping

```java
// ✓ Intentional fetching
@Query("SELECT o FROM Order o JOIN FETCH o.lines WHERE o.id = :id")
Optional<Order> findByIdWithLines(UUID id);

// Then map
return mapper.toResponse(order);
```

### 6. Use-case oriented DTOs

```java
// ✓ Correct: specific DTOs per use case
public record CreateOrderRequest(...) {}
public record UpdateOrderRequest(...) {}
public record OrderResponse(...) {}
public record OrderSummaryResponse(...) {}

// ✗ Wrong: mega DTO with optional fields
public record OrderDTO(
    @Nullable String field1,
    @Nullable String field2,
    // ... 50 more optional fields
) {}
```

---

## MapStruct Mapper Patterns

### Basic Mapper Structure

```java
@Mapper(componentModel = "spring")
public interface OrderMapper {

    // Request → Command
    CreateOrderCommand toCommand(CreateOrderRequest request);

    // Domain → Response
    OrderResponse toResponse(Order order);

    // Collection mapping (automatic)
    List<OrderResponse> toResponseList(List<Order> orders);
}
```

### Update Mapper with @MappingTarget

```java
@Mapper(
    componentModel = "spring",
    nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE
)
public interface CustomerMapper {

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "version", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    @Mapping(target = "createdBy", ignore = true)
    @Mapping(target = "accountManager", ignore = true) // resolved in service
    void apply(UpdateCustomerRequest dto, @MappingTarget Customer customer);

    CustomerResponse toResponse(Customer customer);
}
```

### Nested Object Mapping

```java
@Mapper(componentModel = "spring")
public interface OrderMapper {

    @Mapping(target = "customerName", source = "customer.name")
    @Mapping(target = "totalAmount", source = "totalAmount.amount")
    @Mapping(target = "currency", source = "totalAmount.currency.currencyCode")
    OrderResponse toResponse(Order order);

    @Mapping(target = "subtotal", expression = "java(line.getUnitPrice().multiply(line.getQuantity()))")
    OrderLineResponse toLineResponse(OrderLine line);
}
```

### Custom Mapping Methods

```java
@Mapper(componentModel = "spring")
public interface OrderMapper {

    OrderResponse toResponse(Order order);

    // Custom mapping for value objects
    default String map(OrderId orderId) {
        return orderId != null ? orderId.value().toString() : null;
    }

    default String map(Money money) {
        return money != null ? money.amount().toPlainString() : null;
    }
}
```

### Mapping with Context (for special cases)

```java
@Mapper(componentModel = "spring")
public interface AuditMapper {

    @Mapping(target = "createdBy", source = "context.userId")
    @Mapping(target = "createdAt", expression = "java(java.time.Instant.now())")
    AuditInfo createAuditInfo(@Context SecurityContext context);
}
```

---

## Package Structure (Business-First)

```
com.acme.app/
├── ordering/                        # Bounded context
│   ├── domain/
│   │   └── model/
│   │       ├── Order.java
│   │       └── OrderLine.java
│   ├── application/
│   │   ├── PlaceOrderUseCase.java
│   │   ├── command/
│   │   │   └── PlaceOrderCommand.java
│   │   └── mapper/
│   │       └── OrderMapper.java     # MapStruct mapper
│   ├── infrastructure/
│   │   └── persistence/
│   │       └── JpaOrderRepository.java
│   └── api/
│       ├── OrderController.java
│       ├── request/
│       │   ├── CreateOrderRequest.java
│       │   └── UpdateOrderRequest.java
│       └── response/
│           └── OrderResponse.java
└── billing/
    └── ...
```

**Note:** Mappers live in `application/mapper/` within each business module, not in a global `mapping` package.

---

## Use Case Pattern with Mapper

```java
@Service
@Transactional
public class PlaceOrderUseCase {

    private final OrderRepository orderRepository;
    private final CustomerRepository customerRepository;
    private final OrderMapper mapper;

    public PlaceOrderUseCase(OrderRepository orderRepository,
                             CustomerRepository customerRepository,
                             OrderMapper mapper) {
        this.orderRepository = orderRepository;
        this.customerRepository = customerRepository;
        this.mapper = mapper;
    }

    public OrderResponse execute(CreateOrderRequest request) {
        // 1. Map request to command
        PlaceOrderCommand command = mapper.toCommand(request);

        // 2. Resolve relationships
        Customer customer = customerRepository.findById(request.customerId())
            .orElseThrow(() -> new NotFoundException("Customer not found"));

        // 3. Create domain object via factory/constructor
        Order order = Order.create(customer, command.lines());

        // 4. Persist
        orderRepository.save(order);

        // 5. Map to response
        return mapper.toResponse(order);
    }
}

@Service
@Transactional
public class UpdateOrderUseCase {

    private final OrderRepository orderRepository;
    private final CustomerRepository customerRepository;
    private final OrderMapper mapper;

    public UpdateOrderUseCase(OrderRepository orderRepository,
                              CustomerRepository customerRepository,
                              OrderMapper mapper) {
        this.orderRepository = orderRepository;
        this.customerRepository = customerRepository;
        this.mapper = mapper;
    }

    public OrderResponse execute(UUID id, UpdateOrderRequest request) {
        // 1. Load existing aggregate
        Order order = orderRepository.findById(id)
            .orElseThrow(() -> new NotFoundException("Order not found"));

        // 2. Apply scalar fields
        mapper.apply(request, order);

        // 3. Resolve and apply relationships
        if (request.customerId() != null) {
            Customer customer = customerRepository.getReferenceById(request.customerId());
            order.assignCustomer(customer);
        }

        // 4. Domain logic
        order.recalculate();

        // 5. Map to response
        return mapper.toResponse(order);
    }
}
```

---

## Checklist

**Controllers**
- [ ] Accept/return DTOs only
- [ ] No entity access
- [ ] No mapping logic beyond calling service

**Services**
- [ ] Own transactions and orchestration
- [ ] Resolve associations by ID
- [ ] Call domain methods (not setters)
- [ ] Use mappers for structural conversion

**Mappers**
- [ ] Pure structural mapping only
- [ ] `@MappingTarget` for updates
- [ ] `NullValuePropertyMappingStrategy.IGNORE` for partial updates
- [ ] Ignore server-owned fields explicitly
- [ ] No repository/DB access
- [ ] No business logic

**DTOs**
- [ ] Use-case specific (Create/Update/Response)
- [ ] No server-owned fields in requests
- [ ] IDs for associations, not nested objects
