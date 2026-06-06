# Spring Bean Lifecycle

---

## What is the Spring Bean Lifecycle?

The **Spring Bean Lifecycle** defines the sequence of steps every bean goes through from instantiation to destruction. Understanding this lifecycle is critical for proper initialization logic, resource cleanup, AOP proxy behavior, and troubleshooting startup issues.

### Lifecycle Phases Overview

Every bean follows this path:

```
Bean Definition → FactoryPostProcessors → Instantiation →
Populate Properties → Aware Interfaces → Before Init →
@PostConstruct / init-method → After Init (AOP proxies created here) →
Ready for Use → @PreDestroy / destroy-method → Destroyed
```

### Step-by-Step Breakdown

- **Bean Definition** — Spring reads configuration (annotations, XML, or Java config) and creates `BeanDefinition` metadata objects for each bean. These definitions contain class names, scope, and property values.
- **BeanFactoryPostProcessors** — These modify bean definitions before any beans are instantiated. For example, `PropertySourcesPlaceholderConfigurer` resolves `${...}` placeholders in property values.
- **Instantiation** — The bean's constructor is called (with constructor arguments resolved via DI). The bean object comes into existence at this point.
- **Populate Properties** — Spring injects dependencies via setters or `@Autowired` fields using reflection. After this, all dependencies are available.
- **Aware Interfaces** — If the bean implements aware interfaces, Spring invokes them:
  - `BeanNameAware` — Injects the bean's name as defined in the container.
  - `BeanClassLoaderAware` — Injects the class loader used to load the bean class.
  - `ApplicationContextAware` — Injects the `ApplicationContext` for programmatic access to the container.
- **BeanPostProcessor#postProcessBeforeInitialization** — Called for ALL beans before initialization callbacks. Used for wrapping or modifying bean instances.
- **@PostConstruct** — The `javax.annotation.PostConstruct` (or `jakarta.annotation.PostConstruct`) annotated method runs. This is the standard initialization callback.
- **InitializingBean#afterPropertiesSet()** — Spring-specific interface alternative to `@PostConstruct`. Less commonly used but still supported.
- **@Bean(initMethod = "...")** — XML or Java config init method. The last initialization callback to execute.
- **BeanPostProcessor#postProcessAfterInitialization** — Called for ALL beans after initialization. **AOP proxies are created here** for annotations like `@Transactional`, `@Cacheable`, and `@Async`.
- **Bean is Ready** — The bean is fully initialized and available for use by the application.
- **@PreDestroy** — Cleanup callback before the bean is destroyed. Used for releasing resources.
- **DisposableBean#destroy()** — Spring-specific interface for destruction callbacks.
- **@Bean(destroyMethod = "...")** — Configured destroy method for custom cleanup logic.

### AOP Proxy Creation Timing

AOP proxies (for `@Transactional`, `@Cacheable`, `@Async`, etc.) are created in `BeanPostProcessor#postProcessAfterInitialization`. This has an important implication:

```java
@Service
public class MyService {

    @PostConstruct
    public void init() {
        // Runs on the RAW bean — before AOP proxies are created
        // Calling self.doSomething() here does NOT trigger AOP
    }

    @Transactional
    public void doSomething() { /* ... */ }
}
```

**Fix**: If you need transactional behavior during initialization, use `TransactionTemplate` programmatically.

---

## Core Concepts

### Initialization Callbacks

Three ways to define initialization logic, in order of execution:

```java
@Component
public class CacheInitializer {
    private final CacheManager cacheManager;

    public CacheInitializer(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    @PostConstruct
    public void warmCache() {
        // 1st: javax/jakarta annotation
        cacheManager.createCache("products");
    }
}
```

```java
public class MyBean implements InitializingBean {
    @Override
    public void afterPropertiesSet() {
        // 2nd: Spring interface
    }
}
```

```java
@Bean(initMethod = "init")
public MyBean myBean() {
    return new MyBean();
}
// public void init() → 3rd: configured method
```

### Destruction Callbacks

```java
@Component
public class ConnectionManager {
    private Connection connection;

    @PostConstruct
    public void connect() {
        this.connection = createConnection();
    }

    @PreDestroy
    public void disconnect() {
        if (connection != null) {
            connection.close();
        }
    }
}
```

### Lazy Initialization

Beans marked with `@Lazy` are instantiated only when first requested, not at startup:

```java
@Component
@Lazy
public class ExpensiveResource {
    public ExpensiveResource() {
        // Heavy initialization — deferred until first use
    }
}
```

Global lazy initialization: `spring.main.lazy-initialization=true` (defers all beans).

### Proxy Types

| Proxy Type | Requirement | Performance | How It Works |
|-----------|-------------|-------------|--------------|
| JDK Dynamic | Must implement an interface | Fast | Creates a proxy implementing the same interface |
| CGLIB | Any class (no interface needed) | Fast (slightly slower) | Creates a subclass at runtime |

CGLIB is used by default since Spring Boot 2.x. If the bean class is `final`, CGLIB cannot proxy it.

---

## Common Mistakes

- **Calling `@Transactional` from `@PostConstruct`** — The `@Transactional` does not work because AOP proxies are not yet created during `@PostConstruct`. Use `TransactionTemplate` or restructure to use `@EventListener(ContextRefreshedEvent.class)`.
- **Trying to use uninitialized dependencies in a constructor** — Dependencies are not yet injected during constructor execution. Use `@PostConstruct` for initialization that depends on injected fields, not the constructor itself.
- **Throwing exceptions in `@PostConstruct`** — The application context fails to refresh and the application does not start. Handle exceptions gracefully inside init methods with proper logging and fallback logic.
- **Not cleaning up resources** — Failing to implement `@PreDestroy` for closable resources causes memory leaks, exhausted connections, and file handle leaks over time.
- **Circular init dependencies** — Bean A depends on Bean B and Bean B depends on Bean A, causing `BeanCurrentlyInCreationException`. Use `@Lazy` on one side or restructure to eliminate the cycle.
- **Heavy initialization in singleton beans** — Slows application startup significantly and blocks the entire context refresh. Use `@Lazy` for expensive beans or move initialization to background threads.
- **`final` classes with AOP annotations** — CGLIB cannot create a proxy for final classes, so `@Transactional`, `@Cacheable`, and `@Async` are silently ignored. Either remove `final` or use an interface for JDK proxying.

---

## Real-World Scenarios

### Scenario 1: Database Connection Pool Initialization with Validation

An application connects to a PostgreSQL database. At startup, the connection pool must be tested before any requests arrive. A misconfigured database URL should fail the application startup immediately rather than causing failures on the first request.

```java
@Component
public class DatabaseValidator {
    private final DataSource dataSource;

    public DatabaseValidator(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    @PostConstruct
    public void validateConnection() {
        try (Connection conn = dataSource.getConnection()) {
            if (!conn.isValid(5)) {
                throw new IllegalStateException("Database connection pool validation failed");
            }
            log.info("Database connection validated successfully");
        } catch (SQLException e) {
            throw new IllegalStateException("Cannot connect to database at startup", e);
        }
    }

    @PreDestroy
    public void closePool() {
        if (dataSource instanceof HikariDataSource hikari) {
            hikari.close();
            log.info("Database connection pool closed");
        }
    }
}
```

### Scenario 2: Cache Warming After AOP Proxy Creation

A product catalog service uses `@Cacheable` for product lookups. The cache is empty at startup, and the first user to request a popular product experiences a 3-second wait. The cache should be pre-warmed with top-selling products.

```java
@Service
public class ProductService {
    private final ProductRepository productRepository;

    public ProductService(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    @EventListener(ContextRefreshedEvent.class)
    public void warmCache() {
        // Runs AFTER AOP proxies are created — @Cacheable works!
        List<Product> topProducts = productRepository.findTop100ByOrderBySales();
        topProducts.forEach(p -> findById(p.getId())); // Populates cache
        log.info("Warmed cache with {} products", topProducts.size());
    }

    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        // Expensive database call
        return productRepository.findById(id).orElseThrow();
    }
}
```

### Scenario 3: Graceful Shutdown with Resource Cleanup

A messaging service consumes RabbitMQ messages. On application shutdown, the service must finish processing the current message and cleanly close the connection before the JVM exits. Unprocessed messages should be requeued.

```java
@Component
public class MessageConsumer {
    private final AtomicBoolean running = new AtomicBoolean(true);

    @EventListener(ContextClosedEvent.class)
    public void shutdown() {
        log.info("Shutdown signal received, finishing current message...");
        running.set(false);
        // Let the listener loop finish its current message
    }

    @PreDestroy
    public void cleanup() {
        log.info("Closing RabbitMQ connection...");
        // Close channel and connection
        rabbitChannel.close();
    }

    public void consumeLoop() {
        while (running.get() && !Thread.currentThread().isInterrupted()) {
            // Process one message
        }
    }
}
```

---

## Scenario-Based Questions

1. **Q: Your `@PostConstruct` method calls `cacheManager.getCache("products")` but the cache manager is null even though it's injected via the constructor. The constructor shows it's non-null. What's happening?**
   A: This is a timing issue within the bean's own constructor. The injected `cacheManager` is set on the field before the constructor body executes, so it IS non-null in the constructor. However, the `CacheManager` bean itself may not be fully initialized yet — its `@PostConstruct` may not have run. If `CacheManager` has its own dependencies that are initialized after your bean, those are not ready. The fix: move cache-warming logic to `@EventListener(ContextRefreshedEvent.class)` which fires after ALL beans are initialized, or use `@DependsOn("cacheManager")` to ensure ordering.

2. **Q: You implement `BeanPostProcessor` to automatically log every bean's initialization time. Your `BeanPostProcessor` itself seems to skip logging. Why?**
   A: A `BeanPostProcessor` is initialized early in the container lifecycle — before other beans' `@PostConstruct` and `BeanPostProcessor` methods run. However, `BeanPostProcessor` beans themselves go through a special early initialization. Their own post-processing methods are called when the application context processes them, but some beans instantiated before your `BeanPostProcessor` (like infrastructure beans) already passed through the lifecycle. The fix: implement `PriorityOrdered` to control ordering, and log at `postProcessAfterInitialization` which runs after the bean's complete initialization.

3. **Q: Your application context refreshes successfully in development but fails in production with `BeanCreationException` in a `@PostConstruct` that reads a file. Both environments have the same Docker image. What's different?**
   A: The `@PostConstruct` likely reads a file from the filesystem that exists in the development Docker container but not in production. The classpath resource `classpath:data/seed.json` should work in both if the JAR contains it. But an absolute path like `/app/config/data.json` may not exist in production. Fix: Always use classpath resources (`classpath:...`) for files bundled with the application. For external files, use `@Value("${app.config.path}")` with a default that works in both environments, and handle `FileNotFoundException` gracefully in `@PostConstruct`.

4. **Q: You have a bean that implements `SmartLifecycle` with `isRunning()` returning `true` before the bean is fully initialized. Downstream beans that depend on it start using it prematurely. How do you fix this?**
   A: The `SmartLifecycle` contract requires `isRunning()` to accurately reflect the bean's operational state. Set a volatile flag to `true` only after initialization completes:
   ```java
   @Component
   public class ConnectionPoolManager implements SmartLifecycle {
       private volatile boolean running = false;

       @Override
       public void start() {
           // Initialize connection pool
           running = true;
       }

       @Override
       public void stop() {
           running = false;
           // Close connections
       }

       @Override
       public boolean isRunning() {
           return running; // Accurate — only true after start()
       }
   }
   ```

5. **Q: You annotate a `@Configuration` class with `@Lazy`. Sub-beans defined in that configuration are still eagerly initialized. Why?**
   A: `@Lazy` on a `@Configuration` class makes the `@Bean` methods lazy — each bean is created on first request, not at startup. However, if other beans inject these lazy beans eagerly (via constructor injection at their own startup), the lazy beans are created anyway. Also, `@Autowired` on a collection (`List<Bean>`) triggers eager creation of all matching beans. The fix: also mark the injection points as `@Lazy` or use `ObjectFactory<T>` / `Provider<T>` for injection.

6. **Q: Your `@PreDestroy` method closes a network connection. During a rapid restart (undeploy + deploy), the `@PreDestroy` throws an exception because the connection was already closed by the container. How do you make `@PreDestroy` idempotent?**
   A: Make all `@PreDestroy` methods idempotent — they should handle the case where cleanup has already been partially performed:
   ```java
   @PreDestroy
   public void cleanup() {
       if (connection != null && !connection.isClosed()) {
           try {
               connection.close();
           } catch (SQLException e) {
               log.warn("Error closing connection (may already be closed)", e);
           }
       }
       connection = null; // Mark as cleaned up
   }
   ```
   Use a flag or null-check pattern. Never assume `@PreDestroy` is called exactly once or in a specific order.

7. **Q: You have 200 beans with heavy initialization logic in `@PostConstruct`. The application startup takes 8 minutes. Your SRE team requires startup under 60 seconds. What strategies do you use?**
   A: Multiple approaches: (a) Move expensive initialization to `@EventListener(ContextRefreshedEvent.class)` — runs after startup, doesn't block the context refresh. (b) Use `@Lazy` on beans not needed immediately. (c) Use `spring.main.lazy-initialization=true` for global lazy init. (d) Profile startup to identify specific bottlenecks. (e) Move initialization to a background thread with `@Async`. (f) Cache pre-computation — compute expensive data at build time and load it at startup. (g) Use Spring Boot 3.x AOT with `spring-aot` plugin for ahead-of-time processing.

8. **Q: Your `@Bean(initMethod = "init")` and `@PostConstruct` both specify initialization logic. Which one runs first and what happens if the `initMethod` throws an exception?**
   A: `@PostConstruct` runs first (handled by `InitDestroyAnnotationBeanPostProcessor`), then `InitializingBean.afterPropertiesSet()`, and finally the custom `initMethod`. If `initMethod` throws an exception, the bean is in an inconsistent state — `@PostConstruct` already ran, but AOP proxies were already created (proxies are created after all init callbacks). The exception propagates as a `BeanCreationException`, and the bean is discarded. To avoid this, validate in `@PostConstruct` and reserve `initMethod` for non-critical side effects.

9. **Q: Your bean implements `BeanNameAware` but `setBeanName()` is called with the wrong name (a different name than the one you used in `@Bean(name = "myCustomName")`). What could be wrong?**
   A: If your bean is defined both via `@Component` (class-level) and `@Bean` (method-level in a `@Configuration` class), the `@Bean` method definition overrides the component scan. But `BeanNameAware` in this case gets the bean name from the `@Bean` method. If you see a different name, check: (a) Is there another `@Bean` method with the same return type that overrides? (b) Is the class also registered via XML? (c) Is there a `BeanNameGenerator` that customizes naming? Use `@Qualifier` for precise identification instead of relying on bean names.

10. **Q: You have a `BeanPostProcessor` that wraps beans with a proxy. The proxy works, but `@PostConstruct` on the target bean runs twice. Why?**
    A: This happens when the `BeanPostProcessor` creates a new instance (rather than wrapping the original) and the new instance also goes through the lifecycle. A `BeanPostProcessor` should return the SAME bean instance (possibly wrapped/proxied), not a different instance. If you return a new object, that new object may also trigger `BeanPostProcessor` processing, creating an infinite loop or double initialization. Always wrap the original bean: `return Proxy.newProxyInstance(bean.getClass().getClassLoader(), interfaces, handler)` — this returns a proxy of the same instance.

---

## Interview Questions

1. **What are the main phases of the Spring Bean Lifecycle?** 
   A: Bean definition → `BeanFactoryPostProcessor` → Instantiation → Populate properties → Aware interfaces (`BeanNameAware`, `ApplicationContextAware`) → `BeanPostProcessor#postProcessBeforeInitialization` → `@PostConstruct` / `InitializingBean` / `init-method` → `BeanPostProcessor#postProcessAfterInitialization` (AOP proxy created here) → Bean ready → `@PreDestroy` / `DisposableBean` / `destroy-method`.

2. **What is the difference between `BeanFactoryPostProcessor` and `BeanPostProcessor`?** 
   A: `BeanFactoryPostProcessor` operates on bean *definitions* before any beans are instantiated (can modify property values, add placeholders). `BeanPostProcessor` operates on bean *instances* after instantiation but before/after initialization callbacks (can wrap with proxies, modify instances). `BeanFactoryPostProcessors` run first.

3. **When are AOP proxies created in the bean lifecycle?** 
   A: AOP proxies are created in `BeanPostProcessor#postProcessAfterInitialization`. This means that when `@PostConstruct` runs, the bean is still the raw instance — AOP annotations (`@Transactional`, `@Cacheable`, `@Async`) are NOT yet active. This is why calling a `@Transactional` method from `@PostConstruct` does not start a transaction.

4. **What is the execution order of `@PostConstruct`, `InitializingBean.afterPropertiesSet()`, and `init-method`?** 
   A: `@PostConstruct` runs first (Jakarta annotation, processed by `InitDestroyAnnotationBeanPostProcessor`). Then `InitializingBean.afterPropertiesSet()` runs (Spring interface). Finally, the custom `init-method` specified in `@Bean(initMethod = "...")` runs.

5. **What happens if `@PostConstruct` throws an exception?** 
   A: The exception propagates as a `BeanCreationException`. The `ApplicationContext.refresh()` fails and the application does not start. Spring treats initialization failures as fatal. The bean is discarded and not available for injection.

6. **What is the purpose of `@PreDestroy`?** 
   A: `@PreDestroy` is a callback method invoked before a bean is destroyed (during application shutdown or context close). Use it to release resources: close connections, stop threads, flush caches. If `@PreDestroy` throws an exception, Spring logs it and continues destroying other beans.

7. **What is the `SmartLifecycle` interface and when do you use it?** 
   A: `SmartLifecycle` extends `Lifecycle` with ordering support via `getPhase()`. Use it when a bean needs controlled startup/shutdown sequencing — for example, starting a message listener after the database connection pool is ready, and stopping it before the pool is closed. Lower phase values start first and stop last.

8. **How does `@Lazy` affect the bean lifecycle?** 
   A: `@Lazy` defers bean instantiation until the first request. The bean skips eager initialization at startup but goes through the same lifecycle phases (instantiation, population, init callbacks) when first accessed. For `@Lazy @Bean` methods, the method is not called until the bean is requested.

9. **What is `@DependsOn` and when would you use it?** 
   A: `@DependsOn` ensures that specified beans are initialized before the annotated bean. Use it when a bean has implicit dependencies that Spring cannot detect (e.g., a bean that reads a configuration file populated by another bean's `@PostConstruct`, or initializing a database schema before using it).

10. **What is a `BeanPostProcessor` and can you give an example of how Spring uses one?** 
    A: A `BeanPostProcessor` hooks into the bean lifecycle to modify or wrap bean instances. Spring uses them extensively: `AutowiredAnnotationBeanPostProcessor` processes `@Autowired`, `InitDestroyAnnotationBeanPostProcessor` processes `@PostConstruct`/`@PreDestroy`, `AbstractAutoProxyCreator` creates AOP proxies for `@Transactional`, `@Async`, etc.

---

## Developer Recommendations

- **Use `@EventListener(ContextRefreshedEvent.class)` instead of `@PostConstruct` for logic that needs AOP proxies** — AOP proxies (for `@Transactional`, `@Cacheable`, `@Async`) are created *after* `@PostConstruct`. Any initialization that relies on these annotations should use `@EventListener` which fires after the full context is refreshed and all proxies are ready.
- **Always make `@PreDestroy` methods idempotent** — `@PreDestroy` may be called multiple times in edge cases (rapid restart, context refresh). Use null checks or a `closed` flag to ensure cleanup logic runs safely only once.
- **Use `SmartLifecycle` for beans that need ordered startup and shutdown** — If bean A must start before bean B and stop after bean B, `SmartLifecycle.getPhase()` provides explicit ordering. This is essential for message listeners, connection pools, and background task managers.
- **Keep `@PostConstruct` methods simple and fast** — Heavy initialization in `@PostConstruct` blocks the entire application startup. Move expensive operations (cache warming, data loading, external API calls) to background threads, `@Async`, or `@EventListener(ContextRefreshedEvent.class)`.
- **Use `@DependsOn` sparingly and document why** — Excessive `@DependsOn` indicates tight coupling. If bean A depends on bean B's initialization, consider using events instead of direct dependencies. When `@DependsOn` is necessary, document the implicit dependency clearly.
- **Never throw checked exceptions from `@PostConstruct`** — `@PostConstruct` methods can only throw unchecked exceptions. If your initialization logic throws a checked exception, wrap it in a `RuntimeException`. The application will fail to start, but the error message will be clear.
- **Profile startup time with a custom `BeanPostProcessor`** — Implement `BeanPostProcessor` with timing to identify slow beans. Log the time for each bean's `postProcessBeforeInitialization` and `postProcessAfterInitialization`. Focus optimization efforts on beans that take > 1 second.
- **Be aware that `final` classes cannot be proxied by CGLIB** — If a class is `final`, Spring cannot create an AOP proxy via CGLIB. Annotations like `@Transactional`, `@Cacheable`, and `@Async` will be silently ignored. Either remove `final`, implement an interface (JDK proxy), or use `@Scope(proxyMode = ScopedProxyMode.INTERFACES)`.
