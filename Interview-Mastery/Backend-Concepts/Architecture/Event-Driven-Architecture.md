# Event-Driven Architecture

---

## Overview

- **Definition:** A software design pattern where services communicate through the production, detection, consumption, and reaction to events via an intermediary event broker rather than direct request-response calls.
- **Why It Exists:** EDA decouples event producers from consumers, enabling asynchronous, scalable, and loosely coupled systems. It handles variable workloads through buffering, supports independent scaling of producers and consumers, and enables real-time processing across distributed services.
- **Key Concepts:** **Event** (immutable fact about something that happened), **Event Producer** (publishes events), **Event Consumer** (subscribes and processes), **Event Broker** (Kafka, RabbitMQ, Pulsar — routes events), **Topic** (named channel for events), **Partition** (ordered sequence within a topic), **Consumer Group** (set of consumers sharing load), **Delivery Guarantees** (at-most-once, at-least-once, exactly-once), **Dead Letter Queue** (failed events routed for analysis), **Schema Registry** (manages event schemas with compatibility checks)
- **Event-Driven vs Request-Driven** — Request-driven (REST/gRPC) requires both parties to be online and responds immediately. Event-driven allows temporal decoupling — the producer publishes and forgets; the consumer processes when ready. Request-driven is simpler for query operations; event-driven excels for workflows, broadcasting, and cross-system integration.
- **Choosing Between Kafka and RabbitMQ** — Kafka: high throughput (100K+ msgs/sec), persistent storage, replay capability, ordering within partitions, ideal for event streaming and data pipelines. RabbitMQ: flexible routing (exchanges, bindings), lower latency (sub-ms), work queues, ideal for task distribution and complex routing patterns. Kafka is a log; RabbitMQ is a message broker.

---

## Core Concepts

- **Event Types:** Event Notification (lightweight, minimal data — consumers fetch more if needed), Event-Carried State Transfer (event contains all data for processing), Event Sourcing (every state change stored as an event, current state derived by replaying events).
- **Delivery Semantics:** At-most-once (fire and forget, may lose events), At-least-once (event delivered at least once, may duplicate), Exactly-once (requires idempotent consumers + deduplication + transactional producers).
- **Message Ordering:** Kafka guarantees ordering within a partition using the same partition key. RabbitMQ has no ordering guarantee by default. Global ordering requires a single partition (limits parallelism).
- **Outbox Pattern:** Events are written to a database table within the same transaction as the state change, then reliably published to the message broker by a separate process. Prevents the dual-write problem where the DB is updated but the event is never published.
- **Idempotent Consumers:** Store processed event IDs in a deduplication store (Redis with TTL or DB unique constraint) to safely handle duplicate deliveries under at-least-once semantics.
- **Partition Key Design** — The partition key determines ordering and parallelism. Use business entity ID (order ID, user ID) as the partition key to ensure all events for the same entity are processed in order by the same consumer. Avoid using random keys — they distribute evenly but break ordering guarantees.
- **Compacted Topics** — Kafka supports log compaction, where only the latest message for each key is retained. Useful for rebuilding state from events (a "table" semantics): each key represents an entity, and the latest value is its current state. Older versions are automatically removed. Compacted topics serve as the source for KTables in Kafka Streams.

```java
// Idempotent event consumer with dedup
@Component
public class IdempotentEventConsumer {
    @KafkaListener(topics = "payment.events", groupId = "order-service")
    public void handlePaymentCompleted(PaymentCompletedEvent event) {
        String dedupKey = "dedup:payment:" + event.getEventId();
        Boolean alreadyProcessed = redisTemplate.opsForValue().setIfAbsent(dedupKey, "processed", Duration.ofMinutes(5));
        if (Boolean.FALSE.equals(alreadyProcessed)) { log.warn("Duplicate event: {}", event.getEventId()); return; }
        orderRepository.findById(event.getOrderId()).ifPresent(order -> {
            order.setStatus(OrderStatus.PAID);
            orderRepository.save(order);
        });
    }
}

// Outbox publisher — polls and publishes
@Component
public class OutboxPublisher {
    @Scheduled(fixedDelay = 1000)
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> pending = outboxRepository.findByStatusOrderByCreatedAt(OutboxStatus.PENDING);
        for (OutboxEvent event : pending) {
            kafkaTemplate.send(event.getEventType(), event.getAggregateId(), event.getPayload()).get(5, TimeUnit.SECONDS);
            event.setStatus(OutboxStatus.PUBLISHED);
            outboxRepository.save(event);
        }
    }
}
```

---

## Common Mistakes

- **Over-coupling Events to Consumer Expectations** — events should be "facts that happened" not "commands for specific consumers". Design events as business facts, not RPC calls. This *looks correct* because: when you know exactly what the consumer needs to do, it's natural to shape the event around that action — the coupling only becomes visible when a second consumer needs the same event for a different purpose.
- **Lack of Schema Management** — without a schema registry, producer/consumer contracts break silently. Use Avro/Protobuf with Schema Registry for compatibility enforcement. This *looks correct* because: JSON is flexible and schemas seem like overhead for a "simple" event — the silent breakage only surfaces when a producer adds a field that a consumer's deserializer can't handle.
- **No Error Handling in Consumers** — unhandled exceptions stop the consumer. Always catch exceptions, log, send to DLQ, and commit the offset. This *looks correct* because: in development, events always process successfully, so error handling seems like boilerplate — the consumer crashing on a malformed event only becomes a problem in production when a single bad event blocks all subsequent events.
- **Assuming Message Ordering Across Partitions** — Kafka only guarantees ordering within a partition. Multi-partition ordering requires application-level coordination. This *looks correct* because: messages are in order when you look at a single partition in isolation, and the partition key seems to control ordering — the assumption breaks when related events use different partition keys and arrive out of order.
- **Synchronous Blocking in Async Event Handlers** — blocking calls in consumer threads reduce throughput. Use async processing or reactive frameworks. This *looks correct* because: the code compiles and runs correctly, and the consumer framework abstracts thread management — the performance impact of blocking calls is hidden until consumer lag grows under load.
- **Infinite Retry Without DLQ** — A consumer that retries forever on a bad event blocks subsequent events. Always use a dead letter queue after a finite number of retries (typically 3-5). The failed event goes to DLQ, and the consumer commits the offset and continues processing. This *looks correct* because: retrying seems like the safest approach — the event will eventually succeed — but a permanently bad event blocks processing of all later events in the same partition indefinitely.
- **Over-partitioning Topics** — More partitions increase parallelism but also increase overhead (more connections, more rebalancing). Rule of thumb: partitions = max consumers you'll ever need × 1.5. Avoid partitions exceeding 1000 without performance testing. This *looks correct* because: more partitions means more parallelism, so "more is better" seems logical — the increased rebalancing time and connection overhead only becomes apparent when a consumer restart takes 5 minutes instead of 5 seconds.
- **Missing Telemetry on Event Pipeline** — Without monitoring consumer lag, event processing failures, and DLQ depth, issues go undetected until users complain. Monitor: consumer lag per partition, events processed/sec, error rate, DLQ size, processing latency per event. This *looks correct* because: the event pipeline processes silently when everything is working, and metrics dashboards seem like overhead you can add later — you only miss them when a consumer silently stops processing and nobody notices for hours.

---

## Key Design Considerations

- **Schema Evolution Strategy** — backward compatible (new schema reads old data — add optional fields only), forward compatible (old schema reads new data — never remove fields), full compatible (both directions). Use Avro/Protobuf with Schema Registry.
- **Event Versioning** — schema registry with version field, separate topics per version (`orders.v1`, `orders.v2`), or event envelope with version metadata.
- **Strategic Event Design** — events are past-tense immutable facts (`OrderCreated`, `PaymentReceived`); commands are future-tense requests (`ReserveInventory`). Include enough data for autonomous processing (event-carried state transfer). Use unique event IDs for deduplication.
- **Dead Letter Queue Strategy** — structured DLQ with original topic, error details, retry count. Automated retry with exponential backoff. After max retries, move to permanent dead storage and alert operations.
- **Throughput Optimization** — increase partitions for higher throughput, batch consumption reduces overhead, compression (Snappy, Zstd) reduces network/storage. Tune fetch size, linger time, and batch size for throughput vs latency trade-offs.
- **Async Communication Pitfalls** — Eventual consistency means stale data. Out-of-order events require idempotent handlers. Schema evolution needs compatibility management. Debugging async flows requires distributed tracing. Build these into your architecture from day one — retrofitting is much harder.
- **Event Sourcing vs Event-Driven Architecture** — Event Sourcing stores state as an append-only event log; current state is derived by replaying events. Event-Driven Architecture uses events for communication but stores current state. They can be combined (CQRS + Event Sourcing) but are independent patterns. Event Sourcing provides audit trails and temporal queries; EDA provides decoupling and scalability.

---

## Real-World Scenarios

### Scenario 1: Order Processing Pipeline with Event-Carried State Transfer
**Context:** An e-commerce platform has a monolithic order service that synchronously calls Inventory, Payment, Shipping, and Notification services. A single order placement takes 5 seconds and the entire pipeline fails if any downstream service is down.

**Resolution:** Switch to event-driven architecture. When a customer places an order, the Order Service publishes an `OrderPlaced` event containing all necessary data (order ID, customer ID, items with quantities, total amount, shipping address). The Inventory Service consumes this event and reserves stock. The Payment Service charges the customer. The Shipping Service creates a shipment. Each service processes independently and publishes its own event on completion. The entire flow is asynchronous — the customer gets an immediate "Order Received" response while processing happens in the background.

```java
// Producer publishes event with full payload (Event-Carried State Transfer)
@Service
public class OrderService {
    public void placeOrder(PlaceOrderCommand cmd) {
        Order order = new Order(cmd.customerId(), cmd.items());
        orderRepository.save(order);
        kafkaTemplate.send("order.events", order.getId(), new OrderPlacedEvent(
            order.getId(), order.getCustomerId(), order.getTotal(),
            order.getItems().stream().map(Item::toDto).toList(),
            cmd.shippingAddress()));
    }
}

// Consumer processes independently
@Component
public class InventoryConsumer {
    @KafkaListener(topics = "order.events", groupId = "inventory-service")
    public void handle(OrderPlacedEvent event) {
        event.items().forEach(item -> inventoryService.reserve(item.productId(), item.quantity(), event.orderId()));
        kafkaTemplate.send("inventory.events", event.orderId(), new InventoryReservedEvent(event.orderId()));
    }
}
```

### Scenario 2: Real-Time Fraud Detection
**Context:** A payment company needs to detect fraud within 100ms of a transaction event. Transactions flow through multiple analysis stages: velocity check → geolocation anomaly → device fingerprint → ML model scoring.

**Resolution:** Use Kafka Streams for real-time event processing. Each analysis stage is a KStream transformation. Transactions flow through the pipeline with sub-100ms latency. Suspicious transactions are routed to an alert topic for manual review.

```java
@Bean
public KStream<String, Transaction> fraudDetection(StreamsBuilder builder) {
    KStream<String, Transaction> source = builder
        .stream("transactions", Consumed.with(Serdes.String(), new JsonSerde<>(Transaction.class)));

    // Join with customer profile for risk scoring
    KTable<String, CustomerProfile> profiles = builder
        .table("customer-profiles", Materialized.as("profiles"));

    source.join(profiles, (txn, profile) -> {
        double score = profile.baseRiskScore();
        if (txn.amount() > profile.dailyAverage() * 5) score += 0.3;
        if (profile.countRecentTxns(txn.userId()) > 10) score += 0.2;
        // ... more rules
        return new ScoredTransaction(txn, score);
    }).filter((key, scored) -> scored.score() > 0.7)
      .to("suspicious-transactions");
    return source;
}
```

### Scenario 3: Outbox Pattern for Reliable Event Publication
**Context:** A payment service updates a transaction status to "COMPLETED" in its database and must publish a `PaymentCompleted` event. The dual-write problem means if the event publish fails, the system is inconsistent — the database shows paid but downstream services never know.

**Resolution:** Implement the transactional outbox pattern. Within the same database transaction, both the payment status update and the outbox event record are written atomically. A scheduled `OutboxPublisher` polls the outbox table and publishes events reliably with at-least-once delivery.

---

## Scenario-Based Questions

1. **Q: You are building a notification service that must alert users via email, SMS, and push when an order ships. The notification volume spikes during sales (10x normal). How do you design this?**
   - A: Use an event-driven architecture. The Shipping Service publishes an `OrderShipped` event to a topic. Three consumer groups (email, SMS, push) subscribe independently. A message broker (Kafka/RabbitMQ) buffers the spike — consumers process at their own pace. The email service can scale to 10 instances during sales while the SMS service stays at 2. Each channel has its own DLQ for failures. The system decouples notification channels from shipping, allowing each to scale independently and fail independently.

> **Interview follow-up:** The email service is 10 minutes behind processing events, and a user calls support asking why their shipping notification hasn't arrived — how do you communicate to the user that the event is still in the queue without revealing internal processing details?

2. **Q: Your e-commerce system publishes an `OrderPlaced` event. The Inventory service consumes it and publishes `InventoryUpdated`. The Payment service consumes that. This cascading pattern gets hard to trace. How do you improve observability?**
   - A: Implement distributed tracing with a trace ID propagated in event headers. Each event carries `traceparent` (W3C TraceContext). The trace flows through the event chain: OrderService → Kafka → InventoryService → Kafka → PaymentService. Use OpenTelemetry auto-instrumentation for Kafka producers and consumers. This gives you an end-to-end view of event processing latency, identifying bottlenecks (e.g., InventoryService processing time spiking).

> **Interview follow-up:** Distributed tracing across async event flows introduces temporal challenges — a trace might span minutes or hours if a consumer lags. How do you distinguish between a slow consumer and a broken trace when the span duration exceeds your alerting threshold?

3. **Q: Your team is debating between event-carried state transfer (event with full data) vs event notification (event with minimal data, consumers fetch the rest). Which do you choose and why?**
   - A: Prefer event-carried state transfer for autonomy — the event contains everything the consumer needs to process it. This eliminates synchronous callbacks (the consumer doesn't need to call back to the producer for details). Trade-off: larger event size and potential data duplication. Use event notification only when: (a) the data is very large, (b) the data changes frequently between event publication and consumption, or (c) privacy concerns require limiting data in events.

> **Interview follow-up:** You use event-carried state transfer, but now the shipping service stores a `customerEmail` in the event that changes independently — when the customer updates their email in the profile service, the shipping service still has the old email. How do you keep duplicated data in sync without adding synchronous calls?

4. **Q: During a Black Friday sale, your event consumers can't keep up with the producer, and consumer lag climbs to 30 minutes. How do you handle this without losing events?**
   - A: Kafka retains events based on retention policy (typically 7 days), so events are safe. To address the lag: increase partitions (and thus consumers), optimize consumer processing (batch commits, async processing, increase `max.poll.records`), or add more consumer group instances. For sustained spikes, pre-scale consumers before the event. Monitor consumer lag as a critical metric and alert when it exceeds acceptable thresholds.

> **Interview follow-up:** Consumer lag spikes during a flash sale, but scaling consumers doesn't reduce it — you've hit a partition bottleneck where one partition's messages are slow to process. How do you identify the hot partition and rebalance without losing message order?

5. **Q: You're using at-least-once delivery and a consumer processes an event but crashes before committing the offset. The event is reprocessed, causing a duplicate order. How do you prevent this?**
   - A: Idempotent event processing. Store the event ID in a deduplication store before processing. Use Redis `SETNX` with TTL equal to the retention period, or a database unique constraint on `event_id`. If the same event arrives again, the dedup check skips it. Additionally, make the business operation idempotent — an order creation should use `INSERT ... ON CONFLICT DO NOTHING` or equivalent upsert.

> **Interview follow-up:** You use Redis `SETNX` for dedup, but Redis crashes and loses all TTL keys — now the same events are reprocessed as if they're new. How do you protect against dedup store failures?

6. **Q: You need to replay historical events to rebuild a read model that was corrupted. The event store has 500M events. How do you do this efficiently?**
   - A: Create a new projection that processes events from the beginning of the topic. Set the consumer's `auto.offset.reset=earliest`. To speed up replay, increase partitions temporarily. Run the new projection in parallel with the old one. When the new projection catches up to real-time, switch traffic. For very large replays, use Kafka's log compaction to skip superseded events (e.g., for entity state, only the latest event per key matters).

7. **Q: A microservice publishes events to Kafka, but during a deployment, events are published for state that doesn't yet exist from the consumer's perspective (schema mismatch). How do you handle schema evolution?**
   - A: Use a schema registry (Confluent Schema Registry or Apicurio) with compatibility checks. Configure BACKWARD compatibility — new schemas can read data written by old schemas (only add optional fields). The producer registers the new schema; the registry validates it won't break existing consumers. The consumer uses its registered schema to deserialize, ignoring unknown fields. Never remove fields — mark them as deprecated.

> **Interview follow-up:** You mark a field as deprecated, but downstream consumers ignore the deprecation notice and keep relying on it for two more years. How do you enforce a deprecation timeline when you can't control all consumers?

8. **Q: Your system emits 10K events/second. A buggy consumer is stuck in a crash-restart loop, constantly rebalancing and causing rebalance storms. How do you fix this?**
   - A: Implement a dead letter queue. The consumer catches the exception, sends the problematic event to a DLQ topic, and commits the offset (skipping the bad event). This prevents the consumer from crashing. Separately, implement a health check that detects stuck consumers. Use a timeout: if a consumer doesn't commit offsets for N minutes, stop its membership to prevent rebalance storms on the remaining healthy consumers.

9. **Q: You're building a distributed saga using events. How do you ensure that a compensating action executes exactly once when a step fails?**
   - A: Each event in the saga carries a `sagaId` and `stepId`. The compensating action is triggered by a specific failure event. Use idempotent compensation handlers (e.g., a refund operation checks if refund already issued). The orchestrator (or event chain) publishes the compensation event with at-least-once delivery. The consumer checks its dedup store before executing the compensation.

10. **Q: Your events contain Personally Identifiable Information (PII). After 90 days, you need to remove or anonymize the PII from historical events in Kafka. How do you accomplish this?**
    - A: Options: (a) Compacted topics with key-based retention — if you update the record with an anonymized version using the same key, compaction removes the old record. (b) Log compaction with a TTL-based delete — configure `cleanup.policy=compact,delete` with `retention.ms=90days`. (c) For immutable compliance, use a schema with PII in separate, encrypted fields that can be rotated. (d) Event-carried state transfer stores only the PII that's needed; don't include unnecessary PII.

---

## Interview Questions

1. **What is the difference between an event and a command?**
   - A: An event is an immutable fact about something that already happened (past tense: `OrderPlaced`, `PaymentReceived`). A command is a request for action that may be rejected (imperative: `PlaceOrder`, `ChargePayment`).

2. **What is the difference between Kafka and RabbitMQ?**
   - A: Kafka is a distributed commit log optimized for high-throughput, persistent event streaming, replay, and long-term retention. RabbitMQ is a message broker optimized for flexible routing (exchanges/bindings), low latency, and complex messaging patterns (RPC, pub-sub, work queues).

3. **What is a consumer group in Kafka?**
   - A: A set of consumers that collaboratively consume a topic. Each partition is assigned to exactly one consumer in the group. Total throughput = partitions × per-consumer throughput. On consumer failure, partitions rebalance to remaining consumers.

4. **What is at-least-once delivery and how do you handle duplicates?**
   - A: Messages are never lost but may be duplicated. Handle via idempotent consumers: store processed event IDs in a dedup store (Redis with TTL or DB unique constraint) and check before processing.

5. **What is the outbox pattern?**
   - A: A solution to the dual-write problem. Business data and events are written atomically in the same database transaction. A separate process (outbox publisher) reads the outbox table and publishes events to the message broker, ensuring reliable publication.

6. **How do you handle event ordering in Kafka?**
   - A: Use the same partition key for related events (e.g., order ID). Kafka guarantees order within a partition. Global ordering is possible with a single partition (limits parallelism) or with application-level sequence numbers.

7. **What is the dead letter queue pattern?**
   - A: A queue where events that failed processing after all retry attempts are routed for later analysis, manual reprocessing, or automated replay after the bug is fixed.

8. **What is event-carried state transfer?**
   - A: Events contain all the data a consumer needs for processing, so consumers don't need to call back to the producer for additional information. Increases autonomy but duplicates data across services.

9. **How does event-driven architecture improve scalability?**
   - A: Decouples producers and consumers (they scale independently), provides buffering during traffic spikes (message backlogs are handled), enables parallel processing via consumer groups, and supports polyglot persistence (each service chooses its storage).

10. **What is the difference between event notification, event-carried state transfer, and event sourcing?**
    - A: Event notification: minimal event payload, consumers fetch details if needed. Event-carried state transfer: full payload in the event, consumers are autonomous. Event sourcing: events are the primary source of truth — current state is derived by replaying events.

---

## Developer Recommendations

- **Design events as business facts, not procedural commands** — Events should be named in past tense (`OrderPlaced`, `PaymentReceived`) and represent something the business cares about. Avoid creating events like `NotifyCustomer` (a command) or `UpdateInventory` (procedural). When the business says "when an order is placed..." that's `OrderPlaced`. This keeps the event model aligned with business language and makes events reusable across multiple consumers. A fintech startup designed `SendFraudAlert` as an event, coupling billing to a specific action; when they needed to reuse the same data for regulatory reporting, the event was unusable because it captured the procedural intent rather than the business fact.

- **Use the outbox pattern to solve the dual-write problem** — A service that writes to its database and publishes an event in the same operation will inevitably face inconsistency when the publish fails. The outbox pattern writes both to the same database transaction. A separate poller publishes events reliably. This prevents the most common source of data inconsistency in event-driven systems.

- **Make every consumer idempotent** — At-least-once delivery guarantees that duplicates will happen. Every event consumer must handle the same event twice without side effects. Use dedup stores (Redis with TTL), upsert operations instead of inserts, and idempotency keys in external API calls. Test idempotency by replaying events in staging. A payment processing startup skipped dedup testing because the broker guaranteed at-most-once delivery in dev — in production, a Kafka rebalance caused 47 duplicate payment charges before they added idempotency checks.

- **Design event schemas with forward and backward compatibility** — Events live longer than services — an event written today may be consumed by services deployed 2 years from now. Use Avro or Protobuf with a schema registry. Set compatibility rules: BACKWARD (new consumers read old events), FORWARD (old consumers read new events), or FULL (both). Add fields as optional with defaults. Never remove fields.

- **Implement structured error handling: DLQ + retry + alerting** — Every event consumer should have three-stage error handling: retry with exponential backoff for transient errors, DLQ for permanent failures, and alerting when the DLQ grows. Without this, event processing silently stops and data inconsistency grows until someone notices the symptom, not the cause. A food delivery platform had no DLQ: a single malformed order event from a third-party integration caused the payment consumer to crash-loop, blocking all payment events for 4 hours during dinner rush — 3,000 orders were stuck in "processing" state with no alert raised.

- **Monitor consumer lag as a critical business metric** — Consumer lag (how far behind real-time a consumer is) directly impacts user experience. A payment consumer lagging by 5 minutes means users wait 5 minutes for their orders to be confirmed. Alert when lag exceeds acceptable thresholds. Use different thresholds for different consumers: user-facing (30s), analytical (5min), batch (1hr). A logistics company monitored average consumer lag but didn't track per-partition lag — one partition's consumer was down for 6 hours while the average lag looked healthy at 2 minutes, silently losing time-sensitive delivery status updates.
- **Use the transactional outbox pattern for every service that publishes events** — Dual-write (database + event) is a distributed transaction that always fails eventually without the outbox pattern. Write both the entity state change and the event to the same database transaction. A scheduled poller or CDC (Change Data Capture) process publishes the event reliably. This prevents the most common data loss scenario in event-driven systems.
- **Design events for backward and forward compatibility from the start** — Events live longer than services. An event written today may be consumed by services deployed 2 years from now. Use Avro or Protobuf with a schema registry. Set compatibility to BACKWARD or FULL. Add fields as optional with defaults. Never remove fields — mark them as deprecated. Test compatibility in CI by publishing events with both old and new schemas.
