# CQRS (Command Query Responsibility Segregation)

---

## Overview

- **Definition:** An architectural pattern that separates read and write operations into distinct models — Commands handle mutations, Queries handle data retrieval.
- **Why It Exists:** Traditional CRUD uses a single model for both reads and writes, forcing compromises. CQRS allows each model to be optimized independently for its specific workload with different data stores, schemas, scaling strategies, and consistency models.
- **Key Concepts:** **Command** (changes state, returns no data), **Query** (returns data, no side effects), **Command Handler** (validates and executes commands), **Query Handler** (fetches from read model), **Read Model** (denormalized data optimized for queries), **Write Model** (domain model with business logic), **Event Bus** (communicates changes from write to read side), **Projection** (transforms events into read model state)
- **CQRS vs CQS** — CQS (Command-Query Separation) is a class-level principle: methods are either commands (void) or queries (return value, no side effects). CQRS elevates this to an architectural pattern with separate models, separate services, and often separate databases. CQRS applies CQS at the system level.
- **Materialized Views as Read Models** — A materialized view is a pre-computed, denormalized representation of data optimized for specific query patterns. Examples: order summary with customer name and product details (pre-joined and ready to serve), daily sales aggregation (pre-calculated totals), user activity feed (pre-assembled from multiple sources). Materialized views eliminate expensive joins at query time.

---

## Core Concepts

- **Command Processing Pipeline:** Client sends Command → Command Handler validates and executes → Aggregate/Entity persists changes → Event published → Read Model updated asynchronously.
- **Query Processing Pipeline:** Client sends Query → Query Handler fetches from Read Model → Denormalized data returned directly.
- **Separate Models:** Write model contains domain logic, invariants, and validation. Read model is denormalized, pre-joined, and optimized for specific query patterns. They can use the same database (different tables), different databases, or different database technologies.
- **Eventual Consistency:** Read models are updated asynchronously after writes. Typical lag is 10ms–100ms. For critical reads, the write model can be queried directly.
- **Transactional Outbox Pattern** — Publishing events reliably from the write side is a key challenge. The outbox pattern writes both the domain data and the event to the same database transaction. A separate process (outbox publisher) reads from the outbox table and publishes events to the message broker. This ensures the event is never lost — if publish fails, the outbox retains the event for retry.
- **Multiple Read Models per Write Model** — A single write model can serve multiple read models optimized for different query patterns: a search read model (Elasticsearch), a dashboard read model (pre-aggregated in PostgreSQL), a real-time analytics read model (Redis sorted sets). Each projection independently consumes events from the write side, allowing different read models to scale independently.

```java
// Command (write side)
public class CreateOrderCommand {
    private String customerId;
    private List<OrderItemDto> items;
}

// Command Handler
@Component
public class CreateOrderCommandHandler implements CommandHandler<CreateOrderCommand, String> {
    @Transactional
    public String handle(CreateOrderCommand command) {
        Order order = new Order();
        order.setId(UUID.randomUUID().toString());
        order.setCustomerId(command.getCustomerId());
        // ... add items, validate, submit
        orderRepository.save(order);
        eventBus.publish(new OrderCreatedEvent(order.getId(), order.getCustomerId(), order.getTotalAmount(), Instant.now()));
        return order.getId();
    }
}

// Query (read side)
public class GetOrderSummaryQuery {
    private String orderId;
}

// Query Handler
@Component
public class GetOrderSummaryQueryHandler implements QueryHandler<GetOrderSummaryQuery, OrderSummary> {
    public OrderSummary handle(GetOrderSummaryQuery query) {
        return orderSummaryRepository.findById(query.getOrderId())
            .orElseThrow(() -> new ResourceNotFoundException("Order not found"));
    }
}
```

---

## Common Mistakes

- **CQRS for Simple CRUD** — adding CQRS overhead to a simple application that doesn't need separate read/write models.
  - **Why it looks correct:** CQRS is a well-known pattern for scalability, and applying it proactively seems like future-proofing — the overhead of event handling, projections, and eventual consistency management outweighs the benefits when reads and writes are already well-balanced.
- **Coupling Read and Write Models** — using the same entity class for both read and write, defeating the purpose of separation.
  - **Why it looks correct:** DRY suggests reusing the same entity class avoids duplication, and the fields are the same — the coupling becomes a problem when a write-side optimization (like adding an index-friendly column) forces changes to the read model, or vice versa.
- **Ignoring Eventual Consistency** — not handling the lag between write and read model update, causing users to see stale data after a write.
  - **Why it looks correct:** in a monolith, after a write the data is immediately available for reads, so this synchronous guarantee seems like the only correct behavior — the eventual consistency window only becomes visible when a user refreshes their order list after placing an order and doesn't see it.
- **Command Returning Data** — commands should be void (return ID only); if commands return data, they become queries and blur the separation.
  - **Why it looks correct:** returning the created object from a command is convenient for the caller and avoids a follow-up query — the blurring becomes a problem when the command handler starts computing data for display, mixing write logic with read concerns.
- **Duplicating Business Logic in Query Handlers** — query handlers should not duplicate validation or calculation logic from the write side.
- **Not Handling Projection Failures** — If a projection fails to process an event, the read model becomes stale. Implement retry logic with exponential backoff, dead letter queues for persistent failures, and monitoring to alert on lagging projections.
- **Over-normalizing Read Models** — Read models should be denormalized and query-optimized. A read model for an order dashboard should include customer name, email, and order total in a single table — even though the write model stores them in separate tables. The goal is zero joins at query time.

---

## Key Design Considerations

- **CQRS + Event Sourcing Synergy** — write side appends events to an event store; read side projects events to materialized views. Read models can be rebuilt by replaying all events from scratch.
- **Read Model Rebuilding** — drop and recreate read model tables, replay all events from the event store. Supports zero-downtime by creating a new version in parallel and switching when caught up.
- **When to Use Separate Databases** — same database, different tables (simple CQRS); same DB type, different instances (read replicas for scale); different database types (PostgreSQL for writes, Elasticsearch for reads).
- **Transactional Boundaries** — one transaction per aggregate. Use the Outbox pattern to ensure events are published atomically with state changes. Saga pattern for multi-aggregate workflows.
- **Command Validation** — validate in command handlers before executing domain logic. Authorization checks belong in the application layer, not the domain model.
- **Event Versioning in Projections** — Events evolve over time as new fields are added. Projections must handle multiple event versions. Use schema version in event metadata, write event handlers that can process multiple versions (ignore unknown fields), and use the upsert pattern (INSERT ON CONFLICT UPDATE) for idempotent projection updates.
- **CQRS Without Event Sourcing** — CQRS does not require Event Sourcing. The write side can use a traditional database (PostgreSQL, MySQL) with changes propagated via event publication (using the outbox pattern). Event Sourcing is a separate choice about how to store state (as an append-only event log versus current state snapshot).

---

## Real-World Scenarios

### Scenario 1: E-Commerce Order Processing
**Context:** An e-commerce platform has a monolithic `OrderService` handling both order placement (validation, inventory check, payment) and order queries (history, status, analytics). As traffic grows, write-heavy operations contend with complex read queries. A single dashboard query scanning millions of orders blocks a simple order placement.

**Resolution:** Split into CQRS. The write side uses a normalized `orders` table with transactional integrity. The read side maintains denormalized `order_summary` and `customer_dashboard` tables updated asynchronously via domain events. Read queries hit indexed, pre-joined tables returning in <10ms instead of scanning the full order history.

```java
// Write side — command handler
@Component
public class PlaceOrderHandler {
    @Transactional
    public OrderId handle(PlaceOrderCommand cmd) {
        Order order = new Order(cmd.customerId(), cmd.items());
        orderRepository.save(order);
        eventBus.publish(new OrderPlacedEvent(
            order.getId(), cmd.customerId(), order.getTotal(), Instant.now()));
        return order.getId();
    }
}

// Read side — projection
@Component
public class OrderSummaryProjector {
    @EventListener
    public void on(OrderPlacedEvent event) {
        OrderSummary summary = new OrderSummary(
            event.orderId(), event.customerId(), event.total(),
            "PENDING", event.timestamp());
        orderSummaryRepository.save(summary);
    }
}
```

### Scenario 2: Real-Time Analytics Dashboard
**Context:** A SaaS analytics platform ingests 10K events/sec. Users query dashboards (aggregate counts, top-K, trends over time). The same normalized tables used for ingestion also serve dashboard queries, causing contention and slow responses.

**Resolution:** The write side appends events to a time-series database (InfluxDB). A projection continuously runs aggregation pipelines and materializes pre-computed results into a dedicated read database (Elasticsearch). Dashboards query Elasticsearch with sub-second latency. Write throughput is unaffected by complex analytical queries.

### Scenario 3: Multi-Team Development on a Banking System
**Context:** A banking application needs both transaction processing (writes with strict validation) and customer-facing account history (reads with flexible filtering, pagination, export). Two teams need to work independently.

**Resolution:** Team A owns the command model: accounts, transfers, validations — the write database. Team B owns the query model: account statements, spending analysis, PDF exports — the read database. They agree on event schemas as their contract. Team A publishes `TransactionProcessed` events; Team B consumes them to build materialized views. Each team deploys independently, with their own schema and scaling strategy.

## Use Cases

- CQRS shines when read and write workloads have fundamentally different characteristics — different shapes, different frequencies, or different performance requirements. These patterns cover when to split them.

- **High-traffic e-commerce platforms** — separating order placement (write) from order history/search (read)
  - When to use: Order placement must be fast and transactional; order history queries are frequent and need aggregated, denormalized data. The write model uses a normalized schema with constraints; the read model uses pre-joined `order_summary` tables optimized for display. Example: an e-commerce site where the checkout flow writes to a normalized order database while the "My Orders" page queries a read-only replica with pre-computed totals, item counts, and status badges.
  - **Avoid when:** Read and write workloads are balanced and simple — a single model with read replicas is easier to maintain.

- **Reporting and analytics dashboards** — providing fast, aggregated views over operational data
  - When to use: The operational database is normalized for transactions but reporting queries require joins and aggregations that are too slow. CQRS maintains materialized view read models that are pre-computed for specific dashboard queries. Example: a SaaS platform where the write model stores individual billing events and the read model maintains real-time MRR dashboards, aggregated by plan, region, and cohort.
  - **Avoid when:** Reports can be generated from the operational database with acceptable latency — a nightly batch job or database view may suffice.

- **Systems with asymmetric read/write ratios** — services that are read 100x more than written
  - When to use: A resource is created or updated infrequently but queried constantly. The write model can be simple (just persistence and validation) while the read model is heavily optimized, cached, and independently scalable. Example: a content management system where articles are written by editors a few times per day but read by millions of users — CQRS lets the read model use a CDN-cached, denormalized format while the write model enforces editorial workflows.
  - **Avoid when:** Read and write rates are similar — the overhead of maintaining two models isn't justified.

- **Event sourcing integration** — deriving read models from a stream of domain events
  - When to use: Your system already uses event sourcing (every state change is stored as an event). CQRS naturally follows: the event store is the write model, and projections consume events to build read models optimized for different queries. Example: a banking application where every account transaction is stored as an event, and separate projections build a balance read model (current balance), a statement read model (transaction history), and a fraud detection read model (unusual patterns).
  - **Avoid when:** The domain does not require the auditability and traceability of event sourcing — CQRS without event sourcing adds complexity without the corresponding benefits.

---

## Scenario-Based Questions

1. **Q: You are building an order management system where customers place orders and later query order history. The same database handles both operations, and during sales events the system slows down. How do you fix this?**
   - A: Split into CQRS. The write model uses a normalized schema optimized for transactional integrity (foreign keys, constraints, triggers). The read model uses denormalized `order_summary` tables with pre-joined customer and item data. Events flow from write to read asynchronously via a message broker. Writes remain fast during traffic spikes because reads no longer compete for the same database resources. Acceptable trade-off: read model may lag by 50-100ms, which is acceptable for order history queries.

2. **Q: Your team is implementing CQRS but the product owner insists that after placing an order, the user must immediately see it in their order list. How do you handle eventual consistency?**
   - A: For the user who just placed the order, write directly to both the write and read models synchronously within the same transaction (or use transactional outbox with immediate projection for that specific user). For other users, the read model updates asynchronously via events. This hybrid approach gives the placing user immediate feedback while maintaining CQRS benefits for bulk queries. Track the read lag via a metric and alert if it exceeds 2 seconds.

   - **Follow-up:** You write to both models synchronously for the placing user, but the synchronous write to the read model introduces write latency to the command handler — how do you prevent this from degrading the order placement p99 under high load?

3. **Q: You have separate read and write databases. A bug in the projection causes the read database to miss 30 minutes of events. How do you recover without data loss?**
   - A: Rebuild the read model from scratch by replaying all events from the event store. Since the event store is append-only and immutable, you can replay events in order and reconstruct the complete read model. Run the new projection in parallel while the old one still serves traffic, then switch when caught up. This is a key benefit of CQRS + Event Sourcing — read models are disposable and rebuildable.

4. **Q: Your read models are getting complex with 15+ different projections for different query patterns. How do you manage this complexity?**
   - A: Apply the same bounded context thinking from DDD to your projections. Group related projections into modules. Use a single materialized view per query pattern with clearly scoped data. For reporting queries, maintain a separate analytics read model. For real-time dashboards, use a streaming read model (Kafka Streams). Retire projections that are no longer queried.

   - **Follow-up:** A new feature requires data from two separate projections joined together — do you create a third projection that joins data from the first two, or add logic to the query layer that performs the join at read time, and how does each approach impact rebuildability?

5. **Q: A command needs to return data — for example, creating a user and returning the user ID. Does this violate CQRS?**
   - A: It's acceptable to return the generated ID from a command. The principle is that commands shouldn't return business data needed for display. Returning a technical identifier (ID, URI) doesn't violate the pattern. The command handler can return the ID after persisting; the client then queries the read model for any display data.

6. **Q: Your system uses CQRS but you're seeing stale data on dashboard widgets for several seconds after updates. How do you reduce the lag?**
   - A: Optimize the projection pipeline: batch event processing instead of one-at-a-time, use in-memory processing before writing to the read database, increase projection worker threads. Add a "fast lane" for the most critical events (e.g., order status changes) with dedicated projection resources. Monitor projection lag as a critical metric and alert when it exceeds 500ms.

7. **Q: A junior developer puts business validation logic in both the command handler and the query handler. What's wrong with this approach?**
   - A: Business logic duplication creates maintenance nightmares — the two implementations will inevitably diverge. Query handlers should only format and fetch data; they should never duplicate validation, calculation, or business rules from the command side. If a calculation is needed in both places (e.g., tax calculation), extract it into a shared domain service that both sides depend on.

   - **Follow-up:** The shared domain service is now called from both the command handler and the query handler — but the command handler runs in a transactional context with write locks, while the query handler doesn't. How do you prevent the same service method from accidentally being used in a write context when it should only be read-only?

8. **Q: You're migrating from a CRUD monolith to CQRS. How do you do this incrementally without a big-bang rewrite?**
   - A: Use the strangler fig pattern. Start by identifying the most read-heavy endpoint (e.g., dashboard). Create a new read model for it while keeping writes unchanged. Gradually move more queries to the read model. Only then split the write side. Each step is independently deployable and revertable. The monolith's original tables serve as the initial source of truth for both sides.

9. **Q: Your write model needs to support bulk operations (import 10K orders at once). How does this interact with CQRS?**
   - A: Keep the bulk command as a single atomic write operation on the command side — it should succeed or fail as one unit. After persistence, publish individual events for each order. The read model processes these events in batch (using chunked projections) to avoid overwhelming the read database. Consider a dedicated bulk projection that updates the read model in batches of 500.

10. **Q: You're using separate databases for reads and writes, but a deployment that changes the write schema also requires read schema changes. How do you handle this coupling?**
    - A: Decouple the deployment by versioning your events. The write side publishes v1 events for the new schema. The read side has v1 and v2 projections running simultaneously. The v2 projection on the read side consumes v1 events and handles the schema transformation. Deploy write changes first, then read changes after verifying the new events are flowing correctly.

---

## Interview Questions

1. **What does CQRS stand for and what problem does it solve?**
   - A: Command Query Responsibility Segregation. It solves the problem of a single model being suboptimal for both reads and writes by separating them into distinct models, each optimized for its workload.

2. **What is the difference between a command and a query?**
   - A: A command changes state and returns no data (void or ID only). A query returns data and has no side effects. This follows CQS at the architectural level.

3. **Does CQRS require separate databases?**
   - A: No. It requires separate models. They can use the same database (different tables), the same database type (read replicas), or different database technologies (PostgreSQL for writes, Elasticsearch for reads).

4. **Is CQRS always used with Event Sourcing?**
   - A: No. They are independent patterns. CQRS can use traditional CRUD for writes with events only for read model updates. Event Sourcing is a separate concern about how state is stored.

5. **How do you update the read model when the write side changes?**
   - A: The write side publishes events after persisting changes. Projection handlers subscribe to these events and update the read model asynchronously. The outbox pattern ensures reliable event publication.

6. **What is the difference between CQRS and CQS?**
   - A: CQS is a class-level design principle where methods are either commands (void) or queries (return value). CQRS is an architectural pattern with separate models, separate services, and often separate databases.

7. **How do you handle eventual consistency in CQRS?**
   - A: Accept that read models lag behind writes (typically 10-100ms). For critical "read-your-write" scenarios, the command handler can write to both models synchronously. Communicate the async nature to users via UI patterns (loading states, optimistic updates).

8. **How does CQRS help with team scalability?**
   - A: Different teams can work on the command model (domain logic, validation) and query model (performance optimization, denormalization) independently. They agree on event schemas as their contract, enabling parallel development.

9. **What is a projection in CQRS?**
   - A: A projection transforms domain events into a read model state. It subscribes to events, processes them, and updates the query database. Multiple projections can consume the same events for different read models.

10. **How do you handle partial failures in read model updates?**
    - A: Use a dead letter queue for failed events with exponential backoff retry. Ensure idempotent event handlers (same event processed twice produces same result). For catastrophic corruption, rebuild the read model from the event store.

---

## Developer Recommendations

- **Start with separate models in the same database before splitting databases** — CQRS is about model separation, not database separation. Begin with the same database using different tables for read models. This avoids distributed transaction complexity while gaining most benefits. Only add a separate read database when you need different indexing strategies or independent scaling.

- **Use the outbox pattern to reliably publish events from the write side** — Dual-writes (DB update + event publish) are not atomic. If the event publish fails, the write succeeds but the read model never updates. The outbox pattern writes both the domain data and the event to the same DB transaction. A separate relay publishes events, ensuring exactly-once delivery semantics.
  - **Production story:** An analytics platform skipped the outbox pattern and published events after the transaction — when a database failover during a traffic spike caused 15% of events to be lost before publish, it took 3 days to detect and repair the inconsistent data across 12 read models.

- **Keep projections idempotent and rebuildable** — A projection should produce the same read model state when replaying the same events, regardless of how many times it runs. Use upsert operations (not inserts) in projections. Store the last processed event position so projections can resume from failures. Test projection rebuilding in CI to verify correctness.
  - **Production story:** A logistics startup's projection skipped a schema migration for a new event field — by the time they noticed, the projection was 2 million events behind, and rebuilding from scratch took 14 hours because they had never tested the rebuild process and discovered the projection wasn't idempotent.

- **Avoid CQRS for simple CRUD applications** — CQRS adds significant complexity: event handling, projection management, eventual consistency. Only use it when you have genuinely different read and write workloads — high write throughput with complex read queries, or read models that serve different purposes than the write model.

- **Monitor projection lag as a critical SLO** — Projection lag (time between event publication and read model update) is the key health metric for CQRS. Alert if lag exceeds 2 seconds for user-facing queries or 30 seconds for analytical queries. Use dedicated metrics per projection to identify which ones are falling behind.
- **Use read replicas before splitting databases** — Before adopting full CQRS with separate databases, try PostgreSQL read replicas. The write model writes to the primary, and the read model reads from replicas. This provides read scaling and reduced write-contention without the complexity of event-driven projections. Only add projections and separate read databases when read replicas are insufficient.
- **Design projections to be rebuildable from scratch** — A projection should be disposable. If a read model becomes corrupted or needs schema changes, you should be able to drop it and rebuild by replaying all events from the event store. This requires idempotent projection handlers and an append-only event store with sufficient retention. Test projection rebuilding in CI to verify it produces correct results.
