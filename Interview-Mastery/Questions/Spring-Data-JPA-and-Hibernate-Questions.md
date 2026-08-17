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
   - **If asked more:**
      - I would explain the difference between a specification (JPA) and an implementation (Hibernate)
      - The key JPA annotations
      - How I configured the persistence unit with `spring.jpa.*` properties in `application.yml`
2. What is Hibernate?
   - **Answer:**
      - Hibernate is the most popular JPA implementation that handles ORM, caching, lazy loading, and transaction management
      - In my inventory project, Hibernate was the underlying engine for Spring Data JPA — it generated SQL queries, managed the first-level cache, and handled entity state transitions automatically
   - **If asked more:**
      - I would explain Hibernate-specific features beyond JPA: second-level caching with Redis, Hibernate-specific annotations like `@BatchSize`, `@Fetch`, and HQL query language that differs slightly from JPQL
3. Difference between JPA and Hibernate.
   - **Answer:**
      - JPA is a specification (interface), Hibernate is an implementation (concrete class)
      - I used JPA annotations and interfaces like `EntityManager` and `@Entity` to keep my code portable, while Hibernate ran underneath
      - If I ever needed Hibernate-specific features, I used them sparingly to avoid vendor lock-in
   - **If asked more:**
      - I would explain that Spring Data JPA abstracts both, and I prefer JPA standard APIs for most operations
      - I would also discuss the differences between Hibernate 5 and 6, and how the `hibernate.jpa.compliance` settings can enforce strict JPA compliance
4. What is entity?
   - **Answer:**
      - An entity is a Java class mapped to a database table using `@Entity`
      - In my inventory project, I had `PartnerEntity`, `InventoryRecord`, and `AuditLog` entities with fields annotated with `@Column` to define column names, lengths, and nullability in MSSQL
   - **If asked more:**
      - I would explain entity requirements: no-arg constructor, `@Id` field, getters/setters, and the importance of proper `equals()` and `hashCode()` using business keys to avoid issues in collections and lazy loading proxies
5. What is `@Id`?
   - **Answer:**
      - `@Id` marks a field as the primary key of the entity
      - In my CDMS project, I used `@Id` on the `partnerId` field of `PartnerEntity`, with `@GeneratedValue(strategy = GenerationType.IDENTITY)` to let MSSQL auto-increment the primary key
   - **If asked more:**
      - I would explain composite primary keys using `@IdClass` or `@EmbeddedId`, and how I used a composite key in the inventory entity to uniquely identify records by `partnerId` and `serialNumber`
6. What is generated value?
   - **Answer:**
      - `@GeneratedValue` specifies how the primary key is auto-generated
      - In my projects, I used `GenerationType.IDENTITY` for MSSQL auto-increment columns because it was simpler and more efficient than `SEQUENCE` for my use case
   - **If asked more:**
      - I would explain the four generation strategies: AUTO, IDENTITY, SEQUENCE, and TABLE
      - The performance implications of each (IDENTITY disables batch inserts)
      - Why I chose IDENTITY despite the batch insert limitation since my inserts were single-record operations
7. What is repository?
   - **Answer:**
      - A repository is a Spring Data interface that provides CRUD operations without implementation code
      - In my inventory project, I created `InventoryRepository extends JpaRepository<InventoryRecord, Long>` and automatically got methods like `findAll()`, `save()`, `deleteById()`, and derived query methods
   - **If asked more:**
      - I would explain the repository hierarchy: `Repository` → `CrudRepository` → `PagingAndSortingRepository` → `JpaRepository`, and how Spring Data generates the implementation at runtime using `SimpleJpaRepository`
8. What is Spring Data JPA?
   - **Answer:**
      - Spring Data JPA is a Spring module that reduces JPA boilerplate by providing repository abstractions
      - In my CDMS project, I defined `PartnerRepository extends JpaRepository` and Spring Data JPA automatically provided implementations for CRUD, pagination, and derived queries without writing any DAO code
   - **If asked more:**
      - I would explain how Spring Data JPA uses `EntityManager` under the hood
      - How it translates method names to JPQL queries
      - How I used `@Query` annotations for complex MSSQL queries that couldn't be expressed as derived methods
9. What is `JpaRepository`?
   - **Answer:**
      - `JpaRepository` is a Spring Data interface extending `PagingAndSortingRepository` with JPA-specific methods like `flush()`, `saveAndFlush()`, and `deleteInBatch()`
      - I used `JpaRepository` for all my MSSQL entities in the inventory project because I needed pagination and batch operations
   - **If asked more:**
      - I would explain the additional methods `JpaRepository` provides over `CrudRepository`
      - When to use `PagingAndSortingRepository` instead of `JpaRepository` for simpler use cases where JPA-specific methods are not needed
10. Difference between `CrudRepository` and `JpaRepository`.
   - **Answer:**
      - `CrudRepository` provides basic CRUD methods (save, findById, findAll, delete)
      - `JpaRepository` extends it with JPA-specific methods like `flush()`, `saveAndFlush()`, `deleteInBatch()`, and pagination/sorting
      - I used `JpaRepository` in all my projects since I always needed pagination for the partner listing APIs
   - **If asked more:**
      - I would explain that `CrudRepository` is persistence-technology agnostic, while `JpaRepository` is JPA-specific
      - I would also discuss when to use `ListCrudRepository` (Spring Data 3.x) which returns `List` instead of `Iterable`
11. What is derived query method?
   - **Answer:**
      - Derived query methods allow creating queries by naming methods according to a convention
      - In my inventory project, I defined `findByPartnerIdAndStatus(Long partnerId, String status)` and Spring Data JPA automatically generated the JPQL query based on the method name — no need to write queries manually
   - **If asked more:**
      - I would explain the query derivation keywords: `And`, `Or`, `Between`, `Like`, `OrderBy`, `Top`, and how to use `findBy`, `countBy`, `existsBy`, `deleteBy` prefixes
      - I would also mention the pitfall of too-long method names and when to switch to `@Query`
12. What is JPQL?
   - **Answer:**
      - JPQL (Java Persistence Query Language) is an object-oriented query language similar to SQL but operates on entities instead of tables
      - In my CDMS project, I wrote JPQL queries like `SELECT p FROM PartnerEntity p WHERE p.status = :status` to query partners without database-specific SQL
   - **If asked more:**
      - I would explain the difference between JPQL and SQL — JPQL uses entity names and field names, while SQL uses table and column names
      - I would also show how I used `@Query("SELECT p FROM PartnerEntity p WHERE p.lastSyncDate < :date")` for custom queries
13. Difference between JPQL and native query.
   - **Answer:**
      - JPQL is database-independent and works with entity fields; native queries use raw SQL specific to the database
      - In my CDMS project, I used JPQL for standard queries and native queries with `@Query(value = "EXEC sp_generate_report :partnerId", nativeQuery = true)` for executing MSSQL stored procedures
   - **If asked more:**
      - I would explain the trade-offs: JPQL is portable but limited for database-specific features, native queries give full control but break database portability
      - I used native queries only for stored procedures and complex MSSQL window functions
14. What is entity lifecycle?
   - **Answer:**
      - Entity lifecycle has four states: New (transient), Managed (persistent), Detached, and Removed
      - In my inventory project, entities retrieved via `findById()` were in the managed state — any changes to them were automatically persisted at flush time without calling `save()`
   - **If asked more:**
      - I would explain the state transitions: `persist()` moves New → Managed, `merge()` moves Detached → Managed, `remove()` moves Managed → Removed
      - I encountered detached entity issues when I modified an entity outside a transaction and had to use `merge()`
15. What is persistence context?
   - **Answer:**
      - The persistence context is a first-level cache that tracks entity state changes within a transaction
      - In my inventory project, when I loaded a partner entity inside a `@Transactional` service method, the persistence context kept track of all field modifications and flushed them to MSSQL at commit time
   - **If asked more:**
      - I would explain how the persistence context works as a Map of entity type and `@Id`
      - How it ensures repeatable reads within a transaction
      - How clearing it with `clear()` or `detach()` can help with memory when processing large datasets
16. What is dirty checking?
   - **Answer:**
      - Dirty checking is Hibernate's mechanism to detect changes to managed entities and automatically persist them
      - In my CDMS project, I loaded a `PartnerEntity`, updated its `status` field inside a `@Transactional` method, and Hibernate automatically generated an UPDATE query at flush time without me calling `save()`
   - **If asked more:**
      - I would explain how Hibernate implements dirty checking by comparing snapshots taken at load time with current entity state
      - How `hibernate.dirty_checking` configuration can be tuned for performance with large entities
17. What is first-level cache?
   - **Answer:**
      - The first-level cache is Hibernate's session-level cache within the persistence context
      - In my inventory project, loading the same entity twice by `findById()` inside the same transaction returned the cached object without a second database query, improving performance for repeated lookups
   - **If asked more:**
      - I would explain that the first-level cache is always enabled and cannot be disabled
      - It is scoped to the `EntityManager` (session)
      - How `clear()` and `evict()` can be used to manage memory for large batch operations
18. What is second-level cache?
   - **Answer:**
      - The second-level cache is a session-factory-level cache shared across transactions
      - In my cold-chain project, I configured Hibernate's second-level cache with Redis as the cache store using `hibernate-cache-redis`, caching frequently accessed but rarely changed entity data like gateway configurations
   - **If asked more:**
      - I would explain the difference between first-level and second-level cache
      - How to configure second-level cache with `@Cacheable` and `@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)`
      - The cache concurrency strategies: READ_ONLY, READ_WRITE, NONSTRICT_READ_WRITE, and TRANSACTIONAL
19. What is lazy loading?
   - **Answer:**
      - Lazy loading defers loading of associated entities until they are actually accessed
      - In my CDMS project, the `PartnerEntity` had a `@OneToMany(fetch = FetchType.LAZY)` collection of `OrderEntity` — the orders were loaded from MSSQL only when I accessed `partner.getOrders()`
   - **If asked more:**
      - I would explain how lazy loading works through Hibernate proxies
      - The `LazyInitializationException` that occurs when accessing lazy collections outside a transaction
      - How I solved it using `@Transactional` or `JOIN FETCH` queries
20. What is eager loading?
   - **Answer:**
      - Eager loading loads associated entities immediately when the parent entity is fetched
      - In my inventory project, I used `@ManyToOne(fetch = FetchType.EAGER)` on the `InventoryRecord.partner` field because the partner data was always needed alongside inventory data, avoiding the N+1 problem for that relationship
   - **If asked more:**
      - I would explain the default fetch types: `@OneToMany` and `@ManyToMany` default to LAZY, while `@ManyToOne` and `@OneToOne` default to EAGER
      - I would discuss the performance trade-off: eager loading can cause unnecessary joins and Cartesian product issues
21. What is N+1 query problem?
   - **Answer:**
      - The N+1 problem occurs when 1 query fetches parent entities and N additional queries fetch associated collections
      - In my CDMS project, `partnerRepository.findAll()` followed by accessing `partner.getOrders()` in a loop triggered N separate queries — making the report generation extremely slow
   - **If asked more:**
      - I would explain how to detect N+1 by enabling Hibernate SQL logging (`spring.jpa.show-sql=true`)
      - How I identified the N+1 problem in the CDMS project by seeing hundreds of SELECT queries in the logs for what should have been a single query
22. How do you solve N+1 query problem?
   - **Answer:**
      - I solve N+1 using `JOIN FETCH` in JPQL or `@EntityGraph`
      - In my CDMS project, I changed the query from `SELECT p FROM PartnerEntity p` to `SELECT p FROM PartnerEntity p JOIN FETCH p.orders`, which fetched partners and orders in a single SQL query using an INNER JOIN
   - **If asked more:**
      - I would explain the different solutions: `JOIN FETCH` (eager fetch in query), `@EntityGraph` (declarative), `@BatchSize` (lazy batch loading), and Hibernate's `batch_fetch_size` configuration
      - I would also discuss the trade-off: JOIN FETCH can cause duplicate results and Cartesian products with multiple collections
23. What is fetch join?
   - **Answer:**
      - Fetch join is a JPQL feature that loads associations in a single query using JOIN
      - I used `SELECT p FROM PartnerEntity p JOIN FETCH p.orders` in my inventory project to load partners along with their inventory records in one SQL query, completely avoiding the N+1 problem
   - **If asked more:**
      - I would explain that fetch join does not change the fetch type of the association but overrides it for that specific query
      - I would also mention that `LEFT JOIN FETCH` should be used when the association might be null
      - That multiple collections require `SET` instead of `LIST` to avoid duplicates
24. What is entity graph?
   - **Answer:**
      - Entity graphs allow defining fetch plans dynamically using `@NamedEntityGraph` or `@EntityGraph`
      - In my CDMS project, I defined `@NamedEntityGraph(name = "Partner.withOrders", attributeNodes = @NamedAttributeNode("orders"))` and used `@EntityGraph("Partner.withOrders")` on the repository method for flexible fetch strategies
   - **If asked more:**
      - I would explain the difference between `EntityGraphType.FETCH` (replaces default fetch plan with only specified attributes) and `EntityGraphType.LOAD` (adds specified attributes to the default plan)
      - How entity graphs solve N+1 without rewriting JPQL queries
25. What is pagination?
   - **Answer:**
      - Pagination divides large result sets into smaller pages
      - In my CDMS project, I used `Pageable` parameter in repository methods: `Page<PartnerEntity> findAll(Pageable pageable)`
      - The API returned page number, size, total elements, and total pages, allowing the React frontend to display paginated tables
   - **If asked more:**
      - I would explain `PageRequest.of(page, size, Sort)`
      - How Spring Data JPA translates pagination to database-specific SQL (using `OFFSET` and `FETCH NEXT` in MSSQL)
      - The performance downside of large offsets — I switched to keyset pagination for very large datasets
26. What is sorting?
   - **Answer:**
      - Sorting orders query results by specified fields
      - In my inventory project, I used `Sort.by("partnerName").ascending()` in the repository method or passed `Sort` through `Pageable`
      - For multi-field sorting, I used `Sort.by(Order.asc("partnerName"), Order.desc("createdDate"))`
   - **If asked more:**
      - I would explain the difference between dynamic sorting via `Pageable` parameter (user-controlled) and static sorting via `@OrderBy` annotation on entity associations
      - I would also mention the SQL injection risk of field sorting with `Sort.by()` — Spring validates it, but custom implementations need care
27. What is transaction management?
   - **Answer:**
      - Transaction management ensures a group of database operations either all succeed or all fail
      - In my inventory project, I used Spring's declarative transaction management with `@Transactional` on service methods to ensure atomicity — if one validation step failed, the entire batch update was rolled back
   - **If asked more:**
      - I would explain ACID properties
      - The difference between programmatic (TransactionTemplate) and declarative (@Transactional) transaction management
      - How Spring wraps the service in a proxy to handle begin, commit, and rollback automatically
28. What is `@Transactional`?
   - **Answer:**
      - `@Transactional` ensures a set of database operations either all succeed or all roll back
      - In my inventory project, I used it on the batch validation service so that if any partner record failed validation, the entire batch was rolled back to maintain data consistency
   - **If asked more:**
      - I would explain the attributes: `propagation`, `isolation`, `timeout`, `readOnly`, and `rollbackFor`
      - I set `readOnly = true` on read-only methods for performance optimization and `rollbackFor = CustomException.class` to specify which exceptions trigger rollback
29. How does `@Transactional` work internally?
   - **Answer:**
      - Spring uses AOP proxies to wrap `@Transactional` methods
      - When a method annotated with `@Transactional` is called, Spring creates a transaction before method execution and commits or rolls back based on the outcome
      - In my projects, I confirmed this works via CGLIB proxy — which means self-invocation bypasses the proxy and the annotation is ignored
   - **If asked more:**
      - I would explain the `TransactionInterceptor` and `PlatformTransactionManager` chain
      - How `TransactionAspectSupport` manages transaction status
      - The difference between JDBC and JPA transaction managers — I used `JpaTransactionManager` for my MSSQL entities
30. What is transaction propagation?
   - **Answer:**
      - Transaction propagation defines how transactions behave when a transactional method calls another transactional method
      - In my CDMS project, I used `REQUIRED` (default) for most methods, meaning they join the existing transaction
      - For the audit logging method, I used `REQUIRES_NEW` so the audit log was saved even if the main transaction rolled back
   - **If asked more:**
      - I would explain the seven propagation types: REQUIRED, SUPPORTS, MANDATORY, REQUIRES_NEW, NOT_SUPPORTED, NEVER, and NESTED
      - I used REQUIRES_NEW in the inventory project for the notification service to send alerts independently of the main transaction outcome
31. What is transaction isolation?
   - **Answer:**
      - Transaction isolation defines how concurrent transactions interact
      - In my inventory project, I used `@Transactional(isolation = Isolation.READ_COMMITTED)` which is MSSQL's default and prevents dirty reads
      - For the inventory calculation method, I considered `REPEATABLE_READ` to prevent phantom reads during stock computation
   - **If asked more:**
      - I would explain the four isolation levels: READ_UNCOMMITTED, READ_COMMITTED, REPEATABLE_READ, and SERIALIZABLE
      - The problems they prevent: dirty read, non-repeatable read, and phantom read
      - I would also discuss the performance trade-off of higher isolation levels
32. What is rollback behavior?
   - **Answer:**
      - By default, `@Transactional` rolls back on unchecked exceptions (RuntimeException) and commits on checked exceptions
      - In my projects, I customized this with `rollbackFor = {DataAccessException.class, CustomValidationException.class}` to ensure certain checked exceptions also triggered rollback
   - **If asked more:**
      - I would explain `noRollbackFor` for cases where rollback should not happen
      - How I configured declarative rollback rules in XML or Java configuration for finer control over which exceptions roll back or commit the transaction
33. Why does self-invocation break `@Transactional`?
   - **Answer:**
      - Self-invocation bypasses the Spring AOP proxy because the call happens within the same class, not through the injected proxy
      - In my inventory project, I accidentally faced this when method A (annotated with `@Transactional`) called method B (also `@Transactional`) in the same service — the transaction for B was ignored
   - **If asked more:**
      - I would explain the proxy pattern and how Spring creates a CGLIB proxy (or JDK proxy for interfaces)
      - Solutions I used: self-injecting the proxy via `ApplicationContextAware` or `@Lazy`, extracting transactional methods into a separate service, or using `TransactionTemplate` programmatically
34. What is optimistic locking?
   - **Answer:**
      - Optimistic locking assumes conflicts are rare and checks for them at commit time using a version column
      - In my inventory project, I added `@Version` on the `InventoryRecord` entity to prevent concurrent updates — if two users tried to update the same record, the second commit threw `OptimisticLockException`
   - **If asked more:**
      - I would explain how optimistic locking works without database locks (just a version check)
      - The `StaleObjectStateException`
      - How I handled the exception in the service layer by retrying the operation or informing the user that the data was modified by someone else
35. What is pessimistic locking?
   - **Answer:**
      - Pessimistic locking locks the database row at read time to prevent other transactions from modifying it
      - In my CDMS project, I used `@Lock(LockModeType.PESSIMISTIC_WRITE)` on the repository method that calculated partner inventory, ensuring no other transaction could modify the data during the calculation
   - **If asked more:**
      - I would explain the lock types: PESSIMISTIC_READ (shared lock), PESSIMISTIC_WRITE (exclusive lock), and how they translate to `SELECT ... FOR UPDATE` in MSSQL
      - I would discuss the trade-off: pessimistic locking prevents conflicts but reduces concurrency compared to optimistic locking
36. What is `@Version`?
   - **Answer:**
      - `@Version` is a JPA annotation that enables optimistic locking by maintaining a version number
      - In my inventory project, I added `@Version private Long version;` to the `InventoryRecord` entity
      - Hibernate automatically incremented the version on every update and checked it before committing, throwing `OptimisticLockException` on conflict
   - **If asked more:**
      - I would explain that `@Version` works with any numeric type, `Timestamp`, or `Instant`
      - I would also discuss the common mistake of forgetting to include `@Version` and the result — silent data overwrites in concurrent scenarios
37. What are cascade types?
   - **Answer:**
      - Cascade types determine which entity operations propagate to associated entities
      - In my CDMS project, I used `cascade = CascadeType.PERSIST` on the `@OneToMany(mappedBy = "partner")` orders collection so that saving a partner also saved its orders without separate save calls
   - **If asked more:**
      - I would explain the cascade types: ALL, PERSIST, MERGE, REMOVE, REFRESH, DETACH, and the common mistake of using `CascadeType.ALL` on `@ManyToMany` which causes unintended deletes
      - I used only `PERSIST` and `MERGE` in my projects to avoid accidental removals
38. What is orphan removal?
   - **Answer:**
      - Orphan removal automatically deletes child entities when they are removed from the parent's collection
      - In my inventory project, I used `orphanRemoval = true` on the `@OneToMany` partner-orders mapping so that when an order was removed from the partner's order list, it was automatically deleted from MSSQL
   - **If asked more:**
      - I would explain the difference between `orphanRemoval = true` and `cascade = CascadeType.REMOVE` — orphan removal removes children removed from the collection, while cascade REMOVE deletes children when the parent is deleted
      - I used both for different scenarios
39. Difference between `save()` and `saveAndFlush()`.
   - **Answer:**
      - `save()` persists the entity and returns it, but does not immediately flush to the database — the actual INSERT/UPDATE happens at transaction commit
      - `saveAndFlush()` immediately flushes the SQL to the database
      - In my inventory project, I used `saveAndFlush()` when I needed the generated ID immediately for logging purposes
   - **If asked more:**
      - I would explain that `save()` may batch multiple operations before flushing, improving performance
      - `saveAndFlush()` is useful when the next operation depends on the entity being persisted (like calling a stored procedure that needs the entity's ID)
40. What is batch insert?
   - **Answer:**
      - Batch insert groups multiple INSERT statements into a single database round-trip for performance
      - In my CDMS project, I configured `spring.jpa.properties.hibernate.jdbc.batch_size=50` and `hibernate.order_inserts=true` so that inserting 500 partner records in a loop triggered only 10 batch inserts instead of 500 individual queries
   - **If asked more:**
      - I would explain the requirements for batch inserts: IDENTITY generator disables batch inserts (so I used SEQUENCE or TABLE generator)
      - `order_inserts` and `order_updates` must be enabled
      - `rewriteBatchedStatements=true` for MSSQL
      - I learned this when testing batch inserts for the inventory import feature
41. How do you improve JPA performance?
   - **Answer:**
      - I improve JPA performance by enabling batch operations, using fetch joins to avoid N+1 queries, configuring appropriate fetch types, enabling Hibernate query cache for read-heavy data, and monitoring slow queries via `spring.jpa.properties.hibernate.generate_statistics=true`
   - **If asked more:**
      - I would explain specific optimizations I applied: setting `batch_size=50` for bulk inserts, using `@BatchSize` on lazy collections, avoiding `select *` by using projection DTOs, and switching to native SQL or stored procedures for complex reporting queries that couldn't be optimized through JPA
42. When should you use JDBC instead of JPA?
   - **Answer:**
      - Use JDBC when performance is critical for bulk operations, complex reporting queries, or when executing stored procedures
      - In my CDMS project, I used `JdbcTemplate` for the stored procedure execution because it was simpler and faster than mapping complex result sets to entities for the reporting use case
   - **If asked more:**
      - I would explain that JPA overhead (persistence context, dirty checking, lazy loading) becomes significant with thousands of entities in memory
      - For the CDMS nightly batch that processed millions of records, JPA would have caused memory issues — JDBC was the right choice
43. When should you use stored procedures?
   - **Answer:**
      - Use stored procedures for complex reporting logic, long-running batch operations, or when you need database-specific features that cannot be expressed in JPA
      - In my CDMS project, the nightly report generation was implemented as an MSSQL stored procedure because it involved complex joins, aggregations, and conditional logic that was more efficient in SQL
   - **If asked more:**
      - I would explain the benefits: reduced network round-trips (logic runs in the database), better performance for set-based operations, and the ability to use MSSQL-specific features like window functions
      - The trade-off is testing complexity and database portability
44. How do you call stored procedures from Spring?
   - **Answer:**
      - I call stored procedures using `@Procedure` annotation on a repository method, or `JdbcTemplate` for direct execution
      - In my CDMS project, I used `@Procedure(procedureName = "sp_generate_report")` on a method in my JPA repository and mapped the result to a DTO using a custom `ResultSetExtractor`
   - **If asked more:**
      - I would explain the three approaches: `@Procedure` annotation (simplest, but limited to single result sets), `EntityManager.createStoredProcedureQuery()` (more control), and `JdbcTemplate` (full control)
      - I used `JdbcTemplate` for complex stored procedures that returned multiple result sets or had output parameters
45. How do you debug slow JPA queries?
   - **Answer:**
      - I enable Hibernate SQL logging with `spring.jpa.show-sql=true` and `spring.jpa.properties.hibernate.format_sql=true`, and statistics logging with `hibernate.generate_statistics=true`
      - In my CDMS project, this helped me identify N+1 queries and missing indexes by analyzing the generated SQL in the logs
   - **If asked more:**
      - I would explain how I used `spring.jpa.properties.hibernate.use_sql_comments=true` to trace which service method generated each query
      - How I copied slow queries from logs to MSSQL Management Studio to analyze execution plans and identify missing indexes or expensive joins
46. Why do we use the `@Transactional` annotation?
   - **Answer:**
      - `@Transactional` tells Spring to manage transaction boundaries for a method — Spring starts a transaction before the method runs, commits it if the method succeeds, and rolls it back if an exception is thrown
      - This keeps a group of database operations atomic: either all of them succeed or none of them do
      - In my inventory service, saving a batch of serial records and updating partner counts in the same method is wrapped in one transaction so a failure mid-way doesn't leave the data half-updated
   - **If asked more:**
      - I would explain that Spring implements this with AOP proxies — the call goes through a proxy that opens the transaction, runs the method, then commits/rolls back
      - I can mention that `@Transactional` also supports `propagation`, `isolation`, `readOnly`, and `timeout` attributes, and warn about self-invocation: calling a `@Transactional` method from within the same class bypasses the proxy, so the transaction is not applied
47. Can `@Transactional` be applied to private or static methods?
   - **Answer:**
      - No. `@Transactional` has no effect on private or static methods
      - Spring applies transactions through a proxy, and a proxy can only intercept public methods that are called from outside the bean
      - Private methods are called directly inside the class (never through the proxy), and static methods cannot be proxied through an instance proxy, so neither gets a transaction
   - **If asked more:**
      - I would explain the mechanism: Spring creates a JDK dynamic proxy (or CGLIB proxy) around the bean, and only external calls through that proxy are intercepted
      - I would recommend moving transactional logic into a separate Spring bean and injecting it, or using `TransactionTemplate` for programmatic control when the method must remain private or static
      - This is a common bug I watch for — developers annotating a private helper method and expecting it to be transactional
48. What types of exceptions trigger transaction rollback with `@Transactional`?
   - **Answer:**
      - By default, Spring rolls back only on **unchecked** exceptions — `RuntimeException` and `Error`
      - Checked exceptions do **not** trigger a rollback by default, which surprises many developers
      - So a method throwing a checked `SQLException` would still commit
      - To make a checked exception roll back, I add `rollbackFor = Exception.class` (or a specific exception class) on the annotation
   - **If asked more:**
      - I would explain why this default exists: checked exceptions often represent expected business outcomes (like "record not found") where a rollback would be too aggressive, so Spring's convention is to only roll back on runtime errors
      - I can mention the related attributes — `noRollbackFor`, `rollbackForClassName`, `noRollbackForClassName` — and give a concrete example from my service layer where I used `@Transactional(rollbackFor = Exception.class)` on a method that caught a checked exception and rethrew it, to guarantee the whole transaction aborted

