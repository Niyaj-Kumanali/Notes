# Database Normalization

---

## What is Normalization?

**Normalization** is the process of organizing relational database tables to reduce data redundancy and improve data integrity. It involves decomposing tables into smaller, related tables according to defined **normal forms**. Normalization eliminates **update anomalies**, **insertion anomalies**, and **deletion anomalies**. While strict normalization (3NF/BCNF) is the theoretical ideal, production databases typically stop at 3NF and selectively denormalize for performance.

### Goals of Normalization

- **Eliminate data redundancy** — Each fact is stored in exactly one place, reducing storage and eliminating inconsistency risks.
- **Ensure data integrity** — Updates don't create inconsistencies because there is only one place to update each fact.
- **Simplify data maintenance** — Changes to data require modification in only one location, simplifying application logic.
- **Remove anomalies** — Insertion, update, and deletion anomalies are prevented by proper table decomposition.

### Normal Forms Overview

- **1NF** — Atomic columns with no repeating groups. Every column contains a single value, and every row is unique.
- **2NF** — 1NF plus no partial dependency on a composite primary key. Every non-key column must depend on the entire key.
- **3NF** — 2NF plus no transitive dependency on a non-key column. Non-key columns cannot depend on other non-key columns.
- **BCNF** — Every determinant must be a candidate key. Stricter than 3NF and addresses certain edge cases with overlapping candidate keys.
- **4NF** — No multi-valued dependencies. Independent 1:N relationships must be stored in separate tables.
- **5NF** — Every join dependency is implied by candidate keys. Rarely needed in practice.

### Functional Dependency

Attribute Y is **functionally dependent** on X (X → Y) if each X value determines exactly one Y value:
- `employee_id → employee_name` — each employee ID has exactly one name.
- `order_id → customer_id` — each order has exactly one customer.

### Anomalies Normalization Fixes

- **Insertion anomaly** — Cannot insert data without related information (e.g., cannot add a product without a category if all product details are in one table).
- **Update anomaly** — An update must be performed in multiple places (e.g., changing a customer name requires updating every order record referencing that customer).
- **Deletion anomaly** — Deleting data removes unrelated information (e.g., deleting the last order for a customer also deletes the customer's information).

---

## Core Concepts

### First Normal Form (1NF)

**Rule**: Each column contains atomic values. No repeating groups.

**Violation**:
```sql
-- BAD: Multiple phone numbers in one column
CREATE TABLE employee (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    phone_numbers VARCHAR(500) -- '555-0100,555-0101,555-0102'
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
    phone_type VARCHAR(10)
);
```

### Second Normal Form (2NF)

**Rule**: 1NF + every non-key column depends on the **entire** primary key (not just part of a composite key).

**Violation**:
```sql
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    product_name VARCHAR(100), -- depends only on product_id, NOT (order_id, product_id)
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);
```

**Fix**:
```sql
CREATE TABLE products (id INT PRIMARY KEY, name VARCHAR(100));
CREATE TABLE order_items (
    order_id INT,
    product_id INT REFERENCES products(id),
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);
```

### Third Normal Form (3NF)

**Rule**: 2NF + no transitive dependency — non-key columns cannot depend on other non-key columns.

**Violation**:
```sql
CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),  -- transitively depends on customer_id
    customer_city VARCHAR(100),  -- transitively depends on customer_id
    total DECIMAL(10,2)
);
```

**Fix**:
```sql
CREATE TABLE customers (id INT PRIMARY KEY, name VARCHAR(100), city VARCHAR(100));
CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT REFERENCES customers(id),
    total DECIMAL(10,2)
);
```

### Boyce-Codd Normal Form (BCNF)

**Rule**: For every functional dependency X → Y, X must be a candidate key.

**Violation**: Each student has one advisor, each advisor advises multiple students, each advisor belongs to one department. `advisor_id → department_id` but `advisor_id` is not a key:
```sql
CREATE TABLE student_advisor (
    student_id INT PRIMARY KEY,
    advisor_id INT,
    department_id INT
);
```

**Fix**:
```sql
CREATE TABLE student_advisor (student_id INT PRIMARY KEY, advisor_id INT);
CREATE TABLE advisor_department (advisor_id INT PRIMARY KEY, department_id INT);
```

### Fourth Normal Form (4NF)

**Rule**: No multi-valued dependencies. If an employee can have multiple skills AND multiple degrees, these are independent facts that must be stored separately.

**Violation**:
```sql
CREATE TABLE employee_skills_degrees (
    employee_id INT,
    skill VARCHAR(50),
    degree VARCHAR(50),
    PRIMARY KEY (employee_id, skill, degree)
);
-- Inserting a new skill requires repeating all degree values
```

**Fix**:
```sql
CREATE TABLE employee_skill (employee_id INT, skill VARCHAR(50), PRIMARY KEY (employee_id, skill));
CREATE TABLE employee_degree (employee_id INT, degree VARCHAR(50), PRIMARY KEY (employee_id, degree));
```

### Normalized E-Commerce Schema (3NF/BCNF)

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    username VARCHAR(100) NOT NULL,
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
    price DECIMAL(10,2) NOT NULL,
    sku VARCHAR(50) UNIQUE,
    stock_quantity INT NOT NULL DEFAULT 0
);
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING',
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
```

### JPA Entities for Normalized Schema

```java
@Entity
@Table(name = "orders")
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    private String status;
    private BigDecimal total;
    private LocalDateTime createdAt;
}

@Entity
@Table(name = "order_items")
public class OrderItem {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    private Integer quantity;
    private BigDecimal unitPrice;
    private BigDecimal total;
}
```

---

## Common Mistakes

- **Premature over-normalization** — Breaking everything to 5NF or 6NF when 3NF suffices for most applications adds unnecessary complexity and query overhead.
- **Ignoring performance** — A normalized schema requiring 15-join queries for a simple report can be a performance disaster without proper indexing or materialized views.
- **Not recognizing multi-valued dependencies** — Storing independent 1:N relationships in the same table produces cross-product anomalies (4NF violation).
- **Treating normalization as binary** — Normalized versus denormalized is a spectrum, not a binary choice. The right level depends on the workload.
- **Creating too many tables** — Excessive table decomposition leads to complex queries and increased maintenance burden with diminishing returns.
- **Mixing OLTP and OLAP schemas** — OLTP benefits from normalization for fast writes; OLAP benefits from denormalization for fast reads. Use different schemas for different workloads.
- **Forgetting that denormalization is a legitimate optimization** — Denormalization after measurement is engineering; denormalization by default is premature optimization.
- **Using synthetic keys without understanding natural keys** — Natural keys provide business meaning and can improve query performance when used as indexes.
- **Ignoring partial dependencies with composite keys** — Composite primary keys require verifying that every non-key column depends on the entire key, not just part of it.
- **Denormalizing too early** — Optimize only when measurements show a real performance problem, not before.

---

## Real-World Scenarios

### E-Commerce Schema Normalization to 3NF

An orders table stores `customer_name` and `customer_email` alongside `customer_id`. When a customer changes their email, all their order rows must be updated. Normalizing to 3NF eliminates the redundancy:

```sql
CREATE TABLE customers (
    id BIGSERIAL PRIMARY KEY, name VARCHAR(100), email VARCHAR(255) UNIQUE
);
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY, name VARCHAR(255), category_id BIGINT REFERENCES categories(id), price DECIMAL(10,2)
);
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY, customer_id BIGINT REFERENCES customers(id), total DECIMAL(10,2), created_at TIMESTAMP
);
CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY, order_id BIGINT REFERENCES orders(id), product_id BIGINT REFERENCES products(id), quantity INT, unit_price DECIMAL(10,2)
);
```

### Hospital System with 4NF

A hospital database stores doctor specializations and hospital affiliations. Storing both in one table creates multi-valued dependencies. Splitting into separate tables (4NF) fixes this:

```sql
CREATE TABLE doctor_specialties (
    doctor_id INT REFERENCES doctors(id), specialty VARCHAR(50), PRIMARY KEY (doctor_id, specialty)
);
CREATE TABLE doctor_affiliations (
    doctor_id INT REFERENCES doctors(id), hospital_id INT REFERENCES hospitals(id), PRIMARY KEY (doctor_id, hospital_id)
);
```

### Partial Dependency in Order Items

An `order_items` table stores `product_name` alongside `(order_id, product_id)`. Changing a product name requires updating every order item — a 2NF violation:

```sql
-- Before (2NF violation): product_name depends only on product_id
CREATE TABLE order_items (order_id INT, product_id INT, product_name VARCHAR(255), quantity INT, PRIMARY KEY (order_id, product_id));

-- After: product_name lives in products table
CREATE TABLE order_items (order_id INT, product_id INT REFERENCES products(id), quantity INT, PRIMARY KEY (order_id, product_id));
```

## Scenario-Based Questions

- **Q: You are designing the schema for a multi-tenant SaaS platform. Each tenant has customers, products, and orders. How do you design tables to avoid data duplication across tenants while maintaining query performance?** A: Normalize to 3NF with a `tenant_id` column on every table as part of the PK or partitioning key. Use partial indexes per tenant for common queries. For multi-tenant reads, `tenant_id` is highly selective. For cross-tenant admin queries, use a separate analytics database.
- **Q: Your team inherited a database where one table has 200 columns — customer info, order info, and product info all in one table. Queries are slow and the table is 500GB. How do you normalize it without downtime?** A: Plan a phased migration. Phase 1: create normalized tables alongside the denormalized one. Phase 2: set up triggers or dual-writes to keep both in sync. Phase 3: backfill historical data in batches. Phase 4: migrate read queries to the normalized schema. Phase 5: drop the old table. Use Flyway or Liquibase for each step.
- **Q: An employee table has `phone1`, `phone2`, `phone3` columns. Employees now need unlimited phone numbers. What normal form is violated and how do you fix it with minimal application changes?** A: 1NF violation — repeating groups. Create `employee_phones` (employee_id, phone, phone_type, is_primary). For backward compatibility, create a view that aggregates primary phones. Add a trigger to manage the primary phone.
- **Q: A `student_courses` table has `student_id`, `course_id`, `instructor_name`, `instructor_office`. Each course has one instructor, each instructor teaches multiple courses. What normal form is violated?** A: BCNF violation. `instructor_name → instructor_office` but `instructor_name` is not a candidate key. Split into `courses` (course_id, instructor_id) and `instructors` (instructor_id, name, office).
- **Q: Your normalized schema has 40 tables. A reporting query joins 12 tables and takes 15 seconds. The CEO wants a dashboard in 3 seconds. Your schema is in BCNF. How do you reconcile normalization with business requirements?** A: Create a materialized view denormalizing the 12-table join into a flat reporting table. Refresh periodically (every 5-15 minutes). This is a reporting optimization that doesn't compromise the normalized write schema. Document it as intentional denormalization.
- **Q: A project table has `(project_id, employee_id)` as PK with a `role` column. An employee can have multiple roles on the same project. How does this design handle multiple roles?** A: It violates 1NF if `role` is a single column. An employee with two roles needs two rows, making the PK work but creating redundancy. Use `(project_id, employee_id, role)` as PK, or create a separate `project_employee_roles` table. This is a multi-valued dependency that 4NF addresses.
- **Q: You're designing a content management system. Posts have multiple tags, authors, and categories. How do you design this to satisfy 4NF?** A: Create separate junction tables: `post_tags` (post_id, tag_id), `post_authors` (post_id, author_id), `post_categories` (post_id, category_id). Storing all three in one table creates cross-product redundancy — adding a tag requires duplicating all author and category entries.
- **Q: A products table has composite PK `(supplier_id, product_code)` and stores `supplier_name` and `supplier_address`. Is this normalized?** A: No — this is a 2NF violation. `supplier_name` and `supplier_address` depend on `supplier_id`, not the full composite key. Extract a `suppliers` table with `supplier_id` as PK and reference it from products.
- **Q: Your app needs to show customer name on the order history page. In a normalized schema this requires a JOIN. The page loads 50 orders and the JOIN takes 2ms. The team wants to denormalize. What do you advise?** A: Keep it normalized. A 2ms JOIN is negligible. Denormalizing introduces update anomalies — changing a customer name requires updating every historical order. Denormalize only when the join is on a critical 1000+ QPS path and you've measured the bottleneck.
- **Q: A data warehouse fact table has 20 dimension keys. Queries always join the same 5 dimensions. Do you normalize the star schema further?** A: In data warehousing, star schemas are intentionally denormalized. Adding more normalization (snowflake schema) adds joins without significant storage savings for DW workloads. Keep the star schema. Normalization applies more strictly to OLTP than OLAP.

## Interview Questions

- **What is normalization and why is it important?** A: Organizing tables to reduce redundancy and improve data integrity. Eliminates update, insert, and delete anomalies by storing each fact in one place.
- **What is 1NF?** A: Atomic columns (no repeating groups), no multi-valued attributes. Each column contains a single value, each row is unique.
- **What is 2NF?** A: 1NF plus no partial dependencies — every non-key column must depend on the entire primary key, not just part of it.
- **What is 3NF?** A: 2NF plus no transitive dependencies — non-key columns cannot depend on other non-key columns. Every non-key column depends only on the primary key.
- **What is the difference between 3NF and BCNF?** A: BCNF is stricter: every determinant must be a candidate key. A table in 3NF with overlapping candidate keys may violate BCNF.
- **What is a functional dependency?** A: Y is functionally dependent on X (X → Y) if each X value determines exactly one Y value. Example: `employee_id → employee_name`.
- **What is an update anomaly?** A: A data change must be made in multiple places. Example: changing a customer name requires updating every order row referencing that customer.
- **What is 4NF?** A: 3NF or BCNF plus no multi-valued dependencies. Independent 1:N relationships must be stored in separate tables.
- **When would you intentionally stop at 2NF?** A: Rarely, when partial dependencies involve small static lookup data and the join overhead is significant. Document this decision explicitly.
- **How does normalization affect write vs read performance?** A: Normalization improves write performance (one fact updated once) but degrades read performance (more JOINs). Denormalization is the opposite.

## Developer Recommendations

- **Normalize to 3NF by default, denormalize only after measurement** — Starting with 3NF gives clean data integrity. Premature denormalization adds technical debt. Measure actual performance before adding redundancy.
- **Use separate junction tables for independent many-to-many relationships** — Storing skills and languages in one table creates cross-product redundancy (4NF violation). Split into separate tables.
- **Create views for normalized query convenience** — If normalizing adds JOINs, create views that pre-join commonly accessed tables. Provides convenience without storage redundancy.
- **Document all intentional denormalizations** — Add comments explaining why the redundancy exists, what trade-off was made, and how consistency is maintained.
- **Use synthetic PKs but index natural keys** — Synthetic BIGSERIAL or UUID keys are stable and efficient for joins. Add UNIQUE indexes on natural keys (email, SSN) to enforce business uniqueness.
- **Consider partial dependencies in composite key design** — Verify every non-key column depends on the FULL composite key. Move partial dependency columns to separate tables.
- **Don't over-normalize OLTP** — Going beyond 3NF or BCNF to 5NF or 6NF adds complexity with minimal benefit for most OLTP workloads.
