# Spring Bean Lifecycle

---

## 1. Executive Summary

### What Is It?
The Spring Bean Lifecycle defines the sequence of steps a bean goes through from instantiation to destruction. Understanding it is critical for:
- Proper initialization logic
- Resource cleanup
- AOP proxy behavior
- Troubleshooting startup issues

### Lifecycle Phases

```
Bean Definition Loaded → Factory Post Processors → Instantiation
    → Populate Properties → Aware Interfaces → Before Init
    → @PostConstruct / init-method → After Init → Ready for Use
    → @PreDestroy / destroy-method → Bean Destroyed
```

---

## 2. Core Theory

### Detailed Step-by-Step

1. **Bean Definition** — Spring reads config/annotations, creates BeanDefinition metadata
2. **BeanFactoryPostProcessors** — modify bean definitions before instantiation (e.g., PropertySourcesPlaceholderConfigurer)
3. **Instantiation** — bean constructor called (using constructor args)
4. **Populate Properties** — setter injection, field injection (via reflection)
5. **Aware Interfaces** — set bean name, classloader, application context

   ```java
   public class MyBean implements BeanNameAware, ApplicationContextAware {
       private String beanName;
       private ApplicationContext context;
       
       @Override
       public void setBeanName(String name) { this.beanName = name; }
       
       @Override
       public void setApplicationContext(ApplicationContext ctx) {
           this.context = ctx;
       }
   }
   ```

6. **BeanPostProcessor#postProcessBeforeInitialization** — applies to ALL beans (not per-bean)
7. **@PostConstruct** (javax.annotation) — initialization callback
8. **InitializingBean#afterPropertiesSet()** — Spring interface alternative
9. **@Bean(initMethod = "init")** — XML/Java config init method
10. **BeanPostProcessor#postProcessAfterInitialization** — after init (AOP proxies created here!)
11. **Bean is ready for use**
12. **@PreDestroy** — cleanup callback
13. **DisposableBean#destroy()** — Spring interface
14. **@Bean(destroyMethod = "cleanup")** — configured destroy method

---

## 3. Under-the-Hood

### AOP Proxy Creation Timing
AOP proxies are created in `BeanPostProcessor#postProcessAfterInitialization`. This means:
- `@PostConstruct` runs on the **raw bean** (before proxying)
- If `@PostConstruct` calls a method with `@Transactional`, that method is NOT transactional (raw bean doesn't have the proxy)

```java
@Service
public class MyService {
    @PostConstruct
    public void init() {
        // Runs on raw bean — calling self.doSomething() won't trigger AOP
    }
    
    @Transactional
    public void doSomething() { /* ... */ }
}
```

**Fix:** Use `@Transactional` outside init, or use `TransactionTemplate` in init.

### Proxy Types

| Proxy | Created By | Classes | Interfaces | Performance |
|-------|-----------|---------|------------|-------------|
| JDK Dynamic | AOP | Must implement interface | Yes | Fast |
| CGLIB | AOP | Any class | N/A | Fast (slightly slower) |

CGLIB creates a subclass. If the bean is `final`, CGLIB can't proxy it.

---

## 4. Production Code

### 4.1 Proper Initialization

```java
@Component
public class CacheInitializer {
    private final CacheManager cacheManager;
    
    public CacheInitializer(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }
    
    @PostConstruct
    public void warmCache() {
        // Initialize cache after bean is fully wired
        cacheManager.createCache("products");
        List<Product> products = loadHotProducts();
        products.forEach(p -> cacheManager.put("products", p.getId(), p));
    }
}
```

### 4.2 Proper Cleanup

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

### 4.3 Lazy Initialization

```java
@Component
@Lazy // Bean is instantiated only when first requested
public class ExpensiveResource {
    public ExpensiveResource() {
        // Heavy initialization
    }
}
```

---

## 5. Common Mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Calling @Transactional from @PostConstruct | Transaction doesn't work | Use TransactionTemplate |
| 2 | Trying to use uninitialized dependency in init | NPE | Use @PostConstruct (not constructor) |
| 3 | Throwing exception in init | Context refresh fails | Handle gracefully, log |
| 4 | Not cleaning up resources | Resource leak | @PreDestroy for cleanup |
| 5 | Circular init dependencies | BeanCurrentlyInCreationException | @Lazy or restructure |
| 6 | Heavy init in all beans | Slow startup | @Lazy for expensive beans |
| 7 | final class with AOP | Can't create CGLIB proxy | Remove final or use interface |

---

## 6. Cheat Sheet

```
═══ BEAN LIFECYCLE ═══════════════════════════════════════════

┌─ INIT CALLBACKS ───────────────────────────────────────────┐
│ @PostConstruct            (javax.annotation / jakarta)      │
│ InitializingBean          (Spring — afterPropertiesSet())   │
│ @Bean(initMethod="...")   (Java config init method)         │
│ Order of execution:                                         │
│   1. @PostConstruct → 2. afterPropertiesSet → 3. init      │
└─────────────────────────────────────────────────────────────┘

┌─ DESTROY CALLBACKS ────────────────────────────────────────┐
│ @PreDestroy              (javax.annotation / jakarta)       │
│ DisposableBean           (Spring — destroy())              │
│ @Bean(destroyMethod="...") (Java config)                   │
└─────────────────────────────────────────────────────────────┘

┌─ AWARE INTERFACES (Frequently Used) ──────────────────────┐
│ BeanNameAware        — get bean's name                     │
│ ApplicationContextAware — get ApplicationContext           │
│ EnvironmentAware     — get Environment/properties          │
│ ResourceLoaderAware  — get ResourceLoader                  │
│ MessageSourceAware   — get MessageSource (i18n)           │
└─────────────────────────────────────────────────────────────┘

┌─ POST PROCESSORS ──────────────────────────────────────────┐
│ BeanFactoryPostProcessor — modify bean definitions         │
│ BeanPostProcessor       — hooks before/after init (ALL)   │
│   postProcessBeforeInitialization — before init callbacks  │
│   postProcessAfterInitialization  — after init (AOP here) │
└─────────────────────────────────────────────────────────────┘
```
