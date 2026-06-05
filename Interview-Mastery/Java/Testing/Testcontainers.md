# Testcontainers

---

## 1. Executive Summary

Testcontainers is a Java library that spins up throwaway Docker containers for integration testing. It supports databases, message brokers, web browsers, and any Docker image.

### Why Testcontainers?
- **Real infrastructure** — tests against actual databases, not in-memory fakes
- **No mocking** — test real queries, transactions, constraints
- **Isolation** — fresh container per test class
- **CI-friendly** — works with Docker-in-Docker in CI pipelines

---

## 2. Production Code

### 2.1 Database Integration Test

```java
@SpringBootTest
@Testcontainers
class OrderRepositoryTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    void shouldPersistAndFindOrder() {
        Order order = new Order("customer@test.com", BigDecimal.valueOf(150));
        orderRepository.save(order);
        
        Optional<Order> found = orderRepository.findById(order.getId());
        
        assertTrue(found.isPresent());
        assertEquals("customer@test.com", found.get().getCustomerEmail());
    }
}
```

### 2.2 Multiple Containers

```java
@SpringBootTest
@Testcontainers
class OrderServiceIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7")
        .withExposedPorts(6379);
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));
    
    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }
    
    @Test
    void shouldProcessOrderEndToEnd() {
        // Test order creation → Kafka event → consumer processes
    }
}
```

### 2.3 Flyway Migration Test

```java
@SpringBootTest
@Testcontainers
class DatabaseMigrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    
    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired
    private DataSource dataSource;
    
    @Test
    void allMigrationsShouldApply() {
        // Flyway runs automatically with Spring Boot auto-configuration
        try (Connection conn = dataSource.getConnection();
             ResultSet rs = conn.getMetaData().getTables(null, null, "flyway_schema_history", null)) {
            assertTrue(rs.next(), "Flyway schema history table should exist");
        }
        
        // Verify specific table exists after migrations
        JdbcTestUtils.countRowsInTable((JdbcTemplate) jdbcTemplate, "orders");
    }
}
```

---

## 3. Cheat Sheet

```
═══ TESTCONTAINERS ════════════════════════════════════════════

┌─ DEPENDENCIES ─────────────────────────────────────────────┐
│ testcontainers               — Core library                 │
│ testcontainers:postgresql    — PostgreSQL module            │
│ testcontainers:mysql          — MySQL module                │
│ testcontainers:kafka          — Kafka module                │
│ testcontainers:redpanda       — Redpanda module            │
│ testcontainers:mongodb        — MongoDB module              │
│ testcontainers:junit-jupiter  — JUnit 5 integration         │
└─────────────────────────────────────────────────────────────┘

┌─ ANNOTATIONS ──────────────────────────────────────────────┐
│ @Testcontainers           — enables Testcontainers lifecycle │
│ @Container                — field-level container           │
│ @DynamicPropertySource    — inject container properties     │
└─────────────────────────────────────────────────────────────┘

┌─ COMMON CONTAINERS ────────────────────────────────────────┐
│ PostgreSQLContainer<>("postgres:15")    // Database         │
│ MySQLContainer<>("mysql:8")             // MySQL            │
│ KafkaContainer("cp-kafka:7.4.0")        // Kafka            │
│ GenericContainer<>("redis:7")           // Redis            │
│ new ToxiproxyContainer("ghcr.io/...)    // Network chaos    │
│ new LocalStackContainer("localstack")   // AWS mock         │
└─────────────────────────────────────────────────────────────┘

┌─ BEST PRACTICES ───────────────────────────────────────────┐
│ • Use static @Container (shared across tests in class)      │
│ • Set resource limits (.withCreateContainerCmdModifier)     │
│ • Use reusable containers for local dev                     │
│ • Combine with @DynamicPropertySource for config            │
│ • Test migrations with every deployment                     │
│ • Clean up test data between test classes                   │
└─────────────────────────────────────────────────────────────┘
```
