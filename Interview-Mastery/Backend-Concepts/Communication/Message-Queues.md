# Message Queues

---

## Overview

- **Definition:** Message queues are asynchronous communication mechanisms where producers send messages to a queue and consumers process them independently, enabling decoupled, reliable, and scalable message exchange.
- **Why It Exists:** Synchronous communication tightly couples services, limits scalability, and is vulnerable to cascading failures. Queues provide buffering, load leveling, fault tolerance, and guaranteed delivery.
- **Key Concepts:** **Producer** (sends messages), **Consumer** (processes messages), **Queue** (buffer that stores messages until consumed), **Exchange** (routes messages based on rules), **Broker** (server managing queues and routing), **Binding** (link between exchange and queue), **DLQ** (dead letter queue for failed messages).

---

## Core Concepts

### Message Delivery Semantics

- **At-Most-Once:** Message may be lost but never duplicated — fire and forget, no ACK.
- **At-Least-Once:** Message is never lost but may be duplicated — ACK after processing, retry on failure.
- **Exactly-Once:** Message is delivered precisely once — ACK + deduplication + idempotent consumers.

### Message Patterns

```
Point-to-Point:  One producer -> Queue -> One consumer
Publish-Subscribe: One producer -> Topic -> Multiple consumers
Request-Reply:   Producer sends request, consumer replies via callback queue
Dead Letter:     Failed messages routed to DLQ for analysis/replay
```

### Message Lifecycle

1. **Producer** creates and publishes a message to the broker.
2. **Broker** receives, validates, and persists the message.
3. **Broker** routes the message to the appropriate queue(s).
4. **Consumer** polls the queue or receives a push notification.
5. **Consumer** processes the message and sends an ACK.
6. **Broker** removes the acknowledged message from the queue.
7. On NACK, the message is requeued or sent to DLQ.

### Spring Boot JMS Configuration

```java
@Configuration
@EnableJms
public class JmsConfig {
    @Bean
    public JmsListenerContainerFactory<?> jmsListenerContainerFactory(
            ConnectionFactory connectionFactory) {
        DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setConcurrency("3-10");
        factory.setSessionAcknowledgeMode(Session.CLIENT_ACKNOWLEDGE);
        return factory;
    }
}
```

### Producer & Consumer

```java
@Component
public class OrderMessageProducer {
    private final JmsTemplate jmsTemplate;

    public void sendOrderCreated(OrderEvent event) {
        jmsTemplate.convertAndSend("order.created.queue", event,
            message -> {
                message.setStringProperty("eventType", event.getType());
                message.setJMSCorrelationID(UUID.randomUUID().toString());
                message.setJMSExpiration(TimeUnit.HOURS.toMillis(24));
                return message;
            });
    }
}

@Component
public class OrderMessageConsumer {
    @JmsListener(destination = "order.created.queue",
                 containerFactory = "jmsListenerContainerFactory")
    public void handleOrderCreated(OrderEvent event) {
        orderService.processNewOrder(event.getOrderId());
    }
}
```

### Idempotent Consumer

```java
@Component
public class IdempotentConsumer {
    private final Set<String> processedIds = ConcurrentHashMap.newKeySet();

    @JmsListener(destination = "payment.events")
    public void handlePaymentEvent(PaymentEvent event) {
        if (!processedIds.add(event.getEventId())) {
            log.info("Duplicate ignored: {}", event.getEventId());
            return;
        }
        orderService.processPayment(event.getOrderId(), event.getAmount());
    }
}
```

### Retry with Exponential Backoff

```java
@Component
public class RetryMessageConsumer {
    private final RetryTemplate retryTemplate = new RetryTemplate();

    @JmsListener(destination = "critical.events")
    public void handleCriticalEvent(Message message) {
        retryTemplate.execute(context -> {
            try {
                processMessage(message);
                return null;
            } catch (Exception e) {
                if (context.getRetryCount() >= 2) {
                    deadLetterQueue.send(message);
                }
                throw e;
            }
        });
    }
}
```

---

## Common Mistakes

- **Not handling poison messages** — messages that consistently fail block the queue
- **Forgetting idempotency** — duplicate messages are inevitable; consumers must be idempotent
- **Tight coupling** — using RPC-style request-reply instead of async messaging
- **Ignoring message size limits** — large messages consume memory and network
- **No monitoring** — not tracking queue depth, consumer lag, or processing times
- **Blocking consumer threads** — never block in message listeners
- **Swallowing exceptions** — always ACK or NACK; don't silently eat errors
- **No dead letter queue** — failed messages accumulate forever

---

## Key Design Considerations

- **When to Use:** Decoupling microservices, load leveling, async processing, event-driven architectures, reliable delivery
- **When NOT to Use:** Real-time request-response (use gRPC/REST), simple CRUD, small/trivial applications
- **Patterns:** **Event Sourcing**, **CQRS**, **Saga** (distributed transaction coordination), **Transactional Outbox** (reliably publish from DB changes), **Competing Consumers**
- **Monitoring:** Queue depth, consumer lag, processing time, error rates, and backlog alerting thresholds
- **Capacity Planning:** Expected throughput, retention period, cross-region failover
- **Versioning:** Message schema evolution — use serialization with type headers
- **Security:** SSL/TLS in transit, encryption at rest, role-based authorization, message signing, audit logging

```yaml
spring:
  activemq:
    broker-url: tcp://localhost:61616
    pool:
      enabled: true
      max-connections: 50
  jms:
    listener:
      concurrency: 5-20
      max-messages-per-task: 10
```

---

## Real-World Scenarios

### Scenario 1: Order Processing with Competing Consumers
**Context:** An e-commerce platform needs to process 10,000 orders per minute during Black Friday. Each order requires inventory check, payment validation, fraud detection, and analytics recording. Processing must be reliable — no orders lost.

**Resolution:** Use a message queue with competing consumers. The order service publishes an `OrderPlaced` message to a queue. Multiple consumer instances (10-20) subscribe to the same queue — each picks up messages as capacity allows. The queue buffers traffic spikes. If all consumers are busy, messages wait in the queue. Each consumer ACKs after successful processing. Failed messages go to a DLQ after 3 retries.

```java
// Producer
@Component
public class OrderEventProducer {
    private final JmsTemplate jmsTemplate;

    public void sendOrder(OrderEvent event) {
        jmsTemplate.convertAndSend("order.processing.queue", event,
            msg -> {
                msg.setJMSCorrelationID(event.getOrderId());
                msg.setJMSPriority(event.isPremiumCustomer() ? 9 : 4);
                msg.setJMSExpiration(TimeUnit.HOURS.toMillis(24));
                return msg;
            });
    }
}

// Consumer with competing consumer pattern
@Component
public class OrderProcessingConsumer {
    @JmsListener(destination = "order.processing.queue",
                 containerFactory = "jmsListenerContainerFactory",
                 concurrency = "5-10")
    public void processOrder(OrderEvent event) {
        try {
            inventoryService.reserve(event.getProductId(), event.getQuantity());
            paymentService.charge(event.getCustomerId(), event.getTotal());
            fraudDetectionService.analyze(event);
            analyticsService.record(event);
        } catch (Exception e) {
            throw new RuntimeException("Processing failed", e); // Triggers rollback/retry
        }
    }
}
```

### Scenario 2: Dead Letter Queue for Failed Payments
**Context:** A payment processing system receives 50K messages/day. 2% fail due to invalid credit cards, insufficient funds, or expired cards. Without proper handling, these poison messages block the queue and stop processing.

**Resolution:** Implement a DLQ. After 3 failed processing attempts (with exponential backoff), the message is routed to a dead letter queue. A monitoring tool alerts when the DLQ grows. Operations analyzes failed messages, contacts customers for updated payment info, and replays fixed messages.

### Scenario 3: Transactional Outbox for Reliable Events
**Context:** A user service updates a user's email address and must publish a `UserEmailChanged` event. If the database update succeeds but the message publish fails, downstream services have stale data.

**Resolution:** Implement the transactional outbox pattern. Within the same database transaction, both the user email update and an outbox record are written. A scheduled `OutboxRelay` polls for unprocessed outbox records and publishes them reliably. This ensures exactly-once publication of the event.

---

## Scenario-Based Questions

1. **Q: You're building an order processing system that must handle 100x traffic spikes during flash sales. Orders must not be lost. How do you design the messaging infrastructure?**
   - A: Use a message queue with persistent messages and at-least-once delivery. The queue buffers traffic spikes — producers publish freely, consumers process at their own pace. Set queue depth alerts at 80% capacity. Pre-scale consumers during known sale events. Use dead letter queues for failed messages. For the database, batch writes from the consumer to handle the burst. The queue acts as a shock absorber.

2. **Q: During a deployment, a bug causes your consumer to crash-loop on every message. Messages are constantly requeued and reprocessed, blocking the queue. The downstream system is never updated. How do you fix this?**
   - A: Implement a poison message handler. After N failed attempts (e.g., 3), move the message to a DLQ instead of requeuing. Use a retry count header or broker-specific dead letter feature. The consumer continues processing other messages. Analyze DLQ messages to identify the bug, deploy the fix, and replay affected messages. Without DLQ, one bad message can block all processing.

3. **Q: Your event-driven system processes user registrations. A duplicate message causes the same user to be registered twice (duplicate email, username). How do you prevent this?**
   - A: Idempotent consumers. Each message carries a unique event ID (UUID). Before processing, the consumer checks a deduplication store (Redis `SETNX` with TTL of 7 days, or a database unique constraint on `event_id`). If the event was already processed, skip it. Additionally, make the business operation idempotent — the user registration uses `INSERT ... ON CONFLICT (email) DO NOTHING`.

4. **Q: Your queue consumers process messages but one consumer is much slower than others. Messages pile up on that consumer while others sit idle. How do you balance the load?**
   - A: Use competing consumers with appropriate prefetch settings. Set prefetch to 1 (or a low number) so each consumer picks one message, processes it, ACKs it, then picks another. This ensures faster consumers process more messages than slower ones. Avoid setting prefetch too high (100+), which lets fast consumers grab all messages and starve slower ones.

5. **Q: You need to process high-priority orders (premium customers) before standard orders. Your queue is FIFO. How do you implement priority processing?**
   - A: Multiple approaches: (1) Use message priority headers (JMS priority 0-9) — higher priority messages are delivered first. (2) Use separate queues per priority tier (`order.high`, `order.normal`, `order.low`) with dedicated consumers. (3) Use a weighted round-robin consumer that polls high-priority queue 3x more often than normal. For strict priority, separate queues with dedicated consumers is the most reliable.

6. **Q: Your message broker goes down for 10 minutes. When it comes back, messages published during the outage are lost. Producers didn't get errors because they used fire-and-forget. How do you prevent this?**
   - A: Use publisher confirms (ack from broker) instead of fire-and-forget. Configure synchronous sends or async confirms with a callback. On failure, retry with exponential backoff. For critical messages, use the transactional outbox pattern — write the message to a database first, then have a relay publish it. The database survives the broker outage.

7. **Q: You're migrating from ActiveMQ to RabbitMQ. How do you do this without any message loss and zero downtime?**
   - A: Dual-publish strategy: (1) Configure the application to publish to both ActiveMQ and RabbitMQ simultaneously. (2) Gradually migrate consumers from ActiveMQ to RabbitMQ. (3) Monitor both queues to ensure no message loss. (4) Once all consumers are migrated, stop publishing to ActiveMQ. (5) Use a bridge for any messages still in the old queue. This allows rollback at any step.

8. **Q: Your consumer processes messages from a queue, but when it calls an external API that's slow, all consumer threads block, and no messages are processed. How do you implement backpressure?**
   - A: Use prefetch limits — set prefetch to 1-3 so the consumer holds only a few unacknowledged messages. If downstream is slow, the consumer doesn't prefetch more. Monitor queue depth growth as a signal of downstream issues. Implement a circuit breaker on the external API call — if the API is slow, fail fast and NACK the message (sending to DLQ or retry queue). Use separate thread pools for external calls to avoid blocking consumer threads.

9. **Q: Your queue has messages with different processing times: some take 10ms, some take 10 seconds. The slow messages block the fast ones because the queue is FIFO. How do you design around this?**
   - A: Use separate queues for fast and slow operations. Process fast operations in one queue with high concurrency and slow operations in another with fewer, longer-running consumers. Alternatively, use message grouping with a TTL — if a slow message exists, other messages in the same group wait, but messages in different groups proceed independently. For truly independent messages, the slow ones shouldn't block fast ones.

10. **Q: Your application runs in three regions (US, EU, APAC). A message published in the US must be processed in all three regions. How do you design cross-region message replication?**
    - A: Use a hub-and-spoke topology. The US region publishes to a local queue. A replication bridge (ActiveMQ network of brokers, RabbitMQ shovel/federation) copies the message to EU and APAC regions asynchronously. Each region's consumers process independently. For active-active, configure bidirectional replication with conflict resolution (last-writer-wins). Monitor replication lag as a critical metric.

---

## Interview Questions

1. **What is a message queue and why use one?**
   - A: A message queue is a buffer that stores messages between producers and consumers, enabling asynchronous, decoupled communication. Benefits: load leveling (buffers traffic spikes), fault tolerance (messages persisted until consumed), independent scaling (producers and consumers scale separately), and reliable delivery.

2. **What are the three message delivery semantics?**
   - A: At-most-once (message may be lost, never duplicated — fire and forget), At-least-once (message never lost, may be duplicated — ACK after processing), Exactly-once (message delivered precisely once — requires idempotent consumers + transactional broker support).

3. **What is a dead letter queue (DLQ)?**
   - A: A queue where messages that failed processing after all retry attempts are routed. Prevents poison messages from blocking the main queue. Operations analyzes DLQ messages, fixes underlying issues, and replays them. Essential for any production messaging system.

4. **What is the competing consumers pattern?**
   - A: Multiple consumer instances subscribe to the same queue. Each message is delivered to exactly one consumer. As consumers finish processing, they pick up the next message. Scales linearly with consumer count. Best for parallelizable workloads where message order doesn't matter.

5. **What is the transactional outbox pattern?**
   - A: A solution to the dual-write problem. Business data and outbox event are written in the same database transaction. A separate process (outbox relay) polls the outbox table and publishes events to the message broker. Ensures atomicity between DB write and message publication.

6. **How do you handle poison messages?**
   - A: Implement a retry counter (either in message headers or via broker DLQ feature). After N failed attempts (typically 3), route to a DLQ instead of requeuing. Set a TTL on retry queues. Monitor DLQ and alert on growth. Provide tools for operators to analyze and replay DLQ messages.

7. **What is the difference between point-to-point and publish-subscribe?**
   - A: Point-to-point (queue): one message consumed by one consumer — competing consumers share the load. Publish-subscribe (topic): one message consumed by all subscribers independently — each subscriber gets a copy. Choose based on whether you need fan-out or load balancing.

8. **How do you implement message ordering?**
   - A: Use a single partition/queue per entity (e.g., all messages for order 123 go to the same partition). Use a consistent partition key (order ID). Single consumer per partition processes messages sequentially. For global ordering, use a single partition (limits throughput).

9. **How do you implement idempotent consumers?**
   - A: Store processed message IDs in a deduplication store (Redis with TTL, database unique constraint). Before processing, check if the ID was already processed. Make business operations idempotent (upserts, not inserts). Test idempotency by replaying messages.

10. **How do you migrate between message brokers without downtime?**
    - A: Dual-publish strategy: publish to both old and new brokers. Gradually migrate consumers. Monitor both for message loss. Once migration is complete, stop publishing to the old broker. Use a bridge for remaining messages. Each step is revertable.

---

## Developer Recommendations

- **Always configure a dead letter queue** — Without a DLQ, a single poison message can block your entire queue. The message is requeued repeatedly, consuming resources and preventing other messages from being processed. Configure DLQ with TTL and max delivery count. Monitor DLQ depth and alert on growth — a growing DLQ indicates bugs or configuration issues that need attention.

- **Make consumers idempotent even with at-most-once delivery** — "At-most-once" sounds like you don't need idempotency, but network retries, consumer crashes, and broker failovers can still cause duplicates. The safest approach is to always make your consumer processing idempotent. Use upsert operations, check-then-act patterns within database transactions, and store deduplication keys.

- **Use prefetch limits to control consumer behavior** — High prefetch (100+) causes fast consumers to grab all messages, preventing fair distribution. Low prefetch (1-3) ensures each consumer picks one message at a time, distributing work fairly. For long-running processing tasks, use low prefetch. For high-throughput, short tasks, higher prefetch is acceptable. Rule of thumb: prefetch = 2 × desired concurrency × processing time (seconds).

- **Never use sync sends in hot paths** — Synchronous message publishing blocks the producer thread until the broker acknowledges. For high-throughput applications, this kills performance. Use async sends with callbacks (correlation IDs) or batch sends. The only exception is critical messages where you need immediate confirmation that the broker accepted the message.

- **Implement backpressure to prevent consumer overload** — A consumer that reads messages faster than it can process creates an ever-growing backlog in the consumer's memory. Use prefetch limits, monitor queue depth, and implement dynamic concurrency adjustment. If the downstream system is saturated, the consumer should stop pulling messages, not buffer them indefinitely.

- **Monitor queue depth, consumer lag, and processing time** — Queue depth tells you if producers are outpacing consumers. Consumer lag (time since the last message was published to the oldest unprocessed message) is your operational health metric. Processing time (P50/P95/P99) per message helps identify slow operations. Set up dashboards and alerts for all three.
