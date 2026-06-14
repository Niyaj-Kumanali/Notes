# Apache Kafka

---

## Overview

- **Definition:** Apache Kafka is a distributed event streaming platform built on a commit log architecture, providing high-throughput, fault-tolerant publish-subscribe messaging.
- **Why It Exists:** Traditional message brokers can't handle the scale, durability, and replayability needed for event streaming, data pipelines, and real-time analytics at internet scale.
- **Key Concepts:** **Broker** (server storing data), **Topic** (message category), **Partition** (ordered, immutable sequence within a topic), **Producer** (publishes messages), **Consumer** (reads messages), **Consumer Group** (load-balanced consumers), **Offset** (position identifier per partition), **ZooKeeper/KRaft** (cluster coordination).

---

## Core Architecture

- **Commit Log:** Append-only sequential I/O — the foundation of Kafka's speed.
- **Partition Leader & Followers:** Each partition has one leader (handles all reads/writes) and N in-sync replica (ISR) followers. On leader failure, an ISR follower is elected.
- **Zero Copy:** Data streams from disk to network without passing through application memory.

```
Topic
  |-- Partition 0 (ordered, immutable)
  |       |-- Record 0 (offset 0)
  |       |-- Record 1 (offset 1)
  |-- Partition 1
  |       |-- Record 0
  |-- Partition 2 (replicated to N brokers)
```

- **Retention Policies:** Time-based (delete after X hours), Size-based (delete oldest when topic exceeds Y bytes), Compact (keep latest per key).

### Producer Request Flow

1. Producer sends batch to partition leader.
2. Leader appends to its local log.
3. Based on `acks`:
   - `acks=0` — fire and forget (fastest, may lose data)
   - `acks=1` — leader acknowledges (default, moderate durability)
   - `acks=all` — all ISR replicas acknowledge (strongest durability)
4. Leader responds with offset.

### Consumer Poll Loop

```java
while (true) {
    ConsumerRecords<String, Order> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, Order> record : records) {
        processOrder(record.value());
    }
    consumer.commitAsync();
}
```

- **Consumer Groups:** Partitions are divided among consumers in a group. If a consumer fails, partitions rebalance to the remaining consumers.

---

## Common Mistakes

- **Too few partitions** — limits parallelism and throughput.
  - **Why it looks correct:** The topic works fine with a single partition in dev — the throughput ceiling only becomes visible under production traffic.
- **Consumer lag not monitored** — silently loses processing capability.
  - **Why it looks correct:** The consumers appear healthy (no errors, no crashes) — the growing lag is invisible until processing delays show up in customer-facing metrics.
- **No idempotent producer** — duplicate messages on failure.
  - **Why it looks correct:** Duplicates only happen when a producer retries after a timeout — a rare edge case that looks like the network's fault, not the configuration.
- **Auto offset commit enabled** — risk of missing messages on crash.
  - **Why it looks correct:** During normal operation the consumer processes everything — the gap only appears after a crash, when the committed offset has moved past unprocessed messages.
- **Synchronous production in hot path** — blocks and reduces throughput.
  - **Why it looks correct:** Each send returns quickly in isolation — the blocking delay accumulates across thousands of sends per second, throttling the producer.
- **No dead letter topic** — failed messages are lost forever.
  - **Why it looks correct:** Throwing the exception and moving on keeps the consumer running — the message is silently dropped and never investigated.
- **Rebalance storms** — frequent rebalances halt processing.
  - **Why it looks correct:** Each rebalance completes in seconds — the cumulative downtime across dozens of rebalances per hour is invisible to any single monitoring check.
- **Incorrect partitioning strategy** — uneven load across consumers.
  - **Why it looks correct:** Partition counts are balanced during assignment — the uneven data distribution across partitions only appears under real traffic patterns.
- **Ignoring 1MB default message size limit** — payloads exceeding 1MB are silently rejected by the broker.
  - **Why it looks correct:** Small test messages always succeed — the error surfaces only in production when a legitimate payload hits the ceiling.
- **Too many partitions** — increases overhead and rebalance time.
  - **Why it looks correct:** More partitions means more parallelism — the overhead in file descriptors, leader elections, and rebalance latency grows non-linearly and catches teams off guard.

---

## Key Design Considerations

- **Cluster Sizing:** 3 brokers minimum, 5 for production, odd number for quorum
- **Topic Design:** 3x replication factor; partitions = desired throughput / per-partition throughput
- **Partition Count:** Aim for 10–50 partitions per broker
- **Exactly-Once Semantics:** Idempotent producer + transactional producer/consumer + idempotent sinks
- **Schema Registry:** Avro/Protobuf schema management with compatibility checks
- **Disaster Recovery:** MirrorMaker 2 for cross-cluster replication
- **Monitoring:** Consumer lag, request rates, disk usage, network throughput
- **Security:** SSL/TLS, SASL (SCRAM, Kerberos), ACL-based authorization

### Spring Boot Configuration

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092,localhost:9093,localhost:9094
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      compression-type: snappy
      batch-size: 16384
      linger-ms: 5
    consumer:
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      group-id: order-service
      auto-offset-reset: earliest
      enable-auto-commit: false
      max-poll-records: 500
    listener:
      concurrency: 3
      ack-mode: manual_immediate
```

### Producer with Callback

```java
@Service
public class OrderEventProducer {
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void sendOrderCreated(OrderEvent event) {
        String key = event.getOrderId();
        ListenableFuture<SendResult<String, OrderEvent>> future =
            kafkaTemplate.send("order.created", key, event);
        future.whenComplete((result, ex) -> {
            if (ex == null) {
                log.info("Sent: topic={}, partition={}, offset={}",
                    result.getRecordMetadata().topic(),
                    result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset());
            } else {
                sendToDeadLetter(event, ex);
            }
        });
    }
}
```

### Consumer with Manual Acknowledgment

```java
@Component
public class OrderEventConsumer {
    @KafkaListener(topics = "order.created", concurrency = "3")
    public void handleOrderCreated(
            ConsumerRecord<String, OrderEvent> record,
            Acknowledgment acknowledgment) {
        try {
            orderService.processNewOrder(record.value());
            acknowledgment.acknowledge();
        } catch (Exception e) {
            if (getRetryCount(record) >= 3) {
                deadLetterProducer.send(record.value(), e);
                acknowledgment.acknowledge();
                return;
            }
            acknowledgment.nack(1000);
        }
    }
}
```

### Kafka Streams Processing

```java
@Configuration
@EnableKafkaStreams
public class OrderStreamProcessor {
    @Bean
    public KStream<String, OrderEvent> orderStream(StreamsBuilder builder) {
        KStream<String, OrderEvent> source = builder
            .stream("order.created",
                Consumed.with(Serdes.String(), new JsonSerde<>(OrderEvent.class)));

        KStream<String, OrderEvent> highValue = source
            .filter((key, event) -> event.getAmount().compareTo(new BigDecimal("1000")) > 0);
        highValue.to("order.high-value",
            Produced.with(Serdes.String(), new JsonSerde<>(OrderEvent.class)));

        KTable<String, Long> counts = source
            .groupBy((key, event) -> event.getStatus())
            .count(Materialized.as("order-status-counts"));
        counts.toStream().to("order.status-counts",
            Produced.with(Serdes.String(), Serdes.Long()));

        return source;
    }
}
```

---

## Real-World Scenarios

### Scenario 1: Duplicate Events from Producer Retry
**Context:** Your order service uses Kafka to publish `OrderCreated` events. A transient network error causes the producer to retry. The retry succeeds, but the original request also succeeded (the broker acknowledged but the ACK was lost). The consumer receives the same event twice — the order is processed twice, inventory is decremented twice, and the user is double-charged.

**Resolution:** Enable idempotent producer. With `enable.idempotence=true`, the producer assigns a unique producer ID (PID) and sequence number to each message. The broker deduplicates messages with the same PID and sequence number. This guarantees exactly-once publishing to a single partition.

```java
@Bean
public ProducerFactory<String, OrderEvent> producerFactory() {
    Map<String, Object> config = new HashMap<>();
    config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
    config.put(ProducerConfig.ACKS_CONFIG, "all");
    config.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
    config.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
    return new DefaultKafkaProducerFactory<>(config);
}
```

### Scenario 2: Consumer Lag Causing Processing Delays
**Context:** A consumer group processes orders from the `order.created` topic. During a flash sale, 1M orders are published in 1 minute. The consumer processes 5,000 msg/s per partition. With 3 partitions, throughput is 15,000 msg/s — far below the 16,666 msg/s incoming rate. Consumer lag grows by 1,666 msg/s. After 5 minutes, lag reaches 500,000 messages. Order confirmations are delayed by 30+ minutes.

**Resolution:** (1) Increase partitions to match desired throughput: target = 16,666 / 5,000 = ~4 partitions. (2) Increase consumer concurrency to match partition count: `@KafkaListener(concurrency = "4")`. (3) Add consumer instances to the same group (up to the partition count). (4) Optimize processing: batch consumption with `max.poll.records=1000`, use async processing, batch database writes. (5) Monitor lag with Kafka Lag Exporter and auto-scale consumers.

### Scenario 3: Message Ordering Across Partitions
**Context:** An e-commerce system publishes order events to Kafka for processing. A single order (order ID: "ORD-12345") goes through multiple lifecycle events: `OrderCreated`, `PaymentProcessed`, `OrderShipped`. These events end up in different partitions (because the default partitioner hashes the key, but different lifecycle stages use different keys). The consumer receives events out of order — `OrderShipped` arrives before `OrderCreated`. The order processing logic fails because it tries to ship an order that doesn't exist yet.

**Resolution:** Use a consistent key for all events belonging to the same order. By setting the message key to the order ID, all events for the same order route to the same partition. Within a partition, Kafka guarantees order. The consumer processes events for each order sequentially.

```java
// All order lifecycle events use orderId as the key
kafkaTemplate.send("order.events", event.getOrderId(), event);
// This ensures OrderCreated, PaymentProcessed, OrderShipped
// all go to the same partition and are processed in order
```

## Use Cases

- **Event sourcing and audit logging** — order lifecycle events, user activity streams, or financial transactions
  - Append-only log provides an immutable record. Consumers replay events from any point in time. Retention policies enable both real-time and batch consumption.
  - **Avoid when:** message throughput is low (<100 msg/sec) — Kafka's batching and commit-log design adds overhead compared to simpler queues.

- **Stream processing** — real-time fraud detection, anomaly monitoring, or data enrichment pipelines
  - Kafka Streams or ksqlDB processes records within Kafka without external systems. Stateful operations (joins, aggregations, windowing) run in the stream processor.
  - **Avoid when:** processing logic is complex and requires external coordination — use Kafka Connect to bridge to a stream processor (Flink, Spark).

- **Data pipeline between systems** — database CDC to search index, data warehouse ingestion, or cache refresh
  - Kafka Connect ingests changes from databases (Debezium CDC) and streams them to Elasticsearch, S3, or data warehouses. Exactly-once semantics prevent data loss or duplication.
  - **Avoid when:** the pipeline is simple point-to-point — a direct API integration or lightweight queue may be simpler to operate.

- **Log aggregation and metrics** — centralizing application logs, infrastructure metrics, or tracing data
  - Producers emit structured log/metric records to Kafka topics. Consumers index them into Elasticsearch, Loki, or a time-series database for querying.
  - **Avoid when:** log volume is low and retention requirements are minimal — direct shipping to Elasticsearch or Loki is simpler.

- **Pub/Sub for large-scale event distribution** — notifications, webhook delivery, or feature flag updates
  - Topics with consumer groups allow multiple independent subscribers. Each consumer group gets every message, enabling fan-out at scale.
  - **Avoid when:** messages need routing based on content — RabbitMQ's exchange bindings are more flexible for complex routing topologies.

---

## Scenario-Based Questions

1. **Q: Your real-time fraud detection system uses Kafka Streams. Transactions are consumed from a topic, processed, and results are output. Under load, the system processes a transaction, then a few seconds later receives a cancellation for the same transaction. The fraud detection should not flag cancelled transactions. How do you design this with Kafka Streams?**
   - A: (1) Use a KStream for transactions and a separate KStream for cancellations. (2) Use a KTable (backed by a compacted topic) that stores transaction statuses. (3) Join the transaction stream with the status KTable to filter out cancelled transactions. (4) Use event-time processing with a 1-hour window to handle late-arriving cancellations. (5) Output only confirmed (non-cancelled) fraud alerts.

2. **Q: Your Kafka consumer processes payment events and writes results to a database. After a crash, some events are processed twice — the database has duplicate payment records. How do you implement exactly-once semantics between Kafka and your database?**
    - A: (1) Enable idempotent producer on the write side. (2) Use Kafka transactions: consume with `isolation.level=read_committed`, process the event, and write the result to DB within a Kafka transaction. (3) Commit the Kafka offset only after the database transaction commits — use a transactional outbox pattern: write the result to DB and the offset to a separate table in the same DB transaction. (4) Use idempotent upserts (`INSERT ... ON CONFLICT DO UPDATE`) in the database — the same event processed twice produces the same result.
    - **Interview follow-up:** Kafka transactions keep the offset and the result in a single atomic commit across producer and consumer — but the database write happens outside Kafka's control. How do you make the DB write and the offset commit truly atomic?

3. **Q: Your topic has 6 partitions and 3 consumers in a group. One consumer crashes. How does Kafka handle the rebalance, and how do you minimize the impact on processing?**
   - A: (1) The group coordinator detects the consumer's session timeout (default 45s). (2) A rebalance triggers: all consumers stop processing, surrender their partitions, and the group leader reassigns partitions (sticky strategy assigns the 6 partitions to the 2 remaining consumers — 3 partitions each). (3) To minimize impact: use `session.timeout.ms=10s` for faster failure detection, use static group membership (`group.instance.id`) to avoid full rebalances, use cooperative sticky rebalancer (incremental rebalancing that doesn't stop all consumers).

4. **Q: Your Kafka cluster has 3 brokers. A broker fails and all partitions on that broker become unavailable. The topic has replication factor 3. What happens?**
   - A: (1) The controller detects the broker failure via ZooKeeper/KRaft session timeout. (2) For each partition that had its leader on the failed broker, a new leader is elected from the in-sync replicas (ISR) on the remaining brokers. (3) With replication factor 3, each partition has 2 surviving replicas — one becomes the new leader. (4) Producers and consumers are redirected to the new leader. (5) No data loss if `min.insync.replicas=2` and `acks=all`. (6) The cluster continues operating with reduced replication until the broker recovers or replacement comes online.

5. **Q: Your Kafka producer sends 10,000 messages/second. Each message is a large JSON payload (500KB). The broker's default message size is 1MB. Some messages exceed 1MB and are rejected. How do you handle this?**
   - A: (1) Increase `max.message.bytes` on the broker (e.g., 5MB), `max.request.size` on the producer, and `max.partition.fetch.bytes` on the consumer. (2) Compress the payload: enable `compression.type=snappy` or `zstd` — JSON compresses well (often 5-10x). (3) Better approach: store large payloads in external storage (S3, HDFS) and send only the reference (path + metadata) in Kafka. This keeps Kafka's message size small and avoids large payloads in the broker's page cache. (4) For streams, consider splitting large messages into chunks with a reassembly protocol.

6. **Q: You need to orchestrate a saga for order processing: Reserve Inventory → Process Payment → Ship Order. If Payment fails, you must cancel Inventory. How do you implement this with Kafka?**
   - A: (1) Create a command topic per saga step (`order.inventory.command`, `order.payment.command`). (2) Create a reply topic for responses (`order.saga.reply`). (3) The orchestrator publishes "ReserveInventory" to the command topic. (4) Inventory service processes, publishes a reply (success/failure) to the reply topic. (5) On success, orchestrator publishes "ProcessPayment". On failure, publishes "CancelInventory" compensation. (6) Use a transactional producer to atomically emit the next command or compensation — prevents partial saga state. (7) Use a KTable to track saga state (keyed by order ID) for recovery after crashes.

7. **Q: Your consumer group processes order events. One consumer takes 15 minutes to process a single message (PDF generation). The session timeout is 45 seconds. The consumer is kicked out of the group during processing. How do you handle long-running processing without triggering rebalances?**
    - A: (1) Increase `max.poll.interval.ms` to 20 minutes — allows the consumer to take longer between polls. (2) Use manual offset commits: commit the offset before the long-running process (at-least-once). (3) Offload the long-running work to a separate thread/executor — the consumer thread continues polling to send heartbeats and avoid being considered dead. (4) Use pause/resume: `consumer.pause(partition)` before the long task, `consumer.resume(partition)` after. (5) Increase `heartbeat.interval.ms` to detect actual failures sooner while accommodating long processing.
    - **Interview follow-up:** If you commit the offset before the long process and the consumer crashes mid-way, the message is lost — how do you reconcile at-least-once vs exactly-once when the processing can't be split into poll-sized chunks?

8. **Q: You need geographic routing: orders from EU go to EU consumers, orders from US go to US consumers. How do you design the Kafka partitioner?**
   - A: (1) Implement a custom `Partitioner` interface: extract the region from the message key or value (e.g., from a `region` field or a key prefix like `eu-order-123`). (2) Assign specific partition ranges for each region: EU = partitions 0-3, US = partitions 4-7, APAC = partitions 8-11. (3) Hash the region to its partition range. (4) Configure: `partitioner.class=com.example.RegionPartitioner`. (5) Consumer groups are also region-specific: EU consumers subscribe to partitions 0-3 only. This ensures data locality and reduces cross-region network traffic.

9. **Q: Your Kafka cluster spans two datacenters (primary DC and DR DC). You need to replicate a critical topic from primary to DR with <10 second lag. How do you configure this?**
   - A: (1) Use MirrorMaker 2 (MM2) configured in active-passive mode. (2) MM2 runs connectors on the DR cluster that consume from the primary cluster's topic and produce to the DR topic. (3) Set `replication.factor=3` in primary, `replication.factor=2` in DR (fewer brokers). (4) Tune for low latency: `consumer.poll.timeout.ms=100`, `producer.acks=1`, `producer.compression.type=zstd`. (5) Monitor lag via MM2's `Checkpoint` topics — alerts if lag exceeds 10 seconds. (6) For active-active, use bidirectional replication with separate topics per DC (e.g., `orders_us_east`, `orders_us_west`) to avoid conflicts.

10. **Q: Your banking application uses Kafka for event sourcing. Each account's state changes are stored as events. To rebuild the account state, you must replay all events from the beginning. After 6 months, replay takes 30 minutes. How do you speed up state rebuilding?**
    - A: (1) Use Kafka's log compaction: keep only the latest state per key (account ID). Compacted topics delete old events and retain only the latest value for each key. (2) Store periodic snapshots: persist the account state in a database every 1000 events. On restart, load the latest snapshot and replay only events newer than the snapshot. (3) Use Kafka Streams' state stores (RocksDB) that persist state locally. On restart, the state store is recovered from the local RocksDB instance (fast) rather than replaying all events. (4) Partition by account ID: each partition handles a subset of accounts, enabling parallel replay.
    - **Interview follow-up:** Log compaction retains only the latest value per key — what happens to the ordering guarantees if a consumer reads a compacted topic where intermediate events were deleted? Can the consumer still reconstruct state correctly?

---

## Interview Questions

1. **What is Apache Kafka and what are its core components?**
   - A: A distributed event streaming platform based on a commit log. Core components: Broker (server), Topic (message category), Partition (ordered log), Producer (writes), Consumer (reads), Consumer Group (load-balanced consumers), ZooKeeper/KRaft (coordination).

2. **What is the difference between Kafka and traditional message brokers (RabbitMQ)?**
   - A: Kafka is pull-based with an append-only log — consumers poll for data. It persists messages on disk for replay. RabbitMQ is push-based with exchange routing. Kafka excels at high-throughput streaming and event sourcing; RabbitMQ excels at complex routing and RPC patterns.

3. **How does Kafka achieve high throughput?**
   - A: Sequential disk I/O (append-only log), zero-copy data transfer (disk → network without application memory), batching (producer batches, consumer fetches batches), partitioning (parallelism), compression (snappy, zstd), and page cache usage.

4. **What is a consumer group and how does it work?**
   - A: A consumer group divides topic partitions among its members. Each partition is consumed by exactly one consumer in the group. If a consumer fails, partitions rebalance to remaining members. This enables horizontal scaling of consumption.

5. **How do you guarantee message ordering in Kafka?**
   - A: Within a partition, messages are ordered. Use a consistent message key (e.g., order ID) so related messages go to the same partition. Different partitions have no ordering guarantees. For total order, use a single partition (limits throughput).

6. **What are idempotent producers and when should you use them?**
   - A: Idempotent producers (enable.idempotence=true) prevent duplicate messages from producer retries. The broker deduplicates by (producer_id, sequence_number). Use for exactly-once semantics. Required for exactly-once delivery.

7. **How does consumer rebalancing work?**
   - A: When a consumer joins/leaves, the group coordinator triggers a rebalance. Consumers stop processing, surrender partitions, and the leader reassigns partitions using range, round-robin, or sticky strategy. Sticky minimizes unnecessary partition movement.

8. **What is the difference between at-least-once, at-most-once, and exactly-once semantics in Kafka?**
   - A: At-least-once: offset committed after processing (risk of duplicate). At-most-once: offset committed before processing (risk of data loss). Exactly-once: idempotent producer + transactions + read_committed consumer. Most Kafka systems use at-least-once with idempotent sinks.

9. **How do you choose the number of partitions for a topic?**
   - A: Partitions = max(throughput_target / per_partition_throughput, max_consumers_needed). Rule of thumb: 10-50 partitions per broker. Too few limits parallelism; too many increases rebalance time and file descriptor usage.

10. **How do you monitor Kafka?**
    - A: Consumer lag (difference between latest offset and committed offset), request rates (produce/fetch), disk usage, network throughput, under-replicated partitions, broker health, ZooKeeper/KRaft status. Tools: Confluent Control Center, Kafka Lag Exporter, Prometheus + Grafana.

---

## Developer Recommendations

- **Always enable idempotent producers** — Without `enable.idempotence=true`, producer retries during transient errors can create duplicate messages. The performance cost of idempotency is negligible (single-digit percentage overhead), and the safety gain is enormous — it eliminates an entire class of data integrity bugs. Set `acks=all` and `enable.idempotence=true` on every producer. This is the single highest-impact configuration change you can make.
  - **Production story:** A team once ran a payment processing pipeline without idempotent producers — a brief network blip caused 2,000 duplicate payment events, double-charging customers before the issue was caught.

- **Use a consistent message key for related events** — Kafka guarantees order only within a partition. If events for the same entity (order, user, account) use different keys, they land in different partitions and are processed out of order. Always use the entity ID as the message key. This ensures all events for an entity go to the same partition and are processed sequentially. The trade-off: uneven partition load if some entities have more events. Acceptable for most use cases.

- **Monitor consumer lag as a primary operational metric** — Consumer lag is the distance between the latest produced offset and the last committed offset. Rising lag means consumers are falling behind. Set alerts on lag thresholds (e.g., >10,000 messages). Investigate lag by checking: consumer processing time, partition count, network bottlenecks, and downstream dependencies (database, external API). Auto-scale consumers when lag exceeds a threshold.

- **Configure appropriate replication factor and min.insync.replicas** — Replication factor 3 is the standard: tolerates one broker failure without data loss. Set `min.insync.replicas=2` with `acks=all` — the producer waits for at least 2 replicas to acknowledge. This prevents the "last write" from being lost if the leader fails immediately after acknowledging. The trade-off: higher latency for each produce request (wait for 2 acks). Worth it for data durability.

- **Use a dead letter topic for poison messages** — A message that cannot be processed (malformed, unexpected schema, business rule violation) will cause the consumer to retry, fail, retry, fail indefinitely if not handled. Implement a dead letter topic: after N retries (typically 3), publish the failed message and original error to a `dead-letter` topic. The consumer acknowledges the original message and continues processing. This prevents a single bad message from blocking the entire partition.
  - **Production story:** In one incident, a schema compatibility change between services caused every event from a deprecated field to fail deserialization — without a DLQ, the consumer stalled for 6 hours while the teams debugged; with a DLQ, it would have been a 10-minute replay.

- **Design topics for event sourcing with log compaction** — For event-sourced systems, use Kafka's log compaction (`cleanup.policy=compact`). Compaction keeps only the latest state per key, allowing new consumers to rebuild state by reading only the compacted topic (which is much smaller than the full event log). Combine with periodic snapshots stored externally for fast recovery. The trade-off: event history is lost after compaction — use a separate "append-only" topic if you need full audit history.
