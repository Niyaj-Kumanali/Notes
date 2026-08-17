# Spring Data JPA and Hibernate Questions

## Questions

1. What is JPA?
2. What is Hibernate?
3. Difference between JPA and Hibernate.
4. What is entity?
5. What is `@Id`?
6. What is generated value?
7. What is repository?
8. What is Spring Data JPA?
9. What is `JpaRepository`?
10. Difference between `CrudRepository` and `JpaRepository`.
11. What is derived query method?
12. What is JPQL?
13. Difference between JPQL and native query.
14. What is entity lifecycle?
15. What is persistence context?
16. What is dirty checking?
17. What is first-level cache?
18. What is second-level cache?
19. What is lazy loading?
20. What is eager loading?
21. What is N+1 query problem?
22. How do you solve N+1 query problem?
23. What is fetch join?
24. What is entity graph?
25. What is pagination?
26. What is sorting?
27. What is transaction management?
28. What is `@Transactional`?
29. How does `@Transactional` work internally?
30. What is transaction propagation?
31. What is transaction isolation?
32. What is rollback behavior?
33. Why does self-invocation break `@Transactional`?
34. What is optimistic locking?
35. What is pessimistic locking?
36. What is `@Version`?
37. What are cascade types?
38. What is orphan removal?
39. Difference between `save()` and `saveAndFlush()`.
40. What is batch insert?
41. How do you improve JPA performance?
42. When should you use JDBC instead of JPA?
43. When should you use stored procedures?
44. How do you call stored procedures from Spring?
45. How do you debug slow JPA queries?
46. Why do we use the `@Transactional` annotation?
47. Can `@Transactional` be applied to private or static methods?
48. What types of exceptions trigger transaction rollback with `@Transactional`?

---

## Answers

1. What is JPA?
   - **Answer:**
      - JPA (Java Persistence API) is a specification for object-relational mapping in Java
      - In my CDMS project, I used JPA entities mapped to MSSQL tables with `@Entity`, `@Table`, and `@Column` annotations to interact with the partner and report data without writing manual SQL
      - JPA is a specification (interface), while Hibernate is a concrete implementation (class) — just like JDBC is a specification and a database driver provides the implementation
      - Key JPA annotations include `@Entity`, `@Table`, `@Column`, `@Id`, `@GeneratedValue`, `@OneToMany`, `@ManyToOne`, `@MappedSuperclass`, and `@Embeddable`
      - Configured the persistence unit with `spring.jpa.*` properties in `application.yml` — `spring.jpa.hibernate.ddl-auto` for schema generation, `spring.jpa.show-sql` for SQL logging, and `spring.jpa.properties.hibernate.dialect` for database compatibility
2. What is Hibernate?
   - **Answer:**
      - Hibernate is the most popular JPA implementation that handles ORM, caching, lazy loading, and transaction management
      - In my inventory project, Hibernate was the underlying engine for Spring Data JPA — it generated SQL queries, managed the first-level cache, and handled entity state transitions automatically
      - Hibernate-specific features beyond JPA include second-level caching (e.g., with Redis using `hibernate-cache-redis`), annotations like `@BatchSize` for batch loading and `@Fetch` for fetch strategies, and HQL (Hibernate Query Language) which is similar to JPQL but supports Hibernate-specific extensions like bulk UPDATE and DELETE operations
3. Difference between JPA and Hibernate.
   - **Answer:**
      - JPA is a specification (interface), Hibernate is an implementation (concrete class)
      - I used JPA annotations and interfaces like `EntityManager` and `@Entity` to keep my code portable, while Hibernate ran underneath
      - If I ever needed Hibernate-specific features, I used them sparingly to avoid vendor lock-in
      - Spring Data JPA abstracts both the specification and implementation, allowing providers to be swapped; preferred JPA standard APIs for most operations to maintain portability
      - Hibernate 6 introduced the Jakarta Persistence namespace (`jakarta.persistence.*` instead of `javax.persistence.*`), improved SQL generation, and better standards compliance; `hibernate.jpa.compliance` settings (e.g., `hibernate.jpa.compliance.query`, `hibernate.jpa.compliance.close`) can enforce strict JPA compliance at the cost of Hibernate-specific optimizations
4. What is entity?
   - **Answer:**
      - An entity is a Java class mapped to a database table using `@Entity`
      - In my inventory project, I had `PartnerEntity`, `InventoryRecord`, and `AuditLog` entities with fields annotated with `@Column` to define column names, lengths, and nullability in MSSQL
      - Entity requirements: a no-arg constructor (required by Hibernate for proxy instantiation), an `@Id` field marking the primary key, getters/setters for all mapped fields, and proper `equals()` and `hashCode()` using business keys (not the `@Id`) to avoid issues with Sets, collections, and lazy loading proxies
5. What is `@Id`?
   - **Answer:**
      - `@Id` marks a field as the primary key of the entity
      - In my CDMS project, I used `@Id` on the `partnerId` field of `PartnerEntity`, with `@GeneratedValue(strategy = GenerationType.IDENTITY)` to let MSSQL auto-increment the primary key
      - Composite primary keys use `@IdClass` (defining a separate POJO key class) or `@EmbeddedId` (embedding an `@Embeddable` key class); used `@EmbeddedId` in the inventory entity with a composite key containing `partnerId` and `serialNumber` to uniquely identify inventory records
6. What is generated value?
   - **Answer:**
      - `@GeneratedValue` specifies how the primary key is auto-generated
      - In my projects, I used `GenerationType.IDENTITY` for MSSQL auto-increment columns because it was simpler and more efficient than `SEQUENCE` for my use case
      - Four generation strategies: AUTO (lets Hibernate choose based on the database dialect), IDENTITY (uses database auto-increment columns), SEQUENCE (uses database sequences), and TABLE (simulates sequences using a dedicated database table)
      - IDENTITY disables batch inserts because Hibernate must flush after each insert to retrieve the generated ID; SEQUENCE supports batching and is preferred for high-throughput scenarios; TABLE adds overhead from row-level locking on a separate table
      - Chose IDENTITY despite the batch insert limitation because inserts were single-record operations where the overhead of configuring a sequence was not justified
7. What is repository?
   - **Answer:**
      - A repository is a Spring Data interface that provides CRUD operations without implementation code
      - In my inventory project, I created `InventoryRepository extends JpaRepository<InventoryRecord, Long>` and automatically got methods like `findAll()`, `save()`, `deleteById()`, and derived query methods
      - Repository hierarchy: `Repository` (marker interface) → `CrudRepository` (CRUD methods) → `PagingAndSortingRepository` (pagination/sorting) → `JpaRepository` (flush, batch, query-by-example); Spring Data generates the implementation at runtime using `SimpleJpaRepository` via dynamic proxy generation
8. What is Spring Data JPA?
   - **Answer:**
      - Spring Data JPA is a Spring module that reduces JPA boilerplate by providing repository abstractions
      - In my CDMS project, I defined `PartnerRepository extends JpaRepository` and Spring Data JPA automatically provided implementations for CRUD, pagination, and derived queries without writing any DAO code
      - Under the hood, Spring Data JPA delegates to `EntityManager` for all persistence operations
      - Method names are parsed by Spring Data's query derivation mechanism, which translates them to JPQL (e.g., `findByStatus` becomes `SELECT e FROM Entity e WHERE e.status = :status`)
      - Used `@Query` annotations for complex queries that couldn't be expressed as derived methods, such as multi-table joins or MSSQL-specific logic
9. What is `JpaRepository`?
   - **Answer:**
      - `JpaRepository` is a Spring Data interface extending `PagingAndSortingRepository` with JPA-specific methods like `flush()`, `saveAndFlush()`, and `deleteInBatch()`
      - I used `JpaRepository` for all my MSSQL entities in the inventory project because I needed pagination and batch operations
      - `JpaRepository` adds `flush()`, `saveAndFlush()`, `saveAllAndFlush()`, `deleteInBatch()`, `deleteAllInBatch()`, `getReferenceById()`, and `findAll(Sort)` / `findAll(Pageable)` over `CrudRepository`
      - `PagingAndSortingRepository` is preferred when only pagination and sorting are needed without JPA-specific methods like `flush()` or `deleteInBatch()`, keeping the repository contract more portable
10. Difference between `CrudRepository` and `JpaRepository`.
    - **Answer:**
       - `CrudRepository` provides basic CRUD methods (save, findById, findAll, delete)
       - `JpaRepository` extends it with JPA-specific methods like `flush()`, `saveAndFlush()`, `deleteInBatch()`, and pagination/sorting
       - I used `JpaRepository` in all my projects since I always needed pagination for the partner listing APIs
       - `CrudRepository` is persistence-technology agnostic and could theoretically work with any persistence mechanism, while `JpaRepository` is specifically tied to JPA with methods like `flush()` and `saveAndFlush()`
       - `ListCrudRepository` (introduced in Spring Data 3.x) returns `List` instead of `Iterable` from methods like `findAll()`, which is more convenient and avoids the need to cast or convert when `List` is required
11. What is derived query method?
    - **Answer:**
       - Derived query methods allow creating queries by naming methods according to a convention
       - In my inventory project, I defined `findByPartnerIdAndStatus(Long partnerId, String status)` and Spring Data JPA automatically generated the JPQL query based on the method name — no need to write queries manually
       - Query derivation keywords include `And`, `Or`, `Between`, `Like`, `OrderBy`, `Top`/`First`, `IgnoreCase`, and prefixes like `findBy`, `countBy`, `existsBy`, `deleteBy` that define the operation type
       - Method names become unreadable beyond 3-4 conditions; switch to `@Query` annotations for complex or multi-join queries to maintain readability
12. What is JPQL?
    - **Answer:**
       - JPQL (Java Persistence Query Language) is an object-oriented query language similar to SQL but operates on entities instead of tables
       - In my CDMS project, I wrote JPQL queries like `SELECT p FROM PartnerEntity p WHERE p.status = :status` to query partners without database-specific SQL
       - JPQL uses entity class names and Java field names (e.g., `PartnerEntity`, `partnerName`), while SQL uses database table names and column names (e.g., `t_partners`, `partner_name`)
       - Custom JPQL queries are defined via `@Query` annotations, e.g., `@Query("SELECT p FROM PartnerEntity p WHERE p.lastSyncDate < :date")` for time-based filtering
13. Difference between JPQL and native query.
    - **Answer:**
       - JPQL is database-independent and works with entity fields; native queries use raw SQL specific to the database
       - In my CDMS project, I used JPQL for standard queries and native queries with `@Query(value = "EXEC sp_generate_report :partnerId", nativeQuery = true)` for executing MSSQL stored procedures
       - JPQL is portable across databases but limited to what the JPA specification supports; native queries give full SQL control for database-specific features (window functions, CTEs, stored procedures) but break portability
       - Native queries should be reserved for stored procedures and complex SQL features (e.g., window functions, CTEs) that cannot be expressed in JPQL
14. What is entity lifecycle?
    - **Answer:**
       - Entity lifecycle has four states: New (transient), Managed (persistent), Detached, and Removed
       - In my inventory project, entities retrieved via `findById()` were in the managed state — any changes to them were automatically persisted at flush time without calling `save()`
       - State transitions: `persist()` moves NEW → MANAGED, `merge()` moves DETACHED → MANAGED (returns a new managed instance), `remove()` moves MANAGED → REMOVED, and DETACHED state is reached when the persistence context is closed or the entity is explicitly detached
       - Detached entity issues occur when an entity is modified outside a transaction (after the persistence context closes); the changes are not tracked, and `merge()` must be called to re-attach and persist the modifications
15. What is persistence context?
    - **Answer:**
       - The persistence context is a first-level cache that tracks entity state changes within a transaction
       - In my inventory project, when I loaded a partner entity inside a `@Transactional` service method, the persistence context kept track of all field modifications and flushed them to MSSQL at commit time
       - Internally, the persistence context is a `Map<Object, EntityEntry>` keyed by entity type and `@Id` — it guarantees at most one managed instance per database row
       - It ensures repeatable reads within a transaction by returning the same managed instance for repeated `find()` calls, avoiding stale or inconsistent data
       - `clear()` removes all entities from the context (they become detached), which helps manage memory when processing large datasets; `detach()` removes a specific entity so further changes are not tracked
16. What is dirty checking?
    - **Answer:**
       - Dirty checking is Hibernate's mechanism to detect changes to managed entities and automatically persist them
       - In my CDMS project, I loaded a `PartnerEntity`, updated its `status` field inside a `@Transactional` method, and Hibernate automatically generated an UPDATE query at flush time without me calling `save()`
       - Hibernate takes a snapshot of each entity's state when it is loaded into the persistence context; at flush time, it compares the current state against the snapshot to detect which fields changed, generating UPDATE SQL only for modified columns
       - Hibernate's enhanced dirty checking (enabled via bytecode enhancement) can be tuned for large entities to track only modified fields rather than comparing all columns, reducing snapshot comparison overhead
17. What is first-level cache?
    - **Answer:**
       - The first-level cache is Hibernate's session-level cache within the persistence context
       - In my inventory project, loading the same entity twice by `findById()` inside the same transaction returned the cached object without a second database query, improving performance for repeated lookups
       - The first-level cache is always enabled and cannot be disabled — it is fundamental to how Hibernate manages entity state within a session
       - Scoped to the `EntityManager` (session) — each persistence context maintains its own first-level cache, and entities cached in one context are not visible to another
       - `clear()` removes all entities from the cache (making them detached); `evict()` removes a single entity — both are useful for managing memory during large batch operations to prevent the persistence context from growing too large
18. What is second-level cache?
    - **Answer:**
       - The second-level cache is a session-factory-level cache shared across transactions
       - In my cold-chain project, I configured Hibernate's second-level cache with Redis as the cache store using `hibernate-cache-redis`, caching frequently accessed but rarely changed entity data like gateway configurations
       - First-level cache is session-scoped (per persistence context) and always enabled; second-level cache is session-factory-scoped (shared across all sessions) and must be explicitly configured
       - Enable with `hibernate.cache.use_second_level_cache=true` and configure per entity using `@Cacheable` and `@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)` — the concurrency strategy controls how concurrent read/write access is handled
       - Cache concurrency strategies: `READ_ONLY` (safe for immutable data, throws exception on update), `READ_WRITE` (uses soft locks to ensure consistency), `NONSTRICT_READ_WRITE` (best-effort, may return stale data), and `TRANSACTIONAL` (JTA only, strongest consistency guarantee)
19. What is lazy loading?
    - **Answer:**
       - Lazy loading defers loading of associated entities until they are actually accessed
       - In my CDMS project, the `PartnerEntity` had a `@OneToMany(fetch = FetchType.LAZY)` collection of `OrderEntity` — the orders were loaded from MSSQL only when I accessed `partner.getOrders()`
       - Lazy loading works via Hibernate proxies — when an entity with `FetchType.LAZY` is loaded, Hibernate creates a proxy object that intercepts access to the association and loads the data on first use
       - `LazyInitializationException` occurs when accessing a lazy-loaded collection after the session (and persistence context) has closed — the proxy cannot initialize because the database connection is no longer available
       - Solved by keeping the session open with `@Transactional(readOnly = true)` on the calling method, or by using `JOIN FETCH` in JPQL to eagerly load the association in a single query
20. What is eager loading?
    - **Answer:**
       - Eager loading loads associated entities immediately when the parent entity is fetched
       - In my inventory project, I used `@ManyToOne(fetch = FetchType.EAGER)` on the `InventoryRecord.partner` field because the partner data was always needed alongside inventory data, avoiding the N+1 problem for that relationship
       - Default fetch types: `@OneToMany` and `@ManyToMany` default to `LAZY` (associations loaded on demand); `@ManyToOne` and `@OneToOne` default to `EAGER` (association loaded immediately with the parent)
       - Eager loading can cause unnecessary joins and Cartesian product issues — when multiple EAGER associations exist on an entity, Hibernate generates queries with multiple JOINs that multiply rows and degrade performance
21. What is N+1 query problem?
    - **Answer:**
       - The N+1 problem occurs when 1 query fetches parent entities and N additional queries fetch associated collections
       - In my CDMS project, `partnerRepository.findAll()` followed by accessing `partner.getOrders()` in a loop triggered N separate queries — making the report generation extremely slow
       - Detect N+1 by enabling `spring.jpa.show-sql=true` or `spring.jpa.properties.hibernate.format_sql=true` and watching for repeated identical queries for associated collections in the logs
       - Identified the N+1 problem in the CDMS project by seeing hundreds of SELECT queries in the logs when iterating over partner orders — the initial query returned N partners, and each partner's orders triggered an additional SELECT
22. How do you solve N+1 query problem?
    - **Answer:**
       - I solve N+1 using `JOIN FETCH` in JPQL or `@EntityGraph`
       - In my CDMS project, I changed the query from `SELECT p FROM PartnerEntity p` to `SELECT p FROM PartnerEntity p JOIN FETCH p.orders`, which fetched partners and orders in a single SQL query using an INNER JOIN
       - Solutions: `JOIN FETCH` (eagerly loads associations in a single query via JPQL), `@EntityGraph` (declaratively specifies which associations to fetch), `@BatchSize` (loads lazy associations in batches of N when first accessed), and Hibernate's `hibernate.batch_fetch_size` (globally configures batch loading for all lazy associations)
       - Trade-off: `JOIN FETCH` with multiple collections causes Cartesian product (row multiplication) and duplicate results; use `Set` instead of `List` for the collection type to deduplicate, or use `@BatchSize` which avoids the Cartesian product by issuing separate batched queries
23. What is fetch join?
    - **Answer:**
       - Fetch join is a JPQL feature that loads associations in a single query using JOIN
       - I used `SELECT p FROM PartnerEntity p JOIN FETCH p.orders` in my inventory project to load partners along with their inventory records in one SQL query, completely avoiding the N+1 problem
       - A fetch join does not change the entity's fetch type definition — it only overrides it for that specific query, loading the association eagerly just for that execution
       - Use `LEFT JOIN FETCH` (instead of `JOIN FETCH`) when the association might be null or empty, to avoid filtering out parent entities that have no associated children
       - When fetching multiple collections with `JOIN FETCH`, the result contains duplicate parent rows due to the Cartesian product; use `Set` instead of `List` as the collection type to allow Hibernate to deduplicate
24. What is entity graph?
    - **Answer:**
       - Entity graphs allow defining fetch plans dynamically using `@NamedEntityGraph` or `@EntityGraph`
       - In my CDMS project, I defined `@NamedEntityGraph(name = "Partner.withOrders", attributeNodes = @NamedAttributeNode("orders"))` and used `@EntityGraph("Partner.withOrders")` on the repository method for flexible fetch strategies
       - `EntityGraphType.FETCH` replaces the default fetch plan — only the specified attributes are fetched and all others become LAZY; `EntityGraphType.LOAD` adds the specified attributes to the default plan, preserving existing EAGER associations
       - Entity graphs solve N+1 by declaring which associations to fetch at the repository method level via `@EntityGraph`, avoiding the need to rewrite JPQL queries with `JOIN FETCH`
25. What is pagination?
    - **Answer:**
       - Pagination divides large result sets into smaller pages
       - In my CDMS project, I used `Pageable` parameter in repository methods: `Page<PartnerEntity> findAll(Pageable pageable)`
       - The API returned page number, size, total elements, and total pages, allowing the React frontend to display paginated tables
       - `PageRequest.of(page, size, Sort)` creates a `Pageable` that specifies the page number (0-indexed), page size, and optional sort order
       - Spring Data JPA translates `Pageable` into database-specific SQL — MSSQL uses `OFFSET ... ROWS FETCH NEXT ... ROWS`, PostgreSQL uses `LIMIT ... OFFSET`, and MySQL uses `LIMIT ... OFFSET ...`
       - Large offsets degrade performance because the database must scan and discard all preceding rows; keyset pagination (using `WHERE id > :lastSeenId ORDER BY id LIMIT :size`) avoids this by seeking directly to the starting point
26. What is sorting?
    - **Answer:**
       - Sorting orders query results by specified fields
       - In my inventory project, I used `Sort.by("partnerName").ascending()` in the repository method or passed `Sort` through `Pageable`
       - For multi-field sorting, I used `Sort.by(Order.asc("partnerName"), Order.desc("createdDate"))`
       - Dynamic sorting uses `Sort` passed through `Pageable` (user-controlled, applied at query time); static sorting uses `@OrderBy` on entity associations (e.g., `@OneToMany @OrderBy("createdDate DESC")`) to define a fixed sort order for the collection
       - `Sort.by(field)` is validated by Spring Data against entity field names for derived queries, but custom implementations or native queries must sanitize sort parameters to prevent SQL injection
27. What is transaction management?
    - **Answer:**
       - Transaction management ensures a group of database operations either all succeed or all fail
       - In my inventory project, I used Spring's declarative transaction management with `@Transactional` on service methods to ensure atomicity — if one validation step failed, the entire batch update was rolled back
       - ACID: Atomicity (all operations succeed or all roll back), Consistency (data remains valid after transaction), Isolation (concurrent transactions don't interfere), Durability (committed data survives system failures)
       - Declarative (`@Transactional`) uses annotations and AOP proxies for cleaner, less invasive code; programmatic (`TransactionTemplate`) gives explicit control over transaction boundaries but clutters business logic with transaction boilerplate
       - Spring wraps the service bean in a CGLIB proxy that intercepts method calls, starts a transaction before the method, and commits or rolls back after — the business logic remains transaction-framework-agnostic
28. What is `@Transactional`?
    - **Answer:**
       - `@Transactional` ensures a set of database operations either all succeed or all roll back
       - In my inventory project, I used it on the batch validation service so that if any partner record failed validation, the entire batch was rolled back to maintain data consistency
       - Key attributes: `propagation` (how the method joins an existing transaction), `isolation` (concurrency control level), `timeout` (seconds before transaction times out), `readOnly` (optimization hint for read-only transactions), and `rollbackFor` (specific exceptions that trigger rollback)
       - `readOnly = true` optimizes read-only methods (Hibernate skips dirty checking and the database may use read-only locks); `rollbackFor = CustomException.class` specifies which checked exceptions should trigger rollback instead of committing
29. How does `@Transactional` work internally?
    - **Answer:**
       - Spring uses AOP proxies to wrap `@Transactional` methods
       - When a method annotated with `@Transactional` is called, Spring creates a transaction before method execution and commits or rolls back based on the outcome
       - In my projects, I confirmed this works via CGLIB proxy — which means self-invocation bypasses the proxy and the annotation is ignored
       - `TransactionInterceptor` (an AOP advice) intercepts the method call, delegates to `PlatformTransactionManager` to begin/commit/rollback, and `TransactionAspectSupport` holds the transaction status in a `ThreadLocal`
       - `TransactionAspectSupport` stores transaction status in a `ThreadLocal` variable, allowing nested method calls to participate in the same transaction via `TransactionSynchronizationManager`
       - `DataSourceTransactionManager` manages transactions for plain JDBC; `JpaTransactionManager` integrates with JPA's `EntityManager` and persistence context — used `JpaTransactionManager` for MSSQL entities to ensure the persistence context and database transactions are synchronized
30. What is transaction propagation?
    - **Answer:**
       - Transaction propagation defines how transactions behave when a transactional method calls another transactional method
       - In my CDMS project, I used `REQUIRED` (default) for most methods, meaning they join the existing transaction
       - For the audit logging method, I used `REQUIRES_NEW` so the audit log was saved even if the main transaction rolled back
       - Seven propagation types: `REQUIRED` (join existing or create new), `SUPPORTS` (join existing or run without), `MANDATORY` (must join existing, throws exception if none), `REQUIRES_NEW` (always create new, suspends existing), `NOT_SUPPORTED` (suspend existing, run without), `NEVER` (must not have existing, throws exception), `NESTED` (create savepoint within existing)
       - `REQUIRES_NEW` was used for the notification service so alerts were committed independently — if the main transaction rolled back, notifications already sent were preserved
31. What is transaction isolation?
    - **Answer:**
       - Transaction isolation defines how concurrent transactions interact
       - In my inventory project, I used `@Transactional(isolation = Isolation.READ_COMMITTED)` which is MSSQL's default and prevents dirty reads
       - For the inventory calculation method, I considered `REPEATABLE_READ` to prevent phantom reads during stock computation
       - Four isolation levels: `READ_UNCOMMITTED` (reads uncommitted data), `READ_COMMITTED` (reads only committed data), `REPEATABLE_READ` (same query returns same results within transaction), `SERIALIZABLE` (full isolation, transactions execute as if serialized)
       - Dirty read (reading uncommitted changes) is prevented by `READ_COMMITTED` and above; non-repeatable read (same query returns different values) is prevented by `REPEATABLE_READ` and above; phantom read (new rows appear between queries) is prevented by `SERIALIZABLE`
       - Higher isolation levels increase locking and reduce concurrency — `SERIALIZABLE` holds locks for the entire transaction, while `READ_COMMITTED` releases read locks immediately, making it the best balance for most applications
32. What is rollback behavior?
    - **Answer:**
       - By default, `@Transactional` rolls back on unchecked exceptions (RuntimeException) and commits on checked exceptions
       - In my projects, I customized this with `rollbackFor = {DataAccessException.class, CustomValidationException.class}` to ensure certain checked exceptions also triggered rollback
       - `noRollbackFor` specifies exceptions that should NOT trigger rollback — useful when certain exceptions represent expected outcomes that should not undo prior successful operations
       - Declarative rollback rules can be configured via `@Transactional(rollbackFor = ..., noRollbackFor = ...)` on methods, or via `<tx:advice>` XML configuration for broader, class-level rules — allowing fine-grained control over which exceptions commit and which roll back
33. Why does self-invocation break `@Transactional`?
    - **Answer:**
       - Self-invocation bypasses the Spring AOP proxy because the call happens within the same class, not through the injected proxy
       - In my inventory project, I accidentally faced this when method A (annotated with `@Transactional`) called method B (also `@Transactional`) in the same service — the transaction for B was ignored
       - Spring creates a CGLIB proxy (subclass-based) or JDK dynamic proxy (interface-based) around the bean — the proxy intercepts external calls and manages the transaction, but internal method calls within the same class bypass the proxy entirely
       - Solutions: self-inject the proxy via `@Lazy` injection or `ApplicationContextAware`, extract transactional methods into a separate `@Service` class, or use `TransactionTemplate` for programmatic transaction control that doesn't depend on proxy interception
34. What is optimistic locking?
    - **Answer:**
       - Optimistic locking assumes conflicts are rare and checks for them at commit time using a version column
       - In my inventory project, I added `@Version` on the `InventoryRecord` entity to prevent concurrent updates — if two users tried to update the same record, the second commit threw `OptimisticLockException`
       - Optimistic locking does not acquire database locks — it reads the entity's version number, and at commit time, verifies the version has not changed by issuing `UPDATE ... WHERE version = :oldVersion`; if another transaction modified the row, the WHERE clause matches zero rows and an exception is thrown
       - `StaleObjectStateException` (or `OptimisticLockException` in JPA) is thrown when the version check fails, indicating the entity was modified by another transaction since it was read
       - Handled by catching the exception in the service layer, either retrying the operation (for idempotent operations) or returning an error message to the user indicating the record was modified by another user
35. What is pessimistic locking?
    - **Answer:**
       - Pessimistic locking locks the database row at read time to prevent other transactions from modifying it
       - In my CDMS project, I used `@Lock(LockModeType.PESSIMISTIC_WRITE)` on the repository method that calculated partner inventory, ensuring no other transaction could modify the data during the calculation
       - `PESSIMISTIC_READ` acquires a shared lock (allows concurrent reads but blocks writes); `PESSIMISTIC_WRITE` acquires an exclusive lock (blocks all other reads and writes); in MSSQL these translate to `SELECT ... WITH (UPDLOCK)` and `SELECT ... WITH (XLOCK)` respectively
       - Pessimistic locking prevents conflicts by blocking concurrent access, but reduces throughput and can cause deadlocks; optimistic locking allows higher concurrency but requires conflict handling at application level — prefer optimistic for low-contention scenarios and pessimistic for high-contention critical sections
36. What is `@Version`?
    - **Answer:**
       - `@Version` is a JPA annotation that enables optimistic locking by maintaining a version number
       - In my inventory project, I added `@Version private Long version;` to the `InventoryRecord` entity
       - Hibernate automatically incremented the version on every update and checked it before committing, throwing `OptimisticLockException` on conflict
       - `@Version` works with `int`, `long`, `Integer`, `Long`, `short`, `BigDecimal`, `Timestamp`, and `Instant` — Hibernate automatically manages the version increment on updates
       - Forgetting `@Version` means Hibernate has no way to detect concurrent modifications — two transactions reading and updating the same row will silently overwrite each other's changes (last-write-wins), leading to data loss
37. What are cascade types?
    - **Answer:**
       - Cascade types determine which entity operations propagate to associated entities
       - In my CDMS project, I used `cascade = CascadeType.PERSIST` on the `@OneToMany(mappedBy = "partner")` orders collection so that saving a partner also saved its orders without separate save calls
       - Cascade types: `ALL` (propagates all operations), `PERSIST` (save children with parent), `MERGE` (merge children with parent), `REMOVE` (delete children with parent), `REFRESH` (refresh children from DB), `DETACH` (detach children from context); common mistake: `CascadeType.ALL` on `@ManyToMany` causes unintended deletes from the join table when one side is removed
       - Used only `PERSIST` and `MERGE` on `@OneToMany` associations to propagate saves without accidentally deleting unrelated child records
38. What is orphan removal?
    - **Answer:**
       - Orphan removal automatically deletes child entities when they are removed from the parent's collection
       - In my inventory project, I used `orphanRemoval = true` on the `@OneToMany` partner-orders mapping so that when an order was removed from the partner's order list, it was automatically deleted from MSSQL
       - `orphanRemoval = true` deletes a child entity when it is removed from the parent's collection (e.g., `parent.getChildren().remove(child)` triggers DELETE); `cascade = CascadeType.REMOVE` deletes children only when the parent entity itself is deleted — they address different scenarios
       - Used both: `orphanRemoval = true` for cleaning up removed items from collections, and `CascadeType.REMOVE` for propagating parent deletion to dependent children
39. Difference between `save()` and `saveAndFlush()`.
    - **Answer:**
       - `save()` persists the entity and returns it, but does not immediately flush to the database — the actual INSERT/UPDATE happens at transaction commit
       - `saveAndFlush()` immediately flushes the SQL to the database
       - In my inventory project, I used `saveAndFlush()` when I needed the generated ID immediately for logging purposes
       - `save()` queues the INSERT/UPDATE in the persistence context and may batch it with other operations before flushing at transaction commit, improving performance by reducing database round-trips
       - `saveAndFlush()` immediately writes the SQL to the database, useful when the next operation depends on the entity being persisted (e.g., calling a stored procedure that requires the generated ID, or verifying a unique constraint before proceeding)
40. What is batch insert?
    - **Answer:**
       - Batch insert groups multiple INSERT statements into a single database round-trip for performance
       - In my CDMS project, I configured `spring.jpa.properties.hibernate.jdbc.batch_size=50` and `hibernate.order_inserts=true` so that inserting 500 partner records in a loop triggered only 10 batch inserts instead of 500 individual queries
       - Batch inserts require `GenerationType.SEQUENCE` or `TABLE` — `IDENTITY` disables batching because Hibernate must flush after each insert to retrieve the auto-generated ID
       - `hibernate.order_inserts=true` and `hibernate.order_updates=true` must be enabled to group INSERT/UPDATE statements by entity type, which is required for batching to work
       - `rewriteBatchedStatements=true` in the MSSQL JDBC connection URL rewrites individual INSERT statements into multi-row `INSERT ... VALUES (...), (...)` syntax, significantly improving batch insert performance
       - This was discovered during testing of the inventory import feature, where unbatched inserts were taking several minutes and batch configuration reduced the time to seconds
41. How do you improve JPA performance?
    - **Answer:**
       - I improve JPA performance by enabling batch operations, using fetch joins to avoid N+1 queries, configuring appropriate fetch types, enabling Hibernate query cache for read-heavy data, and monitoring slow queries via `spring.jpa.properties.hibernate.generate_statistics=true`
       - Specific optimizations: `hibernate.jdbc.batch_size=50` for bulk inserts, `@BatchSize(size = 50)` on lazy collections to load them in batches, projection DTOs (JPQL constructor expressions) to avoid SELECT * and load only needed columns, and native SQL or stored procedures for complex reporting queries that resist JPA optimization
42. When should you use JDBC instead of JPA?
    - **Answer:**
       - Use JDBC when performance is critical for bulk operations, complex reporting queries, or when executing stored procedures
       - In my CDMS project, I used `JdbcTemplate` for the stored procedure execution because it was simpler and faster than mapping complex result sets to entities for the reporting use case
       - JPA overhead becomes significant with thousands of entities: the persistence context tracks every entity (memory), dirty checking compares snapshots at flush (CPU), and lazy loading generates additional queries (I/O) — these costs are unacceptable for bulk operations
       - The CDMS nightly batch processed millions of records where JPA's persistence context would have caused OutOfMemoryError — JDBC's `JdbcTemplate` with `RowMapper` processes rows one at a time without loading all entities into memory
43. When should you use stored procedures?
    - **Answer:**
       - Use stored procedures for complex reporting logic, long-running batch operations, or when you need database-specific features that cannot be expressed in JPA
       - In my CDMS project, the nightly report generation was implemented as an MSSQL stored procedure because it involved complex joins, aggregations, and conditional logic that was more efficient in SQL
       - Benefits: reduced network round-trips (logic executes in the database engine), better performance for set-based operations (the database optimizer can plan the entire operation), and access to MSSQL-specific features like window functions, CTEs, temp tables, and bulk operations
       - Trade-offs: stored procedures are harder to unit test (require database integration tests), create vendor lock-in (MSSQL T-SQL is not portable to PostgreSQL), and business logic becomes harder to refactor since it lives outside the application codebase
44. How do you call stored procedures from Spring?
    - **Answer:**
       - I call stored procedures using `@Procedure` annotation on a repository method, or `JdbcTemplate` for direct execution
       - In my CDMS project, I used `@Procedure(procedureName = "sp_generate_report")` on a method in my JPA repository and mapped the result to a DTO using a custom `ResultSetExtractor`
       - Three approaches: `@Procedure` annotation (simplest, declaratively mapped to a repository method, but limited to single result sets); `EntityManager.createStoredProcedureQuery()` (more control over input/output parameters and cursors); `JdbcTemplate` (full control, supports multiple result sets, output parameters, and complex parameter mapping)
       - Used `JdbcTemplate` for complex stored procedures that returned multiple result sets or had output parameters, since `@Procedure` does not handle these scenarios well
45. How do you debug slow JPA queries?
    - **Answer:**
       - I enable Hibernate SQL logging with `spring.jpa.show-sql=true` and `spring.jpa.properties.hibernate.format_sql=true`, and statistics logging with `hibernate.generate_statistics=true`
       - In my CDMS project, this helped me identify N+1 queries and missing indexes by analyzing the generated SQL in the logs
       - `hibernate.use_sql_comments=true` prepends a comment to each generated SQL query identifying the origin (e.g., `/* criteria query */` or the `@Query` method name), making it easy to trace which service method generated a slow query
       - Copied the generated SQL from logs into MSSQL Management Studio's execution plan analyzer to identify table scans, missing indexes, expensive key lookups, and parameter sniffing issues
46. Why do we use the `@Transactional` annotation?
    - **Answer:**
       - `@Transactional` tells Spring to manage transaction boundaries for a method — Spring starts a transaction before the method runs, commits it if the method succeeds, and rolls it back if an exception is thrown
       - This keeps a group of database operations atomic: either all of them succeed or none of them do
       - In my inventory service, saving a batch of serial records and updating partner counts in the same method is wrapped in one transaction so a failure mid-way doesn't leave the data half-updated
       - Spring implements this with AOP proxies — the call goes through a CGLIB proxy that opens the transaction before the method, runs the business logic, then commits if the method succeeds or rolls back if an exception is thrown
       - `@Transactional` supports `propagation`, `isolation`, `readOnly`, and `timeout` attributes for fine-grained control; a critical pitfall is self-invocation — calling a `@Transactional` method from within the same class bypasses the proxy, so the transaction is not applied
47. Can `@Transactional` be applied to private or static methods?
    - **Answer:**
       - No. `@Transactional` has no effect on private or static methods
       - Spring applies transactions through a proxy, and a proxy can only intercept public methods that are called from outside the bean
       - Private methods are called directly inside the class (never through the proxy), and static methods cannot be proxied through an instance proxy, so neither gets a transaction
       - The mechanism: Spring creates a JDK dynamic proxy (for interfaces) or CGLIB proxy (for classes) around the bean — only external calls through that proxy are intercepted; private methods are called directly within the class and static methods cannot be proxied through an instance proxy
       - The fix: move transactional logic into a separate `@Service` bean and inject it, or use `TransactionTemplate` for programmatic transaction control when the method must remain private or static
       - This is a common silent bug — the `@Transactional` annotation is simply ignored with no warning, leading developers to believe their transactional logic is protected when it is not
48. What types of exceptions trigger transaction rollback with `@Transactional`?
    - **Answer:**
       - By default, Spring rolls back only on **unchecked** exceptions — `RuntimeException` and `Error`
       - Checked exceptions do **not** trigger a rollback by default, which surprises many developers
       - So a method throwing a checked `SQLException` would still commit
       - To make a checked exception roll back, I add `rollbackFor = Exception.class` (or a specific exception class) on the annotation
       - The default exists because checked exceptions often represent expected business outcomes (like "record not found" or validation failures) where a rollback would be too aggressive — Spring's convention is to only roll back on runtime errors, which indicate unexpected system failures
       - Related attributes: `noRollbackFor` (exceptions that should not trigger rollback), `rollbackForClassName` / `noRollbackForClassName` (string-based matching); used `@Transactional(rollbackFor = Exception.class)` on a method that caught a checked exception and rethrew it, guaranteeing the entire transaction aborted instead of silently committing
