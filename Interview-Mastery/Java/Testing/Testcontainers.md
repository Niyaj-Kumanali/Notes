# Testcontainers

---

## Overview

- **Definition:** Testcontainers is a Java library that spins up throwaway Docker containers for integration testing. It supports databases, message brokers, web browsers, and any Docker image.

- **Why Testcontainers?**
  - **Real infrastructure** — tests against actual databases, not in-memory fakes (H2)
  - **No mocking** — test real queries, transactions, constraints, and stored procedures
  - **Isolation** — fresh container per test class (no shared state)
  - **CI-friendly** — works with Docker-in-Docker in CI pipelines

---

## Core Concepts

- **@Testcontainers:** Enables Testcontainers lifecycle management for the test class. Starts containers before tests, stops them after.

- **@Container:** Marks a field as a container instance. Static containers are shared across all tests in the class; instance containers are started per test.

- **@DynamicPropertySource:** Injects container connection details (URL, port, credentials) into Spring's Environment property source before the application context starts.

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
        Order order = new Order("test@test.com", BigDecimal.valueOf(150));
        orderRepository.save(order);

        Optional<Order> found = orderRepository.findById(order.getId());
        assertTrue(found.isPresent());
    }
}
```

---

## Common Containers

| Container Class | Use Case |
|----------------|----------|
| `PostgreSQLContainer` | PostgreSQL database |
| `MySQLContainer` | MySQL database |
| `MongoDBContainer` | MongoDB database |
| `KafkaContainer` | Apache Kafka |
| `GenericContainer` | Any Docker image |
| `ToxiproxyContainer` | Network chaos testing |
| `LocalStackContainer` | AWS services mock |

```java
// Multiple containers
@Container
static GenericContainer<?> redis = new GenericContainer<>("redis:7")
    .withExposedPorts(6379);

@Container
static KafkaContainer kafka = new KafkaContainer(
    DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));
```

---

## Best Practices

- **Use static `@Container`** — shared across tests in the class to avoid restarting for every test
- **Set resource limits** — `.withCreateContainerCmdModifier(cmd -> cmd.withMemory(...))`
- **Use reusable containers for local dev** — `.withReuse(true)` keeps containers running between test runs
- **Combine with `@DynamicPropertySource`** for Spring Boot configuration
- **Test migrations** — verify Flyway/Liquibase migrations apply cleanly on the real database
- **Clean up test data between test classes** — use `@Sql` or `@DirtiesContext`
- **Always pull specific tags** — `postgres:15` not `postgres:latest` for reproducible builds

---

## Common Mistakes

- **Non-static `@Container`** — creates a new container per test, very slow. This *looks correct* because the `@Container` annotation is present and the container starts — the developer only notices the problem when the test suite takes 10 minutes instead of 30 seconds. Use `static` for containers shared across all tests.
- **Forgetting `@Testcontainers`** — container lifecycle is not managed, container never starts. This *looks correct* because the `@Container` annotation is present on the field and the code compiles — the container simply never starts, and the test fails with a confusing connection refused error.
- **Wrong `@DynamicPropertySource` method signature** — must be `static void` with `DynamicPropertyRegistry` parameter. This *looks correct* because the method compiles and the IDE may not flag it — the method simply never executes, and the Spring context uses default properties.
- **Using `localhost` instead of container host** — containers run in their own network; use `postgres::getJdbcUrl` instead of hardcoded strings. This *looks correct* because `localhost:5432` is the standard PostgreSQL URL and works when the developer has a local PostgreSQL running — the test connects to the wrong database without error.
- **Container version mismatch with local database** — test against the same database version used in production. This *looks correct* because the tests pass against `postgres:15` while production runs `postgres:14` — the minor version difference rarely causes issues until a query uses a feature not in production.
- **Not handling container startup failures** — container may fail to start on resource-constrained CI runners. This *looks correct* because the container starts reliably on a developer's machine with sufficient resources — the startup failure only appears in CI with limited Docker memory.

---

## Real-World Scenarios

### Scenario 1: Database Migration Testing Across Environments

A team uses Flyway for schema migrations. A developer writes a migration that adds a `JSONB` column with a PostgreSQL-specific index. The migration passes with H2 in unit tests but fails in production because H2 doesn't support `JSONB`. The team needs a CI gate that catches database-specific SQL before deployment.

```java
@SpringBootTest
@Testcontainers
class FlywayMigrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withCopyFileToContainer(
            MountableFile.forClasspathResource("test-data.sql"),
            "/docker-entrypoint-initdb.d/");

    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.flyway.enabled", () -> "true");
    }

    @Autowired
    private DataSource dataSource;

    @Test
    void allMigrationsShouldApplySuccessfully() {
        JdbcTemplate jdbc = new JdbcTemplate(dataSource);
        List<String> tables = jdbc.queryForList(
            "SELECT table_name FROM information_schema.tables WHERE table_schema = 'public'", String.class);
        assertThat(tables).contains("orders", "users", "products");

        // Verify PostgreSQL-specific features
        String columnType = jdbc.queryForObject(
            "SELECT data_type FROM information_schema.columns WHERE table_name = 'orders' AND column_name = 'metadata'",
            String.class);
        assertEquals("jsonb", columnType);
    }
}
```

The test starts a real PostgreSQL 15 container, applies all Flyway migrations, and verifies table structure with PostgreSQL-specific queries. Without this test, a migration using `JSONB`, `ARRAY`, `ENUM`, or `GIN` indexes would silently pass H2 tests and fail in production. The `MountableFile` seeds initial data to test migration idempotency.

### Scenario 2: Integration Testing with Multiple Containers

A microservice uses PostgreSQL, Redis (caching), and Kafka (event bus). An integration test for the order processing flow must verify that an order flows through the database, triggers a cache invalidation in Redis, and publishes an event to Kafka.

```java
@SpringBootTest
@Testcontainers
class OrderFlowIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
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

    @Autowired
    private OrderService orderService;

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    @Test
    void shouldFlowEndToEnd() {
        Order order = orderService.createOrder(new CreateOrderRequest("test@test.com", List.of("item-1")));

        assertNotNull(order.getId());
        assertThat(redisTemplate.opsForValue().get("cache:order:" + order.getId())).isNull(); // cache invalidated

        await().atMost(10, TimeUnit.SECONDS)
            .untilAsserted(() -> {
                OrderEvent received = kafkaTemplate.receive("order.created", 0, 0);
                assertThat(received).isNotNull();
                assertThat(received.orderId()).isEqualTo(order.getId());
            });
    }
}
```

Three containers are managed by `@Container` annotations — they start in parallel, share a network, and are torn down after the test class. The `@DynamicPropertySource` method injects the dynamically allocated ports into Spring's Environment. Awaitility handles the async Kafka assertion with polling. This test catches integration bugs that unit tests (with mocks) would miss: serialization issues, transaction boundaries, and cache invalidation ordering.

### Scenario 3: Network Resilience Testing with Toxiproxy

A payment service retries database operations on connection failure. The retry logic has exponential backoff, a maximum of 3 retries, and a circuit breaker that opens after 5 consecutive failures. This behavior must be tested without actually crashing the database.

```java
@SpringBootTest
@Testcontainers
class DatabaseResilienceTest {
    @Container
    static ToxiproxyContainer toxiproxy = new ToxiproxyContainer(
            "ghcr.io/shopify/toxiproxy:2.8.0");

    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
    static ToxiproxyContainer.ContainerProxy proxy;

    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", () ->
            "jdbc:postgresql://" + toxiproxy.getHost() + ":" + proxy.getProxyPort() + "/test");
    }

    @BeforeAll
    static void setup() {
        postgres.start();
        proxy = toxiproxy.getProxy(postgres, 5432);
    }

    @Test
    void shouldRetryOnConnectionTimeout() {
        proxy.toxics().latency("latency", ToxicDirection.DOWNSTREAM, 5_000);

        assertDoesNotThrow(() -> orderService.findOrders());
        // Verify the retry happened
        assertThat(orderService.getRetryCount()).isLessThanOrEqualTo(3);
    }

    @Test
    void shouldOpenCircuitBreakerAfterFailures() {
        proxy.setConnectionCut(true);

        for (int i = 0; i < 10; i++) {
            assertThrows(CircuitBreakerOpenException.class,
                () -> orderService.findOrders());
        }

        // After circuit opens, it should fail fast (no DB call attempted)
        assertThat(proxy.getConnectionCount()).isLessThan(10);
    }
}
```

Toxiproxy sits between the application and PostgreSQL, injecting latency or cutting the connection entirely. The `latency` toxic simulates a slow database (5s response time) to test client-side timeouts. The `connectionCut` toxic simulates a network partition. The circuit breaker test verifies that after enough failures, the application stops calling the database entirely (fail-fast). This tests resilience patterns without needing actual network failures.

---

## Scenario-Based Questions

1. **Q: You are migrating a Spring Boot application from H2 in-memory database to PostgreSQL. The application has 200 repository tests that use `@DataJpaTest` with H2. Management wants to move to production in 2 weeks. How do you safely transition without rewriting all tests?**
   A: Add Testcontainers PostgreSQL tests alongside existing H2 tests, then gradually migrate:
   ```java
   // Phase 1: Both run in CI (H2 for speed, PostgreSQL for accuracy)
   @Tag("h2")
   @DataJpaTest
   class OrderRepositoryH2Test { /* existing H2 tests */ }

   @Tag("postgres")
   @DataJpaTest
   @Testcontainers
   @AutoConfigureTestDatabase(replace = NONE) // don't replace with H2
   class OrderRepositoryPostgresTest {
       @Container static PostgreSQLContainer<?> pg = new PostgreSQLContainer<>("postgres:15");
       @DynamicPropertySource static void props(DynamicPropertyRegistry r) { /* ... */ }
       // Same test methods as H2 test — run against real PostgreSQL
   }
   ```
   Strategy: (1) Keep H2 tests for fast local feedback (1s). (2) Add PostgreSQL variants that run in CI (30s). (3) Run both in CI for 1 sprint to catch all differences. (4) Remove H2 tests when PostgreSQL coverage is complete. Common differences caught: `BOOLEAN` vs `BIT`, `LONGVARCHAR` vs `TEXT`, constraint deferrability, sequence allocation size, and `Enum` ordinal vs string mapping.

2. **Q: A developer onboards to the project and runs tests locally for the first time. Testcontainers takes 5 minutes to pull the PostgreSQL image. The developer's internet is slow. The tests fail because Docker Desktop isn't running. How do you make the first-run experience smooth?**
   A: Pre-pull images in a build script, and add a clear error message when Docker is unavailable:

   > **Interview follow-up:** The candidate suggested `@Testcontainers(disabledWithoutDocker = true)` to skip tests when Docker is unavailable. The developer's machine passes CI checks locally by skipping all Testcontainers tests. They submit a PR that introduces a PostgreSQL-specific query using `JSONB` — all tests pass locally (because they were skipped), all unit tests pass in CI, but the integration tests in CI catch the incompatibility only after 15 minutes of pipeline time. How would you design the local dev workflow so that a developer must have Testcontainers working before they can merge, without forcing every team member to always run the full integration suite?
   ```java
   // Build script (Maven/Gradle): pre-pulls images before tests
   // mvn validate or gradle --no-daemon testClasses pulls images

   // Graceful fallback when Docker is unavailable
   @Testcontainers(disabledWithoutDocker = true) // Skip tests if Docker missing
   @SpringBootTest
   class OrderRepositoryTest {
       @Container
       static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15");
   }
   ```
   `@Testcontainers(disabledWithoutDocker = true)` skips the entire test class if Docker isn't available, with a clear log message. For first-run speed: (1) Add a `DockerSetup` task in the build that runs `docker pull` before tests. (2) Use `withReuse(true)` — the container stays running between test runs. (3) Document in the README: "Run `docker pull postgres:15` once before first test." (4) For CI, cache the Docker image layer in the CI cache.

3. **Q: A service uses MongoDB with unique indexes and GridFS for file storage. You want to test repository methods with Testcontainers. The MongoDB container takes 15 seconds to start and each test class starts a new container. The test suite has 10 test classes. How do you share one MongoDB container across all test classes?**
   A: Create a shared container in an abstract base class with manual lifecycle control:
   ```java
   public abstract class MongoTestBase {
       private static final MongoDBContainer mongo = new MongoDBContainer("mongo:7");

       static {
           mongo.start();
       }

       @DynamicPropertySource
       static void configureProperties(DynamicPropertyRegistry registry) {
           registry.add("spring.data.mongodb.uri", mongo::getReplicaSetUrl);
       }
   }

   class UserRepositoryTest extends MongoTestBase { /* ... */ }
   class FileStoreTest extends MongoTestBase { /* ... */ }
   ```
   The container starts once in the static initializer and stays running for all test classes. `@DynamicPropertySource` runs before the Spring context starts, ensuring the correct connection URI. For parallel execution, use `synchronized` on the container start or use Testcontainers' built-in singleton container support. The image `mongo:7` uses MongoDB's replica set mode which supports transactions. This reduces test time from 15s × 10 = 150s to 15s + 9 × 0.5s = 19.5s.

4. **Q: A microservice uses S3 (via LocalStack) to store user-uploaded files. An integration test uploads a file, verifies it's stored, downloads it, and checks the content. The test passes locally but fails in CI because LocalStack's S3 API behavior differs from production AWS S3 in subtle ways. How do you write tests that work reliably?**
   A: Pin LocalStack to a specific version and use the S3 API compatibility mode:
   ```java
   @Container
   static LocalStackContainer localstack = new LocalStackContainer(
           DockerImageName.parse("localstack/localstack:3.0.0"))
       .withServices(Service.S3);

   @DynamicPropertySource
   static void properties(DynamicPropertyRegistry registry) {
       registry.add("spring.cloud.aws.s3.endpoint",
           () -> localstack.getEndpointOverride(Service.S3).toString());
       registry.add("spring.cloud.aws.credentials.access-key", () -> "test");
       registry.add("spring.cloud.aws.credentials.secret-key", () -> "test");
       registry.add("spring.cloud.aws.region.static", () -> "us-east-1");
   }

   @Test
   void shouldUploadAndDownloadFile() {
       String key = "test/" + UUID.randomUUID() + ".txt";
       s3Client.putObject(bucket, key, "Hello World");

       byte[] data = s3Client.getObjectAsBytes(bucket, key);
       assertEquals("Hello World", new String(data));
   }
   ```
   Pin the specific LocalStack version (`3.0.0`) — `latest` can change behavior between CI runs. Use `getEndpointOverride()` which returns the correct local URL with port. For production parity: (1) Use `withLegacyEndpointMode()` if needed for SDK v1 compatibility. (2) Test S3 event notifications with LocalStack's notification system. (3) For S3 consistency model differences (read-after-write vs eventual), document the divergence between LocalStack and production AWS.

5. **Q: A test uses `@Container` PostgreSQL with a Flyway migration that creates 20 tables. Each test method in the class modifies data. After running 20 test methods, the database has accumulated test data. The 21st test fails because it assumes an empty database. How do you handle test isolation with the shared container?**
   A: Use `@Sql` to reset data between tests or truncate tables in `@BeforeEach`:
   ```java
   @SpringBootTest
   @Testcontainers
   class OrderRepositoryTest {
       @Autowired private JdbcTemplate jdbc;

       @BeforeEach
       void cleanDatabase() {
           // Truncate all tables in correct order (respect foreign keys)
           jdbc.execute("TRUNCATE TABLE order_items CASCADE");
           jdbc.execute("TRUNCATE TABLE orders CASCADE");
           jdbc.execute("TRUNCATE TABLE users CASCADE");
       }

       @Test
       @Sql(statements = "INSERT INTO users (id, email) VALUES (1, 'test@test.com')")
       void testWithInitialData() {
           // ...
       }
   }
   ```
   For Spring Boot, use `@Sql(executionPhase = BEFORE_TEST_METHOD)` to run setup scripts and `@Sql(executionPhase = AFTER_TEST_METHOD)` for cleanup. For many test classes, extract the cleanup to a base class or use a `@BeforeEach` in an abstract base. TRUNCATE with CASCADE handles foreign key dependencies. For large schemas, consider restoring a database snapshot between test classes (using `pg_restore`) instead of running all migrations for each class.

6. **Q: A team writes integration tests with Testcontainers that connect to PostgreSQL, Redis, Kafka, and LocalStack (S3). Starting 4 containers per test class takes 2 minutes. The test suite has 30 test classes, totaling 60 minutes. How do you reduce total CI time to under 15 minutes?**
   A: Create a single integration test suite that shares all containers:

   > **Interview follow-up:** The candidate proposed an interface with shared static containers. After 6 months, one of the 30 test classes needs a specific PostgreSQL extension (`pg_stat_statements`) that requires a different container image and startup command. Adding this to the shared PostgreSQL container would affect all 29 other test classes, potentially breaking them. How would you design the shared container approach to allow per-test-class customization (different images, different startup parameters) while still sharing the container startup cost?
   ```java
   @SpringBootTest
   @Testcontainers
   public interface SharedContainers {
       PostgreSQLContainer<?> POSTGRES = new PostgreSQLContainer<>("postgres:15");
       GenericContainer<?> REDIS = new GenericContainer<>("redis:7-alpine").withExposedPorts(6379);
       KafkaContainer KAFKA = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

       @DynamicPropertySource
       static void properties(DynamicPropertyRegistry registry) {
           registry.add("spring.datasource.url", POSTGRES::getJdbcUrl);
           registry.add("spring.redis.host", REDIS::getHost);
           registry.add("spring.redis.port", () -> REDIS.getMappedPort(6379));
           registry.add("spring.kafka.bootstrap-servers", KAFKA::getBootstrapServers);
       }
   }

   // Multiple test classes implement the same interface
   class OrderRepositoryTest implements SharedContainers { /* ... */ }
   class KafkaEventTest implements SharedContainers { /* ... */ }
   ```
   All containers are `static` and initialized once per JVM. Test classes implement an interface that exposes the shared containers. The Spring context is also shared (cached by Spring Boot) — the `@DynamicPropertySource` runs once. Total time: 2 minutes (container startup) + 30 × 5s (test execution) = ~4.5 minutes. Use `@DirtiesContext` sparingly — it destroys the shared context and forces a restart.

7. **Q: A service uses Kafka transactions (exactly-once semantics). An integration test produces a message within a transaction, consumes it, and verifies the side effect. The test passes even when transactions are broken (the service processes the message before the transaction commits). How do you write a test that verifies transactional exactly-once processing?**
   A: Use Kafka's transactional API and verify that messages are consumed only after the transaction commits:
   ```java
   @Test
   void shouldProcessMessagesExactlyOnce() {
       // Start a Kafka transaction
       kafkaTemplate.executeInTransaction(operations -> {
           operations.send("orders", new OrderEvent("order-1", "CREATED"));
           operations.send("orders", new OrderEvent("order-1", "UPDATED"));
           return true;
       });

       // Consumer should get both messages exactly once
       await().atMost(10, TimeUnit.SECONDS)
           .untilAsserted(() -> {
               List<OrderEvent> received = testConsumer.getReceivedEvents();
               assertThat(received).hasSize(2);
               assertThat(received.get(0).status()).isEqualTo("CREATED");
               assertThat(received.get(1).status()).isEqualTo("UPDATED");
           });

       // Verify no duplicates
       long uniqueOrderIds = testConsumer.getReceivedEvents().stream()
           .map(OrderEvent::orderId).distinct().count();
       assertEquals(1, uniqueOrderIds);
   }

   @Test
   void shouldNotProcessAbortedTransaction() {
       // Start a transaction that is rolled back
       assertThrows(RuntimeException.class, () ->
           kafkaTemplate.executeInTransaction(operations -> {
               operations.send("orders", new OrderEvent("order-2", "CREATED"));
               throw new RuntimeException("rollback!");
           })
       );

       // Consumer should NOT receive the message (transaction was aborted)
       await().during(5, TimeUnit.SECONDS)
           .untilAsserted(() -> {
               List<OrderEvent> received = testConsumer.getReceivedEvents();
               assertThat(received).isEmpty();
           });
   }
   ```
   `executeInTransaction()` commits the transaction only if the lambda returns normally. If an exception is thrown, the transaction is rolled back and the message is never committed. The `during(5, SECONDS)` assertion verifies the negative case: the message does NOT arrive within 5 seconds. This tests the atomicity guarantee: either all messages in the transaction are delivered or none are.

8. **Q: A developer adds a Testcontainers test that uses `GenericContainer` with a custom image. The image entrypoint requires environment variables that differ between local dev and CI (API keys, secrets). How do you pass environment-specific configuration to Testcontainers without hardcoding secrets?**
   A: Use environment variables or `.env` files with `withEnv()`:
   ```java
   @Container
   static GenericContainer<?> customService = new GenericContainer<>("my-service:1.0")
       .withEnv("API_KEY", System.getenv("TEST_API_KEY"))
       .withEnv("ENVIRONMENT", "test")
       .withEnv("LOG_LEVEL", "DEBUG");
   ```
   Never hardcode secrets in test code. For local development, use a `.env` file loaded at runtime:
   ```java
   // Load from .env.test file
   Dotenv dotenv = Dotenv.configure().filename(".env.test").load();
   customService.withEnv("API_KEY", dotenv.get("API_KEY"));
   ```
   For CI, set secrets in the CI environment variables (`TEST_API_KEY`). Testcontainers supports `withCopyToContainer()` for config files mounted into the container. For complex configurations, create a `docker-compose.yml` for the external service and use `DockerComposeContainer` — it supports variable substitution from environment variables.

9. **Q: An integration test with Testcontainers starts 4 containers (PostgreSQL, Redis, Kafka, Elasticsearch). One of the containers fails to start in CI with "no space left on device". The Docker image cache has accumulated 10GB of old images. How do you manage Docker disk space in CI?**
   A: Add Docker cleanup to the CI pipeline and use `.withReuse(false)` in CI:
   ```yaml
   # CI pipeline — clean Docker before and after
   jobs:
     test:
       steps:
         - name: Clean Docker
           run: docker system prune -af --volumes
         - name: Run tests
           run: mvn verify
         - name: Post-cleanup
           run: docker system prune -af --volumes
   ```
   Additional strategies: (1) Use smaller images: `postgres:15-alpine` (200MB vs 400MB), `redis:7-alpine` (30MB vs 120MB). (2) Set resource limits on containers:
   ```java
   new PostgreSQLContainer<>("postgres:15-alpine")
       .withCreateContainerCmdModifier(cmd -> cmd.getHostConfig()
           .withMemory(512 * 1024 * 1024L)  // 512MB max
           .withCpuCount(1L));
   ```
   (3) Combine containers where possible — use `Redpanda` instead of Kafka (smaller image, single binary). (4) Cache the Testcontainers `~/.testcontainers.properties` in CI with `testcontainers.reuse.enable=true` and a persistent Docker volume.

10. **Q: A service sends notifications to Slack, PagerDuty, and email when critical errors occur. You want to write an integration test that verifies notifications are sent for specific error scenarios without actually sending real notifications. How do you mock external notification services with Testcontainers?**
    A: Use `MockServerContainer` (or WireMock via `GenericContainer`) to simulate the notification endpoints:
    ```java
    @Container
    static MockServerContainer mockServer = new MockServerContainer(
        DockerImageName.parse("mockserver/mockserver:5.15.0"));

    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("notification.slack.url",
            () -> "http://" + mockServer.getHost() + ":" + mockServer.getServerPort() + "/slack");
        registry.add("notification.pagerduty.url",
            () -> "http://" + mockServer.getHost() + ":" + mockServer.getServerPort() + "/pagerduty");
    }

    @BeforeEach
    void setupMocks() {
        mockServerClient = new MockServerClient(mockServer.getHost(), mockServer.getServerPort());
        mockServerClient.reset();
    }

    @Test
    void shouldNotifyOnCriticalError() {
        mockServerClient.when(
            request().withPath("/slack").withMethod("POST")
        ).respond(response().withStatusCode(200));

        service.triggerCriticalError();

        mockServerClient.verify(
            request().withPath("/slack")
                .withBody(containsString("CRITICAL")),
            Times.atLeast(1)
        );
    }
    ```
    MockServer acts as a real HTTP server at the configured URL. The application posts notifications to it as if it were the real Slack/PagerDuty. The test verifies the request body content, HTTP method, and headers. Unlike `@MockBean` which mocks the Java interface, MockServer tests the entire HTTP client stack — including serialization, timeouts, and retries. Each test class gets its own MockServer container, and `mockServerClient.reset()` clears expectations between tests.

---

## Interview Questions

1. **What is Testcontainers and why would you use it?**
   A: Testcontainers is a Java library that spins up disposable Docker containers for integration testing. It provides real infrastructure (databases, message brokers, browsers) instead of in-memory fakes (H2) or mocks. This catches environment-specific bugs: SQL syntax differences, driver behavior, charset encoding, and constraint enforcement. It works in CI with Docker-in-Docker and supports container reuse for fast local development.

2. **What is the difference between `@Container` on a static vs instance field?**
   A: `static @Container` — container starts once before all tests in the class and stops after the last test. `instance @Container` — container starts before each test method and stops after each test. Static is preferred: container startup is expensive (5-30s), and the test database can be cleaned between tests with DDL (truncate) rather than restarting. Instance containers are used for truly isolated tests (e.g., testing different database versions).

3. **What is `@DynamicPropertySource` used for?**
   A: `@DynamicPropertySource` injects container connection details (URL, port, credentials) into Spring's `Environment` before the application context starts. It must be `static void` with a `DynamicPropertyRegistry` parameter. This allows the application to connect to the dynamically allocated container ports without hardcoding. It runs once per test class (or once per context if shared across classes).

4. **What is the difference between `PostgreSQLContainer` and `GenericContainer`?**
   A: `PostgreSQLContainer` is a type-safe wrapper with convenience methods: `getJdbcUrl()`, `getUsername()`, `getPassword()`, `getDatabaseName()`, and supports initialization scripts. `GenericContainer` is the generic API for any Docker image — you manually configure ports, environment, commands, and health checks. Use typed containers for databases; use `GenericContainer` for custom services, caches, or tools.

5. **How do you share a container across multiple test classes?**
   A: Use a static container in an abstract base class or an interface with default methods. The container starts in a `static` block and is shared across all test classes that extend/implement the base. Use `@DynamicPropertySource` in the base class to inject the connection properties. `@DirtiesContext` should be avoided as it forces a new Spring context and may restart the container.

6. **What is container reuse and how do you enable it?**
   A: Container reuse (`withReuse(true)`) keeps the container running between test runs. After the first run that starts the container, subsequent runs connect to the existing container instead of starting a new one. This reduces test startup from 30s to <1s. Enable with: `testcontainers.reuse.enable=true` in `~/.testcontainers.properties` and `.withReuse(true)` on the container definition. The container is destroyed after a configurable idle timeout.

7. **What is Toxiproxy and how is it used with Testcontainers?**
   A: Toxiproxy is a network chaos engineering tool that sits between the application and a service to inject failures: latency, connection drops, packet loss, bandwidth limits. With Testcontainers, `ToxiproxyContainer` creates a proxy between the application and a database/message broker. Tests can cut connections, add latency, or corrupt data to verify resilience patterns (retries, circuit breakers, timeouts).

8. **What is the difference between `@Testcontainers` and `@Container`?**
   A: `@Testcontainers` (class-level) enables automatic lifecycle management: it starts all `@Container`-annotated fields before tests and stops them after. Without `@Testcontainers`, containers are not automatically started — you must call `container.start()` manually. Always use both: `@Testcontainers` on the class and `@Container` on each container field.

9. **How do you test database migrations with Testcontainers?**
   A: Use a Spring Boot test with a real database container, Flyway/Liquibase auto-configuration, and assertions against the actual schema:
   ```java
   @SpringBootTest @Testcontainers
   class MigrationTest {
       @Container static PostgreSQLContainer<?> db = new PostgreSQLContainer<>("postgres:15");
       @DynamicPropertySource static void props(DynamicPropertyRegistry r) { /* map to datasource */ }
       @Autowired DataSource dataSource;
       @Test void verifyMigrations() { /* check tables, indexes, constraints exist */ }
   }
   ```
   This catches: database-specific SQL syntax, type mapping differences, constraint ordering, and migration ordering issues before production deployment.

10. **How do you handle Testcontainers tests that are sensitive to Docker resource constraints in CI?**
    A: Set explicit container resource limits: `.withCreateContainerCmdModifier(cmd -> cmd.getHostConfig().withMemory(512 * 1024 * 1024L).withCpuCount(1))`. Increase startup timeout: `.withStartupTimeout(Duration.ofMinutes(5))`. Retry on failure: `.withStartupAttempts(3)`. Use Alpine-based images (smaller, faster to pull). Pin specific image tags (never `latest`). Use `@Testcontainers(disabledWithoutDocker = true)` to gracefully skip tests when Docker is unavailable.

---

## Developer Recommendations

- **Use Testcontainers over H2 for database integration tests** — H2 has subtle differences from production databases: different SQL dialect, different constraint behavior (unique constraints that fail in PostgreSQL may pass in H2), different data types, and no support for features like `JSONB`, `ARRAY`, `GIN` indexes. A migration that works against H2 but fails on PostgreSQL is a production outage waiting to happen. Testcontainers gives you the real database in a disposable container.

- **Use `static @Container` and container reuse for fast local development** — Starting a container for each test method adds 30s per test — 20 tests = 10 minutes. Use `static @Container` (starts once per class) and `withReuse(true)` (stays running across test runs). Clean data between tests with `TRUNCATE` or `DELETE` rather than restarting the container. First run takes 30s; subsequent runs take <1s.

- **Use `@DynamicPropertySource` over hardcoded connection strings** — Containers use dynamically allocated ports. Hardcoding `jdbc:postgresql://localhost:5432/test` will fail when the container uses a different port. `@DynamicPropertySource` injects the actual container URL into Spring's Environment at runtime. The method must be `static void` with `DynamicPropertyRegistry` parameter.

- **Pin specific Docker image tags, never use `latest`** — `postgres:latest` changes when PostgreSQL releases a new version, potentially breaking your tests without code changes. Always use a specific tag: `postgres:15`, `postgres:16`. Match the production database version exactly. For reproducible builds, use digest references: `postgres@sha256:abc123...`. Update the tag intentionally when upgrading production.

- **Use `@Tag("integration")` to separate fast unit tests from slow container tests** — Tag all Testcontainers-based tests with `@Tag("integration")`. Configure Maven Surefire to exclude them locally (`mvn test` runs unit tests only, <1s) and Maven Failsafe to include them in CI (`mvn verify` runs all tests). This keeps the local dev loop fast while still running comprehensive checks in CI.

- **Use `Awaitility` for async assertions with Testcontainers** — `Thread.sleep()` is flaky (too short = intermittent failure, too long = slow tests). Awaitility polls until the condition is met or the timeout expires: `await().atMost(10, SECONDS).untilAsserted(() -> assertThat(queue).hasSize(1))`. Use for Kafka consumers, async event handlers, and database replication lag. The default poll interval is 100ms — increase for slow operations.

- **Use `Toxiproxy` for resilience testing** — Don't just test the happy path. Use Toxiproxy to simulate network failures: database timeouts, connection resets, high latency. Verify that circuit breakers open, retries happen with correct backoff, and fallback responses are returned. A service that crashes when the database is slow is not production-ready. Toxiproxy tests catch these issues before deployment.
