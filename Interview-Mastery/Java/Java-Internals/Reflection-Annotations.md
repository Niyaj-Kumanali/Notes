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

**Q: A developer uses `Field.setAccessible(true)` to modify a private static final field in a Java 8 library class at runtime. After migrating to Java 17, the call throws `InaccessibleObjectException`. What changed and how do you fix it?**

- Java 9 introduced the module system (JPMS). By default, named modules encapsulate their packages, preventing reflective access to private members of classes in other modules. `setAccessible(true)` fails because the module `java.base` (or the library's module) does not open the affected package. Fixes: (1) Add JVM flags `--add-opens <module>/<package>=<accessing-module>`. (2) If you control the library, add an `open` or `opens` directive in `module-info.java`. (3) Consider whether the design can avoid reflective modification of private final fields.
- **Interview follow-up:** How does `--illegal-access=permit` (Java 9–16) differ from the Java 17+ behavior?

**Q: A developer creates a dynamic proxy for a `List<String>` interface and the `add(String)` method appears to work, but calling `addAll(Collection<? extends String>)` throws `ClassCastException`. Why?**

- The `InvocationHandler` is likely passing the arguments directly to the target method via `method.invoke(target, args)`. For `addAll`, the varargs argument might be passed as an `Object[]` containing a single `Collection` instead of the `Collection` itself due to incorrect argument forwarding. The fix: ensure the handler correctly forwards the method arguments array without wrapping. Also verify that `Proxy.isProxyClass()` and correct casting are used.
- **Interview follow-up:** How does the `InvocationHandler` receive arguments for methods with varargs versus methods with array parameters?

**Q: An annotation processor generates code at compile time using `AbstractProcessor`. The generated code references a type that exists at compile time but is removed from the classpath at runtime. What happens and how do you prevent this?**

- If the generated code imports a type that is absent at runtime, the JVM throws `NoClassDefFoundError` when the generated code is loaded. The annotation processor should ensure that all referenced types are either part of the standard library, provided as compile-only dependencies, or inlined as constants during code generation. Use the `Filer` API to create the generated source file with fully qualified imports, and validate availability via `Elements.getTypeElement()` during processing.
- **Interview follow-up:** How would you design an annotation processor to fail gracefully when its required types are missing?

**Q: A testing framework uses `MethodHandles.lookup()` to find and invoke `@BeforeEach` and `@AfterEach` lifecycle methods. On Java 9+, the lookup for a method in a nested class fails with `IllegalAccessException`. Why?**

- In Java 9+, `Lookup` has restricted access. A `Lookup` object obtained from one class cannot access private members of another class unless the lookup class is in the same module and has the appropriate access. For nested classes, the JVM treats them as separate classes. The fix: use `MethodHandles.privateLookupIn(targetClass, lookup)` (Java 9+) which produces a lookup with full access to `targetClass`. Alternatively, use `MethodHandles.lookup().in(nestedClass)` for access to protected/public members.
- **Interview follow-up:** How does `privateLookupIn` differ from calling `MethodHandles.lookup()` directly in each class?

**Q: A library dynamically generates classes using bytecode manipulation (ASM). The generated class implements an interface, but `instanceof` checks against that interface fail intermittently. What is the likely cause?**

- The generated class was loaded by a different classloader than the interface. In Java, a class is uniquely identified by (classloader, fully-qualified name). If the generated class is loaded by a custom classloader while the interface is loaded by the application classloader, `instanceof` returns false because the JVM sees them as different types. Fix: ensure the generated class is loaded by the same classloader that loaded the interface, typically via `proxyClassLoader.loadClass()` or by specifying the target classloader in the generation API.
- **Interview follow-up:** How does `Proxy.newProxyInstance` handle classloader resolution, and what classloader does it use for the generated proxy class?

**Q: A developer uses `Class.forName("com.example.DynamicClass")` in a plugin system where plugins are loaded from JAR files. After the plugin is unloaded, the class is still accessible. What is the memory issue?**

- `Class.forName()` loads the class into the caller's classloader, which may be the application classloader — preventing the plugin's classloader and all its classes from being garbage collected. This causes a classloader leak. Fix: use a dedicated `URLClassLoader` per plugin, load classes through that classloader, and call `close()` on the classloader when the plugin is unloaded. For class lookup, use `classLoader.loadClass("com.example.DynamicClass")` instead of `Class.forName()`.
- **Interview follow-up:** How does the JVM's class unloading mechanism detect that a classloader is no longer reachable?

**Q: A developer defines a custom annotation `@Auditable` with `@Retention(RUNTIME)` and uses `getAnnotation()` in an AOP interceptor. The interceptor returns `null` even though the annotation is present in the source code. What should they check?**

- The most common cause is applying the annotation to a method that is inherited from an interface. `getAnnotation()` on a class method does not inherit annotations from interface methods. The developer should check whether `@Auditable` is placed on the interface method or the implementation method. If on the interface, the annotation processor should use `Spring's AnnotationUtils.findAnnotation()` or explicitly walk the interface hierarchy. Also verify that the retention policy is RUNTIME, not CLASS.
- **Interview follow-up:** How does Spring's `AnnotationUtils.findAnnotation()` differ from `java.lang.Class.getAnnotation()` in terms of inherited annotation resolution?

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

- **What is the difference between `@Inherited` and `@Repeatable`?**
  - `@Inherited` causes an annotation on a superclass to be inherited by subclasses (class-level only). `@Repeatable` allows the same annotation to appear multiple times on the same element. They serve completely different purposes: one controls annotation propagation through inheritance, the other allows multiple instances of the same annotation. Both can be used together on the same annotation type.

- **How does `Proxy.newProxyInstance` generate the proxy class?**
  - `Proxy.newProxyInstance` calls `Proxy.getProxyClass` which generates a class at runtime. The generated class extends `Proxy` and implements the specified interfaces. It delegates every method call to an `InvocationHandler.invoke()` call. The class is cached (weakly) for each unique combination of interfaces. The bytecode is generated using internal sun.misc or java.lang.reflect.Proxy methods, not public ASM — the implementation is JVM-specific.

- **What is the difference between `getAnnotation()` and `getDeclaredAnnotation()`?**
  - `getAnnotation()` returns the annotation if present on the element or inherited from a superclass (for class-level `@Inherited` annotations). `getDeclaredAnnotation()` returns the annotation only if directly present on the element, ignoring inherited annotations. For non-inherited annotations, both return the same result.

- **What are type-use annotations and where can they be applied?**
  - Type-use annotations (Java 8+, `@Target(ElementType.TYPE_USE)`) can appear anywhere a type is used: generic type arguments (`List<@NonNull String>`), array levels (`String @NonNull []`), method return types, throws clauses, and `new` expressions (`new @NonNull MyClass()`). They enable compile-time type checking frameworks like Checker Framework and are retained in bytecode for runtime processing.

- **What is the `Unsafe` class and why is it considered dangerous for reflection-like operations?**
  - `sun.misc.Unsafe` provides low-level operations: direct memory access, object field offset calculation, CAS operations, and class loading without initialization. It bypasses Java's safety guarantees — it can corrupt memory, create objects without calling constructors, and break encapsulation. Its use is strongly discouraged and will be restricted in future JDK versions. `VarHandle` (Java 9+) is the safe replacement for most Unsafe-based CAS/field operations.

- **How does Spring resolve constructor parameters for `@Autowired` injection?**
  - Spring uses `Class.getDeclaredConstructors()` to find all constructors. It identifies the constructor annotated with `@Autowired` (or the default constructor if none). For each parameter, it resolves the type using `ParameterizedType` if available (to handle generic types like `List<String>`) and looks up the matching bean in the application context. Constructor injection is preferred over field injection because it enables immutable fields and easier testing.

- **What is the difference between `MethodHandle.invoke()` and `MethodHandle.invokeExact()`?**
  - `invoke()` allows type adaptation: the JIT can insert boxing, unboxing, and widening conversions to match the method type. `invokeExact()` requires an exact type match — no conversions are performed. `invokeExact()` is faster because no adaptation overhead is incurred. Use `invoke()` for convenience and `invokeExact()` in performance-critical code where the exact method type is known at the call site.

- **How does the `javax.annotation.processing.AbstractProcessor` work?**
  - An annotation processor extends `AbstractProcessor` and overrides `process()`. It is discovered via META-INF/services or the `-processor` javac flag. The `RoundEnvironment` provides elements (types, methods, fields) annotated with the requested annotation types. The processor can generate new source files, create compiler warnings/errors, and check for annotation misuse. Processors run in rounds — one round per annotation processing iteration.

- **What is the difference between `Class#isInstance()` and `instanceof`?**
  - `Class.isInstance(obj)` dynamically checks if `obj` is an instance of the class represented by the `Class` object. It is equivalent to `obj instanceof MyClass` but works with a dynamically determined class. Both perform the same runtime check. `isInstance()` is useful in reflective code where the target class is unknown at compile time, such as serialization frameworks.

- **How does `Array.newInstance()` work, and when would you use it?**
  - `Array.newInstance(Class<?> componentType, int length)` creates a new array with the specified component type at runtime. It is used when the array type is not known at compile time — for example, in a generic method that needs to create an array of type `T[]`. Since `new T[length]` is illegal due to erasure, `Array.newInstance(componentType, length)` is the reflective alternative.

- **What is a `ClassValue` and how does it relate to `ThreadLocal`?**
  - `ClassValue` is a mechanism for associating values with classes, similar to `ThreadLocal` for threads. It provides `get(Class<?>)` which lazily computes a value per class using `computeValue(Class<?>)`. It uses a `WeakHashMap` internally, allowing classes to be garbage collected. It is ideal for caching reflective metadata per class without causing classloader leaks, unlike static `ConcurrentHashMap` caches.

- **How does `java.lang.invoke.LambdaMetafactory` create lambda instances?**
  - `LambdaMetafactory.metafactory()` is the bootstrap method for `invokedynamic` call sites of lambda expressions. It receives the method handle of the lambda body, the target functional interface, and the captured arguments. It generates an inner class (or uses method handles directly) that implements the functional interface. The resulting `CallSite` is linked to the `invokedynamic` instruction, and subsequent calls use the linked method handle directly.

- **What is the difference between `@Retention(RetentionPolicy.CLASS)` and `RetentionPolicy.RUNTIME` in terms of annotation processing?**
  - CLASS-retained annotations are available in `.class` files and to compile-time annotation processors but not via runtime reflection. RUNTIME-retained annotations are also available at runtime via `getAnnotation()`. CLASS is suitable for code generation tools (Lombok, AutoValue) where runtime access is unnecessary. RUNTIME is required for frameworks like Spring and JUnit that inspect annotations reflectively at runtime.

- **How do you create a custom annotation that validates method parameters at compile time?**
  - Create the annotation with `@Target(ElementType.PARAMETER)` and `@Retention(RetentionPolicy.CLASS)`. Write an `AbstractProcessor` that implements `process()`, checks `RoundEnvironment.getElementsAnnotatedWith(YourAnnotation.class)`, and casts elements to `VariableElement`. Use `Elements` utility to verify parameter types, check enclosing method signatures, and produce `javac` error messages via `processingEnv.getMessager().printMessage(Diagnostic.Kind.ERROR, ...)`.

- **What happens when you call `Method.invoke()` on a method that throws a checked exception not declared in the calling code?**
  - The actual exception is wrapped in `InvocationTargetException`. The caller catches `InvocationTargetException` and calls `getCause()` to retrieve the original exception. The original exception is an `InvocationTargetException`'s cause and can be of any type (checked or unchecked). The caller can then rethrow the cause using exception chaining or handle it specifically. This design allows reflective invocation to bypass compile-time checked exception checking.

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
