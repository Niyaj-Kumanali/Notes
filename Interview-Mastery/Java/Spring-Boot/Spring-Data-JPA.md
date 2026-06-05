# Spring Data JPA

---

## 1. Executive Summary

### What Is It?
Spring Data JPA is a framework that simplifies data access by automatically generating repository implementations at runtime. It builds on JPA (Jakarta Persistence) and Hibernate, providing a consistent, declarative approach to database access.

### Why Does It Exist?
Before Spring Data JPA, developers wrote:
- JPA boilerplate: `EntityManager`, `persist()`, `merge()`, `find()`, `createQuery()`
- DAO classes with 20+ lines per CRUD method
- Manual pagination, sorting, and query building

Spring Data JPA eliminates all of this with repository interfaces.

### Core Components

| Component | Purpose |
|-----------|---------|
| `JpaRepository<T, ID>` | Full CRUD + JPA-specific features |
| `CrudRepository<T, ID>` | Basic CRUD |
| `PagingAndSortingRepository<T, ID>` | Pagination + sorting |
| `@Entity` | JPA entity (database table) |
| `@Id` | Primary key |
| `@GeneratedValue` | Auto-generated ID |
| `@Column` | Column mapping |
| `@OneToMany`, `@ManyToOne` | Relationships |
| `@Query` | Custom JPQL/SQL query |
| `@Modifying` | UPDATE/DELETE query |
| `@EntityGraph` | Fetch plan |

---

## 2. Core Theory

### Repository Hierarchy

```
Repository<T, ID>          (marker interface)
    │
CrudRepository<T, ID>      (CRUD operations)
    │
PagingAndSortingRepository<T, ID>  (pagination + sorting)
    │
JpaRepository<T, ID>       (flush, batch, JPA-specific)
```

### Query Derivation
Spring Data JPA derives queries from method names:

```java
public interface UserRepository extends JpaRepository<User, Long> {
    
    // SELECT u FROM User u WHERE u.email = ?1
    Optional<User> findByEmail(String email);
    
    // WHERE u.lastName = ?1 AND u.firstName = ?2
    List<User> findByLastNameAndFirstName(String lastName, String firstName);
    
    // WHERE u.active = ?1 ORDER BY u.createdAt DESC
    List<User> findByActiveTrueOrderByCreatedAtDesc();
    
    // WHERE u.createdAt > ?1
    List<User> findByCreatedAtAfter(LocalDateTime date);
    
    // WHERE u.email LIKE %?1%
    List<User> findByEmailContaining(String partial);
    
    // WHERE u.department IN ?1
    List<User> findByDepartmentIn(List<String> departments);
    
    // Page<User> — paginated
    Page<User> findByActive(boolean active, Pageable pageable);
    
    // Slice<User> — lazy pagination (no count query)
    Slice<User> findByRole(String role, Pageable pageable);
}
```

### Page vs Slice

| Aspect | Page | Slice |
|--------|------|-------|
| Total count | Executes COUNT query | No COUNT query |
| next page aware | No | Yes (hasNext()) |
| Performance | Slower (COUNT query) | Faster |
| Use case | Need total pages | Infinite scroll |

### @Query — Custom JPQL

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

## 3. Under-the-Hood

### Repository Proxy Creation
1. Spring scans for interfaces extending `Repository`
2. Creates JDK dynamic proxy at startup
3. Proxy method calls delegated to `SimpleJpaRepository` for CRUD
4. Derived query methods parsed by `PartTreeJpaQuery`
5. `@Query` methods handled by `NamedQuery` or JPA Query

### Entity State Transitions

```
NEW (transient)          — not persisted, no ID
    ↓ persist() / merge()
MANAGED (persistent)     — associated with PersistenceContext
    ↓ close() / clear() / evict()
DETACHED                 — not associated, has ID
    ↓ merge()
MANAGED (persistent)
    ↓ remove()
REMOVED (deleted)
```

### N+1 Query Problem

```java
// BAD: N+1 — 1 query for orders + N queries for each customer's orders
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

---

## 4. Production Code

### 4.1 Audit Fields

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

@Entity
public class User extends Auditable {
    @Id @GeneratedValue
    private Long id;
    // ...
}

// Enable auditing
@Configuration
@EnableJpaAuditing
public class JpaConfig {
    
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext()
            .getAuthentication())
            .map(auth -> auth.getName())
            .orElse("SYSTEM");
    }
}
```

### 4.2 Specification Pattern (Dynamic Queries)

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
@Service
public class UserSearchService {
    private final UserRepository userRepository;
    
    public Page<User> search(UserSearchRequest request) {
        Specification<User> spec = Specification
            .where(UserSpecifications.hasEmail(request.getEmail()))
            .and(UserSpecifications.activeOnly())
            .and(UserSpecifications.createdBetween(request.getFrom(), request.getTo()));
        
        return userRepository.findAll(spec, request.toPageable());
    }
}
```

### 4.3 Projection (DTO, not Entity)

```java
// Interface projection (no full entity load)
public interface UserSummary {
    Long getId();
    String getEmail();
    String getFullName();
}

// Class projection
public record UserDto(Long id, String email, String fullName) {}

public interface UserRepository extends JpaRepository<User, Long> {
    
    // Interface projection
    List<UserSummary> findAllProjectedBy();
    
    // Class projection with JPQL
    @Query("SELECT new com.example.UserDto(u.id, u.email, u.name) FROM User u")
    List<UserDto> findAllDto();
}
```

---

## 5. Common Mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | N+1 queries | Performance disaster | JOIN FETCH / EntityGraph |
| 2 | LazyInitializationException | Session closed | Load eagerly or use transaction |
| 3 | Not using @Transactional on modifying queries | Lazy loading fails | Add @Transactional |
| 4 | Using entities as DTOs | Over-fetching, circular JSON | Use projections |
| 5 | CascadeType.ALL everywhere | Unintended deletes | Explicit cascade types |
| 6 | FetchType.EAGER | Cartesian product joins | LAZY + EntityGraph |
| 7 | Not specifying @Column | Unexpected column names | Explicit @Column definition |
| 8 | Equals/hashCode based on DB ID | Lost in HashSet before persist | Business key or UUID |
| 9 | Large IN clause | SQL performance issues | Batch, use JOIN |
| 10 | Ignoring database indexes | Full table scans | Monitor and index |

---

## 6. Cheat Sheet

```
═══ SPRING DATA JPA ══════════════════════════════════════════

┌─ REPOSITORY METHODS ───────────────────────────────────────┐
│ save(entity)        — persist or merge                      │
│ findById(id)        — find by primary key                   │
│ findAll()           — all entities                          │
│ findAll(Specification) — dynamic query                      │
│ delete(entity)      — remove entity                         │
│ count()             — entity count                          │
│ existsById(id)      — check existence                       │
│ flush()             — force SQL flush                       │
│ saveAndFlush(entity) — save + flush                         │
└─────────────────────────────────────────────────────────────┘

┌─ QUERY KEYWORDS ───────────────────────────────────────────┐
│ And, Or, Is, Equals, Between, LessThan, GreaterThan,       │
│ After, Before, IsNull, IsNotNull, Like, NotLike,           │
│ Containing, StartingWith, EndingWith, OrderBy,             │
│ First, Top, Distinct, Count, Exists, IgnoreCase            │
└─────────────────────────────────────────────────────────────┘

┌─ FETCHING STRATEGIES ──────────────────────────────────────┐
│ Default: @ManyToOne(EAGER), @OneToMany(LAZY)               │
│ Use @EntityGraph or JOIN FETCH for eager loading            │
│ Prefer DTO projections over entities for reads              │
│ Batch fetching: @BatchSize(size = 20)                       │
└─────────────────────────────────────────────────────────────┘

┌─ BEST PRACTICES ───────────────────────────────────────────┐
│ • Always specify fetch strategy explicitly                   │
│ • Use DTO projections for read-only queries                  │
│ • Add @Transactional on modifying operations                 │
│ • Use @Modifying + @Query for bulk operations               │
│ • Handle LazyInitializationException with JOIN FETCH        │
│ • Use Specification for dynamic queries                     │
│ • Avoid CascadeType.ALL unless fully understood             │
└─────────────────────────────────────────────────────────────┘
```
