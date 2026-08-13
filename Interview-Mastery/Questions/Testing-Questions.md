# Testing Questions

## Questions

1. What is unit testing?
2. What is integration testing?
3. Difference between unit and integration test.
4. What is JUnit?
5. What is Mockito?
6. What is mock?
7. What is stub?
8. Difference between mock and spy.
9. What is `@Mock`?
10. What is `@InjectMocks`?
11. What is `@MockBean`?
12. What is `@SpringBootTest`?
13. What is `@WebMvcTest`?
14. What is Testcontainers?
15. Why use Testcontainers?
16. How do you test repository layer?
17. How do you test service layer?
18. How do you test controller layer?
19. How do you test Kafka consumer?
20. How do you test scheduled jobs?
21. How do you test security rules?
22. What is code coverage?
23. Is 100% coverage always good?
24. What is regression testing?
25. What is smoke testing?
26. What is API testing?
27. What is Postman?
28. How do you test error cases?
29. How do you test performance changes?
30. How do you validate database optimization results?

---

## Answers

1. What is unit testing?
   - **Answer:** Unit testing verifies that a single, small unit of code — typically one method or one class — behaves correctly in complete isolation from its collaborators. External dependencies such as databases, HTTP clients, message brokers, and other services are replaced with test doubles like mocks or stubs, so any failure can be attributed to the unit under test and nothing else. A well-written unit test is deterministic, executes in milliseconds, and gives a precise diagnosis when it fails. The FIRST principles summarize the desired properties: Fast, Isolated, Repeatable, Self-validating, and Timely. Unit tests occupy the base of the test pyramid — the largest layer — because they are cheap to write, fast to run, and cover individual business rules exhaustively. Test-Driven Development (TDD) formalizes the cycle: write a failing test first, implement the minimal production code to satisfy it, then refactor while keeping the test green. In Java, JUnit 5 provides the test runner and assertions, while Mockito supplies mocks for dependencies. A typical unit test mocks the repository, invokes one service method, and asserts on the result:

   ```java
   class ValidationServiceTest {
       @Mock private SerialRepository repository;
       @InjectMocks private ValidationService service;

       @Test
       void rejectsInvalidSerial() {
           assertFalse(service.isValid("BAD!"));
       }
   }
   ```

   A unit test should never start a Spring context, open a real connection, or perform network I/O; when it does, it stops being a unit test and becomes an integration test. This isolation is what makes the unit layer fast enough to run on every commit and reliable enough to trust.

2. What is integration testing?
   - **Answer:** Integration testing verifies that multiple components work together correctly, exercising the real collaboration between them rather than replacing dependencies with doubles. Typical targets include the interaction between a service and a real database, the flow from a REST controller through the full request pipeline, or the consumption of messages from a real broker. These tests catch problems that unit tests structurally cannot, such as misconfigured bean wiring, incorrect SQL, wrong JSON field names, or mismatched API contracts. In the Spring Boot world, integration tests are usually built with `@SpringBootTest`, which starts the entire application context, and real external systems are often supplied by Testcontainers. Because a full context boot and real infrastructure are involved, integration tests are orders of magnitude slower than unit tests, so they sit in the middle of the test pyramid and are written in smaller numbers. They add most value at the seams of the system — places where code crosses from one boundary to another, such as the repository boundary, the HTTP boundary, or the messaging boundary. Good practice is to limit the number of full-context integration tests and to keep the rest as narrowly scoped slice tests. The goal is not to repeat what unit tests already verify, but to prove that the individually correct pieces agree on their shared interfaces and configuration.

3. Difference between unit and integration test.
   - **Answer:** The essential difference is scope: a unit test exercises one unit in isolation with all collaborators replaced by test doubles, while an integration test exercises several real components together to verify their interaction. Unit tests answer "does this method implement its logic correctly?"; integration tests answer "do these components actually work when connected?" A unit test for a service mocks the repository, so a database outage cannot affect it; an integration test uses a real database, so it validates the actual SQL and dialect behavior. Unit tests run in milliseconds and are fully deterministic; integration tests take seconds or minutes because they boot Spring contexts and infrastructure. The test pyramid expresses the recommended ratio — roughly 70% unit tests, 20% integration tests, and 10% end-to-end tests — reflecting the inverse relationship between speed and coverage breadth. Integration tests are also the safety net for refactoring: when a refactor changes an internal structure but preserves behavior, unit tests confirm the units and integration tests confirm the whole flow still holds. End-to-end tests, at the top of the pyramid, drive the full deployed system through its real user interfaces, validating the system as a whole. Each layer compensates for the blind spots of the layer below, which is why a healthy suite contains all three.

4. What is JUnit?
   - **Answer:** JUnit is the de-facto standard testing framework for Java, with JUnit 5 (Jupiter) being the current generation. It provides the test lifecycle, the `@Test` annotation to mark test methods, and a rich assertion library such as `assertEquals`, `assertTrue`, `assertThrows`, and `assertAll`. Lifecycle methods are controlled with `@BeforeEach` and `@AfterEach` for per-test setup and teardown, and `@BeforeAll` and `@AfterAll` for class-level initialization. Parameterized tests allow one test method to run against many inputs, which is ideal for data-driven validation logic:

   ```java
   @ParameterizedTest
   @ValueSource(strings = {"", "  ", "abc"})
   void rejectsInvalidSerials(String input) {
       assertFalse(validator.isValid(input));
   }
   ```

   Other sources include `@CsvSource` for multiple columns and `@MethodSource` for computed datasets. `@DisplayName` gives tests readable names, `@Nested` organizes related tests into a hierarchy, and `@Timeout` enforces execution-time limits to catch regressions in performance. In Maven builds, the Surefire plugin runs JUnit 5 tests as part of the standard test phase, making every `mvn test` execution a complete run of the unit suite. JUnit's tight integration with CI means a failing assertion produces a clear, machine-readable report that fails the build and identifies the exact failing case.

5. What is Mockito?
   - **Answer:** Mockito is a mocking framework for Java that creates test doubles to isolate the unit under test from its dependencies. The core workflow is to mock a dependency, stub the behavior it should return, exercise the code under test, and then verify the interactions that took place. Stubbing is done with `when(...).thenReturn(...)` for normal values, `when(...).thenThrow(...)` or `doThrow(...)` for exceptions, and `doAnswer(...)` for computed responses:

   ```java
   when(repository.findById(42L)).thenReturn(Optional.of(entity));
   when(repository.save(any())).thenThrow(new DataIntegrityViolationException("dup"));
   ```

   Verification checks that methods were actually invoked, which distinguishes behavior verification from simple state checking: `verify(repository).save(expectedEntity)` asserts the method was called, and `verify(repository, times(2)).findById(any())` asserts the call count. Argument matchers like `any()`, `eq()`, `anyLong()`, and `isNull()` make stubbing and verification flexible without losing precision. `@Mock` and `@InjectMocks` reduce boilerplate by generating mocks and injecting them automatically. A `spy()` wraps a real object and allows overriding only selected methods, which is useful when most real behavior should be kept. Mockito also supports ordered verification with `InOrder` when call sequencing matters, and lenient vs strict stubbing to detect unused stubs that indicate over-specification. The overall effect is that business logic can be tested deterministically without a database, network, or any other real dependency.

6. What is mock?
   - **Answer:** A mock is a test double that not only provides canned responses but also records how it was used, allowing the test to verify behavior rather than just outcomes. Unlike a real dependency, a mock is completely under the control of the test: its methods can be configured to return specific values, throw specific exceptions, or do nothing at all. A mock makes a test fast, deterministic, and isolated, because it removes network latency, database state, and external failures from the equation. The defining capability of a mock is behavior verification: after the code under test runs, the test can assert that specific methods were called with the expected arguments and the expected frequency. For example, a service that must save an entity on every call can be tested by asserting `verify(repository).save(entity)` — if the code forgets to persist, the test fails even though the returned value looks correct. Mocks are therefore the right tool when the goal is to test how the code interacts with its collaborators. They are not appropriate for verifying that a third-party library works correctly, which is the job of integration tests against the real dependency.

7. What is stub?
   - **Answer:** A stub is a test double that returns predefined responses to specific method calls, making the environment around the code under test predictable. Stubs are about state, not behavior: they provide the inputs the code needs so its output can be checked, but they do not record or assert on how they were called. For example, stubbing `repository.findById(42L)` to return a known entity lets the test focus on how the service reacts to a found entity versus a missing one. Stubs are the simplest form of test double and are typically configured with `when(...).thenReturn(...)` in Mockito. The essential difference from a mock is intent: a stub supplies data so the test can verify the resulting state, whereas a mock records interactions so the test can verify the calling behavior. In Mockito, the same `when/thenReturn` mechanism is used for both, and the distinction becomes visible in whether the test calls `verify`. Stubs shine when the test's goal is to answer "given this input, does the code produce the correct result?" rather than "did the code call the dependency in the expected way?"

8. Difference between mock and spy.
   - **Answer:** A mock is a completely fake object that has no real implementation; every method returns a default value (or a stubbed value) and does nothing unless configured. A spy, by contrast, wraps a real instance, delegating to the real methods by default while allowing individual methods to be stubbed when needed. Concretely, `mock(Repository.class)` produces an object with no real behavior, while `spy(new Repository())` keeps the real logic and lets the test override only selected methods:

   ```java
   AuditService spy = spy(new AuditService());
   doNothing().when(spy).send(anyString());
   spy.record(42L); // real record() logic, but send() is skipped
   verify(spy).record(42L);
   ```

   Mocks are the safer default because they cannot accidentally trigger real side effects such as database writes or network calls. Spies are useful in a narrow set of situations: legacy code that cannot be refactored to allow clean injection, or when verifying that a method delegates correctly while stubbing only a small part of its behavior. The main risks of spies are calling real methods during stubbing (partial mocking side effects) and silently depending on real implementations that may change. As a rule of thumb, prefer mocks for all collaborators in new code, and reach for spies only when partial behavior genuinely must be preserved.

9. What is `@Mock`?
   - **Answer:** `@Mock` is a Mockito annotation that marks a field to be replaced by an automatically generated mock when the test class is initialized. It is the declarative equivalent of calling `Mockito.mock(SomeClass.class)` and eliminates the boilerplate of manual mock creation. The mocks are created when the test framework initializes Mockito, either through `@ExtendWith(MockitoExtension.class)` in JUnit 5 or by calling `MockitoAnnotations.openMocks(this)` in a `@BeforeEach` method. Each `@Mock` field holds a mock for one dependency, so a test class typically declares one mock per collaborator of the class under test:

   ```java
   @ExtendWith(MockitoExtension.class)
   class InventoryServiceTest {
       @Mock private SerialRepository repository;
       @Mock private AuditClient auditClient;
       @InjectMocks private InventoryService service;
   }
   ```

   Using the annotation keeps test setup concise, and the strict-stubs feature of `MockitoExtension` flags unused stubs, catching tests that over-specify or lose their meaning. `@Mock` is scoped to the plain unit test — it does not touch the Spring application context at all. When a mock needs to live inside the Spring context instead, `@MockBean` is the corresponding mechanism.

10. What is `@InjectMocks`?
    - **Answer:** `@InjectMocks` creates a real instance of the class under test and automatically injects the mocks declared with `@Mock` into its dependencies. This removes the need to manually construct the class and pass every collaborator in a constructor call. Mockito performs the injection by trying constructor injection first, then setter injection, and finally field injection, so a class with a single explicit constructor produces the most reliable results. For injection to work correctly, the dependency types declared in the class must be compatible with the mock types declared in the test. A classic example:

    ```java
    @Mock private SerialRepository repository;
    @InjectMocks private InventoryService service;
    ```

    Here `InventoryService` is instantiated for real, and the `SerialRepository` mock is wired into it automatically. `@InjectMocks` only injects into the object under test; it does not create a mock of that object, because the whole point is to exercise the real implementation against mocked collaborators. Keeping the class under test as a real instance is what makes `@InjectMocks` tests genuine unit tests rather than tests of interactions between doubles.

11. What is `@MockBean`?
    - **Answer:** `@MockBean` places a mock into the Spring application context, replacing the real bean of that type for the duration of the test. It belongs to the Spring Test framework, not to Mockito, and is used in Spring Boot slice and integration tests where the context must resolve dependencies. In a controller slice test, the real service bean is swapped for a mock so the test focuses purely on the web layer:

    ```java
    @WebMvcTest(InventoryController.class)
    class InventoryControllerTest {
        @Autowired private MockMvc mockMvc;
        @MockBean private InventoryService service;
    }
    ```

    Every injection point in the context that needs `InventoryService` receives the same mock, which is what makes slice testing feasible without booting the real business layer. The cost is a slower context and an additional dependency on the Spring Test framework, so `@MockBean` is inappropriate for plain unit tests where `@Mock` is the lightweight option. Note that `@MockBean` mocks reset after each test, which keeps tests isolated but means stubbing must be set up per test. In Spring Boot 3.4+, `@MockBean` is deprecated in favor of `@MockitoBean`, which behaves the same way using Mockito's own bean-creation mechanism. The rule of thumb: use `@Mock` for unit tests and `@MockBean`/`@MockitoBean` only when the mock must participate in the Spring context.

12. What is `@SpringBootTest`?
    - **Answer:** `@SpringBootTest` starts the full Spring Boot application context, which makes it the standard tool for true integration testing of the whole application. It validates that all beans wire together, that configuration properties resolve, and that the complete request-to-database flow works as a whole. The `webEnvironment` attribute controls the web layer: `MOCK` (the default) simulates the servlet environment with MockMvc, `RANDOM_PORT` starts a real server on a random port, and `NONE` disables the web layer entirely. With `RANDOM_PORT`, tests interact over real HTTP using `TestRestTemplate` or `RestAssured`, which exercises serialization, filters, and error handling exactly as production does. Real external dependencies such as databases are usually provided through Testcontainers rather than mocked, because the point of the test is to verify genuine integration:

    ```java
    @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
    class OrderFlowIT {
        @Container @ServiceConnection static PostgreSQLContainer<?> db = new PostgreSQLContainer<>("postgres:16");
        @LocalServerPort private int port;
    }
    ```

    Because a full context boot is expensive, `@SpringBootTest` classes should be kept few and focused on critical end-to-end flows, while slice tests cover narrower concerns. `@ActiveProfiles("test")` selects test-specific configuration so the tests do not depend on developer-local settings. `@SpringBootTest` does not replace unit tests; it complements them by proving that individually verified pieces actually work together under real Spring wiring.

13. What is `@WebMvcTest`?
    - **Answer:** `@WebMvcTest` is a Spring Boot slice test that loads only the web layer — controllers, `@ControllerAdvice`, converters, filters, and validation — instead of the entire application context. Dependencies of the controllers, typically services, are replaced with `@MockBean` or `@MockitoBean` mocks so the test exercises only the HTTP concerns. It is substantially faster than `@SpringBootTest` because most beans never get instantiated. Tests drive the controller through MockMvc, which can assert status codes, response headers, JSON content, and error responses without starting a real server:

    ```java
    @WebMvcTest(InventoryController.class)
    class InventoryControllerTest {
        @Autowired private MockMvc mockMvc;
        @MockitoBean private InventoryService service;

        @Test
        void returns404WhenSerialUnknown() throws Exception {
            when(service.findBySerial("X")).thenReturn(Optional.empty());
            mockMvc.perform(get("/api/inventory/X"))
                   .andExpect(status().isNotFound());
        }
    }
    ```

    `@WebMvcTest` is where controller validation, request mapping, and exception handling get verified: malformed JSON triggers 400, missing auth yields 401 or 403, and `@ExceptionHandler` responses are checked. It also lets security rules be tested with `@WithMockUser`. What it does not cover is the real service and repository behavior, so business-logic correctness still needs its own unit tests. The slicing principle generalizes: `@DataJpaTest` for the data layer, `@JsonTest` for serialization, and so on, each faster than the full context.

14. What is Testcontainers?
    - **Answer:** Testcontainers is a Java library that manages disposable Docker containers for tests, giving access to real databases, message brokers, and other services instead of in-memory substitutes. A container is declared with `@Container` inside a test class annotated with `@Testcontainers`, and the library handles pull, start, and cleanup:

    ```java
    @Testcontainers
    class RepositoryIT {
        @Container @ServiceConnection
        static PostgreSQLContainer<?> db = new PostgreSQLContainer<>("postgres:16");
    }
    ```

    `@ServiceConnection` automatically configures Spring Boot datasource properties from the container, removing manual property wiring. Tests run against the exact same database engine, version, and features as production, which eliminates the class of bugs caused by "works on H2, fails on the real database." Each test run gets a fresh container, so state from one test cannot leak into another, and containers are torn down automatically afterward. The library supports many images out of the box — PostgreSQL, MySQL, MSSQL, Kafka, Redis, Elasticsearch — and custom images can be wrapped when needed. The main tradeoff is speed: starting a container takes seconds, so Testcontainers tests are significantly slower than unit tests and are usually reserved for integration and repository layers. Running them in CI is straightforward because modern CI runners support Docker, making them the de-facto standard for reliable integration testing.

15. Why use Testcontainers?
    - **Answer:** Testcontainers exists because in-memory or embedded substitutes do not faithfully reproduce the behavior of the real system they replace. A typical failure mode is SQL dialect drift: an H2 database accepts queries and functions that the production database rejects, so tests pass locally and break in production. Real differences include window functions, full-text search, collation rules, JSON operators, and stored procedures — precisely the features that mature applications depend on. By running a genuine container image, Testcontainers ensures the SQL actually executed matches production, and it also validates that schema migrations applied correctly. The tradeoff is startup time and resource usage, so the pragmatic strategy is to use Testcontainers for the repository and integration layers where correctness matters most, and faster slice or unit tests everywhere else. Containers are created per test class and shared where safe, keeping the extra cost bounded. Even when a substitute like H2 is used, it makes sense to retain one Testcontainers-based suite that runs against the real engine to catch dialect drift. Ultimately the choice is between a small, reliable suite with real infrastructure versus a fast suite that can silently diverge from production reality.

16. How do you test repository layer?
    - **Answer:** The repository layer is tested with `@DataJpaTest`, a Spring Boot slice that loads only JPA infrastructure — entity mappings, repositories, and the data source — without starting the rest of the application. Each test is transactional by default and rolls back at the end, so tests are isolated from one another. The database is typically either an embedded database for speed or a real one via Testcontainers for dialect fidelity. A typical test saves entities, invokes a repository method, and asserts on the results:

    ```java
    @DataJpaTest
    class SerialRepositoryTest {
        @Autowired private SerialRepository repository;

        @Test
        void findsBySerialIgnoringCase() {
            repository.save(new Serial("ABC-123"));
            assertTrue(repository.existsBySerialIgnoreCase("abc-123"));
        }
    }
    ```

    This verifies that derived query methods generate the expected SQL and that `@Query` annotations, native queries, pagination, and sorting behave correctly against a real schema. Projection interfaces and entity graph settings can also be validated here, because the test exercises actual SQL execution rather than mock behavior. Since the tests run against a real database, schema migration scripts are validated at the same time. `@DataJpaTest` is substantially faster than `@SpringBootTest` because only the data-layer beans are loaded, keeping repository tests fast enough to run on every build.

17. How do you test service layer?
    - **Answer:** The service layer holds the business logic and is best tested as pure unit tests with JUnit and Mockito, mocking the repositories and external clients it depends on. The class under test is created with `@InjectMocks`, its collaborators are declared with `@Mock`, and each test stubs the exact data the scenario needs. Happy-path scenarios assert the expected return value or entity state, while the surrounding cases get equal attention: validation failures, resource-not-found, duplicate keys, and exceptions propagated from the persistence layer:

    ```java
    @Test
    void throwsWhenSerialAlreadyExists() {
        when(repository.existsBySerial("ABC-123")).thenReturn(true);
        assertThrows(DuplicateSerialException.class,
            () -> service.register(new SerialRequest("ABC-123")));
    }
    ```

    `verify` confirms that side effects — saving an entity, calling an audit client, publishing an event — actually happened with the expected arguments. Tests must be independent: each test sets up only its own mocks and data, so reordering or running them in parallel changes nothing. Edge cases such as null inputs, empty collections, and boundary values complete the matrix. Because everything is mocked, these tests run in milliseconds and can cover dozens of scenarios, which is why the service layer dominates the unit-test layer of the pyramid.

18. How do you test controller layer?
    - **Answer:** The controller layer is tested with `@WebMvcTest` and MockMvc, which loads only the web slice and drives controllers through the real HTTP request pipeline. Service dependencies are mocked with `@MockBean` or `@MockitoBean`, so the test focuses exclusively on mapping, validation, status codes, and JSON output. MockMvc builds requests and asserts on the full response:

    ```java
    mockMvc.perform(post("/api/inventory")
               .contentType(MediaType.APPLICATION_JSON)
               .content("{\"serial\":\"\"}"))
           .andExpect(status().isBadRequest())
           .andExpect(jsonPath("$.errors.serial").exists());
    ```

    The test matrix covers the standard statuses: 200/201 for success, 400 for bean-validation failures, 404 for unknown resources, and 401/403 for security violations. `jsonPath` assertions verify the structure and values of the response body, and custom error responses from `@ControllerAdvice` are validated here as well. Request content-type handling, path variables, query parameters, and headers are all exercised through real Spring MVC machinery. Security rules are tested in this slice with `@WithMockUser` and by sending requests without authentication to assert the expected 401. Because only the web layer is loaded, the suite is fast and can comprehensively cover every endpoint and its edge cases.

19. How do you test Kafka consumer?
    - **Answer:** Kafka consumers are tested at two levels: the message-handling logic as a plain unit test, and the actual consumption flow with a real or embedded broker. The handler logic should be extracted into a plain method so it can be unit-tested with mocked collaborators, covering message parsing, validation, and error handling. The integration layer can use `@EmbeddedKafka` from Spring Kafka, which starts an in-memory broker for lightweight tests, or Testcontainers with a real Kafka image when fidelity to production matters:

    ```java
    @EmbeddedKafka(partitions = 1, topics = "inventory.events")
    class InventoryConsumerIT {
        @Autowired private KafkaTemplate<String, String> kafkaTemplate;
        @Autowired private InventoryConsumer consumer;

        @Test
        void processesMessage() {
            kafkaTemplate.send("inventory.events", "{\"serial\":\"ABC-123\"}");
            await().atMost(10, TimeUnit.SECONDS)
                   .untilAsserted(() -> verify(consumer).handle(any()));
        }
    }
    ```

    Because consumption is asynchronous, tests must wait for delivery with `awaitility` or a timeout rather than asserting immediately. The test matrix includes happy-path processing, deserialization errors, poison-pill messages that cannot be parsed, and retry behavior when downstream processing fails. Offset-commit semantics can be observed by checking that the consumer continues to receive messages after a restart. The critical point is that consumer tests should assert on the side effect of consumption — data stored, API called, event re-published — and not merely on the message having been received.

20. How do you test scheduled jobs?
    - **Answer:** The scheduler itself — the Spring `@Scheduled` annotation — is framework behavior and does not need to be tested; what matters is the logic the job executes. The job body should be extracted into a plain, injectable service so it can be tested directly with JUnit and Mockito, exactly like any other service method. The test then covers the job's outcomes: what it reads, what it computes, and what it writes, using mocked dependencies for full isolation:

    ```java
    @Test
    void reconciliationSkipsAlreadyReconciled() {
        when(repository.findUnreconciledSince(any())).thenReturn(List.of());
        reconciliationService.run();
        verify(repository, never()).markReconciled(any());
    }
    ```

    Idempotency is an especially important property to test: running the job twice should produce the same result and not create duplicates. Failure handling matters too — a job should not die silently or corrupt data when one batch fails, so tests verify retry or skip behavior. If an end-to-end verification is needed, `@SpringBootTest` with a short fixed-delay schedule can trigger the job and assert on the final state. This layered approach gives the speed of unit tests for the logic and the confidence of an integration test for the wiring, without depending on wall-clock timing in the fast test suite.

21. How do you test security rules?
    - **Answer:** Security rules are tested at the web layer with `@WebMvcTest`, where Spring Security filters run inside the real request pipeline. The negative cases come first: an unauthenticated request must receive 401, and an authenticated user without the required role must receive 403. Role-based access is simulated with `@WithMockUser`, which injects an authentication into the security context for the request:

    ```java
    @Test
    @WithMockUser(roles = "USER")
    void userCannotDeleteInventory() throws Exception {
        mockMvc.perform(delete("/api/inventory/42"))
               .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void adminCanDeleteInventory() throws Exception {
        mockMvc.perform(delete("/api/inventory/42"))
               .andExpect(status().isOk());
    }
    ```

    When a custom `UserDetailsService` or token-based authentication is used, tests can seed a real user in the context or use `@WithUserDetails`. Method-level security with `@PreAuthorize` and `@Secured` is validated the same way, since the annotation checks the authority stored in the security context. CSRF protection is tested by performing mutating requests without the CSRF token and asserting the expected failure. The full matrix — anonymous, wrong role, right role, disabled account — gives confidence that authorization rules behave as specified without booting the entire application.

22. What is code coverage?
    - **Answer:** Code coverage measures what percentage of the codebase was executed during the test run, answering "how much code do tests touch?" rather than "how well is the code tested?". Tools like JaCoCo for Java instrument the bytecode and report several metrics: line coverage, branch coverage, and instruction coverage. Branch coverage is the more meaningful number because it reports whether both the true and false arms of each `if` and each case of a `switch` were exercised. Coverage is typically enforced as a CI quality gate, for example a JaCoCo rule that fails the build if branch coverage drops below a threshold. Sensible configuration excludes generated code, DTOs, and configuration classes from the measurement, because covering boilerplate inflates the number without improving confidence. Coverage is reported per class, per package, and overall, so a low number on a specific service class immediately points to untested business logic. The numbers guide where to add tests, but they must be interpreted carefully: a covered line can still contain a wrong assertion, and an uncovered branch is a definite gap. Coverage is best used as a floor that catches regressions, not as a target to maximize blindly.

23. Is 100% coverage always good?
    - **Answer:** No — 100% coverage guarantees that every line executed, but it says nothing about the quality of the assertions attached to those executions. A test can reach every branch while asserting nothing meaningful, or can assert outcomes that are trivially true, and the coverage number would look perfect. Coverage cannot detect tests that miss the interesting scenarios, such as the boundary values, the failure paths, or the interactions that matter. Mutation testing is a stronger measure: it changes the code slightly — for example inverting a condition or removing a method call — and runs the suite; if no test fails, the tests failed to lock in that behavior. This directly exposes assertions that are too weak to detect real bugs. Practical guidance is to aim for high coverage on business-critical logic such as validation rules, calculations, and error handling, and to accept lower coverage on glue code. Chasing 100% everywhere usually means writing low-value tests that make the suite harder to maintain while adding little protection. The right mindset is that coverage is a diagnostic tool — it tells you where tests are missing — not a guarantee of correctness.

24. What is regression testing?
    - **Answer:** Regression testing verifies that new changes do not break behavior that previously worked, guarding against the reintroduction of fixed bugs or the corruption of existing features. In practice it is an automated safety net: every time code changes, the existing suite runs and any test that fails signals an unintended behavior change. The strength of the net depends on the breadth of the suite, which is why the test pyramid matters — many fast unit tests catch logic regressions early, while integration and end-to-end tests catch wiring and cross-component regressions. Running the full suite on every pull request in CI means regressions are discovered minutes after the change, when the context is fresh and the fix is cheap. Tests should be written for every fixed bug, so the regression suite grows with the system's history of known failure modes. When a regression does slip through, the standard practice is to add a failing test first, then fix the code, so the case is protected forever after. Regression testing is what makes refactoring and continuous delivery safe, because the suite absorbs the risk that otherwise would be carried by manual testing.

25. What is smoke testing?
    - **Answer:** Smoke testing is a shallow but fast check that the application is actually alive and its core paths work, performed right after a deployment or build. Its purpose is to catch show-stopping problems — wrong configuration, missing dependencies, failure to start — before any deeper testing or real traffic is attempted. Typical smoke checks are a health endpoint returning 200, a simple read endpoint returning a well-formed response, and a basic database round-trip. A good smoke suite runs in under a minute and asserts only the coarsest level of correctness, because it is executed on every environment promotion, not just in the test phase. In CI/CD, smoke tests run immediately after deployment to staging or production and abort the pipeline on failure, preventing a broken build from reaching users. Because they are so cheap, they can run against every deployment, including scheduled builds that do not trigger the full suite. Smoke testing is complementary to regression testing: regression answers "did anything break?", while smoke answers "is the thing even up?". Together they form a two-stage gate — fast triage first, deep verification second.

26. What is API testing?
    - **Answer:** API testing validates a REST interface by sending real HTTP requests and asserting on the full response: status code, headers, body content, and schema. It covers the success paths — correct inputs returning 200/201 with the expected JSON — and the failure paths: invalid input yielding 400, missing authentication yielding 401, wrong roles yielding 403, and unknown resources yielding 404. Tests also check contract details like content-type, response headers, pagination metadata, and field types, which is where client integrations often break. API tests can be written in the test suite with Spring's `TestRestTemplate` or with RestAssured for more expressive request/response assertions, and the same tests can target a locally booted app or a deployed environment. In CI, API tests validate that the build's wire contract matches what consumers expect before anything is released. Contract testing extends the idea: producer and consumer sides agree on a shared contract (for example with Spring Cloud Contract) and verify independently. Together with performance checks on latency and throughput, API testing provides the outer safety net that unit and integration tests, by nature, cannot provide for the wire-level contract.

27. What is Postman?
    - **Answer:** Postman is a tool for developing, testing, and documenting APIs through a graphical interface, built around the concept of collections of requests. A collection groups related requests and can carry environment variables — base URLs, tokens, credentials — so the same requests work against dev, test, and production environments without editing them. Pre-request scripts run JavaScript before a request to compute dynamic values, typically to fetch an authentication token and store it in a variable. Test scripts run after the response and contain assertions, such as checking that the status code is 200 and that `jsonPath` matches the expected value:

    ```javascript
    pm.test("returns 200 with inventory", () => {
        pm.response.to.have.status(200);
        pm.expect(pm.response.json().serial).to.eql("ABC-123");
    });
    ```

    Collections are stored in the repo as version-controlled JSON, so the API surface is reviewable and shared across team members. The same collections can be executed headlessly in CI using Newman, Postman's command-line runner, turning manually built requests into automated regression tests. Postman also doubles as living API documentation and a tool for ad-hoc exploration during development. Its role is pragmatic: fast manual iteration during development, plus a bridge to automation for teams that do not want to write test code by hand.

28. How do you test error cases?
    - **Answer:** Error cases are tested deliberately: the test sends inputs designed to fail and asserts that the application responds with the correct error contract. The matrix splits into client errors (4xx) and server errors (5xx). Client errors include missing required fields, invalid formats, out-of-range values, unknown IDs, and duplicate entries, which should produce 400 or 404 with a structured error body:

    ```java
    mockMvc.perform(post("/api/inventory").content("{}"))
           .andExpect(status().isBadRequest())
           .andExpect(jsonPath("$.errors.serial").exists());
    ```

    Server errors are typically triggered by stubbing a dependency to throw — a repository throwing `DataIntegrityViolationException` or a remote client timing out — and asserting that the `@ControllerAdvice` handler maps it to the correct 500 response instead of leaking a stack trace. The tests verify that the error body is stable, contains actionable messages, and does not expose internal implementation details. Error handling in the business layer is tested at the unit level with `assertThrows` to confirm the right exception types and messages. Combined, these tests ensure that consumers of the API always receive a predictable, documented failure mode rather than an unhandled exception.

29. How do you test performance changes?
    - **Answer:** Performance testing compares measurements before and after a change under identical conditions, because a single absolute number is meaningless without a baseline. The workload must be reproducible — the same script or request mix, the same dataset, and the same environment — so differences reflect the code change and not noise. Metrics to capture include response time (p50, p95, p99), throughput, and resource usage such as CPU, memory, and I/O. Tools like JMeter or Gatling drive load tests, while Spring Actuator metrics or a profiler capture the system side of the measurement. Warm-up is essential: caches, connection pools, and JIT compilation must be warmed before timing begins, otherwise early-run numbers are misleading. Multiple runs reduce variance, and the comparison is made on the distribution, not on a single lucky run. Consistency also requires considering concurrent load, because a change that looks fine at low concurrency can regress badly under contention. If a measured regression appears, profiling identifies the hotspot, and the change is either optimized or reverted. Performance testing is only trustworthy when conditions are tightly controlled; otherwise the results are anecdotal.

30. How do you validate database optimization results?
    - **Answer:** Database optimization is validated by comparing execution plans and resource metrics before and after the change, using the same realistic dataset and workload. The key indicators in the plan are the access method — index seek versus index scan — and the operators, such as a nested-loop join versus a hash join, plus the estimated vs actual row counts. Logical reads and I/O counts from commands like `SET STATISTICS IO` quantify how much data the engine touches, which usually dominates wall-clock time more than CPU. The before and after runs must use realistic data volumes, because an optimization that works on a tiny table can be catastrophic at production scale, and the optimizer makes different decisions as row counts grow. Correctness must be verified alongside speed: the optimized query must return identical results, which is best confirmed by comparing output sets with a hash or checksum. Parameter sniffing means the validation should also cover representative parameter values, since one value may produce a good plan and another a bad one. After the change, the query is monitored in production for at least one full reporting cycle to confirm the improvement holds under real load. The discipline is to prove the change with evidence — plans, reads, timings — rather than trusting that an index or rewrite is better by intuition.
