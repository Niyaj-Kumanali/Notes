# Spring Core

---

## What is Spring Core?

**Spring Core** is the foundation module of the Spring Framework that provides the **Inversion of Control (IoC)** container and **Dependency Injection (DI)** capabilities. It is responsible for managing the complete lifecycle of Java objects — from instantiation to destruction — so that developers can focus on business logic rather than object wiring.

### Inversion of Control (IoC)

Traditional applications create their own objects using `new`. With IoC, the **Spring container** creates and manages objects, then **injects** them where needed. This shifts control from the application to the container.

Example of traditional vs IoC approach:

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

The Spring container follows a well-defined startup sequence:

```
1. Load configuration (XML, annotations, or Java config)
2. Scan for beans and read bean definitions
3. Resolve inter-bean dependencies
4. Register BeanPostProcessors
5. Instantiate and wire beans
6. Initialize eager singletons
7. Publish ContextRefreshedEvent
```

### Key Stereotype Annotations

- **`@Component`** — Generic stereotype for any Spring-managed bean. It is the base annotation that all other stereotypes are meta-annotated with.
- **`@Service`** — Specialization of `@Component` for service-layer beans. It adds semantic meaning and makes the layer intent clear.
- **`@Repository`** — Specialization for DAO/repository beans. Spring adds translation of persistence exceptions (e.g., `DataAccessException`) automatically.
- **`@Controller`** — Specialization for web controller beans in Spring MVC. Used with request mapping annotations to handle HTTP requests.

---

## Core Concepts

### Dependency Injection via Java Configuration

Java configuration uses `@Configuration` classes with `@Bean` methods to declare beans:

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

Every bean goes through a detailed lifecycle pipeline:

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

- **Using field injection in production code** — Dependencies are hidden, making the class harder to test and impossible to instantiate outside the container. Always prefer constructor injection for explicit and testable code.
- **Not specifying `@Qualifier` when multiple beans of the same type exist** — Spring throws `NoUniqueBeanDefinitionException`. Use `@Primary` for a default or `@Qualifier` for explicit selection to avoid ambiguity.
- **Using `BeanFactory` when `ApplicationContext` is needed** — `BeanFactory` lacks event support, i18n, and AOP integration. Use `ApplicationContext` unless you have a specific resource-constrained reason not to.
- **Circular dependencies with constructor injection** — Spring cannot resolve circular constructor dependencies and throws `BeanCurrentlyInCreationException`. Fix by extracting a shared interface, using `@Lazy` on one side, or restructuring the code.
- **Heavy initialization in singleton beans** — Slows application startup significantly. Consider `@Lazy` for expensive beans or move heavy initialization to `@PostConstruct` or `@EventListener(ContextRefreshedEvent.class)`.
- **Forgetting `@Configuration` on Java config classes** — Without it, `@Bean` methods are not intercepted and return new instances every time instead of singleton beans from the container.
- **Misunderstanding proxy behavior** — Self-invocation (calling a `@Transactional` method from within the same class) bypasses the proxy, so annotations like `@Transactional` and `@Cacheable` don't work.

---

## Real-World Scenarios

### Scenario 1: Multi-Tenant SaaS Platform with Conditional Bean Registration

A SaaS platform serves hundreds of tenants, each with custom feature flags. Some tenants use Stripe, others use PayPal. You need to register only the payment gateways relevant to the requesting tenant.

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

The tenant context determines which profile is active, and Spring only instantiates the needed beans — saving memory and startup time.

### Scenario 2: E-Commerce Order Processing with Circular Dependency

`OrderService` needs `InventoryService` to reserve stock, and `InventoryService` needs `OrderService` to check pending orders. This circular dependency crashes startup with `BeanCurrentlyInCreationException`.

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

A reporting microservice has an `ExpensiveReportEngine` that loads ML models into memory (2GB). Most API calls are for lightweight dashboards, and reports are generated only on-demand.

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

This keeps the 2GB model out of memory for the 90% of requests that don't need it.

---

## Scenario-Based Questions

1. **Q: You are migrating a 10-year-old Spring 3 application with XML config to Spring Boot with annotation-based config. The legacy app has 200+ bean definitions in XML. How do you approach this incrementally without a big-bang rewrite?**
   A: Use `@ImportResource` to load legacy XML config alongside Java config. Migrate one module at a time: create `@Configuration` classes for new beans, move bean definitions from XML to `@Bean` methods, and remove `@ImportResource` entries as modules are fully migrated. Use `@Profile` to run old and new implementations in parallel during the transition.

2. **Q: Your startup fails because `ServiceA` depends on `ServiceB` which depends on `ServiceA`. Both teams insist they cannot restructure. You cannot use setter injection per team policy. How do you break the cycle?**
   A: Introduce an event-driven architecture. Extract an `ApplicationEvent` (e.g., `OrderPlacedEvent`) and have `ServiceA` publish it. `ServiceB` listens via `@EventListener` rather than being called directly. This removes the compile-time dependency while keeping constructor injection. Alternatively, use `@Lazy` on one side of the constructor — Spring creates a proxy that resolves lazily, breaking the cycle.

3. **Q: A singleton `CacheManager` is injected into a `prototype`-scoped `TaskRunner`. Every new `TaskRunner` should get a fresh `CacheManager` but Spring keeps returning the same one. How do you fix this?**
   A: This is the prototype-in-singleton anti-pattern. Use `ObjectFactory<CacheManager>` or `Provider<CacheManager>` in the `TaskRunner` to retrieve a new instance on every call:
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

4. **Q: You have 50 `@Service` classes and the team cannot agree on package naming. Startup is 90s because Spring scans the entire classpath. How do you bring it under 15s?**
   A: Replace broad `@ComponentScan` with explicit `basePackages`. Exclude unused auto-configurations via `@SpringBootApplication(exclude = ...)`. Set `spring.main.lazy-initialization=true` to defer beans that are not needed at startup. Profile the startup with `-Dspring.autoconfigure.logging=true` to see which auto-configurations are matched. Finally, use Spring Boot 3.x AOT compilation to pre-resolve bean definitions at build time.

5. **Q: Your `@Configuration` class has a `@Bean` method that calls another `@Bean` method directly. Both beans are singleton-scoped, but Spring creates two different instances. What went wrong?**
   A: The `@Configuration` class itself must be annotated with `@Configuration` (not `@Component`). When `@Configuration` is present, Spring creates a CGLIB proxy for the class that intercepts `@Bean` method calls and returns the singleton instance from the container. If you use `@Component` instead, the inter-bean references are not intercepted, and each call creates a new instance:
   ```java
   @Configuration  // NOT @Component — CGLIB proxy is critical
   public class AppConfig {
       @Bean
       public A a() { return new A(b()); }  // Intercepted: returns singleton B
       @Bean
       public B b() { return new B(); }
   }
   ```

6. **Q: A junior developer annotated every service class with `@Component` instead of `@Service`. Your monitoring team relies on stereotype-based filtering to detect slow services. How do you fix this without changing 200 files?**
   A: Create a meta-annotation or a custom stereotype that `@Service` provides. Since `@Service` is a specialization of `@Component`, the beans work either way. Use a BeanPostProcessor that logs a warning when beans match specific packages but lack `@Service`:
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
   Then schedule a sprint to correct them gradually.

7. **Q: You need to inject a `RestTemplate` that connects to an external API. The API team gives you three environments (dev, staging, prod) with different base URLs. How do you configure this without rebuilding?**
   A: Define the URL in `application-{profile}.yml` and inject it via `@Value`:
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
   Switch profiles at deploy time: `--spring.profiles.active=prod`. No code change, no rebuild.

8. **Q: Your `@Autowired` constructor has 12 parameters. The code works but the team is unhappy. How do you refactor this?**
   A: Twelve constructor parameters violate the Single Responsibility Principle. Group related dependencies into single-purpose objects. For example, extract `OrderConfiguration` containing `OrderRepository`, `PaymentGateway`, `InventoryClient`, and `NotificationService`. Then inject `OrderConfiguration` instead:
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
   This reduces the constructor to 2-3 parameters and logically groups concerns.

9. **Q: Your application context fails to refresh because `@PostConstruct` in a `@Configuration` class calls a bean method that depends on a not-yet-initialized bean. How do you sequence initialization correctly?**
   A: The `@Configuration` class itself is instantiated early in the lifecycle. Do not use `@PostConstruct` in `@Configuration` classes for logic that depends on other beans. Instead, create a separate `@Component` with `@PostConstruct`, or use `@EventListener(ContextRefreshedEvent.class)` to run initialization after all beans are ready:
   ```java
   @Component
   public class DataInitializer {
       @EventListener(ContextRefreshedEvent.class)
       public void init() {
           // All beans are fully initialized here
       }
   }
   ```

10. **Q: You have three beans of type `DataSource` (main DB, reporting DB, analytics DB). Your `@Repository` classes should each use a specific one. How do you wire this cleanly?**
    A: Use `@Qualifier` at both the bean declaration and injection point, or create custom qualifier annotations:
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

---

## Interview Questions

1. **What is Inversion of Control (IoC)?** 
   A: IoC is a principle where the control of object creation and lifecycle is transferred from the application to a container. Instead of calling `new`, objects declare their dependencies, and the Spring container injects them. This decouples object creation from business logic.

2. **What is the difference between `BeanFactory` and `ApplicationContext`?** 
   A: `BeanFactory` is the basic IoC container that lazily instantiates beans. `ApplicationContext` extends `BeanFactory` with eager singleton initialization, event publishing, internationalization (i18n), AOP integration, and `BeanPostProcessor` registration. Use `ApplicationContext` in almost all cases.

3. **How does Spring resolve a circular dependency with constructor injection?** 
   A: Spring cannot resolve circular dependencies with constructor injection — it throws `BeanCurrentlyInCreationException`. The fix involves: (a) using `@Lazy` on one side to defer initialization, (b) switching to setter injection on one side (Spring creates a proxy), or (c) restructuring code to eliminate the cycle, often by introducing events or extracting a shared dependency into a third class.

4. **What is the difference between `@Component` and `@Bean`?** 
   A: `@Component` is a class-level annotation that Spring auto-detects via component scanning. `@Bean` is a method-level annotation placed inside a `@Configuration` class that explicitly declares a bean. Use `@Component` for your own classes and `@Bean` for third-party classes you cannot annotate.

5. **What are bean scopes in Spring?** 
   A: Singleton (default) — one instance per container. Prototype — new instance per request/injection. Request — one per HTTP request. Session — one per HTTP session. Application — one per ServletContext. WebSocket — one per WebSocket session.

6. **How does `@Lazy` work and when would you use it?** 
   A: `@Lazy` defers bean instantiation until first use instead of at startup. Use it for expensive beans (ML models, report engines) that are not always needed, to reduce startup time. On `@Configuration` classes, it makes all `@Bean` methods lazy. With `@Autowired`, it creates a proxy that initializes the real bean on first method call.

7. **Explain the bean lifecycle in Spring.** 
   A: Bean definition loaded → `BeanFactoryPostProcessor` → Instantiation → Populate properties → Aware interfaces (`BeanNameAware`, `ApplicationContextAware`) → `BeanPostProcessor#postProcessBeforeInitialization` → `@PostConstruct` / `InitializingBean` / `init-method` → `BeanPostProcessor#postProcessAfterInitialization` (AOP proxies created here) → Ready for use → `@PreDestroy` / `DisposableBean` / `destroy-method`.

8. **What happens when a prototype bean is injected into a singleton bean?** 
   A: The prototype bean is created once when the singleton is instantiated. All future calls use the same prototype instance. To get a new instance each time, inject `ObjectFactory<T>`, `Provider<T>`, or use `@Lookup` method injection.

9. **What is a `BeanPostProcessor` and how does it differ from a `BeanFactoryPostProcessor`?** 
   A: A `BeanPostProcessor` operates on bean instances — it runs before and after initialization callbacks and can wrap beans with proxies (used for `@Transactional`, `@Async`). A `BeanFactoryPostProcessor` operates on bean definitions before any beans are created — it modifies property values or adds placeholder resolution. `BeanFactoryPostProcessors` run first.

10. **How do you conditionally register a bean in Spring?** 
    A: Use `@ConditionalOnProperty`, `@ConditionalOnClass`, `@ConditionalOnMissingBean`, or `@Profile`. For custom logic, implement `Condition` and use `@Conditional(MyCondition.class)`. Example: `@ConditionalOnProperty(name = "feature.x.enabled", havingValue = "true")`.

---

## Developer Recommendations

- **Use constructor injection over field injection** — Constructor injection makes dependencies explicit, enables immutability (`final` fields), and fails at compile time if a required bean is missing. Field injection hides dependencies and fails at runtime with a `NullPointerException`.
- **Use `@Service`, `@Repository`, `@Controller` over bare `@Component`** — Each provides semantic meaning and enables targeted AOP (e.g., Spring automatically translates persistence exceptions in `@Repository` classes). Using `@Component` everywhere loses this behavior and makes code harder to navigate.
- **Prefer `@Configuration` over XML** — Java config is type-safe, refactorable, and keeps bean definitions close to the code. XML config requires switching contexts and cannot be checked at compile time. Use `@ImportResource` only for gradual migration from legacy XML.
- **Use `@Qualifier` or custom qualifier annotations over `@Primary`** — `@Primary` silently picks a default when multiple beans exist, which can surprise future developers. Explicit `@Qualifier` makes the selection obvious. Custom qualifier annotations (like `@ReportingDb`) are even better because they convey business intent.
- **Avoid injecting `ApplicationContext` directly** — It ties your code to the Spring container and signals that the design may have a missing abstraction. Instead, inject the specific dependency needed. If you must access the container, implement `ApplicationContextAware` only as a last resort.
- **Use `@Lazy` for expensive beans that are not always needed** — Deferring bean creation improves startup time and reduces memory. However, be aware that the first request will be slower (paying the deferred cost). Document this trade-off clearly.
- **Use `ObjectFactory<T>` or `Provider<T>` for obtaining prototype beans from singletons** — Direct injection of a prototype bean into a singleton gives you only one instance. `ObjectFactory` calls `getBean()` on every request, respecting the prototype scope. This is cleaner than using `ApplicationContext.getBean()` manually.
- **Limit constructor parameters to 5-6; beyond that, refactor** — Too many constructor parameters indicates a class has too many responsibilities. Use facade, aggregate configuration objects, or split the class. The pain of excessive parameters is a design smell, not a Spring limitation.
