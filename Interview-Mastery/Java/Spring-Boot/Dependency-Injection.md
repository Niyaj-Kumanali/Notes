# Spring Dependency Injection

---

## 1. Executive Summary

### What Is It?
Dependency Injection (DI) is a design pattern where objects receive their dependencies from an external source (the IoC container) rather than creating them internally.

### Types of DI

| Type | When Injected | Pros | Cons |
|------|---------------|------|------|
| **Constructor** | Object creation | Immutable, testable, required deps | Verbose for many deps |
| **Setter** | After construction | Optional deps, mutable | Can leave object partially initialized |
| **Field** | After construction by reflection | Concise | Hard to test, hidden deps, not for production code |

### Why Use DI?
- **Decoupling** — classes depend on abstractions, not concretions
- **Testability** — inject mocks easily
- **Flexibility** — swap implementations without code changes
- **Lifecycle management** — container handles creation and destruction

---

## 2. Core Theory

### Constructor Injection (Recommended for Required Dependencies)

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final NotificationService notificationService;
    
    // Explicit constructor — Spring auto-wires by default since Spring 4.3
    public OrderService(OrderRepository orderRepository,
                        PaymentGateway paymentGateway,
                        NotificationService notificationService) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
        this.notificationService = notificationService;
    }
}
```

### Setter Injection (For Optional Dependencies)

```java
@Service
public class EmailService {
    private MailSender mailSender;
    private TemplateEngine templateEngine;
    
    @Autowired(required = false) // Optional dependency
    public void setMailSender(MailSender mailSender) {
        this.mailSender = mailSender;
    }
}
```

### Field Injection (Avoid in production code)

```java
@Service
public class ReportService {
    @Autowired
    private ReportRepository repository; // Hidden dependency
    
    @Autowired
    private PdfGenerator pdfGenerator; // Hidden dependency
}
```

**Why avoid field injection:**
- Can't create the object without reflection (harder to test)
- Dependencies are hidden (not visible in constructor/setters)
- No way to mark as final (can be mutated after construction)
- Requires Spring container (can't be used outside Spring context)

### Qualifier & Primary

```java
// Multiple implementations of same interface
public interface PaymentGateway { ... }

@Component @Qualifier("stripe")
public class StripeGateway implements PaymentGateway { ... }

@Component @Qualifier("paypal")
public class PayPalGateway implements PaymentGateway { ... }

// Usage
@Service
public class CheckoutService {
    private final PaymentGateway gateway;
    
    public CheckoutService(@Qualifier("stripe") PaymentGateway gateway) {
        this.gateway = gateway;
    }
}

// OR @Primary for default
@Component @Primary
public class DefaultGateway implements PaymentGateway { ... }
```

### @Value Injection

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

### Injection by Name (deprecated pattern)

```java
// Deprecated — use @Qualifier instead
@Service
public class MyService {
    @Resource(name = "specificBean")
    private SomeDependency dependency;
}
```

---

## 3. Under-the-Hood

### Autowiring Resolution
```
1. Field/method/constructor has @Autowired
2. Container finds matching bean(s) by type
3. If exactly one → inject it
4. If more than one → look for @Primary → @Qualifier → @Resource(name)
5. If none and required=true → throw NoSuchBeanDefinitionException
6. If none and required=false → leave null (careful with NPE!)
```

### Circular Dependencies

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

**Spring resolves with:**
- Constructor injection: throws `BeanCurrentlyInCreationException` (cannot resolve cycle)
- Setter/field injection: creates a proxy, injects it, resolves later
- **Fix:** extract shared interface, use events, use @Lazy on one side, or restructure

---

## 4. Production Code

### 4.1 Factory Pattern with DI

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

### 4.2 Conditional Injection

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

### 4.3 Bad vs Good — Constructor Overload

```java
// BAD: Field injection — hidden deps, not testable without Spring
@Service
public class BadOrderService {
    @Autowired private OrderRepository repo;
    @Autowired private PaymentGateway gateway;
    @Autowired private EmailService email;
    
    public void process(Order order) {
        // ...
    }
}

// GOOD: Constructor injection — explicit, final, testable
@Service
public class GoodOrderService {
    private final OrderRepository repo;
    private final PaymentGateway gateway;
    private final EmailService email;
    
    public GoodOrderService(OrderRepository repo, PaymentGateway gateway, EmailService email) {
        this.repo = repo;
        this.gateway = gateway;
        this.email = email;
    }
}
```

---

## 5. Cheat Sheet

```
═══ DEPENDENCY INJECTION ═══════════════════════════════════

┌─ INJECTION TYPES ──────────────────────────────────────────┐
│ Constructor  ★ Best   — required, final, testable           │
│ Setter       ◆ OK     — optional deps                      │
│ Field        ❌ Avoid  — hidden deps, untestable            │
└─────────────────────────────────────────────────────────────┘

┌─ AUTO-CONFIGURATION ───────────────────────────────────────┐
│ @ConditionalOnClass     — has class on classpath            │
│ @ConditionalOnMissingBean — only if not already defined     │
│ @ConditionalOnProperty  — property has value                │
│ @ConditionalOnExpression — SpEL expression true             │
│ @Profile                — specific environment              │
└─────────────────────────────────────────────────────────────┘

┌─ RULES ────────────────────────────────────────────────────┐
│ • Prefer constructor injection for required deps            │
│ • Use setter injection only for optional deps               │
│ • Never use field injection in production code              │
│ • @Primary for default impl, @Qualifier for specific        │
│ • Avoid circular dependencies (use @Lazy or restructure)   │
│ • Keep constructor parameter list manageable (< 7 params)   │
│ • Extract many deps → they are a smell (too many resp.)    │
└─────────────────────────────────────────────────────────────┘
```
