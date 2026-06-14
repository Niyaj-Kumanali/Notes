# JUnit 5

---

## Overview

- **Definition**
  - JUnit 5 is the latest version of the Java testing framework, designed as a modular platform of three cooperating modules.
  - **JUnit Platform** — the foundation that launches tests on the JVM and integrates with IDEs (IntelliJ, Eclipse), build tools (Maven Surefire, Gradle), and CI systems.
  - **JUnit Jupiter** — the programming model containing all the annotations, assertion methods, and extension APIs that developers use to write tests.
  - **JUnit Vintage** — provides backward compatibility for running JUnit 4 and even JUnit 3 tests unchanged under the JUnit 5 platform.

- **Motivation**
  - JUnit 5 was motivated by limitations in JUnit 4, including a rigid runner-based extension model, poor support for parameterized tests without third-party libraries, and the inability to run multiple runners in a single test class.
  - The new architecture uses extensions (which replace both runners and rules from JUnit 4) with a more composable model, provides built-in parameterized tests with rich data source annotations, supports nested test classes for hierarchical organization, and leverages Java 8+ features like lambdas in assertions for lazy message evaluation and streams for test data generation.

---

## Annotations

- **Overview**
  - JUnit 5 provides a comprehensive set of annotations for structuring tests.

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

- **Detail**
  - `@Test` — marks a method as a test case.
  - `@ParameterizedTest` — runs the same test logic with different arguments from sources like `@ValueSource`, `@CsvSource`, `@MethodSource`, and `@EnumSource`.
  - `@RepeatedTest` — runs a test a specified number of times for stress testing.
  - `@BeforeAll` / `@AfterAll` — run once before and after all tests in a class and must be static unless `@TestInstance(Lifecycle.PER_CLASS)` is used.
  - `@BeforeEach` / `@AfterEach` — run before and after each individual test method, providing per-test setup and cleanup.
  - `@DisplayName` — customizes the test name in reports and IDE output with spaces and even emojis.
  - `@Disabled` — skips a test with an optional reason string.
  - `@Tag` — categorizes tests for selective execution in build tool configurations.
  - `@Nested` — allows grouping related tests within inner classes for better organization.
  - `@Timeout` — causes a test to fail if it exceeds the specified duration.

---

## Assertions

- **Overview**
  - JUnit Jupiter provides assertion methods in `org.junit.jupiter.api.Assertions`, all of which accept an optional `String message` parameter or a `Supplier<String>` for lazy message evaluation (the message is only constructed on failure).

- **Key Assertions**
  - `assertEquals()` / `assertNotEquals()` — compare expected and actual values using `equals()`.
  - `assertTrue()` / `assertFalse()` — check boolean conditions.
  - `assertNull()` / `assertNotNull()` — check for null references.
  - `assertThrows()` — verifies that a code block throws a specific exception and returns the exception for further assertions on its message or cause.
  - `assertDoesNotThrow()` — verifies that a code block completes without any exception.
  - `assertAll()` — the most powerful assertion: executes multiple assertions as lambdas and reports every failure at once, rather than failing fast on the first one.
  - `assertTimeout()` / `assertTimeoutPreemptively()` — verify that a code block completes within a given duration, with the preemptive version running in a separate thread.

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
```

---

## Test Lifecycle

- **Execution Order**
  - The JUnit 5 test lifecycle follows a well-defined order:
    - `@BeforeAll` runs once at the start of the test class.
    - For each test method: `@BeforeEach` runs (creating fresh state), the `@Test` method executes, and `@AfterEach` runs (cleaning up state).
    - After all tests complete, `@AfterAll` runs once for final cleanup.

- **Instance Management**
  - By default, each test method creates a new instance of the test class, ensuring tests are isolated.
  - The `@TestInstance(Lifecycle.PER_CLASS)` annotation changes this behavior so the same instance is reused for all tests, which allows `@BeforeAll` and `@AfterAll` to be non-static but requires careful management of shared state in `@BeforeEach` to prevent test-order dependencies.

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

---

## Parameterized Tests

- **Definition**
  - Parameterized tests run the same test logic with different inputs, eliminating repetitive test methods where only the input data changes.

- **Data Sources**
  - `@ValueSource` — provides an array of literal values (strings, ints, longs, doubles) for simple cases.
  - `@CsvSource` — supplies argument tuples as comma-separated values with automatic type conversion.
  - `@MethodSource` — references a static or instance method that returns `Stream<Arguments>`, `Arguments[]`, or any collection of argument tuples — the most flexible source for complex test data.
  - `@CsvFileSource` — loads test data from a CSV file on the classpath, ideal for large datasets maintained separately from the test code.
  - `@EnumSource` — provides all or a subset of an enum's constants.

- **Naming**
  - The `@ParameterizedTest` annotation includes a `name` attribute for readable test names in reports, such as `name = "[{index}] input={0}, expected={1}"`.

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

---

## Common Mistakes

- **Not using `assertAll()`**
  - Multiple assertions on the same logical object causes unnecessary fix-rerun cycles.
  - Each individual `assertEquals` is standard JUnit usage, and the first failure throws an exception — the developer fixes it, reruns, and never realizes that other assertions also failed.
  - Without `assertAll`, the first assertion failure throws an exception, skipping the remaining assertions. You fix the first failure, rerun, find the second failure, fix, rerun — N cycles for N failures.
  - `assertAll()` executes all assertions and reports every failure at once, so you fix everything in one pass.

- **Declaring `@BeforeAll` or `@AfterAll` as non-static**
  - Without `@TestInstance(Lifecycle.PER_CLASS)` causes a `JUnitException` at runtime.
  - The method compiles and the IDE may not flag it — the error only surfaces when the test class actually runs.

- **Using `@Test` instead of `@ParameterizedTest`**
  - Causes only the first argument set to run, silently ignoring the rest.
  - The first test case passes, giving the impression that the test is working — the remaining cases are never executed.

- **Using `Thread.sleep()` for async testing**
  - Instead of Awaitility introduces flakiness.
  - `Thread.sleep(3000)` is a straightforward approach and usually works on a developer's fast machine — the flakiness only appears on slow CI runners where the async operation occasionally takes 3.1 seconds.

---

## Real-World Scenarios

### Scenario 1: Testing a Payment Service with Multiple States

- **Context**
  - A payment service processes transactions through a state machine: PENDING → AUTHORIZED → CAPTURED (or FAILED).
  - Each state transition has specific business rules, side effects (email notification, audit log entry, inventory deduction), and error states.
  - The test suite must cover every valid transition, every error case, and verify that side effects occur or are suppressed correctly.

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

- **What makes this work**
  - Using `@Nested` classes groups tests by state transition, making the test report read like a specification.
  - `assertAll()` reports all field-level and side-effect failures at once — if the status is wrong AND the auth code is null AND the email wasn't sent, all three failures appear in a single test run.
  - `verify(..., never())` explicitly asserts that side effects do not happen on failure paths, preventing the common bug where error paths accidentally trigger email, audit, or inventory deductions.

### Scenario 2: Parameterized Repository Tests

- **Context**
  - A data access layer must handle edge cases for different data types: null values, empty strings, special characters with apostrophes, SQL injection attempts, and maximum-length values.
  - Writing a separate test method for each edge case creates 20 repetitive methods that are hard to maintain and extend when new edge cases are discovered.

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

- **What makes this work**
  - `@MethodSource` cleanly separates test data from test logic — adding a new edge case (like a Unicode name or a 256-character name) adds exactly one line to the data stream with zero changes to the test method.
  - `assertAll()` ensures all field-level assertions run even if one fails, so a single run identifies whether the name mapping, email mapping, or timestamp generation is broken.
  - This pattern collapses 20 repetitive test methods into one parameterized method with 6 data points.

### Scenario 3: Testing Async Event Processing

- **Context**
  - A microservice publishes domain events after processing orders. An asynchronous event handler sends confirmation emails and updates analytics.
  - The test must verify that these side effects happen within a reasonable time without resorting to unreliable `Thread.sleep()` calls.

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

- **What makes this work**
  - Awaitility's `await().untilAsserted()` polls the verification lambda periodically (default 100ms intervals) until it passes or the 5-second timeout expires.
  - This is strictly more reliable than `Thread.sleep(3000)` which wastes 3 seconds even when the event completes in 50ms, and fails intermittently when the event takes 3.1 seconds on a slow CI machine.
  - The `@MockBean` annotations replace real Spring beans with Mockito mocks, so side-effect verification is fast, isolated, and does not require a real email server.

## Use Cases

- **Unit testing Java classes** — testing individual methods in isolation from external dependencies
  - `@Test` methods with JUnit assertions (`assertEquals`, `assertTrue`, `assertThrows`). `@DisplayName` for readable test names. `@ParameterizedTest` for data-driven tests.
  - **Avoid when:** the test requires Spring context — `@SpringBootTest` is better suited; `@ExtendWith(SpringExtension.class)` bridges JUnit 5 and Spring.

- **Integration testing with extensions** — testing code that needs database, file system, or network access
  - Custom `Extension` (replacing JUnit 4 `@Rule`) manages test lifecycle. Temporary directories via `@TempDir`. Database setup with `@BeforeEach`/`@AfterEach`.
  - **Avoid when:** the integration requires external infrastructure — combine with Testcontainers for a real database instance.

- **Parameterized testing** — running the same test with multiple input/output combinations
  - `@ValueSource`, `@CsvSource`, `@MethodSource` provide test data. Test runs once per combination. Reduces boilerplate compared to loop-based testing.
  - **Avoid when:** each test case has unique assertion logic — individual test methods are clearer when the expected behavior differs per input.

- **Migration from JUnit 4 to 5** — incrementally adopting JUnit 5 without rewriting existing tests
  - `junit-vintage-engine` allows JUnit 4 tests to run under the JUnit 5 platform. Migrate file-by-file. JUnit 5's `@Nested` and `@DisplayNameGeneration` improve test organization.
  - **Avoid when:** the project is new — start with JUnit 5 directly.

- **Conditional test execution** — skipping tests based on OS, Java version, or environment variables
  - `@EnabledOnOs`, `@DisabledOnJre`, `@EnabledIfEnvironmentVariable`. Tests are conditionally executed at runtime. Useful for platform-specific tests.
  - **Avoid when:** the condition makes tests invisible to reviewers — a comment explaining why a test is disabled is more transparent than a conditional annotation.

---

## Scenario-Based Questions

**Q: You are migrating a legacy application from JUnit 4 to JUnit 5. The team has 2000 tests, custom JUnit 4 Rules (`@Rule`), and `@RunWith(Parameterized.class)`. The migration must be incremental — both frameworks must coexist. How do you plan the migration?**

- **Solution**
  - Add the `junit-vintage-engine` dependency, which allows JUnit 4 tests to run under the JUnit 5 platform without any code changes. All 2000 existing tests continue passing on the same build.
  - Then migrate incrementally: replace `@Rule TemporaryFolder` with `@TempDir` (built-in JUnit 5, no additional library), replace `ExpectedException` rule with `assertThrows()`, replace `@RunWith(Parameterized.class)` with `@ParameterizedTest` and its data sources, and replace custom Rules with JUnit 5 Extensions using `@ExtendWith`.
  - Migrate in priority order: the most failure-prone test classes first (where better diagnostics have the highest ROI), and the least-coupled modules first (where changes are safest).
  - Run both engines in CI — the build must pass the entire suite at every step.
  - Remove the vintage engine dependency only when zero JUnit 4 imports remain in the source code.

---

**Q: A team has flaky tests that fail 1 in 20 runs. The failures are inconsistent — different tests fail on different runs. The team starts ignoring test failures. How do you systematically find and fix flaky tests?**

- **Solution**
  - Reproduce the flakiness by running the suspicious test 100 times with `@RepeatedTest(100)`, then analyze the root cause category.
  - Shared mutable state between tests accounts for roughly 40% of flaky tests — fix with `@BeforeEach` reset and immutable fixtures.
  - Time-dependent tests using `System.currentTimeMillis()` or `new Date()` account for 25% — fix by injecting a `Clock` that can be controlled in tests.
  - Asynchronous operations without proper synchronization account for 20% — fix by using Awaitility instead of `Thread.sleep()`.
  - Environment-dependent tests that assume specific OS, locale, or timezone account for 10% — fix with `@DisabledIfEnvironmentVariable` or `@TempDir`.
  - Test ordering dependencies account for 5% — fix with `@TestMethodOrder(MethodName)` to make order deterministic.
  - Track flaky tests in a dedicated CI job that runs each test 5 times and reports non-deterministic failures separately from deterministic ones.

---

**Q: A test suite has 2000 tests. Running them sequentially takes 45 minutes. Some tests share a database and cannot run concurrently. How do you parallelize safely?**

- **Solution**
  - Enable JUnit 5's parallel execution and use `@ResourceLock` to serialize tests that share the same external resource:

```java
// Enable in junit-platform.properties:
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.config.strategy=fixed
junit.jupiter.execution.parallel.config.fixed.parallelism=4

// Isolate tests that share resources:
@ResourceLock("database")
class DatabaseTest {
    @Test void test1() { /* exclusive DB access */ }
    @Test void test2() { /* exclusive DB access */ }
}

// Tests with different resource locks run in parallel:
class UnitTest {
    @Test void testA() { /* no lock — parallel with DB tests */ }
    @Test void testB() { /* no lock — parallel with DB tests */ }
}
```

  - Test classes without `@ResourceLock` run in full parallel. Tests with the same `@ResourceLock` value are serialized against each other but can run in parallel with tests that have different or no locks. The expected improvement is from 45 minutes to 15-20 minutes, with the serialized database tests being the bottleneck. For further improvement, use Testcontainers with separate database schemas per test class so concurrent tests each have their own isolated database.

> **Interview follow-up:** The candidate proposed using `@ResourceLock("database")` and `@ResourceLock("filesystem")` to isolate shared resources. A developer creates a new test class with 30 test methods that use the database, but forgets to add `@ResourceLock("database")`. During parallel execution, this class runs concurrently with other database tests, causing intermittent failures that are hard to reproduce. How would you enforce, through CI or static analysis, that every test class interacting with a shared resource is annotated with the correct `@ResourceLock`?

---

**Q: A developer writes `assertEquals(42, compute())` without a message. The test fails in CI and the developer can't tell which assertion failed or what the actual value was. How do you enforce descriptive assertion messages across the team?**

- **Solution**
  - Use AssertJ for fluent assertions that generate descriptive failure messages automatically without manual message strings. `assertThat(order.getTotal()).isEqualByComparingTo(BigDecimal.valueOf(100))` produces "Expected: BigDecimal<100> but was: BigDecimal<95>" without any developer-written message.
  - For teams that prefer JUnit assertions, create custom assertion wrappers and use the lambda supplier form `assertTrue(condition, () -> "message built on failure")` so the message construction has zero cost for passing tests.
  - Add a static analysis rule (ErrorProne or SpotBugs) that flags bare `assertEquals` and `assertTrue` calls without a message parameter, enforcing the team convention through automated code review.

> **Interview follow-up:** The candidate suggested AssertJ for descriptive messages. The team adopts AssertJ but a junior developer writes `assertThat(actual).isEqualTo(expected).isEqualTo(expected2)` thinking both conditions will be checked. In reality, `isEqualTo(expected2)` is compared against the result of `isEqualTo(expected)`, which returns `AbstractAssert` — the test passes even though `expected2` is wrong. How would you prevent this misuse through code review guidelines or static analysis?

---

**Q: A microservice depends on 4 external APIs (payment gateway, shipping, email, fraud detection). Unit tests mock these. Integration tests call real APIs. The integration tests are slow (30s per test) and flaky (network issues). How do you design a test strategy that balances speed, confidence, and reliability?**

- **Solution**
  - Use a three-layer test pyramid: unit tests (fastest, mock everything) on every commit, contract tests (medium, verify API contracts) on every PR, and integration tests (slowest, use Testcontainers) nightly or before deployment.
  - Unit tests with `@ExtendWith(MockitoExtension.class)` and `@Mock` all external services — they run in milliseconds and catch logic errors.
  - Contract tests use `@WebMvcTest` with `MockMvc` and WireMock to verify HTTP serialization, validation, and status codes without starting the full application.
  - Integration tests use `@Testcontainers` with real PostgreSQL and an in-memory SMTP server (GreenMail) — they run nightly and catch problems that mocks cannot simulate, such as SQL dialect differences, encoding issues, and transaction boundary bugs.

---

**Q: A large test class has 50 test methods that share complex setup via `@BeforeEach`. The setup takes 3 seconds. Developers start combining multiple assertions into single test methods to reduce setup overhead. The tests become harder to debug. How do you balance setup cost with test isolation?**

- **Solution**
  - Use `@TestInstance(Lifecycle.PER_CLASS)` to run the expensive setup once in `@BeforeAll`, and use `@BeforeEach` for a lightweight state reset from a template:

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

  - The expensive setup runs once, reducing total time from 3 seconds × 50 tests = 150 seconds to 3 seconds + 0.1 seconds × 50 = 8 seconds. The state reset in `@BeforeEach` (copying a template) is much cheaper than recreating the entire fixture.
  - For shared mutable state, `PER_CLASS` with explicit reset is the right approach — the key is ensuring that `@BeforeEach` fully restores a clean state so tests remain independent.

---

**Q: A test verifies that an email is sent when an order is placed. The email service is mocked. The test passes. In production, the email template engine throws a NullPointerException because a template variable is missing. The mock didn't exercise the template rendering. How do you catch this class of bug?**

- **Solution**
  - The mock replaced the entire `EmailService`, including the template rendering engine that contains the bug.
  - Add an integration test that uses the real `EmailService` with an in-memory SMTP server (GreenMail) to verify the actual template rendering:

```java
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

  - The rule is: mock I/O boundaries (the SMTP server), not business logic (the template engine). The integration test uses the real `EmailService` with its template engine but substitutes an in-memory SMTP server for the real one. This catches template syntax errors, encoding issues, and missing template variables that unit tests with mocks miss entirely.

---

**Q: You have 500 parameterized test cases in a `@MethodSource`. When one case fails, JUnit reports the failure but doesn't tell you which input data caused it. How do you make parameterized test failures self-diagnosing?**

- **Solution**
  - Use the `name` attribute of `@ParameterizedTest` to control how test names appear in reports, making failed inputs immediately identifiable:

```java
@ParameterizedTest(name = "[{index}] email={0}, expected={1}")
@MethodSource("emailValidationData")
void testEmailValidation(String email, boolean expected) {
    assertEquals(expected, validator.isValid(email));
}
```

  - The `name` attribute supports `{index}` (the invocation index), `{0}`, `{1}`, etc. (argument values). A failure in CI reports: `testEmailValidation[3] email=null-test, expected=false FAILED`.
  - For additional context, use `Named.of("description", argument)` in the `@MethodSource` to attach human-readable descriptions to arguments. This eliminates the need to manually match failure stack traces to test inputs.

---

**Q: A service method calls `repository.save()` which returns the saved entity with a generated ID. The repository is mocked. Mockito's `when(repo.save(any())).thenReturn(order)` returns the input order which has a null ID because the test set it up that way. How do you test ID generation with mocks?**

- **Solution**
  - Use `thenAnswer()` to capture the input entity and simulate ID generation within the answer:

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

  - `thenAnswer()` captures the `Order` object that was passed to `save()` and sets an ID on it before returning, simulating what the database would do. This is more realistic than `thenReturn(order)` which returns the exact same object without any generated fields.
  - For more complex scenarios, use `ArgumentCaptor` to capture and assert intermediate state before the return value is produced.

---

**Q: A test uses `@TempDir` for file-based tests. Occasionally, cleanup fails on Windows because files are locked by antivirus or search indexing. The `@AfterEach` cleanup throws an exception, masking the real test failure. How do you handle file cleanup robustly?**

- **Solution**
  - Use `@TempDir` which is managed by JUnit itself — it automatically deletes the temporary directory after all tests complete and retries on failure. JUnit 5.9+ supports `@TempDir(retry = true)` which retries cleanup if it fails.
  - If cleanup still fails after retries, JUnit logs a warning but does not fail the test, ensuring that cleanup failures never mask genuine assertion failures.
  - For manual file cleanup in tests, use `Files.deleteIfExists()` which does not throw an exception if the file is already deleted, and call `System.gc()` before cleanup on Windows to release any lingering file handles from the test process.

---

## Interview Questions

- **What is JUnit 5 and what are its three modules?**
  - JUnit 5 is the latest Java testing framework consisting of JUnit Platform (launches tests on the JVM, integrates with IDEs and build tools), JUnit Jupiter (the programming model with annotations, assertions, and extensions), and JUnit Vintage (backward compatibility for running JUnit 4 tests).
  - This modular design allows JUnit 4 and 5 tests to coexist in the same project.

- **What is the difference between `@BeforeEach` and `@BeforeAll`?**
  - `@BeforeEach` runs before each test method and is used for per-test setup like creating fresh objects and resetting state.
  - `@BeforeAll` runs once before all tests in the class and is used for expensive setup like database connections.
  - `@BeforeAll` must be `static` unless `@TestInstance(Lifecycle.PER_CLASS)` is used.
  - The lifecycle order: `@BeforeAll` → `@BeforeEach` → `@Test` → `@AfterEach` → repeat → `@AfterAll`.

- **What is `assertAll` and why would you use it?**
  - `assertAll` executes all supplied assertion lambdas and reports every failure at once, even if some assertions fail. Without it, the first assertion failure throws an exception and skips remaining checks.
  - Use `assertAll` to group related assertions (like all fields of an object) so a single test run shows all broken fields, eliminating the fix-rerun loop.

- **What is the difference between `@ParameterizedTest` and `@RepeatedTest`?**
  - `@ParameterizedTest` runs test logic with different inputs from data sources like `@ValueSource`, `@CsvSource`, `@MethodSource`, and `@EnumSource`.
  - `@RepeatedTest` runs the same test N times with identical inputs for stress testing or flaky test reproduction.
  - Parameterized tests are for data-driven testing; repeated tests are for reliability testing.

- **How do you handle test timeouts in JUnit 5?**
  - Use `@Timeout` on test methods or at class level.
  - `assertTimeout()` runs assertions in the calling thread and checks elapsed time after completion.
  - `assertTimeoutPreemptively()` runs assertions in a separate thread and aborts on timeout.
  - Use `@Timeout` for simple deadlines; use `assertTimeoutPreemptively()` for operations that may hang indefinitely.

- **What are `@Nested` test classes and when should you use them?**
  - `@Nested` classes are non-static inner classes that group related test methods.
  - Each nested class can have its own `@BeforeEach`/`@AfterEach` methods in addition to the outer class's lifecycle.
  - Use `@Nested` to organize tests by feature (creation, retrieval, deletion), to share setup within a group, and to produce readable test reports that read like specifications.

- **What is the difference between `@Tag` and `@Disabled`?**
  - `@Tag("slow")` categorizes tests for selective execution — you can include or exclude tags in Maven Surefire or Gradle configuration.
  - `@Disabled("reason")` permanently skips a test.
  - Use `@Tag` to separate fast unit tests from slow integration tests. Use `@Disabled` only for temporarily broken tests that need fixing.

- **How do you test exception scenarios in JUnit 5?**
  - Use `assertThrows(ExpectedException.class, () -> codeThatThrows())` which returns the exception for further assertions on its message, cause, or state.
  - Use `assertDoesNotThrow(() -> safeCode())` for the positive case.
  - Always prefer `assertThrows` over try-catch with `fail()` — it is more concise and produces better failure messages.

- **What is `@TestMethodOrder` and when would you use it?**
  - `@TestMethodOrder(MethodName.class)` or `@TestMethodOrder(OrderAnnotation.class)` controls test execution order.
  - Tests should ideally be order-independent, but ordered execution is useful for shared expensive setup, integration tests with sequential dependencies, and reproducing CI-specific order issues.
  - The default in JUnit 5 is deterministic but unspecified.

- **How do you integrate JUnit 5 with Spring Boot?**
  - Use `@SpringBootTest` for full application context, `@WebMvcTest` for web layer only, `@DataJpaTest` for JPA/repository tests, and `@ExtendWith(SpringExtension.class)` for non-Spring-Boot Spring tests.
  - Use `@MockBean` and `@SpyBean` to replace beans with mocks.
  - `@DynamicPropertySource` injects dynamic configuration (like Testcontainers connection details) into the Spring Environment.

---

## Developer Recommendations

- **Prefer `assertAll` over multiple sequential assertions**
  - Sequential assertions fail fast on the first failure, hiding subsequent failures and requiring multiple fix-rerun cycles.
  - `assertAll` executes all assertions and reports every failure at once, enabling you to fix all broken fields in a single pass.
  - Use it to group all assertions for a single logical check, such as all fields of a returned object.

- **Use `@ParameterizedTest` with `@MethodSource` instead of repetitive test methods**
  - A dozen test methods that differ only in input data are harder to maintain and extend than one parameterized test with a data stream.
  - `@MethodSource` separates test data from test logic — adding a new edge case adds one line to the data stream, not a new method.
  - Use `@CsvSource` for simple literal values and `@MethodSource` for complex test objects.

- **Use `@Nested` classes to organize large test classes**
  - Group related tests under inner classes like `@Nested class CreationTests`, `@Nested class ValidationTests`, with each group getting its own `@BeforeEach` and `@DisplayName`.
  - This makes test reports readable and helps developers find relevant tests quickly.

- **Use `@TempDir` instead of manual temporary file management**
  - Manual temp file creation with `Files.createTempDirectory()` in `@BeforeEach` and manual cleanup in `@AfterEach` is error-prone — forgotten cleanup leaks files, and `Files.delete()` throws on Windows when files are locked by antivirus.
  - `@TempDir` creates a temporary directory per test method, automatically cleans up with retry on failure, and does not mask test failures with cleanup exceptions.

- **Use `@Tag` to separate fast unit tests from slow integration tests**
  - Tag integration tests with `@Tag("integration")` and configure Maven Surefire to exclude them during local development with `mvn test` (which runs only untagged fast tests) while including them in CI with `mvn verify` (which runs everything).
  - This keeps the local development feedback loop under one second while still running comprehensive checks before deployment.

- **Use AssertJ over JUnit assertions for richer, automatic failure messages**
  - JUnit's `assertEquals(expected, actual)` tells you the two values but not what they represent.
  - AssertJ's `assertThat(actual).isEqualTo(expected).hasSize(3).isIn(a, b, c)` generates descriptive messages like "Expected status to be one of [PENDING, ACTIVE] but was CANCELLED".
  - The fluent API also prevents the common argument order error between expected and actual values.

- **Avoid shared mutable state between tests**
  - This is the single most common cause of flaky tests. Shared state in static fields, files, or databases creates order-dependent tests that fail intermittently.
  - Reset all state in `@BeforeEach`, use `@TempDir` for file-based tests, use `@DirtiesContext` for Spring context isolation, and provide each test with its own fresh test data rather than sharing fixtures between tests.
