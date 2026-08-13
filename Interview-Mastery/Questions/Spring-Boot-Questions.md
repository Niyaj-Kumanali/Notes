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
   - **Answer:** Spring Framework is a lightweight, modular, open-source framework for building enterprise-grade Java applications. Its core value proposition is Inversion of Control (IoC): instead of application code creating and wiring its own objects, control of object creation and lifecycle is handed to the Spring IoC container. At the center of this is dependency injection (DI), where the container resolves and supplies the dependencies a class declares, rather than the class constructing them with `new`. Spring is intentionally modular — it is composed of roughly 20 modules grouped into Core Container, Data Access/Integration, Web, AOP/Aspects, Messaging, and Test. This lets an application pull in only what it needs; a batch job might use Core plus JDBC, while a web app adds Spring MVC. The framework also standardizes cross-cutting concerns: transaction management via `@Transactional` (with a consistent abstraction over JDBC, JPA, Hibernate), data access through `JdbcTemplate` and `JdbcRepository`, aspect-oriented programming through Spring AOP, and declarative validation and messaging. Because the container is the central registry, code stays decoupled — a service depends on an interface, and the container decides which implementation to inject, which makes switching implementations and writing unit tests straightforward. Spring does not require a servlet container or application server to exist up front; with XML or Java configuration it can bootstrap in any JVM. Over time Spring has evolved from XML-based configuration to annotation-based and now auto-configuration-based setup, but the underlying principles — a managed bean container, declarative configuration, and layered modularity — remain unchanged. Understanding Spring first is essential because Spring Boot, Spring Data, Spring Security, and Spring Cloud all build on the same container and configuration model.

2. What is Spring Boot?
   - **Answer:** Spring Boot is a convention-over-configuration framework built on top of the Spring Framework that makes stand-alone, production-grade Spring applications easy to create. It removes the heavy manual wiring historically required with plain Spring by providing three pillars: starter dependencies, auto-configuration, and an embedded server. A starter, such as `spring-boot-starter-web`, is a curated Maven dependency that bundles everything needed for a capability — for the web starter that means Spring MVC, Jackson, embedded Tomcat, and validation support. Auto-configuration inspects the classpath at startup and conditionally creates beans only when the relevant libraries and missing custom beans are present; adding `spring-boot-starter-data-jpa` automatically configures a DataSource, `EntityManagerFactory`, and `PlatformTransactionManager` with sensible defaults. The embedded server means a single runnable JAR contains Tomcat (or Jetty/Undertow) inside it, so a service can be started with `java -jar app.jar` with no external web server installation or deployment descriptor. Boot also provides externalized configuration through `application.properties`/`application.yml`, environment variables, and command-line arguments with a well-defined precedence order. Production concerns are addressed out of the box: Spring Boot Actuator exposes health, metrics, and environment endpoints, and DevTools offers rapid restart and live reload for development. Every Spring Boot application is still a Spring application — the container, DI, AOP, and MVC behavior are identical — Boot simply wires them up automatically based on sensible defaults that can be overridden.

3. Why use Spring Boot?
   - **Answer:** Spring Boot exists to minimize configuration effort and time-to-first-request while keeping the full power of the Spring ecosystem. The main reasons to choose it are rapid development, easier dependency management, embedded deployment, and production readiness. Instead of writing hundreds of lines of XML or Java config for a DataSource, an entity manager, a view resolver, and a dispatcher servlet, Boot auto-configures them from the starter on the classpath. Starters also solve dependency version alignment: Boot's dependency management BOM pins compatible versions of Spring, Jackson, Hibernate, and the servlet API, eliminating "works on machine A, fails on machine B" version drift. Because the application is packaged as a fat/executable JAR with an embedded server, deployment is reduced to copying a file and running it — the same artifact runs on a laptop, a test server, or a container image, which makes CI/CD and Docker pipelines simpler. The opinionated defaults are a sensible baseline, yet everything is overridable through properties or explicit bean definitions, so a team never hits a wall where Boot's choice cannot be replaced. Boot's ecosystem is also huge: Actuator gives operational endpoints, Spring Data reduces data-access boilerplate, and Spring Cloud builds distributed-systems features on the same foundation. The main trade-off is magic — when auto-configuration behaves unexpectedly, developers must understand conditional annotations and the auto-configuration report to debug it. Overall Boot is the default choice for new Spring services because it lets teams focus on business logic rather than infrastructure plumbing.

4. Difference between Spring and Spring Boot.
   - **Answer:** Spring Framework is the foundational platform providing DI, AOP, transaction abstraction, and Spring MVC; Spring Boot is an additional layer that sits on top and makes Spring dramatically easier to configure and run. Plain Spring requires the developer to assemble the container manually — declaring a `ContextLoaderListener`, an XML or `@Configuration`-based `ApplicationContext`, component scan packages, property placeholders, and bean definitions — and to package the application into a WAR for an external servlet container. Spring Boot changes all of that: it embeds a web server, applies auto-configuration via `@EnableAutoConfiguration`, and pulls in dependencies through curated starters. Under the hood Boot is still pure Spring — the same `ApplicationContext`, the same `DispatcherServlet`, the same bean lifecycle — it just programmatically assembles defaults based on classpath inspection and `@Conditional` checks. Plain Spring gives total, explicit control and is appropriate when fine-grained, hand-tuned wiring is required or when building on top of a pre-existing container; Boot is preferred for new services where speed, convention, and production features matter more than explicit control. Boot also adds operational concerns plain Spring leaves to the team: Actuator, config property binding, profile-aware configuration, and graceful-shutdown hooks. Another distinction is versioning and ecosystem: Spring Boot aligns and curates the dependency tree via its BOM, whereas with plain Spring the team must manage version compatibility themselves. In practice, virtually all modern Spring applications are Spring Boot applications, so the difference is best framed as "Boot is Spring with opinionated, auto-configured, embeddable defaults."

5. What is auto-configuration?
   - **Answer:** Auto-configuration is Spring Boot's mechanism for creating beans automatically based on the dependencies present on the classpath and the properties in the environment. When the application starts, Boot looks for classes listed in `META-INF/spring.factories` (or `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`) under the key `EnableAutoConfiguration`; each such class is a candidate auto-configuration. Each auto-configuration class is guarded by `@Conditional` annotations so it only takes effect under specific circumstances. For example, `DataSourceAutoConfiguration` activates only when a `DataSource` implementation is on the classpath, `HttpMessageConvertersAutoConfiguration` when Jackson is present, and `WebMvcAutoConfiguration` when Spring MVC is present and the application is a web application. Two conditionals dominate: `@ConditionalOnClass` (a library is present) and `@ConditionalOnMissingBean` (the developer has not already defined the bean, in which case the auto-configured one backs off). This back-off behavior is what makes customization possible — define a custom `RestTemplateBuilder` or `CacheManager` bean and the auto-configured default quietly disappears. Auto-configuration also respects property values: `@ConditionalOnProperty(name = "spring.cache.type", havingValue = "redis")` activates Redis caching only when configured. Because beans are created before application code runs, auto-configuration classes are ordered (`@AutoConfigureBefore`, `@AutoConfigureAfter`) to resolve dependencies between them. The result is that adding a starter and a few properties is enough to wire up web, persistence, security, and messaging infrastructure, while any bean explicitly defined by the application always wins over the automatic default.

6. How does Spring Boot auto-configuration work?
   - **Answer:** At startup, `SpringApplication.run()` constructs an `ApplicationContext` whose bean definition reader includes an `AutoConfigurationImportSelector`. This selector reads the candidate auto-configuration class names from `META-INF/spring.factories` (under `EnableAutoConfiguration`) or, in newer versions, from the `AutoConfiguration.imports` file, and filters them through two passes of `@Conditional` evaluation. The first pass evaluates conditions that do not depend on the bean factory (classpath checks, property checks, resource checks); the second pass evaluates conditions that do (such as `@ConditionalOnBean` and `@ConditionalOnMissingBean`) once other beans are registered. Only classes whose conditions all pass become part of the configuration. `@ConditionalOnClass` activates only when certain classes are on the classpath; `@ConditionalOnMissingBean` backs off when the application already declared the bean; `@ConditionalOnProperty` triggers on configuration values. Each auto-configuration is itself a `@Configuration` class, and ordering between them is declared with `@AutoConfigureBefore`, `@AutoConfigureAfter`, and `@AutoConfigureOrder` so that, for example, `DataSourceAutoConfiguration` runs before `JpaRepositoriesAutoConfiguration`. The outcome of all condition evaluations is exposed at runtime: passing `--debug` (or `debug=true` in properties) prints an auto-configuration report showing which configurations were matched, which were not matched and why, and which beans were excluded. Actuator's `/conditions` endpoint exposes the same report over HTTP. Understanding this flow matters because the "magic" of Boot is really just deterministic conditional bean registration — when an auto-configured bean conflicts with expected behavior, the report and the condition reasons are the first place to look, and a custom bean or `spring.autoconfigure.exclude` property resolves most issues.

7. What is starter dependency?
   - **Answer:** A starter dependency is a Maven (or Gradle) artifact that bundles a coherent set of libraries for a specific capability under the naming convention `spring-boot-starter-*`. Instead of the developer listing and versioning each library — say, Spring MVC, Jackson databind, Jackson datatype JSR310, embedded Tomcat, and `spring-web` — adding one starter brings all of them transitively with compatible versions. The Spring Boot parent POM (`spring-boot-starter-parent`) uses dependency management to pin those versions, so the starter typically needs no `<version>` element. Common examples are `spring-boot-starter-web` (Spring MVC + embedded Tomcat + Jackson + validation), `spring-boot-starter-data-jpa` (Hibernate + Spring Data JPA + HikariCP), `spring-boot-starter-security`, `spring-boot-starter-actuator`, and `spring-boot-starter-validation`. Each starter is paired with an auto-configuration class that only activates when the starter's libraries are present, which is why merely adding the dependency triggers the corresponding configuration. Starters also support customization — `spring-boot-starter-tomcat` can be swapped for `spring-boot-starter-jetty` or `spring-boot-starter-undertow` by excluding the default and adding the replacement, while keeping `starter-web` for the rest. A team can also build a custom starter by writing an auto-configuration class, registering it in `META-INF/spring.factories`, and publishing it; the custom starter then brings in both dependencies and configuration in one artifact. The practical benefit is a dramatic reduction in build-file complexity and a far lower risk of version conflicts between Spring ecosystem libraries.

8. What is embedded server?
   - **Answer:** An embedded server is a servlet container — Tomcat by default, or Jetty/Undertow — packaged inside the application JAR so the application is a self-contained executable. `java -jar app.jar` starts the JVM, bootstraps the Spring context, and launches the embedded Tomcat on the configured port, typically 8080. This eliminates the traditional two-artifact deployment model (a WAR plus an external application server) and everything that comes with it: installing and patching Tomcat, defining `server.xml`, deploying exploded archives, and matching server and app versions. Configuration happens through properties rather than server XML: `server.port` changes the port, `server.servlet.context-path` changes the base path, and `server.ssl.*` enables HTTPS with a bundled keystore. The embedded server participates in the same lifecycle as the application context — it is started during context refresh and shut down gracefully via the shutdown hook, which enables Boot's graceful shutdown features for in-flight requests. Because the container is a library dependency, it is easy to swap: excluding `spring-boot-starter-tomcat` and adding `spring-boot-starter-jetty` or `spring-boot-starter-undertow` changes the runtime without code changes. This embeddability is what makes Spring Boot services natural for containers and Kubernetes — the image is simply the JAR plus a JRE, with no supervisor process needed for the servlet engine. The trade-off is that embedded servers are tuned for a single application, so heavyweight enterprise features like shared JNDI resources across applications are deliberately out of scope; when such features are needed, the application can still be packaged as a WAR and deployed to an external container.

9. What is IoC?
   - **Answer:** Inversion of Control (IoC) is a design principle where the flow of a program and the creation of its objects are handed over from the application code to a framework or container, inverting the traditional dependency chain. In conventional code, class A creates its own dependencies with `new B()`, which tightly couples A to B's concrete type and lifecycle. With IoC, a class only declares what it needs (typically via a constructor parameter or field), and the container decides when to create those dependencies and injects them. In Spring, the IoC container is the `ApplicationContext` (the older, lighter `BeanFactory` also exists); it instantiates beans, wires their dependencies, manages their scopes, and applies lifecycle callbacks. This inversion has several concrete payoffs: code depends on abstractions rather than implementations, so implementations can be swapped through configuration alone; objects are easier to unit-test because dependencies can be mocked and injected directly; and lifecycle concerns such as lazy initialization, proxies, and cleanup are centralized rather than scattered. IoC is a broad principle that predates Spring — it also appears in servlet containers (a servlet does not run its own `main`), JUnit (the framework invokes test methods), and GUI frameworks — so "Spring does IoC" is really "Spring provides a container that implements the principle." The container's control extends to more than creation: it also manages AOP proxies, transaction boundaries, and event publishing, all transparently to the beans it manages.

10. What is dependency injection?
    - **Answer:** Dependency Injection (DI) is the concrete mechanism through which the IoC container fulfills the principle of Inversion of Control: the container supplies an object's dependencies rather than the object creating them. A class declares its needs — in Spring typically through a constructor parameter — and the container resolves each declared type against its bean registry and passes the instance in. For example, a `PaymentService` that needs an `OrderRepository` simply declares it in its constructor and the container provides a bean that matches the `OrderRepository` type (and possibly the `@Qualifier` name). DI comes in three flavors in Spring: constructor injection (dependencies passed via the constructor, recommended), setter injection (a setter method invoked after construction), and field injection (the container assigns a private field via reflection, generally discouraged). The container resolves dependencies at startup, so a missing or ambiguous bean fails fast with a clear `NoSuchBeanDefinitionException` or `NoUniqueBeanDefinitionException` rather than failing randomly at runtime. DI decouples construction from use: the same class can be instantiated in a unit test with mocks and in production with real collaborators, with zero changes to the class itself. Because dependencies are explicit and visible, DI also makes design problems obvious — a class with seven constructor parameters is a clear signal to refactor. The two variants of DI Spring uses are `@Autowired` on constructors/fields/setters and `@Bean`/`@Component` bean definitions; the container then manages the entire object graph, including lifecycle and destruction.

11. Types of dependency injection.
    - **Answer:** Spring supports three types of dependency injection: constructor injection, setter injection, and field injection. Constructor injection passes dependencies as parameters of the class constructor; Spring calls the constructor with the resolved beans, and since Java 4.3 a single constructor needs no `@Autowired` at all. This form is strongly recommended by the Spring team because dependencies are final and immutable, the object is fully initialized and valid immediately after construction, testing requires only passing the collaborators to the constructor, and a missing dependency fails fast at startup. Setter injection calls a setter method after construction with the resolved bean; it is useful for optional dependencies and when a component must be constructed with a no-arg constructor (for example with certain frameworks), but it leaves the object in a partially initialized state until the setters run and allows the state to change later. Field injection annotates a private field with `@Autowired` and lets the container assign it via reflection; it is the most concise but also the most problematic — the object is created with a no-arg constructor, dependencies are invisible to the outside, tests must use reflection or the container to wire fields, and there is no way to make a field final. A clear example of constructor injection:

    ```java
    @Service
    public class PaymentService {
        private final OrderRepository orderRepository;

        public PaymentService(OrderRepository orderRepository) {
            this.orderRepository = orderRepository;
        }
    }
    ```

    All three ultimately instruct the container to resolve and inject beans, but constructor injection is the production default because it enforces immutability, testability, and fail-fast validation. Mixed use is possible (constructor for mandatory, setter for optional), but field injection is best avoided in application code.

12. Constructor injection vs field injection.
    - **Answer:** Constructor injection supplies dependencies through the class constructor and is the recommended approach; field injection assigns dependencies directly to private fields via reflection and is widely discouraged. Constructor injection makes every dependency a final field, so the object is immutable and always in a consistent, fully-wired state — there is no window where a partially constructed instance exists. It is also the only form that works naturally outside the container: a unit test constructs the object with real or mocked collaborators without reflection or a test context. Missing or ambiguous dependencies surface as an immediate startup failure, making wiring mistakes obvious. Field injection, by contrast, hides dependencies inside private fields, so the class's requirements are not visible to callers, it cannot be constructed cleanly in a unit test, and it creates a risk of `NullPointerException` in tests that do not initialize the fields. It also prevents making dependencies final and can mask circular-dependency problems that constructor injection would catch immediately. Spring's documentation and team have consistently stated a preference for constructor injection, which is why the single-constructor case needs no `@Autowired` annotation at all. Practical convention is to use constructor injection for mandatory dependencies and reserve setter injection for truly optional ones; Lombok's `@RequiredArgsConstructor` can generate the constructor from final fields, keeping the code concise while preserving the benefits.

13. What is a Spring bean?
    - **Answer:** A Spring bean is any object that the Spring IoC container instantiates, assembles, and otherwise manages as part of its `ApplicationContext`. Beans are the building blocks of a Spring application: services, repositories, controllers, templates, connection factories, and configuration objects all become beans. A bean is defined either by a stereotype annotation such as `@Component`, `@Service`, `@Repository`, or `@Controller` (discovered by component scanning), or by a `@Bean` method inside a `@Configuration` class. Each bean has an identity in the container: a name (derived from the class or explicitly set), a type, a scope (singleton by default), and a set of dependencies that the container wires in. The container controls the bean's full lifecycle — instantiation, dependency population, initialization callbacks, and destruction — and hands out the same instance (for singleton scope) wherever the bean is injected. Beans are looked up by type and optionally qualified by name, so an interface with multiple implementations requires a `@Qualifier` or `@Primary` to disambiguate. Being a bean also means the object can participate in container features: AOP proxies (transaction management, `@Async`, caching), event publishing, and environment-based activation via `@Profile`. The key mental model is that application code never constructs beans with `new`; instead the container does, which is why the term "bean" refers to the managed instance and its metadata together.

14. What is bean scope?
    - **Answer:** Bean scope defines the lifecycle and visibility of a bean instance — how many instances the container creates and how long each lives. The core scopes are singleton (default) and prototype, plus web-aware scopes: request, session, application, and websocket. In singleton scope, the container creates exactly one instance per bean name per `ApplicationContext`, caches it, and reuses it for every injection and lookup; this is ideal for stateless components like services and repositories. In prototype scope, the container creates a brand-new instance each time the bean is requested by injection, `getBean()`, or lookup, and it does not manage the prototype's destruction callbacks — the caller is responsible for cleanup. Request scope gives one instance per HTTP request (e.g., a per-request authorization context), session scope one per HTTP session (shopping-cart style state), and application scope one per `ServletContext`. Scopes are declared with `@Scope("prototype")` or `@Scope(value = WebApplicationContext.SCOPE_REQUEST, proxyMode = ScopedProxyMode.TARGET_CLASS)`; because a singleton bean cannot hold a prototype dependency directly (the prototype is resolved once at wiring time), the container injects a scoped proxy or `ObjectFactory<PrototypeBean>` to defer lookup. Choosing scope is a thread-safety decision: singletons are shared across threads and must be stateless or internally synchronized, while prototypes trade memory and allocation cost for isolation. In practice most beans should stay singleton; prototype is used sparingly for stateful, short-lived objects.

15. Difference between singleton and prototype scope.
    - **Answer:** Singleton scope makes the container create one instance of the bean for the entire `ApplicationContext` and return that same instance on every injection or lookup; prototype scope makes the container create a new instance every time the bean is requested. This difference has three practical consequences: identity, lifecycle, and thread-safety. With a singleton, all clients share the same object, so `==` comparisons between injected instances are true and any mutable field is shared across all threads — requiring statelessness or synchronization. With a prototype, every consumer gets a distinct object with its own state, at the cost of repeated allocation and garbage-collection pressure. Lifecycle handling also differs: the container fully manages the singleton, including `@PreDestroy` and `DisposableBean` callbacks on shutdown, but for prototypes the container only creates and wires the instance and does not track it afterwards, so destruction callbacks are not invoked. A subtlety is that a singleton depending on a prototype gets the prototype resolved exactly once at construction, which defeats the prototype's purpose unless a scoped proxy or `ObjectFactory<...>` (or `@Lookup`) is used to defer each resolution. Choosing between them follows the statefulness rule: stateless collaborators (services, repositories) are singletons; objects that carry per-use state should either be made local (constructed by the code) or prototype-scoped. Singleton is the sensible default in Spring, and the container also warns about singletons that hold mutable instance state without synchronization.

16. What is application context?
    - **Answer:** The `ApplicationContext` is the central Spring IoC container: it holds the bean definitions, creates and manages the bean instances, and exposes methods to look them up. It extends the `BeanFactory` interface and adds enterprise features such as internationalization (`MessageSource`), event propagation (`ApplicationEventPublisher`), resource loading (`ResourceLoader`), environment abstraction (`Environment`), and lifecycle management (`Lifecycle`). When a Spring Boot application calls `SpringApplication.run(...)`, an `AnnotationConfigApplicationContext` (or `AnnotationConfigServletWebServerApplicationContext` for web apps) is constructed behind the scenes: it scans for annotated classes, processes auto-configuration, instantiates singletons, wires dependencies, and invokes initialization callbacks before returning control. At runtime, beans are typically obtained through injection rather than direct lookup, but programmatic access is available via `ApplicationContext.getBean(SomeType.class)`, which is occasionally needed in legacy code or for dynamic lookups. The context has a parent-child hierarchy in advanced setups — a child context can see beans from the parent, enabling shared infrastructure across modules — though Spring Boot applications normally use a single flat context. Lifecycle is significant: the context must be refreshed to build the container and closed to trigger bean destruction and release resources; Boot wires this to the JVM shutdown hook so embedded servers and connection pools shut down gracefully. The context is also where conditionals, profiles, and property resolution come together, which is why "the context is the application" is a reasonable summary — nearly all of Spring's behavior is context-driven.

17. What is bean lifecycle?
    - **Answer:** The bean lifecycle is the ordered sequence of steps the container performs from instantiation to destruction of each bean. It runs roughly as follows: the container reads bean definitions, instantiates the object (typically via its constructor, and singleton beans are usually instantiated eagerly at startup unless `@Lazy`), then populates properties and performs dependency injection. After construction and wiring, awareness callbacks fire if the bean implements marker interfaces: `BeanNameAware`, `BeanClassLoaderAware`, `BeanFactoryAware`, and `ApplicationContextAware` give the bean access to container objects. Next come `BeanPostProcessor` hooks — `postProcessBeforeInitialization` runs before the bean's own initialization — then the initialization itself in the order `@PostConstruct`, `InitializingBean.afterPropertiesSet()`, and a custom `initMethod` specified in `@Bean(initMethod = "...")`. After `postProcessAfterInitialization`, the bean is fully initialized; this is also the stage where AOP proxies wrap the target, which is why `@Async` and `@Transactional` work. At shutdown, the reverse happens: `@PreDestroy`, then `DisposableBean.destroy()`, then a custom `destroyMethod`, then `BeanPostProcessor`-related destruction hooks. Bean definition-level configuration can override parts of this, and `@PostConstruct`/`@PreDestroy` are generally preferred because they are standard (JSR-250), simple, and independent of Spring interfaces. A common use of the lifecycle is loading reference data after the bean is ready — with the caveat that `@PostConstruct` should not call `@Transactional` methods directly, since the proxy and transaction manager are not yet fully in place at that point; `@EventListener(ApplicationReadyEvent.class)` is a safer alternative. Understanding the ordering matters for debugging AOP, transactions, and initialization-time failures.

18. What is `@Component`?
    - **Answer:** `@Component` is a generic stereotype annotation that marks a Java class as a Spring-managed bean, meaning the container will discover it during component scanning, instantiate it, and register it in the `ApplicationContext`. The scanning is triggered by `@ComponentScan` (implicitly included in `@SpringBootApplication`), which scans the base package and its sub-packages for classes annotated with `@Component` and its meta-annotations. The default bean name is derived from the class name in lowerCamelCase — `InventoryValidator` becomes `inventoryValidator` — or it can be overridden with `@Component("name")`. Once registered, the bean is a normal container citizen: it can have dependencies injected into its constructor, it participates in AOP proxying, and it is available for injection elsewhere. `@Component` is intentionally generic — it carries no semantic meaning about the class's role. That is why Spring provides specialized stereotypes: `@Service` for the business layer, `@Repository` for persistence, and `@Controller`/`@RestController` for web endpoints, all of which are themselves meta-annotated with `@Component` so that scanning picks them up identically. The general guidance is to prefer the specialized annotations so the architecture is visible from the annotations alone and so layer-specific behavior (like `@Repository`'s exception translation) applies; plain `@Component` is appropriate for infrastructure helpers, converters, or utility beans that do not fit a layer. `@Component` is a class-level annotation only, and it does not work for third-party classes — those must be registered with a `@Bean` method instead.

19. Difference between `@Component`, `@Service`, `@Repository`, and `@Controller`.
    - **Answer:** All four are stereotype annotations that register a class as a Spring bean, and `@Service`, `@Repository`, and `@Controller` are all meta-annotated with `@Component`, so component scanning treats them identically in terms of bean creation. The differences are semantic and behavioral. `@Component` is the generic marker with no implied role. `@Service` designates the business/service layer and is functionally a synonym for `@Component` — it adds no special processing but makes the layered architecture explicit and is the conventional home of `@Transactional` business logic. `@Repository` marks the persistence/data-access layer and does add real behavior: it enables Spring's persistence exception translation, so vendor-specific exceptions such as `SQLException` and JPA's `PersistenceException` are converted into Spring's unified `DataAccessException` hierarchy through `PersistenceExceptionTranslationPostProcessor`. `@Controller` marks MVC web controllers; Spring MVC detects it and maps `@RequestMapping`-annotated methods, and a plain `@Controller` typically returns a view name (resolved by a `ViewResolver` for Thymeleaf/JSP) unless methods are annotated `@ResponseBody`. `@RestController` combines `@Controller` + `@ResponseBody`, serializing each method's return value directly to JSON/XML. Choosing the right annotation is mostly about clarity and consistency — `@Service` for business logic, `@Repository` for data access, `@RestController` for JSON APIs — and about activating the specific behaviors (`@Repository` translation, controller mapping) that the layer requires. Using `@Component` everywhere works mechanically but loses the architectural signal and the persistence exception translation that `@Repository` provides.

20. What is `@SpringBootApplication`?
    - **Answer:** `@SpringBootApplication` is a convenience annotation placed on the main class that combines three others: `@Configuration` (the class is a source of bean definitions), `@EnableAutoConfiguration` (turn on Spring Boot's auto-configuration machinery), and `@ComponentScan` (scan the package and sub-packages of the main class for components). It is equivalent to declaring all three explicitly, and it centralizes the bootstrap contract of a Boot application. Because `@ComponentScan` is tied to the package of the annotated class, the main class should sit at the root of the package tree; classes outside that package are not scanned unless `basePackages` or `scanBasePackages` is configured. The annotation also exposes attributes that map through to its meta-annotations: `exclude` and `excludeName` forward to `@EnableAutoConfiguration` (to disable specific auto-configurations such as `DataSourceAutoConfiguration` when no database is intended), and `scanBasePackageClasses`, `scanBasePackages`, `nameGenerator`, and others forward to `@ComponentScan`. When `SpringApplication.run(MainClass.class, args)` executes, it uses this annotation to identify the configuration source, build the `ApplicationContext`, run auto-configuration, and start the embedded web server. In web applications the main class typically extends `SpringBootServletInitializer` only when packaging as a WAR for an external container. Since Spring Boot 2, the annotation also implicitly configures lazy proxy resolution and other defaults, making it the single entry point that ties scanning, auto-configuration, and bean-definition sources together.

21. What does `@EnableAutoConfiguration` do?
    - **Answer:** `@EnableAutoConfiguration` activates Spring Boot's automatic bean configuration. It is an importing annotation: it uses `AutoConfigurationImportSelector` to read the list of candidate auto-configuration classes from `META-INF/spring.factories` (under the `EnableAutoConfiguration` key) or, in newer versions, from `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Each candidate is then evaluated against `@Conditional` annotations, and only those whose conditions pass contribute beans to the context. This is why adding `spring-boot-starter-web` results in a configured `DispatcherServlet`, `ObjectMapper`, and error-handling infrastructure without any explicit configuration: the web auto-configurations match the conditions (Spring MVC and Tomcat on the classpath, no user-defined equivalents). Auto-configuration is designed to back off when the developer defines the relevant bean — `@ConditionalOnMissingBean` — so explicit definitions always take precedence, and it is ordered with `@AutoConfigureBefore`/`@AutoConfigureAfter` so interdependent configurations (DataSource before JPA, for example) apply in a deterministic sequence. The annotation is normally included transitively inside `@SpringBootApplication`, but it can be used on its own or tuned via `exclude`/`excludeName` attributes or the `spring.autoconfigure.exclude` property when a specific auto-configuration causes problems. Because it is the heart of Boot's convention-over-configuration, it is also the primary source of "magic" — when auto-configured behavior is surprising, the auto-configuration report (`--debug` or Actuator `/conditions`) explains exactly which configurations matched, which did not, and why.

22. What is `@Configuration`?
    - **Answer:** `@Configuration` marks a class as a source of bean definitions: its `@Bean`-annotated methods define beans, and the container uses the returned objects as instances registered in the `ApplicationContext`. It is the modern replacement for Spring XML configuration. A defining detail is that `@Configuration` classes are themselves proxied via CGLIB so that calls between `@Bean` methods obey singleton semantics: when one `@Bean` method invokes another inside the same class, the container intercepts the call and returns the cached singleton rather than executing the method again. This is what guarantees a single `DataSource` or `RestTemplate` instance across a configuration class. A minimal example:

    ```java
    @Configuration
    public class CacheConfig {
        @Bean
        public CacheManager cacheManager() {
            return new ConcurrentMapCacheManager("products", "prices");
        }
    }
    ```

    Because the class is proxied, `@Bean` methods can also receive method parameters that the container resolves from existing beans, enabling clean dependency wiring between configuration-produced beans. `@Configuration` differs from `@Component` in this proxying behavior: a `@Bean` method inside a `@Component` class is not intercepted, so calling it directly creates new instances. Configuration classes can use the full container feature set — `@Profile`, `@Conditional`, `@PropertySource`, and method-level `@Bean(initMethod = ..., destroyMethod = ...)`. They are the idiomatic place to define third-party beans (clients, templates, pools) that cannot be annotated with `@Component`, and to centralize wiring that component scanning alone cannot express.

23. What is `@Bean`?
    - **Answer:** `@Bean` is a method-level annotation that declares the method's return value as a bean managed by the container. It is used inside `@Configuration` classes (and, less commonly, `@Component` classes) and gives full programmatic control over instantiation — which is precisely why it is the tool for registering third-party objects whose classes cannot be annotated, such as `RestTemplate`, `PasswordEncoder`, `ObjectMapper`, `RedisTemplate`, or connection factories. The bean's name defaults to the method name and can be overridden with `@Bean(name = "...")`; the bean type is the method's return type. Lifecycle hooks can be attached declaratively: `@Bean(initMethod = "start", destroyMethod = "stop")` invokes named methods, and by default the container infers a `close`/`shutdown` destroy method on known types. Method parameters of a `@Bean` method are resolved from the container, so a `@Bean` method can accept a `DataSource` or `Properties` and build on it. A representative example:

    ```java
    @Configuration
    public class WebClientConfig {
        @Bean
        public RestTemplate restTemplate() {
            return new RestTemplateBuilder().connectTimeout(Duration.ofSeconds(5)).build();
        }
    }
    ```

    Since the class is a `@Configuration`, Spring proxies the bean method so repeated calls return the cached singleton rather than a new instance. `@Bean` is the natural companion to auto-configuration: many Boot auto-configurations declare their beans with `@Bean` methods guarded by `@ConditionalOnMissingBean`, which is exactly why defining a `@Bean` of the same type overrides the default. Choose `@Bean` when instantiation requires custom logic, arguments, or third-party types; choose `@Component` for own classes discoverable by scanning.

24. Difference between `@Bean` and `@Component`.
    - **Answer:** Both register beans in the Spring container, but they differ in where they are applied, how beans are discovered, and how much control the developer has. `@Component` is a class-level annotation applied to application's own classes; the container discovers it via component scanning, instantiates it with its constructor (with dependencies injected), and names the bean from the class. `@Bean` is a method-level annotation inside a `@Configuration` class; the bean is created by executing the method, which is explicit application code, so instantiation can involve custom logic, arguments, conditional branching, and third-party types. This is the deciding factor: classes the team owns and that can be annotated should use `@Component` (or its specializations `@Service`, `@Repository`, `@Controller`); classes the team does not control — libraries and framework types — must use `@Bean`, since they cannot be annotated for scanning. A `@Bean` method also allows multiple beans of the same class with different configurations under different names, and it can declare lifecycle callbacks via `initMethod`/`destroyMethod`. There is a behavioral subtlety in the proxy: inside a `@Configuration` class, calls between `@Bean` methods are intercepted so the singleton is returned, whereas inside a `@Component` class the same call constructs a new instance, breaking singleton semantics. Both mechanisms ultimately register beans in the same container and can be mixed freely; the choice is driven by ownership of the type and the need for explicit construction control. In summary, `@Component` is "this class is a bean," while `@Bean` is "this method creates a bean."

25. What is `application.properties`?
    - **Answer:** `application.properties` is the default configuration file Spring Boot reads at startup to externalize application settings. It is a flat `key=value` file, located on the classpath root (typically `src/main/resources`), whose keys follow a hierarchical, dot-separated naming convention such as `server.port=8080` or `spring.datasource.url=jdbc:mysql://localhost:3306/app`. These values are bound into the `Environment` and consumed in three main ways: by auto-configuration (Boot reads `spring.datasource.*`, `server.*`, `spring.jpa.*`, etc. to tune automatically created beans), by `@Value("${key}")` injection into fields, and by `@ConfigurationProperties` classes that bind whole property trees to typed POJOs. Boot searches for configuration in a defined order, and the same key set in several places is resolved by precedence: command-line arguments override profile-specific files, which override the plain `application.properties`, which overrides defaults inside the JAR. Profile-specific variants named `application-{profile}.properties` (for example `application-prod.properties`) override the base file when that profile is active. Beyond the main file, additional sources can be added via `@PropertySource`, the `spring.config.import` property, or environment variables, making deployment-time tuning possible without rebuilding the artifact. Keeping credentials and environment-specific values out of the file (using environment variables or a secrets manager) is an important security practice, since the file is packaged into the JAR. The main benefits are separation of configuration from code, easy overrides per environment, and no recompilation required to change runtime behavior.

26. Difference between `application.properties` and `application.yml`.
    - **Answer:** Both are Spring Boot configuration files with identical binding semantics — values end up in the same `Environment` and configure the same properties — but they differ in format and readability. `.properties` is a flat list of `key=value` lines, where hierarchy is encoded by repeating the prefix, e.g. `spring.datasource.url=...`, `spring.datasource.username=...`, which becomes repetitive for deeply nested settings. `.yml` (YAML) expresses the same structure with indentation, so nested groups are written once:

    ```yaml
    spring:
      datasource:
        url: jdbc:h2:mem:testdb
        username: sa
      jpa:
        hibernate:
          ddl-auto: validate
    ```

    This is typically more readable and less error-prone for large configurations, and YAML natively supports lists (useful for `spring.profiles`, cache names, or security matchers). Practical differences matter: YAML files are not parsed by the older `PropertySource` loading — `@PropertySource` does not support `.yml`, and YAML is also not a valid option in some legacy tooling — and YAML's use of indentation means formatting mistakes silently alter meaning or fail to parse. Both formats support profile-specific variants (`application-dev.properties` / `application-dev.yml`) and the same property keys, so a team can migrate between them freely. Boot also supports `application.yaml` and, when both `.properties` and `.yml` are present, `.properties` takes precedence over `.yml`. The choice is largely stylistic: `.properties` is simpler and more familiar, `.yml` scales better for nested, multi-profile configurations. In either format, every value lands in the same `Environment` abstraction, so auto-configuration and `@ConfigurationProperties` binding behave identically.

27. What are profiles?
    - **Answer:** Profiles are Spring's mechanism for environment-specific bean definitions and configuration values. A profile is simply a named set of beans and property sources that are active only when that profile is enabled, which allows a single codebase to run differently in development, testing, staging, and production. Configuration is profile-aware through files named `application-{profile}.yml` (or `.properties`), which override the base `application.yml` for matching keys; bean definitions are profile-aware through `@Profile("dev")` on `@Configuration` classes or `@Bean` methods, so infrastructure like an in-memory database or a stub client can exist only in the dev context. The active profile is selected via `spring.profiles.active`, settable as a JVM argument (`-Dspring.profiles.active=prod`), an environment variable (`SPRING_PROFILES_ACTIVE`), or a property, and multiple profiles can be activated at once (`prod,highmem`). Profiles can also be grouped (`spring.profiles.group.prod=prod-db,prod-metrics`) so a single activation pulls in a coherent set, and profile-specific documents inside one YAML file (separated by `---` and `spring.config.activate.on-profile`) can be used instead of separate files. A bean whose class carries `@Profile` is only registered when at least one of the listed profiles is active; beans with no profile are always active, so the absence of an active profile is not a failure. Profiles are a configuration-time concern — they do not change application code, only which beans and property sources participate — which is why they are the standard way to run the same artifact across environments.

28. How do you externalize configuration?
    - **Answer:** Spring Boot externalizes configuration by reading from many ordered sources and merging them into a single `Environment`, so the same key can be supplied differently per deployment without code changes. The precedence order (highest wins) is roughly: command-line arguments, `SPRING_APPLICATION_JSON`/Java system properties, OS environment variables, profile-specific files (`application-{profile}.properties`/`.yml`), the base `application.properties`/`.yml` (with `application.yaml` and `bootstrap` files in older versions), and finally defaults hard-coded in code. OS environment variables are matched case-insensitively with relaxed binding — `SPRING_DATASOURCE_URL` maps to `spring.datasource.url` — which makes them ideal for cloud deployments where secrets must not sit in a committed file. For sensitive values, environment variables or a secret store (Vault, cloud provider secrets) are appropriate, and placeholders allow composition: `spring.datasource.url: ${DATABASE_URL}`. Command-line arguments override everything, which is convenient for one-off runs: `java -jar app.jar --server.port=9090`. Beyond simple keys, `@PropertySource` on a `@Configuration` class loads an additional properties file, and `spring.config.import` (Spring Boot 2.4+) imports further files or config servers. Strongly-typed consumption comes from `@ConfigurationProperties` classes, which bind a whole prefix to a POJO validated at startup. The core idea is that runtime behavior is driven by the environment, not by rebuilding the artifact, and the ordered merging guarantees predictable, debuggable overrides at every level.

29. What is `@Value`?
    - **Answer:** `@Value` is a Spring annotation that injects a single externalized property value into a field, constructor parameter, or method parameter. It resolves the placeholder at injection time from the `Environment`, supporting default values with the `${key:default}` syntax and SpEL expressions with `#{...}` for computed values. For example:

    ```java
    @Service
    public class ReportService {
        @Value("${report.batch-size:1000}")
        private int batchSize;

        @Value("${report.output-dir:./reports}")
        private String outputDir;
    }
    ```

    `@Value` is simple and convenient for one-off values, but it has limitations: it returns the raw value (with type conversion happening on the fly but no IDE validation), it does not validate that a mandatory key exists unless the placeholder lacks a default (in which case startup fails), and it scatters configuration references across fields, making the set of properties a class depends on hard to see. For a small number of scalar settings `@Value` is perfectly fine; for related groups of properties, nested structures, or values that need validation, `@ConfigurationProperties` is the better tool because it binds whole trees to a typed, validated POJO. `@Value` can also inject values from sources other than property files, such as system properties and environment variables, and it works in `@Configuration` classes to parameterize `@Bean` definitions. On construction, `@Value` placeholders are resolved during dependency injection, so a missing placeholder without a default raises `IllegalArgumentException` at startup — a fail-fast behavior that is generally desirable. The annotation is also used to reference other properties inside property files themselves via nested placeholders like `${base.url}/api`.

30. What is `@ConfigurationProperties`?
    - **Answer:** `@ConfigurationProperties` binds an entire group of externalized properties — a prefix plus all its nested keys — to a strongly-typed Java object, replacing dozens of scattered `@Value` fields with one validated POJO. A prefix maps the class to the config tree:

    ```java
    @ConfigurationProperties(prefix = "inventory.validation")
    public class InventoryValidationProperties {
        private int batchSize = 100;
        private List<String> allowedSerialPrefixes = List.of();
        private Duration timeout = Duration.ofSeconds(5);
        // getters and setters
    }
    ```

    With `inventory.validation.batch-size=500` in `application.yml`, Boot binds the value into the `batchSize` field, using relaxed binding (kebab-case, camelCase, underscore all accepted) and automatic conversion to types like `Duration`, `DataSize`, and `List<String>`. The class becomes a bean when registered with `@ConfigurationProperties` + `@Component`, or more commonly via `@EnableConfigurationProperties(InventoryValidationProperties.class)` or `@ConfigurationPropertiesScan`, which scans for such classes. Because it is a bean, it can be injected anywhere like any other dependency. Additional features include JSR-380 validation — annotate fields with `@NotNull`, `@Min`, etc. and add `@Validated` to validate at startup, failing fast when configuration is wrong — and Java bean validation on nested `@Valid` structures. Compared with `@Value`, `@ConfigurationProperties` provides type safety, IDE support (autocomplete and validation of known keys when using the `spring-boot-configuration-processor`), a single visible object per config domain, and easy reuse in tests. It also maps naturally to immutable records in recent Spring Boot versions. The trade-off is slightly more ceremony: a dedicated class per configuration group is the recommended pattern for anything beyond a couple of scalar settings.

31. What is actuator?
    - **Answer:** Actuator is Spring Boot's production-ready module that exposes operational information about a running application over HTTP (or JMX) endpoints. Adding `spring-boot-starter-actuator` automatically configures a set of endpoints such as `/actuator/health` (application and component health), `/actuator/info` (arbitrary application metadata), `/actuator/metrics` (JVM, HTTP, and custom metrics), `/actuator/env` (all externalized property values), `/actuator/beans` (the full bean catalog), `/actuator/configprops` (validated configuration properties), `/actuator/loggers` (view and change log levels at runtime), `/actuator/mappings` (URL-to-handler map), `/actuator/heapdump` and `/actuator/threaddump`, and `/actuator/conditions` (the auto-configuration report). Endpoints are enabled (`management.endpoint.<name>.enabled`) and exposed (`management.endpoints.web.exposure.include/exclude`) independently, so the team decides which reach the network. Actuator integrates with Micrometer, Boot's metrics facade, so `/actuator/metrics` can feed Prometheus via `micrometer-registry-prometheus` and then Grafana dashboards. Health checks are extensible: a custom `HealthIndicator` reports the availability of a database, message broker, or external service, and can set the overall status to `UP`, `DOWN`, or `OUT_OF_SERVICE`; liveness and readiness probes (`/actuator/health/liveness` and `/actuator/health/readiness`) map directly to Kubernetes probes. The `@Endpoint` annotation allows building custom management endpoints. Actuator turns a black-box JAR into a diagnosable service — the operational value is that health, metrics, and configuration are inspectable without adding application code. Because several endpoints expose sensitive data, exposure should be restricted in production and protected by Spring Security.

32. Which actuator endpoints are useful in production?
    - **Answer:** In production the most valuable Actuator endpoints are `/actuator/health` for liveness and readiness probes, `/actuator/metrics` for runtime observability, `/actuator/info` for build and deployment metadata, and `/actuator/loggers` for changing log levels without restarting the process. `/health` is the primary signal for load balancers and Kubernetes (liveness/readiness) and aggregates custom `HealthIndicator`s, so a failing database or broker flips the status and takes the instance out of rotation. `/metrics` exposes JVM memory, garbage collection, thread counts, HTTP request counts and timings, and cache hit ratios via Micrometer; wired to a Prometheus/Grafana stack it provides the dashboards teams actually operate by. `/info` can be populated from `META-INF/build-info.properties` (via the Spring Boot Maven plugin's `build-info` goal) and Git commit metadata, which answers "which version is deployed" instantly. `/loggers` lets an operator raise the log level for a specific package during an incident and lower it afterwards, with no redeploy. Endpoints that expose internals — `/env` (property values, possibly including secrets), `/heapdump`, `/threaddump`, `/beans`, `/configprops`, `/mappings` — are useful for debugging but should be restricted to trusted networks or management users. `management.endpoints.web.exposure.include=health,info,metrics,loggers` is a sound production default, and each sensitive endpoint can be selectively enabled with additional management server or role-based access. The key production principle is least privilege: expose what operations genuinely needs, and use Spring Security to protect the rest.

33. How do you secure actuator endpoints?
    - **Answer:** Actuator endpoints are secured with defense in depth: limit what is exposed, isolate the management surface, and protect it with authentication and authorization. The first layer is exposure control — by default only `/health` is exposed over HTTP; the rest must be whitelisted explicitly with `management.endpoints.web.exposure.include`, and `exclude` removes undesired ones, e.g. `management.endpoints.web.exposure.include=health,info,metrics,loggers`. Some endpoints (notably `/shutdown`) are enabled only if explicitly switched on. The second layer is network isolation: `management.server.port=<port>` runs the management endpoints on a separate port (or even a separate address via `management.server.address`), so operational endpoints can be reachable only on an internal interface or firewall zone. The third layer is application-level security with Spring Security: define a dedicated `SecurityFilterChain` for the management path so that, for example, `/actuator/**` requires `ADMIN` authority while `/actuator/health` stays open for probes, for example:

    ```java
    @Bean
    public SecurityFilterChain actuatorChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/actuator/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().hasRole("ACTUATOR"))
            .httpBasic(Customizer.withDefaults())
            .build();
    }
    ```

    Health indicators should also be filtered so they do not leak implementation details; `management.endpoint.health.show-details=when-authorized` (or `never`) avoids exposing connection details publicly. In a cloud/container setting, exposing only `/health` publicly, keeping metrics internal, and relying on the orchestrator for probing is common. Any custom `@Endpoint` should be treated like a management endpoint and protected by the same rules, and secrets in `/env` (or elsewhere) must never be exposed to unauthorized users.

34. What is Spring Boot DevTools?
    - **Answer:** Spring Boot DevTools is a development-time-only dependency (`org.springframework.boot:spring-boot-devtools`) that speeds up the edit–compile–run loop. Its headline feature is automatic restart: when classes on the classpath change, DevTools restarts the application automatically, which is significantly faster than a full manual restart because it uses two classloaders — a base classloader that loads libraries that rarely change and a restart classloader that reloads only application classes. It also integrates LiveReload so a browser auto-refreshes when resources change, which is handy when developing a web UI. DevTools disables HTTP caching by default and, when a template engine like Thymeleaf is present, sets it to not cache templates, so edits appear immediately. It provides a few extra conveniences: a `spring.devtools.restart.trigger-file` to coordinate restart in teams or IDEs, exclusion of resources from triggering restart, and support for remote development where changes are pushed to a remote running app (with security implications). Critically, DevTools must never be on the classpath in production — its restart classloaders and property defaults are for developer machines only; the standard packaging (the executable JAR) ignores DevTools automatically, but the dependency should still be scoped as `runtime`/`optional`. DevTools also clears caches on restart and keeps a consistent in-memory state so restarts behave like fresh boots. It does not change production behavior at all, which is why it is safe to include in the build while remaining inert at runtime.

35. How do you handle exceptions globally?
    - **Answer:** Global exception handling in Spring Boot is done with `@RestControllerAdvice` (or `@ControllerAdvice`) classes containing `@ExceptionHandler` methods. A central handler class intercepts exceptions thrown by any controller and converts them into consistent JSON error responses with proper HTTP status codes, so controllers can focus on the happy path and error contract lives in one place. A typical implementation:

    ```java
    @RestControllerAdvice
    public class GlobalExceptionHandler {
        @ExceptionHandler(ResourceNotFoundException.class)
        public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse(404, "NOT_FOUND", ex.getMessage(), Instant.now()));
        }

        @ExceptionHandler(MethodArgumentNotValidException.class)
        public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
            String message = ex.getBindingResult().getFieldErrors().stream()
                .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
                .collect(Collectors.joining(", "));
            return ResponseEntity.badRequest().body(new ErrorResponse(400, "VALIDATION_FAILED", message, Instant.now()));
        }

        @ExceptionHandler(Exception.class)
        public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new ErrorResponse(500, "INTERNAL_ERROR", "An unexpected error occurred", Instant.now()));
        }
    }
    ```

    Exception resolution follows specificity: the most specific handler for the thrown type wins, and a catch-all `@ExceptionHandler(Exception.class)` provides a safety net for unexpected failures (without leaking stack traces to the client). Each handler controls the status code, response body shape, and headers, and logging should happen at the appropriate level inside the handler or via AOP. Errors from validation (`MethodArgumentNotValidException`), `@RequestBody` conversion (`HttpMessageNotReadableException`), 404s (`NoResourceFoundException`), and `DataAccessException` (from `@Repository` translation) are all common targets. The same approach works for REST (with `@RestControllerAdvice`) and supports `@ResponseStatus` on custom exceptions as a lighter alternative. The benefit is a uniform, predictable error contract across every endpoint, which frontend clients and API consumers can rely on.

36. What is `@ControllerAdvice`?
    - **Answer:** `@ControllerAdvice` is a Spring annotation that applies cross-cutting behavior to all controllers in the application, most commonly global exception handling via `@ExceptionHandler` methods, but also global model attributes (`@ModelAttribute`), data binding initialization (`@InitBinder`), and response body advice. Spring registers one bean of this type and applies it across controllers by default (a `basePackages` or `assignableTypes` attribute can restrict it to subsets). `@RestControllerAdvice` is a convenience variant that combines `@ControllerAdvice` with `@ResponseBody`, so every `@ExceptionHandler` method's return value is automatically serialized to JSON/XML — this is the standard choice for REST APIs because no per-method `@ResponseBody` is needed. The typical use is a single `GlobalExceptionHandler` that centralizes the error contract: one class defines `@ExceptionHandler` methods for validation failures, missing resources, authorization failures, data-access errors, and generic exceptions, each producing the same `ErrorResponse` shape (code, message, timestamp, trace ID), so clients see one consistent format across every endpoint. Because the advice wraps the whole controller layer, exceptions thrown inside services and repositories propagate up and are caught there too. Ordering matters when multiple advice classes exist: `@Order` or `@ControllerAdvice(order = ...)` determines precedence, and more specific exception types win over a catch-all handler in the same class. The mechanism works through `ExceptionHandlerExceptionResolver`, which Spring MVC consults after a controller method throws, matching the thrown exception against registered handlers. This centralization keeps controllers thin, prevents duplicated try/catch logic, and makes error handling testable in isolation with `MockMvc`.

37. What is `@ExceptionHandler`?
    - **Answer:** `@ExceptionHandler` is a method-level annotation that declares: "this method handles the listed exception types when they are thrown during request handling." It is used inside a controller to handle that controller's exceptions locally, or — much more commonly — inside an `@ControllerAdvice`/`@RestControllerAdvice` class to handle them globally for all controllers. The handler method receives the exception (and optionally the request and handler details), and its return value is treated like any controller return value: with `@RestControllerAdvice` it is serialized directly to the response body, or it can return a `ResponseEntity<T>` for full control over status, headers, and body. The resolver picks the most specific matching handler: if `@ExceptionHandler(DataAccessException.class)` and `@ExceptionHandler(SQLException.class)` both exist and `BadSqlGrammarException` (a `DataAccessException` subclass) is thrown, the handler whose exception type is closest in the hierarchy wins. Multiple exception types can be handled by one method — `@ExceptionHandler({ItemNotFoundException.class, OrderNotFoundException.class})` — and a catch-all `@ExceptionHandler(Exception.class)` guards against anything unhandled. A minimal example:

    ```java
    @ExceptionHandler(ItemNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleItemNotFound(ItemNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("ITEM_NOT_FOUND", ex.getMessage()));
    }
    ```

    When no handler matches, Spring's default error handling applies (the Basic Error Controller / `/error`), producing Boot's default error JSON. `@ExceptionHandler` also pairs with custom exceptions carrying `@ResponseStatus` or with `ResponseEntity`-based responses for fine-grained status control. Well-designed handler methods return a stable error contract and never expose raw stack traces to clients, while logging the full exception server-side.

38. What is validation in Spring Boot?
    - **Answer:** Validation in Spring Boot is built on the Bean Validation API (JSR-380), implemented by Hibernate Validator, and wired in automatically when `spring-boot-starter-validation` is on the classpath. Validation is declared declaratively with constraint annotations on DTO fields — `@NotNull`, `@NotBlank`, `@Size`, `@Min`/`@Max`, `@Pattern`, `@Email` — and triggered when the object is validated, typically with `@Valid` on a `@RequestBody` parameter. For example:

    ```java
    public record CreateItemRequest(
        @NotBlank(message = "name is required") String name,
        @Pattern(regexp = "^[A-Z0-9]+$", message = "invalid serial number") String serialNumber,
        @Min(1) BigDecimal price
    ) {}
    ```

    When validation fails, Spring throws `MethodArgumentNotValidException` (for `@RequestBody`) or `ConstraintViolationException` (for method parameters), which a `@RestControllerAdvice` handler turns into a 400 response with per-field messages. Validation is not limited to request bodies: `@Validated` at class level enables validation of `@RequestParam` and `@PathVariable` constraints, and `@Valid` on nested objects cascades validation into their fields. Constraint groups (empty marker interfaces passed to `@Validated`) allow different rule sets for create vs. update operations. Validation also applies to configuration: `@Validated` + JSR-380 annotations on `@ConfigurationProperties` classes fail startup when a bound value is invalid, catching misconfiguration early. Custom validators extend the framework by writing a constraint annotation plus a `ConstraintValidator` implementation. Validation improves API robustness by catching malformed input at the boundary, before any service or persistence code runs, and the framework standardizes error messages instead of relying on ad-hoc checks.

39. What is `@Valid`?
    - **Answer:** `@Valid` is the standard Java Bean Validation annotation (from `jakarta.validation`) that triggers validation of the annotated object's constraint annotations. In a Spring MVC controller, placing it on a `@RequestBody` parameter makes Spring run the JSR-380 validator on the deserialized object before the handler method is invoked:

    ```java
    @PostMapping("/items")
    public Item createItem(@Valid @RequestBody CreateItemRequest request) {
        return itemService.save(request);
    }
    ```

    If any constraint on `CreateItemRequest` (or on nested objects reached via `@Valid` fields) fails, the handler is not called; instead Spring throws `MethodArgumentNotValidException`, which a global exception handler converts into a 400 response with the field errors. `@Valid` can also be applied to `@RequestPart` for multipart bodies, to method parameters such as `@PathVariable`/`@RequestParam` when combined with `@Validated` at class level, and to nested fields to cascade validation into child objects and collections (`List<@Valid Item>`). Because validation happens at the framework boundary, service and repository code can assume inputs already satisfy their declared constraints. `@Valid` only triggers validation configured through JSR-380 annotations; for Spring-specific extensions like validation groups (different rules per operation), the container must instead use Spring's `@Validated`, which is a meta-annotation that carries `@Valid` semantics plus group support. A common mistake is forgetting `@Valid`, which silently skips all constraint checking and lets invalid data flow into the service layer.

40. Difference between `@Valid` and `@Validated`.
    - **Answer:** `@Valid` is the standard JSR-380 annotation that triggers Bean Validation on the annotated parameter or field; `@Validated` is Spring's annotation that triggers the same validation but adds Spring-specific capabilities, most importantly validation groups. With `@Valid` alone, all constraints always apply. With `@Validated(GroupA.class)` the container only enforces constraints belonging to `GroupA`, which enables different rules for the same DTO in different scenarios — a classic example is stricter rules on create than on update:

    ```java
    public interface OnCreate {}
    public interface OnUpdate {}

    public record CreateItemRequest(
        @NotBlank(groups = OnCreate.class) String name,
        @NotNull(groups = OnUpdate.class) Long id
    ) {}

    // Controller
    public Item create(@Validated(OnCreate.class) @RequestBody CreateItemRequest request) { ... }
    public Item update(@Validated(OnUpdate.class) @RequestBody CreateItemRequest request) { ... }
    ```

    Technically, `@Validated` is itself meta-annotated with `@Valid`, so it provides `@Valid` behavior as well. Another difference is scope: `@Valid` is applied at parameter/field level, whereas `@Validated` is also applied at the class level to enable validation of method parameters like `@RequestParam` and `@PathVariable` (`ConstraintViolationException` is then thrown instead of `MethodArgumentNotValidException`). Validation groups are declared as empty marker interfaces, and a constraint that specifies no `groups` belongs to the default group, which is the only one `@Valid` enforces. In practice, REST controllers default to `@Valid`; `@Validated` is chosen when groups or method-parameter validation is needed. Both ultimately delegate to the same `Validator` from Hibernate Validator.

41. What is scheduling in Spring Boot?
    - **Answer:** Scheduling in Spring Boot is the framework-managed execution of methods at fixed intervals, fixed delays, or cron-based times, enabled by adding `@EnableScheduling` to a configuration class (it is often placed on the main class alongside `@SpringBootApplication`). Once enabled, any `@Scheduled`-annotated public method runs automatically under the container's `TaskScheduler`, no manual thread management required. The scheduling infrastructure is a `ThreadPoolTaskScheduler`; by default Spring Boot configures a pool sized to available processors (or a single thread in older versions), and a custom bean of type `TaskScheduler`/`SchedulingConfigurer` overrides the defaults. Scheduled methods run asynchronously relative to the request lifecycle and are typically used for maintenance work — cache warming, data aggregation, report generation, cleanup of stale records, and integration polling. Because scheduled methods are ordinary Spring beans, they can use dependency injection, transactions, and the rest of the framework, and they are proxied like any bean, so `@Transactional` and `@Async` behavior apply. Failure handling is important: an uncaught exception in a scheduled method is logged by the scheduler's error handler (which can be customized) and the task stops for that cycle but does not kill the process; with fixed-delay semantics a repeated failure simply stops future runs until fixed. Scheduling should be combined with idempotency or locking when multiple instances run the same job, and with sufficient thread pooling when several tasks overlap. Boot also supports programmatic scheduling via a `SchedulingConfigurer` for registrations that depend on configuration values.

42. What is `@Scheduled`?
    - **Answer:** `@Scheduled` is the annotation that marks a public method for automatic execution according to a schedule. It supports three scheduling modes via attributes: `fixedRate`, `fixedDelay`, and `cron`. `fixedRate = 5000` runs the method every five seconds measured from the start of each run, without waiting for the previous run to finish. `fixedDelay = 5000` waits five seconds after the previous run completes before starting the next, guaranteeing no overlap. `cron = "0 0 2 * * ?"` runs at a specific calendar time (here 2:00 AM daily); the standard Spring cron format has six fields (second, minute, hour, day-of-month, month, day-of-week), and `zone` can be set to pin the timezone. Initial delay is configured with `initialDelay` (in ms) or via cron's `0`-delayed variants, useful when the application needs warm-up time before the first run. A typical use:

    ```java
    @Component
    public class ReportScheduler {
        @Scheduled(cron = "0 0 2 * * ?", zone = "UTC")
        public void generateNightlyReport() {
            reportService.buildDailyReport();
        }
    }
    ```

    The method must be public, return void (or a future-returning type in some setups), and take no arguments; it must be called from outside the class because Spring schedules it through a proxy, so self-invocation of a `@Scheduled` (or `@Async`) method bypasses the schedule. Scheduling only works when `@EnableScheduling` is active. It is common to read the schedule values from configuration using placeholders, e.g. `@Scheduled(cron = "${report.cron:0 0 2 * * ?}")`, so ops can change timing without rebuilding. Under the hood each annotated method is registered as a `ScheduledTask` on the `TaskScheduler`, which dispatches it on a worker thread.

43. Difference between fixed rate and fixed delay.
    - **Answer:** `fixedRate` schedules the next run at a fixed interval measured from the start of the current run, regardless of how long that run takes; `fixedDelay` schedules the next run at a fixed interval measured from the completion of the previous run. With `fixedRate = 10s`, if a run takes 12 seconds, runs begin at 0s, 12s, 22s... because a new invocation fires as soon as the timer window closes while the previous one is still running. With `fixedDelay = 10s`, a run that takes 12 seconds means the next begins at 22s — the gap is always 10 seconds after completion, so executions never overlap. The two are therefore suited to different workloads: `fixedDelay` is the safe choice for jobs that must never run concurrently and where drift is acceptable (batch updates, report generation, anything mutating shared state). `fixedRate` suits monitoring/keep-alive tasks where a consistent start cadence matters more than avoiding overlap, or tasks guaranteed to be short relative to the interval. The main hazard of `fixedRate` is overlap: if the default single-threaded scheduler is in use and the task outruns the interval, invocations queue up or starve other scheduled tasks, and duplicate concurrent execution of a non-idempotent job can corrupt data. Fixed-delay jobs that are slow still drift (the cycle becomes interval + runtime), which matters for jobs that must hit a precise clock time — for those a cron expression is the correct tool. In general, default to `fixedDelay` for correctness and reserve `fixedRate` for cases where cadence dominates.

44. What is cron expression?
    - **Answer:** A cron expression is a compact schedule string that Spring parses into exact firing times. Spring's variant uses six fields — second, minute, hour, day-of-month, month, day-of-week — separated by spaces (`"0 0 2 * * ?"` means at 2:00:00 every day; a seventh, optional year field exists in some dialects). Each field accepts single values (`5`), ranges (`10-20`), lists (`1,15,30`), wildcards (`*` = every unit), and increments (`*/5` = every 5 units). Two fields deserve special attention: `?` means "no specific value" and is used in exactly one of day-of-month/day-of-week to avoid ambiguity (e.g., `0 0 12 ? * MON` = noon every Monday), while `*` means "every value". Month and day-of-week names are case-insensitive (`JAN`, `MON`). Common examples: `0 0/15 * * * ?` every 15 minutes, `0 0 0/1 * * ?` every hour at the top of the hour, `0 0 2 ? * 1#1` the first Sunday of each month (1#1). Two recurring pitfalls: cron fires in the server's default timezone unless the `zone` attribute pins one (`@Scheduled(cron = "...", zone = "UTC")`), and a scheduled run that takes longer than the interval overlaps with the next fire — cron does not skip a missed fire when the previous run is still executing. Cron is the right tool when work must happen at wall-clock times (nightly batches, market-open jobs) rather than relative intervals. Spring also accepts predefined macros like `@Scheduled(cron = "@daily")` in some contexts, though the raw expression is the widely used form.

45. How do you configure scheduled task thread pool?
    - **Answer:** By default Spring Boot uses a single-threaded scheduler for `@Scheduled` tasks, so multiple jobs can queue behind one long-running job. The fix is to provide a `TaskScheduler` bean (Boot auto-detects a bean of that type and uses it) with a configured `ThreadPoolTaskScheduler`:

    ```java
    @Configuration
    public class SchedulingConfig {
        @Bean(name = "taskScheduler")
        public ThreadPoolTaskScheduler taskScheduler() {
            ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
            scheduler.setPoolSize(5);
            scheduler.setThreadNamePrefix("scheduled-");
            scheduler.setErrorHandler(t -> log.error("Scheduled task failed", t));
            scheduler.setWaitForTasksToCompleteOnShutdown(true);
            scheduler.setAwaitTerminationSeconds(30);
            scheduler.initialize();
            return scheduler;
        }
    }
    ```

    Alternatively, implement `SchedulingConfigurer.configureTasks(ScheduledTaskRegistrar registrar)` and call `registrar.setScheduler(...)`, which also allows programmatic registration of `@Scheduled`-style tasks from configuration values. The pool size should reflect the number of jobs and how much overlap is acceptable: too few threads and jobs queue behind each other (same problem as single-threaded), too many and idle threads consume memory. The `ErrorHandler` is important because an uncaught exception in a scheduled method can silently stop subsequent runs; logging failures keeps them visible. `setWaitForTasksToCompleteOnShutdown(true)` and an `awaitTerminationSeconds` value let in-flight jobs finish during a graceful shutdown instead of being cut off. A scheduler should also be sized together with the async thread pool: they are separate pools, and a job that internally calls `@Async` methods consumes threads from both. In multi-instance deployments, the pool configuration must be combined with a distributed lock so concurrent instances do not duplicate job execution.

46. How do you prevent scheduled jobs from running on all pods?
    - **Answer:** With multiple instances (pods, replicas) of the same application, every instance would execute the same `@Scheduled` jobs, causing duplicate work, double side effects, and wasted resources. The standard solution is a distributed lock so exactly one instance runs a given job at a time. The most common library is ShedLock, which coordinates via a shared store (Redis, JDBC table, Mongo, DynamoDB) and guards `@Scheduled` methods with `@SchedulerLock`:

    ```java
    @Component
    public class ReportScheduler {
        @Scheduled(cron = "0 0 2 * * ?")
        @SchedulerLock(name = "nightly-report", lockAtMostFor = "PT15M", lockAtLeastFor = "PT5M")
        public void generateReport() {
            reportService.buildReport();
        }
    }
    ```

    `lockAtMostFor` bounds how long the lock can be held if the instance dies mid-run; `lockAtLeastFor` prevents re-execution of fast jobs within the same window. The store needs a schema, and ShedLock can be configured as a `@Bean` (`ShedLockProvider`) using the shared Redis/JDBC connection. Alternative approaches include Spring Integration's `RdbmsLockRegistry` or `RedisLockRegistry` with a `SchedulerLock` advisory pattern, database-specific row locks (`SELECT ... FOR UPDATE` with a job table) combined with `@Transactional`, and leader-election mechanisms from Spring Cloud (`@EnableLeaderElection` / Spring Cloud Kubernetes leader). For non-critical jobs, a simpler pattern is a distributed counters/lock in Redis via `RedissonLock` or `RedisTemplate` SET NX with expiry. Whatever the mechanism, the job itself should still be idempotent as a second line of defense, and `lockAtMostFor` must comfortably exceed the expected maximum runtime so a healthy instance is never prevented from re-acquiring. ShedLock's `lockAtLeastFor`/`lockAtMostFor` and its presence in Boot starters make it the de facto default for straightforward cases.

47. What is async processing?
    - **Answer:** Asynchronous processing means a method or piece of work is dispatched to run on a separate thread, letting the caller return immediately instead of blocking on the work's completion. In Spring Boot this is typically done with `@Async` methods backed by a `TaskExecutor` (an abstraction over `ThreadPoolExecutor`). The value is responsiveness and decoupling: the HTTP request thread returns as soon as the async call is enqueued, so slow operations — sending email, writing audit logs, calling third-party APIs, generating reports — do not add latency to the API response. Async is distinct from reactive programming (which uses event loops and back-pressure rather than blocking threads) and from parallelism (which is about splitting CPU work across cores rather than fire-and-forget dispatch). The critical engineering concerns are the thread pool and failure handling: an unconstrained pool grows unboundedly under load and can exhaust memory or overwhelm downstream systems, so the executor's core/max pool sizes, queue capacity, and rejection policy must be tuned to the workload; `CallerRunsPolicy` (run the task on the caller's thread when the pool is full) is a common back-pressure choice. Because the caller no longer waits, results are communicated via `CompletableFuture<T>` return types, callbacks, or an outbox/queue, and errors must be caught inside the async method or handled by an `AsyncUncaughtExceptionHandler` — an exception in a fire-and-forget task otherwise disappears. Async also breaks transactional guarantees if the caller's transaction is expected to cover the async work, since the work runs on a different thread with its own transaction context. Used with bounded pools and clear failure semantics, async processing is a simple, effective way to make services responsive and resilient to slow dependencies.

48. What is `@Async`?
    - **Answer:** `@Async` marks a method to be executed asynchronously by a `TaskExecutor` instead of on the caller's thread. Enabling it requires `@EnableAsync` on a configuration class. The annotated method must be public, and it must be invoked through the Spring proxy — i.e., called from another bean — because the call is intercepted and dispatched to the executor; a self-invocation (`this.method()`) bypasses the proxy and runs synchronously, a classic silent bug. The return type is constrained: void for fire-and-forget, or `CompletableFuture<T>`/`Future<T>` if the caller wants the result or to await completion. For example:

    ```java
    @Service
    public class AuditService {
        @Async
        public CompletableFuture<Boolean> recordAudit(Long orderId, String action) {
            auditRepository.save(new AuditLog(orderId, action, Instant.now()));
            return CompletableFuture.completedFuture(true);
        }
    }
    ```

    The caller then does `auditService.recordAudit(...)`, which returns immediately. `@Async` composes with the bean lifecycle: the proxy is applied at initialization, so the method runs in the container's `TaskScheduler`/`TaskExecutor` — Boot's default `SimpleAsyncTaskExecutor` in absence of a custom one is not a real pool (it spawns a new thread per task, which is dangerous under load), so a properly sized `ThreadPoolTaskExecutor` bean should be defined. Transactions inside an `@Async` method start their own transaction (the caller's transaction, if any, does not propagate to the new thread). Exceptions thrown inside a void `@Async` method are handled by the `AsyncUncaughtExceptionHandler`; for `CompletableFuture` returns the exception surfaces in the future. `@Async` is the right tool for fire-and-forget work with bounded pools, while heavier concurrency needs (parallel fan-out, streaming) point toward reactive or explicit executor usage.

49. How do you configure async executor?
    - **Answer:** Async execution is configured by defining a `TaskExecutor` bean — typically `ThreadPoolTaskExecutor` — that the container uses for `@Async` methods. Boot auto-detects a bean of type `Executor`/`TaskExecutor` and routes `@Async` calls through it; a single bean is used unless `@Async("name")` targets a specific one. A tuned configuration:

    ```java
    @Configuration
    @EnableAsync
    public class AsyncConfig {
        @Bean(name = "taskExecutor")
        public ThreadPoolTaskExecutor taskExecutor() {
            ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
            executor.setCorePoolSize(10);
            executor.setMaxPoolSize(25);
            executor.setQueueCapacity(100);
            executor.setThreadNamePrefix("async-");
            executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
            executor.initialize();
            return executor;
        }
    }
    ```

    The numbers matter: core threads are created up to `corePoolSize`; beyond that tasks are queued (up to `queueCapacity`); only when the queue is full does the pool grow toward `maxPoolSize`; if that is also saturated, the rejection policy applies. `CallerRunsPolicy` runs the task on the submitting thread, which applies natural back-pressure rather than dropping work — preferable to `AbortPolicy`, which throws and loses the task. `setWaitForTasksToCompleteOnShutdown(true)` with `setAwaitTerminationSeconds(...)` drains in-flight tasks during graceful shutdown. The default `SimpleAsyncTaskExecutor` (used when no executor bean exists) creates an unbounded number of threads and must be replaced in anything beyond trivial use. Exceptions: an `AsyncConfigurer` can declare the executor and an `AsyncUncaughtExceptionHandler` together, capturing failures from void async methods. Sizing should be based on the workload's I/O-vs-CPU profile and downstream capacity; the async pool and the scheduler pool are separate, and both must be considered together when a scheduled job fans out to async work.

50. What is caching in Spring Boot?
    - **Answer:** Caching stores the results of expensive operations so repeated calls with the same inputs can be served from memory instead of recomputing or re-querying. Spring Boot supports caching through the Spring Cache abstraction — a set of interfaces (`CacheManager`, `Cache`, key generation) with pluggable providers: in-memory `ConcurrentMapCacheManager`, `CaffeineCacheManager`, and distributed stores like Redis or Ehcache. Caching is enabled with `@EnableCaching` and applied declaratively with `@Cacheable`, `@CachePut`, and `@CacheEvict` annotations on service methods, where the method's arguments form the cache key (or an explicit SpEL `key`). The abstraction sits in front of the method: `@Cacheable` checks the cache first and only executes the method on a miss. The payoff is lower database/network load, lower latency, and better throughput for hot data — reference data, configuration, catalogs, session-adjacent lookups — that changes rarely. Design considerations include: which data is cacheable (stale-tolerant, small-to-moderate size), the TTL/expiration strategy, cache invalidation on writes, and consistency between multiple instances (each instance with local memory must evict on the writer, or use a shared store like Redis). The abstraction makes the provider swappable with configuration (`spring.cache.type=redis`), so the same `@Cacheable` code runs against Caffeine locally and Redis in production. `CacheManager` selection and provider specifics are auto-configured when the matching library is on the classpath. Metrics from Actuator (cache hit/miss) help tune TTLs, and `@CacheEvict`/`@CachePut` keep data coherent. Caching is a performance technique, so it should be applied to measured bottlenecks and always coupled with an invalidation strategy.

51. What is `@Cacheable`?
    - **Answer:** `@Cacheable` is the annotation that marks a method's result as cacheable: on invocation, the cache abstraction checks the named cache using a key derived from the method arguments, and only executes the underlying method if the key is absent, storing the result afterwards. On subsequent calls with the same arguments, the cached value is returned and the method body is skipped entirely — the classic read-through pattern:

    ```java
    @Service
    public class DeviceService {
        @Cacheable(cacheNames = "deviceConfigs", key = "#deviceId")
        public DeviceConfig getConfig(String deviceId) {
            return deviceRepository.findByDeviceId(deviceId);
        }
    }
    ```

    The default key is built from all method parameters; `key` overrides it with SpEL (`#deviceId`, `#root.methodName`, composite keys like `#tenantId + ':' + #deviceId`). `condition` controls whether caching is considered at all (evaluated before the method), and `unless` prevents storing a result (evaluated on the returned value, e.g. `unless = "#result == null"` or `unless = "#result.isExpired()"`). Cache names are backed by a `CacheManager` — in-memory, Caffeine, or Redis — and TTL/eviction policy lives at the manager/provider level, not in the annotation. Important subtleties: caching is applied through a proxy, so a self-invocation inside the same bean bypasses the cache (and the method then runs every time); return values are stored by reference in simple providers, so mutating a returned object corrupts the cache; and `@Cacheable` inside one transaction participates in the same call. Because the annotation sits at the boundary of the service, `@Cacheable` should wrap read-only, idempotent, deterministic methods. Invalidations belong in the write path: `@CachePut` to refresh, or `@CacheEvict` after mutations. Since the cache silently hides the real method, it must be evicted/expired in sync with data changes or it serves stale data.

52. Difference between `@Cacheable`, `@CachePut`, and `@CacheEvict`.
    - **Answer:** The three annotations manipulate the cache at different points of method execution. `@Cacheable` checks the cache before the method: if the key exists, the cached value is returned and the method is not called; if absent, the method runs and its result is stored — so the method is the read-through loader. `@CachePut` always executes the method and always writes the result into the cache, regardless of whether the key was present — it is the write-through/update annotation, used on update operations to refresh the cached value in the same call that persists the change. `@CacheEvict` does not care about the method's result; it removes entries from the cache before or after the method runs — used on delete/update operations to drop stale data. Example semantics:

    ```java
    @Cacheable(cacheNames = "products", key = "#id")          // read-through: load on miss
    Product find(long id) { ... }

    @CachePut(cacheNames = "products", key = "#product.id")    // always write
    Product update(Product product) { ... }

    @CacheEvict(cacheNames = "products", key = "#id")          // remove one entry
    void delete(long id) { ... }

    @CacheEvict(cacheNames = "products", allEntries = true)    // clear whole cache
    void reload() { ... }
    ```

    `@CacheEvict` supports `allEntries` to clear a whole cache and `beforeInvocation = true` to evict before the method runs (so eviction still happens if the method throws), which is the safe choice when failure should still invalidate stale data. Multiple annotations can be combined with `@Caching` (e.g., evict one cache and put another). A key point: `@Cacheable` writes only on miss (unless configured), `@CachePut` writes always, `@CacheEvict` never writes — choosing between them is deciding whether the method is a loader, an updater, or an invalidator. Keeping them consistent — put/evict wherever the underlying data mutates — is what prevents stale reads.

53. How do you use Redis cache with Spring Boot?
    - **Answer:** Using Redis as the cache provider requires the Redis starter, connection configuration, and a `RedisCacheManager` (or the generic `spring.cache.type=redis`). The dependency `spring-boot-starter-data-redis` brings in Lettuce and Spring Data Redis; the connection is configured with `spring.data.redis.host`, `port`, and optional password in `application.yml`. With `@EnableCaching` active and the starter present, Boot auto-configures a `RedisCacheManager`, so `@Cacheable` on service methods immediately uses Redis. Tuning is done through a `RedisCacheManagerBuilderCustomizer` bean:

    ```java
    @Bean
    public RedisCacheManagerBuilderCustomizer redisCacheCustomizer() {
        return builder -> builder
            .withCacheConfiguration("deviceConfigs",
                RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofMinutes(30))
                    .serializeValuesWith(SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer())));
    }
    ```

    Redis caches store values as serialized bytes, so serialization matters: JSON via `GenericJackson2JsonRedisSerializer` (readable, interoperable) or binary Java/Kryo serialization (compact). `RedisTemplate` is the direct-access API for imperative operations (`opsForValue().get/set`, `opsForHash()`), separate from the cache abstraction and useful when annotations are insufficient. Failure handling is a real concern: a Redis outage should degrade gracefully — the cache abstraction can be wrapped with a fallback so reads fall through to the database, and TTLs bound memory growth. Multi-instance deployments benefit from Redis automatically because all pods share one cache, eliminating per-instance invalidation problems (though eviction via `@CacheEvict` still must hit all pods or use Redis pub/sub, which ShedLock-style coordination or a cache stampede guard can address). Redis also serves other roles — distributed locks, pub/sub, rate limiting, session storage — so caching often reuses existing infrastructure. The trade-off is added network latency per cache access, so Redis caching is most valuable when the alternative is an expensive query or call.

54. How do you write REST APIs in Spring Boot?
    - **Answer:** REST APIs in Spring Boot are built with `@RestController` controllers, which combine `@Controller` with `@ResponseBody` so each handler method's return value is serialized to JSON (via Jackson) and written to the HTTP response. Class-level `@RequestMapping("/api/items")` defines the base path; method-level `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`, and `@PatchMapping` map HTTP verbs. Requests are bound with `@RequestBody` (JSON payload deserialized into a DTO/record), `@PathVariable` (path segment), `@RequestParam` (query string), and `@RequestHeader`; responses are wrapped in `ResponseEntity<T>` when the status code, headers, or body need explicit control:

    ```java
    @RestController
    @RequestMapping("/api/items")
    public class ItemController {
        private final ItemService itemService;

        public ItemController(ItemService itemService) {
            this.itemService = itemService;
        }

        @GetMapping("/{id}")
        public ResponseEntity<ItemDto> getItem(@PathVariable Long id) {
            return ResponseEntity.ok(itemService.findById(id));
        }

        @PostMapping
        @ResponseStatus(HttpStatus.CREATED)
        public ItemDto createItem(@Valid @RequestBody CreateItemRequest request) {
            return itemService.create(request);
        }
    }
    ```

    Good REST practice is layered: controllers handle HTTP concerns (mapping, binding, validation), services hold business logic, and repositories handle persistence; DTOs rather than entities cross the boundary so internal models do not leak. Status codes should be meaningful (200/201/204/404/400/409), errors should follow a uniform contract via `@RestControllerAdvice`, inputs validated with `@Valid`, and the API documented with OpenAPI. `ResponseEntity` allows `Location` headers, `Cache-Control`, and conditional `ETag` handling for more advanced clients. Consistent naming, versioning (commonly `/api/v1/...`), content negotiation via `Accept`, and idempotency for writes are the characteristics that make an API usable at scale. Boot's embedded server plus starters mean a working endpoint exists within minutes, while the discipline of resource modeling and DTO boundaries keeps it maintainable.

55. How do you version APIs?
    - **Answer:** API versioning lets multiple versions of the same contract coexist so consumers migrate gradually without breaking. The most common strategy is URI path versioning: `/api/v1/items` and `/api/v2/items`, implemented simply as a path segment in `@RequestMapping("/api/v1/items")` and `/api/v2/items`. It is explicit — the version is visible in logs, monitoring, and documentation — and easy to route at a gateway or load balancer. Alternatives include header-based versioning (e.g., `Accept: application/vnd.company.v2+json` or a custom `X-API-Version` header), which keeps URLs clean but hides the version from logs and requires content-negotiation or interceptor logic; query-parameter versioning (`/items?version=2`), which is simple but pollutes URLs and is easily forgotten by consumers; and host/subdomain versioning for coarse API generations. Spring Boot supports custom versioning via `WebMvcConfigurer` interceptors or by mapping both path patterns and dispatching on a header, but path versioning is the default recommendation for its explicitness. Practical guidance: version at the point of a breaking change (removed/renamed fields, changed semantics), keep the old version alive for a deprecation window, and design v2 as additive where possible (new fields, new endpoints) so migration is optional. Backward compatibility also involves response shape: adding fields is usually safe; changing types or deleting fields is not. Because versioned controllers duplicate logic, extract shared service logic and keep only the contract mapping (DTO conversion) version-specific. There is no single right answer — the choice balances discoverability, routing simplicity, and how clients consume the API.

56. How do you document APIs?
    - **Answer:** REST APIs in Spring Boot are documented with the OpenAPI 3 specification, served through `springdoc-openapi`, which auto-generates an OpenAPI document from the application's controllers and model classes. Adding `org.springdoc:springdoc-openapi-starter-webmvc-ui` makes `/v3/api-docs` return the JSON/YAML spec and `/swagger-ui.html` render an interactive Swagger UI. The generated document is enriched with annotations: `@Operation` adds a human-readable description, `@Parameter` documents individual parameters, `@ApiResponse` (or `@ApiResponses`) describes status codes and error schemas, and `@Tag` groups related endpoints. For example:

    ```java
    @Operation(summary = "Create an item", description = "Validates and stores a new item")
    @ApiResponses({
        @ApiResponse(responseCode = "201", description = "Created"),
        @ApiResponse(responseCode = "400", description = "Validation failed",
                     content = @Content(schema = @Schema(implementation = ErrorResponse.class)))
    })
    @PostMapping
    public ResponseEntity<ItemDto> create(@Valid @RequestBody CreateItemRequest request) { ... }
    ```

    Because annotations and DTOs drive the spec, documentation stays in sync with code — the main alternative is hand-written OpenAPI YAML files, which decouple docs from code but drift. The OpenAPI spec enables more than docs: API clients can be generated, mock servers can be produced, and contract tests can validate that the running API matches the spec. Configuration is externalized (`springdoc.api-docs.enabled`, `springdoc.swagger-ui.enabled`) so docs can be disabled in production or scoped to staging. Security schemes (Bearer/JWT) are declared once in an `OpenAPI` bean and referenced by all secured operations, so Swagger UI gains an Authorize button. Request/response examples, default values, and schemas defined with `@Schema` improve the consumer experience; grouping controllers into tagged sections keeps large APIs navigable.

57. What is Swagger/OpenAPI?
    - **Answer:** OpenAPI is a vendor-neutral, machine-readable specification (originally derived from Swagger 2.0) for describing REST APIs in JSON or YAML: every path, HTTP method, parameter, request/response schema, error code, and security scheme. Swagger is the ecosystem around the spec — the older "Swagger" umbrella now refers mainly to the tooling (Swagger UI, Swagger Editor, code generators), while "OpenAPI" denotes the specification itself. An OpenAPI document's core sections are `openapi` (version), `info` (title/version/description), `servers` (base URLs), `paths` (per-endpoint operations with parameters and responses), `components` (reusable schemas, parameters, and security schemes), and `security` (global auth requirements). In Spring Boot, `springdoc-openapi` derives this document automatically from `@RestController` classes, method signatures, and DTOs, so the spec is generated rather than hand-maintained; the document powers the interactive Swagger UI (`/swagger-ui.html`) where consumers can inspect endpoints and execute requests. The spec's value is that it is the contract: frontend teams read it to integrate, QA can generate tests from it, code generators can produce client SDKs in many languages, and API gateways and mock servers can consume it directly. OpenAPI 3.0 added clearer request-body modeling (`requestBody` vs `parameters`), examples, and reworked security definitions compared with Swagger 2.0. Standardization means one document speaks to all tools — documentation, mocking, testing, and governance are unified. Security definitions (Bearer tokens, API keys, OAuth flows) are declared declaratively, and the same spec serves as both documentation and a conformance baseline.

58. What is Spring Boot testing?
    - **Answer:** Spring Boot testing is the framework's approach to testing applications across the test pyramid, from fast isolated unit tests to full-context integration tests, with a cohesive set of tools and slices. At the unit level, plain JUnit 5 tests construct service objects with mocks (Mockito) and need no Spring context at all — fast and deterministic, covering most business logic. `@SpringBootTest` loads the complete `ApplicationContext` (replacing the real environment with an embedded database, test properties, or `@MockBean` dependencies) for integration tests that verify beans wire together and end-to-end flows work. Between the two, slice tests load only a relevant portion of the context: `@WebMvcTest` starts just the web layer with `MockMvc` (controllers, filters, MVC config), `@DataJpaTest` boots only the JPA/`@Repository` layer on an embedded database, and `@JsonTest` tests JSON serialization in isolation — each dramatically faster than the full context. `@MockBean`/`@MockitoBean` replace specific beans with mocks for isolated tests, and `Testcontainers` spins up real databases/brokers so tests run against the same software as production. Spring provides utilities like `TestRestTemplate`, `WebTestClient`, `MockMvc`, `@TestPropertySource` (or `properties` attribute) to override configuration, and `@DirtiesContext` to reset the context when tests mutate it. Boot also aligns with `@SpringBootTest`'s embedded web environment options (`MOCK`, `RANDOM_PORT`, `DEFINED_PORT`). Good practice is the pyramid: many fast unit tests, a moderate number of slice tests, and a smaller set of high-value `@SpringBootTest`/Testcontainer integration tests — prioritizing speed while still covering wiring, transactions, and framework behavior that mocks cannot.

59. What is `@SpringBootTest`?
    - **Answer:** `@SpringBootTest` is the annotation that boots the entire Spring Boot application context for integration testing. It locates the `@SpringBootApplication` class (via the test class's package or an explicit `classes` attribute), starts the context exactly as production would, and wires it into the test, so all auto-configuration, beans, and properties participate. By default it uses `webEnvironment = WebEnvironment.MOCK`, creating the context without a real server so `MockMvc` can drive requests in-process; `RANDOM_PORT` starts the embedded server on a random port and is paired with `TestRestTemplate` or `WebTestClient` for real HTTP calls; `DEFINED_PORT` uses the configured port. Because the full context is expensive to build, JUnit caches the context between tests with the same configuration (`ApplicationContext` caching), so a suite of `@SpringBootTest` classes sharing settings builds the context once. Real-world adjustments are the norm: the production database is replaced with an embedded H2 or Testcontainers DB, external services are stubbed with `@MockBean`, and environment specifics are overridden with `@TestPropertySource` or the `properties` attribute (`@SpringBootTest(properties = "spring.datasource.url=jdbc:h2:mem:test")`). `@Transactional` on test methods makes each test roll back, isolating mutations between tests (with the caveat that code spawning its own transactions or threads escapes the rollback). `@DirtiesContext` rebuilds the context after tests that change singleton state or add beans. `@SpringBootTest` validates that the wiring actually works — misconfigured beans, auto-configuration conflicts, and property binding failures surface here — which makes it the key check before deployment, used sparingly because of its cost.

60. What is `@WebMvcTest`?
    - **Answer:** `@WebMvcTest` is a slice test annotation that loads only the web layer of a Spring Boot application, in contrast to `@SpringBootTest` which loads the whole context. It builds a `MockMvc` environment with the specified controllers (e.g., `@WebMvcTest(ItemController.class)`), MVC infrastructure such as `HandlerMapping`/`HandlerAdapter`, Jackson message converters, `@ControllerAdvice` exception handlers, filters, and validation — but no services, repositories, security rules beyond the mock, or other application beans. External collaborators are replaced with `@MockBean` (or `@MockitoBean`) mocks so a controller test stays fast and focused on HTTP concerns: URL mapping, parameter binding, validation, serialization, status codes, and error handling:

    ```java
    @WebMvcTest(ItemController.class)
    class ItemControllerTest {
        @Autowired MockMvc mockMvc;
        @MockBean ItemService itemService;

        @Test
        void returnsCreatedItem() throws Exception {
            when(itemService.create(any())).thenReturn(new ItemDto(1L, "x"));
            mockMvc.perform(post("/api/items").contentType(APPLICATION_JSON).content("{\"name\":\"x\"}"))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").value(1));
        }
    }
    ```

    Because `@WebMvcTest` does not start the full context, tests run in a fraction of the time of `@SpringBootTest`, making them the default for controller testing. The trade-off: real services, database behavior, and security filter chains are not exercised, so security-related tests may need `@AutoConfigureMockMvc` with `@SpringBootTest` or explicit mock filters. Common use cases are verifying request validation (missing/incorrect fields produce 400 with the right error body), response serialization, path/query binding, and `@ControllerAdvice` handling. The general slice principle — load the layer under test, mock the rest — applies here: `@DataJpaTest`, `@JsonTest`, and `@DataRedisTest` are siblings that apply the same idea to other layers.

61. Difference between the traditional Java Singleton design pattern and the Spring singleton bean scope.
    - **Answer:** The GoF Singleton pattern guarantees that exactly one instance of a class exists per JVM by privatizing the constructor and exposing a static `getInstance()` that lazily or eagerly creates and caches the single instance. Spring's singleton scope is a different mechanism: it means the Spring IoC container creates and caches exactly one instance per bean definition (per bean name per `ApplicationContext`) and injects that same instance everywhere — but the class itself is an ordinary Java class with a normal constructor, so nothing prevents `new MyService()` elsewhere. The instance control lives in the container, not in the class, which is the decisive difference: with the GoF pattern the class enforces its own uniqueness and the developer cannot choose to create more instances; with Spring, uniqueness is a container-level policy that can be changed by switching to prototype scope or by simply constructing the object directly. Consequences follow from this inversion. Testability: a GoF singleton's global static state is hard to isolate and inject; a Spring singleton is just an injected dependency that can be replaced by a mock or prototype in tests. Lifecycle: Spring manages construction, dependency wiring, initialization callbacks, and destruction for its singleton, whereas a GoF singleton self-manages and typically has no container lifecycle. Scope boundaries: the GoF pattern guarantees one instance per classloader, while Spring guarantees one per `ApplicationContext`, so two contexts can each have their own singleton of the same class. Thread-safety: both share a single instance across threads, so mutable state must be guarded — a Spring singleton must be stateless or synchronized, same as any shared object. In practice, Spring applications use Spring singleton scope for beans and do not hand-roll GoF singletons; the GoF pattern remains relevant only in non-Spring utility code where a container is absent.

62. Explain in detail `@Component`, `@Bean`, `@Configuration`, `@Repository`, `@Service`, `@RestController`, and `@Controller`. Also, why are there dedicated annotations?
    - **Answer:** All these annotations ultimately register beans in the Spring container, but they fall into two families with different mechanics. `@Component`, `@Repository`, `@Service`, `@Controller`, and `@RestController` are class-level stereotype annotations discovered by component scanning; `@Configuration` plus `@Bean` is the explicit, method-level way to define beans. `@Component` is the generic stereotype marking any class as a managed bean; scanning picks it up and the container instantiates it, injecting its constructor dependencies. `@Service` and `@Repository` are specializations of `@Component` that do not change scanning behavior but carry layer semantics: `@Service` marks the business/service layer (the conventional home of `@Transactional` logic), and `@Repository` marks the persistence layer and — uniquely — enables Spring's persistence exception translation, wrapping vendor exceptions such as `SQLException` and JPA `PersistenceException` into the `DataAccessException` hierarchy via `PersistenceExceptionTranslationPostProcessor`. `@Controller` marks MVC controllers whose methods are mapped by Spring MVC and typically return view names; `@RestController` is `@Controller` + `@ResponseBody`, so every method's return value is serialized directly to JSON/XML in the response body — the standard for REST APIs. `@Configuration` marks a class as a source of bean definitions: it is CGLIB-proxied so `@Bean` methods follow singleton semantics even when called from other methods in the same class. `@Bean` is a method-level annotation inside such a class whose return value becomes a bean; it is the mechanism for registering third-party or conditionally created objects that cannot be annotated for scanning. A representative contrast:

    ```java
    @Component
    public class GreetingService { }                      // discovered by scanning

    @Configuration
    public class HttpClients {
        @Bean
        public RestTemplate restTemplate() {              // explicit construction
            return new RestTemplateBuilder().connectTimeout(Duration.ofSeconds(5)).build();
        }
    }
    ```

    Why the dedicated annotations? Technically only `@Component` is needed for scanning, but the specializations give two things: semantic clarity — the layer of every class is visible from its annotation — and layer-specific behavior, of which `@Repository`'s exception translation and `@Controller`/`@RestController`'s web handling are the concrete examples. `@Service` additionally aligns with Spring's AOP and transaction conventions (pointcuts often target `@Service`/`@Transactional` classes), and tools and documentation systems can introspect a layered architecture from the annotations. The differences that matter in practice: `@Component`/`@Service`/`@Repository` vs `@Bean` (class-level scanning vs method-level explicit registration, own classes vs third-party types), `@Configuration` vs `@Component` proxying (calls between `@Bean` methods return the cached singleton only in `@Configuration`), and `@Controller` vs `@RestController` (`@ResponseBody` applied to every method). The annotations are complementary: scanning handles the application's own classes; `@Bean` methods handle everything else.
