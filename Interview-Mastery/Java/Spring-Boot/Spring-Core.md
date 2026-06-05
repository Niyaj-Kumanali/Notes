# Spring Core

---

## 1. Executive Summary

### What Is It?
Spring Core is the foundation of the Spring Framework. It provides an **Inversion of Control (IoC)** container that manages object lifecycles and dependency injection. The core container is defined by the `org.springframework.core`, `beans`, and `context` packages.

### Why Does It Exist?
Before Spring, Java EE used heavyweight EJBs with complex JNDI lookups, verbose XML, and tight coupling. Spring Core introduced:
- **Lightweight containers** — POJO-based, no special server required
- **Declarative DI** — configure wiring, not program it
- **AOP** — separate cross-cutting concerns
- **Testability** — POJOs are easy to mock and test

### Core Modules

| Module | Purpose |
|--------|---------|
| `spring-core` | IoC container, resource loading, type conversion |
| `spring-beans` | Bean factory, bean definitions, BeanWrapper |
| `spring-context` | ApplicationContext, event publishing, internationalization |
| `spring-expression` | SpEL (Spring Expression Language) |
| `spring-aop` | AOP framework (method interception) |

### IoC Container Types

| Container | Description |
|-----------|-------------|
| `BeanFactory` | Basic container (lazy initialization) |
| `ApplicationContext` | Full container (eager init, events, i18n, AOP) — use in most cases |
| `ConfigurableApplicationContext` | Extensible, refreshable, closeable |
| `WebApplicationContext` | Web-aware, scopes: request, session, application |

---

## 2. Core Theory

### Inversion of Control (IoC)
Traditional: your code creates objects. IoC: the container creates objects and **injects** them into your code.

```
Traditional:   Service → new Database() → new Logger()
IoC (Spring):  Service ← inject Database, Logger ← container manages
```

### ApplicationContext Initialization

```
1. Load configuration (XML, annotations, Java config)
2. Scan for beans / read bean definitions
3. Inter-bean dependency resolution
4. BeanPostProcessor registration
5. Bean instantiation and wiring
6. Initialize singletons (eager)
7. Publish ContextRefreshedEvent
```

### Spring Bean vs Plain Java Object

| Aspect | POJO | Spring Bean |
|--------|------|-------------|
| Instantiation | `new MyClass()` | Spring container |
| Lifecycle | JVM GC | Spring manages (init/destroy callbacks) |
| Scopes | N/A | singleton, prototype, request, session |
| Wiring | Manual DI | @Autowired, XML, Java config |
| AOP | N/A | Can be proxied for advice |

---

## 3. Under-the-Hood

### BeanFactory vs ApplicationContext

| Feature | BeanFactory | ApplicationContext |
|---------|-------------|-------------------|
| Bean instantiation | Lazy by default | Eager (singletons at startup) |
| Events | No | Yes — ApplicationEvent |
| i18n (MessageSource) | No | Yes |
| Resource loading | Basic | ResourceLoader (classpath:file:url:) |
| AOP integration | Manual | Built-in |
| Annotation scanning | No | Yes — @ComponentScan |

### Bean Processing Pipeline

```
Bean definition loaded
    ↓
BeanFactoryPostProcessor (modify definitions)
    ↓
Instantiation (constructor or factory method)
    ↓
Populate properties (setters/@Autowired)
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

---

## 4. Production Code

### 4.1 Configuration Example

```java
@Configuration
@ComponentScan(basePackages = "com.company.service")
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

### 4.2 Bean Scopes in Production

```java
@Service
@Scope("singleton") // Default — one instance per container
public class CacheService { /* ... */ }

@Component
@Scope("prototype") // New instance per injection point
public class TaskProcessor { /* ... */ }

@Controller
@Scope("request") // One instance per HTTP request
public class RequestContext { /* ... */ }

@Controller
@Scope("session") // One instance per HTTP session
public class UserPreferences { /* ... */ }

@Controller
@Scope("application") // One instance per ServletContext
public class AppWideCounter { /* ... */ }
```

---

## 5. Cheat Sheet

```
═══ SPRING CORE ═══════════════════════════════════════════════

┌─ CONTAINER ────────────────────────────────────────────────┐
│ BeanFactory — basic IoC                                    │
│ ApplicationContext — full container (use this)              │
│ AnnotationConfigApplicationContext — Java config            │
│ ClassPathXmlApplicationContext — XML config                 │
└─────────────────────────────────────────────────────────────┘

┌─ SCOPES ───────────────────────────────────────────────────┐
│ singleton  — one per container (default)                    │
│ prototype  — new per injection                             │
│ request    — one per HTTP request                          │
│ session    — one per HTTP session                          │
│ application — one per ServletContext                        │
│ websocket  — one per WebSocket session                     │
└─────────────────────────────────────────────────────────────┘

┌─ LIFECYCLE ────────────────────────────────────────────────┐
│ @PostConstruct → InitializingBean → init-method             │
│ @PreDestroy → DisposableBean → destroy-method              │
│ BeanPostProcessor → hooks before/after init                │
└─────────────────────────────────────────────────────────────┘

┌─ KEY ANNOTATIONS ──────────────────────────────────────────┐
│ @Configuration — Java config class                          │
│ @Bean — method produces a bean                              │
│ @Component/@Service/@Repository/@Controller — stereotype    │
│ @Autowired — inject dependency                              │
│ @Value — inject property value                              │
│ @Scope — define bean scope                                  │
│ @Lazy — defer initialization                                │
│ @Primary — preferred bean when multiple candidates          │
│ @Qualifier — specific bean by name                          │
└─────────────────────────────────────────────────────────────┘
```
