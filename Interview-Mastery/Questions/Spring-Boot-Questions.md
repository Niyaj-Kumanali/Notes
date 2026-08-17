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
      - In my CDMS project at Talentpace, I used Spring to manage service layers, repository injection, and transaction boundaries for MSSQL operations
   - **If asked more:**
      - I would explain the modular architecture: Core Container, Data Access, Web, AOP, and how I chose specific modules for my REST APIs instead of pulling the entire framework
2. What is Spring Boot?
   - **Answer:**
      - Spring Boot is Spring's opinionated auto-configuration layer that removes boilerplate setup
      - In my projects, I just added `spring-boot-starter-web`, `spring-boot-starter-data-jpa`, and `spring-boot-starter-security`, and everything was pre-configured for Tomcat, Hibernate, and security defaults
   - **If asked more:**
      - I would explain how `@SpringBootApplication` combines `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan`, and how I used these annotations across my CDMS and inventory microservices
3. Why use Spring Boot?
   - **Answer:**
      - Spring Boot cuts development time by providing embedded Tomcat, auto-configuration, production-ready features like Actuator, and easy externalized configuration
      - In my inventory project, I deployed a Spring Boot JAR on EC2 and didn't need to manage a separate Tomcat install
   - **If asked more:**
      - I would explain the concrete time savings: I could go from a new starter project to a deployed API with JWT security and JPA in under an hour, which was critical for fast client demos
4. Difference between Spring and Spring Boot.
   - **Answer:**
      - Spring is the core framework providing DI, AOP, and MVC; Spring Boot adds auto-configuration, embedded servers, and starter dependencies on top
      - For my cold-chain APIs, I used Spring Boot so I didn't need to manually configure DispatcherServlet or Hibernate — Boot handled that automatically
   - **If asked more:**
      - I would explain that Spring Boot still uses Spring under the hood but adds `spring.factories` auto-configuration classes, and I'd show how I customized them with `application.yml` properties in my projects
5. What is auto-configuration?
   - **Answer:**
      - Auto-configuration is Spring Boot's ability to automatically configure beans based on dependencies in the classpath
      - For example, adding `spring-boot-starter-data-jpa` automatically configured my MSSQL DataSource, EntityManager, and TransactionManager in the inventory project
   - **If asked more:**
      - I would explain how `@ConditionalOnClass`, `@ConditionalOnMissingBean`, and `@ConditionalOnProperty` work internally, and how I used these to conditionally configure Redis cache only when the Redis dependency was present
6. How does Spring Boot auto-configuration work?
   - **Answer:**
      - Spring Boot scans `META-INF/spring.factories` for `EnableAutoConfiguration` classes and applies `@Conditional` checks to decide which beans to create
      - For my CDMS project, this meant when I added `spring-boot-starter-web`, Boot automatically configured Jackson, DispatcherServlet, and error handling without any XML
   - **If asked more:**
      - I would explain the `AutoConfigurationImportSelector` mechanism, how `@Conditional` prevents conflicting beans, and how I debugged auto-configuration using `--debug` flag and Actuator's `/conditions` endpoint
7. What is starter dependency?
   - **Answer:**
      - A starter dependency is a curated Maven POM that bundles related libraries
      - For my inventory validation API, I used `spring-boot-starter-validation` which brought in Hibernate Validator and its transitive dependencies, so I could use `@NotBlank` and `@Pattern` on DTOs immediately
   - **If asked more:**
      - I would explain how starters follow the naming convention `spring-boot-starter-*`, and how to create a custom starter by defining auto-configuration classes and a `spring.factories` file
8. What is embedded server?
   - **Answer:**
      - An embedded server is a web server bundled inside the application JAR
      - My Spring Boot JARs for CDMS and cold-chain projects ran on embedded Tomcat, which I configured by setting `server.port`, `server.servlet.context-path`, and SSL properties in `application.yml`
   - **If asked more:**
      - I would explain the difference between Tomcat, Jetty, and Undertow, and when I switched to Undertow for better performance in the cold-chain project that handled high-frequency IoT API calls
9. What is IoC?
   - **Answer:**
      - Inversion of Control means the framework controls object creation and lifecycle instead of the developer
      - In all my Talentpace projects, the Spring IoC container managed my service, repository, and security filter beans — I just defined them and injected where needed
   - **If asked more:**
      - I would explain the IoC container types — BeanFactory vs ApplicationContext — and how I used `ApplicationContext.getBean()` in a rare case to dynamically fetch cache manager beans based on tenant config
10. What is dependency injection?
   - **Answer:**
      - Dependency Injection is when Spring provides required objects instead of the class creating them
      - In my CDMS project, I used constructor injection for service and repository dependencies, which made testing easier with mocks and kept dependencies immutable and explicit
   - **If asked more:**
      - I would explain the three injection types with a preference for constructor injection, and discuss how Spring resolves circular dependencies using three-level cache in singleton scope
11. Types of dependency injection.
   - **Answer:**
      - There are three types: constructor injection, setter injection, and field injection
      - In all my production code, I strictly used constructor injection because it enforces immutability and makes testing straightforward
      - I never used field injection in Talentpace code since it hides dependencies and breaks testability
   - **If asked more:**
      - I would explain how Spring validates dependencies at startup, why field injection can cause NullPointerException in tests, and how I used `@RequiredArgsConstructor` from Lombok to reduce boilerplate while keeping constructor injection
12. Constructor injection vs field injection.
   - **Answer:**
      - Constructor injection makes dependencies explicit, immutable, and mandatory
      - I used constructor injection throughout my CDMS and inventory services
      - Field injection hides dependencies and makes unit tests harder because you cannot inject mocks through the constructor easily
   - **If asked more:**
      - I would explain that the Spring team recommends constructor injection, and show how I used constructor injection with Lombok's `@RequiredArgsConstructor` to keep the code clean in all my REST controllers and service classes
13. What is a Spring bean?
   - **Answer:**
      - A Spring bean is a Java object managed by the Spring IoC container
      - In my projects, classes annotated with `@Service`, `@Repository`, or `@Component` became beans
      - For example, my `InventoryValidationService` was a bean with singleton scope, reused across multiple API calls
   - **If asked more:**
      - I would explain bean naming conventions, how beans are registered via `@ComponentScan` or `@Bean` methods, and how I used `@Scope("prototype")` for a stateful validation context in the inventory engine
14. What is bean scope?
   - **Answer:**
      - Bean scope determines the lifecycle and visibility of a bean
      - Singleton scope creates one instance per container, which I used for all my service beans
      - Prototype creates a new instance every request
      - In my cold-chain project, I used prototype scope for one-time export DTOs that carried mutable state
   - **If asked more:**
      - I would explain web-aware scopes like request and session, and how I accidentally discovered singleton scope issues with `@Async` methods — the proxy behavior requires public non-static methods
15. Difference between singleton and prototype scope.
   - **Answer:**
      - Singleton creates one instance shared across the whole application, which I used for all stateless services like `PartnerService` and `ReportService`
      - Prototype creates a new instance every time it is injected or requested, which I used sparingly for objects with request-specific state
   - **If asked more:**
      - I would explain the performance trade-off: singleton saves memory but can have thread-safety issues, while prototype avoids state conflicts but increases GC pressure
      - I would describe how I resolved a thread-safety issue in a singleton service by removing instance variables
16. What is application context?
   - **Answer:**
      - ApplicationContext is the Spring IoC container that manages bean lifecycle, event propagation, and internationalization
      - In my projects, `AnnotationConfigApplicationContext` was created behind the scenes by `SpringApplication.run()`, and I occasionally used `ApplicationContextAware` to access beans programmatically
   - **If asked more:**
      - I would explain the ApplicationContext hierarchy, how it differs from BeanFactory, and how I used `ConfigurableApplicationContext.close()` in a test `@AfterClass` method to clean up the context
17. What is bean lifecycle?
   - **Answer:**
      - Bean lifecycle goes through: instantiation, property population, initialization callbacks (`@PostConstruct`, `InitializingBean`), bean is ready, then destruction callbacks (`@PreDestroy`, `DisposableBean`)
      - In my CDMS project, I used `@PostConstruct` to load reference data from MSSQL into a cache after the bean was initialized
   - **If asked more:**
      - I would explain the full sequence: BeanPostProcessors, `@PostConstruct`, `afterPropertiesSet()`, custom init-method, and how I used `BeanPostProcessor` to log bean initialization times for performance monitoring
18. What is `@Component`?
   - **Answer:**
      - `@Component` is a stereotype annotation that marks a class as a Spring-managed bean
      - In my projects, I typically used its specializations like `@Service` and `@Repository` instead of plain `@Component`, since they add semantic meaning and enable persistence exception translation
   - **If asked more:**
      - I would explain how `@ComponentScan` discovers these annotations, how custom stereotype annotations can be created, and how I used `@Component` for utility classes like `JwtUtil` that didn't fit the service/repository pattern
19. Difference between `@Component`, `@Service`, `@Repository`, and `@Controller`.
   - **Answer:**
      - All four register Spring beans, but `@Service` is a service layer specialization, `@Repository` enables persistence exception translation, and `@Controller` marks web controllers
      - In my projects, I used `@Service` for business logic like `InventoryValidationService`, `@Repository` for DAO layers, and `@Controller` for REST endpoints
   - **If asked more:**
      - I would explain that `@Repository` adds `PersistenceExceptionTranslationPostProcessor` to convert SQLExceptions into Spring's `DataAccessException`, which I relied on in the CDMS project when handling MSSQL constraint violations
20. What is `@SpringBootApplication`?
   - **Answer:**
      - `@SpringBootApplication` is a convenience annotation combining `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan`
      - Every one of my Talentpace projects had this on the main class, enabling auto-configuration and scanning all beans under the base package
   - **If asked more:**
      - I would explain how to exclude specific auto-configurations using `exclude` parameter, and how I used `@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})` when setting up a test that didn't need database connectivity
21. What does `@EnableAutoConfiguration` do?
   - **Answer:**
      - `@EnableAutoConfiguration` enables Spring Boot's auto-configuration mechanism that creates beans based on classpath dependencies
      - When I added `spring-boot-starter-web`, it automatically configured `DispatcherServlet`, `Jackson ObjectMapper`, and error handling through `ErrorMvcAutoConfiguration`
   - **If asked more:**
      - I would explain that it's backed by `AutoConfigurationImportSelector` which reads `spring.factories` files, and how I used `spring.autoconfigure.exclude` in `application.properties` when I needed to disable a conflicting auto-configuration for Redis
22. What is `@Configuration`?
   - **Answer:**
      - `@Configuration` marks a class as a source of bean definitions using `@Bean` methods
      - In my inventory project, I had a `RedisConfig` class annotated with `@Configuration` where I defined `RedisTemplate` and `CacheManager` beans with custom serialization settings
   - **If asked more:**
      - I would explain the difference between `@Configuration` with `@Bean` vs `@Component` with `@Autowired`, and how Spring proxies `@Configuration` classes using CGLIB to ensure singleton bean semantics
23. What is `@Bean`?
   - **Answer:**
      - `@Bean` is a method-level annotation that tells Spring to register the returned object as a bean in the container
      - In my cold-chain project, I used `@Bean` in a `@Configuration` class to create `KafkaTemplate`, `RedisTemplate`, and `RestTemplate` with project-specific configurations
   - **If asked more:**
      - I would explain `@Bean` lifecycle callbacks using `initMethod` and `destroyMethod`, and how I configured the `KafkaTemplate` bean with custom serializer properties and retry configuration for the IoT data pipeline
24. Difference between `@Bean` and `@Component`.
   - **Answer:**
      - `@Bean` is used in `@Configuration` classes for third-party or manually configured beans; `@Component` is for your own classes
      - I used `@Bean` for `RedisTemplate`, `KafkaTemplate`, and `PasswordEncoder` since these were framework classes, while my services used `@Component` derivatives
   - **If asked more:**
      - I would explain that `@Bean` gives full control over instantiation and configuration, while `@Component` relies on classpath scanning
      - I used `@Bean` when I needed to pass constructor arguments like database connection pools or Redis connection factory
25. What is `application.properties`?
   - **Answer:**
      - `application.properties` is Spring Boot's default configuration file for externalizing settings
      - In my CDMS project, I stored MSSQL connection URL, username, password, Hibernate DDL strategy, and server port in this file, keeping config separate from code
   - **If asked more:**
      - I would explain property loading order, profile-specific properties like `application-dev.properties`, and how I used `@Value` and `@ConfigurationProperties` to bind these values to Java objects in my services
26. Difference between `application.properties` and `application.yml`.
   - **Answer:**
      - Both serve the same purpose but `.properties` is flat key-value while `.yml` uses hierarchical indentation
      - I preferred `.yml` for my projects because it is more readable for nested configs like `spring.datasource.*` and `spring.jpa.*`, reducing duplication
   - **If asked more:**
      - I would explain YAML list support and how it improves readability for multi-profile configuration
      - I'd also mention that YAML does not support `@PropertySource` natively and how I worked around this in the inventory project
27. What are profiles?
   - **Answer:**
      - Profiles allow environment-specific bean definitions and configurations
      - I defined `application-dev.yml`, `application-staging.yml`, and `application-prod.yml` for my cold-chain project, each with different database URLs, log levels, and cache settings
   - **If asked more:**
      - I would explain how `spring.profiles.active` is set via environment variable or JVM argument, and how I used `@Profile("dev")` to conditionally load mock beans for local testing of the Kafka pipeline
28. How do you externalize configuration?
   - **Answer:**
      - Spring Boot externalizes config through application properties, environment variables, command-line arguments, and `@ConfigurationProperties`
      - In my inventory project, I kept database credentials in environment variables on the EC2 instance and accessed them via `${DATABASE_URL}` in `application.yml`
   - **If asked more:**
      - I would explain the `PropertySource` ordering, how I used `@ConfigurationProperties(prefix = "inventory.validation")` to bind nested properties to a POJO, and the benefit of type-safe configuration over `@Value`
29. What is `@Value`?
   - **Answer:**
      - `@Value` injects a property value from configuration files into a field or parameter
      - In my CDMS project, I used `@Value("${report.batch-size:1000}")` to configure batch processing size for stored procedure execution, with a sensible default
   - **If asked more:**
      - I would explain SpEL support in `@Value` for dynamic evaluation, and the trade-off between `@Value` and `@ConfigurationProperties` — `@Value` is simpler but `@ConfigurationProperties` provides type safety and IDE validation
30. What is `@ConfigurationProperties`?
   - **Answer:**
      - `@ConfigurationProperties` binds entire property hierarchies to strongly-typed Java objects
      - In my inventory engine, I had an `InventoryProperties` class annotated with `@ConfigurationProperties(prefix = "inventory")` that held validation thresholds, batch sizes, and retry counts
   - **If asked more:**
      - I would explain `@EnableConfigurationProperties`, nested POJO binding, and how I used `@Validated` with JSR-303 annotations to validate configuration values at startup, preventing production issues from misconfigured properties
31. What is actuator?
   - **Answer:**
      - Actuator provides production-ready HTTP endpoints for monitoring and managing Spring Boot applications
      - In my cold-chain project deployed on EC2, I enabled `/health`, `/metrics`, and `/info` endpoints to monitor API health and response times without building custom monitoring endpoints
   - **If asked more:**
      - I would explain the different endpoint categories, how to expose endpoints via `management.endpoints.web.exposure.include`, and how I extended Actuator by adding a custom `HealthIndicator` that checked Kafka connectivity and MSSQL availability
32. Which actuator endpoints are useful in production?
   - **Answer:**
      - `/health` for liveness checks, `/metrics` for JVM and request metrics, `/info` for application metadata, and `/loggers` for runtime log level changes
      - In production, I exposed only `/health` and `/info` publicly and kept others behind the firewall for security
   - **If asked more:**
      - I would explain how I integrated `/metrics` with Prometheus using `micrometer-registry-prometheus`, and how `/loggers` helped me debug the cold-chain IoT pipeline by enabling DEBUG logging for the Kafka consumer without restarting the JAR
33. How do you secure actuator endpoints?
   - **Answer:**
      - Actuator endpoints can be secured by restricting exposure, using separate management ports, and applying Spring Security
      - In my projects, I set `management.endpoints.web.exposure.exclude=*` and only exposed specific endpoints with `include=health,info`, then added `management.server.port=8081`
   - **If asked more:**
      - I would explain how to use `@RolesAllowed` on custom Actuator endpoints, and how I configured a separate security filter chain for the management port with IP whitelist through a custom `WebSecurityConfigurerAdapter`
34. What is Spring Boot DevTools?
   - **Answer:**
      - DevTools provides automatic restart, live reload, and remote debugging for development
      - During local development of my CDMS project, I used DevTools for automatic restart when Java files changed, and I used the LiveReload server to refresh the Swagger UI in the browser
   - **If asked more:**
      - I would explain how DevTools uses two classloaders, how to exclude resources from restart, and why I never enabled DevTools in production — it can leak sensitive information and causes performance overhead
35. How do you handle exceptions globally?
   - **Answer:**
      - I handle exceptions globally using `@ControllerAdvice` combined with `@ExceptionHandler` methods
      - In my CDMS project, I created a `GlobalExceptionHandler` class that caught `DataAccessException`, `MethodArgumentNotValidException`, and custom business exceptions, returning consistent JSON error responses with proper HTTP status codes
   - **If asked more:**
      - I would explain the exception handling hierarchy, how I mapped `MSSQLException` constraint violations to user-friendly messages, and how I logged stack traces selectively using MDC to include request IDs in logs
36. What is `@ControllerAdvice`?
   - **Answer:**
      - `@ControllerAdvice` is a global interceptor for controllers that enables cross-cutting exception handling, data binding, and model attributes
      - In my inventory project, I used a single `@ControllerAdvice` class to handle validation errors, authentication failures, and database constraint violations across all endpoints
   - **If asked more:**
      - I would explain the difference between `@ControllerAdvice` and `@RestControllerAdvice`, and how I customized the response body with `ErrorResponse` DTOs containing error code, message, timestamp, and trace ID for debugging
37. What is `@ExceptionHandler`?
   - **Answer:**
      - `@ExceptionHandler` defines a method to handle specific exceptions thrown by controllers
      - In my global handler, I had methods like `handleValidationException(MethodArgumentNotValidException)` returning 400 with field-level errors, and `handleResourceNotFound(ResourceNotFoundException)` returning 404
   - **If asked more:**
      - I would explain the priority of exception handlers, how to handle multiple exception types in one method, and how I used `ResponseEntity.exceptionHandler()` for fine-grained control over response headers and status codes in my projects
38. What is validation in Spring Boot?
   - **Answer:**
      - Validation in Spring Boot uses Bean Validation API (JSR-380) with annotations like `@NotNull`, `@Size`, and `@Pattern` on DTO fields
      - In my inventory API, I validated incoming serial numbers with `@Pattern(regexp = "^[A-Z0-9]+$")` and checked mandatory fields with `@NotBlank`
   - **If asked more:**
      - I would explain how validation integrates with `@Valid` in `@RequestBody` parameters, custom validation annotations I created for inventory-specific rules, and how the validation errors are automatically handled by `MethodArgumentNotValidException`
39. What is `@Valid`?
   - **Answer:**
      - `@Valid` triggers JSR-380 bean validation on request bodies, query parameters, or path variables
      - In my inventory project, I annotated `@RequestBody InventoryRequest` with `@Valid` in the controller, which automatically validated all field constraints before the service method was called
   - **If asked more:**
      - I would explain the difference between `@Valid` and `@Validated`, and how `@Validated` supports validation groups — I used validation groups in the CDMS project to have different validation rules for create vs update operations
40. Difference between `@Valid` and `@Validated`.
   - **Answer:**
      - `@Valid` is standard JSR-380 that triggers validation; `@Validated` is Spring's variant that adds support for validation groups
      - In my CDMS project, I used `@Validated` with groups like `OnCreate.class` and `OnUpdate.class` to apply different rules for POST and PUT endpoints
   - **If asked more:**
      - I would explain how to define validation groups using empty interfaces, and how I integrated group validation with `@RequestParam` and `@PathVariable` using `@Validated` at the class level
41. What is scheduling in Spring Boot?
   - **Answer:**
      - Scheduling in Spring Boot uses `@EnableScheduling` and `@Scheduled` annotations to run tasks periodically
      - In my CDMS project, I scheduled nightly stored procedure execution at 2 AM using a cron expression to refresh report data without manual intervention
   - **If asked more:**
      - I would explain the `TaskScheduler` abstraction, how to configure thread pools for scheduled tasks, and the importance of handling failures in scheduled jobs using try-catch blocks to prevent silent task termination
42. What is `@Scheduled`?
   - **Answer:**
      - `@Scheduled` marks a method to be executed on a schedule
      - In my inventory project, I used `@Scheduled(fixedDelay = 300000)` on a method that reconciled inventory data every 5 minutes after the previous run completed, ensuring no overlapping executions
   - **If asked more:**
      - I would explain the three modes: `fixedRate`, `fixedDelay`, and `cron`, and how I used `cron = "0 0 2 * * ?"` in the CDMS project for the nightly ETL batch without needing any external job scheduler initially
43. Difference between fixed rate and fixed delay.
   - **Answer:**
      - `fixedRate` triggers every N milliseconds regardless of whether the previous execution finished; `fixedDelay` waits N milliseconds after the previous execution completes
      - In my CDMS project, I used `fixedDelay` for the stored procedure job because overlapping runs would corrupt report data
   - **If asked more:**
      - I would explain the risk of `fixedRate` causing thread starvation when tasks take longer than the interval, and how I mitigated this in the inventory project by configuring a custom `ThreadPoolTaskScheduler` with a bounded queue
44. What is cron expression?
   - **Answer:**
      - A cron expression defines schedule using six or seven fields: second, minute, hour, day-of-month, month, day-of-week, and optional year
      - In my cold-chain project, I used `0 0/15 * * * ?` to run temperature data aggregation every 15 minutes without needing a separate cron job on the server
   - **If asked more:**
      - I would explain cron syntax with examples, how `?` and `*` differ, and the common mistake of forgetting that cron runs in the server's timezone — I had to adjust the timezone using `zone` attribute in `@Scheduled`
45. How do you configure scheduled task thread pool?
   - **Answer:**
      - By default, `@Scheduled` uses a single-threaded executor
      - In my inventory project, I configured a `ThreadPoolTaskScheduler` bean with `pool-size=5` and a custom `ErrorHandler` that logged failures without killing the scheduler, preventing a single failed task from blocking the others
   - **If asked more:**
      - I would explain how to set a custom `SchedulingConfigurer` with `@Configuration`, and the trade-offs of using a shared thread pool vs dedicated pools for critical vs non-critical scheduled jobs
46. How do you prevent scheduled jobs from running on all pods?
   - **Answer:**
      - When running multiple instances, scheduled jobs need a coordination mechanism
      - In my projects deployed on single EC2 instances, this wasn't an issue, but I planned using ShedLock with Redis to ensure only one pod executes a scheduled job at a time by acquiring a distributed lock
   - **If asked more:**
      - I would explain how ShedLock locks are persisted in a database table or Redis, how to configure lock duration, and how I would integrate ShedLock with `@Scheduled` using `@SchedulerLock(name = "nightlyReport")` for the CDMS batch job
47. What is async processing?
   - **Answer:**
      - Async processing allows methods to run in a separate thread without blocking the caller
      - In my cold-chain project, I used async processing for sending email notifications when temperature excursions exceeded thresholds, so the API response wasn't delayed by the email SMTP call
   - **If asked more:**
      - I would explain the difference between async, reactive, and parallel processing, how async improves API responsiveness, and the threading implications — including the risk of thread pool exhaustion if not configured properly
48. What is `@Async`?
   - **Answer:**
      - `@Async` marks a method for execution in a separate thread
      - In my inventory project, I annotated the `AuditLogService.saveAuditLog()` method with `@Async` so that writing audit records to MSSQL wouldn't block the main API response, improving perceived performance
   - **If asked more:**
      - I would explain that `@Async` requires `@EnableAsync` and only works on public methods called from outside the class (self-invocation bypasses the proxy)
      - I would also explain how I used `CompletableFuture` return types to handle async results
49. How do you configure async executor?
   - **Answer:**
      - I configure a `ThreadPoolTaskExecutor` bean with custom core pool size, max pool size, queue capacity, and rejection policy
      - In my cold-chain project, I set `corePoolSize=10`, `maxPoolSize=25`, and `CallerRunsPolicy` to handle spikes in sensor data processing without losing tasks
   - **If asked more:**
      - I would explain the `AsyncConfigurer` interface, how to handle uncaught exceptions in async methods using `AsyncUncaughtExceptionHandler`, and the impact of queue capacity on memory during traffic bursts in the IoT pipeline
50. What is caching in Spring Boot?
   - **Answer:**
      - Caching stores frequently accessed data in memory to reduce database load and improve response times
      - In my cold-chain project, I used Redis cache for storing temperature threshold configurations and partner device mappings, reducing repeated MSSQL queries for data that rarely changed
   - **If asked more:**
      - I would explain the `@EnableCaching` annotation, cache abstraction layers, cache managers (InMemory vs Redis), and how I measured cache hit ratios in production using Actuator metrics to tune TTL values
51. What is `@Cacheable`?
   - **Answer:**
      - `@Cacheable` stores the method result in cache and returns it on subsequent calls with the same arguments
      - I applied `@Cacheable("deviceConfigs")` on the method that fetched gateway-to-device mappings in the cold-chain project, which reduced MSSQL round trips from hundreds per minute to only a few cache misses
   - **If asked more:**
      - I would explain the cache key generation using `key` attribute and SpEL, conditional caching with `condition` and `unless`, and how I invalidated the cache proactively when device configurations were updated via admin API
52. Difference between `@Cacheable`, `@CachePut`, and `@CacheEvict`.
   - **Answer:**
      - `@Cacheable` reads and stores; `@CachePut` always executes and updates the cache; `@CacheEvict` removes entries
      - In my CDMS project, I used `@CachePut` on the partner update method to refresh cached data, and `@CacheEvict(allEntries = true)` on the data reload endpoint to clear stale entries before repopulation
   - **If asked more:**
      - I would explain `@Caching` for combining multiple cache annotations, and how I used `@CacheEvict(beforeInvocation = true)` to evict cache before method execution when failure should still result in cache being cleared
53. How do you use Redis cache with Spring Boot?
   - **Answer:**
      - I add `spring-boot-starter-data-redis`, configure Redis connection in `application.yml`, define a `RedisCacheManager` bean, and use `@Cacheable` on service methods
      - In the cold-chain project, I configured Redis with TTL of 30 minutes for sensor config cache and used `RedisTemplate` for direct operations
   - **If asked more:**
      - I would explain the difference between `RedisCacheManager` and `RedisTemplate`, how to configure serialization (I used JSON serialization with Jackson2JsonRedisSerializer), and how I handled Redis connection failures by falling back to MSSQL queries
54. How do you write REST APIs in Spring Boot?
   - **Answer:**
      - I use `@RestController` with `@RequestMapping` for class-level mapping and `@GetMapping`, `@PostMapping`, etc. for HTTP methods
      - In my CDMS project, I created `PartnerController` with endpoints like `GET /api/partners`, `POST /api/partners/sync`, and `PUT /api/partners/{id}`, returning `ResponseEntity` for status control
   - **If asked more:**
      - I would explain REST best practices — proper HTTP methods, status codes, request/response DTOs, content negotiation, and how I versioned my APIs using URL path prefix like `/v1/` in the CDMS project
55. How do you version APIs?
   - **Answer:**
      - I version APIs through the URL path prefix like `/api/v1/partners`
      - In my inventory project, I maintained backward compatibility by keeping v1 endpoints while adding new fields in v2 request/response DTOs, allowing partners to migrate gradually without breaking their integrations
   - **If asked more:**
      - I would explain other versioning strategies: header-based (`Accept-version`), query parameter (`?version=1`), and content negotiation
      - I chose URL path versioning for simplicity since it's explicit in logs and easy to route
56. How do you document APIs?
   - **Answer:**
      - I document REST APIs using Swagger/OpenAPI 3.0 with `springdoc-openapi` library
      - In my CDMS project, I added `@Operation` and `@ApiResponse` annotations on controllers to describe endpoints, request bodies, and error responses, making it easy for the frontend team to integrate without constant back-and-forth
   - **If asked more:**
      - I would explain how I customized the Swagger UI with bearer token support for JWT, how I grouped endpoints by tags, and how I used `springdoc.swagger-ui.enabled=false` in production to expose docs only on staging environments
57. What is Swagger/OpenAPI?
   - **Answer:**
      - Swagger/OpenAPI is a specification for documenting REST APIs in a machine-readable format (JSON/YAML)
      - In my inventory project, I integrated `springdoc-openapi-starter-webmvc-ui` which auto-generated OpenAPI docs from `@RestController` annotations, and provided an interactive Swagger UI at `/swagger-ui.html`
   - **If asked more:**
      - I would explain how OpenAPI 3.0 differ from Swagger 2.0, how I defined reusable components (schemas, security schemes) in the OpenAPI config, and how the generated docs helped QA write automated tests using the OpenAPI spec
58. What is Spring Boot testing?
   - **Answer:**
      - Spring Boot testing uses `@SpringBootTest` for full context integration tests and slice tests for focused layers
      - In my CDMS project, I wrote integration tests that loaded the full Spring context and tested the REST endpoint from HTTP request to MSSQL persistence using an H2 in-memory database
   - **If asked more:**
      - I would explain the testing pyramid, how to use `@TestContainers` for testing with real MSSQL, and the importance of `@DirtiesContext` for cleaning up state between tests that modify the application context
59. What is `@SpringBootTest`?
   - **Answer:**
      - `@SpringBootTest` loads the complete Spring application context for integration testing
      - In my inventory project, I used `@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)` with `TestRestTemplate` to send real HTTP requests to the API and verify response status, headers, and body
   - **If asked more:**
      - I would explain how to override properties with `@TestPropertySource` or `properties` attribute, how to use `@MockBean` for external dependencies, and how I configured the test to use an embedded H2 database instead of the production MSSQL
60. What is `@WebMvcTest`?
   - **Answer:**
      - `@WebMvcTest` loads only the web layer for controller unit testing — it does not load services, repositories, or security
      - In my CDMS project, I used `@WebMvcTest(PartnerController.class)` with `@MockBean` for the service dependency and tested request mapping, validation, and response serialization
   - **If asked more:**
      - I would explain the difference between `@WebMvcTest` and `@SpringBootTest`, how I used `MockMvc` for performing requests and assertions, and why `@WebMvcTest` was faster because it avoided scanning all beans in the context
61. Difference between the traditional Java Singleton design pattern and the Spring singleton bean scope.
   - **Answer:**
      - The GoF Singleton pattern guarantees a class has exactly one instance and exposes it via a private constructor and static `getInstance()`
      - Spring's singleton bean scope means the IoC container creates exactly one bean instance per bean name per container, but the class itself is a normal class — I can still create other instances with `new`
      - Spring manages the instance lifecycle (creation, dependency wiring, destruction) through the container, while the GoF Singleton manages itself
      - Practically, in a Spring app, `@Component`/`@Service` beans are singletons by default, so I don't hand-roll GoF Singletons — the container handles it
   - **If asked more:**
      - I can explain key differences: GoF singleton is hard to test (global state, no constructor injection) while Spring singletons are easily mockable because they are injected
      - GoF uses static state shared across the whole JVM, Spring singleton scope is per `ApplicationContext`
      - I can mention prototype scope as the alternative in Spring, and that a Spring singleton is thread-safe only if the bean is stateless or synchronized — same as any shared object
      - I'd also explain when I'd still use a GoF-style singleton — in non-Spring utility code — and how `@Bean` methods returning singletons differ from `@Component` (proxy behavior, etc.)
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
         - Every method's return value is serialized directly to JSON/XML in the HTTP response body, which is what I use for my REST APIs
      - `@Configuration` — marks a class as a source of bean definitions; the class contains `@Bean` methods
         - Spring processes it with CGLIB proxying so `@Bean` methods follow singleton semantics even if called directly within the config class
      - `@Bean` — a method-level annotation (used inside a `@Configuration` or `@Component` class) that tells Spring to use the method's return value as a bean definition
         - It is how I register beans that are not our own classes — like a `RestTemplate`, `PasswordEncoder`, or `DataSource`
      - **Why dedicated annotations?**
         - They are not functionally required — only `@Component` is technically needed for scanning — but they give **semantic clarity** (I can see at a glance which layer a class belongs to) and enable **layer-specific behavior**
         - `@Repository` adds exception translation, `@Controller`/`@RestController` get picked up for web handling, `@Service` is used by Spring's transaction and AOP conventions, and tools (like Spring docs and some code generators) can detect layered architecture from them
   - **If asked more:**
      - I can explain that `@Component` vs `@Bean` differ in usage — `@Component` is class-level and discovered by scanning; `@Bean` is method-level and explicitly declared, useful for third-party classes and conditional wiring
      - I can explain `@Configuration` vs `@Component` proxy behavior: calling a `@Bean` method inside a `@Configuration` class returns the cached singleton, while inside a `@Component` class it creates a new instance
      - I can mention `@RestController` vs `@Controller` — a plain `@Controller` returns a view name unless a method is annotated `@ResponseBody`, whereas `@RestController` applies `@ResponseBody` to every method
      - I'd also show a small config example with `@Bean` for a `RestTemplate` and mention `@Repository` translating JPA exceptions like `DataIntegrityViolationException` in my service code

