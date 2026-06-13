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
