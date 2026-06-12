# Spring Data JPA

---

## What is Spring Data JPA?

**Spring Data JPA** is a framework that dramatically simplifies data access by automatically generating repository implementations at runtime. It builds on top of **JPA (Jakarta Persistence)** and **Hibernate**, providing a consistent, declarative approach to database access without writing boilerplate code.

### Repository Hierarchy

- **`Repository<T, ID>`** — The marker interface at the top of the hierarchy. It is a central interface that Spring Data uses to identify repositories and enable scanning.
- **`CrudRepository<T, ID>`** — Provides basic CRUD operations: `save()`, `findById()`, `findAll()`, `count()`, `deleteById()`, `existsById()`. This is the minimum interface for most use cases.
- **`PagingAndSortingRepository<T, ID>`** — Extends `CrudRepository` with pagination and sorting via `findAll(Pageable)` and `findAll(Sort)`. Use when you need paginated data access.
- **`JpaRepository<T, ID>`** — Extends `PagingAndSortingRepository` with JPA-specific methods like `flush()`, `saveAndFlush()`, and batch operations. This is the recommended interface for most Spring Boot applications.

### Query Derivation from Method Names

Spring Data JPA parses method names to automatically generate JPQL queries. The method name follows a strict pattern:

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // SELECT u FROM User u WHERE u.email = ?1
    Optional<User> findByEmail(String email);

    // WHERE u.lastName = ?1 AND u.firstName = ?2
    List<User> findByLastNameAndFirstName(String lastName, String firstName);

    // WHERE u.active = true ORDER BY u.createdAt DESC
    List<User> findByActiveTrueOrderByCreatedAtDesc();

    // WHERE u.createdAt > ?1
    List<User> findByCreatedAtAfter(LocalDateTime date);

    // WHERE u.email LIKE %?1%
    List<User> findByEmailContaining(String partial);

    // WHERE u.department IN ?1
    List<User> findByDepartmentIn(List<String> departments);

    // Paginated query
    Page<User> findByActive(boolean active, Pageable pageable);

    // Lazy pagination (no count query)
    Slice<User> findByRole(String role, Pageable pageable);
}
```

### `@Query` — Custom JPQL and Native Queries

When method naming is insufficient, use `@Query` for explicit JPQL or native SQL:

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("SELECT o FROM Order o WHERE o.status = :status ORDER BY o.createdAt DESC")
    List<Order> findByStatus(@Param("status") OrderStatus status);

    @Query(value = "SELECT * FROM orders WHERE total > :minTotal",
           nativeQuery = true)
    List<Order> findLargeOrders(@Param("minTotal") BigDecimal minTotal);

    @Modifying
    @Query("UPDATE Order o SET o.status = :status WHERE o.createdAt < :date")
    int bulkUpdateStatus(@Param("status") OrderStatus status,
                         @Param("date") LocalDateTime date);
}
```

---

## Core Concepts

### Entity State Transitions

Understanding entity states is critical for correct JPA usage:

```
NEW (transient)          — not persisted, no ID assigned
    ↓ persist() / merge()
MANAGED (persistent)     — associated with PersistenceContext
    ↓ close() / clear() / evict()
DETACHED                 — not associated, has ID
    ↓ merge()
MANAGED (persistent)
    ↓ remove()
REMOVED (deleted)
```

- **NEW** — Entity created with `new`, but not yet known to JPA. No database identity assigned.
- **MANAGED** — Entity is associated with a persistence context. Changes are automatically tracked and synchronized to the database on flush.
- **DETACHED** — Entity was once managed but the persistence context has closed. JPA no longer tracks changes, and you must call `merge()` to re-attach.
- **REMOVED** — Entity marked for deletion. Removed from the database on the next flush or transaction commit.

### Page vs Slice

- **`Page<T>`** — Includes total count (executes a COUNT query). Best when you need total pages for navigation UI with page numbers.
- **`Slice<T>`** — No COUNT query. Only knows if there is a next page via `hasNext()`. Significantly better performance for infinite scroll patterns on large datasets.

### N+1 Query Problem

The N+1 problem occurs when you fetch entities and then access their lazy-loaded associations in a loop:

```java
// BAD: N+1 queries — 1 for customers + N for each customer's orders
List<Customer> customers = customerRepository.findAll();
for (Customer c : customers) {
    System.out.println(c.getOrders().size()); // N extra queries!
}

// GOOD: JOIN FETCH
@Query("SELECT DISTINCT c FROM Customer c JOIN FETCH c.orders")
List<Customer> findAllWithOrders();

// GOOD: Entity Graph
@EntityGraph(attributePaths = {"orders"})
@Query("SELECT c FROM Customer c")
List<Customer> findAllWithOrders();
```

### Auditing

Spring Data JPA provides automatic auditing of `createdAt`, `updatedAt`, `createdBy`, and `updatedBy`:

```java
@EntityListeners(AuditingEntityListener.class)
@MappedSuperclass
public abstract class Auditable {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(updatable = false)
    private String createdBy;

    @LastModifiedBy
    private String updatedBy;
}

@Configuration
@EnableJpaAuditing
public class JpaConfig {
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(
            SecurityContextHolder.getContext().getAuthentication())
            .map(auth -> auth.getName())
            .orElse("SYSTEM");
    }
}
```

### Specifications (Dynamic Queries)

The `Specification` pattern enables building dynamic queries programmatically:

```java
public class UserSpecifications {

    public static Specification<User> hasEmail(String email) {
        return (root, query, cb) ->
            email == null ? null : cb.equal(root.get("email"), email);
    }

    public static Specification<User> activeOnly() {
        return (root, query, cb) -> cb.isTrue(root.get("active"));
    }

    public static Specification<User> createdBetween(LocalDate from, LocalDate to) {
        return (root, query, cb) -> {
            if (from == null && to == null) return null;
            if (from == null) return cb.lessThan(root.get("createdAt"), to.atStartOfDay());
            if (to == null) return cb.greaterThan(root.get("createdAt"), from.atStartOfDay());
            return cb.between(root.get("createdAt"), from.atStartOfDay(), to.atStartOfDay());
        };
    }
}

// Usage
Specification<User> spec = Specification
    .where(UserSpecifications.hasEmail(request.getEmail()))
    .and(UserSpecifications.activeOnly());
Page<User> users = userRepository.findAll(spec, pageable);
```

### Projections (DTO, not Entity)

Avoid loading full entities when you only need a subset of fields:

```java
// Interface projection — no full entity load
public interface UserSummary {
    Long getId();
    String getEmail();
    String getFullName();
}

// Class projection with JPQL
@Query("SELECT new com.example.UserDto(u.id, u.email, u.name) FROM User u")
List<UserDto> findAllDto();
```

---

## Common Mistakes

- **N+1 queries** — Fetching entities in a loop causes N extra SQL queries, turning fast operations into slow ones. This *looks correct* because each individual query succeeds and returns data — the performance problem only manifests as the dataset grows, and without SQL logging enabled, the extra queries are invisible. Fix with `JOIN FETCH` or `@EntityGraph` to eagerly load associations in a single query.
- **`LazyInitializationException`** — Accessing a lazy-loaded association outside a transaction throws this exception. This *looks correct* because the entity reference contains the correct ID, and the getter method exists — the code reads naturally, and the error only fires when the getter is actually called outside the session. Fix by loading eagerly within the transaction, using `JOIN FETCH`, or switching to DTO projections.
- **Not using `@Transactional` on modifying queries** — Without it, lazy loading fails and changes may not be flushed to the database. This *looks correct* because `save()` returns immediately without errors — the data may eventually be flushed by OSIV or by a subsequent transactional method, masking the missing annotation. Always add `@Transactional` on service methods that modify data.
- **Using entities as DTOs** — Over-fetching data, circular JSON references during serialization, and performance issues. This *looks correct* because returning an entity directly is the path of least resistance — the data is correct during development, and the performance impact is only visible under load. Use DTO projections instead to select only the fields you need.
- **`CascadeType.ALL` everywhere** — Unintended cascading deletes can wipe out large parts of the database accidentally. This *looks correct* because `CascadeType.ALL` is the quickest way to make related entities persist, and during development with minimal data, cascading never causes visible problems. Be explicit about which cascade types you actually need (`PERSIST`, `MERGE`).
- **`FetchType.EAGER` on associations** — Causes Cartesian product joins that fetch massive amounts of data even when not needed. This *looks correct* because EAGER loading eliminates `LazyInitializationException`, and for small datasets the performance impact is negligible — the problem only grows as the entity graph deepens. Use `LAZY` as default and `@EntityGraph` for specific queries.
- **Not specifying `@Column`** — Results in unexpected column names and default lengths that may not match your schema. This *looks correct* because Hibernate generates a schema that works, and the application runs without errors — the mismatch only surfaces when a DBA reviews the production schema or a migration script fails. Always explicitly define column names and constraints.
- **`equals()` and `hashCode()` based on database ID** — Objects lose identity before persistence (ID is null for new entities). This *looks correct* because the natural choice for equality is the primary key — and it works correctly once the entity is persisted. The bug only manifests before persisting, when two new entities compare as equal despite being different objects. Use a business key or UUID that is stable across the entity lifecycle.

---

## Real-World Scenarios

### Scenario 1: E-Commerce Order Listing with N+1 Performance Crisis

An e-commerce product listing endpoint returns orders with their line items. As traffic grows, the endpoint slows down exponentially. Each order triggers N additional SQL queries for line items.

```java
// Before: implicit N+1 — 1 query for orders + N for items
@GetMapping("/orders")
public List<Order> getOrders() {
    return orderRepository.findAll(); // Triggers N+1 SQL queries
}

// After: explicit JOIN FETCH eliminates N+1
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items JOIN FETCH o.customer")
List<Order> findAllWithDetails();

// Alternative: EntityGraph for dynamic fetch plans
@EntityGraph(attributePaths = {"items", "customer"})
@Query("SELECT o FROM Order o")
List<Order> findAllWithGraph();
```

   The fix reduced page load time from 8s to 200ms for an order with 50 items.

   > **Interview follow-up:** The candidate used `JOIN FETCH` for two associations (items, customer). If `Order` has 20 fields and 8 associations, joining all of them creates a Cartesian product — 100 orders × 50 items × 1 customer = 5000 rows in the result set. How would you balance the number of JOIN FETCHes against the row explosion, and when would you switch to batch fetching with `@BatchSize` instead?

### Scenario 2: Multi-Tenant SaaS with Dynamic Query Filters

A SaaS dashboard allows admins to filter users by any combination of 15 fields (status, role, registration date, plan type, etc.). Writing 2^15 derived query methods is impossible.

```java
public class UserFilterSpecification {
    public static Specification<User> build(UserFilterRequest filter) {
        Specification<User> spec = Specification.where(null);
        if (filter.getStatus() != null)
            spec = spec.and((root, q, cb) -> cb.equal(root.get("status"), filter.getStatus()));
        if (filter.getRole() != null)
            spec = spec.and((root, q, cb) -> cb.equal(root.get("role"), filter.getRole()));
        if (filter.getRegisteredAfter() != null)
            spec = spec.and((root, q, cb) ->
                cb.greaterThan(root.get("createdAt"), filter.getRegisteredAfter()));
        if (filter.getPlanType() != null)
            spec = spec.and((root, q, cb) -> cb.equal(root.get("plan").get("type"), filter.getPlanType()));
        return spec;
    }
}

// Usage with pagination
Page<User> users = userRepository.findAll(
    UserFilterSpecification.build(filter), pageable);
```

### Scenario 3: Reporting Module with Read-Optimized DTO Projections

A reporting module needs to display a table of 100K+ orders with only 5 fields (ID, date, customer name, total, status). Loading full entities causes massive memory pressure and slow serialization.

```java
// Interface projection — no full entity load, no lazy-loading surprises
public interface OrderReportRow {
    Long getId();
    LocalDateTime getOrderDate();
    String getCustomerName();
    BigDecimal getTotal();
    String getStatus();
}

@Query("SELECT o.id AS id, o.orderDate AS orderDate, c.name AS customerName, " +
       "o.total AS total, o.status AS status " +
       "FROM Order o JOIN o.customer c ORDER BY o.orderDate DESC")
Page<OrderReportRow> findReportRows(Pageable pageable);
```

The projection approach used 15MB heap instead of 200MB with full entities.

---

## Scenario-Based Questions

1. **Q: You notice that a simple `findAll()` on an `Order` entity with 20 `@OneToMany` associations takes 30 seconds and generates 150+ SQL queries. No exceptions, but the production database CPU is at 100%. How do you fix this?**
   A: This is a severe N+1 problem. Start by identifying which associations trigger the extra queries (enable `spring.jpa.show-sql=true`). Use `@EntityGraph(attributePaths = {...})` on specific repository methods to eagerly fetch only the associations needed for that query. For list endpoints, use DTO projections to avoid loading full entities. Set `FetchType.LAZY` on all associations as the default — never use `FetchType.EAGER` on collections. For the most critical queries, use `JOIN FETCH` in JPQL, but be careful of Cartesian product explosions when joining multiple collections.

2. **Q: Your team runs a bulk update that sets `status = 'archived'` on 500K orders. The operation loads every entity into memory, causing an OOM. How do you fix this without rewriting the entire service?**
   A: Use a `@Modifying @Query` with JPQL UPDATE instead of loading entities:
   ```java
   @Modifying
   @Transactional
   @Query("UPDATE Order o SET o.status = :status WHERE o.createdAt < :cutoff")
   int bulkArchive(@Param("status") OrderStatus status, @Param("cutoff") LocalDateTime cutoff);
   ```
   This translates to a single SQL UPDATE — no entities loaded, no memory issue. Return `int` to get the count of affected rows for auditing.

3. **Q: A @ManyToOne association is fetched eagerly (default) and every list endpoint includes a join to the parent table. The parent table is small and rarely changes. How do you optimize without touching every repository method?**
   A: Override the default fetch plan at the entity level by using `@ManyToOne(fetch = FetchType.LAZY)`. Then, for specific queries that need the parent, use `@EntityGraph` or `JOIN FETCH`. This is a global fix: change the fetch type on the annotation, and only queries that explicitly request the join will execute it. Note that Jackson serialization of lazy proxies requires `@JsonIgnoreProperties` or DTOs to avoid `LazyInitializationException`.

4. **Q: You have a custom `@Query` that calls a stored procedure, but the `EntityManager` cache returns stale data for subsequent reads in the same transaction. What's happening?**
   A: The stored procedure bypasses the Hibernate first-level cache. After calling it, the persistence context is out of sync with the database. Call `entityManager.flush()` before the stored procedure and `entityManager.clear()` after to force fresh data loading on subsequent reads. For native queries, use `@Modifying` which automatically clears the cache.

5. **Q: Your application runs on Kubernetes with 10 replicas. Each replica has its own second-level cache. After an update on one replica, the other 9 return stale data. How do you solve this without Redis?**
   A: Use a distributed cache invalidation strategy: (a) Publish a `CacheInvalidationEvent` via RabbitMQ or Kafka when data changes, and have all replicas listen and evict their local caches. (b) Reduce TTL to a tolerance window (e.g., 60s) so drift is bounded. (c) Better yet, migrate to a shared distributed cache like Redis. For Hibernate second-level cache specifically, use Hazelcast or Redis as a shared store.

6. **Q: Your `@OneToMany` collection uses `CascadeType.ALL` and a developer accidentally deletes an `Order` which cascades to delete `Customer`, `Address`, and `PaymentHistory`. How do you prevent this?**
   A: Remove `CascadeType.ALL` and be explicit about cascade types. Never cascade `REMOVE` or `ALL` from parent to child unless you are certain about the consequences. Use `CascadeType.PERSIST` and `CascadeType.MERGE` only. For delete operations, implement a soft-delete pattern:
   ```java
   @Column(nullable = false)
   private boolean deleted = false;

   @Query("UPDATE Order o SET o.deleted = true WHERE o.id = :id")
   @Modifying
   void softDelete(@Param("id") Long id);
   ```

7. **Q: You need to search across 5 tables with 12 optional filters. The query must return paginated results with a total count. Building the query with JPQL string concatenation feels fragile. What pattern do you use?**
   A: Use Spring Data JPA `Specification` with `JpaSpecificationExecutor`. Build the query programmatically with a fluent API:
   ```java
   Specification<Order> spec = Specification
       .where(hasStatus(request.getStatus()))
       .and(createdBetween(request.getFrom(), request.getTo()))
       .and(hasCustomerInSegment(request.getSegment()))
       .and(totalGreaterThan(request.getMinTotal()));
   Page<Order> result = orderRepository.findAll(spec, pageable);
   ```
   Each method returns a `Specification` that composes with `and()` / `or()`. This is type-safe, testable, and avoids string concatenation.

8. **Q: You have a `User` entity with a `@OneToOne` `Address`. Loading 1000 users with `findAll()` generates 1001 SQL queries. You add `JOIN FETCH` but suddenly the result has 1000 rows and the Address query is still separate. Why?**
   A: If the `JOIN FETCH` is not being applied, check: (a) the `findAll()` method is from `JpaRepository` which has its own query — you cannot override it. Create a custom method `findAllWithAddress()`. (b) If using `@EntityGraph`, the repository must be a `JpaSpecificationExecutor` or use `@EntityGraph` on the method. (c) For `@OneToOne`, the default fetch is `EAGER`, so it actually fetches with a separate select (not a join) by default. Set `fetch = LAZY` and use `@EntityGraph(attributePaths = "address")` on a custom query.

9. **Q: A production incident shows that calling `save()` on 10K entities in a loop takes 45 seconds. How do you speed this up?**
   A: Use batch inserts. Configure `spring.jpa.properties.hibernate.jdbc.batch_size=50` and `spring.jpa.properties.hibernate.order_inserts=true`. Then use `saveAll()` instead of `save()` in a loop:
   ```java
   @Transactional
   public void importOrders(List<Order> orders) {
       orderRepository.saveAll(orders); // Batches into groups of 50
   }
   ```
   Also set `spring.jpa.properties.hibernate.generate_statistics=true` to verify batching works. For truly massive imports (100K+), consider using JDBC batch updates directly or `EntityManager` with manual flush/clear cycles every N records.

10. **Q: You need to return a list of `OrderSummary` (id, total, status) for an admin dashboard. The `Order` entity has 30 fields and multiple lazy associations. Loading full entities causes massive over-fetching. How do you implement this?**
    A: Use a DTO projection with JPQL constructor expression:
    ```java
    @Query("SELECT new com.app.dto.OrderSummary(o.id, o.total, o.status) FROM Order o")
    List<OrderSummary> findAllSummaries();
    ```
    Or use an interface-based projection:
    ```java
    public interface OrderSummary {
        Long getId();
        BigDecimal getTotal();
        String getStatus();
    }
    List<OrderSummary> findAllProjectedBy(); // No JPQL needed
    ```
    Hibernate generates a SELECT with only the 3 columns — no associations loaded, no lazy proxy issues, minimal memory footprint.

---

## Interview Questions

1. **What is the difference between `JpaRepository`, `CrudRepository`, and `PagingAndSortingRepository`?** 
   A: `CrudRepository` provides basic CRUD methods. `PagingAndSortingRepository` extends it with `findAll(Pageable)` and `findAll(Sort)`. `JpaRepository` extends both with JPA-specific methods like `flush()`, `saveAndFlush()`, and `deleteInBatch()`. Prefer `JpaRepository` in most Spring Boot applications.

2. **What is the N+1 query problem and how do you solve it?** 
   A: The N+1 problem occurs when you fetch N entities and then access their lazy associations in a loop, generating N extra SQL queries. Fix using `JOIN FETCH` in JPQL, `@EntityGraph` for dynamic fetch plans, or `@BatchSize` for batch lazy loading. Monitor with `spring.jpa.show-sql=true` or Hibernate statistics.

3. **What is the difference between `Page` and `Slice`?** 
   A: `Page` executes an extra `COUNT` query to determine total pages, useful for pagination UIs with page numbers. `Slice` skips the count query and only knows if a next page exists via `hasNext()`. Use `Slice` for infinite scroll to avoid the performance cost of `COUNT` on large tables.

4. **What are entity state transitions in JPA?** 
   A: NEW (transient) — not persisted, no ID. MANAGED — associated with a persistence context, changes are auto-tracked. DETACHED — was managed but the persistence context closed. REMOVED — marked for deletion. Transitions: `persist()` (NEW→MANAGED), `merge()` (DETACHED→MANAGED), `remove()` (MANAGED→REMOVED), `detach()`/`clear()`/`close()` (MANAGED→DETACHED).

5. **What is the difference between `save()` and `saveAndFlush()`?** 
   A: `save()` persists the entity but batches the SQL execution — the INSERT may not execute immediately. `saveAndFlush()` forces immediate SQL execution. Use `saveAndFlush()` when you need the generated ID immediately or before executing a native query that references the unflushed data.

6. **How do you implement soft delete in Spring Data JPA?** 
   A: Add a `deleted` boolean column and use `@Where(clause = "deleted = false")` on the entity to automatically filter soft-deleted records. Override delete methods with `@Query` UPDATE statements. For queries that need to include deleted records, use a separate repository or native queries.

7. **What is the `LazyInitializationException` and how do you prevent it?** 
   A: It occurs when accessing a lazy-loaded association outside of an active Hibernate session (transaction). Prevent by: (a) using `JOIN FETCH` to eagerly load needed associations within the transaction, (b) using `@EntityGraph`, (c) DTO projections (best), or (d) disabling OSIV (not recommended as a fix). Never rely on `spring.jpa.open-in-view=true` as it masks the real problem.

8. **How do you use `@Query` with native SQL and pagination?** 
   A: Provide a count query explicitly: `@Query(value = "...", countQuery = "SELECT count(*) FROM ...", nativeQuery = true)`. Spring Data JPA cannot derive the count query from native SQL automatically.

9. **What is the purpose of `@Modifying` annotation?** 
   A: `@Modifying` indicates that a `@Query` is an INSERT, UPDATE, DELETE, or DDL statement. It triggers `EntityManager.flush()` and `clear()` automatically (unless `clearAutomatically = false`). Always combine with `@Transactional`. The method should return `int` (affected rows) or `void`.

10. **What are Spring Data JPA Specifications and when should you use them?** 
    A: Specifications provide a type-safe, programmatic way to build dynamic queries. Use them when you have optional, combinable filters that cannot be expressed with derived method names. The repository must extend `JpaSpecificationExecutor`. Specifications are composable with `and()` / `or()` and work seamlessly with pagination.

---

## Developer Recommendations

- **Use `JOIN FETCH` or `@EntityGraph` over `FetchType.EAGER`** — Eager fetching is a global default that always loads the association, even when not needed, causing Cartesian products and memory waste. `JOIN FETCH` is query-specific and explicit. `@EntityGraph` gives you query-specific fetch plans without changing the entity.
- **Use DTO projections instead of full entities for read operations** — Loading full entities fetches all columns and triggers lazy associations. Interface-based projections generate optimized SELECT queries with only the needed columns. This reduces memory, network, and serialization overhead significantly.
- **Use `@Modifying` with `@Query` for bulk operations** — Loading 100K entities just to update one field is wasteful and causes OOM. A JPQL UPDATE translates to a single SQL statement and performs in milliseconds. Always return `int` to verify the affected row count.
- **Prefer `LAZY` fetching on all associations** — `@ManyToOne` defaults to `EAGER`, which can silently join large tables on every query. Set `fetch = FetchType.LAZY` explicitly and use `@EntityGraph` when you need the association. This prevents performance surprises as the entity grows.
- **Be explicit about cascade types** — `CascadeType.ALL` is dangerous: deleting a parent cascades to all children, potentially wiping out large parts of the database. Use only the cascade types you need (`PERSIST`, `MERGE`) and never cascade `REMOVE` without careful consideration.
- **Use `saveAll()` and configure batch inserts for bulk operations** — Calling `save()` in a loop is slow because each call flushes the persistence context. Configure `hibernate.jdbc.batch_size=50` and use `saveAll()` to batch INSERTs into a single JDBC batch. Monitor with `hibernate.generate_statistics=true`.
- **Always use `@Transactional` on service-layer methods that modify data** — Without it, lazy loading fails outside the repository call, and Hibernate may not flush changes. Keep transaction boundaries at the service layer, not the repository layer, to ensure consistency across multiple repository calls.
- **`equals()` and `hashCode()` should not be based on the database ID** — New entities have `null` ID, so adding them to a `Set` before persisting breaks equality. Use a business key (UUID, unique natural key) or keep the default `Object` equality.
