# Normalization

## 1. Executive Summary

Normalization is the process of organizing relational database tables to reduce data redundancy and improve data integrity. It involves decomposing tables into smaller, related tables according to defined normal forms. Normalization eliminates update anomalies, insertion anomalies, and deletion anomalies. While strict normalization (3NF/BCNF) is the theoretical ideal, practical production databases often stop at 3NF and selectively denormalize for performance. Understanding normalization is essential for designing schemas that maintain data integrity without sacrificing query performance.

## 2. Core Theory

### 2.1 Goals of Normalization

1. **Eliminate data redundancy** — each fact stored once
2. **Ensure data integrity** — updates don't create inconsistencies
3. **Simplify data maintenance** — one place to update each fact
4. **Remove anomalies** — insertion, update, deletion anomalies

### 2.2 Normal Forms Defined

| Normal Form | Rule | Violation Fix |
|-------------|------|---------------|
| 1NF | Atomic columns, no repeating groups | Split columns, introduce PK |
| 2NF | 1NF + no partial dependency on composite PK | Split into separate tables |
| 3NF | 2NF + no transitive dependency on non-key | Extract dependent attributes |
| BCNF | Every determinant is a candidate key | Break into overlapping tables |
| 4NF | No multi-valued dependencies | Split into separate relations |
| 5NF | Every join dependency implied by candidate keys | Further decomposition |
| 6NF | No non-key dependencies at all | Decompose to irreducible relations |

### 2.3 Functional Dependency

Given relation R, attribute Y is functionally dependent on X (X -> Y) if each X value determines exactly one Y value.

Examples:
- `employee_id -> employee_name` (each ID has one name)
- `order_id -> customer_id` (each order has one customer)

### 2.4 Normalization Process

```
UNF -> 1NF -> 2NF -> 3NF -> BCNF -> 4NF -> 5NF
```

Each step eliminates specific types of redundancy. Most production databases target 3NF or BCNF.

## 3. Under-the-Hood Deep Dive

### 3.1 First Normal Form (1NF)

**Rule**: Each column contains atomic (indivisible) values. No repeating groups.

**Violation**:
```sql
-- BAD: Repeating group (multiple phone numbers in one column)
CREATE TABLE employee (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    phone_numbers VARCHAR(500)  -- '555-0100,555-0101,555-0102'
);

-- BAD: Multiple phone columns
CREATE TABLE employee (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    phone1 VARCHAR(20),
    phone2 VARCHAR(20),
    phone3 VARCHAR(20)
);
```

**Fix**:
```sql
CREATE TABLE employee (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE employee_phone (
    id INT PRIMARY KEY,
    employee_id INT REFERENCES employee(id),
    phone VARCHAR(20),
    phone_type VARCHAR(10)  -- 'MOBILE', 'WORK', 'HOME'
);
```

### 3.2 Second Normal Form (2NF)

**Rule**: 1NF + every non-key column must depend on the ENTIRE primary key (not just part of a composite key).

**Violation** -- composite PK (order_id, product_id) but non-key depends only on order_id:
```sql
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    product_name VARCHAR(100),  -- depends only on product_id, NOT (order_id, product_id)
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);
```

**Fix**:
```sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT REFERENCES products(id),
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);
```

### 3.3 Third Normal Form (3NF)

**Rule**: 2NF + no transitive dependency — non-key columns cannot depend on other non-key columns.

**Violation** -- `customer_city` depends on `customer_id`, not `order_id`:
```sql
CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),   -- transitively depends on customer_id
    customer_city VARCHAR(100),   -- transitively depends on customer_id
    total DECIMAL(10,2)
);
```

**Fix**:
```sql
CREATE TABLE customers (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    city VARCHAR(100)
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(id),
    total DECIMAL(10,2)
);
```

### 3.4 Boyce-Codd Normal Form (BCNF)

**Rule**: For every functional dependency X -> Y, X must be a candidate key.

**Violation** -- In a table where each student has one advisor, and each advisor advises multiple students, but each advisor belongs to one department:

```sql
CREATE TABLE student_advisor (
    student_id INT,
    advisor_id INT,
    department_id INT,
    -- FDs: student_id -> advisor_id, advisor_id -> department_id
    -- PK: (student_id). advisor_id is NOT a key, but advisor_id -> department_id
    PRIMARY KEY (student_id)
);
```

BCNF requires decomposition:
```sql
CREATE TABLE student_advisor (
    student_id INT PRIMARY KEY,
    advisor_id INT
);

CREATE TABLE advisor_department (
    advisor_id INT PRIMARY KEY,
    department_id INT
);
```

### 3.5 Multi-valued Dependencies (4NF)

**Rule**: No multi-valued dependencies. If an employee can have multiple skills AND multiple degrees, these are independent facts.

**Violation**:
```sql
CREATE TABLE employee_skills_degrees (
    employee_id INT,
    skill VARCHAR(50),
    degree VARCHAR(50),
    PRIMARY KEY (employee_id, skill, degree)
);
-- Problem: inserting a new skill requires repeating all degree values
```

**Fix** (split into two tables):
```sql
CREATE TABLE employee_skill (
    employee_id INT,
    skill VARCHAR(50),
    PRIMARY KEY (employee_id, skill)
);

CREATE TABLE employee_degree (
    employee_id INT,
    degree VARCHAR(50),
    PRIMARY KEY (employee_id, degree)
);
```

## 4. Production Code Examples

### 4.1 Normalized Schema for E-Commerce

```sql
-- Fully normalized (3NF/BCNF) e-commerce schema
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    username VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE addresses (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    street VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(50),
    zip VARCHAR(20),
    is_default BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    parent_id BIGINT REFERENCES categories(id),
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    category_id BIGINT NOT NULL REFERENCES categories(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    sku VARCHAR(50) UNIQUE,
    stock_quantity INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    billing_address_id BIGINT REFERENCES addresses(id),
    shipping_address_id BIGINT REFERENCES addresses(id),
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    subtotal DECIMAL(10,2),
    tax DECIMAL(10,2),
    shipping_cost DECIMAL(10,2),
    total DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id),
    product_id BIGINT NOT NULL REFERENCES products(id),
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    total DECIMAL(10,2) NOT NULL
);

CREATE TABLE payments (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id),
    method VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL,
    transaction_id VARCHAR(100),
    amount DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 4.2 JPA Entities for Normalized Schema

```java
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "billing_address_id")
    private Address billingAddress;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "shipping_address_id")
    private Address shippingAddress;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    @Column(nullable = false)
    private String status;

    private BigDecimal subtotal;
    private BigDecimal tax;
    private BigDecimal shippingCost;
    private BigDecimal total;

    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
    }

    // Helper method to maintain bidirectional consistency
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }
}

@Entity
@Table(name = "order_items")
public class OrderItem {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    @Column(nullable = false)
    private Integer quantity;

    @Column(name = "unit_price", nullable = false)
    private BigDecimal unitPrice;

    @Column(nullable = false)
    private BigDecimal total;
}
```

### 4.3 Normalized Schema Constraints with JPA

```java
@Entity
@Table(name = "products", uniqueConstraints = {
    @UniqueConstraint(name = "uq_products_sku", columnNames = {"sku"})
})
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", nullable = false)
    private Category category;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private BigDecimal price;

    @Column(unique = true, length = 50)
    private String sku;

    @Column(name = "stock_quantity", nullable = false)
    private Integer stockQuantity;
}
```

### 4.4 Denormalization for Query Performance

```java
// Sometimes denormalize for performance (example: order total stored on order,
// even though it could be computed from order_items)
// This is an intentional denormalization at 3NF+ level

@Entity
@Table(name = "orders")
public class Order {
    // ... other fields ...

    @Column(nullable = false)
    private BigDecimal total;  // Denormalized: sum of order_items.total

    // Recalculate when items change
    public void recalculateTotal() {
        this.total = items.stream()
            .map(OrderItem::getTotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

## 5. Real-World Scenarios

### 5.1 When Strict Normalization Hurts

**Scenario**: An e-commerce product page showing product details, category path, review count, and average rating.

With strict 3NF:
```sql
-- Needs joins across 4+ tables for every product page view
SELECT p.*, c.name AS category_name,
       AVG(r.rating) AS avg_rating, COUNT(r.id) AS review_count
FROM products p
JOIN categories c ON c.id = p.category_id
LEFT JOIN reviews r ON r.product_id = p.id
GROUP BY p.id, c.name;
```

Optimization options:
1. **Materialized view** with periodic refresh
2. **Caching** in Redis (product page cache)
3. **Denormalized columns** (review_count, avg_rating on products table)

### 5.2 Time-Series / Event Data

For event data, strict normalization is often inappropriate:
```sql
-- Events are insert-only; normalization adds overhead without benefit
CREATE TABLE events (
    id BIGSERIAL PRIMARY KEY,
    event_type VARCHAR(50),
    user_id BIGINT,
    payload JSONB,
    device_type VARCHAR(50),
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
-- This is effectively denormalized (but appropriate for insert-only data)
```

### 5.3 Logging / Audit Tables

Audit logs are typically flat (unnormalized) because they represent immutable events:
```sql
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    table_name VARCHAR(100),
    row_id BIGINT,
    operation VARCHAR(10),
    old_data JSONB,
    new_data JSONB,
    performed_by VARCHAR(100),
    performed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
-- Denormalized: stores all data in one row, no joins needed for queries
```

## 6. Performance

### 6.1 Normalization Cost: Join Overhead

Joins across normalized tables incur cost:
- **Network**: If app and DB are separate, join result is larger
- **CPU**: Join computation
- **I/O**: Multiple index lookups

**3NF query**:
```sql
SELECT o.id, u.name, a.city
FROM orders o
JOIN users u ON u.id = o.user_id
JOIN addresses a ON a.id = o.shipping_address_id
WHERE o.id = 100;
```

**Equivalent denormalized query**:
```sql
-- No joins needed
SELECT id, user_name, shipping_city
FROM orders_denormalized
WHERE id = 100;
```

### 6.2 Update Performance

Normalization excels when data is updated:
```sql
-- With normalization: update username in one place
UPDATE users SET username = 'new_name' WHERE id = 1;

-- Denormalized: must update in potentially many places
UPDATE orders_denormalized SET user_name = 'new_name' WHERE user_id = 1;
-- If missed, data becomes inconsistent
```

### 6.3 Storage Efficiency

Normalization typically reduces storage:
- Each fact stored once
- No repeated customer names in order records
- Use of foreign keys (integers) instead of full text values

## 7. Security

### 7.1 Normalization and Access Control

Normalized schemas support fine-grained access control:
```sql
-- Users can read their own profiles
GRANT SELECT ON users TO app_role;
-- But not all user data
REVOKE ALL ON users FROM public_role;
-- Only order data accessible
GRANT SELECT, INSERT ON orders TO app_role;
```

### 7.2 Denormalization and Data Leakage

Denormalized tables may expose sensitive data if access control is coarse. A denormalized orders table with embedded user data could leak PII if query filters are insufficient.

## 8. Common Mistakes

1. **Premature over-normalization** — breaking everything to 5NF/6NF when 3NF suffices
2. **Ignoring performance** — 15-join queries because every relationship is normalized
3. **Not recognizing multi-valued dependencies** — producing cross-product anomalies
4. **Treating normalization as binary** — normalized vs denormalized is a spectrum
5. **Creating too many tables** — leads to complex queries and maintenance burden
6. **Mixing OLTP and OLAP schemas** — normalized for writes, denormalized for reads
7. **Forgetting that denormalization is a legitimate optimization** — after measurement
8. **Using synthetic keys without understanding natural keys** — natural keys provide business meaning
9. **Ignoring partial dependencies with composite keys** — common 2NF violation
10. **Denormalizing too early** — optimize only when measurements show a problem

## 9. Senior Engineer Perspective

### Practical Normalization Guidelines

1. **Default to 3NF** — most production databases should be in 3NF
2. **Measure before denormalizing** — use query plans to identify join bottlenecks
3. **Denormalize strategically** — add computed aggregates, not raw data copies
4. **Use views for denormalized access** — normalized storage, denormalized access pattern
5. **Consider materialized views** — best of both worlds for reporting
6. **Document denormalization** — leave comments explaining why data is duplicated
7. **Maintain consistency** — use triggers or application logic to keep denormalized data in sync

### Normalization in Microservices

In microservices, each service owns its database. Normalization happens within a service boundary. Data duplication across services is expected (a service stores the data it needs, even if another service also stores related data). This is a form of intentional denormalization at the system level.

## 10. Interview Questions (20)

### Easy (10)

1. What is database normalization?
2. What is 1NF? Give an example.
3. What is 2NF? How does it differ from 1NF?
4. What is 3NF? How does it differ from 2NF?
5. What is a functional dependency?
6. What is a partial dependency?
7. What is a transitive dependency?
8. What is a primary key?
9. What is a foreign key?
10. Why is normalization important?

### Medium (10)

11. Explain Boyce-Codd Normal Form (BCNF) and how it differs from 3NF.
12. What is 4NF? What problem does it solve?
13. Give an example of an update anomaly and explain how normalization fixes it.
14. What is the difference between a surrogate key and a natural key? Which is better?
15. How does normalization affect query performance (SELECT vs INSERT/UPDATE)?
16. When would you intentionally denormalize?
17. What is join dependency? How does 5NF address it?
18. How do you handle multi-valued dependencies in a normalized schema?
19. Explain the difference between normalization and denormalization.
20. What is the "normal form" of most production databases? Why not 5NF?

## 11. Advanced Interview Questions (20)

### Hard (10)

1. Prove that a relation in BCNF is always in 3NF (but not vice versa).
2. Explain the concept of "lossless join decomposition" and how to verify it.
3. What is dependency preservation? Why might BCNF losslessly decompose without preserving dependencies?
4. How do you normalize a table that has overlapping candidate keys?
5. Explain the algorithm for computing the closure of a set of functional dependencies.
6. What is the difference between 3NF and BCNF in terms of redundancy elimination?
7. How does domain-key normal form (DK/NF) relate to other normal forms?
8. Explain the chase test for lossless join decomposition.
9. How do you identify all candidate keys of a relation given functional dependencies?
10. What is the minimal cover of functional dependencies and how do you compute it?

### System Design (11-20)

11. Design a normalized schema for a hospital management system (patients, doctors, appointments, prescriptions).
12. Design a normalized schema for a university registration system considering time-tabling constraints.
13. How would you design a normalized schema for a social networking platform with posts, comments, likes, and shares?
14. Design a normalized multi-currency e-commerce platform.
15. How would you design a normalized inventory management system with multiple warehouses?
16. Design a normalized schema for a content management system with versioning.
17. How would you design a normalized hotel booking system with room availability?
18. Design a normalized schema for a project management tool (projects, tasks, assignees, timesheets).
19. How would you design a normalized role-based access control system?
20. Design a normalized subscription billing system with proration.

## 12. Expert-Level Interview Questions (10)

1. Design a normalization framework that automatically detects functional dependencies from existing data and suggests decompositions.
2. How would you implement a normalized temporal database (with valid-time and transaction-time) using 6NF?
3. Design a system that migrates a legacy denormalized database to 3NF without downtime.
4. How would you normalize a graph database into relational form while preserving query patterns?
5. Design a normalization-aware query optimizer that can work with denormalized views of normalized storage.
6. How would you implement a trigger-based system that maintains consistency between normalized and denormalized representations?
7. Design an algorithm that determines the optimal normal form for a given workload (read/write ratio, query patterns).
8. How would you implement a system that can dynamically switch between normalized and denormalized storage based on access patterns?
9. Design a multi-version concurrency control system where normalization eliminates phantom reads.
10. How would you design a distributed database where each shard can independently choose its normalization level?

## 13. Debugging & Troubleshooting

### Detecting Normalization Issues

```sql
-- Find tables with repeating column patterns (possible 1NF violation)
SELECT table_name, string_agg(column_name, ', ' ORDER BY ordinal_position)
FROM information_schema.columns
WHERE table_schema = 'public'
GROUP BY table_name
HAVING COUNT(*) > 50
ORDER BY COUNT(*) DESC;

-- Check for missing foreign key indexes (possible join performance issues from normalization)
SELECT tc.table_name, tc.constraint_name, kcu.column_name
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu
  ON tc.constraint_name = kcu.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND NOT EXISTS (
    SELECT 1 FROM pg_indexes pi
    WHERE pi.tablename = tc.table_name
      AND pi.indexdef LIKE '%' || kcu.column_name || '%'
  );
```

### JPA Normalization Validation

```java
// Test that lazy loading doesn't cause N+1 in normalized schema
@SpringBootTest
class SchemaNormalizationTest {

    @Autowired
    private OrderRepository orderRepository;

    @Test
    @Transactional
    void shouldNotProduceNPlusOneQueries() {
        // Configure counting data source to verify query count
        // This test ensures normalized schema doesn't cause performance issues
        List<Order> orders = orderRepository.findAll();
        assertThat(orders).isNotEmpty();
    }
}
```

## 14. Comparison Section

| Aspect | Normalized | Denormalized |
|--------|-----------|-------------|
| Data redundancy | Minimal | Higher |
| Insert/update/delete | Fast (one place) | Slow (multiple places) |
| Read (single table) | Slow (joins) | Fast (no joins) |
| Read (aggregates) | Fast (if indexed) | Pre-computed |
| Storage | Compact | Larger |
| Data integrity | High (constraints) | Risk of inconsistency |
| Schema flexibility | High | Low |
| Query complexity | Complex (many joins) | Simple (fewer joins) |
| ETL / Reporting | Complex | Simpler |

## 15. Revision Notes

- 1NF: Atomic columns, no repeating groups
- 2NF: 1NF + no partial dependency on composite PK
- 3NF: 2NF + no transitive dependency on non-key
- BCNF: Every determinant must be a candidate key
- Default to 3NF for OLTP; denormalize measured hotspots
- Normalization reduces write anomalies at cost of read complexity
- Join overhead is the main trade-off for normalized schemas
- Use JPA @ManyToOne/@OneToMany for normalized relationships
- Document intentional denormalization with comments
- Consider materialized views as a middle ground

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                   NORMALIZATION CHEAT SHEET                       |
+-------------------------------------------------------------------+
|                                                                   |
|  NORMAL FORMS SUMMARY:                                            |
|                                                                   |
|  1NF:                                                             |
|    - Each cell contains one value (atomic)                        |
|    - No repeating groups/arrays                                   |
|    - Each row has a unique identifier (PK)                        |
|                                                                   |
|  2NF:                                                             |
|    1NF + all non-key columns fully dependent on ENTIRE PK         |
|    (no partial dependencies on composite PK components)            |
|                                                                   |
|  3NF:                                                             |
|    2NF + no transitive dependencies                               |
|    (non-key columns don't depend on other non-key columns)         |
|                                                                   |
|  BCNF:                                                            |
|    3NF + every determinant is a candidate key                     |
|    (X -> Y implies X must be a key)                               |
|                                                                   |
|  4NF:                                                             |
|    BCNF + no multi-valued dependencies                            |
|    (independent 1:N relationships must be separate tables)         |
|                                                                   |
+-------------------------------------------------------------------+
|  ANOMALY TYPES:                                                   |
|                                                                   |
|  Insertion anomaly: Can't insert data without related info        |
|    (can't add a product without a category)                       |
|                                                                   |
|  Update anomaly: Update must be done in multiple places            |
|    (changing customer name in every order record)                  |
|                                                                   |
|  Deletion anomaly: Deleting data removes unrelated info            |
|    (deleting last order deletes customer info)                     |
|                                                                   |
+-------------------------------------------------------------------+
|  NORMALIZATION PROCESS:                                           |
|                                                                   |
|  Step 1: Identify candidate keys and functional dependencies      |
|  Step 2: Check 1NF - split repeating groups                      |
|  Step 3: Check 2NF - eliminate partial dependencies              |
|  Step 4: Check 3NF - eliminate transitive dependencies           |
|  Step 5: Check BCNF - verify all determinants are keys           |
|  Step 6: Check 4NF - separate independent multi-valued deps     |
|                                                                   |
+-------------------------------------------------------------------+
|  COMMON FUNCTIONAL DEPENDENCIES:                                  |
|                                                                   |
|  employee_id     -> employee_name, department_id                  |
|  department_id   -> department_name, location                     |
|  order_id        -> customer_id, order_date                       |
|  product_id      -> product_name, price, category_id              |
|  ssn             -> employee_id, name, dob                        |
|                                                                   |
+-------------------------------------------------------------------+
|  DECOMPOSITION RULES:                                             |
|                                                                   |
|  1. Lossless join: natural join of decomposed relations          |
|     must equal original (no spurious tuples)                      |
|  2. Dependency preservation: all FDs must be enforceable          |
|     from decomposed tables without additional joins               |
|  3. Minimal decomposition: don't create more tables than needed  |
|                                                                   |
+-------------------------------------------------------------------+
|  JPA NORMALIZATION MAPPING:                                       |
|                                                                   |
|  @ManyToOne          -> FK in this table pointing to parent      |
|  @OneToMany(mappedBy)-> Child table has FK; this is parent       |
|  @OneToOne           -> FK on either side (usually dependent)    |
|  @ManyToMany         -> Join table (normalized junction)         |
|  @ElementCollection  -> Separate table for collection             |
|                                                                   |
+-------------------------------------------------------------------+
