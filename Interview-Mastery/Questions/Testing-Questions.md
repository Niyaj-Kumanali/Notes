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
   - **Answer:**
      - Tests individual components in isolation, mocking external dependencies
      - In our inventory project, wrote JUnit + Mockito tests for the validation service - mocking the repository and testing validation logic with various serial number formats
      - FIRST principles: Fast (no I/O, run in milliseconds), Isolated (no shared state between tests), Repeatable (same result every time), Self-validating (clear pass/fail), Timely (written alongside production code). The test pyramid: unit tests at the base (fast, numerous, cheap), integration tests in the middle (slower, fewer, test component interaction), end-to-end tests at the top (slow, few, expensive, verify full system behavior)
2. What is integration testing?
   - **Answer:**
      - Verifies that components work together correctly, often involving real databases or message brokers
      - In our cold-chain project, used `@SpringBootTest` with Testcontainers to test the full Kafka consumer to InfluxDB flow
      - Integration tests catch configuration issues that unit tests miss — wrong connection strings, mismatched serialization formats, missing bean wiring, incorrect Spring profiles. We balanced coverage by writing unit tests for all business logic and integration tests for critical paths: database queries, message publishing/consuming, and API endpoint wiring
3. Difference between unit and integration test.
   - **Answer:**
      - Unit tests test a single component in isolation (mocked dependencies), while integration tests test multiple components together (real dependencies)
      - Our inventory validation service had unit tests for rules and integration tests for the full submission flow from controller to database
      - The test pyramid ratio we followed: roughly 70% unit tests (fast feedback, test every branch), 20% integration tests (verify wiring and real interactions), 10% end-to-end tests (critical user journeys). Unit tests run in milliseconds; integration tests take seconds to minutes depending on infrastructure. Unit tests isolate failures to a single class; integration tests may fail due to infrastructure issues
4. What is JUnit?
   - **Answer:**
      - The standard testing framework for Java with annotations like `@Test` and assertions
      - Used JUnit 5 (Jupiter) for all tests with Maven Surefire plugin running them in the CI pipeline
      - JUnit 5 features: `@ParameterizedTest` with `@CsvSource` or `@MethodSource` for running the same test with multiple inputs, `@Nested` classes to group related tests and share setup, `@DisplayName` for readable test names, `@Tag` for filtering tests by category, `@BeforeEach`/`@AfterEach` for per-test setup/teardown, and `@BeforeAll`/`@AfterAll` for class-level lifecycle
5. What is Mockito?
   - **Answer:**
      - Creates mock objects for testing components in isolation
      - In our service-layer tests, used `when(repository.findById(any())).thenReturn(Optional.of(entity))` to test business logic without needing a database
      - `verify(mock).methodCall(args)` checks that a method was called with specific arguments. Argument matchers like `any()`, `eq()`, `contains()` make assertions flexible. `spy()` creates a partial mock of a real object, stubbing only selected methods. `@InjectMocks` automatically injects `@Mock` fields into the class under test, handling constructor, setter, or field injection
6. What is mock?
   - **Answer:**
      - A fake object that mimics a real dependency with expectations set during the test
      - In testing our inventory service, mocked the repository to return predefined data, making tests fast, isolated, and deterministic
      - Mocks vs stubs: stubs are state-based — you set up predefined return values and the test passes if the code produces the right result. Mocks are interaction-based — you set expectations (this method must be called exactly once with these arguments) and verify those interactions after execution. Mockito mocks combine both capabilities: you can stub behavior and verify calls on the same mock
7. What is stub?
   - **Answer:**
      - Returns predefined responses to method calls, making the test environment predictable
      - In our CDMS tests, stubbed stored procedure calls to return known result sets, testing reporting logic without running the actual procedure
      - Stubs focus on state testing — provide data, run the code, assert the output. Mocks focus on interaction behavior — verify that the code called the right methods in the right order with the right arguments. A stub doesn't care if you call it once or ten times; a mock will fail if expectations aren't met
8. Difference between mock and spy.
   - **Answer:**
      - A mock creates a completely fake object with no real behavior, while a spy wraps a real object and allows overriding specific methods
      - Used mocks for repositories (completely fake) and spies when we needed some real behavior but wanted to stub others
      - Prefer mocks by default — they are explicit about behavior and have no hidden side effects. Spies call real methods by default, which means they can trigger side effects (database writes, network calls) if the real implementation is not carefully managed. Use a spy only when you genuinely need to test a real class with one or two methods overridden
9. What is `@Mock`?
   - **Answer:**
      - A Mockito annotation that creates and injects a mock for the annotated field
      - Annotated repository dependencies with `@Mock` in test classes, using `MockitoAnnotations.openMocks(this)` or `@ExtendWith(MockitoExtension.class)` for initialization
      - `@Mock` reduces boilerplate compared to calling `Mockito.mock()` manually for each field. With `@ExtendWith(MockitoExtension.class)`, Mockito handles the lifecycle automatically — creates mocks before each test and closes them after, preventing resource leaks. `@Mock` is for plain unit tests; `@MockBean` is for Spring context tests
10. What is `@InjectMocks`?
   - **Answer:**
      - Creates an instance of the annotated class and injects mocks into its dependencies
      - In our service tests, the service class had `@InjectMocks`, and Mockito automatically injected the `@Mock` repositories, eliminating manual constructor calls
      - Mockito injection order: constructor injection first (matches by type, largest constructor), then setter injection, then field injection directly. The class under test should ideally have a single constructor to avoid ambiguity. If multiple mocks match the same type, Mockito injects the matching `@Mock` by name — keeping field names consistent with the production class avoids mismatches
11. What is `@MockBean`?
   - **Answer:**
      - Adds a mock to the Spring application context, replacing any existing bean
      - In our `@WebMvcTest` controller tests, used `@MockBean` to mock service layer beans while testing only the controller layer
      - `@MockBean` vs `@Mock`: `@MockBean` registers a mock in the Spring ApplicationContext — necessary for Spring slice tests (`@WebMvcTest`, `@DataJpaTest`) where beans are wired by the container. It is slower because the Spring context must start. `@Mock` is for plain unit tests with no Spring context — faster but cannot replace beans in the DI container. Use `@MockBean` when testing through Spring, `@Mock` otherwise
12. What is `@SpringBootTest`?
   - **Answer:**
      - Loads the full Spring Boot application context for integration testing
      - In our cold-chain project, used it with Testcontainers to test the full flow from controller to Kafka to InfluxDB, verifying Spring bean wiring and configuration
      - `webEnvironment = RANDOM_PORT` starts a real embedded server for testing with `TestRestTemplate` or `WebTestClient`, useful for testing filters, security, and serialization end-to-end. `webEnvironment = MOCK` (default) uses MockMvc without starting a server. `@ActiveProfiles("test")` loads test-specific configuration (test databases, mocked external services, debug logging) separate from dev and production profiles
13. What is `@WebMvcTest`?
   - **Answer:**
      - Loads only the web layer for focused controller testing
      - Used it with `@MockBean` for services, testing endpoint mappings, request validation, status codes, and error responses without loading the full context
      - Faster than `@SpringBootTest` because it only instantiates the controller, `WebMvcConfigurer`, filters, and `@ControllerAdvice` — no service or repository beans. `MockMvc` sends simulated HTTP requests and asserts on the result: `mockMvc.perform(get("/api/items")).andExpect(status().isOk()).andExpect(jsonPath("$.name").value("expected"))`. This tests serialization, validation annotations, and exception handling without starting a real server
14. What is Testcontainers?
   - **Answer:**
      - Provides disposable Docker containers for integration testing
      - In our cold-chain project, used it to spin up real Kafka and InfluxDB containers during tests, ensuring tests ran against the same versions as production
      - Testcontainers improves reliability over in-memory alternatives (H2, embedded Kafka) by running the exact same database or broker version as production. `@Container` defines the container and `@Testcontainers` manages its lifecycle (start before tests, stop after). Containers are reused across test classes with `reuse=true` to avoid repeated startup overhead. Supports PostgreSQL, MySQL, Kafka, Redis, and any Docker image
15. Why use Testcontainers?
   - **Answer:**
      - Ensures tests run against real databases, not in-memory simulations that behave differently
      - H2 did not support all MSSQL features our stored procedures used, so Testcontainers with a real MSSQL container caught compatibility issues earlier
      - The tradeoff: container startup adds seconds to test execution, but you gain confidence that tests reflect real behavior. We ran Testcontainers tests in CI on every pull request but excluded them from local fast-feedback loops during development. For developers who needed it, a shared container was available via `TestcontainersReuse`. The reliability gain outweighed the speed cost for critical integration paths
16. How do you test repository layer?
   - **Answer:**
      - I use `@DataJpaTest` which loads only JPA beans and uses an embedded or Testcontainers database
      - In CDMS, tested custom queries, pagination, and sorting - verifying derived queries generated correct SQL and returned expected results
      - `@DataJpaTest` auto-configures an in-memory database, `EntityManager`, and repository beans. For native queries and `@Query` annotations, tested with parameter binding using `@Param` and verified results against expected datasets loaded with `@Sql` or `TestEntityManager`. Projection interfaces were tested by asserting the returned DTO contained only the selected fields with correct values
17. How do you test service layer?
   - **Answer:**
      - I use JUnit and Mockito, mocking repository dependencies and verifying business logic
      - In the inventory service, mocked the repository, then tested validation rules, error handling, and edge cases like duplicate serials and invalid formats
      - Happy path: mock repository to return valid data, call the service method, assert the result. Validation failures: mock validation to throw exceptions, assert the correct error is propagated. Resource not found: mock `findById` to return empty, assert `EntityNotFoundException` is thrown. Concurrent modifications: use optimistic locking exceptions to test retry or conflict resolution. Every test is independent — no shared state between tests, each sets up its own mocks
18. How do you test controller layer?
   - **Answer:**
      - I use `@WebMvcTest` with MockMvc, mocking service layer beans
      - In our cold-chain API tests, verified GET endpoints returned correct JSON, POST with invalid data returned 400 with validation errors, and endpoints required proper authentication
      - Response status codes: assert 200 for successful GET, 201 for POST, 400 for invalid input, 404 for missing resources. Headers: verify `Content-Type: application/json` and pagination headers. JSON structure: use `jsonPath()` to assert field values, array lengths, and nested objects. Security filters: test with `@WithMockUser(roles = "ADMIN")` for role-based access, and manually set JWT tokens in headers for token-based authentication tests
19. How do you test Kafka consumer?
   - **Answer:**
      - I use `@EmbeddedKafka` from Spring Kafka test for lightweight tests or Testcontainers with a real Kafka container
      - In our cold-chain project, published test messages, verified the consumer processed them, and asserted expected data was stored in InfluxDB
      - Deserialization errors: publish malformed JSON, verify the consumer handles the error without crashing (dead letter queue or logged error). Poison pills: send a message that cannot be processed, verify the consumer retries a configured number of times then stops. Offset commit behavior: verify offsets are committed only after successful processing. Consumer retry logic: mock the downstream service to fail initially, verify the consumer retries before giving up or sending to a dead letter topic
20. How do you test scheduled jobs?
   - **Answer:**
      - I extract the job logic into a testable service and test that directly, plus an integration test that triggers the scheduler and verifies the outcome
      - In our inventory system, the reconciliation job was tested by calling the reconciliation service directly with test data
      - Do not test the scheduler framework itself — `@Scheduled` is Spring's responsibility. Focus on the logic: ensure idempotency by running the job twice with the same data and verifying no duplicate processing. Test failure handling by injecting exceptions and verifying the job logs errors and continues rather than crashing. Use `@Scheduled(fixedDelay = ...)` rather than `fixedRate` so jobs don't overlap. For integration tests, manually trigger the scheduled method and assert side effects
21. How do you test security rules?
   - **Answer:**
      - I use `@WithMockUser` for role-based access and manually set authentication for token-based tests
      - Tested that unauthenticated requests returned 401, wrong roles got 403, and correct roles could access endpoints
      - Method-level security with `@PreAuthorize`: test that a method annotated with `@PreAuthorize("hasRole('ADMIN')")` throws `AccessDeniedException` when called with a non-admin user. CSRF protection: verify POST/PUT/DELETE requests fail without a valid CSRF token when CSRF is enabled, and pass when the token is included. Test with `SecurityMockMvcRequestPostProcessors.csrf()` in MockMvc. Also test that security filters (JWT validation, rate limiting) are properly wired and reject invalid tokens
22. What is code coverage?
   - **Answer:**
      - Code coverage measures the percentage of code executed by tests
      - Used JaCoCo with Maven, targeting 70-80% coverage for service layers with clear exclusion rules for DTOs, configurations, and generated code
      - High coverage number does not guarantee good tests — a test that calls a method without assertions still counts toward coverage. Branch coverage is more meaningful than line coverage: it verifies both true and false paths of `if` statements are exercised. We focused branch coverage on business-critical code (validation, calculations, error handling) and excluded boilerplate: Lombok-generated getters/setters, configuration classes, DTOs, and repository interfaces with only derived queries
23. Is 100% coverage always good?
   - **Answer:**
      - No. 100% coverage can be misleading if tests only check simple paths with no meaningful assertions
      - We focused coverage on business-critical code - validation logic, calculations, error handling - not on boilerplate getters or configuration classes
      - Mutation testing is a stronger measure than line coverage: it introduces bugs (mutants) into the production code and checks if the test suite catches them. For example, changing `<` to `<=` or removing a method call. If tests still pass after the mutation, they are not testing that code path effectively. Tools like PIT (pitest) for Java automate this. Mutation score (percentage of mutants killed) indicates true test quality better than coverage percentage
24. What is regression testing?
   - **Answer:**
      - Regression testing ensures new changes do not break existing functionality
      - In our CI pipeline, the full test suite ran on every pull request - unit, integration, and smoke tests - catching regressions before they reached production
      - Automated regression testing reduces the fear of making changes, especially for critical paths like inventory validation where a regression could cause financial discrepancies. A comprehensive regression suite acts as a safety net: developers refactor and add features confidently because the suite catches unintended side effects. We categorized regression tests by priority — critical business flows ran first, full suite ran in parallel. Failed regressions blocked the merge
25. What is smoke testing?
   - **Answer:**
      - A quick check that the application starts and core functionality works
      - In our Jenkins pipeline, after deploying to staging, smoke tests checked the health endpoint returned 200, a simple API call worked, and the dashboard loaded - all within 60 seconds
      - Smoke tests catch obvious failures early — wrong configuration, missing dependencies, broken database connections — and prevent wasting time on deeper testing of a broken system. They run immediately after deployment as a gate: if smoke tests fail, the deployment is rolled back before any further testing. They are intentionally lightweight — no assertion on business logic, just verify the system is alive and responsive
26. What is API testing?
   - **Answer:**
      - Validates REST endpoints by sending HTTP requests and verifying status codes, headers, response body, and performance
      - Used Postman for manual testing and automated API tests in the CI pipeline using Spring Boot's TestRestTemplate
      - Test valid requests returning 200 with expected JSON body, invalid input returning 400 with validation error messages, unauthorized requests returning 401, forbidden access returning 403, missing resources returning 404. Edge cases: empty responses (204 No Content), large payloads (ensure no truncation), pagination parameters (limit, offset, sorting), and content negotiation (Accept header with unsupported media type returning 406)
27. What is Postman?
   - **Answer:**
      - A tool for developing and testing APIs through a graphical interface
      - Created Postman collections for each API module with environment variables, pre-request scripts for authentication, and test scripts for assertions
      - Collections were version-controlled in the repo and used for QA testing and CI via Newman (Postman's CLI runner). Newman runs collections as part of the Jenkins pipeline, failing the build if any request returns an unexpected status code or response. Environment variables allowed the same collection to run against local, staging, and production. Pre-request scripts handled token refresh so tests always had valid authentication
28. How do you test error cases?
   - **Answer:**
      - I intentionally send invalid inputs and assert the correct error response
      - In the inventory API, sent missing fields, invalid formats, and duplicate entries - verifying 400/422 status codes and checking error messages were actionable for partners
      - Client errors (4xx): test missing required fields, invalid field formats, duplicate entries, and unauthorized access. Server errors (5xx): mock downstream service failures or database exceptions and verify the API returns a 500 with a safe error message (no stack trace or internal details exposed). Timeout scenarios: mock a slow dependency and verify the caller receives a timeout error rather than hanging indefinitely. Use `assertThrows` or `@Test(expected = ...)` for expected exceptions in unit tests
29. How do you test performance changes?
   - **Answer:**
      - I run the same workload before and after changes, measuring response times, throughput, and resource usage
      - In CDMS optimization, ran stored procedures with realistic data, recorded execution plans, measured elapsed time - from 5 hours to under 12 minutes
      - Consistent test conditions are essential: same data volume, same server load, warm caches (run the query once to populate the buffer pool before measuring). Tools like JMeter simulate realistic concurrent load — not just single-user response time, but throughput under 50, 100, or 500 concurrent users. Compare p50, p95, and p99 latencies, not just averages. Monitor CPU, memory, and I/O during the test to ensure the optimization didn't shift the bottleneck elsewhere
30. How do you validate database optimization results?
   - **Answer:**
      - I compare execution plans before and after changes, checking index usage (seek vs scan), logical reads, and actual execution time with realistic data
      - For CDMS, captured plans, identified expensive operators, applied changes, and verified index seeks replaced scans
      - Test with production-like data volumes — a query that runs in 10ms on 1000 rows may take minutes on 10M rows. Validate correctness by comparing output: the optimized query must return the same result set as the original. Monitor optimized procedures in production for at least one full reporting cycle to confirm they handle edge cases (month-end, year-end, empty datasets) that synthetic tests may miss. Use `SET STATISTICS IO ON` and `SET STATISTICS TIME ON` to measure logical reads and CPU time before and after
