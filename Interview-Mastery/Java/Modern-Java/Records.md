# Records

## Overview

- **Definition** — A record is a transparent data carrier class that declares components in its header and has the compiler generate the constructor, accessors, equals, hashCode, and toString.
- **Why It Exists** — To eliminate the boilerplate of writing POJOs with manual constructors, getters, equals, hashCode, and toString while also providing semantic transparency that the class is "just the data."
- **Historical Context** — Inspired by algebraic data types in functional languages (Scala case classes, Kotlin data classes) and introduced in JDK 16 as a final feature after two preview rounds (JDK 14, JDK 15).
- **Key Concepts** — **Canonical constructor** matches component list exactly; **compact constructor** omits parameters and allows validation before implicit assignment; **custom constructor** must delegate via `this(...)`; **accessor methods** use component names (e.g. `point.x()`) not JavaBean `getX()`; **equals/hashCode/toString** are auto-derived from all components; **cannot extend** any class (records implicitly extend `java.lang.Record`); **cannot declare instance fields** beyond the component list; **local records** defined inside methods for intermediate grouping; **records as map keys** work correctly because of structural equality.

## Core Concepts

- Declaration syntax: `record Point(int x, int y) { }`
- Canonical constructor is generated automatically and assigns all component parameters to the corresponding component fields.
- Compact constructor lets you add validation or normalization without repeating the parameter list:

  ```java
  public record Range(int start, int end) {
    public Range {
      if (start > end) throw new IllegalArgumentException("start must be <= end");
    }
  }
  ```

- Custom constructor must delegate to the canonical constructor via `this(...)`.
- Accessor methods are named after the component (e.g. `point.x()`) and return the component value; they are not JavaBean-style getters.
- `equals` compares all components structurally; two records are equal if every component is equal.
- `hashCode` is derived from all components using a standard combiner.
- `toString` returns a concise representation such as `Point[x=1, y=2]`.
- Records implicitly extend `java.lang.Record` and cannot extend any other class.
- No instance fields can be added beyond the declared components.

  ```java
  // COMPILE ERROR: instance field not allowed
  public record Person(String name) {
    private int age; // illegal
  }
  ```

- Records can implement interfaces, define static fields, static methods, and instance methods that use the components.
- Local records can be declared inside a method, constructor, or initializer block, making them useful for intermediate data transformations without polluting the namespace.

  ```java
  public List<String> process(List<Item> items) {
    record Group(String key, List<Item> items) { }
    // use Group locally
  }
  ```

- Records work as map keys because equals and hashCode are reliably derived from all components.
- JPA/Hibernate no-arg constructor workaround: use Lombok's `@Builder` or define a custom constructor that provides default values for all components. Alternatively, use a fake no-arg constructor with sentinel values.

  ```java
  // Workaround for Hibernate
  public record Person(Long id, String name) {
    public Person { Objects.requireNonNull(name); }
    // Hibernate can call the canonical constructor reflectively with defaults
  }
  ```

## Common Mistakes

- **Adding instance fields to a record**
  - Declaring non-component instance fields results in a compile error.
  - **Why it looks correct:** In a regular class you can add any field; the record syntax looks class-like.
  - The fix: add the value as a component in the record header, or compute it in an accessor method from the existing components.

- **Expecting JavaBean-style getters**
  - Calling `getX()` on a record component does not compile unless explicitly defined.
  - **Why it looks correct:** Frameworks (JPA, JSON libraries) often expect `getX()` conventions.
  - The fix: configure your serialization framework to use the record's accessor methods, or add explicit JavaBean-style methods to the record body.

- **Using records with JPA without a no-arg constructor**
  - Hibernate requires a no-arg constructor for entity instantiation; records only have a canonical constructor.
  - **Why it looks correct:** Records look like regular classes and many developers assume JPA works with them out of the box.
  - The fix: use a persistence-friendly pattern such as a DTO record separate from the entity, or rely on Hibernate 6+ support for records with reflection-based instantiation.

- **Assuming records are mutable**
  - All record components are `private final` fields with no setters.
  - **Why it looks correct:** Regular POJOs typically have setters; records look similar at declaration.
  - The fix: if mutation is needed, use a regular class or create a new record instance with `with` semantics (manually or via a `withX` method).

- **Extending a record**
  - Records cannot be extended because they are implicitly final.
  - **Why it looks correct:** Regular classes can be extended; records are declared with `record` but inheritance is not obvious.
  - The fix: use composition or interfaces to share behavior between record types.

## Real-World Scenarios

### DTO in a REST API

- Replace POJO DTOs with records for request and response bodies.

  ```java
  public record CreateUserRequest(String name, String email) { }
  public record UserResponse(long id, String name, String email) { }
  ```

- Records give free equals and toString for logging and debugging.
- Serialization frameworks (Jackson, Gson) support records natively since Jackson 2.12 and Gson 2.9.

### Map Key in an In-Memory Cache

- Use a record to represent a composite key in a `HashMap` or `ConcurrentHashMap`.

  ```java
  record CacheKey(String region, String id) { }
  Map<CacheKey, Data> cache = new ConcurrentHashMap<>();
  ```

- Structural equals/hashCode ensure correct lookups without manual implementation.

### Intermediate Data Grouping

- Use local records inside a method to group data temporarily without creating top-level types.

  ```java
  public Map<String, List<Order>> groupByRegion(List<Order> orders) {
    record RegionTotal(String region, BigDecimal total) { }
    // map orders to RegionTotal, then collect
  }
  ```

## Scenario-Based Questions

**Q: You are migrating a large codebase from traditional POJOs to records. A POJO called `Address` is used as a JPA entity and also as a DTO. How would you handle the migration?**

- Split `Address` into a JPA entity (remain a regular class) and a `record AddressDto(...)` for the DTO. Records are not suitable as JPA entities due to the no-arg constructor requirement and immutability, but they are ideal for DTOs.
- **Interview follow-up:** How would you handle bidirectional relationships if you used records for JPA entities?

**Q: Why does `record Point(int x, int y) { }` provide a proper equals/hashCode but a regular POJO with the same fields does not?**

- The compiler generates equals and hashCode automatically based on all components in the record header. A regular POJO must manually override Object.equals and Object.hashCode, which is error-prone and often forgotten.
- **Interview follow-up:** If you add an instance helper method to a record that returns a derived value, should that value be part of equals/hashCode?

## Interview Questions

- **What is the difference between a canonical constructor and a compact constructor in a record?**
  - The canonical constructor has a parameter list that exactly matches the record components and assigns each parameter to the corresponding field. A compact constructor omits the parameter list and the implicit field assignment; you only write validation or normalization logic. At the end of the compact constructor body, the compiler inserts the field assignments.

- **Can a record be extended by another class or record?**
  - No. Every record is implicitly final and extends `java.lang.Record`. Java does not allow multiple inheritance, so a record cannot extend any other class.

- **What are local records and when would you use them?**
  - Local records are records declared inside a method, constructor, or initializer block. They are useful for intermediate data grouping within a single method, reducing namespace pollution and keeping related data together.

- **How do records work with serialization?**
  - Records have special serialization behavior: the serialized form is derived from the component list and the canonical constructor is used for deserialization. Custom `writeObject`/`readObject` methods are not allowed. This makes serialization safer and more predictable compared to regular classes.

## Developer Recommendations

- **Use records for DTOs and API payloads**
  - The compiler generates boilerplate code, reducing bugs and maintenance effort.
  - Declare the record with the same fields as the API contract; for JSON binding, configure Jackson or Gson to use the record's accessor methods.

- **Avoid records for JPA entities**
  - JPA relies on no-arg constructors, proxies, and mutable state, all of which conflict with record semantics.
  - Keep entities as regular classes and use records for projection DTOs returned from repository queries.

- **Prefer records as map keys over custom key classes**
  - The structural equals and hashCode guarantee correct behavior without manual implementation.
  - Define a small record type for the composite key and use it directly in `HashMap` or `ConcurrentHashMap`.

- **Use compact constructors for validation**
  - The compact constructor syntax reduces duplication while ensuring every constructed instance satisfies invariants.
  - Add validation logic inside the compact constructor body and let the compiler handle field assignment automatically.
  - **Production story:** A team using compact constructors for value objects caught invalid data at construction time instead of relying on downstream validation, eliminating a class of runtime bugs.
