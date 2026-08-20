# Spring Boot Questions

## Questions

1. What is Spring Framework?
2. What is Spring Boot?
3. Why use Spring Boot?
4. Difference between Spring and Spring Boot.
5. What is auto-configuration?
6. How does Spring Boot auto-configuration work?
7. What is starter dependency?
8. What is embedded server?
9. What is IoC?
10. What is dependency injection?
11. Types of dependency injection.
12. Constructor injection vs field injection.
13. What is a Spring bean?
14. What is bean scope?
15. Difference between singleton and prototype scope.
16. What is application context?
17. What is bean lifecycle?
18. What is `@Component`?
19. Difference between `@Component`, `@Service`, `@Repository`, and `@Controller`.
20. What is `@SpringBootApplication`?
21. What does `@EnableAutoConfiguration` do?
22. What is `@Configuration`?
23. What is `@Bean`?
24. Difference between `@Bean` and `@Component`.
25. What is `application.properties`?
26. Difference between `application.properties` and `application.yml`.
27. What are profiles?
28. How do you externalize configuration?
29. What is `@Value`?
30. What is `@ConfigurationProperties`?
31. What is actuator?
32. Which actuator endpoints are useful in production?
33. How do you secure actuator endpoints?
34. What is Spring Boot DevTools?
35. How do you handle exceptions globally?
36. What is `@ControllerAdvice`?
37. What is `@ExceptionHandler`?
38. What is validation in Spring Boot?
39. What is `@Valid`?
40. Difference between `@Valid` and `@Validated`.
41. What is scheduling in Spring Boot?
42. What is `@Scheduled`?
43. Difference between fixed rate and fixed delay.
44. What is cron expression?
45. How do you configure scheduled task thread pool?
46. How do you prevent scheduled jobs from running on all pods?
47. What is async processing?
48. What is `@Async`?
49. How do you configure async executor?
50. What is caching in Spring Boot?
51. What is `@Cacheable`?
52. Difference between `@Cacheable`, `@CachePut`, and `@CacheEvict`.
53. How do you use Redis cache with Spring Boot?
54. How do you write REST APIs in Spring Boot?
55. How do you version APIs?
56. How do you document APIs?
57. What is Swagger/OpenAPI?
58. What is Spring Boot testing?
59. What is `@SpringBootTest`?
60. What is `@WebMvcTest`?
61. Difference between the traditional Java Singleton design pattern and the Spring singleton bean scope.
62. Explain in detail `@Component`, `@Bean`, `@Configuration`, `@Repository`, `@Service`, `@RestController`, and `@Controller`. Also, why are there dedicated annotations?

---

## Answers

1. What is Spring Framework?
   - **Answer:**
      - Spring Framework is a lightweight, modular framework for building Java enterprise applications using IoC and dependency injection
      - Spring is used to manage service layers, repository injection, and transaction boundaries for database operations
      - Modular architecture: Core Container, Data Access, Web, AOP — specific modules can be chosen for REST APIs instead of pulling the entire framework
2. What is Spring Boot?
   - **Answer:**
      - Spring Boot is Spring's opinionated auto-configuration layer that removes boilerplate setup
      - By adding `spring-boot-starter-web`, `spring-boot-starter-data-jpa`, and `spring-boot-starter-security`, everything is pre-configured for Tomcat, Hibernate, and security defaults
      - `@SpringBootApplication` combines `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan` — these annotations are used across microservices
3. Why use Spring Boot?
   - **Answer:**
      - Spring Boot cuts development time by providing embedded Tomcat, auto-configuration, production-ready features like Actuator, and easy externalized configuration
      - A Spring Boot JAR can be deployed on EC2 without needing to manage a separate Tomcat install
      - Concrete time savings: it is possible to go from a new starter project to a deployed API with JWT security and JPA in under an hour, which is critical for fast client demos
4. Difference between Spring and Spring Boot.
   - **Answer:**
      - Spring is the core framework providing DI, AOP, and MVC; Spring Boot adds auto-configuration, embedded servers, and starter dependencies on top
      - Spring Boot eliminates the need to manually configure DispatcherServlet or Hibernate — Boot handles that automatically
      - Spring Boot still uses Spring under the hood but adds `spring.factories` auto-configuration classes — these can be customized with `application.yml` properties
5. What is auto-configuration?
   - **Answer:**
      - Auto-configuration is Spring Boot's ability to automatically configure beans based on dependencies in the classpath
      - For example, adding `spring-boot-starter-data-jpa` automatically configures the DataSource, EntityManager, and TransactionManager
      - Internally, `@ConditionalOnClass`, `@ConditionalOnMissingBean`, and `@ConditionalOnProperty` control which auto-configurations activate — these can be used to conditionally configure Redis cache only when the Redis dependency is present
6. How does Spring Boot auto-configuration work?
   - **Answer:**
      - Spring Boot scans `META-INF/spring.factories` for `EnableAutoConfiguration` classes and applies `@Conditional` checks to decide which beans to create
      - When adding `spring-boot-starter-web`, Boot automatically configures Jackson, DispatcherServlet, and error handling without any XML
      - The `AutoConfigurationImportSelector` mechanism reads auto-configuration classes, `@Conditional` annotations prevent conflicting beans, and auto-configuration can be debugged using the `--debug` flag and Actuator's `/conditions` endpoint
7. What is starter dependency?
   - **Answer:**
      - A starter dependency is a curated Maven POM that bundles related libraries
      - For example, `spring-boot-starter-validation` brings in Hibernate Validator and its transitive dependencies, so `@NotBlank` and `@Pattern` can be used on DTOs immediately
      - Starters follow the naming convention `spring-boot-starter-*`, and a custom starter can be created by defining auto-configuration classes and a `spring.factories` file
8. What is embedded server?
   - **Answer:**
      - An embedded server is a web server bundled inside the application JAR
      - Spring Boot JARs run on embedded Tomcat by default, which can be configured by setting `server.port`, `server.servlet.context-path`, and SSL properties in `application.yml`
      - Tomcat is the default; Jetty and Undertow are alternatives — Undertow can be used for better performance in high-throughput scenarios
9. What is IoC?
   - **Answer:**
      - Inversion of Control means the framework controls object creation and lifecycle instead of the developer
      - The Spring IoC container manages service, repository, and security filter beans — they are defined and injected where needed
      - The IoC container has two main types: BeanFactory (lazy, lightweight) and ApplicationContext (eager, full-featured) — `ApplicationContext.getBean()` can be used in rare cases to dynamically fetch beans based on runtime configuration
10. What is dependency injection?
   - **Answer:**
      - Dependency Injection is when Spring provides required objects instead of the class creating them
      - Constructor injection is recommended for service and repository dependencies, which makes testing easier with mocks and keeps dependencies immutable and explicit
      - There are three injection types with a preference for constructor injection — Spring resolves circular dependencies using a three-level cache in singleton scope
11. Types of dependency injection.
   - **Answer:**
      - There are three types: constructor injection, setter injection, and field injection
      - In production code, constructor injection should be strictly used because it enforces immutability and makes testing straightforward
      - Field injection is discouraged since it hides dependencies and breaks testability
      - Spring validates dependencies at startup; field injection can cause NullPointerException in tests because mocks cannot be injected through the constructor — `@RequiredArgsConstructor` from Lombok can reduce boilerplate while keeping constructor injection
12. Constructor injection vs field injection.
   - **Answer:**
      - Constructor injection makes dependencies explicit, immutable, and mandatory
      - Constructor injection should be used throughout service layers
      - Field injection hides dependencies and makes unit tests harder because mocks cannot be injected through the constructor easily
      - The Spring team recommends constructor injection, and Lombok's `@RequiredArgsConstructor` keeps the code clean in REST controllers and service classes
13. What is a Spring bean?
   - **Answer:**
      - A Spring bean is a Java object managed by the Spring IoC container
      - Classes annotated with `@Service`, `@Repository`, or `@Component` become beans
      - For example, a service class can be a bean with singleton scope, reused across multiple API calls
      - Beans follow naming conventions and are registered via `@ComponentScan` or `@Bean` methods — `@Scope("prototype")` can be used for stateful validation contexts
14. What is bean scope?
   - **Answer:**
      - Bean scope determines the lifecycle and visibility of a bean
      - Singleton scope creates one instance per container, which is used for all service beans
      - Prototype creates a new instance every request
      - Prototype scope can be used for one-time-use objects that carry mutable state
      - Web-aware scopes include request and session — singleton scope issues can occur with `@Async` methods where the proxy behavior requires public non-static methods
15. Difference between singleton and prototype scope.
   - **Answer:**
      - Singleton creates one instance shared across the whole application, which is used for all stateless services
      - Prototype creates a new instance every time it is injected or requested, which should be used sparingly for objects with request-specific state
      - Performance trade-off: singleton saves memory but can have thread-safety issues, while prototype avoids state conflicts but increases GC pressure — thread-safety issues in singleton services can be resolved by removing instance variables
16. What is application context?
   - **Answer:**
      - ApplicationContext is the Spring IoC container that manages bean lifecycle, event propagation, and internationalization
      - `AnnotationConfigApplicationContext` is created behind the scenes by `SpringApplication.run()`, and `ApplicationContextAware` can be used to access beans programmatically
      - ApplicationContext has a hierarchy and differs from BeanFactory by providing event publishing, i18n, and eager bean initialization — `ConfigurableApplicationContext.close()` can be used in test `@AfterClass` methods to clean up the context
17. What is bean lifecycle?
   - **Answer:**
      - Bean lifecycle goes through: instantiation, property population, initialization callbacks (`@PostConstruct`, `InitializingBean`), bean is ready, then destruction callbacks (`@PreDestroy`, `DisposableBean`)
      - `@PostConstruct` can be used to load reference data from a database into a cache after the bean is initialized
      - The full sequence includes BeanPostProcessors, `@PostConstruct`, `afterPropertiesSet()`, and custom init-method — `BeanPostProcessor` can be used to log bean initialization times for performance monitoring
18. What is `@Component`?
   - **Answer:**
      - `@Component` is a stereotype annotation that marks a class as a Spring-managed bean
      - Its specializations like `@Service` and `@Repository` are typically preferred over plain `@Component`, since they add semantic meaning and enable persistence exception translation
      - `@ComponentScan` discovers these annotations during classpath scanning, custom stereotype annotations can be created, and `@Component` can be used for utility classes like `JwtUtil` that don't fit the service/repository pattern
19. Difference between `@Component`, `@Service`, `@Repository`, and `@Controller`.
   - **Answer:**
      - All four register Spring beans, but `@Service` is a service layer specialization, `@Repository` enables persistence exception translation, and `@Controller` marks web controllers
      - `@Service` is used for business logic, `@Repository` for DAO layers, and `@Controller` for REST endpoints
      - `@Repository` adds `PersistenceExceptionTranslationPostProcessor` to convert SQLExceptions into Spring's `DataAccessException`, which handles exception translation for database constraint violations
20. What is `@SpringBootApplication`?
   - **Answer:**
      - `@SpringBootApplication` is a convenience annotation combining `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan`
      - This annotation is typically placed on the main class, enabling auto-configuration and scanning all beans under the base package
      - Specific auto-configurations can be excluded using the `exclude` parameter — for example, `@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})` when setting up a test that doesn't need database connectivity
21. What does `@EnableAutoConfiguration` do?
   - **Answer:**
      - `@EnableAutoConfiguration` enables Spring Boot's auto-configuration mechanism that creates beans based on classpath dependencies
      - When `spring-boot-starter-web` is added, it automatically configures `DispatcherServlet`, `Jackson ObjectMapper`, and error handling through `ErrorMvcAutoConfiguration`
      - It is backed by `AutoConfigurationImportSelector` which reads `spring.factories` files — `spring.autoconfigure.exclude` in `application.properties` can be used to disable a conflicting auto-configuration
22. What is `@Configuration`?
   - **Answer:**
      - `@Configuration` marks a class as a source of bean definitions using `@Bean` methods
      - A configuration class annotated with `@Configuration` can define `RedisTemplate` and `CacheManager` beans with custom serialization settings
      - There is a difference between `@Configuration` with `@Bean` vs `@Component` with `@Autowired` — Spring proxies `@Configuration` classes using CGLIB to ensure singleton bean semantics, so calling a `@Bean` method inside a `@Configuration` class returns the cached singleton rather than creating a new instance
23. What is `@Bean`?
   - **Answer:**
      - `@Bean` is a method-level annotation that tells Spring to register the returned object as a bean in the container
      - `@Bean` in a `@Configuration` class can create managed instances like `KafkaTemplate`, `RedisTemplate`, and `RestTemplate` with project-specific configurations
      - `@Bean` supports lifecycle callbacks using `initMethod` and `destroyMethod` — for example, a `KafkaTemplate` bean can be configured with custom serializer properties and retry configuration for data pipelines
24. Difference between `@Bean` and `@Component`.
   - **Answer:**
      - `@Bean` is used in `@Configuration` classes for third-party or manually configured beans; `@Component` is for your own classes
      - `@Bean` is used for framework classes like `RedisTemplate`, `KafkaTemplate`, and `PasswordEncoder`, while application services use `@Component` derivatives
      - `@Bean` gives full control over instantiation and configuration, while `@Component` relies on classpath scanning — `@Bean` is preferred when passing constructor arguments like database connection pools or Redis connection factory
25. What is `application.properties`?
   - **Answer:**
      - `application.properties` is Spring Boot's default configuration file for externalizing settings
      - Database connection properties, Hibernate DDL strategy, and server port are configured in this file, keeping config separate from code
      - Property loading has a specific order, profile-specific properties like `application-dev.properties` override defaults, and `@Value` and `@ConfigurationProperties` can bind these values to Java objects
26. Difference between `application.properties` and `application.yml`.
   - **Answer:**
      - Both serve the same purpose but `.properties` is flat key-value while `.yml` uses hierarchical indentation
      - `.yml` is often preferred for nested configs like `spring.datasource.*` and `spring.jpa.*`, reducing duplication
      - YAML supports list syntax for cleaner multi-profile configuration, but YAML does not support `@PropertySource` natively — a workaround is to use `application.yml` as the primary config source
27. What are profiles?
   - **Answer:**
      - Profiles allow environment-specific bean definitions and configurations
      - Profile-specific files like `application-dev.yml`, `application-staging.yml`, and `application-prod.yml` can be defined, each with different database URLs, log levels, and cache settings
      - `spring.profiles.active` is set via environment variable or JVM argument, and `@Profile("dev")` can conditionally load mock beans for local testing
28. How do you externalize configuration?
   - **Answer:**
      - Spring Boot externalizes config through application properties, environment variables, command-line arguments, and `@ConfigurationProperties`
      - Database credentials can be kept in environment variables and accessed via `${DATABASE_URL}` in `application.yml`
      - `PropertySource` defines loading order; `@ConfigurationProperties(prefix = "inventory.validation")` can bind nested properties to a POJO, which provides type safety and IDE validation over raw `@Value` usage
29. What is `@Value`?
   - **Answer:**
      - `@Value` injects a property value from configuration files into a field or parameter
      - `@Value("${report.batch-size:1000}")` can configure batch processing size for stored procedure execution, with a sensible default
      - `@Value` supports SpEL for dynamic evaluation, but `@ConfigurationProperties` is preferred for grouped properties because it provides type safety and IDE validation — `@Value` is simpler for individual property injection
30. What is `@ConfigurationProperties`?
   - **Answer:**
      - `@ConfigurationProperties` binds entire property hierarchies to strongly-typed Java objects
      - A properties class annotated with `@ConfigurationProperties(prefix = "inventory")` can hold validation thresholds, batch sizes, and retry counts
      - `@EnableConfigurationProperties` activates binding, nested POJO binding is supported, and `@Validated` with JSR-303 annotations can validate configuration values at startup, preventing production issues from misconfigured properties
31. What is actuator?
   - **Answer:**
      - Actuator provides production-ready HTTP endpoints for monitoring and managing Spring Boot applications
      - `/health`, `/metrics`, and `/info` endpoints can be enabled to monitor API health and response times without building custom monitoring endpoints
      - Actuator has different endpoint categories exposed via `management.endpoints.web.exposure.include`, and it can be extended by adding a custom `HealthIndicator` that checks external service connectivity and database availability
32. Which actuator endpoints are useful in production?
   - **Answer:**
      - `/health` for liveness checks, `/metrics` for JVM and request metrics, `/info` for application metadata, and `/loggers` for runtime log level changes
      - In production, only `/health` and `/info` should be exposed publicly while others are kept behind the firewall for security
      - `/metrics` integrates with Prometheus using `micrometer-registry-prometheus`, and `/loggers` helps debug pipelines by enabling DEBUG logging for specific components without restarting the JAR
33. How do you secure actuator endpoints?
   - **Answer:**
      - Actuator endpoints can be secured by restricting exposure, using separate management ports, and applying Spring Security
      - Setting `management.endpoints.web.exposure.exclude=*` and only exposing specific endpoints with `include=health,info`, then adding `management.server.port=8081` restricts access
      - Custom Actuator endpoints can be secured with `@RolesAllowed`, and a separate security filter chain can be configured for the management port with IP whitelist through a custom `WebSecurityConfigurerAdapter`
34. What is Spring Boot DevTools?
   - **Answer:**
      - DevTools provides automatic restart, live reload, and remote debugging for development
      - DevTools can be used during local development for automatic restart when Java files change, and the LiveReload server can refresh the Swagger UI in the browser
      - DevTools uses two classloaders for fast restart, resources can be excluded from restart, and DevTools should never be enabled in production — it can leak sensitive information and causes performance overhead
35. How do you handle exceptions globally?
   - **Answer:**
      - Exceptions are handled globally using `@ControllerAdvice` combined with `@ExceptionHandler` methods
      - A `GlobalExceptionHandler` class can catch `DataAccessException`, `MethodArgumentNotValidException`, and custom business exceptions, returning consistent JSON error responses with proper HTTP status codes
      - The exception handling hierarchy maps specific exceptions to handlers — database constraint violations can be mapped to user-friendly messages and stack traces logged selectively using MDC to include request IDs in logs
36. What is `@ControllerAdvice`?
   - **Answer:**
      - `@ControllerAdvice` is a global interceptor for controllers that enables cross-cutting exception handling, data binding, and model attributes
      - A single `@ControllerAdvice` class can handle validation errors, authentication failures, and database constraint violations across all endpoints
      - `@RestControllerAdvice` is the REST-specific variant combining `@ControllerAdvice` + `@ResponseBody` — the response body can be customized with `ErrorResponse` DTOs containing error code, message, timestamp, and trace ID for debugging
37. What is `@ExceptionHandler`?
   - **Answer:**
      - `@ExceptionHandler` defines a method to handle specific exceptions thrown by controllers
      - Methods like `handleValidationException(MethodArgumentNotValidException)` returning 400 with field-level errors, and `handleResourceNotFound(ResourceNotFoundException)` returning 404 can be defined
      - Exception handler priority determines which handler matches when multiple could apply, multiple exception types can be handled in one method, and `ResponseEntity` provides fine-grained control over response headers and status codes
38. What is validation in Spring Boot?
   - **Answer:**
      - Validation in Spring Boot uses Bean Validation API (JSR-380) with annotations like `@NotNull`, `@Size`, and `@Pattern` on DTO fields
      - Incoming data can be validated with `@Pattern(regexp = "^[A-Z0-9]+$")` and mandatory fields checked with `@NotBlank`
      - Validation integrates with `@Valid` in `@RequestBody` parameters, custom validation annotations can be created for business-specific rules, and validation errors are automatically handled by `MethodArgumentNotValidException`
39. What is `@Valid`?
   - **Answer:**
      - `@Valid` triggers JSR-380 bean validation on request bodies, query parameters, or path variables
      - Annotating `@RequestBody` with `@Valid` in the controller automatically validates all field constraints before the service method is called
      - `@Valid` is the standard JSR-380 annotation while `@Validated` is Spring's variant that adds support for validation groups — validation groups can differentiate between create vs update operations
40. Difference between `@Valid` and `@Validated`.
   - **Answer:**
      - `@Valid` is standard JSR-380 that triggers validation; `@Validated` is Spring's variant that adds support for validation groups
      - `@Validated` with groups like `OnCreate.class` and `OnUpdate.class` can apply different rules for POST and PUT endpoints
      - Validation groups are defined using empty interfaces, and group validation integrates with `@RequestParam` and `@PathVariable` using `@Validated` at the class level
41. What is scheduling in Spring Boot?
   - **Answer:**
      - Scheduling in Spring Boot uses `@EnableScheduling` and `@Scheduled` annotations to run tasks periodically
      - A nightly stored procedure execution can be scheduled at 2 AM using a cron expression to refresh report data without manual intervention
      - The `TaskScheduler` abstraction manages execution, thread pools should be configured for scheduled tasks, and handling failures using try-catch blocks prevents silent task termination
42. What is `@Scheduled`?
   - **Answer:**
      - `@Scheduled` marks a method to be executed on a schedule
      - `@Scheduled(fixedDelay = 300000)` can run a reconciliation method every 5 minutes after the previous run completes, ensuring no overlapping executions
      - Three modes are available: `fixedRate`, `fixedDelay`, and `cron` — `cron = "0 0 2 * * ?"` can schedule nightly ETL batches without needing any external job scheduler initially
43. Difference between fixed rate and fixed delay.
   - **Answer:**
      - `fixedRate` triggers every N milliseconds regardless of whether the previous execution finished; `fixedDelay` waits N milliseconds after the previous execution completes
      - `fixedDelay` is preferred for batch jobs because overlapping runs would corrupt data
      - `fixedRate` risks thread starvation when tasks take longer than the interval — this can be mitigated by configuring a custom `ThreadPoolTaskScheduler` with a bounded queue
44. What is cron expression?
   - **Answer:**
      - A cron expression defines schedule using six or seven fields: second, minute, hour, day-of-month, month, day-of-week, and optional year
      - `0 0/15 * * * ?` can run data aggregation every 15 minutes without needing a separate cron job on the server
      - Cron syntax uses `?` for no specific value and `*` for all values — a common mistake is forgetting that cron runs in the server's timezone, so the `zone` attribute in `@Scheduled` can be used to adjust it
45. How do you configure scheduled task thread pool?
   - **Answer:**
      - By default, `@Scheduled` uses a single-threaded executor
      - A `ThreadPoolTaskScheduler` bean with `pool-size=5` and a custom `ErrorHandler` that logs failures without killing the scheduler can prevent a single failed task from blocking the others
      - A custom `SchedulingConfigurer` with `@Configuration` sets the thread pool, and the trade-off is between a shared thread pool vs dedicated pools for critical vs non-critical scheduled jobs
46. How do you prevent scheduled jobs from running on all pods?
   - **Answer:**
      - When running multiple instances, scheduled jobs need a coordination mechanism
      - ShedLock with Redis can ensure only one pod executes a scheduled job at a time by acquiring a distributed lock
      - ShedLock locks are persisted in a database table or Redis, lock duration is configurable, and ShedLock integrates with `@Scheduled` using `@SchedulerLock(name = "nightlyReport")` for batch jobs
47. What is async processing?
   - **Answer:**
      - Async processing allows methods to run in a separate thread without blocking the caller
      - Async processing can be used for sending email notifications when certain events occur, so the API response isn't delayed by the email SMTP call
      - Async differs from reactive and parallel processing — async improves API responsiveness but carries threading implications including the risk of thread pool exhaustion if not configured properly
48. What is `@Async`?
   - **Answer:**
      - `@Async` marks a method for execution in a separate thread
      - An `@Async`-annotated method for writing audit records won't block the main API response, improving perceived performance
      - `@Async` requires `@EnableAsync` and only works on public methods called from outside the class — self-invocation bypasses the proxy — and `CompletableFuture` return types can be used to handle async results
49. How do you configure async executor?
   - **Answer:**
      - A `ThreadPoolTaskExecutor` bean is configured with custom core pool size, max pool size, queue capacity, and rejection policy
      - Setting `corePoolSize=10`, `maxPoolSize=25`, and `CallerRunsPolicy` can handle spikes in data processing without losing tasks
      - The `AsyncConfigurer` interface allows custom configuration, `AsyncUncaughtExceptionHandler` handles uncaught exceptions in async methods, and queue capacity impacts memory during traffic bursts
50. What is caching in Spring Boot?
   - **Answer:**
      - Caching stores frequently accessed data in memory to reduce database load and improve response times
      - Redis cache can store frequently accessed configuration data and entity mappings, reducing repeated database queries for data that rarely changed
      - `@EnableCaching` activates the cache abstraction layer, cache managers include InMemory and Redis, and cache hit ratios in production can be measured using Actuator metrics to tune TTL values
51. What is `@Cacheable`?
   - **Answer:**
      - `@Cacheable` stores the method result in cache and returns it on subsequent calls with the same arguments
      - Applying `@Cacheable("deviceConfigs")` on a method that fetches mappings can reduce database round trips from hundreds per minute to only a few cache misses
      - Cache key generation uses the `key` attribute with SpEL, conditional caching is supported via `condition` and `unless`, and the cache can be proactively invalidated when underlying data is updated
52. Difference between `@Cacheable`, `@CachePut`, and `@CacheEvict`.
   - **Answer:**
      - `@Cacheable` reads and stores; `@CachePut` always executes and updates the cache; `@CacheEvict` removes entries
      - `@CachePut` can be used on update methods to refresh cached data, and `@CacheEvict(allEntries = true)` on data reload endpoints to clear stale entries before repopulation
      - `@Caching` combines multiple cache annotations on a single method, and `@CacheEvict(beforeInvocation = true)` evicts cache before method execution when failure should still result in cache being cleared
53. How do you use Redis cache with Spring Boot?
   - **Answer:**
      - Add `spring-boot-starter-data-redis`, configure Redis connection in `application.yml`, define a `RedisCacheManager` bean, and use `@Cacheable` on service methods
      - Redis can be configured with TTL for cache entries and `RedisTemplate` can be used for direct operations
      - `RedisCacheManager` handles cache abstraction while `RedisTemplate` provides direct Redis operations — serialization can be configured with Jackson2JsonRedisSerializer for JSON storage, and Redis connection failures can be handled by falling back to database queries
54. How do you write REST APIs in Spring Boot?
   - **Answer:**
      - Use `@RestController` with `@RequestMapping` for class-level mapping and `@GetMapping`, `@PostMapping`, etc. for HTTP methods
      - A controller can have endpoints like `GET /api/partners`, `POST /api/partners/sync`, and `PUT /api/partners/{id}`, returning `ResponseEntity` for status control
      - REST best practices include proper HTTP methods, status codes, request/response DTOs, content negotiation, and API versioning using URL path prefix like `/v1/`
55. How do you version APIs?
   - **Answer:**
      - APIs can be versioned through the URL path prefix like `/api/v1/partners`
      - Backward compatibility can be maintained by keeping v1 endpoints while adding new fields in v2 request/response DTOs, allowing consumers to migrate gradually without breaking their integrations
      - Other versioning strategies include header-based (`Accept-version`), query parameter (`?version=1`), and content negotiation — URL path versioning is often chosen for simplicity since it's explicit in logs and easy to route
56. How do you document APIs?
   - **Answer:**
      - REST APIs are documented using Swagger/OpenAPI 3.0 with `springdoc-openapi` library
      - `@Operation` and `@ApiResponse` annotations on controllers describe endpoints, request bodies, and error responses, making it easy for frontend teams to integrate without constant back-and-forth
      - The Swagger UI can be customized with bearer token support for JWT, endpoints grouped by tags, and `springdoc.swagger-ui.enabled=false` in production to expose docs only on staging environments
57. What is Swagger/OpenAPI?
   - **Answer:**
      - Swagger/OpenAPI is a specification for documenting REST APIs in a machine-readable format (JSON/YAML)
      - Integrating `springdoc-openapi-starter-webmvc-ui` auto-generates OpenAPI docs from `@RestController` annotations and provides an interactive Swagger UI at `/swagger-ui.html`
      - OpenAPI 3.0 differs from Swagger 2.0 in structure and schema support — reusable components (schemas, security schemes) can be defined in the OpenAPI config, and the generated docs help QA write automated tests using the OpenAPI spec
58. What is Spring Boot testing?
   - **Answer:**
      - Spring Boot testing uses `@SpringBootTest` for full context integration tests and slice tests for focused layers
      - Integration tests can load the full Spring context and test REST endpoints from HTTP request to database persistence using an H2 in-memory database
      - The testing pyramid guides test distribution, `@TestContainers` enables testing with real databases, and `@DirtiesContext` cleans up state between tests that modify the application context
59. What is `@SpringBootTest`?
   - **Answer:**
      - `@SpringBootTest` loads the complete Spring application context for integration testing
      - `@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)` with `TestRestTemplate` can send real HTTP requests to the API and verify response status, headers, and body
      - Properties can be overridden with `@TestPropertySource` or the `properties` attribute, `@MockBean` replaces external dependencies, and tests can be configured to use an embedded H2 database instead of the production database
60. What is `@WebMvcTest`?
   - **Answer:**
      - `@WebMvcTest` loads only the web layer for controller unit testing — it does not load services, repositories, or security
      - `@WebMvcTest(PartnerController.class)` with `@MockBean` for the service dependency can test request mapping, validation, and response serialization
      - `@WebMvcTest` is faster than `@SpringBootTest` because it avoids scanning all beans in the context, `MockMvc` performs requests and assertions, and it isolates controller logic from service and repository layers
61. Difference between the traditional Java Singleton design pattern and the Spring singleton bean scope.
   - **Answer:**
      - The GoF Singleton pattern guarantees a class has exactly one instance and exposes it via a private constructor and static `getInstance()`
      - Spring's singleton bean scope means the IoC container creates exactly one bean instance per bean name per container, but the class itself is a normal class — other instances can still be created with `new`
      - Spring manages the instance lifecycle (creation, dependency wiring, destruction) through the container, while the GoF Singleton manages itself
      - Practically, in a Spring app, `@Component`/`@Service` beans are singletons by default, so there's no need to hand-roll GoF Singletons — the container handles it
      - GoF singleton is hard to test (global state, no constructor injection) while Spring singletons are easily mockable because they are injected
      - GoF uses static state shared across the whole JVM, Spring singleton scope is per `ApplicationContext`
      - Prototype scope is the alternative in Spring, and a Spring singleton is thread-safe only if the bean is stateless or synchronized — same as any shared object
      - GoF-style singletons are still useful in non-Spring utility code, and `@Bean` methods returning singletons differ from `@Component` in proxy behavior
62. Explain in detail `@Component`, `@Bean`, `@Configuration`, `@Repository`, `@Service`, `@RestController`, and `@Controller`. Also, why are there dedicated annotations?
   - **Answer:**
      - All these are **stereotype** annotations that register a class as a Spring bean, which the container then manages with dependency injection
      - `@Component` — the generic stereotype for any Spring-managed bean
         - Spring scans for it with `@ComponentScan` and registers the instance in the container
      - `@Service` — a specialization of `@Component` for the business/service layer
         - Semantically marks service classes; functionally identical to `@Component` (Spring treats it the same during scanning)
      - `@Repository` — a specialization for the persistence/data-access layer
         - On top of registering a bean, it enables Spring's exception translation (wraps vendor-specific exceptions like SQLException into Spring's `DataAccessException`) and is detected for persistence features
      - `@Controller` — a specialization for MVC controllers that return views (Spring MVC / Thymeleaf)
         - It is detected by `DispatcherServlet` for `@RequestMapping` handling
      - `@RestController` — a convenience annotation combining `@Controller` + `@ResponseBody`
         - Every method's return value is serialized directly to JSON/XML in the HTTP response body, making it suitable for REST APIs
      - `@Configuration` — marks a class as a source of bean definitions; the class contains `@Bean` methods
         - Spring processes it with CGLIB proxying so `@Bean` methods follow singleton semantics even if called directly within the config class
      - `@Bean` — a method-level annotation (used inside a `@Configuration` or `@Component` class) that tells Spring to use the method's return value as a bean definition
         - It is used to register beans that are not your own classes — like a `RestTemplate`, `PasswordEncoder`, or `DataSource`
      - **Why dedicated annotations?**
         - They are not functionally required — only `@Component` is technically needed for scanning — but they give **semantic clarity** (it's possible to see at a glance which layer a class belongs to) and enable **layer-specific behavior**
         - `@Repository` adds exception translation, `@Controller`/`@RestController` get picked up for web handling, `@Service` is used by Spring's transaction and AOP conventions, and tools (like Spring docs and some code generators) can detect layered architecture from them
      - `@Component` vs `@Bean` differ in usage — `@Component` is class-level and discovered by scanning; `@Bean` is method-level and explicitly declared, useful for third-party classes and conditional wiring
      - `@Configuration` vs `@Component` proxy behavior: calling a `@Bean` method inside a `@Configuration` class returns the cached singleton, while inside a `@Component` class it creates a new instance
      - `@RestController` vs `@Controller` — a plain `@Controller` returns a view name unless a method is annotated `@ResponseBody`, whereas `@RestController` applies `@ResponseBody` to every method
      - A config example with `@Bean` for a `RestTemplate` and `@Repository` translating JPA exceptions like `DataIntegrityViolationException` in service code
