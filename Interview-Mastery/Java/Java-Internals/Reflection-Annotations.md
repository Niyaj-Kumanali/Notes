# Reflection and Annotations

## Overview

- **Definition** — Reflection is the ability of running Java code to inspect and dynamically invoke classes, methods, fields, and constructors at runtime. Annotations are metadata attached to program elements, readable via reflection.
- **Why It Exists** — Reflection enables frameworks and libraries to operate on types without knowing them at compile time — essential for DI containers (Spring), serialization (Jackson), ORM (Hibernate), and testing (JUnit). Annotations provide declarative metadata that can be processed at compile time (APT), class-load time, or runtime.
- **Historical Context** — Reflection has been part of Java since JDK 1.1 (1997). Annotations were introduced in Java 5 (JSR 175), along with the `apt` tool. Java 6 added Pluggable Annotation Processing (JSR 269). Java 7 introduced `invokedynamic` (JSR 292). Java 8 added `@Repeatable` and type-use annotations. MethodHandles (Java 7+) and `VarHandle` (Java 9+) provide faster alternatives for many reflection use cases.
- **Key Concepts** — **Class.forName** dynamically loads a class. **getDeclaredMethods/Fields** introspects members. **setAccessible** suppresses access checks. **Method invocation** calls methods reflectively. **Dynamic proxies** create runtime implementations of interfaces. **@Retention** controls annotation lifetime. **@Target** restricts annotation placement. **@Inherited** propagates annotations to subclasses. **@Repeatable** allows multiple instances. **MethodHandles** and **invokedynamic** are faster, type-safe alternatives to reflection.

## Core Concepts

- **Class.forName** — Loads a class by its fully qualified name string. Returns the `Class<?>` object. The class's static initializer runs as a side effect. Overloads: `Class.forName(name)` loads and initializes; `Class.forName(name, initialize, classLoader)` allows deferring initialization.

  ```java
  Class<?> clazz = Class.forName("com.example.MyClass");
  ```

- **getDeclaredMethods / getDeclaredFields** — Returns `Method[]` / `Field[]` for members declared directly in the class (excluding inherited members). `getMethods()` / `getFields()` return public members including inherited ones. Each `Method` provides `getName()`, `getParameterTypes()`, `getReturnType()`, and `invoke()`.

  ```java
  for (Method m : clazz.getDeclaredMethods()) {
      System.out.println(m.getName());
  }
  ```

- **setAccessible** — Called on `AccessibleObject` (superclass of `Field`, `Method`, `Constructor`) to suppress Java language access control checks. Allows accessing private, protected, or package-private members from outside the class. In Java 9+ module system, `setAccessible` may fail with `InaccessibleObjectException` unless the module opens the package to the caller.

  ```java
  Field f = clazz.getDeclaredField("secret");
  f.setAccessible(true);
  String val = (String) f.get(instance);
  ```

- **Method invocation** — `Method.invoke(targetObject, args...)` calls the reflected method. Handles boxing of primitives and varargs expansion. Throws `InvocationTargetException` wrapping any exception thrown by the actual method. The caller must unwrap the cause to diagnose failures.

  ```java
  Method m = clazz.getMethod("compute", int.class, String.class);
  Object result = m.invoke(instance, 42, "test");
  ```

- **Dynamic proxies** — `java.lang.reflect.Proxy` generates a class at runtime that implements a specified set of interfaces. All method calls on the proxy are dispatched to an `InvocationHandler`.

  ```java
  InvocationHandler handler = (proxy, method, args) -> {
      System.out.println("Called: " + method.getName());
      return method.invoke(target, args);
  };
  MyInterface proxy = (MyInterface) Proxy.newProxyInstance(
      MyInterface.class.getClassLoader(),
      new Class<?>[] { MyInterface.class },
      handler);
  ```

- **Annotation retention policy** — `@Retention(RetentionPolicy.SOURCE)` — annotation discarded by compiler; not present in .class files (e.g., `@Override`). `@Retention(RetentionPolicy.CLASS)` — present in .class files but not available at runtime (default). `@Retention(RetentionPolicy.RUNTIME)` — available via reflection at runtime (e.g., `@Test`, `@Autowired`).

- **@Target** — Specifies where an annotation can be placed: `ElementType.TYPE` (class, interface, enum), `FIELD`, `METHOD`, `PARAMETER`, `CONSTRUCTOR`, `LOCAL_VARIABLE`, `ANNOTATION_TYPE`, `PACKAGE`, `TYPE_PARAMETER` (Java 8), `TYPE_USE` (Java 8), `MODULE` (Java 9). If absent, the annotation can be placed anywhere.

- **@Inherited** — If a superclass has an annotation marked `@Inherited`, subclasses inherit the annotation automatically. Only applies to class-level annotations. Querying the subclass for the annotation returns the superclass's instance. `@Inherited` does not work for method, field, or parameter annotations.

- **@Repeatable** — Allows the same annotation to be applied multiple times to the same element. Requires a container annotation that holds an array of the repeatable annotation. Java 8+.

  ```java
  @Repeatable(Schedules.class)
  public @interface Schedule { String time(); }
  public @interface Schedules { Schedule[] value(); }

  @Schedule(time="10:00") @Schedule(time="14:00")
  void run() { }
  ```

- **Processing annotations at runtime** — Use `getAnnotation(Class<T>)`, `getDeclaredAnnotations()`, or `getAnnotationsByType(Class<T>)` (which handles @Repeatable transparently). Frameworks scan the classpath at startup, identify annotated classes, and create metadata registries. Spring's `ClassPathScanningCandidateComponentProvider` is a production example.

  ```java
  if (clazz.isAnnotationPresent(MyAnnotation.class)) {
      MyAnnotation ann = clazz.getAnnotation(MyAnnotation.class);
  }
  ```

- **Performance overhead** — Reflection is 50–100x slower than direct method calls. Sources of overhead: dynamic dispatch, boxing/unboxing of primitives, array allocation for varargs, access checks, and JIT deoptimization (the JIT may not inline or compile reflectively-called methods as aggressively). Calling `setAccessible` eliminates access checks but adds the cost of the native method itself.

- **MethodHandles** — `java.lang.invoke.MethodHandle` provides a typed, directly executable reference to a method, field, or constructor. Lookup via `MethodHandles.lookup()`. MethodHandles can be faster than reflection because the JIT can inline them better — they are lower-level, with no access checks at invocation time (performed at lookup time).

  ```java
  MethodHandle handle = MethodHandles.lookup()
      .findVirtual(String.class, "length", MethodType.methodType(int.class));
  int len = (int) handle.invoke("hello");
  ```

- **invokedynamic** — A bytecode instruction (JSR 292, Java 7+) that defers method call site resolution to a bootstrap method. The bootstrap method returns a `CallSite` containing a `MethodHandle`. Used by Java for lambda expressions, string concatenation, and dynamic languages. invokedynamic can be faster than reflection because the JIT can inline and optimize the resulting call site.

## Common Mistakes

- **Using reflection in hot paths**
  - Calling `Method.invoke()` inside a tight loop, such as a serialization hot path or a request dispatcher, causing 10x–100x slowdown compared to direct calls.
  - **Why it looks correct:** Reflection makes code generic and DRY — one reflection-based dispatcher replaces many if/else blocks. Developers do not measure the performance impact.
  - Cache the `Method` object (still slower than direct calls but better than re-lookup). Better: use `MethodHandle` which the JIT can inline. Best: generate bytecode or use `invokedynamic` to achieve near-direct-call performance.

- **Calling setAccessible without understanding Java module system**
  - Reflective access to private members of classes in other JARs fails with `InaccessibleObjectException` on Java 9+ because modules encapsulate their packages.
  - **Why it looks correct:** The code compiles and works in Java 8. The developer assumes `setAccessible(true)` is the universal access key.
  - Add `--add-opens` JVM flags for internal frameworks, or ensure libraries are modularized with `opens` directives in `module-info.java`. For new code, avoid deep reflection into third-party libraries.

- **Forgetting to handle InvocationTargetException**
  - Catching `Exception` around `Method.invoke()` and losing the root cause. `InvocationTargetException` wraps the original exception thrown by the reflected method.
  - **Why it looks correct:** The compiler forces a catch of `IllegalAccessException` and `InvocationTargetException`. Developers catch `Exception` broadly to silence both and do not unwrap.
  - Always catch `InvocationTargetException` separately and call `getCause()` to rethrow or log the original exception. Never swallow the cause.

- **Leaking Class references in long-lived caches**
  - Storing `Class<?>` references or `Method` objects in static collections without clearing them. This prevents class unloading and causes metaspace leaks in container environments.
  - **Why it looks correct:** Caches improve reflection performance. Developers add them during optimization without considering classloader lifecycle.
  - Use `WeakHashMap<Class<?>, ...>` or `ClassValue` for reflective caches tied to class identity. Clear caches in undeploy hooks for container-based deployments.

- **Overusing dynamic proxies for simple delegation**
  - Creating a `Proxy` instance with an `InvocationHandler` when a simple lambda, decorator class, or interface default method would suffice. The proxy adds overhead for method dispatch and object allocation.
  - **Why it looks correct:** Dynamic proxies look elegant and can wrap any interface. They are overused in learning examples and overly-engineered code.
  - Use dynamic proxies only when the set of interfaces to implement is unknown at compile time. For fixed interfaces, prefer static delegation or default methods.

## Real-World Scenarios

### Dependency injection in a custom framework

- A lightweight DI framework scans a package for classes annotated with `@Component`, creates instances via `Constructor.newInstance()`, and injects `@Inject`-annotated fields via `Field.set()`. During startup, the framework processes 2000 classes.

  ```java
  Set<Class<?>> components = scanner.findAnnotated(Component.class);
  for (Class<?> clazz : components) {
      Object instance = clazz.getDeclaredConstructor().newInstance();
      for (Field field : clazz.getDeclaredFields()) {
          if (field.isAnnotationPresent(Inject.class)) {
              field.setAccessible(true);
              Object dependency = container.get(field.getType());
              field.set(instance, dependency);
          }
      }
      container.register(clazz, instance);
  }
  ```

- The framework caches `Constructor` and `Field` objects per class to avoid repeated lookups. With 2000 classes, startup time is under 500ms in production.

### Dynamic proxy for method-level authorization

- A service layer uses `Proxy.newProxyInstance` to wrap service interfaces with an `InvocationHandler` that checks authorization before each method call.

  ```java
  public class AuthorizationHandler implements InvocationHandler {
      private final Object target;
      private final String role;

      @Override
      public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
          if (method.isAnnotationPresent(RequiresAdmin.class) && !"ADMIN".equals(role)) {
              throw new SecurityException("Access denied");
          }
          return method.invoke(target, args);
      }
  }
  ```

- The handler reads the method's declared annotations at invocation time. Caching the annotation check result per method reduces overhead to negligible levels.

### Runtime schema generation with annotations

- An ORM framework processes `@Table` and `@Column` annotations on entity classes to generate SQL schema DDL and map result sets. At startup, it reads `@Table(tableName)` from the class, `@Column(columnName, type)` from each field, and builds a metadata model.

  ```java
  for (Field field : clazz.getDeclaredFields()) {
      Column col = field.getAnnotation(Column.class);
      if (col != null) {
          schema.addColumn(col.name(), col.type(), field.getName());
      }
  }
  ```

- This approach adds a one-time startup cost proportional to the number of entity classes (typically under 100ms for 500 entities). The generated metadata is then used in the hot path (result set mapping) via direct code generation rather than reflection.

## Scenario-Based Questions

**Q: A Spring Boot application takes 30 seconds to start. Profiling shows most of the time is in reflection-based classpath scanning. How can you reduce it?**

- The startup delay is caused by `ClassPathScanningCandidateComponentProvider` scanning the classpath for `@Component`, `@Service`, `@Repository`, etc. Solutions: (1) Use `spring-context-indexer` (adds `@Indexed` annotation and generates a compile-time index). (2) Specify explicit packages in `@ComponentScan` to limit scan scope. (3) Use Spring Boot 3.x's automatic configuration with AOT compilation. (4) Replace runtime DI scanning with compile-time annotation processing (e.g., Dagger, Micronaut, Quarkus).
- **Interview follow-up:** How does Spring's `ClassPathScanningCandidateComponentProvider` use reflection internally, and how does the compile-time index approach differ?

**Q: A developer writes a generic JSON serializer using reflection. When serializing 100,000 objects, the throughput is 10x lower than Jackson. What should they change?**

- The reflection-based serializer re-looks up fields for each object or does not cache `Field` objects. Performance loss comes from repeated `getDeclaredFields()`, `setAccessible()`, and per-call `Field.get()`. Fix: (1) Cache the `Field[]` per class and `setAccessible(true)` once. (2) Use `MethodHandles` via `lookup().unreflectGetter(field)` for each field, which the JIT can inline. (3) Generate bytecode at runtime using ASM to produce direct access code, which is exactly what Jackson does internally.
- **Interview follow-up:** What is the performance difference between `Field.get()`, `MethodHandle.invoke()`, and a generated accessor class, and why?

**Q: A library uses `Package.getPackage("com.example")` at runtime to check the `@javax.xml.ws.ServiceMode` annotation. In Java 9+, it returns `null`. Why?**

- In Java 9+, `Package.getPackage()` may return `null` for packages in named modules because the method only searches the caller's class loader for packages. The module system no longer automatically exposes all packages. Fix: use `Class.forName("com.example.SomeKnownClass").getPackage()` to get the `Package` object via a known class, then check the annotation. Better: avoid package-level annotations altogether.
- **Interview follow-up:** What other reflection APIs changed behavior in the Java 9+ module system?

## Interview Questions

- **What is the difference between `getMethods()` and `getDeclaredMethods()`?**
  - `getMethods()` returns all public methods, including inherited public methods from superclasses and interfaces. `getDeclaredMethods()` returns all methods declared directly in the class (public, protected, default, private) but excludes inherited methods.

- **How do dynamic proxies work and what are their limitations?**
  - `Proxy.newProxyInstance()` creates a synthetic class implementing the specified interfaces. Each method call on the proxy is dispatched to the `InvocationHandler.invoke()` method. Limitations: only works with interfaces (not classes), incurs overhead for each method call, and creates a new class in the JVM's generated class namespace. For class-based proxying, use `cglib` or `ByteBuddy`.

- **What is the difference between `@Retention(CLASS)` and `@Retention(RUNTIME)`?**
  - `@Retention(CLASS)` means the annotation is present in the `.class` file but is not loaded by the JVM at runtime — it cannot be read via reflection. `@Retention(RUNTIME)` means the annotation is available at runtime via `getAnnotation()` or `getDeclaredAnnotations()`. RUNTIME is required for frameworks that process annotations reflectively; CLASS is useful for compile-time annotation processors that do not need runtime access.

- **Explain how `invokedynamic` is used in lambda expressions.**
  - When the compiler encounters a lambda expression, it generates an `invokedynamic` call site with a bootstrap method (`LambdaMetafactory.metafactory()`) instead of creating an anonymous inner class. At runtime, the bootstrap method determines the target functional interface and returns a `CallSite` containing a `MethodHandle` that points to the lambda body. This decouples lambda generation from the bytecode format — the JVM can later optimize the linkage without recompiling the caller.

- **What is a `MethodHandle` and how does it differ from `java.lang.reflect.Method`?**
  - A `MethodHandle` is a typed, directly executable reference to a method, field, or constructor. Unlike `Method`, it performs access checks at lookup time (not invocation time). It is faster because the JIT can more easily inline the call — `MethodHandle` is a low-level pointer that can be intrinsified.

## Developer Recommendations

- **Cache reflective metadata, never re-lookup**
  - Repeated calls to `getDeclaredMethods()`, `getAnnotation()`, or `getMethod()` create new objects and traverse the class's metadata each time. In a hot path, this multiplies overhead.
  - Use `ConcurrentHashMap<Class<?>, List<Field>>` or `ClassValue` to cache looked-up members once per class. Set accessible flags at cache time, not per invocation. For MethodHandles, cache the `MethodHandle` or its unreflected variant.
  - **Production story:** A serialization library looked up `Field` objects for every object serialized, causing a 3-second startup and 5x throughput loss compared to a cached version. Switching to `ClassValue<Field[]>` eliminated the per-call lookup and reduced serialization time from 20us to 2us per object.

- **Prefer MethodHandles over java.lang.reflect.Method in new code**
  - MethodHandles are faster, typed, and provide better JIT optimization opportunities. They also integrate naturally with `invokedynamic` for lazy call site resolution.
  - Use `MethodHandles.lookup().unreflectGetter(field)` to create a getter handle, `unreflectSetter(field)` for a setter, and `unreflect(method)` for a method. Compare performance using JMH before adopting widely.
  - **Production story:** A real-time data pipeline used reflection to call user-defined transformation functions. Migrating from `Method.invoke()` to `MethodHandle.invoke()` reduced dispatch latency from 500ns to 80ns per call, enabling a 3x increase in throughput without additional hardware.

- **Use annotations for configuration, not behavior**
  - Annotations are metadata. Storing complex logic or state within annotation processing code creates systems that are hard to debug and reason about. Over-processing annotations at runtime also adds startup cost.
  - Keep annotation definitions minimal — a name and a few primitive values. Move complex processing to separate handler classes referenced by the annotation. Validate annotation usage at compile time via `AbstractProcessor` rather than waiting for runtime errors.
  - **Production story:** A framework used `@Retry(value=3, backoff=500, exceptions=...)` with the retry logic embedded in annotation processing. When a production issue required conditional retry (skip retry for certain errors), the team had to extend the annotation syntax and add branching logic inside reflection-based code. Moving the retry policy to a `RetryPolicy` class referenced by the annotation made the system testable and flexible.

- **Avoid reflection where compile-time code generation suffices**
  - Many reflection use cases — serialization, object mapping, proxy generation — can be moved to compile time using annotation processing (`AbstractProcessor`) or code generation (Annotation Processor + JavaPoet).
  - Evaluate whether a compile-time approach (Jakarta EE, Dagger, MapStruct, AutoValue) meets the requirement before writing runtime reflection logic. Compile-time generation yields faster startup, smaller footprint, and innate type safety.
  - **Production story:** A team replaced a runtime JSON serializer using reflection with MapStruct generated mappers. Startup time dropped from 12 seconds to 1.5 seconds, and throughput improved by 8x. The compile-time validation caught five mismatched field types that previously caused runtime errors in production.
