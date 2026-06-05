# JUnit 5

---

## Overview

- **Definition:** JUnit 5 is the latest version of the Java testing framework. It consists of three modules:
  - **JUnit Platform** — launches tests on JVM, integrates with IDEs and build tools
  - **JUnit Jupiter** — the programming model (annotations, assertions, extensions)
  - **JUnit Vintage** — backward compatibility with JUnit 4 tests

- **Why JUnit 5?**
  - More flexible extension model (extensions replace runners and rules)
  - Parameterized tests built-in
  - Nested test classes for better organization
  - Lambda-friendly assertions with better failure messages
  - Java 8+ features (streams, Optionals, lambdas)

---

## Annotations

| Annotation | Purpose |
|-----------|---------|
| `@Test` | Marks a test method |
| `@ParameterizedTest` | Test with different arguments |
| `@RepeatedTest` | Repeat test N times |
| `@BeforeAll` | Static method, runs once before all tests |
| `@AfterAll` | Static method, runs once after all tests |
| `@BeforeEach` | Instance method, runs before each test |
| `@AfterEach` | Instance method, runs after each test |
| `@DisplayName` | Custom test name (spaces, emojis) |
| `@Disabled` | Skip test with reason |
| `@Tag` | Filter tests by tag (e.g., "slow", "integration") |
| `@Nested` | Inner test class for grouping |
| `@Timeout` | Fail test if exceeds duration |

---

## Assertions

- **Definition:** JUnit Jupiter provides assertions in `org.junit.jupiter.api.Assertions`. All assertions support a `String message` parameter for custom failure messages.

```java
import static org.junit.jupiter.api.Assertions.*;

@Test
void standardAssertions() {
    assertEquals(4, calculator.add(2, 2));
    assertNotEquals(5, calculator.add(2, 2));
    assertTrue(result.isValid());
    assertFalse(result.hasErrors());
    assertNull(nullValue);
    assertNotNull(nonNullValue);
    assertThrows(IllegalArgumentException.class,
        () -> calculator.divide(1, 0));
    assertDoesNotThrow(() -> calculator.divide(1, 2));
}

@Test
void groupedAssertions() {
    assertAll("address",
        () -> assertEquals("John", address.getFirstName()),
        () -> assertEquals("Doe", address.getLastName()),
        () -> assertNotNull(address.getZipCode())
    );
}
// assertAll runs ALL assertions and reports all failures at once
```

---

## Test Lifecycle

```java
class OrderServiceTest {

    private OrderService orderService;

    @BeforeAll
    static void setupDatabase() {
        // Initialize test database — runs once
    }

    @BeforeEach
    void setUp() {
        orderService = new OrderService();  // Fresh instance before each test
    }

    @Test
    @DisplayName("Should create order with valid request")
    void shouldCreateOrder() {
        CreateOrderRequest request = new CreateOrderRequest("test@test.com", List.of("item-1"));
        Order order = orderService.createOrder(request);
        assertNotNull(order.getId());
    }

    @AfterEach
    void tearDown() {
        orderService = null;  // Clean up after each test
    }

    @AfterAll
    static void cleanupDatabase() {
        // Clean up test database — runs once
    }
}
```

- **Lifecycle Order:** `@BeforeAll` → `@BeforeEach` → `@Test` → `@AfterEach` → (repeat for next test) → `@AfterAll`

---

## Parameterized Tests

- **Definition:** Run the same test logic with different inputs, reducing repetitive test code.

```java
@ParameterizedTest
@ValueSource(strings = {"racecar", "radar", "level", "refer"})
void palindromes(String word) {
    assertTrue(StringUtils.isPalindrome(word));
}

@ParameterizedTest
@CsvSource({
    "1, 1, 2",
    "2, 3, 5",
    "10, 20, 30"
})
void addition(int a, int b, int expected) {
    assertEquals(expected, calculator.add(a, b));
}

@ParameterizedTest
@MethodSource("provideOrders")
void testOrderProcessing(Order order, boolean expected) {
    assertEquals(expected, orderService.process(order));
}

static Stream<Arguments> provideOrders() {
    return Stream.of(
        Arguments.of(new Order(100, "ACTIVE"), true),
        Arguments.of(new Order(0, "INACTIVE"), false)
    );
}
```

- **Other Sources:** `@EnumSource`, `@CsvFileSource`, `@ArgumentsSource`

---

## Common Mistakes

- **Not using `assertAll` for multiple assertions** — first failure skips remaining checks. Use `assertAll` to see all failures at once.
- **Non-static `@BeforeAll`/`@AfterAll`** — must be static unless test instance lifecycle is `PER_CLASS`.
- **Forgetting `@Test` annotation** — method won't run as a test.
- **Ignoring exception testing** — use `assertThrows()` instead of try-catch with `fail()`.
- **Tests depending on order** — tests should be independent. Use `@TestMethodOrder` only when absolutely necessary.
- **Hardcoded test data** — use `@CsvSource` or `@MethodSource` for cleaner parameterized tests.
- **Not using `@DisplayName`** — method names alone don't convey test intent as well as descriptive display names.

---

## Real-World Scenarios

### Scenario 1: Testing a Payment Service with Multiple States

A payment service processes transactions through states: PENDING → AUTHORIZED → CAPTURED (or FAILED). Each state transition has specific business rules, side effects (email notification, audit log, inventory deduction), and error states. The test suite must cover every transition.

```java
class PaymentServiceTest {
    @Nested
    class AuthorizationTests {
        @Test
        void shouldAuthorizeValidCard() {
            Payment payment = paymentService.authorize(new PaymentRequest("visa", "4111111111111111", 100.00));
            assertAll(
                () -> assertEquals(Status.AUTHORIZED, payment.getStatus()),
                () -> assertNotNull(payment.getAuthCode()),
                () -> verify(emailService).sendPaymentConfirmation(payment.getEmail())
            );
        }

        @Test
        void shouldFailForExpiredCard() {
            assertThrows(CardDeclinedException.class,
                () -> paymentService.authorize(new PaymentRequest("visa", "4000000000000002", 100.00)));
            verify(emailService, never()).sendPaymentConfirmation(any());
        }
    }
}
```

Using `@Nested` classes groups tests by state transition. `assertAll` reports all failures at once (identifying which field/behavior is wrong without rerunning). `verify(..., never())` ensures side effects don't happen on failure paths. This structure (arrange per transition, assert across state + side effects + return value) is the Arrange-Act-Assert pattern applied to complex state machines.

### Scenario 2: Parameterized Repository Tests

A data access layer must handle edge cases for different data types: nulls, empty strings, special characters, SQL injection attempts, and max-length values. Writing a separate test method for each case is repetitive.

```java
@ParameterizedTest
@MethodSource("provideUsers")
void shouldPersistAndRetrieveUser(User user) {
    userRepository.save(user);
    User found = userRepository.findById(user.getId()).orElseThrow();
    assertAll(
        () -> assertEquals(user.getName(), found.getName()),
        () -> assertEquals(user.getEmail(), found.getEmail()),
        () -> assertNotNull(found.getCreatedAt())
    );
}

static Stream<Arguments> provideUsers() {
    return Stream.of(
        Arguments.of(new User("normal", "user@example.com")),
        Arguments.of(new User("a", "short@example.com")),
        Arguments.of(new User("a".repeat(255), "long@example.com")),
        Arguments.of(new User("O'Brien", "special@example.com")),
        Arguments.of(new User(null, "null-name@example.com")),
        Arguments.of(new User("sql-inject", "x' OR '1'='1"))
    );
}
```

`@MethodSource` provides test data separately from test logic — adding a new edge case adds one line to the data stream. `assertAll` ensures all assertions run even if one fails, so you know exactly which field is wrong. This pattern reduces 20 repetitive test methods to one parameterized method with 6 data points.

### Scenario 3: Testing Async Event Processing

A microservice publishes domain events after processing orders. An async event handler sends emails and updates analytics. The test must verify these side effects happen within a reasonable time.

```java
@SpringBootTest
class OrderEventTest {
    @Autowired private OrderService orderService;
    @MockBean private EmailService emailService;
    @MockBean private AnalyticsService analyticsService;

    @Test
    void shouldPublishEventsOnOrderCreation() {
        CreateOrderRequest request = new CreateOrderRequest("test@test.com", List.of("item-1"));

        Order order = orderService.createOrder(request);

        await().atMost(5, TimeUnit.SECONDS)
            .untilAsserted(() -> {
                verify(emailService).sendOrderConfirmation(order.getId());
                verify(analyticsService).trackEvent("order.created", order.getId());
            });
    }
}
```

Awaitility's `await().untilAsserted()` polls the verification periodically until it passes or the timeout expires. This is more reliable than `Thread.sleep()` which is flaky (too short = intermittent failures, too long = slow tests). The `@MockBean` replaces real beans with mocks so side-effect verification is fast and isolated. For true async testing, combine `CompletableFuture` with `await()`.

---

## Scenario-Based Questions

1. **Q: You are migrating a legacy application from JUnit 4 to JUnit 5. The team has 2000 tests, custom JUnit 4 Rules (`@Rule`), and `@RunWith(Parameterized.class)`. The migration must be incremental — both frameworks must coexist. The team is concerned about regressions. How do you plan the migration?**
   A: Use `junit-vintage-engine` to run existing JUnit 4 tests unchanged while writing new tests in JUnit 5:
   ```xml
   <dependency>
       <groupId>org.junit.vintage</groupId>
       <artifactId>junit-vintage-engine</artifactId>
       <scope>test</scope>
   </dependency>
   ```
   Migration steps: (1) Add the vintage engine — all 2000 tests continue passing. (2) Replace `@Rule TemporaryFolder` with `@TempDir` (built-in JUnit 5). (3) Replace `ExpectedException` with `assertThrows()`. (4) Replace `@RunWith(Parameterized.class)` with `@ParameterizedTest`. (5) Replace custom Rules with JUnit 5 Extensions. (6) Migrate in priority order: test classes with most failures first, least changed modules first. Run both engines in CI — the build should pass the entire suite. Remove the vintage engine only when zero JUnit 4 imports remain.

2. **Q: A team has flaky tests that fail 1 in 20 runs. The failures are inconsistent — different tests fail on different runs. The team starts ignoring test failures. How do you systematically find and fix flaky tests?**
   A: Use `@RepeatedTest` to reproduce flakiness, then analyze root cause:
   ```java
   @RepeatedTest(100) // run 100 times to reproduce flaky failure
   void testConcurrentAccess() {
       // run the test
   }
   ```
   Root causes by frequency: (1) Shared mutable state between tests (40%) — fix with `@BeforeEach` reset. (2) Time-dependent tests (25%) — use `Clock` injection, not `System.currentTimeMillis()`. (3) Async without proper waiting (20%) — use Awaitility instead of `Thread.sleep()`. (4) Environment assumptions (10%) — use `@DisabledIfEnvironmentVariable` or `@TempDir`. (5) Test ordering dependencies (5%) — use `@TestMethodOrder(MethodName)` to make deterministic. Track flaky tests in a dedicated CI job that runs tests 5x and reports non-deterministic failures.

3. **Q: A test suite has 2000 tests. Running them sequentially takes 45 minutes. You want to run them in parallel on 4 cores. Some tests share a database and cannot run concurrently. How do you parallelize safely?**
   A: Use JUnit 5's parallel execution with resource locks:
   ```java
   // Enable in junit-platform.properties:
   junit.jupiter.execution.parallel.enabled=true
   junit.jupiter.execution.parallel.config.strategy=fixed
   junit.jupiter.execution.parallel.config.fixed.parallelism=4

   // Isolate tests that share resources:
   class DatabaseTest {
       @ResourceLock("database")
       @Test void test1() { /* exclusive DB access */ }

       @ResourceLock("database")
       @Test void test2() { /* exclusive DB access */ }
   }

   // Tests with different resource locks run in parallel:
   class UnitTest {
       @Test void testA() { /* no lock — parallel with DB tests */ }
       @Test void testB() { /* no lock — parallel with DB tests */ }
   }
   ```
   Test classes without `@ResourceLock` run in parallel. Tests with the same `@ResourceLock` are serialized. For database tests, use `@ResourceLock("database")` to serialize database access while unit tests run freely. Expected improvement: 45 min → 15-20 min (limited by the serialized database tests). For further improvement, use Testcontainers with separate schemas.

4. **Q: A developer writes `assertEquals(42, compute())` without a message. The test fails in CI and the developer can't tell which assertion failed or what the actual value was. How do you enforce descriptive assertion messages across the team?**
   A: Use a custom assertion wrapper with descriptive messages:
   ```java
   public static void assertPositiveBalance(BigDecimal balance) {
       assertTrue(balance.compareTo(BigDecimal.ZERO) >= 0,
           () -> "Expected non-negative balance but got " + balance);
   }

   public static <T> void assertNotEmpty(Optional<T> value, String description) {
       assertTrue(value.isPresent(),
           () -> "Expected " + description + " to be present but was empty");
   }
   ```
   Better: use AssertJ for fluent assertions with automatic descriptions:
   ```java
   import static org.assertj.core.api.Assertions.*;

   @Test
   void testOrder() {
       assertThat(order.getTotal()).isEqualByComparingTo(BigDecimal.valueOf(100));
       assertThat(order.getItems()).hasSize(3);
       assertThat(order.getStatus()).isIn(Status.PENDING, Status.ACTIVE);
   }
   ```
   AssertJ generates descriptive failure messages automatically: "Expected status to be one of [PENDING, ACTIVE] but was CANCELLED". The lambda supplier in `assertTrue(condition, () -> message)` ensures the message is only constructed on failure (no performance cost for passing tests). Add a static analysis rule (ErrorProne, SpotBugs) to flag bare `assertEquals` without message.

5. **Q: A microservice depends on 4 external APIs (payment gateway, shipping, email, fraud detection). Unit tests mock these. Integration tests call real APIs. The integration tests are slow (30s per test) and flaky (network issues). How do you design a test strategy that balances speed, confidence, and reliability?**
   A: Use the test pyramid with 3 layers:
   ```java
   // Layer 1: Unit tests (fast, 1000 tests, <1s) — mock everything
   @ExtendWith(MockitoExtension.class)
   class OrderServiceTest {
       @Mock PaymentGateway paymentGateway;
       @Mock ShippingService shippingService;
       @InjectMocks OrderService orderService;
       @Test void shouldProcessOrder() { /* ... */ }
   }

   // Layer 2: Contract tests (medium, 50 tests, 2s) — verify HTTP contracts
   @SpringBootTest(webEnvironment = MockWebEnvironment)
   class OrderControllerTest {
       @Autowired private MockMvc mockMvc;
       // Test JSON serialization, validation, HTTP status codes
   }

   // Layer 3: Integration tests (slow, 10 tests, 5min) — use Testcontainers
   @Testcontainers
   @SpringBootTest
   class OrderIntegrationTest {
       @Container static PostgreSQLContainer<?> db = new PostgreSQLContainer<>("postgres:15");
       // Test database interactions with real PostgreSQL
   }
   ```
   Unit tests run on every commit (1s feedback). Contract tests run on every PR (2s). Integration tests run nightly or before deployment (5min). External API calls are mocked at unit level and tested via wiremock at contract level. Real API integration is minimal — tested once per release.

6. **Q: A large test class has 50 test methods that share complex setup via `@BeforeEach`. The setup takes 3 seconds (creating files, populating database). Developers start combining multiple assertions into single test methods to reduce setup overhead. The tests become harder to debug. How do you balance setup cost with test isolation?**
   A: Use `@TestInstance(Lifecycle.PER_CLASS)` with `@BeforeAll` for the expensive setup and `@BeforeEach` for state reset:
   ```java
   @TestInstance(Lifecycle.PER_CLASS)
   class FileProcessingTest {
       private Path tempDir;
       private List<Record> testData;

       @BeforeAll
       void createExpensiveResources() throws IOException {
           tempDir = Files.createTempDirectory("test-");
           testData = generateTestData(1000); // done once
       }

       @BeforeEach
       void setUp() throws IOException {
           // Copy clean state from template — faster than recreating
           Files.copy(Path.of("template.db"), tempDir.resolve("test.db"), REPLACE_EXISTING);
       }

       @AfterAll
       void cleanup() {
           FileUtils.deleteDirectory(tempDir.toFile());
       }

       // 50 tests here
   }
   ```
   The expensive setup (generating data, creating directories) runs once. `@BeforeEach` copies a clean state from a template — much faster than recreating from scratch. This reduces test time from 3s × 50 = 150s to 3s (setup) + 0.1s × 50 (reset) = 8s. For shared mutable state, use `PER_CLASS` with explicit reset rather than `PER_METHOD` (the default).

7. **Q: A test verifies that an email is sent when an order is placed. The email service is mocked. The test passes. In production, the email template engine throws a NullPointerException because a template variable is missing. The mock didn't exercise the template rendering. How do you catch this class of bug?**
   A: This is a mock blind spot — the mock replaces the entire email service, including template rendering:
   ```java
   // Unit test catches service logic but not template rendering
   @Test
   void shouldSendConfirmationEmail() {
       orderService.createOrder(request);
       verify(emailService).sendConfirmation(any(Email.class));
   }

   // Integration test catches template rendering with real engine
   @SpringBootTest
   @Testcontainers
   class EmailIntegrationTest {
       @Autowired private EmailService emailService; // real, not mocked
       @Autowired private JavaMailSender mailSender; // real SMTP via GreenMail

       @Test
       void shouldRenderConfirmationEmail() {
           MimeMessage message = emailService.createConfirmationEmail(order);
           assertNotNull(message.getContent()); // would throw if template fails
           assertTrue(message.getContent().toString().contains(order.getCustomerName()));
       }
   }
   ```
   Use GreenMail (in-memory SMTP) or Mailhog for testing real email rendering. The integration test keeps the real `EmailService` with its template engine but uses an in-memory SMTP server instead of a real one. This catches template errors, encoding issues, and missing variables. The rule: mock I/O boundaries (SMTP, HTTP), not business logic (template rendering).

8. **Q: You have 500 parameterized test cases in a `@MethodSource`. When one case fails, JUnit reports the failure but doesn't tell you which input data caused it. Debugging requires manually matching the failure to the input. How do you make parameterized test failures self-diagnosing?**
   A: Use `@DisplayName` with the method source or include the input in the test name:
   ```java
   @ParameterizedTest(name = "[{index}] email={0}, expected={1}")
   @MethodSource("emailValidationData")
   void testEmailValidation(String email, boolean expected) {
       assertEquals(expected, validator.isValid(email));
   }

   // Also: include context in the assertion message
   static Stream<Arguments> emailValidationData() {
       return Stream.of(
           Arguments.of(Named.of("valid standard email", "user@example.com"), true),
           Arguments.of(Named.of("missing @ symbol", "invalid-email"), false)
       );
   }
   ```
   The `name` attribute in `@ParameterizedTest` controls the test name in reports: `testEmailValidation[1] email=user@example.com, expected=true`. The `Named.of()` approach adds descriptions to each argument. For CI reports, this makes failures immediately identifiable: "testEmailValidation[3] email=null-test, expected=false FAILED" tells you exactly which input failed.

9. **Q: A service method calls `repository.save()` which returns the saved entity with a generated ID. The test needs to verify the returned entity has a non-null ID. The repository is mocked. Mockito's `when(repo.save(any())).thenReturn(order)` returns the input order which has a null ID because the test set it up that way. How do you test ID generation with mocks?**
   A: Use `thenAnswer()` to simulate ID generation:
   ```java
   @Test
   void shouldGenerateIdOnSave() {
       Order inputOrder = new Order(null, "test@test.com", 100.00);

       when(orderRepository.save(any())).thenAnswer(invocation -> {
           Order order = invocation.getArgument(0);
           order.setId(1L); // simulate DB-generated ID
           return order;
       });

       Order result = orderService.createOrder(inputOrder);

       assertNotNull(result.getId());
       assertEquals(1L, result.getId());
   }
   ```
   `thenAnswer()` captures the input and modifies it — simulating what a database would do. This is more realistic than `thenReturn(order)` which returns the exact same object without the generated ID. For more complex scenarios, use `ArgumentCaptor` to capture and assert intermediate state. The key: mock the behavior, not just the return value.

10. **Q: A test uses `@TempDir` for file-based tests. The test creates files, processes them, and cleans up. Occasionally, the cleanup fails on Windows because files are locked by antivirus or search indexing. The `@AfterEach` cleanup throws an exception, masking the real test failure. How do you handle file cleanup robustly?**
    A: Use `@TempDir` which automatically cleans up after the test class (not per test) and retries on failure:
    ```java
    class FileProcessingTest {
        @TempDir
        Path tempDir; // JUnit manages cleanup

        @Test
        void shouldProcessCsvFile() throws IOException {
            Path csvFile = tempDir.resolve("test.csv");
            Files.writeString(csvFile, "id,name\n1,Alice");
            List<Record> records = csvProcessor.process(csvFile);
            assertEquals(1, records.size());
        }
        // No cleanup needed — @TempDir handles it
    }
    ```
    JUnit 5.9+ `@TempDir` retries cleanup on failure (configurable with `@TempDir(retry = true)`). If cleanup fails, it logs a warning but doesn't fail the test. For manual cleanup, use `Files.deleteIfExists()` (no exception if file doesn't exist) instead of `Files.delete()` (throws if missing). For Windows specifically, add `System.gc()` before cleanup to release any lingering file handles from the test process.

---

## Interview Questions

1. **What is JUnit 5 and what are its three modules?**
   A: JUnit 5 is the latest Java testing framework with three modules: JUnit Platform (launches tests, integrates with IDEs/build tools), JUnit Jupiter (the programming model with annotations, assertions, extensions), and JUnit Vintage (backward compatibility for JUnit 4 tests). This modular design allows running JUnit 4 and 5 tests side by side.

2. **What is the difference between `@BeforeEach` and `@BeforeAll`?**
   A: `@BeforeEach` runs before each test method — used for per-test setup (fresh objects, reset state). `@BeforeAll` runs once before all tests — used for expensive setup (database connections, file systems). `@BeforeAll` must be `static` unless `@TestInstance(Lifecycle.PER_CLASS)` is used. The lifecycle order: `@BeforeAll` → `@BeforeEach` → `@Test` → `@AfterEach` → (repeat) → `@AfterAll`.

3. **What is `assertAll` and why would you use it?**
   A: `assertAll` executes all supplied assertions and reports all failures together, even if some fail. Without it, the first assertion failure stops execution — you fix that, rerun, find the next failure, fix, rerun (N cycles for N failures). `assertAll` groups related assertions (like all fields of an object), so a single test run shows all failures at once. It accepts `Executable` lambdas.

4. **What is the difference between `@ParameterizedTest` and `@RepeatedTest`?**
   A: `@ParameterizedTest` runs the same test logic with different inputs (from `@ValueSource`, `@CsvSource`, `@MethodSource`, `@CsvFileSource`). `@RepeatedTest` runs the same test N times with the same inputs — used for stress testing, flaky test reproduction, or verifying non-deterministic behavior. Use parameterized tests for data-driven testing; use repeated tests for reliability testing.

5. **How do you handle test timeouts in JUnit 5?**
   A: Use `@Timeout` annotation on test methods or at class level. `assertTimeout()` runs the assertion in the calling thread; `assertTimeoutPreemptively()` runs it in a separate thread and aborts on timeout. Use `@Timeout` for simple deadline enforcement; use `assertTimeoutPreemptively()` for operations that may hang indefinitely. Timeouts should be large enough to avoid flakiness from GC pauses or system load.

6. **What are `@Nested` test classes and when should you use them?**
   A: `@Nested` classes group related test methods within a test class. They must be non-static inner classes. Each nested class can have its own `@BeforeEach`/`@AfterEach` methods, which run in addition to the outer class's lifecycle methods. Use for: grouping tests by feature (create order, get order, cancel order), separating setup for different scenarios, and improving test report readability.

7. **What is the difference between `@Tag` and `@Disabled`?**
   A: `@Tag("slow")` categorizes tests for filtering — you can include/exclude tags in build tools (Maven Surefire, Gradle). `@Disabled("reason")` permanently skips a test (like `@Ignore` in JUnit 4). Use `@Tag` for selective execution (unit vs integration, fast vs slow). Use `@Disabled` for temporarily broken tests that need fixing.

8. **How do you test exception scenarios in JUnit 5?**
   A: Use `assertThrows(ExpectedException.class, () -> codeThatThrows())`. It returns the exception for further assertions on message, cause, or state:
   ```java
   ValidationException ex = assertThrows(ValidationException.class,
       () -> validator.validate(null));
   assertEquals("Input must not be null", ex.getMessage());
   ```
   Use `assertDoesNotThrow(() -> safeCode())` for the positive case. Avoid try-catch with `fail()` — `assertThrows` is more concise and produces better failure messages.

9. **What is `@TestMethodOrder` and when would you use it?**
   A: `@TestMethodOrder(MethodName.class)` or `@TestMethodOrder(OrderAnnotation.class)` controls test execution order. Tests should ideally be independent (order shouldn't matter), but ordered execution is useful for: shared expensive setup (first test creates resources, subsequent tests use them), integration tests with sequential dependencies, and reproducing CI-specific order issues. Default in JUnit 5 is a deterministic but unspecified order.

10. **How do you integrate JUnit 5 with Spring Boot?**
    A: Use `@SpringBootTest` for full application context, `@WebMvcTest` for web layer only, `@DataJpaTest` for JPA/repository tests, and `@ExtendWith(SpringExtension.class)` for non-Spring-Boot Spring tests. Use `@MockBean` and `@SpyBean` to replace beans with mocks in the Spring context. `@DynamicPropertySource` injects Testcontainers connection details into the Spring Environment.

---

## Developer Recommendations

- **Prefer `assertAll` over multiple sequential assertions** — Sequential assertions fail fast: the first failure stops execution, hiding subsequent failures. You fix, rerun, find the next failure, repeat. `assertAll` executes all assertions and reports every failure at once. Use it to group assertions for a single logical check (e.g., all fields of a returned object). This reduces fix-rerun cycles from N to 1.

- **Use `@ParameterizedTest` with `@MethodSource` over repetitive test methods** — A dozen test methods with different inputs is harder to maintain and extend than one parameterized test with a data stream. `@MethodSource` separates test data from test logic — adding a new edge case adds one line to the data stream, not a new method. Use `@CsvSource` for simple cases and `@MethodSource` for complex test data.

- **Use `@Nested` classes to organize large test classes** — A test class with 50+ test methods is hard to navigate. Group related tests with `@Nested` inner classes: `@Nested class CreationTests { ... }`, `@Nested class ValidationTests { ... }`. Each group gets its own `@BeforeEach` and `@DisplayName`. This makes test reports readable and helps developers find relevant tests quickly.

- **Use `@TempDir` instead of manual temporary file management** — Manual temp file creation in `@BeforeEach` + cleanup in `@AfterEach` is error-prone (forgotten cleanup, Windows file locks). `@TempDir` creates a temp directory per test method and automatically retries cleanup on failure. It works across platforms and handles edge cases like antivirus locks on Windows.

- **Use `@Tag` to separate fast unit tests from slow integration tests** — Tag integration tests with `@Tag("integration")`. Configure Maven Surefire to exclude slow tags for local development (`mvn test` runs only fast tests) and include all tags in CI (`mvn verify` runs everything). This keeps the local dev loop fast (<1s for unit tests) while still running comprehensive checks in CI.

- **Use AssertJ over JUnit assertions for richer failure messages** — `assertEquals` tells you expected vs actual. AssertJ's `assertThat(actual).isEqualTo(expected).hasSize(3).isIn(status1, status2)` generates descriptive failure messages automatically: "Expected status to be one of [PENDING, ACTIVE] but was CANCELLED". The fluent API also prevents argument order errors (`assertEquals(expected, actual)` vs `assertEquals(actual, expected)`).

- **Avoid shared mutable state between tests** — Tests should be independent and runnable in any order. Shared state (static fields, files, databases) causes order-dependent failures that are hard to debug. Reset state in `@BeforeEach`. Use `@TempDir` for files, `@DirtiesContext` for Spring context isolation, and fresh test data per test instead of shared fixtures.
