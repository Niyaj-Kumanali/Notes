# Event-Driven Architecture

## 1. Executive Summary

Event-Driven Architecture (EDA) is a software design pattern where services communicate through the production, detection, consumption, and reaction to events. Unlike request-response patterns (REST), EDA decouples event producers from consumers through an intermediary event broker (Kafka, RabbitMQ, AWS EventBridge). This enables asynchronous, scalable, and loosely coupled systems ideal for distributed and reactive applications.

## 2. Core Theory

### Key Concepts
- **Event:** A significant change in state or occurrence. Events are immutable facts that have happened.
- **Event Producer:** Service that detects state changes and publishes events.
- **Event Consumer:** Service that subscribes to and processes events.
- **Event Broker:** Middleware that receives, stores, and routes events (Kafka, RabbitMQ, Pulsar).
- **Channel:** The communication medium (topic, queue, stream).

### Event Types
1. **Event Notification:** Lightweight notification that something happened. Contains minimal data (ID, type, timestamp). Consumers fetch additional data if needed.
2. **Event-Carried State Transfer:** Event contains all relevant data for processing. Consumers don't need to call back the producer.
3. **Event Sourcing:** Every state change is stored as an event. Current state is derived by replaying events.

### Communication Patterns
- **Publish/Subscribe:** One-to-many. One event, multiple consumers.
- **Point-to-Point:** One-to-one. Competing consumers pattern.
- **Fan-Out:** Event is delivered to all subscribers.
- **Dead Letter Queue (DLQ):** Failed events routed for later analysis.

## 3. Under-the-Hood Deep Dive

### Event Processing Semantics
- **At-most-once:** Fire and forget. Event may be lost.
- **At-least-once:** Event delivered at least once. May have duplicates.
- **Exactly-once:** Event delivered exactly once. Hard to achieve in distributed systems. Requires idempotent consumers + deduplication.

### Message Ordering
- **Kafka:** Ordered within a partition. Global ordering requires single partition (limits parallelism).
- **RabbitMQ:** No ordering guarantee by default. Use single consumer or consistent hashing exchange.

### Exactly-Once Semantics with Idempotent Consumers

```java
@Component
public class IdempotentEventConsumer {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    @Autowired
    private OrderRepository orderRepository;

    private static final Duration DEDUP_WINDOW = Duration.ofMinutes(5);

    @KafkaListener(topics = "payment.events", groupId = "order-service")
    public void handlePaymentCompleted(PaymentCompletedEvent event) {
        String dedupKey = "dedup:payment:" + event.getEventId();

        // Check if already processed
        Boolean alreadyProcessed = redisTemplate.opsForValue()
            .setIfAbsent(dedupKey, "processed", DEDUP_WINDOW);

        if (Boolean.FALSE.equals(alreadyProcessed)) {
            log.warn("Duplicate event: {}", event.getEventId());
            return;
        }

        // Process event
        orderRepository.findById(event.getOrderId()).ifPresent(order -> {
            order.setStatus(OrderStatus.PAID);
            order.setPaymentId(event.getPaymentId());
            orderRepository.save(order);
        });
    }
}
```

### Event Sourcing and CQRS

```java
// Event Store - append-only log of events
@Service
public class EventStore {

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    private static final String EVENT_STORE_TOPIC = "event-store.account";

    public void appendEvent(AccountEvent event) {
        kafkaTemplate.send(
            EVENT_STORE_TOPIC,
            event.getAccountId(),
            event
        );
    }

    public List<AccountEvent> getEvents(String accountId) {
        // Read events from Kafka topic (using consumer)
        // For production, use a dedicated event store DB
        return eventRepository.findByAccountIdOrderByVersion(accountId);
    }
}

// Aggregate root reconstructs state from events
public class Account {
    private String id;
    private BigDecimal balance;
    private int version;

    public static Account replay(List<AccountEvent> events) {
        Account account = new Account();
        for (AccountEvent event : events) {
            account.apply(event);
        }
        return account;
    }

    private void apply(AccountEvent event) {
        switch (event.getType()) {
            case ACCOUNT_CREATED -> this.balance = BigDecimal.ZERO;
            case FUNDS_DEPOSITED -> this.balance = this.balance.add(
                ((FundsDepositedEvent) event).getAmount());
            case FUNDS_WITHDRAWN -> this.balance = this.balance.subtract(
                ((FundsWithdrawnEvent) event).getAmount());
        }
        this.version = event.getVersion();
    }
}
```

## 4. Production Code Examples

### Kafka Event Producer/Consumer

```java
// Event
@Data
@Builder
public class OrderCreatedEvent {
    private String eventId;
    private Long orderId;
    private Long userId;
    private BigDecimal totalAmount;
    private List<OrderItem> items;
    private Instant timestamp;
}

// Producer
@Service
public class OrderEventProducer {

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    private static final String TOPIC = "order.events";

    public void publishOrderCreated(Order order) {
        OrderCreatedEvent event = OrderCreatedEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .orderId(order.getId())
            .userId(order.getUserId())
            .totalAmount(order.getTotalAmount())
            .items(order.getItems())
            .timestamp(Instant.now())
            .build();

        // Use order ID as key for partition ordering
        kafkaTemplate.send(TOPIC, order.getId().toString(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to send event: {}", event.getEventId(), ex);
                } else {
                    log.debug("Event sent: {} at offset {}",
                        event.getEventId(), result.getRecordMetadata().offset());
                }
            });
    }
}

// Consumer
@Service
public class OrderEventConsumer {

    @Autowired
    private InventoryService inventoryService;

    @Autowired
    private NotificationService notificationService;

    @KafkaListener(
        topics = "order.events",
        groupId = "inventory-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void handleOrderCreated(OrderCreatedEvent event) {
        log.info("Processing order created event: {}", event.getOrderId());

        // Reserve inventory for each item
        event.getItems().forEach(item ->
            inventoryService.reserve(item.getProductId(), item.getQuantity()));

        // Send notification
        notificationService.sendOrderConfirmation(event.getOrderId(), event.getUserId());
    }

    @KafkaListener(
        topics = "order.events",
        groupId = "analytics-service"
    )
    public void handleOrderCreatedForAnalytics(OrderCreatedEvent event) {
        // Separate consumer group for analytics - processes independently
        analyticsService.recordOrder(event);
    }
}
```

### RabbitMQ Event Configuration

```java
@Configuration
public class RabbitMQConfig {

    public static final String ORDER_EXCHANGE = "order.exchange";
    public static final String INVENTORY_QUEUE = "inventory.queue";
    public static final String NOTIFICATION_QUEUE = "notification.queue";
    public static final String DEAD_LETTER_QUEUE = "order.dlq";

    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange(ORDER_EXCHANGE);
    }

    @Bean
    public Queue inventoryQueue() {
        return QueueBuilder.durable(INVENTORY_QUEUE)
            .withArgument("x-dead-letter-exchange", "")
            .withArgument("x-dead-letter-routing-key", DEAD_LETTER_QUEUE)
            .build();
    }

    @Bean
    public Queue notificationQueue() {
        return QueueBuilder.durable(NOTIFICATION_QUEUE).build();
    }

    @Bean
    public Binding inventoryBinding() {
        return BindingBuilder.bind(inventoryQueue())
            .to(orderExchange())
            .with("order.created.*");
    }

    @Bean
    public Binding notificationBinding() {
        return BindingBuilder.bind(notificationQueue())
            .to(orderExchange())
            .with("order.#"); // Matches order.created, order.cancelled, etc.
    }
}
```

### Event-Driven Saga Orchestrator

```java
@Service
public class OrderSagaOrchestrator {

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    private static final String SAGA_TOPIC = "saga.order";

    @KafkaListener(topics = "saga.order", groupId = "saga-orchestrator")
    public void handleSagaStep(SagaEvent event) {
        switch (event.getStep()) {
            case RESERVE_INVENTORY -> startInventoryReservation(event);
            case PROCESS_PAYMENT -> startPaymentProcessing(event);
            case CONFIRM_ORDER -> confirmOrder(event);
            case COMPENSATE -> compensate(event);
        }
    }

    private void startInventoryReservation(SagaEvent event) {
        try {
            // Publish reserve inventory command
            kafkaTemplate.send("inventory.commands", event);
        } catch (Exception e) {
            // Compensation
            publishCompensation(event, SagaStep.RESERVE_INVENTORY);
        }
    }

    private void publishCompensation(SagaEvent event, SagaStep failedStep) {
        SagaEvent compensationEvent = SagaEvent.builder()
            .sagaId(event.getSagaId())
            .orderId(event.getOrderId())
            .step(SagaStep.COMPENSATE)
            .failedStep(failedStep)
            .build();
        kafkaTemplate.send(SAGA_TOPIC, compensationEvent);
    }

    private void compensate(SagaEvent event) {
        // Execute compensating actions in reverse order
        switch (event.getFailedStep()) {
            case PROCESS_PAYMENT -> refundPayment(event);
            case RESERVE_INVENTORY -> releaseInventory(event);
        }
    }
}
```

### Event-Driven Outbox Pattern

```java
// Entity with outbox
@Entity
@Table(name = "orders")
public class Order {
    @Id
    private Long id;
    private Long userId;
    private BigDecimal totalAmount;
    private String status;

    @OneToMany(cascade = ALL)
    @JoinColumn(name = "order_id")
    private List<OutboxEvent> outboxEvents = new ArrayList<>();

    public void addOutboxEvent(String type, String payload) {
        this.outboxEvents.add(OutboxEvent.builder()
            .eventId(UUID.randomUUID().toString())
            .aggregateType("order")
            .aggregateId(this.id.toString())
            .eventType(type)
            .payload(payload)
            .build());
    }
}

// Outbox table
@Entity
@Table(name = "outbox_events")
@Data
@Builder
public class OutboxEvent {
    @Id
    private String eventId;
    private String aggregateType;
    private String aggregateId;
    private String eventType;
    private String payload;
    @Enumerated(EnumType.STRING)
    private OutboxStatus status;
    private Instant createdAt;
}

// Outbox publisher - polls and publishes
@Component
public class OutboxPublisher {

    @Autowired
    private OutboxEventRepository outboxRepository;

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    @Scheduled(fixedDelay = 1000)
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> pendingEvents = outboxRepository
            .findByStatusOrderByCreatedAt(OutboxStatus.PENDING);

        for (OutboxEvent event : pendingEvents) {
            try {
                kafkaTemplate.send(
                    event.getEventType(),
                    event.getAggregateId(),
                    event.getPayload()
                ).get(5, TimeUnit.SECONDS);

                event.setStatus(OutboxStatus.PUBLISHED);
                outboxRepository.save(event);
            } catch (Exception e) {
                log.error("Failed to publish outbox event: {}", event.getEventId(), e);
                event.setStatus(OutboxStatus.FAILED);
                outboxRepository.save(event);
            }
        }
    }
}
```

## 5. Real-World Scenarios

### Order Processing Pipeline
```
[Order Service] --OrderCreated--> [Kafka: order.events]
                                      |
                    +-----------------+------------------+
                    |                                    |
            [Inventory Service]                   [Payment Service]
            (reserve stock)                       (process payment)
                    |                                    |
                    +----------> [Kafka: saga.events] <--+
                                      |
                              [Notification Service]
                              (email/SMS confirmation)
```

### Real-Time Analytics Pipeline
```
[Web App Events] -> [Kafka: raw.events]
                        |
                   [Flink/KSQL]
                        |
           +------------+------------+
           |            |            |
    [Clickhouse]   [Redis]     [S3 Archive]
    (real-time     (dashboards)  (historical)
     analytics)
```

## 6. Performance

### Throughput Comparison

| Broker | Throughput | Latency | Persistence | Ordering |
|--------|-----------|---------|-------------|----------|
| Kafka | 1M+ msg/s | ~10ms | Durable disk | Per partition |
| RabbitMQ | 100K msg/s | ~1ms | Optional | Weak |
| Pulsar | 1M+ msg/s | ~5ms | Segment store | Per partition |
| AWS SQS | 10K msg/s | ~100ms | 14 day retention | Best-effort |

### Scaling Considerations
- Increase partitions for higher throughput.
- Batch consumption reduces per-message overhead.
- Compression (Snappy, Zstd) reduces network/storage.
- Tune fetch size, linger time, and batch size for throughput vs latency.

### Kafka Producer Tuning
```yaml
spring:
  kafka:
    producer:
      acks: all
      batch-size: 65536
      linger-ms: 5
      compression-type: snappy
      properties:
        max.in.flight.requests.per.connection: 5
        enable.idempotence: true
```

## 7. Security

### Event Encryption

```java
@Configuration
public class EncryptionConfig {

    @Bean
    public Serializer<Object> encryptingSerializer() {
        return new AesGcmSerializer("encryption-key-32bytes");
    }

    @Bean
    public Deserializer<Object> encryptingDeserializer() {
        return new AesGcmDeserializer("encryption-key-32bytes");
    }
}
```

### Authentication
- Kafka: SASL/PLAIN, SCRAM, or SSL authentication.
- RabbitMQ: username/password, LDAP, or SSL certificates.
- AWS EventBridge: IAM roles and policies.

### Authorization
Schema Registry with authorization: only trusted producers can register schemas.
Consumer group authorization: restrict which consumers can read which topics.

### Best Practices
- Encrypt sensitive fields in events (PII, payment data).
- Use mTLS for broker-to-client communication.
- Audit log of all event publications/consumptions.
- Event schemas with data classification metadata.

## 8. Common Mistakes

### Mistake 1: Over-coupling Events to Consumer Expectations
Events should be "facts that happened" not "commands for specific consumers".

### Mistake 2: Lack of Schema Management
Without a schema registry, producer/consumer contracts break silently.

### Mistake 3: No Error Handling in Consumers
```java
// WRONG - unhandled exception stops the consumer
@KafkaListener(topics = "orders")
public void handle(String event) {
    process(event); // throws RuntimeException -> consumer stops
}

// RIGHT - catch and handle failures
@KafkaListener(topics = "orders")
public void handle(String event) {
    try {
        process(event);
    } catch (Exception e) {
        // Log, send to DLQ, or retry later
        kafkaTemplate.send("orders.dlq", event);
        log.error("Failed to process event", e);
    }
}
```

### Mistake 4: Assuming Message Ordering Across Partitions
Kafka only guarantees ordering within a partition. Multi-partition ordering requires application-level coordination.

### Mistake 5: Synchronous Blocking in Async Event Handlers
```java
// WRONG - blocks the consumer thread
@KafkaListener(topics = "orders")
public void handle(String event) {
    restTemplate.getForEntity("http://slow-api", String.class); // blocks
    process(event);
}

// RIGHT - use async processing
@KafkaListener(topics = "orders")
public void handle(String event) {
    CompletableFuture.runAsync(() -> process(event));
}
```

## 9. Senior Engineer Perspective

### Event Schema Evolution Strategy

1. **Backward Compatible:** New schema can read old data. Add optional fields only.
2. **Forward Compatible:** Old schema can read new data. Never remove fields.
3. **Full Compatible:** Both directions. Add optional fields, never remove or make required.

Use Avro/Protobuf with Schema Registry. Evolve schemas by adding fields with defaults.

### Event Versioning Approaches
- **Schema Registry:** Single schema with version field in event metadata.
- **Separate Topics:** `orders.v1`, `orders.v2` - run both until consumers migrate.
- **Event Envelope:** `{ "type": "OrderCreated", "version": 2, "data": {...} }`.

### Strategic Event Design
- Events are past-tense, immutable facts: `OrderCreated`, `PaymentReceived`.
- Commands are future-tense requests: `ReserveInventory`, `SendEmail`.
- Separate event topics by domain: `order.events`, `payment.events`.
- Event size: include enough data for autonomous processing (event-carried state transfer).

### Dead Letter Queue Strategy
```java
@Component
@Slf4j
public class DeadLetterQueueHandler {

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    private static final String DLQ_TOPIC = "events.dlq";

    public void sendToDLQ(String originalTopic, Object event, Exception cause) {
        DLEvent dlqEvent = DLEvent.builder()
            .originalTopic(originalTopic)
            .originalEvent(event)
            .errorMessage(cause.getMessage())
            .errorType(cause.getClass().getName())
            .timestamp(Instant.now())
            .retryCount(0)
            .build();

        kafkaTemplate.send(DLQ_TOPIC, dlqEvent);
    }

    @Scheduled(fixedDelay = 300000) // Every 5 minutes
    public void retryDLQEvents() {
        // Re-process events from DLQ with backoff
        // After max retries, move to permanent dead storage
    }
}
```

## 10. Interview Questions (20: 10 easy + 10 medium)

### Easy

1. **Q:** What is an event-driven architecture?
   **A:** An architecture where services communicate by producing and consuming events, decoupling producers from consumers.

2. **Q:** What is the difference between event and command?
   **A:** Event is a fact that happened in the past (immutable). Command is a request for action (may be rejected).

3. **Q:** What is pub/sub messaging?
   **A:** A pattern where publishers send events to topics/channels and subscribers receive all events published to subscribed topics.

4. **Q:** What is a message broker?
   **A:** Middleware that receives events from producers and routes them to consumers (Kafka, RabbitMQ, Pulsar).

5. **Q:** What is the difference between Kafka and RabbitMQ?
   **A:** Kafka is a distributed log optimized for high-throughput, persistent streaming. RabbitMQ is a message broker optimized for flexible routing and low latency.

6. **Q:** What is a consumer group in Kafka?
   **A:** A set of consumers that collaboratively consume a topic. Each partition is assigned to one consumer in the group.

7. **Q:** What is at-least-once delivery?
   **A:** Every event is delivered at least once to consumers, but may be duplicated.

8. **Q:** What is exactly-once delivery?
   **A:** Every event is delivered exactly once, requiring idempotent consumers and deduplication.

9. **Q:** What is event sourcing?
   **A:** Storing all state changes as a sequence of events. Current state is derived by replaying events.

10. **Q:** What is the outbox pattern?
    **A:** Storing events in a database table within the same transaction as the state change, then reliably publishing them to the message broker.

### Medium

11. **Q:** How do you handle event ordering in Kafka?
    **A:** Use the same partition key for related events (e.g., order ID). Order is guaranteed within a partition.

12. **Q:** What is the dead letter queue pattern?
    **A:** A queue where failed events are routed for later inspection, retry, or manual processing.

13. **Q:** How do you handle duplicate events?
    **A:** Idempotent event processing: store processed event IDs in a deduplication store (Redis, DB) with TTL.

14. **Q:** Explain the saga pattern in event-driven architecture.
    **A:** A sequence of local transactions with compensating actions on failure. Orchestrated (central coordinator) or choreographed (event-driven).

15. **Q:** What is event-carried state transfer?
    **A:** Events contain all the data needed for processing, so consumers don't need to call back to the producer.

16. **Q:** How do you version events in a message broker?
    **A:** Schema Registry (Avro/Protobuf versioned schemas), envelope with version field, or separate topics per version.

17. **Q:** What is the difference between topic and queue?
    **A:** Topic: fan-out delivery to all subscribers. Queue: point-to-point delivery to one consumer.

18. **Q:** How do you monitor Kafka consumer lag?
    **A:** Using `kafka-consumer-groups --describe --group groupId`, Prometheus metrics, or Kafka Manager UI.

19. **Q:** What is backpressure in event-driven systems?
    **A:** When consumers cannot keep up with producers. Handled via consumer lag monitoring, rate limiting, or scaling consumers.

20. **Q:** How does event-driven architecture improve scalability?
    **A:** Decouples producers and consumers, allows independent scaling, buffering via message queues, and parallel processing.

## 11. Advanced Interview Questions (20: 10 hard + 10 system design)

### Hard

1. **Q:** Design an event ordering mechanism that guarantees total order across all partitions.
    **A:** Use a single partition for ordered events (limits throughput), or assign global sequence numbers (sequencer service with atomic increments), or use a distributed coordination service (ZooKeeper). For practical purposes, per-key ordering (partition by aggregate ID) is usually sufficient.

2. **Q:** How do you implement exactly-once semantics across multiple Kafka topics?
    **A:** Use Kafka's transactional API: producer sends events across partitions/topics atomically within a transaction. Consumer reads committed transactions. This requires `isolation.level=read_committed` and idempotent producer.

3. **Q:** Design a retry mechanism with exponential backoff for event processing failures.
    **A:** Use a retry topic with delayed consumption. Schedule delivery using Kafka's timestamps: failed event published to retry topic with future timestamp. Consumer filters events by timestamp. Retry count tracked in event headers. After max retries, send to DLQ.

4. **Q:** How do you handle schema evolution in a streaming platform with 100+ event types?
    **A:** Schema Registry (Confluent/Apicurio) with Avro. Backward/forward compatibility rules enforced at registry level. Automated CI/CD check: any schema change must pass compatibility validation. Deprecated events have min/max version info for consumers.

5. **Q:** Explain the competing consumers pattern and its trade-offs.
    **A:** Multiple consumers process from the same queue, each message handled once. Increases throughput but loses ordering. Trade-offs: ordering lost, need idempotent consumers, more complex error handling.

6. **Q:** Design a system for event replay that can rebuild state across 50 services.
    **A:** Maintain global event store (Kafka with infinite retention or S3 archive). Services snapshot state periodically (every N events or time). On replay: restore latest snapshot, then replay subsequent events from event store. Replay orchestrator coordinates point-in-time consistency across services.

7. **Q:** How do you implement cross-datacenter event replication?
    **A:** Kafka MirrorMaker 2: replicate topics from primary to secondary datacenter. Active-active: bidirectional replication, handle conflicts with last-writer-wins or CRDTs. Consider network latency: use async replication. Failover: promote secondary to primary.

8. **Q:** Design an event-driven system for real-time fraud detection.
    **A:** Transaction events published to Kafka. Stream processing (Flink/KSQL) on event stream: windowed aggregation, pattern matching (e.g., multiple failed attempts in 5 min). Scoring model as a service called via Kafka RPC. Alert event published for high-risk transactions. Low-latency: <100ms end-to-end.

9. **Q:** What are the challenges of idempotent event processing at scale?
    **A:** Deduplication store becomes bottleneck. Solutions: probabilistic dedup (Bloom filters), time-bounded dedup windows, sharded dedup store (Redis Cluster). For Kafka, use transactional API. Balance: wider dedup window = more memory, narrower = risk of duplicates.

10. **Q:** How do you test event-driven systems?
    **A:** Unit test event handlers in isolation. Integration test with embedded Kafka (Testcontainers). Contract tests for event schemas. Chaos testing: broker failure, consumer lag, network partitions. Trace-based testing: assert expected events produced for given input.

### System Design

11. **Q:** Design a real-time notification system for a social media platform (1B events/day).
    **A:** Kafka for event ingestion (likes, comments, follows). Stream processor groups events by user (tumbling windows, 1 min). Aggregated notification event published to notification topic. Delivery via: push (WebSocket/MQTT for mobile), email (batched, hourly digest), in-app (polling). Dedup: ignore if user already notified.

12. **Q:** Design an event-driven order processing system with 99.99% reliability.
    **A:** Order Service writes to outbox table (DB transaction). Outbox publisher -> Kafka (order-events). Multiple consumer groups: Inventory, Payment, Shipping, Notification. Saga orchestrator coordinates multi-step flow. DLQ for failed events with automatic retry (3 attempts, exponential backoff). Monitoring: consumer lag, error rate, throughput dashboards.

13. **Q:** Design a global event streaming platform for IoT telemetry (10M devices).
    **A:** MQTT broker at edge for device connectivity. Kafka as central event backbone. Partition by device ID for ordering. Kafka Connect for S3 archival (Parquet). Stream processing (Kafka Streams) for anomaly detection. Time-series DB (Cassandra) for device state. Downstream: Alerts, Analytics, ML training.

14. **Q:** Design an event-driven financial trading system.
    **A:** Ultra-low latency path: Aeron (UDP-based messaging) for market data. Kafka for order events (durable). In-memory state for order book. FPGA-based matching engine. Event sourcing for full audit trail. CQRS: write model (matching engine), read model (positions, P&L). Circuit breakers for risk checks.

15. **Q:** Design a CDC (Change Data Capture) pipeline for real-time data sync across services.
    **A:** Debezium captures DB changes (PostgreSQL WAL, MySQL binlog). CDC events published to Kafka. Multiple consumers: downstream service cache invalidation, search index (Elasticsearch), analytics (ClickHouse), data warehouse (Snowflake via Kafka Connect). Exactly-once via Kafka Connect + idempotent sinks.

16. **Q:** Design an event-driven microservices platform for a hotel booking system.
    **A:** Services: Inventory, Reservation, Payment, Notification, Pricing. Events: `RoomAvailabilityChanged`, `ReservationCreated`, `PaymentCompleted`, `BookingConfirmed`. Saga orchestrator: reserve room -> process payment -> confirm booking -> send confirmation. Compensating actions: cancel reservation, release room, process refund.

17. **Q:** Design a real-time recommendation engine using event-driven architecture.
    **A:** User interaction events (view, click, purchase) -> Kafka. Stream processor computes user profile (sliding window of recent interactions). Feature store (Redis) for real-time features. ML model served via gRPC. Recommendations published to user-specific Kafka topic. Client subscribes for real-time updates.

18. **Q:** Design an event-driven system for a multiplayer online game.
    **A:** Game state events (player move, shoot, collect) -> Kafka or Redis Streams (low latency). Game server partition by match ID. In-memory state for active matches. Events persist to Cassandra for replay. Player matchmaking: event-driven, published when player enters queue. Push events to players via WebSocket.

19. **Q:** Design an event-driven CI/CD pipeline.
    **A:** Git push event -> Webhook -> EventBridge -> CI pipeline event. Pipeline orchestrator (Argo Workflows) listens for events. Build event -> Test services -> Artifact published event -> Deploy to staging -> E2E test event -> Deploy to production. Each step emits event, next step subscribes. Automatic rollback on failure event.

20. **Q:** Design an event-driven system for a food delivery platform.
    **A:** Order placed -> `OrderPlaced` event. Restaurant service: accepts/rejects order -> `OrderAccepted`/`OrderRejected`. Driver matching: `DriverAssigned`. Food preparation: `OrderReady`. Delivery: `OutForDelivery`, `Delivered`. All events -> Kafka. CQRS for order tracking (write: state machine, read: materialized view). Real-time tracking via WebSocket + Kafka Streams.

## 12. Expert-Level Interview Questions (10: architect-level)

1. **Q:** Design a globally distributed event store with strong consistency and low latency.
    **A:** FoundationDB or CockroachDB as event store (distributed SQL with ACID). Events written with global timestamp (TrueTime/Clock-SI). Kafka for streaming to consumers. Raft consensus for strong consistency. CQRS: event store = write model, materialized views = read model. Conflict-free: deterministic event processing ensures same result regardless of order.

2. **Q:** How would you design an event-driven platform that supports 1M+ event types with schema governance?
    **A:** Domain-driven event naming: `domain.context.event.v1`. Central schema registry with automated compatibility checks. Event taxonomy documented and searchable. Event design reviews for new types. Automated code generation from schemas. Deprecation: mark events as deprecated, consumers must migrate within 6 months. Delete unused event types after migration.

3. **Q:** Design a dead letter queue system that supports automatic recovery and root cause analysis.
    **A:** DLQ with structured metadata: original topic, consumer group, error details, retry count, message payload. Automated retry: exponential backoff (10s, 30s, 90s, 5min, 15min, 1h, 4h, 12h, 24h). After max retries: move to permanent dead storage (S3). Alert on DLQ depth. Analyst dashboard for batch re-processing. ML-based root cause suggestion.

4. **Q:** How do you implement end-to-end exactly-once processing in a multi-stage event pipeline?
    **A:** Kafka transactional API on producer side. Consumer: idempotent processing with transactional offset commits (consume-transform-produce loop atomically in transaction). Use `isolation.level=read_committed`. Sink services: idempotent with dedup keys from upstream event ID. Monitoring: tracking gaps in sequence numbers per partition.

5. **Q:** Design an event-driven system that can handle 100x traffic spikes without degradation.
    **A:** Kafka as buffer: elastic storage that absorbs spikes. Auto-scaling consumers based on consumer lag metric. Backpressure: Kafka retention absorbs backpressure naturally. Circuit breakers on downstream services to prevent overload. Priority levels: high-value events processed first (payment), low-value delayed (analytics). Spike capacity planning: 100x of normal throughput reserved.

6. **Q:** How would you design an event-driven architecture for a financial system requiring full auditability and regulatory compliance?
    **A:** Event sourcing for all state changes: every mutation is an event. Immutable event store (append-only). Events signed with private key for non-repudiation. Audit log: all events stored indefinitely with integrity checks (Merkle tree on event chain). GDPR: separate PII events encrypted at rest, decryption key accessible only by authorized services.

7. **Q:** Design a multi-tenant event-driven platform with tenant isolation and fairness.
    **A:** Physical isolation: dedicated Kafka cluster for enterprise tenants. Logical isolation: shared cluster, topics per tenant, quota-based (rate limits per producer/consumer, storage quotas, partition limits). Fairness: weighted fair queuing across tenants, no noisy-neighbor. Tenant throttling: if one tenant consumes 80% of resources, throttle to protect others.

8. **Q:** How do you ensure causal consistency in a distributed event-driven system?
    **A:** Use vector clocks or hybrid logical clocks (HLC) attached to each event. Events carry causal dependencies. Consumer tracks observed vector clock. If event's dependency not satisfied, delay processing. Kafka partition ordering provides natural causality within key. Cross-partition causality requires application-level clock comparison.

9. **Q:** Design an event-driven platform that supports both real-time and batch processing from the same event stream.
    **A:** Unified log (Kafka) for all events. Real-time path: stream processing (Kafka Streams, Flink) with millisecond latency. Batch path: same events persisted to S3/Parquet via Kafka Connect S3 Sink. Batch processing (Spark, Trino) on S3 data. Lambda Architecture: real-time layer + batch layer with merging. Kappa Architecture: single streaming pipeline with replay for batch.

10. **Q:** How would you design a governance system for event contracts across 200+ microservices?
    **A:** Schema Registry as source of truth. Contract-first development: API designer tool for events. Automated linting: naming conventions, field types, metadata requirements. CICD pipeline: schema change triggers compatibility check, creates new version, publishes to registry. Service ownership tracking: each event type has owning team+service. Deprecation dashboard: shows consumers of deprecated events.

## 13. Debugging & Troubleshooting

### Common EDA Issues

**Issue: Messages lost in production**
- Check producer acks: `acks=all` ensures leader + replicas confirm.
- Check consumer auto-commit: disable auto-commit, commit manually after processing.
- Check retention: topic retention may be too short (default 7 days).
- Check consumer group offsets: not committed properly.

**Issue: High consumer lag**
- Check consumer processing speed (too slow).
- Increase partitions + consumers.
- Check for blocking operations in consumer.
- Check if consumer crashed (rebalancing).

**Issue: Duplicate event processing**
- Verify dedup store is working.
- Check if consumer rebalancing causes re-processing.
- Check idempotency: is processing idempotent?

**Issue: Schema incompatibility**
- Check schema version in event: does consumer support it?
- Check Schema Registry: is the schema registered?
- Check consumer code: does it handle unknown fields?

### Debugging Commands
```bash
# Kafka consumer group status
kafka-consumer-groups --bootstrap-server kafka:9092 \
  --group order-service --describe

# Kafka topics and partitions
kafka-topics --bootstrap-server kafka:9092 --list
kafka-topics --bootstrap-server kafka:9092 \
  --topic order.events --describe

# Read messages from beginning
kafka-console-consumer --bootstrap-server kafka:9092 \
  --topic order.events --from-beginning --max-messages 10

# Check consumer offsets
kafka-consumer-groups --bootstrap-server kafka:9092 \
  --group order-service --describe --verbose

# RabbitMQ queues and bindings
rabbitmqctl list_queues name messages consumers
rabbitmqctl list_bindings
```

### Monitoring Tools
```yaml
# Prometheus Kafka metrics
kafka_consumer_lag{consumer_group="order-service"}
kafka_topic_partition_current_offset{topic="order.events"}
kafka_consumer_group_lag{group="order-service"}

# Alerts
- alert: HighConsumerLag
  expr: kafka_consumer_lag > 10000
  for: 5m
```

## 14. Comparison Section

### Event-Driven vs Request-Response

| Aspect | Event-Driven | Request-Response |
|--------|-------------|------------------|
| Coupling | Loose (via broker) | Tight (direct call) |
| Scalability | High (async, buffered) | Limited by synchronous |
| Latency | Higher (broker) | Lower (direct) |
| Reliability | Higher (persistent) | Lower (failure cascade) |
| Complexity | Higher | Lower |
| Testing | Harder | Easier |
| Visibility | Harder (async flow) | Easier (sync trace) |

### Kafka vs RabbitMQ vs Pulsar

| Feature | Kafka | RabbitMQ | Pulsar |
|---------|-------|----------|--------|
| Model | Distributed log | Message broker | Log + broker |
| Throughput | 1M+ msg/s | 100K msg/s | 1M+ msg/s |
| Latency | ~10ms | ~1ms | ~5ms |
| Ordering | Per partition | Exchange type dependent | Per partition |
| Retention | Configurable time/size | Ack-based deletion | Time/size + backlog |
| Persistence | Durable by default | Optional (lazy queues) | Durable by default |
| Routing | Topic-based | Exchange (direct, topic, fanout, headers) | Topic-based |
| Management | Kafka Connect, KSQL, Schema Registry | Management UI, plugins | Pulsar Functions, IO connectors |

### Event Sourcing vs Event Notification

| Aspect | Event Sourcing | Event Notification |
|--------|---------------|-------------------|
| State | Derived from events | Current state in DB |
| Events | Source of truth | Just notifications |
| Audit | Built-in | Requires additional |
| Complexity | High | Low |
| Storage | Append-only event store | Events + DB state |
| Query | Must replay events | Direct DB query |

## 15. Revision Notes

### Quick Recap
- **Event**: Immutable fact about something that happened.
- **Producer/Consumer**: Services that produce/consume events via broker.
- **Broker**: Kafka, RabbitMQ, Pulsar, AWS EventBridge.
- **Guarantees**: At-most-once, at-least-once, exactly-once.
- **Ordering**: Per key/partition (Kafka), per queue (RabbitMQ).
- **Idempotency**: Required for safe retries and at-least-once delivery.
- **Outbox Pattern**: Event persistence + publication within same DB transaction.
- **DLQ**: Failed events routed for later inspection.
- **Schema Registry**: Manages event schemas with compatibility checks.
- **Saga**: Multi-step process with compensating actions.

### Event Design Best Practices
- Events are past tense: `OrderCreated`, `PaymentReceived`.
- Include enough data for autonomous processing.
- Use a unique event ID for deduplication.
- Include timestamp and originating service metadata.
- Version events explicitly.
- Keep schemas backward and forward compatible.

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                 EVENT-DRIVEN ARCHITECTURE CHEAT SHEET              |
+-------------------------------------------------------------------+
| COMPONENT         | ROLE                                           |
+-------------------+------------------------------------------------+
| Event Producer    | Detects state change, publishes event           |
| Event Broker      | Routes events from producers to consumers      |
| Event Consumer    | Subscribes to and processes events              |
| Dead Letter Queue | Stores failed events for analysis              |
| Schema Registry   | Manages event schemas and versions             |
| Stream Processor  | Transforms/filters/aggregates event streams    |
+-------------------+------------------------------------------------+
| DELIVERY GUARANTEES               | ORDERING GUARANTEES               |
+-----------------------------------+-----------------------------------+
| At-most-once: fire and forget     | Kafka: ordered per partition key  |
| At-least-once: retry on failure   | RabbitMQ: ordered per queue        |
| Exactly-once: dedup + idempotent  | Global order: single partition     |
+-----------------------------------+-----------------------------------+
| KAFKA COMPONENTS                                                    |
+-------------------------------------------------------------------+
| Topic              | A named channel for related events             |
| Partition          | Ordered sequence of events within a topic     |
| Producer           | Publishes events to topics                    |
| Consumer           | Subscribes to and processes events            |
| Consumer Group     | Set of consumers sharing consumption load     |
| Offset             | Position of consumer within a partition       |
| Broker             | A Kafka server in the cluster                 |
+-------------------------------------------------------------------+
| RELIABILITY PATTERNS                                               |
+-------------------------------------------------------------------+
| Outbox Pattern    | Write event to DB in same transaction         |
|                    | as state change, then publish to broker       |
| Idempotent        | Dedup by event ID, process only once          |
| Consumer          | Use SETNX/unique constraint                    |
| Dead Letter Queue | Route failed events after retries exhausted   |
| Retry Topic       | Re-process events with backoff                |
| Saga              | Local txns + compensating actions on failure  |
+-------------------------------------------------------------------+
| SPRING BOOT / KAFKA                                                 |
+-------------------------------------------------------------------+
| @KafkaListener(topics, groupId)   | Listen to Kafka topics           |
| KafkaTemplate.send(topic, key,    | Send event to topic              |
|   value)                          |                                  |
| @EnableKafka                      | Enable Kafka annotations         |
| @EnableKafkaStreams               | Enable Kafka Streams             |
| ProducerFactory /                 | Configure serialization, acks,   |
| ConsumerFactory                   | group config                     |
+-------------------------------------------------------------------+
```
