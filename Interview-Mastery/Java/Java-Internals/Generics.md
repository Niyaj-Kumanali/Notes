# Generics

## Overview

- **Definition** — Generics enable types (classes, interfaces, methods) to be parameterized, providing stronger type checks at compile time.
- **Why It Exists** — Before generics, collections held `Object` references, requiring explicit casts that could fail at runtime. Generics shift type safety to compile time, eliminating unchecked casts and enabling reusable, type-safe abstractions.
- **Historical Context** — Generics were introduced in Java 5 (JSR 14), inspired by C++ templates and GJ (Generic Java) by Gilad Bracha, Martin Odersky, and Philip Wadler. Erasure was chosen for backward compatibility — generic code interoperates with pre-generic libraries without recompilation.
- **Key Concepts** — **Type parameters** (`<T>`) declare formal type variables on classes/methods. **Bounded type parameters** (`<T extends Number>`) restrict allowed types. **Wildcards** (`? extends T`, `? super T`) express variance. **Type erasure** removes generic type info at runtime, replacing with bounds or `Object`. **Heap pollution** occurs when a parameterized variable references an object of a different parameterized type. **Bridge methods** are synthetic compiler inserts preserving polymorphism under erasure. **PECS** (Producer Extends, Consumer Super) guides wildcard choice. **Raw types** (e.g., `List` without type arg) bypass generics for legacy compatibility. **Reification** means a type is fully available at runtime — arrays reify, generics do not.

## Core Concepts

- **Type parameters** are placeholders declared in angle brackets (e.g., `<T>`, `<K, V>`). They are replaced by concrete type arguments at use sites.

  ```java
  public class Box<T> {
      private T value;
      public void set(T value) { this.value = value; }
      public T get() { return value; }
  }
  Box<Integer> box = new Box<>();
  ```

- **Generic methods** introduce their own type parameters before the return type, independent of class-level parameters.

  ```java
  public static <T> T identity(T arg) { return arg; }
  ```

- **Bounded type parameters** constrain the type argument to a subtype of a bound class or interface.

  ```java
  public static <T extends Comparable<T>> T max(T a, T b) {
      return a.compareTo(b) > 0 ? a : b;
  }
  ```

- **Wildcards** express variance in generic types. `? extends T` (covariant) allows reading `T` values safely. `? super T` (contravariant) allows writing `T` values safely. Unbounded `?` works when any type is acceptable.

  ```java
  void copy(List<? super T> dest, List<? extends T> src) { }
  ```

- **Type erasure** is the compiler process of removing all generic type information and replacing type parameters with their leftmost bound or `Object`. The resulting bytecode contains no generics.

- **Heap pollution** happens when a variable of a parameterized type refers to an object not of that type, typically through unchecked operations or varargs.

  ```java
  List<String>[] array = new List[10]; // unchecked warning
  String s = array[0].get(0);         // ClassCastException at runtime
  ```

- **Bridge methods** are synthetic methods inserted by the compiler to maintain polymorphism after erasure. For example, when a subclass overrides a generic method with a concrete type, the compiler generates a bridge with erased signature that delegates to the concrete method.

- **PECS principle** stands for Producer Extends, Consumer Super. If a collection provides (produces) elements, use `? extends T`. If it accepts (consumes) elements, use `? super T`. If both, use exact type.

- **Raw types** are generic classes used without type arguments (e.g., `List` instead of `List<String>`). They exist for backward compatibility with pre-Java 5 code but suppress all generic type checking, leading to unchecked warnings and potential heap pollution.

- **Generic vs array covariance** — Arrays are reified and covariant: `String[]` is a subtype of `Object[]`. Generics are invariant and erased: `List<String>` is not a subtype of `List<Object>`. Mixing both (generic array creation) is illegal because it would break type safety.

- **Reification** — A type is reified if fully available at runtime. Arrays in Java are reified (runtime knows `String[]` vs `Object[]`). Generics are not — after erasure, only raw types or bounds exist at runtime. This is why `instanceof List<String>` is a compile error.

## Common Mistakes

- **Using raw types**
  - Declaring `List list = new ArrayList();` instead of `List<String> list = new ArrayList<>();`. This suppresses all generic type checks.
  - **Why it looks correct:** Legacy code and older tutorials use raw types; they compile without error and work at runtime with unchecked casts.
  - Always parameterize generic types. Use raw types only when interfacing with pre-generic libraries.

- **Confusing ? extends with ? super**
  - Using `? extends T` for a consumer collection. You cannot add elements to a `List<? extends Number>` except `null`.
  - **Why it looks correct:** Developers new to generics assume wildcards are interchangeable, or that `extends` is always safer.
  - Apply PECS: if a method writes to a parameter, use `? super T`. If it reads, use `? extends T`.

- **Generic array creation**
  - Writing `new List<String>[10]` causes a compile error: "generic array creation".
  - **Why it looks correct:** Arrays of concrete parameterized types seem intuitive; developers do not realize arrays and generics have incompatible variance rules.
  - Use `List<List<String>>` or `ArrayList` of the generic type instead of a raw array.

- **Ignoring unchecked warnings**
  - Suppressing `@SuppressWarnings("unchecked")` without understanding the source of the warning can hide heap pollution risks.
  - **Why it looks correct:** The code compiles and runs, so developers assume the warning is harmless.
  - Investigate each unchecked warning. Refactor to eliminate it or add runtime checks (e.g., `Collections.checkedList`).

- **Varargs and heap pollution**
  - Declaring a method with generic varargs `void foo(List<String>... args)` creates a heap pollution risk because the varargs array is reified.
  - **Why it looks correct:** Varargs with generics compile with a warning; the `@SafeVarargs` annotation looks like a simple fix.
  - Use `@SafeVarargs` only when the method does not modify the varargs array. Consider `List<List<String>>` as an alternative.

## Real-World Scenarios

### Designing a type-safe event bus

- An event bus with `Class<T>` keys and `Consumer<T>` handlers requires bounded wildcards to handle event subtypes. Using `? extends T` for the event class and `? super T` for consumers ensures type safety without proliferation of unchecked casts.

  ```java
  public class EventBus {
      private Map<Class<?>, List<Consumer<?>>> handlers = new HashMap<>();
      public <T> void register(Class<T> type, Consumer<? super T> handler) {
          handlers.computeIfAbsent(type, k -> new ArrayList<>()).add(handler);
      }
      @SuppressWarnings("unchecked")
      public <T> void publish(T event) {
          Class<?> type = event.getClass();
          List<Consumer<?>> list = handlers.get(type);
          if (list != null) {
              for (Consumer<?> c : list) {
                  ((Consumer<T>) c).accept(event);
              }
          }
      }
  }
  ```

### Library API with fluent builders

- A generic builder pattern uses bounded type parameters to enforce compile-time method ordering. Each builder step returns a new subtype that restricts which methods are available next.

  ```java
  public class QueryBuilder<T extends QueryBuilder<T>> {
      public T where(String clause) { return self(); }
      public T orderBy(String col) { return self(); }
      @SuppressWarnings("unchecked")
      protected T self() { return (T) this; }
  }
  public final class FinalQueryBuilder extends QueryBuilder<FinalQueryBuilder> {
      public List<Result> execute() { }
  }
  ```

### API contract for collection transformations

- A utility method that transforms a `Map<K, V>` into a `Map<K, R>` needs both `? extends V` (producer) and careful handling of the result type. Using bounded wildcards on parameters while keeping the return type exact prevents callers from accidentally receiving raw types.

## Use Cases

Generics shine when you need type-safe, reusable abstractions over different data types — the compiler enforces contracts that would otherwise require manual casts and runtime checks.

- **Type-safe collections** — Eliminate `ClassCastException` by letting the compiler verify element types at compile time.
  - Declare `List<String>` instead of raw `List` so that adding an `Integer` is rejected immediately. Example: any JDK collection used in production code.
  - **Avoid when:** the collection must hold heterogeneous types — consider a sealed interface or a union type pattern instead.

- **Generic APIs and utility methods** — Write one method that works across many types without duplicating code.
  - Define `<T> T requireNonNull(T obj)` or `Collections.emptyList()` with type inference. Example: a JSON parser that deserializes into `<T> T fromJson(String, Class<T>)`.
  - **Avoid when:** the algorithm depends on concrete type capabilities (e.g., arithmetic) — use bounded type parameters with a specific superclass.

- **Wildcard-based API flexibility** — Accept inputs and produce outputs in a way that respects subtyping relationships.
  - Apply PECS: `? extends T` for producers (read from), `? super T` for consumers (write to). Example: `void copy(List<? extends T> src, List<? super T> dest)`.
  - **Avoid when:** the method both reads and writes the same parameter — use an exact type parameter instead.

- **Type-safe builders** — Carry compile-time type information through a fluent builder chain.
  - Parameterize the builder class so that `build()` returns the exact type without casting. Example: `HttpClientBuilder<Req, Res>` that ends with `.build()` yielding `HttpClient<Req, Res>`.
  - **Avoid when:** the builder is simple (1-2 fields) — a constructor or static factory is clearer.

## Scenario-Based Questions

**Q: Your team's library exposes a `Transformer<A, B>` interface. Users report ClassCastException when passing a `Transformer<String, Object>` where `Transformer<Object, String>` is expected. How do you fix this?**

- The issue is that generics are invariant. `Transformer<String, Object>` is not a subtype of `Transformer<Object, String>`. If the library reads from `A` and produces `B`, the input `A` should use `? extends A` (producer) and the output `B` should use `? super B` (consumer) in the API method signatures. Apply PECS to wildcard the parameters.
- **Interview follow-up:** How would you redesign the `Transformer` interface to support both covariance and contravariance in a single class?

**Q: A logging framework uses `List<Object>` internally but a client passes a `List<String>` reference to an API expecting `List<Object>`. The compiler rejects the call. Explain why and provide a solution.**

- Generics are invariant: `List<String>` is not a subtype of `List<Object>`. The fix is to use `List<? extends Object>` (which is effectively `List<?>`) in the API parameter. This accepts any `List<T>` and allows reading `Object` values. If the API also needs to write, the parameter should be `List<? super String>` instead.
- **Interview follow-up:** Why does Java not allow covariance for mutable collections, and how does this relate to the array covariance "bug"?

**Q: A developer writes `List<String>[] array = new List<String>[5];` and gets a compile error. Why does Java forbid generic array creation?**

- Generic array creation is illegal because arrays are reified (they know their component type at runtime) while generics are erased. Allowing `new List<String>[5]` would let the runtime believe the array's component type is `List<String>`, but erasure means only raw `List` exists. This could be exploited to violate type safety — for example, assigning a `List<Integer>` to an element and then reading a `String` with no ArrayStoreException.
- **Interview follow-up:** Can you construct a code example where generic array creation would cause heap pollution without a compiler error, if the JVM allowed it?

**Q: A developer uses `List<? extends Number>` as a method parameter and tries to add a new `Integer` value inside the method, but the compiler rejects it. Why?**

- The wildcard `? extends Number` makes the list a producer only — you cannot add elements (except null) because the compiler does not know the specific subtype of Number. If the actual runtime type were `List<Double>`, adding an Integer would break heap safety. To accept elements, use `? super Number` (consumer) instead, following PECS.
- **Interview follow-up:** How would you design a method that both reads from and writes to a generic collection parameter?

**Q: A team creates `Wrapper<int>` but the compiler rejects it. Why does Java not allow primitives as type arguments?**

- Type arguments must be reference types because generics work through erasure — at runtime, type parameters are replaced with `Object` or their bounds. Primitives are not subtypes of `Object`. Autoboxing to `Integer` handles assignment but does not change the generic type constraint. Project Valhalla aims to address this with value types and specialized generics.
- **Interview follow-up:** How would you implement a generic collection that avoids boxing overhead for primitives without Valhalla?

**Q: A utility method declares `<T> void sort(List<T> list, Comparator<? super T> cmp)`. A caller passes `List<Integer>` and `Comparator<Number>`. Does this compile?**

- Yes. The comparator parameter uses `? super T`, where T is inferred as Integer. `Comparator<Number>` is valid because Number is a supertype of Integer. The wildcard accepts any comparator whose type is T or a supertype of T (contravariance). This follows PECS — the comparator consumes T values to compare them.
- **Interview follow-up:** What happens if the comparator parameter is changed to `Comparator<T>` instead of `Comparator<? super T>`?

**Q: In a generic class `Box<T extends Comparable<T>>`, can you create `Box<Object>`?**

- No, because `Object` does not implement `Comparable<Object>`. The bounded type parameter `<T extends Comparable<T>>` requires that T implements the Comparable interface parameterized with itself. Only types satisfying this self-referential bound are valid as type arguments.
- **Interview follow-up:** How would you redesign the bound so that sibling subclasses can be compared with each other?

**Q: An API returns `List<List<String>>` and a caller assigns it to `List<List<?>>`. Is this assignment safe?**

- Yes, this is safe. `List<String>` is a subtype of `List<?>`, so the assignment is valid. However, through the `List<List<?>>` reference you can only read elements safely — you cannot add arbitrary `List<T>` values because the wildcard prevents insertion of unknown types. The compiler preserves type safety through capture constraints.
- **Interview follow-up:** What is the difference between `List<List<?>>` and `List<? extends List<?>>` in terms of what you can add?

**Q: A library uses `Class<T>` as a type token: `public <T> T create(Class<T> type)`. A caller passes a raw `Class` (without angle brackets). What happens?**

- Using the raw type suppresses generic type checking. The compiler emits an unchecked warning, and the method's return type is erased to `Object`. The caller needs an explicit cast to recover the desired type. Fix: always specify the type argument, e.g., `Class<MyClass>` or `Class<?>`. Type tokens rely on the caller to provide the correct reified type.
- **Interview follow-up:** How does Guice's `TypeLiteral<T>` solve the problem that `Class<T>` cannot represent parameterized types?

**Q: A developer writes `static <T> T[] toArray(List<T> list)` and tries `return (T[]) list.toArray()`. Under what circumstances does this fail at runtime?**

- The cast `(T[])` is an unchecked cast due to erasure — at runtime T is erased to `Object`, so the cast becomes `(Object[])` and the actual array is `Object[]`. If the caller assigns the result to `String[]`, a `ClassCastException` occurs at the assignment point. Use `list.toArray(T[]::new)` with an array constructor reference to create the correct reified type at runtime.
- **Interview follow-up:** How does `Arrays.copyOf` and the `Array.newInstance` reflective method help create type-safe arrays in generic code?

## Interview Questions

- **Explain how type erasure works in Java and give an example of a problem it causes.**
  - The compiler removes all generic type parameters and replaces them with their leftmost bound or `Object`. For `List<String>`, the bytecode uses raw `List` with casts inserted at call sites. This causes problems with `instanceof` checks (`list instanceof List<String>` is illegal), reflection (generic types are invisible at runtime), and overload resolution (two methods differing only by type parameter erase to the same signature).

- **What is the PECS principle and when would you use `? super T` vs `? extends T`?**
  - PECS stands for Producer Extends, Consumer Super. Use `? extends T` when you only read elements from a structure. Use `? super T` when you only write elements. For example, `Collections.copy(List<? super T> dest, List<? extends T> src)`. Using the wrong bound causes compilation errors — you cannot add anything but `null` to a `? extends` collection.

- **What is heap pollution and how can it be prevented?**
  - Heap pollution occurs when a variable of a parameterized type points to an object that is not of that parameterized type, often via unchecked operations, raw types, or generic varargs. Prevention: avoid mixing raw and parameterized types, use `@SafeVarargs` only when the method does not modify the varargs array, and prefer `List<List<T>>` over generic arrays. Enable `-Xlint:unchecked` to detect risky code.

- **How do bridge methods relate to generics?**
  - When a subclass overrides a generic method with a specific type (e.g., `class MyList implements Comparable<MyList>`), the compiler generates a synthetic bridge method with the erased signature (`compareTo(Object)`) that delegates to the concrete method. This ensures polymorphism works at the bytecode level, since the JVM dispatches based on erased signatures.

- **What is the difference between `List<?>` and `List<Object>`?**
  - `List<?>` is a homogenous list of unknown type — you can read elements as `Object` but cannot add any element (except `null`). `List<Object>` is a list that explicitly accepts any `Object` instance — you can both read and write `Object` values. `List<String>` is a subtype of `List<?>` but not of `List<Object>`.

- **What is the difference between a bounded type parameter and a wildcard?**
  - A bounded type parameter (`<T extends Number>`) declares a named type variable with an upper bound used within a class or method body. A wildcard (`? extends Number`) is an anonymous type argument used at use sites. Type parameters are used for relationships between multiple arguments or return types; wildcards express variance constraints without introducing a named type.

- **Can you use generics with enums or anonymous classes?**
  - Enums cannot declare type parameters because the compiler implicitly extends `Enum<E>`, and generics do not support the kind of self-referential inheritance enums require. Anonymous classes cannot declare type parameters either — their generic types are inferred from the parent type at creation. Workarounds include using generic interfaces that enums implement or passing type information through constructor arguments.

- **What is the impact of type erasure on method overloading?**
  - Erasure prevents overloading methods that differ only by generic type parameters. For example, `void process(List<String>)` and `void process(List<Integer>)` erase to `void process(List)`, causing a compile error. This is why generic specialization cannot use method overloading — the JVM's method dispatch is based on erased signatures.

- **How does the compiler infer type parameters for a generic method?**
  - The compiler performs type inference by examining method arguments and the expected return type (target type inference, enhanced in Java 8). It finds the smallest type satisfying all constraints. For example, `Collections.emptyList()` with `List<String> list = Collections.emptyList()` infers String from the assignment target. If inference is ambiguous, the developer must provide an explicit type witness.

- **What is the typesafe heterogeneous container pattern?**
  - Proposed by Joshua Bloch, this pattern uses `Class<T>` as keys in a `Map<Class<?>, Object>` to store and retrieve values of arbitrary types in a type-safe manner. Each value is cast to the type represented by its key. Example: `public <T> void put(Class<T> type, T instance)` and `public <T> T get(Class<T> type)`. The unchecked cast is isolated inside the container, safe if the key-value contract is maintained.

- **How does the compiler handle `@SuppressWarnings("unchecked")`?**
  - It suppresses compiler warnings for unchecked operations — situations where the compiler cannot prove type safety due to erasure. Common cases: casting raw types to parameterized types, generic varargs, and unchecked conversions. The annotation does not make the code safe; it only silences the warning. Each use should be accompanied by manual verification of type safety.

- **What is the difference between `List<? extends T>` and `List<T>` as a method parameter?**
  - `List<T>` accepts exactly `List<T>` (invariant) — you can both read and write T elements. `List<? extends T>` accepts `List` of any subtype of T (covariant) — you can only read T elements (adding anything except null is forbidden). Use `List<T>` when the method both reads and writes; use `List<? extends T>` when it only reads.

- **How do you create a generic method with two type parameters that have a mutual constraint?**
  - Mutual constraints are expressed using bounded type parameters referencing each other. Example: `<T extends Comparable<? super T>>` ensures T can compare with itself or its supertypes. Another pattern: `<A, B extends A>` enforces that B is a subtype of A. The compiler checks these bounds during type inference and rejects arguments that violate the relationship.

- **What is the capture of `?` in Java generics?**
  - When the compiler encounters a wildcard, it creates an anonymous "capture" variable representing the unknown type. For `List<?>`, the compiler internally represents it as `List<capture#1 of ?>`. This capture prevents unsafe operations — you cannot add elements because the capture type is unknown. Each wildcard usage gets its own capture variable, enforcing type safety per use site.

- **Can a class implement multiple parameterizations of the same generic interface?**
  - No. A class cannot implement both `Comparable<Person>` and `Comparable<Employee>` because both erase to `Comparable`, creating a conflict. The compiler rejects this. However, a class can implement different generic interfaces (e.g., `Consumer<Person>` and `Supplier<Employee>`) because they have different erased signatures.

- **How does Java infer types for chained generic method calls?**
  - For chained calls like `foo().bar()`, the compiler infers left-to-right. If `foo()` returns `List<T>` and `bar()` expects `List<String>`, the compiler may backtrack and re-infer T as String. Java 8+ enhanced this with target-type inference and poly expressions. If inference fails, explicit type witnesses (`<String>foo()`) resolve the ambiguity.

- **What is the difference between `? extends Object` and an unbounded `?`?**
  - They are semantically equivalent — both accept any type and allow reading as Object. However, unbounded `?` is the idiomatic form and receives special compiler optimization. They are not interchangeable in all syntactic positions: for example, `Class<?>` is preferred over `Class<? extends Object>` because the compiler can optimize unbounded wildcards more aggressively.

- **Why does `Arrays.asList(1, 2, 3)` return `List<Integer>` instead of `List<int>`?**
  - The varargs parameter `T...` accepts reference types only because of erasure and array covariance. Primitives cannot be type arguments, so Java autoboxes `1, 2, 3` to `Integer`. The inferred return type is `List<Integer>`. For efficient primitive collections, use `IntStream` or specialized libraries like Eclipse Collections.

- **What is the Curiously Recurring Template Pattern (CRTP) in Java?**
  - CRTP uses `<T extends MyClass<T>>` where a class parameterizes its base with itself. Example: `class Person implements Comparable<Person>`. This allows the base class to work with the concrete subclass type without casting — used in fluent builders, type-safe enums, and `Comparable` implementations. The pattern leverages generics to preserve subclass type information.

- **How does the compiler decide whether a cast is an unchecked cast vs a checked cast?**
  - A checked cast involves reifiable types and is verified at runtime (e.g., `(String) obj`). An unchecked cast involves non-reifiable types (e.g., `(List<String>) obj`) — the JVM cannot verify the generic portion, so only the raw component type is checked. The compiler issues an unchecked warning for these casts, signaling that type safety cannot be fully enforced at runtime.

## Developer Recommendations

- **Prefer type-safe patterns over raw types**
  - Raw types disable all generic type checking, leading to runtime failures that compile-time checks would have caught.
  - Always parameterize generic types. Use `List<String>` instead of `List`. For legacy interop, isolate the raw type usage behind a checked facade.
  - **Production story:** A team used raw `Map` throughout a caching layer. A production incident occurred when a `Map<String, Integer>` was accidentally populated with `Map<String, Config>` values, causing `ClassCastException` at the first cache hit. After converting to parameterized types and adding runtime `Collections.checkedMap`, the issue never recurred.

- **Apply PECS consistently in API design**
  - Wildcards in method signatures increase flexibility without sacrificing type safety. Leaving them out forces callers into unsafe casts or duplicate overloads.
  - For every collection parameter, decide whether it is a producer, consumer, or both. Producer gets `? extends T`, consumer gets `? super T`. Only omit wildcards when the parameter both reads and writes.
  - **Production story:** A report-generation library defined `void render(List<ReportData> data)` which could only accept `ArrayList<ReportData>`. Callers with `List<CsvReportData>` (a subtype) had to copy into a new list. Switching to `List<? extends ReportData>` eliminated all copying and halved memory usage during report generation.

- **Use @SafeVarargs judiciously**
  - Generic varargs methods always produce a compiler warning because the varargs array is reified and can lead to heap pollution. `@SafeVarargs` suppresses this warning without adding protection.
  - Only apply `@SafeVarargs` when the method body does not store elements into the varargs array or pass it to untrusted code. Prefer `List<T>` over `T...` for public APIs.
  - **Production story:** A logging framework used `@SafeVarargs` on a method that stored varargs into a shared buffer. Under concurrent load, one thread's arguments leaked into another thread's log entry, causing corrupted log output with wrong parameter values. Removing `@SafeVarargs` and switching to `List<Object>` fixed both the warning and the data race.

- **Prefer `List<T>` over `T[]` in new code**
  - Arrays are covariant and reified, which makes them incompatible with generics and prone to runtime `ArrayStoreException`. Lists are invariant, erased, and fully generic-compatible.
  - Replace `T[]` return types with `List<T>`. For internal performance-critical paths where arrays are necessary, use `@SuppressWarnings("unchecked")` on the array creation and document why it is safe.
