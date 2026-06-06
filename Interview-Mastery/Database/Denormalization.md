# Database Denormalization

---

## What is Denormalization?

**Denormalization** is the intentional introduction of redundancy into a database schema to improve read performance. While normalization eliminates redundancy for data integrity, denormalization trades write efficiency and storage for faster reads. This is a performance optimization, not a design failure. Production systems typically operate at a 3NF baseline with selective, documented denormalization for critical query paths.

### Why Denormalize

- **Reduce joins** — Eliminate expensive multi-table joins for frequent queries by combining related data into a single table.
- **Pre-compute aggregates** — Store pre-calculated sums, counts, and averages to avoid expensive aggregations at read time.
- **Optimize for read patterns** — Arrange data in a structure that matches common access patterns, minimizing transformation overhead.
- **Support reporting and analytics** — Flatten complex relational data into wide tables for quick scanning by BI tools.
- **Reduce index overhead** — Fewer tables may mean fewer indexes are needed, reducing write amplification.

### Denormalization Techniques

- **Pre-joined tables** — Merge related tables into one (e.g., order table with customer name directly instead of customer_id).
- **Computed columns** — Pre-calculate and store aggregate values (e.g., review count stored directly on the product table).
- **Derived tables** — Materialize complex query results into physical tables (e.g., daily sales summary table).
- **Array/JSON columns** — Store related data inline as a JSON or array column (e.g., product tags stored directly on the product row).
- **Duplicate columns** — Copy frequently accessed columns across tables (e.g., category name duplicated on the product table).
- **Summary tables** — Pre-aggregated rollups for time-series analytics (e.g., monthly revenue by category).

### Consistency Challenges

Denormalized data must be kept consistent:
- **Application-managed** — Application code updates all copies of the data when changes occur.
- **Trigger-managed** — Database triggers automatically propagate changes to denormalized columns.
- **Materialized views** — The database manages automatic refresh of denormalized structures.
- **Eventual consistency** — Periodic batch sync reconciles copies, acceptable for use cases where slight staleness is tolerated.

### Read vs Write Trade-off

```
Normalized:   Writes are FAST (one table), Reads are SLOW (joins)
Denormalized: Reads are FAST (one table), Writes are SLOW (multiple copies)
```

---

## Core Concepts

### Materialized Views as Managed Denormalization

```sql
-- Auto-maintained denormalized structure
CREATE MATERIALIZED VIEW order_summary AS
SELECT o.id AS order_id,
       o.created_at,
       u.username,
       COUNT(oi.id) AS item_count,
       SUM(oi.total) AS total_amount
FROM orders o
JOIN users u ON u.id = o.user_id
JOIN order_items oi ON oi.order_id = o.id
GROUP BY o.id, u.username;

-- Refresh without blocking readers
REFRESH MATERIALIZED VIEW CONCURRENTLY order_summary;
```

Materialized views offer database-managed denormalization with concurrent refresh support. They are preferred over triggers for reporting and analytics workloads.

### Trigger-Based Consistency

```sql
CREATE OR REPLACE FUNCTION sync_category_name()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE products SET category_name = NEW.name WHERE category_id = NEW.id;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_sync_category_name
    AFTER UPDATE OF name ON categories
    FOR EACH ROW
    EXECUTE FUNCTION sync_category_name();
```

Triggers provide immediate, database-enforced consistency but add latency to every write. Use them for critical paths where consistency is paramount.

### Denormalized E-Commerce Schema

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    user_name VARCHAR(100),        -- DN: copied from users
    user_email VARCHAR(255),       -- DN: copied from users
    billing_city VARCHAR(100),     -- DN: copied from addresses
    shipping_city VARCHAR(100),    -- DN: copied from addresses
    order_item_count INT DEFAULT 0, -- DN: maintained by trigger
    total DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    category_id BIGINT NOT NULL,
    category_name VARCHAR(100),    -- DN: copied from categories
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    review_count INT DEFAULT 0,    -- DN: maintained by trigger
    avg_rating DECIMAL(3,2) DEFAULT 0.00, -- DN: maintained by trigger
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### JPA Entity with Denormalized Fields

```java
@Entity
@Table(name = "products")
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", nullable = false)
    private Category category;

    @Column(name = "category_name")  // Denormalized field
    private String categoryName;

    @Column(nullable = false)
    private String name;

    @Column(name = "review_count")
    private Integer reviewCount;  // Denormalized aggregate

    @Column(name = "avg_rating")
    private BigDecimal avgRating;  // Denormalized aggregate

    @PostLoad
    public void syncCategoryName() {
        if (category != null) {
            this.categoryName = category.getName();
        }
    }
}
```

### Maintaining Denormalized Data (Application Level)

```java
@Service
@Transactional
public class OrderService {
    public Order createOrder(OrderRequest request) {
        User user = userRepository.findById(request.getUserId()).orElseThrow();

        Order order = new Order();
        order.setUserId(user.getId());
        order.setUserName(user.getUsername());  // Denormalized
        order.setUserEmail(user.getEmail());    // Denormalized

        List<OrderItem> items = request.getItems().stream()
            .map(this::toOrderItem).toList();
        order.setItems(items);
        order.setTotal(items.stream()
            .map(OrderItem::getTotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add));

        return orderRepository.save(order);
    }

    // Sync denormalized data on user update
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onUserUpdated(UserUpdatedEvent event) {
        orderRepository.bulkUpdateUserNameAndEmail(
            event.getUserId(), event.getNewUsername(), event.getNewEmail());
    }
}
```

Application-level consistency is flexible but fragile — any code path that modifies the source without updating the denormalized copy causes inconsistency.

### Materialized View with Spring Boot

```java
@Entity
@Immutable  // Read-only for Hibernate
@Table(name = "order_summary")
public class OrderSummary {
    @Id @Column(name = "order_id")
    private Long orderId;
    private LocalDateTime createdAt;
    private String username;
    private Integer itemCount;
    private BigDecimal totalAmount;
    // getters only
}

@Repository
public interface OrderSummaryRepository extends JpaRepository<OrderSummary, Long> {
    List<OrderSummary> findByCreatedAtAfter(LocalDateTime since);
}

@Service
public class OrderSummaryService {
    @Scheduled(cron = "0 0/5 * * * ?")  // Every 5 minutes
    public void refreshMaterializedView() {
        entityManager.createNativeQuery(
            "REFRESH MATERIALIZED VIEW CONCURRENTLY order_summary"
        ).executeUpdate();
    }
}
```

### Reporting Schema (Star Schema)

```sql
-- Fact table (highly denormalized for analytics)
CREATE TABLE sales_facts (
    transaction_id BIGINT,
    product_name VARCHAR(255),        -- DN from products
    product_category VARCHAR(100),    -- DN from categories
    store_name VARCHAR(255),          -- DN from stores
    store_region VARCHAR(100),        -- DN from regions
    customer_tier VARCHAR(20),        -- DN from customers
    quantity INT,
    total DECIMAL(10,2),
    transaction_date DATE
);

-- No joins needed for most BI queries
SELECT store_region, product_category, SUM(total) AS revenue
FROM sales_facts
WHERE transaction_date BETWEEN '2024-01-01' AND '2024-12-31'
GROUP BY store_region, product_category;
```

---

## Common Mistakes

- **Premature denormalization** — Optimizing before measuring the actual bottleneck adds complexity without proven benefit.
- **Denormalizing without documentation** — Future engineers won't know why data is duplicated and may try to normalize it back.
- **Inconsistent copies** — Missing trigger or application logic to keep copies in sync leads to data corruption.
- **Too many denormalized columns** — Creating wide tables with 200+ columns causes row-width performance issues.
- **Copying large objects** — Storing TEXT or JSON blobs in multiple places wastes storage and slows writes.
- **Over-aggregation** — Pre-computing aggregates that could be computed in sub-millisecond queries is wasteful complexity.
- **Ignoring write performance** — Denormalizing without measuring write throughput impact can cripple write-heavy workloads.
- **No reconciliation process** — Failing to detect and fix inconsistencies allows data corruption to compound over time.
- **Using triggers for heavy consistency** — Row-level triggers add latency to every write and don't scale well for bulk operations.
- **Not considering materialized views** — Reinventing what the database already provides leads to fragile custom synchronization code.

---

## Real-World Scenarios

### E-Commerce Product Page with Denormalized Aggregates

A product page displays `review_count` and `avg_rating`. Computing these from the `reviews` table on every page load requires an aggregation JOIN. Denormalizing onto `products` eliminates the JOIN:

```sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    review_count INT DEFAULT 0,
    avg_rating DECIMAL(3,2) DEFAULT 0.00
);

CREATE OR REPLACE FUNCTION update_product_rating()
RETURNS trigger AS $$
BEGIN
    UPDATE products SET
        review_count = (SELECT COUNT(*) FROM reviews WHERE product_id = NEW.product_id),
        avg_rating = (SELECT ROUND(AVG(rating), 2) FROM reviews WHERE product_id = NEW.product_id)
    WHERE id = NEW.product_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### Materialized View for Daily Sales Dashboard

A sales dashboard aggregates daily revenue by region and category. A materialized view pre-computes the result and refreshes hourly:

```sql
CREATE MATERIALIZED VIEW daily_sales_summary AS
SELECT o.created_at::date AS sale_date, p.category_id, c.name AS category_name,
       s.region, SUM(oi.total) AS revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
JOIN categories c ON c.id = p.category_id
JOIN stores s ON s.id = o.store_id
GROUP BY o.created_at::date, p.category_id, c.name, s.region;

REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales_summary;
```

### Social Media Feed with Pre-Joined Data

A social media feed shows 20 posts with author name, avatar, and like count. Joining for each feed load creates 40+ JOINs. Storing denormalized data makes reads instant:

```java
@Entity
@Table(name = "feed_items")
public class FeedItem {
    @Id private Long id;
    private Long authorId;
    private String authorName;      // Denormalized from users
    private String authorAvatarUrl; // Denormalized from users
    private String content;
    private int likeCount;           // Denormalized count
    private LocalDateTime createdAt;
}
```

## Scenario-Based Questions

- **Q: You are building a product listing page that shows product name, category name, and review count. The normalized query joins 4 tables and takes 200ms at 1000 QPS. How do you decide which columns to denormalize?** A: Measure the actual bottleneck first. If the JOIN is fast (<5ms), caching (Redis) may be cheaper than denormalization. Denormalize the most-read, least-changed columns: category name (changes rarely, trigger-synced) and review count (updated frequently but read far more, event-driven update).
- **Q: You denormalized `user_name` and `user_email` onto `orders`. A user changes their email and the reconciliation script finds 5000 orders with the old email. What's the fix?** A: The sync mechanism failed. Fix: add a trigger on `users` updating all related orders on email change. For the inconsistency, run one-time: `UPDATE orders SET user_email = u.email FROM users u WHERE orders.user_id = u.id AND orders.user_email != u.email`. Implement proper sync via trigger (real-time) or event-driven job (eventual consistency).
- **Q: A materialized view aggregating 50M sales records takes 10 minutes to refresh. Queries time out during refresh. How do you fix this?** A: Switch to `REFRESH MATERIALIZED VIEW CONCURRENTLY` which creates a new version and swaps atomically — readers never block. This requires a UNIQUE index on the MV. For faster refresh, incrementally update via summary tables with triggers applying deltas.
- **Q: Your team wants to denormalize customer addresses into every order "for convenience". Orders are read-heavy (1M reads/day) but customers change addresses rarely. What do you recommend?** A: This is reasonable if the order must show the address at time of order (point-in-time snapshot). Store the address at order creation. This is historical accuracy, not just denormalization. Document that orders show shipping address at time of order, not current address.
- **Q: A trigger on `categories` updates `category_name` on 50,000 products. The trigger on rename takes 30 seconds and blocks the UI. How do you decouple this?** A: Remove the synchronous trigger. Publish a `CategoryRenamed` event. A background processor updates products in batches of 1000 with `pg_sleep(0.05)` between batches. Category edits become instant while propagation happens async. Accept eventual consistency (seconds).
- **Q: Your analytics team wants a star schema with a 500GB fact table containing 30 dimension attributes. Queries over the last 7 days are fast, but full-year queries are slow. What's the next optimization?** A: Partition the fact table by month. Each monthly partition is ~40GB. Full-year queries scan only relevant partitions via partition pruning. Consider columnar storage (Parquet, ClickHouse) for analytical workloads, reading only needed columns.
- **Q: A denormalized `order_summary` has inconsistent data: some rows have correct `item_count`, some have stale values. The table is maintained by application code. What went wrong?** A: Application-level consistency is fragile — any code path modifying orders without updating the summary causes inconsistency. Move consistency to the database with triggers, or enforce a single code path. Add a reconciliation job to validate and fix inconsistencies nightly.
- **Q: Your app reads product data 10x more than it writes. The normalized schema requires 5 JOINs for a product page. You're considering denormalization. What metrics guide your decision?** A: Measure: current query latency at P50/P95/P99, database CPU/I/O during peak, write throughput impact, how frequently denormalized columns change, and storage cost. Denormalize only if P99 latency exceeds SLA and other optimizations don't suffice.
- **Q: A trigger updates `review_count` on `products` whenever a review is inserted. What happens with a bulk INSERT of 10,000 reviews?** A: Each review INSERT fires the trigger, updating the product row 10,000 times — massive overhead. Fix: use a statement-level trigger instead of row-level. Better: use a materialized view refreshed periodically, or update via batch job after bulk insert.
- **Q: A CQRS system writes to normalized tables and projects to denormalized read models. The read model is 5 minutes stale. The product owner demands real-time consistency. How do you bridge the gap?** A: For the critical path, use CDC with sub-second latency. Debezium streams WAL changes from PostgreSQL to Kafka, and a stream processor updates the read model with millisecond latency. Layer a cache in front of the real-time projection for the hottest data.

## Interview Questions

- **What is denormalization and why would you use it?** A: Intentional introduction of redundancy to improve read performance. Reduces JOINs, eliminates expensive aggregations, and optimizes for specific query patterns.
- **What are the main denormalization techniques?** A: Pre-joined tables, computed columns (store aggregates), summary tables (pre-aggregated rollups), JSON/array columns, duplicate columns, and materialized views.
- **What are the consistency challenges with denormalization?** A: Denormalized copies must be kept in sync. Sync methods: application-managed, trigger-managed, materialized views, and eventual consistency (batch jobs).
- **What is a materialized view?** A: A materialized view stores the query result as a physical table, refreshed periodically. A regular view runs the query every time. MVs trade staleness for performance.
- **When would you use a trigger for denormalization vs application code?** A: Triggers provide immediate, database-enforced consistency. Application code is more testable but can miss edge cases. Use triggers for critical paths; application code for eventual consistency.
- **What is the read vs write trade-off in denormalization?** A: Normalized: fast writes (one table), slow reads (joins). Denormalized: fast reads (one table), slow writes (multiple copies). The right choice depends on workload ratio.
- **How does denormalization affect storage?** A: Increases storage due to data duplication. Trade-off: storage cost versus query performance. For most applications, storage cost is negligible compared to the performance gain.
- **What is a star schema?** A: A central fact table (denormalized transactional data) surrounded by dimension tables. Heavily denormalized for analytics — most BI queries need few or no JOINs.
- **How do you reconcile inconsistent denormalized data?** A: Schedule a reconciliation job that validates denormalized data against the source of truth, logs discrepancies, and fixes them. Without reconciliation, inconsistencies compound.
- **What's the difference between denormalization and caching?** A: Denormalization stores redundant data in the database schema. Caching stores data in a separate layer (Redis, CDN). Caching can be added or removed without schema changes.

## Developer Recommendations

- **Denormalize only after measuring the actual bottleneck** — Premature denormalization adds complexity. Profile with `EXPLAIN ANALYZE`, measure P99 latency, and identify whether JOINs or data volume is the actual problem.
- **Use materialized views as managed denormalization** — Database-managed, reducing inconsistency risk. Support concurrent refresh without blocking reads. Prefer MVs over triggers for reporting and analytics.
- **Document every denormalized column with rationale** — Add inline comments explaining why each denormalized column exists, what trade-off it makes, and how consistency is maintained.
- **Keep denormalized data consistent with triggers for critical paths** — For data that must always be consistent (invoice totals), use triggers. For non-critical data (trending counts), accept eventual consistency.
- **Limit the number of denormalized columns per table** — A table with 200+ columns causes wide-row performance issues. Denormalize only columns actually needed for query performance.
- **Use CQRS as a formal denormalization pattern** — Write to normalized tables (command model) and project to denormalized tables (query model). Provides clear separation of concerns.
- **Run periodic reconciliation checks** — Schedule a job validating denormalized data against the source of truth. Alert on discrepancies to identify systemic issues before they compound.
