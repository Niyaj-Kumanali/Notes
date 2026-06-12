# RabbitMQ

---

## Overview

- **Definition:** RabbitMQ is an open-source message broker implementing AMQP (Advanced Message Queuing Protocol), written in Erlang, known for its flexible routing through exchanges and bindings.
- **Why It Exists:** Systems need reliable, decoupled communication with flexible routing. RabbitMQ provides robust messaging with multiple exchange types, delivery guarantees, and support for various protocols (AMQP, MQTT, STOMP).
- **Key Concepts:** **Producer** (publishes to exchange), **Exchange** (routes messages based on bindings), **Binding** (link between exchange and queue with a routing key), **Queue** (stores messages until consumed), **Consumer** (receives messages), **DLQ** (dead letter queue for failed messages), **Virtual Host** (isolates tenants).

---

## Exchange Types

- **Direct:** Matches routing key exactly — point-to-point messaging.
- **Topic:** Matches routing key patterns with wildcards (`*` for one word, `#` for zero or more) — pub-sub with filtering.
- **Fanout:** Routes to all bound queues, ignoring the routing key — broadcast.
- **Headers:** Matches message headers (not routing key) — content-based routing.

```
Producer -> Exchange -> (Binding match) -> Queue -> Consumer (ACK)
                                              -> DLQ (on failure)
                                              -> TTL expiry
```

### AMQP Frame Structure

```
| Frame Type (1 byte) | Channel (2 bytes) | Size (4 bytes) | Payload | End (1 byte) |
```

### Publisher Confirms

```java
channel.confirmSelect();
channel.basicPublish(exchange, routingKey, null, message.getBytes());
channel.waitForConfirmsOrDie(5000);
```

- **Quorum Queues:** Raft-based replicated queues for high availability.
- **Lazy Queues:** Messages stored on disk immediately — reduces RAM for large queues.
- **Streams:** Append-only log for replayable message consumption.

### Spring Boot Configuration

```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    virtual-host: /
    listener:
      simple:
        concurrency: 5
        max-concurrency: 10
        acknowledge-mode: manual
        prefetch: 10
```

### Declarative Queues, Exchanges, and Bindings

```java
@Configuration
public class RabbitMQConfig {
    @Bean
    public DirectExchange orderExchange() {
        return ExchangeBuilder.directExchange("order.exchange").durable(true).build();
    }

    @Bean
    public Queue orderCreatedQueue() {
        return QueueBuilder.durable("order.created.queue")
            .deadLetterExchange("order.dlx")
            .deadLetterRoutingKey("order.dead")
            .messageTtl(86400000)
            .maxLength(100000)
            .build();
    }

    @Bean
    public Binding orderCreatedBinding() {
        return BindingBuilder.bind(orderCreatedQueue())
            .to(orderExchange()).with("order.created");
    }

    @Bean
    public Queue deadLetterQueue() {
        return QueueBuilder.durable("order.dlq").build();
    }

    @Bean
    public DirectExchange deadLetterExchange() {
        return ExchangeBuilder.directExchange("order.dlx").build();
    }

    @Bean
    public Binding deadLetterBinding() {
        return BindingBuilder.bind(deadLetterQueue())
            .to(deadLetterExchange()).with("order.dead");
    }
}
```

### Producer with Publisher Confirms

```java
@Service
public class OrderEventPublisher {
    private final RabbitTemplate rabbitTemplate;

    public void publishOrderCreated(OrderEvent event) {
        CorrelationData correlationData = new CorrelationData(UUID.randomUUID().toString());
        rabbitTemplate.convertAndSend("order.exchange", "order.created", event,
            message -> {
                message.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
                message.getMessageProperties().setExpiration("3600000");
                return message;
            }, correlationData);
        correlationData.getFuture().whenComplete((confirm, ex) -> {
            if (confirm != null && confirm.isAck()) {
                log.info("Confirmed: {}", correlationData.getId());
            } else {
                handlePublishFailure(event);
            }
        });
    }
}
```

### Consumer with Manual ACK and DLQ Routing

```java
@Component
public class OrderCreatedConsumer {
    @RabbitListener(queues = "order.created.queue",
                    containerFactory = "rabbitListenerContainerFactory")
    public void handleOrderCreated(OrderEvent event, Message message, Channel channel)
            throws IOException {
        long deliveryTag = message.getMessageProperties().getDeliveryTag();
        try {
            orderService.processNewOrder(event);
            channel.basicAck(deliveryTag, false);
        } catch (RetryableException e) {
            channel.basicNack(deliveryTag, false, true);   // requeue
        } catch (FatalException e) {
            channel.basicNack(deliveryTag, false, false);  // -> DLQ
        }
    }
}
```

### Retry with Dead Lettering

```java
@Bean
public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
        ConnectionFactory connectionFactory) {
    SimpleRabbitListenerContainerFactory factory = new SimpleRabbitListenerContainerFactory();
    factory.setConnectionFactory(connectionFactory);
    factory.setConcurrentConsumers(5);
    factory.setMaxConcurrentConsumers(10);
    factory.setPrefetchCount(10);
    factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
    factory.setDefaultRequeueRejected(false);
    factory.setAdviceChain(RetryInterceptorBuilder.stateless()
        .maxAttempts(3)
        .backOffOptions(1000, 2.0, 10000)
        .recoverer(new RejectAndDontRequeueRecoverer())
        .build());
    return factory;
}
```

### Request-Reply Pattern

```java
@Service
public class OrderStatusClient {
    private final RabbitTemplate rabbitTemplate;

    public OrderStatus requestOrderStatus(String orderId) {
        return (OrderStatus) rabbitTemplate.convertSendAndReceive(
            "order.exchange", "order.status.request",
            new OrderStatusRequest(orderId));
    }
}
```

---

## Common Mistakes

- **No dead letter configuration** — failed messages remain and block the queue.
  - **Why it looks correct:** The message stays in the queue and the consumer keeps trying — "persistence means we'll process it eventually" — but each failed attempt blocks subsequent messages.
- **Forgetting manual ACK** — auto-ACK loses messages on consumer crash.
  - **Why it looks correct:** During normal operation auto-ACK works perfectly — the message is acknowledged immediately and processing happens without error. The loss only surfaces when the consumer crashes between auto-ACK and actual processing.
- **Unlimited queue growth** — set max length and TTL to prevent unbounded growth.
  - **Why it looks correct:** Queues are designed to hold messages — unbounded growth only becomes a problem under sustained consumer lag, at which point RabbitMQ hits the memory watermark and blocks all publishers.
- **Synchronous publishing** — blocks producer thread; use async confirms.
  - **Why it looks correct:** `waitForConfirmsOrDie` returns quickly during low traffic — the blocking cost only appears when broker write latency increases under load, serializing all producer threads.
- **Not handling poison messages** — consistently failing messages need DLQ routing.
  - **Why it looks correct:** A single failure looks transient — "retry and it'll pass" — but poison messages loop forever, consuming consumer resources and stalling the queue.
- **Incorrect exchange type** — using direct when topic or fanout is more appropriate.
  - **Why it looks correct:** Direct exchange works for the initial use case — the design only becomes limiting when new consumers need selective message filtering, requiring a migration.
- **Binding mismatches** — routing keys must match between publisher and consumer bindings.
  - **Why it looks correct:** The code compiles and runs without errors — messages simply disappear into the exchange with no consumer receiving them, a silent failure that looks like "the queue is empty."
- **No monitoring** — queue depth, consumer lag, and message rates must be tracked.
  - **Why it looks correct:** The system works in isolation — the first indication of a problem is a pager alert about cascading failures from an overflowing queue.

---

## Key Design Considerations

- **Cluster Size:** 3 nodes minimum, odd number for quorum queue consensus
- **Memory High Watermark:** Set `vm_memory_high_watermark` to 0.4–0.6
- **Prefetch Tuning:** Low prefetch (1–10) for fair distribution; high prefetch (100–1000) for throughput. Rule: prefetch = 2 × avg processing time (s) × desired throughput
- **Queue Types:** **Quorum** (Raft HA), **Lazy** (disk-based for large queues), **Stream** (append-only log for replay)
- **High Availability:** Use quorum queues with 3–5 nodes; configure `x-quorum-initial-group-size`
- **Architectural Patterns:** Competing Consumers, Work Queues, RPC, Dead Lettering, Priority Queues, Transactional enqueue/dequeue
- **Security:** TLS with client certificates, SASL authentication, per-vhost permissions, topic-level permissions

```yaml
spring:
  rabbitmq:
    ssl:
      enabled: true
      key-store: classpath:client-key.p12
      trust-store: classpath:truststore.jks
```

### Throughput Benchmarks

```
Direct Exchange:    ~50,000 msg/s (persistent)
Fanout Exchange:    ~45,000 msg/s (persistent)
Topic Exchange:     ~40,000 msg/s (persistent)
Non-persistent:     ~100,000 msg/s
```

---

## Real-World Scenarios

### Scenario 1: Poison Message Blocking Queue Processing
**Context:** An order processing queue receives a malformed message — the payload is missing a required `orderId` field. The consumer tries to process it, throws a `NullPointerException`, and rejects the message. RabbitMQ requeues it (because `basicNack` with `requeue=true` was called). The consumer immediately tries again, fails again, requeues again. This cycle repeats indefinitely. The malformed message blocks processing of all other messages behind it in the queue. Order processing is stalled.

**Resolution:** Configure dead letter exchange (DLX) on the queue. Messages that are rejected with `requeue=false` or expire move to the DLX and then to the dead letter queue. Consumers handle processing errors with `basicNack(deliveryTag, false, false)` — don't requeue, route to DLQ instead.

```java
@RabbitListener(queues = "order.created.queue")
public void handleOrderCreated(OrderEvent event, Message message, Channel channel) throws IOException {
    long deliveryTag = message.getMessageProperties().getDeliveryTag();
    try {
        orderService.processNewOrder(event);
        channel.basicAck(deliveryTag, false);
    } catch (Exception e) {
        // Don't requeue — route to DLQ instead
        channel.basicNack(deliveryTag, false, false);
    }
}
```

### Scenario 2: Message Ordering with Multiple Consumers
**Context:** An e-commerce saga processes `OrderCreated`, `PaymentProcessed`, and `OrderShipped` events for the same order. The queue has 3 consumers. The events arrive at the exchange with routing keys `order.created`, `order.paid`, `order.shipped`. These route to different queues (bound with different routing keys). Each queue has its own consumer, processing independently. `OrderShipped` arrives and is consumed before `OrderCreated` because the shipping queue has no backlog while the creation queue has 1000 messages. The shipping logic tries to ship an order that doesn't exist yet.

**Resolution:** Route all events for the same entity to the same queue using a consistent routing key. For order events, use routing key `order.{orderId}`. All events for order 123 go to the same queue with routing key `order.123`. A single consumer processes them sequentially. This guarantees order at the cost of limiting parallelism per order.

```java
// Publish all order lifecycle events with the same routing key
String routingKey = "order." + event.getOrderId();
rabbitTemplate.convertAndSend("order.exchange", routingKey, event);
```

For multi-step sagas, use a single queue per saga instance where all events for that saga route to. Each saga has one consumer.

### Scenario 3: Queue Backlog Causing Memory Exhaustion
**Context:** A notification service consumes from a queue that receives 1M messages/minute during a promotional campaign. The consumer processes at 500K messages/minute — backlog grows at 500K/minute. The queue is a default (non-lazy) queue, so messages are held in RAM. After 10 minutes, RabbitMQ's memory usage reaches `vm_memory_high_watermark` (default 40%). RabbitMQ blocks publishers. The entire cluster stops accepting messages. All services that publish to any queue on this cluster are blocked — cascading failure.

**Resolution:** (1) Configure the queue as lazy (`x-queue-mode=lazy`) — messages are written to disk immediately, not kept in RAM. This increases latency slightly but prevents memory exhaustion. (2) Set queue max length (`x-max-length`) or TTL (`x-message-ttl`) to bound backlog. (3) Scale consumers (more instances, larger prefetch). (4) Use consumer prefetch limits to control flow. (5) Monitor queue length and alert before critical thresholds.

```java
@Bean
public Queue notificationQueue() {
    return QueueBuilder.durable("notification.queue")
        .lazy()                                // Store on disk, not in RAM
        .maxLength(1000000)                    // Max 1M messages
        .overflow(OverflowBehavior.rejectPublish)  // Reject new when full
        .build();
}
```

---

## Scenario-Based Questions

1. **Q: Design an e-commerce order processing system with RabbitMQ. Requirements: Inventory, Payment, Notification, and Analytics each need to process every order independently. Inventory and Payment must process before shipping. How do you route messages?**
    - A: (1) Use a topic exchange with routing key `order.created`. (2) Inventory and Payment bind with routing key `order.created`. (3) When Inventory and Payment both complete, they publish to a different exchange with routing key `order.ready`. (4) Shipping binds to `order.ready`. (5) Notification and Analytics bind to `order.#` to get all order events. (6) Each service has its own queue with manual ACK and DLQ. (7) Use a saga coordinator or correlation ID to track completion of Inventory + Payment before triggering Shipping.
    - **Interview follow-up:** If Inventory succeeds but Payment fails, the saga must compensate — but what happens if the compensation message (CancelInventory) is published but never reaches the inventory consumer? How do you ensure the compensation runs?

2. **Q: How do you implement exactly-once delivery with RabbitMQ between services?**
   - A: (1) Publisher confirms: enable `publisherConfirms(true)` on the channel, wait for confirms, retry on nack. (2) Mandatory flag: `basicPublish` with `mandatory=true` — the broker returns unroutable messages. (3) Consumer manual ACK: process the message, then `basicAck`. Never auto-ACK. (4) Idempotent consumer: deduplicate using a message ID or business key stored in the database (`INSERT ... ON CONFLICT DO NOTHING`). (5) Quorum queues for broker-side reliability. RabbitMQ cannot guarantee exactly-once across network boundaries — at-least-once with idempotent sinks is the practical standard.

3. **Q: Your RabbitMQ cluster uses classic mirrored queues. During a network partition, the cluster splits, and some messages are lost when the partition heals. How do quorum queues prevent this?**
    - A: Quorum queues use the Raft consensus algorithm: a majority of nodes must agree on every operation. During a network partition, only the partition with a majority (e.g., 2 out of 3 nodes) can accept messages. The minority partition stops accepting writes. When the partition heals, the minority's data is discarded (Raft's safety guarantee). Classic mirrored queues use async master-slave replication — during a split-brain, both sides can accept writes, leading to data loss on merge. Quorum queues also have a delivery limit (default 3) — messages that fail repeatedly are automatically dead-lettered, preventing poison message cycling.
    - **Interview follow-up:** Quorum queues require a majority for every operation — what happens to availability when you lose 2 out of 3 nodes? The queue becomes unavailable until a majority is restored. How do you plan for this in a multi-AZ deployment?

4. **Q: Your payment processing service needs to retry failed messages with exponential backoff (1s, 2s, 4s, 8s). How do you implement this without custom code?**
   - A: (1) Create 4 retry queues, each with different TTL: `retry-1s` (TTL=1000ms), `retry-2s` (TTL=2000ms), `retry-4s` (TTL=4000ms), `retry-8s` (TTL=8000ms). (2) Each retry queue is bound to a retry exchange. When TTL expires, the message is dead-lettered back to the original exchange (which routes to the original queue). (3) Use the `x-delayed-message` exchange plugin: set `x-delay` header on each message (in ms). The exchange holds the message until the delay expires, then routes normally. This requires only one retry queue. (4) Track retry count in message headers — after 3 retries, route to DLQ instead of retry.

5. **Q: Your multi-tenant SaaS application uses RabbitMQ. Tenant A's high-volume traffic slows down Tenant B's message processing. How do you isolate tenants?**
   - A: (1) One virtual host (vhost) per tenant — complete isolation, each tenant has its own exchanges, queues, bindings, and permissions. (2) Separate RabbitMQ clusters for premium tenants (dedicated resources). (3) Per-tenant connection limits and queue max length to prevent one tenant from exhausting cluster resources. (4) Use lazy queues for all tenants — no tenant's backlog consumes RAM that affects others. (5) Monitor per-tenant metrics separately and enforce per-tenant throughput quotas at the publisher level.

6. **Q: Your order processing must ensure that orders from the same user are processed in sequence. Different users can be processed in parallel. How do you design the routing?**
   - A: (1) Use a direct exchange with routing key set to `user.{userId}`. (2) Create multiple queues, each bound to a range of user IDs (e.g., `queue-0` handles users with IDs ending in 0, `queue-1` handles IDs ending in 1, etc.). (3) Each queue has one consumer — sequential processing per user within that queue. (4) This provides parallelism across users while ensuring order per user. (5) If user 123 and user 456 are hashed to different queues, they process in parallel. If user 123 has two messages, they go to the same queue and process in order.

7. **Q: Your notification system needs priority processing: password reset emails must be sent within 1 minute, but newsletter emails can wait 1 hour. How do you implement priority queues in RabbitMQ?**
   - A: (1) Set `x-max-priority: 10` on the queue. (2) Producers set `priority` on messages (password_reset=10, newsletter=1). Higher priority messages jump ahead in the queue. (3) For strict priority guarantees: create separate queues per priority level (`priority-high`, `priority-medium`, `priority-low`). Dedicated consumers per queue. (4) Use a topic exchange — a single publisher can route to different queues based on routing key (e.g., `notification.password.reset`, `notification.newsletter`). (5) Monitor priority queue lengths separately.

8. **Q: You need to implement event sourcing for a banking system using RabbitMQ. Events must be replayable. A new microservice needs to rebuild state from scratch by reading past events. How do you design this?**
   - A: (1) Use RabbitMQ streams (available since RabbitMQ 3.9). Streams are append-only logs that keep messages indefinitely (configurable retention). Consumers can replay from any offset or timestamp. (2) Each aggregate type (account, transaction) gets its own stream. (3) New services create a stream consumer starting from offset 0 to replay all events. (4) For fast recovery, store periodic snapshots of aggregate state in a database. On restart, load the latest snapshot and replay only events newer than the snapshot. (5) Use a compacted stream topic for "latest state" per aggregate ID — similar to Kafka's log compaction.

9. **Q: Your RabbitMQ broker runs on a single node. You need high availability without losing messages during node failures. How do you configure this?**
   - A: (1) Set up a 3-node RabbitMQ cluster. (2) Use quorum queues (Raft-based) instead of classic mirrored queues — they provide stronger consistency guarantees and automatic leader election. (3) Configure `x-quorum-initial-group-size=3` for the queue. (4) Publishers use publisher confirms and mandatory flag. (5) Consumers use manual ACK. (6) Use a load balancer in front of the cluster — clients connect to the load balancer, not individual nodes. (7) For cross-datacenter HA, use the Federation plugin to replicate messages to a DR cluster.

10. **Q: Your application publishes 100MB image files as messages directly to RabbitMQ. The broker's memory usage spikes and performance degrades. How do you handle large payloads?**
    - A: (1) Best practice: store the image in external storage (S3, HDFS, local filesystem) and send only the file reference (URL + metadata) in the RabbitMQ message. (2) If external storage is not an option: set `max_message_size` on the broker (e.g., 200MB) and use lazy queues (`x-queue-mode=lazy`) — messages are written to disk immediately, not held in RAM. (3) Set `vm_memory_high_watermark` to a lower value (e.g., 0.3) to leave headroom for message processing. (4) Compress payloads with gzip before publishing. (5) Implement client-side chunking: split the file into 1MB chunks, send as separate messages with a sequence number, and reassemble on the consumer.
    - **Interview follow-up:** With client-side chunking, one chunk fails and is requeued while others succeed — how do you prevent partial reassembly and ensure the entire file is either delivered or discarded atomically?

---

## Interview Questions

1. **What is RabbitMQ and how does it work?**
   - A: An open-source message broker implementing AMQP. Producers publish to exchanges, which route messages to queues based on bindings and routing keys. Consumers subscribe to queues. Supports multiple exchange types (direct, topic, fanout, headers).

2. **What are the different exchange types in RabbitMQ?**
   - A: Direct (exact routing key match), Topic (pattern matching with `*` and `#` wildcards), Fanout (broadcast to all bound queues), Headers (match on message headers). Each serves different routing needs.

3. **What is the difference between RabbitMQ and Kafka?**
   - A: RabbitMQ is push-based with flexible routing (exchanges, bindings) and complex messaging patterns (RPC, pub-sub, work queues). Kafka is pull-based with an append-only log, designed for high-throughput event streaming and replay.

4. **How do you ensure reliable message delivery in RabbitMQ?**
   - A: Publisher confirms (ack from broker), mandatory flag (return unroutable messages), persistent messages (delivery_mode=2), quorum queues for HA, manual consumer ACK, dead letter queues for failed messages.

5. **What are quorum queues?**
   - A: Raft-based replicated queues providing strong consistency, automatic leader election, and network partition tolerance. Require a majority of nodes. Have a delivery limit to prevent poison message cycling. Use instead of classic mirrored queues.

6. **What is a dead letter queue and why is it important?**
   - A: A DLQ stores messages that couldn't be processed (rejected, expired, exceeded max length). Without DLQ, failed messages are requeued indefinitely — blocking processing of other messages. DLQ enables inspection, debugging, and reprocessing.

7. **How do you handle message ordering in RabbitMQ?**
   - A: Use a single queue with one consumer. Route related messages (same entity ID) to the same queue using a consistent routing key. Different entities can use different queues for parallelism.

8. **What is the difference between manual ACK and auto ACK?**
   - A: Auto ACK acknowledges immediately on delivery — if the consumer crashes, the message is lost. Manual ACK acknowledges after processing — if the consumer crashes, the message is redelivered to another consumer. Manual ACK provides at-least-once delivery.

9. **How do you scale RabbitMQ consumers?**
   - A: Add more consumers to the same queue (competing consumers pattern). Increase prefetch count for higher throughput. Use multiple queues for parallelism. Add more nodes to the cluster. Use quorum queues for HA scaling.

10. **What is the purpose of prefetch count?**
    - A: Controls how many messages are sent to a consumer before acknowledgment. Low prefetch (1-10): fair distribution, suitable for varied message processing times. High prefetch (100-1000): higher throughput, risk of uneven load. Rule: prefetch = 2 × avg processing time (s) × desired throughput.

---

## Developer Recommendations

- **Always configure dead letter queues** — Without DLQ, a single malformed message blocks the entire queue indefinitely (reject → requeue → reject → requeue cycle). Every production queue should have a DLX/DLQ configured. Set `x-dead-letter-exchange` and `x-dead-letter-routing-key` on every queue. Monitor DLQ size — a growing DLQ indicates systematic processing failures that need attention, not individual poison messages.
  - **Production story:** During Black Friday one year, a pricing service deployed a schema change that produced messages with a missing `currency` field — the downstream consumer rejected every one. The queue froze for 37 minutes before someone manually purged it. A DLQ would have isolated the bad messages in seconds.

- **Use manual ACK, never auto ACK** — Auto ACK acknowledges messages as soon as they are delivered to the consumer. If the consumer crashes before processing, the message is lost forever. Manual ACK (`basicAck` after successful processing, `basicNack` after failure) provides at-least-once delivery. The trade-off: manual ACK requires more code and can lead to duplicate processing if the ACK is lost. Use idempotent processing to handle duplicates safely.

- **Use quorum queues instead of classic mirrored queues** — Classic mirrored queues use asynchronous master-slave replication — during network partitions, both sides can accept writes, leading to data loss. Quorum queues use Raft consensus — requiring majority approval for every operation, preventing split-brain. Quorum queues also have automatic delivery limits (messages rejected N times go to DLQ). Migrate all production queues to quorum queues.

- **Use lazy queues for any queue that can accumulate backlog** — Default queues hold messages in RAM for performance. When a consumer falls behind (e.g., during a traffic spike), backlog growth consumes RAM until the high-water mark, then RabbitMQ blocks all publishers. Lazy queues write messages to disk immediately — preventing memory exhaustion. The trade-off: slightly higher latency (disk I/O instead of RAM). Acceptable for preventing cascading failures.

- **Set queue max length and TTL** — Unbounded queues grow indefinitely during consumer downtime. A queue with 10M messages consumes resources even if the messages are outdated. Set `x-max-length` (max number of messages) and `x-message-ttl` (max time in queue). Overflow behavior: `drop-head` (delete oldest), `reject-publish` (reject new). For time-sensitive data (notifications, events), TTL is critical — stale messages are worse than lost messages.

- **Use the topic exchange for most production use cases** — Direct exchange is too rigid (exact routing key match only). Fanout is too broad (no filtering). Topic exchange combines the best of both: pattern matching with `*` (one word) and `#` (zero or more) wildcards. This supports pub-sub, filtering, and point-to-point with a single exchange type. Bind multiple queues with different patterns — each gets the subset of messages they need.
