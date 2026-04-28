# Testing with Testcontainers

## Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
<!-- Database-specific -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
```

---

## Base Test Configuration

### Shared Container (Recommended)

```java
@SpringBootTest
@Testcontainers
public abstract class IntegrationTestBase {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:17.6")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @Container
    @ServiceConnection
    static RedisContainer redis = new RedisContainer("redis:7-alpine");
}
```

### Test Class

```java
class OrderServiceIntegrationTest extends IntegrationTestBase {

    @Autowired
    private OrderService orderService;

    @Autowired
    private OrderRepository orderRepository;

    @BeforeEach
    void setUp() {
        orderRepository.deleteAll();
    }

    @Test
    void shouldPlaceOrder() {
        var command = new PlaceOrderCommand(
            CustomerId.of("customer-1"),
            List.of(new OrderItemCommand(ProductId.of("prod-1"), 2, new Money(BigDecimal.TEN, USD)))
        );

        var result = orderService.placeOrder(command);

        assertThat(result.status()).isEqualTo("SUBMITTED");
        assertThat(orderRepository.findById(OrderId.of(result.id()))).isPresent();
    }
}
```

---

## Database Containers

### PostgreSQL

```java
@Container
@ServiceConnection
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:17.6")
    .withDatabaseName("testdb")
    .withInitScript("init.sql");  // Optional: runs on startup
```

### MySQL

```java
@Container
@ServiceConnection
static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8")
    .withDatabaseName("testdb");
```

### MongoDB

```java
@Container
@ServiceConnection
static MongoDBContainer mongo = new MongoDBContainer("mongo:7");
```

---

## Message Broker Containers

### Kafka

```java
@Container
@ServiceConnection
static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
```

### RabbitMQ

```java
@Container
@ServiceConnection
static RabbitMQContainer rabbitmq = new RabbitMQContainer("rabbitmq:3.12-management");
```

---

## Slice Tests with Testcontainers

### @DataJpaTest

```java
@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class OrderRepositoryTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:17.6");

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void shouldFindByCustomerIdWithLines() {
        // given
        var order = new Order("customer-1", Set.of(new OrderLine("prod-1", 2, BigDecimal.TEN)));
        entityManager.persistAndFlush(order);
        entityManager.clear();

        // when
        var result = orderRepository.findByCustomerIdWithLines("customer-1");

        // then
        assertThat(result).hasSize(1);
        assertThat(result.get(0).getLines()).hasSize(1);
    }

    @Test
    void shouldSaveAndFindOrder() {
        var order = new Order("cust-1", Set.of(
            new OrderLine("prod-1", 1, BigDecimal.TEN)
        ));

        var saved = orderRepository.save(order);
        entityManager.flush();
        entityManager.clear();

        assertThat(orderRepository.findById(saved.getId()))
            .isPresent()
            .hasValueSatisfying(o -> assertThat(o.getCustomerId()).isEqualTo("cust-1"));
    }
}
```

### @WebMvcTest

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockitoBean
    private OrderService orderService;

    @Test
    void shouldCreateOrder() throws Exception {
        var dto = new OrderDto("order-1", "cust-1", "SUBMITTED", BigDecimal.TEN, "USD", Instant.now());
        when(orderService.placeOrder(any())).thenReturn(dto);

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {
                        "customerId": "cust-1",
                        "items": [{"productId": "prod-1", "quantity": 1, "unitPrice": 10.00}]
                    }
                    """))
            .andExpect(status().isCreated())
            .andExpect(header().string("Location", "/api/orders/order-1"))
            .andExpect(jsonPath("$.id").value("order-1"));
    }
}
```

---

## Test Data Builders (No Lombok)

```java
public class OrderTestData {

    public static OrderBuilder anOrder() {
        return new OrderBuilder();
    }

    public static class OrderBuilder {
        private CustomerId customerId = CustomerId.of("default-customer");
        private List<OrderLine> lines = new ArrayList<>();
        private OrderStatus status = OrderStatus.DRAFT;

        public OrderBuilder withCustomerId(String id) {
            this.customerId = CustomerId.of(id);
            return this;
        }

        public OrderBuilder withLine(String productId, int quantity, BigDecimal price) {
            this.lines.add(new OrderLine(ProductId.of(productId), quantity, new Money(price, USD)));
            return this;
        }

        public OrderBuilder submitted() {
            this.status = OrderStatus.SUBMITTED;
            return this;
        }

        public Order build() {
            if (lines.isEmpty()) {
                lines.add(new OrderLine(ProductId.of("default-product"), 1, new Money(BigDecimal.TEN, USD)));
            }
            var order = Order.create(customerId, lines);
            if (status == OrderStatus.SUBMITTED) {
                order.submit();
            }
            return order;
        }
    }
}

// Usage in tests
@Test
void shouldCalculateTotal() {
    var order = anOrder()
        .withCustomerId("customer-1")
        .withLine("prod-1", 2, new BigDecimal("10.00"))
        .withLine("prod-2", 1, new BigDecimal("25.00"))
        .build();

    assertThat(order.getTotalAmount().amount()).isEqualByComparingTo("45.00");
}
```

---

## Test Fixtures with @Sql

```java
@SpringBootTest
@Testcontainers
@Sql(scripts = "/test-data/orders.sql", executionPhase = BEFORE_TEST_METHOD)
@Sql(scripts = "/test-data/cleanup.sql", executionPhase = AFTER_TEST_METHOD)
class OrderReportingTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:17.6");

    @Test
    void shouldGenerateMonthlyReport() {
        // test with pre-loaded data
    }
}
```

---

## Async and Event Testing

```java
@SpringBootTest
@Testcontainers
class OrderEventTest extends IntegrationTestBase {

    @Autowired
    private OrderService orderService;

    @SpyBean
    private OrderEventListener eventListener;

    @Test
    void shouldPublishOrderPlacedEvent() {
        var command = createValidOrderCommand();

        orderService.placeOrder(command);

        verify(eventListener, timeout(1000)).handleOrderPlaced(any(OrderPlacedEvent.class));
    }
}
```

---

## Performance: Reusable Containers

For faster test execution, reuse containers across test runs:

```properties
# ~/.testcontainers.properties
testcontainers.reuse.enable=true
```

```java
@Container
@ServiceConnection
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:17.6")
    .withReuse(true);
```

---

## Container Startup Optimization

```java
// Use lightweight images
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:17.6");

// Disable unnecessary features
static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"))
    .withEnv("KAFKA_AUTO_CREATE_TOPICS_ENABLE", "false");
```

---

## Disabling Ryuk (Resource Reaper)

Ryuk is a container that Testcontainers uses to clean up resources. In some CI environments or when using reusable containers, Ryuk can cause issues. Disable it by setting the environment variable:

```bash
TESTCONTAINERS_RYUK_DISABLED=true
```

**When to disable Ryuk:**
- Running tests in CI/CD pipelines where container cleanup is handled separately
- Using reusable containers (`.withReuse(true)`)
- Experiencing resource cleanup conflicts or timeout issues

**Configuration options:**

1. **Environment variable (recommended for CI):**
   ```bash
   export TESTCONTAINERS_RYUK_DISABLED=true
   ./gradlew test
   ```

2. **Testcontainers properties file:**
   ```properties
   # ~/.testcontainers.properties
   ryuk.disabled=true
   ```

3. **System property:**
   ```bash
   ./gradlew test -Dtestcontainers.ryuk.disabled=true
   ```
