# Spring Core

---

## What is Spring Core?

- **Definition**
  - Spring Core is the foundation module of the Spring Framework that provides the **Inversion of Control (IoC)** container and **Dependency Injection (DI)** capabilities.
  - It is responsible for managing the complete lifecycle of Java objects — from instantiation to destruction — so that developers can focus on business logic rather than object wiring.
  - Spring is lightweight Java Framework.
  - It provides a comprehensive programming and configuration model for Java based enterprise application.
  - It is built on core concepts like IoC, DI, and AOP.

### Historical Context

- Spring emerged in 2003 as a response to J2EE 1.3's heavyweight EJB 2.x model, where every business object required home interfaces, remote interfaces, deployment descriptors, and a JNDI lookup.
- Spring's IoC container eliminated the need for EJBs for most applications by providing lightweight DI with POJOs (Plain Old Java Objects).
- Spring 1.x used XML exclusively.
- Spring 2.5 introduced annotation-driven injection (`@Autowired`).
- Spring 3.0 added Java-based `@Configuration` classes.
- Spring Boot (2014) auto-configured the container based on classpath dependencies, making the container effectively invisible for most applications.
- The trend across all versions is toward less configuration ceremony — from hundreds of XML lines to zero explicit configuration in Boot.

### Inversion of Control (IoC)

- Traditional applications create their own objects using `new`. With IoC, the **Spring container** creates and manages objects, then **injects** them where needed. This shifts control from the application to the container.

```java
// Traditional: code creates dependencies
public class OrderService {
    private InventoryService inventory = new InventoryService();
    private PaymentGateway gateway = new StripeGateway();
}

// IoC: container injects dependencies
@Service
public class OrderService {
    private final InventoryService inventory;
    private final PaymentGateway gateway;

    public OrderService(InventoryService inventory, PaymentGateway gateway) {
        this.inventory = inventory;
        this.gateway = gateway;
    }
}
```

### IoC Container Types

- **`BeanFactory`** — The most basic container. It instantiates beans **lazily** (only when requested). Suitable for resource-constrained environments like mobile devices, but lacks enterprise features.
- **`ApplicationContext`** — The full-featured container built on `BeanFactory`. It provides **eager singleton initialization**, **event publishing**, **internationalization (i18n)** via `MessageSource`, and **AOP integration**. This is what you should use in almost all cases.
- **`ConfigurableApplicationContext`** — Extends `ApplicationContext` with lifecycle methods like `refresh()`, `close()`, and `registerShutdownHook()`. Used for programmatic context management.
- **`WebApplicationContext`** — Web-aware variant with additional scopes: `request`, `session`, `application`, and `websocket`. Used in Spring MVC and Spring Boot web applications.

### Bean Scopes

- **`singleton`** — One instance per Spring container. All requests for the same bean ID return the same object. This is the default scope and the most memory-efficient.
- **`prototype`** — A new instance is created every time the bean is injected or requested. Use for stateful beans where each caller needs a fresh instance.
- **`request`** — One instance per HTTP request. Only valid in web-aware contexts like Spring MVC.
- **`session`** — One instance per HTTP session. Useful for user-specific state that persists across requests.
- **`application`** — One instance per `ServletContext`. Similar to singleton but scoped to the web application context.
- **`websocket`** — One instance per WebSocket session. Used for WebSocket-based applications.

```java
@Service
@Scope("singleton")
public class CacheService { }

@Component
@Scope("prototype")
public class TaskRunner { }

@Controller
@Scope("request")
public class RequestScopedController { }
```

### ApplicationContext Initialization Sequence

- The Spring container follows a well-defined startup sequence:

```
1. Load configuration (XML, annotations, or Java config)
2. Scan for beans and read bean definitions
3. Resolve inter-bean dependencies
4. Register BeanPostProcessors
5. Instantiate and wire beans
6. Initialize eager singletons
7. Publish ContextRefreshedEvent
```

### Scale Considerations

- At small scale (one service, ~100 beans), Spring's startup time is negligible (~2-5 seconds).
- At microservice scale with auto-scaling groups that restart instances frequently (every deploy, scaling event, or AZ failure recovery), each second of startup adds significant deployment latency.
- A Spring Boot application with 500 beans, 40 auto-configuration classes, and broad `@ComponentScan` can take 60-90 seconds to start.
- At 10 instances per deploy and 20 deploys per day, that is 5+ hours of cumulative startup time per day.
- Startup optimization — explicit scanning, excluding unused auto-configurations, AOT compilation (Spring 3.x), and lazy initialization — directly reduces deployment cycle time and cloud compute costs.
- At even larger scale (100+ microservices per deploy), slow startup cascades into deployment window violations.

### Key Stereotype Annotations

- **`@Component`** — Generic stereotype for any Spring-managed bean. It is the base annotation that all other stereotypes are meta-annotated with.
- **`@Service`** — Specialization of `@Component` for service-layer beans. It adds semantic meaning and makes the layer intent clear.
- **`@Repository`** — Specialization for DAO/repository beans. Spring adds translation of persistence exceptions (e.g., `DataAccessException`) automatically.
- **`@Controller`** — Specialization for web controller beans in Spring MVC. Used with request mapping annotations to handle HTTP requests.

---

## Core Concepts

### Dependency Injection via Java Configuration

- Java configuration uses `@Configuration` classes with `@Bean` methods to declare beans:

```java
@Configuration
@ComponentScan(basePackages = "com.example.service")
@PropertySource("classpath:application.properties")
public class AppConfig {

    @Bean
    @Scope("prototype")
    public TaskRunner taskRunner() {
        return new TaskRunner();
    }

    @Bean
    public DataSource dataSource(
            @Value("${db.url}") String url,
            @Value("${db.user}") String user,
            @Value("${db.password}") String password) {
        return DataSourceBuilder.create()
            .url(url)
            .username(user)
            .password(password)
            .build();
    }
}
```

### The Bean Processing Pipeline

- Every bean goes through a detailed lifecycle pipeline:

```
Bean definition loaded
    ↓
BeanFactoryPostProcessor (modify bean definitions)
    ↓
Instantiation (constructor or factory method)
    ↓
Populate properties (setters / @Autowired fields)
    ↓
Aware interfaces (BeanNameAware, ApplicationContextAware, etc.)
    ↓
BeanPostProcessor#postProcessBeforeInitialization
    ↓
@PostConstruct / InitializingBean / init-method
    ↓
BeanPostProcessor#postProcessAfterInitialization
    ↓
    → Bean is ready for use
    ↓
@PreDestroy / DisposableBean / destroy-method
```

### Configuration Approaches

- **XML-based configuration** — Beans defined in `applicationContext.xml`. Still supported but legacy and not type-safe. Use only for gradual migration.
- **Annotation-based configuration** — `@Component`, `@Service`, `@Repository`, `@Controller` with component scanning. Convenient but less explicit about bean wiring.
- **Java-based configuration** — `@Configuration` classes with `@Bean` methods. This is the modern, type-safe approach preferred in Spring Boot. It is refactorable and keeps bean definitions close to the code.

### Dependency Injection Mechanisms

- **Constructor injection** — Dependencies provided via constructor arguments. Best for required dependencies. Beans are immutable and always fully initialized. This is the recommended approach.
- **Setter injection** — Dependencies set via setter methods. Best for optional dependencies with defaults that can be changed after construction.
- **Field injection** — Dependencies injected directly into fields via reflection. Avoid in production code — it hides dependencies, prevents immutability, and makes testing harder.

### Key Annotations Reference

- **`@Configuration`** — Marks a class as a source of bean definitions. Enables CGLIB proxying for inter-bean reference interception.
- **`@Bean`** — Indicates that a method produces a bean to be managed by Spring. The method's return type defines the bean type.
- **`@Autowired`** — Marks a constructor, field, or setter for automatic dependency injection. Since Spring 4.3, optional on single-constructor beans.
- **`@Value`** — Injects values from properties files, environment variables, or SpEL expressions. Supports default values with colon syntax.
- **`@Scope`** — Defines the scope of a bean (singleton, prototype, request, session, etc.).
- **`@Lazy`** — Defers bean initialization until first use. Useful for expensive beans not always needed.
- **`@Primary`** — Indicates the preferred bean when multiple candidates of the same type exist.
- **`@Qualifier`** — Specifies which bean to inject by name when multiple candidates exist.

---

## Common Mistakes

- **Using field injection in production code**
  - Dependencies are hidden, making the class harder to test and impossible to instantiate outside the container.
  - **Why it looks correct:** `@Autowired` on a field is concise, removes boilerplate constructor code, and the field is populated before any business method is called — tests that start the full Spring context also work fine. The problem only surfaces when trying to unit-test the class without Spring (e.g., `new OrderService()` gives `NullPointerException` on every method call).
  - Always prefer constructor injection for explicit and testable code.

- **Not specifying `@Qualifier` when multiple beans of the same type exist**
  - Spring throws `NoUniqueBeanDefinitionException`.
  - **Why it looks correct:** having one bean of a type works fine, and adding a second bean of the same type doesn't cause a compiler error — the problem only surfaces at runtime when the container tries to wire the ambiguous dependency.
  - Use `@Primary` for a default or `@Qualifier` for explicit selection to avoid ambiguity.

- **Using `BeanFactory` when `ApplicationContext` is needed**
  - `BeanFactory` lacks event support, i18n, and AOP integration.
  - **Why it looks correct:** `getBean()` works, the application starts, and simple demos function perfectly. The missing features only matter when you add event listeners, message sources, or `BeanPostProcessor`-based features like `@Transactional`.
  - Use `ApplicationContext` unless you have a specific resource-constrained reason not to.

- **Circular dependencies with constructor injection**
  - Spring cannot resolve circular constructor dependencies and throws `BeanCurrentlyInCreationException`.
  - **Why it looks correct:** both services work independently, compile without errors, and the dependency graph appears symmetric and natural. The problem only manifests at startup when Spring cannot decide which bean to create first.
  - Fix by extracting a shared interface, using `@Lazy` on one side, or restructuring the code.

- **Heavy initialization in singleton beans**
  - Slows application startup significantly.
  - **Why it looks correct:** loading caches, warming connections, and precomputing data in constructors improves first-request latency. The cost is paid at startup, which may be acceptable in development but devastates deployment velocity in production with auto-scaling groups restarting frequently.
  - Consider `@Lazy` for expensive beans or move heavy initialization to `@PostConstruct` or `@EventListener(ContextRefreshedEvent.class)`.

- **Forgetting `@Configuration` on Java config classes**
  - Without it, `@Bean` methods are not intercepted and return new instances every time instead of singleton beans from the container.
  - **Why it looks correct:** `@Component` also registers beans, the application starts without errors, and each `@Bean` method returns a valid object. The duplicate instances are only discovered when a bean's state changes unexpectedly or when profiling shows excessive memory usage from multiple instances.

- **Misunderstanding proxy behavior**
  - Self-invocation (calling a `@Transactional` method from within the same class) bypasses the proxy, so annotations like `@Transactional` and `@Cacheable` don't work.
  - **Why it looks correct:** the code compiles, runs, and the IDE autocompletes `this.method()` naturally. The annotation is silently ignored — no error, no log, no warning — making this one of the hardest Spring bugs to find in production.

---

## Real-World Scenarios

### Scenario 1: Multi-Tenant SaaS Platform with Conditional Bean Registration

- **Context**
  - A SaaS platform serves hundreds of tenants, each with custom feature flags. Some tenants use Stripe, others use PayPal. You need to register only the payment gateways relevant to the requesting tenant.

```java
@Configuration
public class PaymentConfig {
    @Bean
    @ConditionalOnProperty(name = "payment.provider", havingValue = "stripe")
    public PaymentGateway stripeGateway() {
        return new StripeGateway();
    }

    @Bean
    @ConditionalOnProperty(name = "payment.provider", havingValue = "paypal")
    public PaymentGateway payPalGateway() {
        return new PayPalGateway();
    }
}
```

- The tenant context determines which profile is active, and Spring only instantiates the needed beans — saving memory and startup time.

### Scenario 2: E-Commerce Order Processing with Circular Dependency

- **Context**
  - `OrderService` needs `InventoryService` to reserve stock, and `InventoryService` needs `OrderService` to check pending orders. This circular dependency crashes startup with `BeanCurrentlyInCreationException`.

```java
// Fix: Extract a shared interface and use events instead of direct calls
public interface StockReservationService {
    boolean reserveStock(Long productId, int quantity);
}

@Service
public class OrderService {
    private final StockReservationService stockService;
    public OrderService(StockReservationService stockService) {
        this.stockService = stockService;
    }
}

@Service
public class InventoryService implements StockReservationService {
    private final ApplicationEventPublisher eventPublisher;
    // No dependency on OrderService — uses events to decouple
}
```

### Scenario 3: Microservice with Lazy-Init Report Engine

- **Context**
  - A reporting microservice has an `ExpensiveReportEngine` that loads ML models into memory (2GB). Most API calls are for lightweight dashboards, and reports are generated only on-demand.

```java
@Service
@Lazy
public class ExpensiveReportEngine {
    public ExpensiveReportEngine() {
        // Load 2GB ML model — deferred until first use
    }
}

@Service
public class ReportService {
    @Autowired
    @Lazy  // Don't inject the actual bean yet
    private ExpensiveReportEngine reportEngine;

    public Report generateReport() {
        return reportEngine.generate(); // Model loaded here, on first call
    }
}
```

- This keeps the 2GB model out of memory for the 90% of requests that don't need it.

---

## Use Cases

- Spring Core's IoC container is the foundation for any Spring application. These use cases help identify when and how to leverage its core capabilities — from bean wiring to conditional configuration.

- **Microservice with environment-specific beans** — A multi-environment deployment (dev/staging/prod) requires different implementations for the same interface (e.g., `DataSource`, `BlobStorage`).
  - Use `@Profile` and `@Conditional` to register beans only for the active environment. Keeps environment logic out of code.
  - **Avoid when:** You need runtime (not deploy-time) switching — use delegation or strategy pattern instead.

- **Plugin-style architecture with dynamically discovered beans** — A notification system adds new channels (email, SMS, push) via separate JARs without modifying the core router.
  - Inject `List<NotificationSender>` to collect all implementations automatically. New senders are picked up without code changes to the router.
  - **Avoid when:** Bean ordering matters and default ordering is insufficient — use `@Order` or `@Priority`.

- **Lazy-init for expensive singleton beans** — A reporting engine loads a 2GB ML model that is only needed for 10% of requests.
  - Mark the bean `@Lazy` and inject with `@Lazy` at the injection point. The model is loaded only on first use.
  - **Avoid when:** The bean is used in `@PostConstruct` of another bean — lazy beans may not be available yet.

- **Custom scoped beans for request-bound state** — A web application needs a user-cached shopping cart scoped to the HTTP session.
  - Use `@Scope(value = "session", proxyMode = ScopedProxyMode.TARGET_CLASS)`. The same bean instance lives for the duration of the session.
  - **Avoid when:** The bean is stateless — singleton scope is more efficient and avoids proxy overhead.

- **Testing with mock dependencies** — A service depends on a payment gateway that is unavailable in CI environments.
  - Override the bean definition in a `@TestConfiguration` class with a mock/stub. Spring's DI container substitutes the real bean for the test double.
  - **Avoid when:** You need to test the wiring itself — use `@SpringBootTest` instead of manual bean overrides.

---

## Scenario-Based Questions

**Q: You are migrating a 10-year-old Spring 3 application with XML config to Spring Boot with annotation-based config. The legacy app has 200+ bean definitions in XML. How do you approach this incrementally without a big-bang rewrite?**

- Use `@ImportResource` to load legacy XML config alongside Java config. Migrate one module at a time: create `@Configuration` classes for new beans, move bean definitions from XML to `@Bean` methods, and remove `@ImportResource` entries as modules are fully migrated. Use `@Profile` to run old and new implementations in parallel during the transition.
- **Interview follow-up:** The candidate suggested incremental migration with `@ImportResource`. During the transition period, some beans are defined in both XML and Java config, causing `BeanDefinitionOverrideException`. How would you detect and prevent duplicate bean definitions during the migration?

---

**Q: Your startup fails because `ServiceA` depends on `ServiceB` which depends on `ServiceA`. Both teams insist they cannot restructure. You cannot use setter injection per team policy. How do you break the cycle?**

- Introduce an event-driven architecture. Extract an `ApplicationEvent` (e.g., `OrderPlacedEvent`) and have `ServiceA` publish it. `ServiceB` listens via `@EventListener` rather than being called directly. This removes the compile-time dependency while keeping constructor injection. Alternatively, use `@Lazy` on one side of the constructor — Spring creates a proxy that resolves lazily, breaking the cycle.
- **Interview follow-up:** The candidate suggested `@Lazy` on one side of the constructor. The `@Lazy` proxy defers initialization, but the first method call on the proxy triggers full initialization. If the first call happens inside the other bean's constructor (during field access), the cycle reappears. How would you ensure the lazy proxy is never accessed during the other bean's construction phase?

---

**Q: A singleton `CacheManager` is injected into a `prototype`-scoped `TaskRunner`. Every new `TaskRunner` should get a fresh `CacheManager` but Spring keeps returning the same one. How do you fix this?**

- This is the prototype-in-singleton anti-pattern. Use `ObjectFactory<CacheManager>` or `Provider<CacheManager>` in the `TaskRunner` to retrieve a new instance on every call:

```java
@Component
@Scope("prototype")
public class TaskRunner {
    private final ObjectFactory<CacheManager> cacheManagerFactory;
    public Task(ObjectFactory<CacheManager> cacheManagerFactory) {
        this.cacheManagerFactory = cacheManagerFactory;
    }
    public void run() {
        CacheManager cm = cacheManagerFactory.getObject(); // fresh instance
    }
}
```

- **Interview follow-up:** The candidate used `ObjectFactory` to get fresh prototype instances. If `CacheManager` itself depends on singleton beans (e.g., a `DataSource`), those dependencies are still singletons — only `CacheManager` is re-created. Does `ObjectFactory.getObject()` also re-initialize the transitive dependency graph, or only the requested bean?

---

**Q: You have 50 `@Service` classes and the team cannot agree on package naming. Startup is 90s because Spring scans the entire classpath. How do you bring it under 15s?**

- Replace broad `@ComponentScan` with explicit `basePackages`. Exclude unused auto-configurations via `@SpringBootApplication(exclude = ...)`. Set `spring.main.lazy-initialization=true` to defer beans that are not needed at startup. Profile the startup with `-Dspring.autoconfigure.logging=true` to see which auto-configurations are matched. Finally, use Spring Boot 3.x AOT compilation to pre-resolve bean definitions at build time.
- **Interview follow-up:** The candidate recommended explicit `basePackages` to reduce scan scope. If the codebase has 20 internal libraries each providing auto-configuration, those libraries' beans are still scanned regardless of `basePackages`. How would you identify and disable auto-configurations from libraries that the application does not use?

---

**Q: Your `@Configuration` class has a `@Bean` method that calls another `@Bean` method directly. Both beans are singleton-scoped, but Spring creates two different instances. What went wrong?**

- The `@Configuration` class itself must be annotated with `@Configuration` (not `@Component`). When `@Configuration` is present, Spring creates a CGLIB proxy for the class that intercepts `@Bean` method calls and returns the singleton instance from the container. If you use `@Component` instead, the inter-bean references are not intercepted, and each call creates a new instance:

```java
@Configuration  // NOT @Component — CGLIB proxy is critical
public class AppConfig {
    @Bean
    public A a() { return new A(b()); }  // Intercepted: returns singleton B
    @Bean
    public B b() { return new B(); }
}
```

- **Interview follow-up:** The candidate identified the missing `@Configuration` annotation as the root cause. CGLIB proxying requires the class not be `final` and the `@Bean` methods not be `private` or `final`. In a Kotlin codebase, all classes are `final` by default. How would you handle inter-bean references in Kotlin Spring configurations?

---

**Q: A junior developer annotated every service class with `@Component` instead of `@Service`. Your monitoring team relies on stereotype-based filtering to detect slow services. How do you fix this without changing 200 files?**

- Create a meta-annotation or a custom stereotype that `@Service` provides. Since `@Service` is a specialization of `@Component`, the beans work either way. Use a BeanPostProcessor that logs a warning when beans match specific packages but lack `@Service`:

```java
@Component
public class StereotypeValidator implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String name) {
        if (bean.getClass().getPackageName().startsWith("com.app.service")
            && !bean.getClass().isAnnotationPresent(Service.class)) {
            log.warn("{} should use @Service, not @Component", name);
        }
        return bean;
    }
}
```

  - Then schedule a sprint to correct them gradually.

- **Interview follow-up:** The candidate proposed a `BeanPostProcessor` to warn about the wrong stereotype. If a `@Component`-annotated class is in the service package but is a utility helper (e.g., `StringUtils`), the warning is a false positive. How would you distinguish genuine service classes from utility helpers when they share the same package?

---

**Q: You need to inject a `RestTemplate` that connects to an external API. The API team gives you three environments (dev, staging, prod) with different base URLs. How do you configure this without rebuilding?**

- Define the URL in `application-{profile}.yml` and inject it via `@Value`:

```java
@Configuration
public class RestClientConfig {
    @Bean
    public RestTemplate paymentApiRestTemplate(
            @Value("${payment.api.base-url}") String baseUrl) {
        return new RestTemplateBuilder().rootUri(baseUrl).build();
    }
}
```

  - Switch profiles at deploy time: `--spring.profiles.active=prod`. No code change, no rebuild.

- **Interview follow-up:** The candidate used `@Value` with profile-specific properties. If the external API requires client certificates that differ by environment, injecting a URL via `@Value` is insufficient — the entire `RestTemplate` configuration (SSL context, timeouts, interceptors) changes per environment. How would you switch the entire `RestTemplate` bean configuration per profile without conditional logic in the `@Bean` method?

---

**Q: Your `@Autowired` constructor has 12 parameters. The code works but the team is unhappy. How do you refactor this?**

- Twelve constructor parameters violate the Single Responsibility Principle. Group related dependencies into single-purpose objects. For example, extract `OrderConfiguration` containing `OrderRepository`, `PaymentGateway`, `InventoryClient`, and `NotificationService`. Then inject `OrderConfiguration` instead:

```java
@Component
@ConfigurationProperties(prefix = "order")
public record OrderConfig(
    OrderRepository orderRepository,
    PaymentGateway paymentGateway,
    InventoryClient inventoryClient,
    NotificationService notificationService
) {}
```

  - This reduces the constructor to 2-3 parameters and logically groups concerns.

- **Interview follow-up:** The candidate grouped dependencies into an `OrderConfig` record. This reduces constructor parameter count but introduces a new issue: tests that need only one dependency (e.g., `OrderRepository`) must now mock the entire `OrderConfig` record. How would you balance constructor parameter reduction against test isolation?

---

**Q: Your application context fails to refresh because `@PostConstruct` in a `@Configuration` class calls a bean method that depends on a not-yet-initialized bean. How do you sequence initialization correctly?**

- The `@Configuration` class itself is instantiated early in the lifecycle. Do not use `@PostConstruct` in `@Configuration` classes for logic that depends on other beans. Instead, create a separate `@Component` with `@PostConstruct`, or use `@EventListener(ContextRefreshedEvent.class)` to run initialization after all beans are ready:

```java
@Component
public class DataInitializer {
    @EventListener(ContextRefreshedEvent.class)
    public void init() {
        // All beans are fully initialized here
    }
}
```

- **Interview follow-up:** The candidate recommended `@EventListener(ContextRefreshedEvent.class)`. If the `ContextRefreshedEvent` listener itself needs beans that are created by `BeanFactoryPostProcessors` (which modify bean definitions before any beans are instantiated), could those beans be unavailable? At what point in the lifecycle does `ContextRefreshedEvent` fire relative to `BeanPostProcessor` registration?

---

**Q: You have three beans of type `DataSource` (main DB, reporting DB, analytics DB). Your `@Repository` classes should each use a specific one. How do you wire this cleanly?**

- Use `@Qualifier` at both the bean declaration and injection point, or create custom qualifier annotations:

```java
@Target({FIELD, PARAMETER, METHOD})
@Retention(RUNTIME)
@Qualifier
public @interface ReportingDb {}

@Configuration
public class DataSourceConfig {
    @Bean @Primary
    public DataSource mainDataSource() { ... }
    @Bean @ReportingDb
    public DataSource reportingDataSource() { ... }
}

@Repository
public class ReportingRepository {
    private final DataSource dataSource;
    public ReportingRepository(@ReportingDb DataSource dataSource) {
        this.dataSource = dataSource;
    }
}
```

- **Interview follow-up:** The candidate created custom qualifier annotations. If a third `DataSource` (analytics) is added, do you create yet another custom annotation? At what point would you switch to a `DataSource` routing approach (e.g., `AbstractRoutingDataSource`) instead of adding annotations, and what are the trade-offs?

---

## Interview Questions

- **What is Inversion of Control (IoC)?**
  - IoC is a principle where the control of object creation and lifecycle is transferred from the application to a container. Instead of calling `new`, objects declare their dependencies, and the Spring container injects them. This decouples object creation from business logic.

- **What is the difference between `BeanFactory` and `ApplicationContext`?**
  - `BeanFactory` is the basic IoC container that lazily instantiates beans. `ApplicationContext` extends `BeanFactory` with eager singleton initialization, event publishing, internationalization (i18n), AOP integration, and `BeanPostProcessor` registration. Use `ApplicationContext` in almost all cases.

- **How does Spring resolve a circular dependency with constructor injection?**
  - Spring cannot resolve circular dependencies with constructor injection — it throws `BeanCurrentlyInCreationException`. The fix involves: (a) using `@Lazy` on one side to defer initialization, (b) switching to setter injection on one side (Spring creates a proxy), or (c) restructuring code to eliminate the cycle, often by introducing events or extracting a shared dependency into a third class.

- **What is the difference between `@Component` and `@Bean`?**
  - `@Component` is a class-level annotation that Spring auto-detects via component scanning. `@Bean` is a method-level annotation placed inside a `@Configuration` class that explicitly declares a bean. Use `@Component` for your own classes and `@Bean` for third-party classes you cannot annotate.

- **What are bean scopes in Spring?**
  - Singleton (default) — one instance per container. Prototype — new instance per request/injection. Request — one per HTTP request. Session — one per HTTP session. Application — one per ServletContext. WebSocket — one per WebSocket session.

- **How does `@Lazy` work and when would you use it?**
  - `@Lazy` defers bean instantiation until first use instead of at startup. Use it for expensive beans (ML models, report engines) that are not always needed, to reduce startup time. On `@Configuration` classes, it makes all `@Bean` methods lazy. With `@Autowired`, it creates a proxy that initializes the real bean on first method call.

- **Explain the bean lifecycle in Spring.**
  - Bean definition loaded → `BeanFactoryPostProcessor` → Instantiation → Populate properties → Aware interfaces (`BeanNameAware`, `ApplicationContextAware`) → `BeanPostProcessor#postProcessBeforeInitialization` → `@PostConstruct` / `InitializingBean` / `init-method` → `BeanPostProcessor#postProcessAfterInitialization` (AOP proxies created here) → Ready for use → `@PreDestroy` / `DisposableBean` / `destroy-method`.

- **What happens when a prototype bean is injected into a singleton bean?**
  - The prototype bean is created once when the singleton is instantiated. All future calls use the same prototype instance. To get a new instance each time, inject `ObjectFactory<T>`, `Provider<T>`, or use `@Lookup` method injection.

- **What is a `BeanPostProcessor` and how does it differ from a `BeanFactoryPostProcessor`?**
  - A `BeanPostProcessor` operates on bean instances — it runs before and after initialization callbacks and can wrap beans with proxies (used for `@Transactional`, `@Async`). A `BeanFactoryPostProcessor` operates on bean definitions before any beans are created — it modifies property values or adds placeholder resolution. `BeanFactoryPostProcessors` run first.

- **How do you conditionally register a bean in Spring?**
  - Use `@ConditionalOnProperty`, `@ConditionalOnClass`, `@ConditionalOnMissingBean`, or `@Profile`. For custom logic, implement `Condition` and use `@Conditional(MyCondition.class)`. Example: `@ConditionalOnProperty(name = "feature.x.enabled", havingValue = "true")`.

---

## Developer Recommendations

- **Use constructor injection over field injection**
  - Constructor injection makes dependencies explicit, enables immutability (`final` fields), and fails at compile time if a required bean is missing. Field injection hides dependencies and fails at runtime with a `NullPointerException`.
  - **Production story:** A production incident involved a service using field injection for 8 dependencies — when a bean was accidentally excluded via a misconfigured `@Profile`, the service started without errors but threw `NullPointerException` on every request because the field was never injected, requiring a rollback to diagnose.

- **Use `@Service`, `@Repository`, `@Controller` over bare `@Component`**
  - Each provides semantic meaning and enables targeted AOP (e.g., Spring automatically translates persistence exceptions in `@Repository` classes).
  - Using `@Component` everywhere loses this behavior and makes code harder to navigate.

- **Prefer `@Configuration` over XML**
  - Java config is type-safe, refactorable, and keeps bean definitions close to the code.
  - XML config requires switching contexts and cannot be checked at compile time.
  - Use `@ImportResource` only for gradual migration from legacy XML.

- **Use `@Qualifier` or custom qualifier annotations over `@Primary`**
  - `@Primary` silently picks a default when multiple beans exist, which can surprise future developers.
  - Explicit `@Qualifier` makes the selection obvious.
  - Custom qualifier annotations (like `@ReportingDb`) are even better because they convey business intent.

- **Avoid injecting `ApplicationContext` directly**
  - It ties your code to the Spring container and signals that the design may have a missing abstraction.
  - Instead, inject the specific dependency needed. If you must access the container, implement `ApplicationContextAware` only as a last resort.
  - **Production story:** A team once injected `ApplicationContext` into a utility class and called `context.getBean()` throughout the codebase. When upgrading from Spring 4 to Spring Boot 3, the `ApplicationContext` initialization order changed, and the utility class was constructed before the context was fully refreshed — `getBean()` threw `IllegalStateException` in 40 different call sites, requiring a week-long refactor to inject dependencies directly.

- **Use `@Lazy` for expensive beans that are not always needed**
  - Deferring bean creation improves startup time and reduces memory.
  - However, be aware that the first request will be slower (paying the deferred cost). Document this trade-off clearly.

- **Use `ObjectFactory<T>` or `Provider<T>` for obtaining prototype beans from singletons**
  - Direct injection of a prototype bean into a singleton gives you only one instance.
  - `ObjectFactory` calls `getBean()` on every request, respecting the prototype scope.
  - This is cleaner than using `ApplicationContext.getBean()` manually.

- **Limit constructor parameters to 5-6; beyond that, refactor**
  - Too many constructor parameters indicates a class has too many responsibilities.
  - Use facade, aggregate configuration objects, or split the class.
  - The pain of excessive parameters is a design smell, not a Spring limitation.
