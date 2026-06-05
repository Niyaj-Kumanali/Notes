# CQRS (Command Query Responsibility Segregation)

## 1. Executive Summary

CQRS is an architectural pattern that separates read and write operations into distinct models. Commands handle mutations (create, update, delete) while queries handle data retrieval. This separation allows each model to be optimized independently for its specific workload, enabling different data stores, schemas, scaling strategies, and consistency models for reads vs writes.

## 2. Core Theory

### Fundamental Principle
Traditional CRUD uses a single model for both reads and writes. CQRS splits them:
- **Command**: Changes state. Returns no data (or just ID/status). Should be a void operation.
- **Query**: Returns data. Should not change state. Should be idempotent.

### When to Use CQRS
- Different read/write workloads (read-heavy, write-heavy, or both).
- Complex domain logic on writes but simple reads.
- Need to optimize read performance independently.
- Teams need to work on read and write models separately.
- When eventual consistency is acceptable.

### When NOT to Use CQRS
- Simple CRUD applications with no complex domain.
- When strong consistency between read and write is always required.
- Small teams/simple domains where overhead outweighs benefits.

## 3. Under-the-Hood Deep Dive

### Command Processing Pipeline
```
[Client] -> [Command] -> [Command Handler] -> [Aggregate/Entity] -> [Event Store/DB]
                              |
                       [Event Bus] -> [Event Handlers] -> [Read Model Update]
```

### Query Processing Pipeline
```
[Client] -> [Query] -> [Query Handler] -> [Read Model (Materialized View)] -> [Response]
```

### Key Components
- **Command**: DTO with data needed to perform an action. Named imperatively: `CreateOrderCommand`.
- **Command Handler**: Validates command, invokes domain logic, persists changes.
- **Query**: DTO with query parameters. Named declaratively: `GetOrderByIdQuery`.
- **Query Handler**: Fetches data from read model, returns result.
- **Read Model**: Denormalized data optimized for queries (may differ from write model schema).
- **Write Model**: Domain model with business logic, constraints, and invariants.

### Separate Models Example

```java
// Command model (write side)
@Entity
@Table(name = "orders")
public class Order {
    @Id
    private String id;
    private String customerId;
    private BigDecimal totalAmount;
    @Enumerated(EnumType.STRING)
    private OrderStatus status;
    @OneToMany(cascade = ALL)
    private List<OrderLineItem> items;

    // Business logic methods
    public void addItem(Product product, int quantity) {
        // Validate inventory, calculate price, check limits
        this.items.add(new OrderLineItem(product, quantity));
        this.totalAmount = calculateTotal();
    }

    public void submit() {
        if (items.isEmpty()) throw new IllegalStateException("Cannot submit empty order");
        this.status = OrderStatus.SUBMITTED;
    }
}

// Read model (query side) - denormalized for fast retrieval
@Table(name = "order_summaries")
public class OrderSummary {
    @Id
    private String orderId;
    private String customerId;
    private String customerName;    // Denormalized from user service
    private BigDecimal totalAmount;
    private String status;
    private int itemCount;
    private String firstItemName;   // Denormalized
    private Instant createdAt;
}
```

## 4. Production Code Examples

### Command and Command Handler

```java
// Command
@Data
@Builder
public class CreateOrderCommand {
    private String customerId;
    private List<OrderItemDto> items;
}

public class OrderItemDto {
    private String productId;
    private String productName;
    private int quantity;
    private BigDecimal unitPrice;
}

// Command Handler
@Component
public class CreateOrderCommandHandler implements CommandHandler<CreateOrderCommand, String> {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private EventBus eventBus;

    @Override
    @Transactional
    public String handle(CreateOrderCommand command) {
        // Validate
        if (command.getItems() == null || command.getItems().isEmpty()) {
            throw new ValidationException("Order must have at least one item");
        }

        // Create domain aggregate
        Order order = new Order();
        order.setId(UUID.randomUUID().toString());
        order.setCustomerId(command.getCustomerId());

        for (OrderItemDto itemDto : command.getItems()) {
            Product product = productRepository.findById(itemDto.getProductId())
                .orElseThrow(() -> new ProductNotFoundException(itemDto.getProductId()));
            order.addItem(product, itemDto.getQuantity());
        }

        order.submit();

        // Persist
        orderRepository.save(order);

        // Publish event
        eventBus.publish(new OrderCreatedEvent(order.getId(), order.getCustomerId(),
            order.getTotalAmount(), order.getStatus(), Instant.now()));

        return order.getId();
    }
}
```

### Query and Query Handler

```java
// Query
@Data
public class GetOrderSummaryQuery {
    private String orderId;
}

// Query Handler
@Component
public class GetOrderSummaryQueryHandler implements QueryHandler<GetOrderSummaryQuery, OrderSummary> {

    @Autowired
    private OrderSummaryRepository orderSummaryRepository;

    @Override
    public OrderSummary handle(GetOrderSummaryQuery query) {
        return orderSummaryRepository.findById(query.getOrderId())
            .orElseThrow(() -> new ResourceNotFoundException("Order not found"));
    }
}

// Another query - list with filtering
@Data
public class ListCustomerOrdersQuery {
    private String customerId;
    private int page;
    private int size;
    private String statusFilter;
}

@Component
public class ListCustomerOrdersQueryHandler
        implements QueryHandler<ListCustomerOrdersQuery, Page<OrderSummary>> {

    @Autowired
    private OrderSummaryRepository orderSummaryRepository;

    @Override
    public Page<OrderSummary> handle(ListCustomerOrdersQuery query) {
        Pageable pageable = PageRequest.of(query.getPage(), query.getSize(),
            Sort.by("createdAt").descending());

        if (query.getStatusFilter() != null) {
            return orderSummaryRepository
                .findByCustomerIdAndStatus(query.getCustomerId(),
                    OrderStatus.valueOf(query.getStatusFilter()), pageable);
        }

        return orderSummaryRepository.findByCustomerId(query.getCustomerId(), pageable);
    }
}
```

### Read Model Updater (Event Handler)

```java
@Component
public class OrderReadModelUpdater {

    @Autowired
    private OrderSummaryRepository summaryRepository;

    @Autowired
    private UserServiceClient userServiceClient;

    @EventListener
    @Transactional
    public void onOrderCreated(OrderCreatedEvent event) {
        // Fetch customer name from user service (denormalize)
        String customerName = userServiceClient.getUserName(event.getCustomerId());

        OrderSummary summary = new OrderSummary();
        summary.setOrderId(event.getOrderId());
        summary.setCustomerId(event.getCustomerId());
        summary.setCustomerName(customerName);
        summary.setTotalAmount(event.getTotalAmount());
        summary.setStatus(event.getStatus().name());
        summary.setItemCount(event.getItemCount());
        summary.setCreatedAt(event.getTimestamp());

        summaryRepository.save(summary);
    }

    @EventListener
    @Transactional
    public void onOrderStatusChanged(OrderStatusChangedEvent event) {
        summaryRepository.findById(event.getOrderId()).ifPresent(summary -> {
            summary.setStatus(event.getNewStatus().name());
            summaryRepository.save(summary);
        });
    }
}
```

### Separate Database Configuration

```java
@Configuration
public class DatabaseConfig {

    // Write database
    @Bean
    @Primary
    @ConfigurationProperties(prefix = "spring.datasource.write")
    public DataSource writeDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @Primary
    public LocalContainerEntityManagerFactoryBean writeEntityManagerFactory(
            EntityManagerFactoryBuilder builder) {
        return builder
            .dataSource(writeDataSource())
            .packages("com.example.domain.write")
            .persistenceUnit("write")
            .build();
    }

    // Read database
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.read")
    public DataSource readDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean readEntityManagerFactory(
            EntityManagerFactoryBuilder builder) {
        return builder
            .dataSource(readDataSource())
            .packages("com.example.domain.read")
            .persistenceUnit("read")
            .build();
    }

    @Bean
    public PlatformTransactionManager readTransactionManager(
            @Qualifier("readEntityManagerFactory") EntityManagerFactory emf) {
        return new JpaTransactionManager(emf);
    }
}
```

### Command Validation with MediatR-style Bus

```java
// Command Bus (inspired by MediatR pattern)
@Component
public class CommandBus {

    @Autowired
    private ApplicationContext applicationContext;

    @SuppressWarnings("unchecked")
    public <R> R dispatch(Command<R> command) {
        Class<?> handlerClass = resolveHandlerClass(command.getClass());
        CommandHandler<Command<R>, R> handler =
            (CommandHandler<Command<R>, R>) applicationContext.getBean(handlerClass);
        return handler.handle(command);
    }

    private Class<?> resolveHandlerClass(Class<?> commandClass) {
        // Convention: CreateOrderCommand -> CreateOrderCommandHandler
        String handlerName = commandClass.getSimpleName() + "Handler";
        String packageName = commandClass.getPackageName();

        try {
            return Class.forName(packageName + ".handlers." + handlerName);
        } catch (ClassNotFoundException e) {
            throw new HandlerNotFoundException(commandClass);
        }
    }
}

// Usage
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Autowired
    private CommandBus commandBus;

    @Autowired
    private QueryBus queryBus;

    @PostMapping
    public ResponseEntity<String> createOrder(@RequestBody CreateOrderCommand command) {
        String orderId = commandBus.dispatch(command);
        return ResponseEntity.created(URI.create("/api/orders/" + orderId)).body(orderId);
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderSummary> getOrder(@PathVariable String id) {
        GetOrderSummaryQuery query = new GetOrderSummaryQuery(id);
        return ResponseEntity.ok(queryBus.dispatch(query));
    }
}
```

## 5. Real-World Scenarios

### E-Commerce Product Catalog

**Write Model:** Product aggregate with validation rules (SKU uniqueness, pricing rules, inventory thresholds). Event-sourced (ProductCreated, PriceChanged, InventoryUpdated).

**Read Model:** Denormalized product view for search/catalog. Fields from multiple aggregates. Pre-joined for fast queries. Search index (Elasticsearch).

### Banking System

**Write Model:** Account aggregate. Transaction processing with balance validation, overdraft rules, fraud checks.

**Read Model:** Account summary (balance, transactions). Optimized for customer dashboard queries. Updated asynchronously via events.

### Content Management System

**Write Model:** Document aggregate with versioning, approval workflow, permissions.

**Read Model:** Published content (multiple versions). Rendered HTML cached. Search index.

## 6. Performance

### Optimization Strategies

**Read Side:**
- Denormalized tables avoiding joins.
- Redis cache for hot queries.
- Read replicas for horizontal scaling.
- Elasticsearch for full-text search.
- Pagination, filtering, projection in queries.

**Write Side:**
- Event sourcing for audit and replay.
- Batch command processing.
- Optimistic concurrency control.

### Read Model Refresh Lag
```
Event Publication -> Queue -> Read Model Update
Typical lag: 10ms - 100ms (near real-time)
For critical reads: synchronously update read model in same transaction
```

### CQRS Caching Strategy
```java
@Component
public class CachedOrderSummaryRepository {

    @Autowired
    private RedisTemplate<String, OrderSummary> redisTemplate;

    @Autowired
    private OrderSummaryRepository jpaRepository;

    private static final Duration CACHE_TTL = Duration.ofMinutes(5);

    public Optional<OrderSummary> findById(String orderId) {
        String cacheKey = "order_summary:" + orderId;

        OrderSummary cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return Optional.of(cached);
        }

        Optional<OrderSummary> result = jpaRepository.findById(orderId);
        result.ifPresent(summary ->
            redisTemplate.opsForValue().set(cacheKey, summary, CACHE_TTL));
        return result;
    }

    @EventListener
    public void onOrderStatusChanged(OrderStatusChangedEvent event) {
        redisTemplate.delete("order_summary:" + event.getOrderId());
    }
}
```

## 7. Security

### Command Authorization
```java
@Component
public class CancelOrderCommandHandler implements CommandHandler<CancelOrderCommand, Void> {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private AuthorizationService authService;

    @Override
    @Transactional
    public Void handle(CancelOrderCommand command) {
        Order order = orderRepository.findById(command.getOrderId())
            .orElseThrow(() -> new ResourceNotFoundException("Order not found"));

        // Authorization check
        if (!authService.canCancelOrder(command.getUserId(), order)) {
            throw new AccessDeniedException("User cannot cancel this order");
        }

        order.cancel();
        orderRepository.save(order);
        return null;
    }
}
```

### Data Access Control on Read Side
```java
@Component
public class GetOrderSummaryQueryHandler implements QueryHandler<GetOrderSummaryQuery, OrderSummary> {

    @Override
    public OrderSummary handle(GetOrderSummaryQuery query) {
        OrderSummary summary = summaryRepository.findById(query.getOrderId())
            .orElseThrow(() -> new ResourceNotFoundException("Order not found"));

        // Ensure user can only see their own orders (unless admin)
        if (!authService.hasAccess(query.getCurrentUserId(), summary.getCustomerId())) {
            throw new AccessDeniedException("Access denied");
        }

        return summary;
    }
}
```

## 8. Common Mistakes

### Mistake 1: CQRS for Simple CRUD
Adding CQRS overhead to a simple CRUD application that doesn't need separate read/write models.

### Mistake 2: Coupling Read and Write Models
```java
// WRONG - using the same entity for both
@Entity
public class Order {
    // Mix of read-optimized and write-validated fields
    @Column(name = "customer_name") // Denormalized for read, but lives in write model
    private String customerName;
}
```

### Mistake 3: Ignoring Eventual Consistency
Not handling the lag between write and read model update. User creates order, then immediate read shows nothing.

### Mistake 4: Command Returning Data
Commands should be void (return ID only). If commands return data, they're becoming queries.

### Mistake 5: Duplicating Business Logic in Query Handlers
Query handlers should not duplicate validation or calculation logic from write side.

## 9. Senior Engineer Perspective

### CQRS + Event Sourcing Synergy
CQRS pairs naturally with Event Sourcing:
- Write side: append events to event store.
- Read side: project events to materialized views.
- Read models can be rebuilt by replaying events from scratch.

### Read Model Rebuilding
```java
@Component
public class ReadModelRebuilder {

    @Autowired
    private EventStore eventStore;

    @Autowired
    private ApplicationContext applicationContext;

    public void rebuildAll() {
        // Drop and recreate read model tables
        summaryRepository.deleteAll();

        // Replay all events
        List<Event> allEvents = eventStore.getAllEvents();
        for (Event event : allEvents) {
            // Find and invoke appropriate event handler
            publishEvent(event);
        }
    }

    private void publishEvent(Event event) {
        // Use ApplicationEventPublisher to dispatch
        // Each read model updater handles its relevant events
    }
}
```

### When to Use Separate Databases
- **Same database, different tables**: Simple CQRS, eventual consistency is fine.
- **Same database type, different instances**: Read replicas for scale.
- **Different database types**: Optimize each for workload (PostgreSQL for writes, Elasticsearch for reads).

## 10. Interview Questions (20: 10 easy + 10 medium)

### Easy

1. **Q:** What does CQRS stand for?
   **A:** Command Query Responsibility Segregation.

2. **Q:** What is the difference between a command and a query?
   **A:** A command changes state and returns no data. A query returns data and does not change state.

3. **Q:** What is the main benefit of CQRS?
   **A:** Independent optimization of read and write models for their specific workloads.

4. **Q:** Does CQRS require separate databases?
   **A:** No, it requires separate models. They can use the same database, different tables, or different databases.

5. **Q:** What is a command handler?
   **A:** A component that receives a command, validates it, executes business logic, and persists changes.

6. **Q:** What is a read model in CQRS?
   **A:** A denormalized data structure optimized for query performance.

7. **Q:** Is CQRS always used with Event Sourcing?
   **A:** No, they are independent patterns. CQRS can be used without event sourcing.

8. **Q:** What is eventual consistency in CQRS?
   **A:** The read model is updated asynchronously after the write, so there is a small delay before changes are visible.

9. **Q:** Can CQRS help with performance?
   **A:** Yes, by optimizing read queries with denormalized data and separate indexing.

10. **Q:** What is a materialized view in CQRS?
    **A:** A pre-computed read model projection that contains data structured for specific query patterns.

### Medium

11. **Q:** How do you handle validation in CQRS commands?
    **A:** Command handlers validate input, check business rules, and throw exceptions for invalid state.

12. **Q:** How do you update the read model when the write side changes?
    **A:** Write side publishes events after state change. Event handlers subscribe and update the read model accordingly.

13. **Q:** What is the difference between CQRS and CRUD?
    **A:** CRUD uses a single model for all operations. CQRS separates commands (CUD) from queries (R) into distinct models.

14. **Q:** How would you handle a read model that is out of sync with the write model?
    **A:** Implement read model rebuilding from events, detect sync lag, and provide staleness information to clients.

15. **Q:** What is the role of events in CQRS?
    **A:** Events communicate write-side state changes to update read models and trigger side effects.

16. **Q:** How do you implement pagination in CQRS queries?
    **A:** Query parameters include page/size/cursor. Query handler uses database pagination or cursor-based pagination on the read model.

17. **Q:** What is the difference between CQRS and Command-Query Separation (CQS)?
    **A:** CQS is a class-level principle (methods are either commands or queries). CQRS is an architectural pattern with separate models and often separate databases.

18. **Q:** How do you handle transactions across write and read models?
    **A:** You don't. Write transaction completes, event is published, read model updates asynchronously.

19. **Q:** What is a projection in CQRS?
    **A:** A projection transforms events into a read model view. It subscribes to events and updates the query database.

20. **Q:** How does CQRS help team scalability?
    **A:** Different teams can work on command model (domain logic) and query model (performance optimization) independently.

## 11. Advanced Interview Questions (20: 10 hard + 10 system design)

### Hard

1. **Q:** How do you maintain consistency between write and read models under high load?
    **A:** Use idempotent event processing, exactly-once delivery, versioned events. For critical consistency, use transactional outbox + synchronous read model update. Monitor lag and alert on thresholds.

2. **Q:** Design a CQRS system that supports rebuilding read models from an event store with zero downtime.
    **A:** Create a new read model version (e.g., v2) in parallel. Start projecting events from event store to v2. When v2 catches up to real-time, switch queries to v2. Drop v1. Use database views or blue-green read model deployments.

3. **Q:** How do you handle commands that affect multiple aggregates?
    **A:** Use Saga pattern: a long-running process with compensating actions. Command creates saga, which coordinates commands across aggregates via events.

4. **Q:** Design a CQRS system for a real-time bidding platform.
    **A:** Write side: BidCommand processed sequentially by auction ID (Kafka partition). Bid aggregate validates bid > current price. Write to event store. Read side: Current auction state in Redis (sorted set of bids). Stream processor updates Redis from events. Query: webSocket push current state to bidders.

5. **Q:** How do you handle partial failures in read model updates?
    **A:** DLQ for failed events with retry. Idempotent event handlers ensure safe retry. Rebuild read model from event store if corruption. Monitor lag and alert.

6. **Q:** What is the difference between event notification and event sourcing in CQRS?
    **A:** Event notification: events just trigger read model updates (older events lost). Event sourcing: all events stored permanently, read model rebuilt by replaying events from beginning.

7. **Q:** How do you version read models in CQRS?
    **A:** Read model schema version in metadata. Multiple read model versions running in parallel during migration. Version-aware event handlers produce to appropriate version.

8. **Q:** Design a CQRS system where read and write models use different database technologies.
    **A:** Write: PostgreSQL with ACID transactions. Read: Elasticsearch for search, Redis for caching, MongoDB for flexible projections. Event bus (Kafka) for async synchronization.

9. **Q:** How do you implement authorization at both command and query level?
    **A:** Command handlers check permissions before executing mutation. Query handlers filter results based on user's access scope. Use attribute-based access control (ABAC) for fine-grained permissions.

10. **Q:** Design a CQRS system that supports GDPR data deletion.
    **A:** Write side: delete command sets user data to anonymized state. Event store: encrypt PII events, on deletion store anonymization event (not deletion). Read model: rebuild excluding anonymized user data.

### System Design

11. **Q:** Design an e-commerce platform using CQRS.
    **A:** Write: Order aggregate (state machine), Product aggregate (inventory), Cart aggregate. Write DB: PostgreSQL. Read: OrderSummary (denormalized, including customer name, product names), ProductSearch (Elasticsearch), CartView (Redis). Event bus: Kafka, CDC (Debezium) for read model update.

12. **Q:** Design a flight booking system with CQRS.
    **A:** Write: Flight aggregate (capacity management), Booking aggregate (reservation flow). Strong consistency on capacity. Read: FlightSearch (Elasticsearch - cached prices, availability), BookingHistory (MongoDB - user-friendly format). Saga for booking flow: reserve -> pay -> confirm.

13. **Q:** Design a social media news feed using CQRS.
    **A:** Write: Post aggregate, Follow relationship. Read: Feed (Redis list per user, pre-computed on write). Timeline (MongoDB - paginated query). CQRS for feed: fan-out-on-write stores posts in followers' feed lists.

14. **Q:** Design an inventory management system with CQRS.
    **A:** Write: Inventory aggregate (strong consistency on stock count). Read: InventorySummary (Redis sorted sets by stock level for alerts), InventoryReport (time-series data in Cassandra). Event-sourced inventory changes for audit.

15. **Q:** Design a hotel reservation system using CQRS.
    **A:** Write: Room aggregate (calendar of availability), Reservation aggregate. Saga: book room -> charge card -> confirm. Read: RoomSearch (denormalized availability calendar), ReservationHistory (customer view). Caching at query layer.

16. **Q:** Design a banking system using CQRS and Event Sourcing.
    **A:** Write: Account aggregate (events: Deposited, Withdrawn, Transferred). Event Store: PostgreSQL or EventStoreDB. Read: AccountBalance (read model updated via projections), TransactionHistory (SQL or MongoDB). Query: balance, statements, transaction search.

17. **Q:** Design a content management system with CQRS.
    **A:** Write: Document aggregate (versioned, approval workflow). Read: PublishedContent (rendered HTML in Redis), DocumentSearch (Elasticsearch). Draft read model for editors. Multiple read models per content status.

18. **Q:** Design a multi-tenant SaaS platform with CQRS.
    **A:** Tenant isolation: database per tenant for writes, shared Elasticsearch with tenant filter for reads. Write models per tenant with custom validation rules. Read models with tenant-specific projections. Event schema includes tenant ID.

19. **Q:** Design a healthcare records system with CQRS.
    **A:** Write: PatientRecord aggregate (event sourced for audit). Strict access control on commands. Read: PatientSummary (denormalized), ClinicalDashboard (real-time vitals). HIPAA compliance: encrypt PII in events, audit trail of all reads/writes.

20. **Q:** Design a real-time analytics dashboard using CQRS.
    **A:** Write: Raw event ingestion (high throughput, Kafka). Stream processor aggregates events (tumbling windows). Write aggregated results to OLAP store. Read: Dashboard queries from pre-aggregated data. Multiple granularity read models (1min, 1h, 1d aggregates).

## 12. Expert-Level Interview Questions (10: architect-level)

1. **Q:** Design a globally distributed CQRS system with active-active writes and local reads.
    **A:** Each region has a local write leader (Kafka partition leader). CRDTs for conflict resolution. Events replicated across regions asynchronously. Local read models built from local events + replicated cross-region events. Global read models use merged event streams. Conflict resolution strategies: last-writer-wins, CRDT merge, or application-specific reconciliation.

2. **Q:** How would you design CQRS for a system requiring both strong consistency for some queries and eventual consistency for others?
    **A:** Hybrid approach: critical queries read from write model synchronously (same DB). Non-critical queries read from optimized read model. Distinguish via query annotations (`@StrongConsistency` vs `@EventualConsistency`). Route to appropriate data source in query handler.

3. **Q:** Design a CQRS system that handles schema evolution across read and write models for years.
    **A:** Write schema: versioned events with schema registry (Avro/Protobuf). Read schemas: versioned projections. Multiple read model versions running in parallel during migration. Event schema evolution: backward/forward compatible. Read model evolution: new projection version processes events to new format.

4. **Q:** How do you implement a CQRS system with 50+ different read model projections?
    **A:** Projection orchestration: each projection subscribes to relevant events. Projection versioning and parallel run. Selective projection rebuild (by event type or time range). Monitoring per-projection lag. Shared projection infrastructure (stream processing framework like Kafka Streams, Apache Flink).

5. **Q:** Design a CQRS system where writes need to be validated against read model data.
    **A:** Anti-pattern warning: commands should not depend on read model. Instead, validate from write model (aggregate state). For performance, cache write model state in write-optimized form. If cross-aggregate validation needed, use domain service that queries write-side repository.

6. **Q:** How do you implement a CQRS system that provides exactly-once guarantees for read model updates?
    **A:** Idempotent event processing: store processed event IDs in read model transaction. Unique constraint on event ID in read model tables. Use Kafka exactly-once semantics with transactional API. Read model update within same transaction as dedup mark.

7. **Q:** Design a CQRS system for a financial risk platform requiring sub-millisecond reads on complex aggregations.
    **A:** Pre-compute all aggregations in read model during event processing. Use Redis or in-memory data grid for sub-millisecond lookups. Multiple read model granularities: raw trades, per-symbol aggregates, portfolio-level risk. Incremental updates on each event (no full recomputation).

8. **Q:** How do you handle backpressure in CQRS read model updating?
    **A:** Event store with retention (Kafka). Consumer lag monitoring. Auto-scale projection workers based on lag. Prioritize critical projections. Drop non-critical projections during load spikes (recover later). Dead letter queue for failed events.

9. **Q:** Design a CQRS system that supports event-sourced aggregates with thousands of events per aggregate.
    **A:** Snapshot strategy: store aggregate snapshot every N events (e.g., 100). On load: restore from latest snapshot, replay remaining events. Snapshot store in fast DB (Redis + PostgreSQL). Configurable snapshot frequency per aggregate type.

10. **Q:** How would you design a CQRS system for a multi-tenant platform where each tenant can define custom fields and read models?
    **A:** Write side: dynamic schema events with tenant-specific metadata. Read side: tenant-specific projections stored in document DB (MongoDB). Tenant schema registry for field definitions. Query handler constructs dynamic queries based on tenant schema. Event processing: tenant-specific stream processors.

## 13. Debugging & Troubleshooting

### Common CQRS Issues

**Issue: Read model out of sync with write model**
- Check event processing lag (Kafka consumer lag).
- Check event handler logs for errors.
- Verify event was published (check event store).
- Rebuild read model if needed.

**Issue: Slow command processing**
- Check database transaction times.
- Check for unnecessary event publication overhead.
- Profile aggregate loading (too many events to replay).

**Issue: Query returning stale data**
- Check TTL on read model cache.
- Check if event handler failed silently.
- Check event ordering (partition assignment issues).

### Debugging Commands
```bash
# Check Kafka consumer lag for read model updaters
kafka-consumer-groups --bootstrap-server kafka:9092 \
  --group order-read-model --describe

# Check event store for specific aggregate
curl -X GET "http://event-store/api/events/order/order-123"

# Query read model directly
psql -h read-db -c "SELECT * FROM order_summaries WHERE order_id = 'order-123'"

# Compare write and read models
diff <(psql -h write-db -c "SELECT id, status FROM orders WHERE id='order-123'") \
     <(psql -h read-db -c "SELECT order_id, status FROM order_summaries WHERE order_id='order-123'")
```

## 14. Comparison Section

### CQRS vs Traditional CRUD

| Aspect | CQRS | Traditional CRUD |
|--------|------|-----------------|
| Model | Separate read/write | Single model |
| Optimization | Per workload | Compromise |
| Consistency | Eventually consistent | Strongly consistent |
| Complexity | Higher | Lower |
| Scalability | Higher (independent) | Limited |
| Query Performance | Optimized (denormalized) | May need joins |
| Audit | Natural (events) | Requires extra |
| Best for | Complex domain, scale | Simple CRUD apps |

### CQRS vs Event Sourcing

| Aspect | CQRS | Event Sourcing |
|--------|------|---------------|
| Focus | Separate read/write models | State as event sequence |
| Storage | Normalized write + denormalized read | Append-only event store |
| Audit | Via events | Built-in |
| Complexity | Medium | High |
| Dependence | Can be used alone | Often used with CQRS |

### CQRS + Event Sourcing vs CQRS + CRUD Write Model

| Aspect | CQRS + Event Sourcing | CQRS + CRUD Write |
|--------|----------------------|-------------------|
| Audit | Full event history | Requires logging |
| Rebuild | Replay all events | Not possible |
| Temporal | Any point-in-time state | Current state only |
| Complexity | Higher | Lower |
| Event Store | Required | Not required |
| Storage | More (all events) | Less (current state) |

## 15. Revision Notes

### Quick Recap
- **Command**: Mutation (create, update, delete). Void return.
- **Query**: Data retrieval. No side effects.
- **Separate Models**: Write model (domain logic) != Read model (query optimization).
- **Eventual Consistency**: Read model updates async after write.
- **Event Bus**: Communicates changes from write to read side.
- **Projection**: Transforms events into read model state.
- **When to Use**: Different read/write workloads, complex domain, need independent scaling.
- **When NOT to Use**: Simple CRUD, strong consistency always needed, small team.

### Key Design Rules
1. Commands change state; queries return state.
2. Commands are named imperatively (`SubmitOrder`); queries declaratively (`GetOrder`).
3. Read models are denormalized and optimized for specific queries.
4. Use events to synchronize write -> read.
5. Monitor read model lag.
6. Rebuild read models from event history when schema changes.

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                      CQRS CHEAT SHEET                              |
+-------------------------------------------------------------------+
| PATTERN             | PURPOSE                           | EXAMPLE  |
+---------------------+-----------------------------------+----------+
| Command             | Change state                      | Create-  |
|                     |                                   | OrderCmd |
| Query               | Retrieve data                     | GetOrder |
|                     | (no side effects)                 | Query    |
| Command Handler     | Validates + executes command      | Create-  |
|                     |                                   | OrderCmd |
|                     |                                   | Handler  |
| Query Handler       | Fetches from read model           | GetOrder |
|                     |                                   | Query    |
|                     |                                   | Handler  |
| Read Model          | Denormalized query structures     | Order-   |
|                     |                                   | Summary  |
| Write Model         | Domain logic + invariants         | Order    |
|                     |                                   | Aggregate|
| Event               | Communication from write to read  | Order-   |
|                     |                                   | Created  |
| Projection          | Transforms events -> read model   | Order-   |
|                     |                                   | Projector|
+---------------------+-----------------------------------+----------+
| ARCHITECTURE                                                     |
+-------------------------------------------------------------------+
| WRITE SIDE          | READ SIDE                                   |
| [Command]           | [Query]                                     |
|   v                 |   v                                         |
| [Command Handler]   | [Query Handler]                             |
|   v                 |   v                                         |
| [Aggregate/Domain]  | [Read Model (denormalized)]                 |
|   v                 |                                             |
| [Event Store/DB]    | [Cache (Redis)] [Search (ES)] [DB]          |
|   v                 |                                             |
| [Event Bus/Kafka] -----> [Event Handler/Projection]               |
+-------------------------------------------------------------------+
| CONSISTENCY                                                        |
+-------------------------------------------------------------------+
| Write always consistent (ACID on write side)                      |
| Read model eventually consistent (async update)                   |
| Lag = time between event publication and read model update        |
| Typical: 10ms - 100ms                                             |
| Can bypass: query write model directly for critical reads         |
+-------------------------------------------------------------------+
| BEST PRACTICES                                                     |
+-------------------------------------------------------------------+
| 1. Commands return void (or ID only)                              |
| 2. Queries are idempotent                                         |
| 3. Read models are denormalized                                   |
| 4. Event handlers are idempotent                                  |
| 5. Monitor read model lag                                         |
| 6. Support read model rebuilding                                  |
| 7. Version events and read models                                 |
| 8. Never use read model for command validation                    |
+-------------------------------------------------------------------+
```
