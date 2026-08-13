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
   - **Answer:** Unit testing tests individual components in isolation, mocking external dependencies. In our inventory project, I wrote JUnit + Mockito tests for the validation service - mocking the repository and testing validation logic with various serial number formats.
   - **If asked more:** I would explain the FIRST principles (Fast, Isolated, Repeatable, Self-validating, Timely) and the test pyramid: unit tests at the base (fast, numerous), integration above, and end-to-end at the top (slow, few).
2. What is integration testing?
   - **Answer:** Integration testing verifies that components work together correctly, often involving real databases or message brokers. In our cold-chain project, we used `@SpringBootTest` with Testcontainers to test the full Kafka consumer to InfluxDB flow.
   - **If asked more:** I would discuss how integration tests catch configuration issues that unit tests miss, and how we balanced unit vs integration coverage.
3. Difference between unit and integration test.
   - **Answer:** Unit tests test a single component in isolation (mocked dependencies), while integration tests test multiple components together (real dependencies). Our inventory validation service had unit tests for rules and integration tests for the full submission flow from controller to database.
   - **If asked more:** I would explain the test pyramid ratio: roughly 70% unit, 20% integration, 10% end-to-end.
4. What is JUnit?
   - **Answer:** JUnit is the standard testing framework for Java with annotations like `@Test` and assertions. We used JUnit 5 (Jupiter) for all tests with Maven Surefire plugin running them in the CI pipeline.
   - **If asked more:** I would discuss JUnit 5 features: parameterized tests for multiple inputs, nested tests for organization, and dynamic tests for data-driven scenarios.
5. What is Mockito?
   - **Answer:** Mockito creates mock objects for testing components in isolation. In our service-layer tests, we used `when(repository.findById(any())).thenReturn(Optional.of(entity))` to test business logic without needing a database.
   - **If asked more:** I would discuss verify for checking method calls, argument matchers, spy for partial mocking, and `@InjectMocks` for automatic mock injection.
6. What is mock?
   - **Answer:** A mock is a fake object that mimics a real dependency with expectations set during the test. In testing our inventory service, we mocked the repository to return predefined data, making tests fast, isolated, and deterministic.
   - **If asked more:** I would distinguish mocks from stubs: stubs provide predefined answers, mocks additionally verify that methods were called with expected parameters.
7. What is stub?
   - **Answer:** A stub returns predefined responses to method calls, making the test environment predictable. In our CDMS tests, we stubbed stored procedure calls to return known result sets, testing reporting logic without running the actual procedure.
   - **If asked more:** I would explain the difference: stubs focus on providing data for state testing, mocks focus on verifying interaction behavior.
8. Difference between mock and spy.
   - **Answer:** A mock creates a completely fake object with no real behavior, while a spy wraps a real object and allows overriding specific methods. We used mocks for repositories (completely fake) and spies when we needed some real behavior but wanted to stub others.
   - **If asked more:** I would recommend mocks by default and spies only when necessary, as spies can have side effects from calling real methods.
9. What is `@Mock`?
   - **Answer:** `@Mock` is a Mockito annotation that creates and injects a mock for the annotated field. We annotated repository dependencies with `@Mock` in test classes, using `MockitoAnnotations.openMocks(this)` or `@ExtendWith(MockitoExtension.class)` for initialization.
   - **If asked more:** I would explain how this reduces boilerplate compared to `Mockito.mock()` calls.
10. What is `@InjectMocks`?
    - **Answer:** `@InjectMocks` creates an instance of the annotated class and injects mocks into its dependencies. In our service tests, the service class had `@InjectMocks`, and Mockito automatically injected the `@Mock` repositories, eliminating manual constructor calls.
    - **If asked more:** I would discuss how Mockito handles injection - constructor preferred, then setter, then field - and the importance of a single constructor.
11. What is `@MockBean`?
    - **Answer:** `@MockBean` adds a mock to the Spring application context, replacing any existing bean. In our `@WebMvcTest` controller tests, we used `@MockBean` to mock service layer beans while testing only the controller layer.
    - **If asked more:** I would explain the difference from `@Mock`: `@MockBean` affects the Spring context (slower but necessary for Spring slice tests), while `@Mock` is for plain unit tests.
12. What is `@SpringBootTest`?
    - **Answer:** `@SpringBootTest` loads the full Spring Boot application context for integration testing. In our cold-chain project, we used it with Testcontainers to test the full flow from controller to Kafka to InfluxDB, verifying Spring bean wiring and configuration.
    - **If asked more:** I would discuss `webEnvironment = RANDOM_PORT` for real HTTP testing with TestRestTemplate and `@ActiveProfiles("test")` for test-specific configuration.
13. What is `@WebMvcTest`?
    - **Answer:** `@WebMvcTest` loads only the web layer for focused controller testing. We used it with `@MockBean` for services, testing endpoint mappings, request validation, status codes, and error responses without loading the full context.
    - **If asked more:** I would explain how it is faster than `@SpringBootTest` and how we used MockMvc for HTTP request assertions.
14. What is Testcontainers?
    - **Answer:** Testcontainers provides disposable Docker containers for integration testing. In our cold-chain project, we used it to spin up real Kafka and InfluxDB containers during tests, ensuring tests ran against the same versions as production.
    - **If asked more:** I would discuss how Testcontainers improves reliability over in-memory alternatives by using real dependencies, with `@Container` and `@Testcontainers` annotations for lifecycle management.
15. Why use Testcontainers?
    - **Answer:** Testcontainers ensures tests run against real databases, not in-memory simulations that behave differently. H2 did not support all MSSQL features our stored procedures used, so Testcontainers with a real MSSQL container caught compatibility issues earlier.
    - **If asked more:** I would discuss the tradeoff: slower (container startup) but more reliable. We ran them in CI only, not during local development for every change.
16. How do you test repository layer?
    - **Answer:** I use `@DataJpaTest` which loads only JPA beans and uses an embedded or Testcontainers database. In CDMS, we tested custom queries, pagination, and sorting - verifying derived queries generated correct SQL and returned expected results.
    - **If asked more:** I would discuss testing native queries and `@Query` annotations with parameter binding and projection interfaces.
17. How do you test service layer?
    - **Answer:** I use JUnit and Mockito, mocking repository dependencies and verifying business logic. In the inventory service, we mocked the repository, then tested validation rules, error handling, and edge cases like duplicate serials and invalid formats.
    - **If asked more:** I would discuss testing happy path, validation failures, resource not found, concurrent modifications, and keeping tests independent and fast.
18. How do you test controller layer?
    - **Answer:** I use `@WebMvcTest` with MockMvc, mocking service layer beans. In our cold-chain API tests, we verified GET endpoints returned correct JSON, POST with invalid data returned 400 with validation errors, and endpoints required proper authentication.
    - **If asked more:** I would discuss testing response status codes, headers, JSON structure, and security filters (JWT, role-based access).
19. How do you test Kafka consumer?
    - **Answer:** I use `@EmbeddedKafka` from Spring Kafka test for lightweight tests or Testcontainers with a real Kafka container. In our cold-chain project, we published test messages, verified the consumer processed them, and asserted expected data was stored in InfluxDB.
    - **If asked more:** I would discuss testing deserialization errors, poison pills, offset commit behavior, and consumer retry logic.
20. How do you test scheduled jobs?
    - **Answer:** I extract the job logic into a testable service and test that directly, plus an integration test that triggers the scheduler and verifies the outcome. In our inventory system, the reconciliation job was tested by calling the reconciliation service directly with test data.
    - **If asked more:** I would discuss avoiding testing the scheduler framework itself and focusing on the logic being idempotent and handling failures gracefully.
21. How do you test security rules?
    - **Answer:** I use `@WithMockUser` for role-based access and manually set authentication for token-based tests. We tested that unauthenticated requests returned 401, wrong roles got 403, and correct roles could access endpoints.
    - **If asked more:** I would discuss testing method-level security (`@PreAuthorize`) and CSRF protection.
22. What is code coverage?
    - **Answer:** Code coverage measures the percentage of code executed by tests. We used JaCoCo with Maven, targeting 70-80% coverage for service layers with clear exclusion rules for DTOs, configurations, and generated code.
    - **If asked more:** I would discuss that high coverage does not guarantee good tests - we focused on branch coverage for business logic rather than line coverage numbers.
23. Is 100% coverage always good?
    - **Answer:** No. 100% coverage can be misleading if tests only check simple paths with no meaningful assertions. We focused coverage on business-critical code - validation logic, calculations, error handling - not on boilerplate getters or configuration classes.
    - **If asked more:** I would explain that mutation testing is a stronger measure than line coverage: it checks if tests actually catch bugs when the code is mutated.
24. What is regression testing?
    - **Answer:** Regression testing ensures new changes do not break existing functionality. In our CI pipeline, the full test suite ran on every pull request - unit, integration, and smoke tests - catching regressions before they reached production.
    - **If asked more:** I would discuss how automated regression testing reduces the fear of making changes, especially for critical paths like inventory validation where a regression could cause financial discrepancies.
25. What is smoke testing?
    - **Answer:** Smoke testing is a quick check that the application starts and core functionality works. In our Jenkins pipeline, after deploying to staging, smoke tests checked the health endpoint returned 200, a simple API call worked, and the dashboard loaded - all within 60 seconds.
    - **If asked more:** I would explain that smoke tests catch obvious failures early (wrong config, missing dependencies) and prevent wasting time on deeper testing of a broken system.
26. What is API testing?
    - **Answer:** API testing validates REST endpoints by sending HTTP requests and verifying status codes, headers, response body, and performance. We used Postman for manual testing and automated API tests in the CI pipeline using Spring Boot's TestRestTemplate.
    - **If asked more:** I would discuss testing valid requests (200), invalid input (400), unauthorized (401/403), not found (404), and edge cases like empty responses and large payloads.
27. What is Postman?
    - **Answer:** Postman is a tool for developing and testing APIs through a graphical interface. We created Postman collections for each API module with environment variables, pre-request scripts for authentication, and test scripts for assertions.
    - **If asked more:** I would discuss how collections were version-controlled in the repo and used for QA testing and CI via Newman.
28. How do you test error cases?
    - **Answer:** I intentionally send invalid inputs and assert the correct error response. In the inventory API, we sent missing fields, invalid formats, and duplicate entries - verifying 400/422 status codes and checking error messages were actionable for partners.
    - **If asked more:** I would discuss testing both client errors (4xx) and server errors (5xx), including database failures (mocked exceptions) and timeout scenarios.
29. How do you test performance changes?
    - **Answer:** I run the same workload before and after changes, measuring response times, throughput, and resource usage. In CDMS optimization, I ran stored procedures with realistic data, recorded execution plans, measured elapsed time - from 5 hours to under 12 minutes.
    - **If asked more:** I would discuss consistent test conditions (same data, server load, warm caches) and tools like JMeter for load testing.
30. How do you validate database optimization results?
    - **Answer:** I compare execution plans before and after changes, checking index usage (seek vs scan), logical reads, and actual execution time with realistic data. For CDMS, I captured plans, identified expensive operators, applied changes, and verified index seeks replaced scans.
    - **If asked more:** I would discuss testing with production-like data volumes, validating correctness by comparing output, and monitoring optimized procedures in production for at least one reporting cycle.

