# Spring Dependency Injection

---

## What is Dependency Injection?

**Dependency Injection (DI)** is a design pattern where objects receive their dependencies from an external source — the IoC container — rather than creating them internally. Spring's DI implementation is the core mechanism behind the framework's **loose coupling** and **testability**.

### Key Concepts:

1. **Why Use DI**:

   - **Decoupling** — Classes depend on abstractions (interfaces), not concrete implementations. You can swap implementations without changing dependent code.
   - **Testability** — Dependencies can be replaced with mocks or stubs in unit tests.
   - **Lifecycle Management** — The container handles creation, initialization, and destruction of objects.
   - **Flexibility** — Different configurations can inject different implementations for different environments (dev, test, production).

2. **Injection Types**:

   - **Constructor Injection** — Dependencies are provided through constructor arguments. This is the **recommended approach** for required dependencies. Beans are immutable and always fully initialized.
   - **Setter Injection** — Dependencies are set via setter methods. Best for **optional dependencies** that have sensible defaults.
   - **Field Injection** — Dependencies are injected directly into fields via reflection. **Avoid in production code** — it hides dependencies, makes testing harder, and prevents immutability.

3. **Constructor Injection (Recommended)** :

   ```java
   @Service
   public class OrderService {
       private final OrderRepository orderRepository;
       private final PaymentGateway paymentGateway;
       private final NotificationService notificationService;

       // Since Spring 4.3, @Autowired is optional on constructors
       // when there is only one constructor
       public OrderService(OrderRepository orderRepository,
                           PaymentGateway paymentGateway,
                           NotificationService notificationService) {
           this.orderRepository = orderRepository;
           this.paymentGateway = paymentGateway;
           this.notificationService = notificationService;
       }
   }
   ```

4. **Setter Injection (For Optional Dependencies)** :

   ```java
   @Service
   public class EmailService {
       private MailSender mailSender;

       @Autowired(required = false)
       public void setMailSender(MailSender mailSender) {
           this.mailSender = mailSender;
       }
   }
   ```

5. **Field Injection (Avoid)** :

   ```java
   @Service
   public class ReportService {
       @Autowired
       private ReportRepository repository; // Hidden, non-final
   }
   ```

---

## Core Concepts

### 1. Handling Multiple Implementations

   When multiple beans implement the same interface:

   ```java
   public interface PaymentGateway { void process(Payment payment); }

   @Component
   @Qualifier("stripe")
   public class StripeGateway implements PaymentGateway { }

   @Component
   @Qualifier("paypal")
   public class PayPalGateway implements PaymentGateway { }

   // Use @Qualifier to select
   @Service
   public class CheckoutService {
       private final PaymentGateway gateway;

       public CheckoutService(@Qualifier("stripe") PaymentGateway gateway) {
           this.gateway = gateway;
       }
   }

   // Or @Primary for a default
   @Component
   @Primary
   public class DefaultGateway implements PaymentGateway { }
   ```

### 2. `@Value` Injection

   Inject properties, SpEL expressions, and resources:

   ```java
   @Service
   public class AppConfig {
       @Value("${app.name}")
       private String appName;

       @Value("${app.timeout:5000}") // Default 5000
       private int timeout;

       @Value("#{systemProperties['user.dir']}") // SpEL
       private String userDir;

       @Value("classpath:data/seed.json")
       private Resource seedFile;
   }
   ```

### 3. Autowiring Resolution Process

   When Spring encounters `@Autowired`, it follows this resolution:

   ```
   1. Find matching bean(s) by type
   2. If exactly one → inject it
   3. If more than one → look for @Primary → @Qualifier
   4. If none and required=true → throw NoSuchBeanDefinitionException
   5. If none and required=false → leave null
   ```

### 4. Factory Pattern with DI

   Inject all implementations of an interface and dispatch dynamically:

   ```java
   @Component
   public class NotificationFactory {
       private final Map<NotificationType, NotificationSender> senders;

       public NotificationFactory(List<NotificationSender> senderList) {
           this.senders = senderList.stream()
               .collect(Collectors.toMap(
                   NotificationSender::getType, Function.identity()
               ));
       }

       public NotificationSender getSender(NotificationType type) {
           NotificationSender sender = senders.get(type);
           if (sender == null) {
               throw new IllegalArgumentException("Unknown type: " + type);
           }
           return sender;
       }
   }
   ```

### 5. Conditional Injection with `@Conditional`

   Create beans conditionally based on environment, properties, or classpath:

   ```java
   @Configuration
   public class DataSourceConfig {

       @Bean
       @ConditionalOnProperty(name = "db.type", havingValue = "mysql")
       public DataSource mysqlDataSource() { return new MySQLDataSource(); }

       @Bean
       @ConditionalOnProperty(name = "db.type", havingValue = "postgres")
       public DataSource postgresDataSource() { return new PostgresDataSource(); }

       @Bean
       @ConditionalOnMissingBean(DataSource.class)
       public DataSource h2DataSource() { return new H2DataSource(); }

       @Bean
       @Profile("dev")
       public DataSource devDataSource() { return new H2DataSource(); }

       @Bean
       @Profile("prod")
       public DataSource prodDataSource() { return new ConnectionPoolDataSource(); }
   }
   ```

### 6. Circular Dependencies

   Circular dependencies occur when Bean A depends on Bean B and Bean B depends on Bean A:

   ```java
   @Service
   public class A {
       private final B b;
       public A(B b) { this.b = b; }
   }

   @Service
   public class B {
       private final A a;
       public B(A a) { this.a = a; }
   }
   ```

   - With **constructor injection**, Spring throws `BeanCurrentlyInCreationException`.
   - With **setter/field injection**, Spring creates a proxy to break the cycle.
   - **Fix**: Extract shared interface, use events, use `@Lazy` on one side, or restructure.

---

## Common Mistakes

1. **Using field injection** — Dependencies are hidden, not final, and the class cannot be instantiated without Spring's container. Always prefer constructor injection.

2. **Too many constructor parameters** — More than 6-7 parameters indicates the class has too many responsibilities. Extract related dependencies into a single object.

3. **Not using `@Qualifier` when needed** — Spring throws `NoUniqueBeanDefinitionException`. Always qualify when multiple beans of the same type exist.

4. **Circular dependencies with constructor injection** — Cannot be resolved. Use `@Lazy`, setter injection, or restructure.

5. **Setting `@Autowired(required = false)` on a constructor** — Does not work the same way as on fields. Use `Optional<T>` for optional constructor dependencies.

6. **Injecting `ApplicationContext` directly** — While possible, it ties your code to the Spring container. Prefer injecting the specific dependency instead.

7. **Mixing injection types inconsistently** — Pick constructor injection as the standard and use it everywhere for consistency.

---

## Real-World Scenarios

### Scenario 1: Microservices Refactor — From Field Injection to Constructor Injection

A team inherits a legacy payment service where every class uses `@Autowired` field injection. During a critical production incident, the `PaymentProcessor` NPEs because `transactionManager` was never injected — but the error only appeared under load when a bean creation race condition occurred at startup.

```java
// Before — field injection: hidden dependencies, no immutability, runtime failures
@Service
public class PaymentProcessor {
    @Autowired private TransactionManager transactionManager;
    @Autowired private NotificationService notificationService;
    @Autowired private AuditService auditService;

    public void process(Payment payment) {
        // NPE here under load — transactionManager is null
    }
}

// After — constructor injection: explicit, immutable, compile-time safety
@Service
public class PaymentProcessor {
    private final TransactionManager transactionManager;
    private final NotificationService notificationService;
    private final AuditService auditService;

    public PaymentProcessor(TransactionManager transactionManager,
                            NotificationService notificationService,
                            AuditService auditService) {
        this.transactionManager = transactionManager;
        this.notificationService = notificationService;
        this.auditService = auditService;
    }
}
```

The refactor turned a race-condition NPE into a clean startup failure, and made unit testing trivial (just pass mocks to the constructor).

### Scenario 2: Multi-Cloud Deployment with Conditional Bean Registration

An application deploys to AWS, GCP, and on-prem. Each environment uses different implementations for blob storage, queue, and secrets manager. The code should not be littered with `if (env == AWS)` checks.

```java
@Configuration
public class CloudConfig {
    @Bean @Profile("aws")
    public BlobStorage awsS3Storage() { return new S3Storage(); }

    @Bean @Profile("gcp")
    public BlobStorage gcsStorage() { return new GcsStorage(); }

    @Bean @Profile("onprem")
    public BlobStorage localFileStorage() { return new LocalFileStorage(); }

    @Bean @ConditionalOnMissingBean
    public BlobStorage devStorage() { return new InMemoryStorage(); }
}

@Service
public class DocumentService {
    private final BlobStorage storage;
    // No environment-specific logic — Spring injects the right one
    public DocumentService(BlobStorage storage) { this.storage = storage; }
}
```

### Scenario 3: Notification Router with Dynamic Strategy Selection

A notification system sends alerts via email, SMS, push, and Slack. The channel depends on the user's preferences, the urgency, and the time of day. Using a strategy pattern with injected implementations keeps it clean.

```java
@Component
public class NotificationRouter {
    private final Map<Channel, NotificationSender> senders;

    public NotificationRouter(List<NotificationSender> senderList) {
        this.senders = senderList.stream()
            .collect(Collectors.toMap(NotificationSender::getChannel, Function.identity()));
    }

    public void send(User user, Message message) {
        Channel channel = determineChannel(user, message);
        NotificationSender sender = senders.get(channel);
        if (sender == null) {
            throw new IllegalArgumentException("No sender for channel: " + channel);
        }
        sender.send(user, message);
    }
}
```

Adding a new channel (e.g., WhatsApp) requires only a new `@Component` class — no changes to the router.

---

## Scenario-Based Questions

1. **Q: Your team has 200+ Spring beans with field injection. A production outage occurs because a required bean was not injected (field stayed null) — but only on a specific server configuration. The issue passed unit tests because tests use `@InjectMocks`. How do you prevent this class of bugs?**
   A: Enforce constructor injection at the code review and tooling level. Use ArchUnit or a custom Checkstyle/PMD rule to ban `@Autowired` on fields. Enable Intellij IDEA's "Field injection is not recommended" inspection as an error. The root cause is that field injection uses reflection and bypasses the constructor — the class can be instantiated in an invalid state. Constructor injection guarantees that once the object exists, all dependencies are present and `final`.

2. **Q: You have an `OrderService` that needs to send notifications. Depending on the order type (express, standard, international), a different `NotificationSender` implementation should be used. The choice depends on a runtime value, not on bean qualifiers. How do you design this?**
   A: Use the strategy pattern with injected implementations via a `Map<OrderType, NotificationSender>`:
   ```java
   @Component
   public class NotificationStrategy {
       private final Map<OrderType, NotificationSender> senders;
       public NotificationStrategy(List<NotificationSender> senderList) {
           this.senders = senderList.stream()
               .collect(Collectors.toMap(NotificationSender::getType, Function.identity()));
       }
       public NotificationSender forOrder(Order order) {
           return senders.get(order.getType());
       }
   }
   ```
   Spring auto-populates the `List` with all `NotificationSender` beans. The strategy is decoupled from both the senders and the callers — adding a new order type requires only a new `@Component`.

3. **Q: Your `@Configuration` class has 15 `@Bean` methods and 20 `@Value` injections. The class is hard to read and every change risks breaking the bean wiring. How do you refactor this?**
   A: Split the large `@Configuration` class into multiple focused configuration classes by concern (e.g., `DataSourceConfig`, `SecurityConfig`, `CacheConfig`, `MessagingConfig`). Group related `@Bean` methods and `@Value` fields together. Use `@ConfigurationProperties` with `@EnableConfigurationProperties` to move property bindings out of `@Configuration` classes entirely:
   ```java
   @ConfigurationProperties(prefix = "app.datasource")
   public record DataSourceProperties(String url, String username, String password, int poolSize) {}
   ```

4. **Q: You need to inject `RestTemplate` into 20 different service classes. URL, timeouts, and interceptors differ for each external API. Creating 20 `@Bean` methods feels wrong. How do you handle this?**
   A: Create a `RestTemplateBuilder` factory or a custom qualifier per API:
   ```java
   @Configuration
   public class RestTemplateConfig {
       @Bean @Qualifier("paymentApi")
       public RestTemplate paymentApiRestTemplate(RestTemplateBuilder builder) {
           return builder.rootUri("https://payment.example.com")
               .setConnectTimeout(Duration.ofSeconds(5))
               .build();
       }
       @Bean @Qualifier("shippingApi")
       public RestTemplate shippingApiRestTemplate(RestTemplateBuilder builder) {
           return builder.rootUri("https://shipping.example.com")
               .setConnectTimeout(Duration.ofSeconds(2))
               .build();
       }
   }
   ```
   Each service uses `@Qualifier("paymentApi")` to pick the right template. The bootstrapping complexity is centralized in the config class.

5. **Q: A developer created a `@Service` that instantiates its dependencies using `new` inside the constructor: `this.service = new EmailService()`. The `EmailService` has its own injected dependencies that are now null. Why does this break DI?**
   A: Using `new EmailService()` bypasses the Spring container entirely. Spring never processes the manually created `EmailService` — no `@Autowired`, `@Value`, `@PostConstruct`, or AOP annotations work on it. The fix is to inject `EmailService` via the constructor and let Spring create it:
   ```java
   @Service
   public class NotificationService {
       private final EmailService emailService;
       public NotificationService(EmailService emailService) {
           this.emailService = emailService; // Spring creates and injects it
       }
   }
   ```
   If `NotificationService` needs to create `EmailService` at runtime (not at startup), inject `ObjectFactory<EmailService>` or use `@Lookup`.

6. **Q: You have 5 implementations of `PaymentGateway`. Each should be used for a specific currency (USD → Stripe, EUR → Adyen, GBP → PayPal, etc.). You don't want to modify the router every time a gateway is added. How do you design the wiring?**
   A: Make each `PaymentGateway` implementation self-declare its supported currency via a method in the interface:
   ```java
   public interface PaymentGateway {
       String getCurrencyCode();
       PaymentResult process(Payment payment);
   }

   @Component
   public class StripeGateway implements PaymentGateway {
       @Override public String getCurrencyCode() { return "USD"; }
   }

   @Component
   public class PaymentRouter {
       private final Map<String, PaymentGateway> gatewayMap;
       public PaymentRouter(List<PaymentGateway> gateways) {
           this.gatewayMap = gateways.stream()
               .collect(Collectors.toMap(PaymentGateway::getCurrencyCode, Function.identity()));
       }
       public PaymentGateway forCurrency(String currencyCode) {
           return gatewayMap.get(currencyCode);
       }
   }
   ```
   Adding a new currency-gateway pair requires only a new `@Component` class — no wiring changes.

7. **Q: Your unit tests use `@Mock` and `@InjectMocks` to test classes with field injection. After switching to constructor injection, `@InjectMocks` still works but you notice tests are more explicit. Is this expected?**
   A: Yes. With constructor injection, `@InjectMocks` resolves constructor parameters by type from the mocks in the test context. The tests become more explicit because the constructor signature documents every dependency. If a new dependency is added, the test constructor call breaks at compile time (not at test runtime), forcing the developer to provide a mock. With field injection, adding `@Autowired` doesn't break the test, and the mock might be forgotten, causing a confusing NPE.

8. **Q: Your microservice uses `@RefreshScope` on beans that should reload when configuration changes. But some beans that depend on `@RefreshScope` beans do not refresh. What's happening?**
   A: `@RefreshScope` creates a proxy that recreates the bean when configuration changes. However, if another singleton bean caches the reference to the old proxy or unwraps the proxy, it won't see the refreshed instance. Always inject `@RefreshScope` beans via interfaces (not concrete classes) and never unwrap the proxy. For beans that need the refreshed properties, also mark them as `@RefreshScope` or inject a `@Value` directly.

9. **Q: Your application starts correctly locally but fails in production with `NoSuchBeanDefinitionException` for a bean that clearly exists. Both environments run the same codebase. What could cause this?**
   A: Production likely has a different classpath or profile. Check: (a) Is the bean's class excluded from component scanning in production due to a different package structure? (b) Is the bean annotated with `@Profile("dev")`? (c) Does a production-only library dependency create a conflict? (d) Is there a `@ConditionalOnProperty` that evaluates differently in production? Use `--debug` on startup to see which beans were registered and why.

10. **Q: Your `@ConfigurationProperties` class has 30 fields. It works but initialization is fragile and property binding errors are hard to debug. How do you improve this?**
    A: Use records for immutable config and enable validation:
    ```java
    @ConfigurationProperties(prefix = "app.order")
    @Validated
    public record OrderProperties(
        @NotNull String apiUrl,
        @Min(1) @Max(60) int timeoutSeconds,
        @NotEmpty String apiKey,
        @NotNull Duration retryDelay,
        List<@NotBlank String> supportedCurrencies
    ) {}
    ```
    Records are immutable by design, have no boilerplate, and fail early with clear errors. Add `spring-boot-configuration-processor` for IDE autocompletion.

---

## Interview Questions

1. **What is Dependency Injection and why is it used?** 
   A: DI is a pattern where objects receive their dependencies from an external container rather than creating them internally. It enables loose coupling, testability (mocks can be injected), and flexibility (implementations can be swapped without changing code).

2. **What are the three types of injection in Spring? Which is preferred?** 
   A: Constructor injection (dependencies via constructor arguments — preferred for required deps), setter injection (via setter methods — for optional deps), field injection (via reflection on fields — avoid in production). Constructor injection is preferred because it enables immutability (`final` fields), guarantees the object is fully initialized, and fails at compile/build time if a dependency is missing.

3. **What is the difference between `@Autowired`, `@Inject`, and `@Resource`?** 
   A: `@Autowired` (Spring-specific) wires by type. `@Inject` (Jakarta CDI) is equivalent to `@Autowired`. `@Resource` (Jakarta) wires by name first, then by type. Prefer `@Autowired` in Spring applications for consistency.

4. **How does Spring resolve ambiguity when multiple beans of the same type exist?** 
   A: Spring follows: (1) If exactly one bean → inject it. (2) If multiple → look for `@Primary`. (3) If `@Primary` is present → inject that one. (4) If not → look for `@Qualifier`. (5) If no qualifier → throw `NoUniqueBeanDefinitionException`. Use `@Qualifier` for explicit selection or `@Primary` for a default.

5. **What is `@Qualifier` and when do you use it?** 
   A: `@Qualifier` specifies which bean to inject when multiple beans of the same type exist. It can be used on the bean definition (to name it) and on the injection point (to select it). Custom qualifier annotations (e.g., `@StripeGateway`) improve readability.

6. **What is the difference between `@Component`, `@Service`, `@Repository`, and `@Controller`?** 
   A: All are stereotypes for Spring-managed beans. `@Service` marks business logic, `@Repository` marks DAOs and enables persistence exception translation, `@Controller` marks web controllers. `@Component` is the generic stereotype. Using specialized annotations makes the layer intent clear and enables targeted AOP.

7. **How do you inject values from properties files?** 
   A: Use `@Value("${property.key}")` for individual values, or `@ConfigurationProperties(prefix = "app")` for groups of related properties. `@Value` supports SpEL expressions and default values: `@Value("${app.timeout:5000}")`.

8. **What is the Spring bean autowiring process?** 
   A: Spring (1) identifies beans by type, (2) if multiple candidates, looks for `@Primary` and `@Qualifier`, (3) if no match and `required=true`, throws `NoSuchBeanDefinitionException`, (4) if `required=false`, leaves the dependency null. For collections (`List<T>`, `Map<String,T>`), Spring injects all beans of type T.

9. **What is a circular dependency and how do you resolve it?** 
   A: A circular dependency occurs when BeanA depends on BeanB and BeanB depends on BeanA. Constructor injection throws `BeanCurrentlyInCreationException`. Fixes: (a) `@Lazy` on one side, (b) setter injection on one side, (c) extract shared logic into a third class, (d) use events to decouple.

10. **What is the difference between `@Lookup` and `ObjectFactory<T>` for prototype bean injection?** 
    A: Both provide a way to get a new prototype instance from a singleton. `@Lookup` is a method-level annotation where Spring overrides the method to call `applicationContext.getBean()`. `ObjectFactory<T>` is a field/constructor injection that calls `getObject()`. `ObjectFactory` is more explicit and testable.

---

## Developer Recommendations

- **Use constructor injection over field injection** — Constructor injection makes dependencies explicit, enables immutability (`final` fields), and fails at compile time if a required bean is missing. Field injection hides dependencies and fails at runtime with a `NullPointerException`.
- **Use `@Qualifier` or custom qualifier annotations over `@Primary` alone** — `@Primary` silently picks a default when multiple beans exist, which can surprise future developers. Always use explicit `@Qualifier` on injection points. Even better, create custom qualifier annotations like `@StripeGateway` that convey business meaning.
- **Keep constructor parameters between 1 and 6** — A constructor with 8+ parameters indicates the class has too many responsibilities. Extract related dependencies into a single `@ConfigurationProperties` record or aggregate configuration object. The constructor should document the class's true dependencies.
- **Use `List<T>` or `Map<String, T>` injection for strategy patterns** — Instead of hard-coding which implementation to use, inject all implementations of an interface. Spring auto-populates the collection. This follows the Open/Closed Principle — adding a new implementation requires no changes to the consuming code.
- **Never use `new` to create classes that have injected dependencies** — Creating a dependency with `new` bypasses the Spring container entirely. No `@Autowired`, `@Value`, `@PostConstruct`, or AOP annotations work. Always inject such dependencies and let Spring manage their lifecycle.
- **Use `ObjectFactory<T>` or `Provider<T>` to get prototype beans from singletons** — Directly injecting a prototype-scoped bean into a singleton fixes it at creation time — you get only one instance. `ObjectFactory` calls `getBean()` on every request, respecting the prototype scope.
- **Use `@ConfigurationProperties` over many `@Value` fields** — A class with 10+ `@Value` annotations is hard to test, refactor, and validate. `@ConfigurationProperties` groups related properties, supports validation, provides IDE autocompletion with the configuration processor, and enables relaxed binding.
- **Mark `@Configuration` classes with `@Configuration`, not `@Component`** — `@Configuration` enables CGLIB proxying, which ensures that `@Bean` inter-method calls return singleton instances from the container. `@Component` does not proxy inter-bean references, causing each call to create a new instance.
