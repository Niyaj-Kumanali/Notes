# Apache Kafka

## 1. Executive Summary

Apache Kafka is a distributed event streaming platform capable of handling trillions of events per day. Originally developed by LinkedIn and open-sourced in 2011, Kafka provides a high-throughput, fault-tolerant, and scalable publish-subscribe messaging system. It is built on a distributed commit log architecture, where messages are persisted to disk and replicated across multiple brokers. Kafka is widely used for event streaming, data pipelines, stream processing, log aggregation, and real-time analytics.

## 2. Core Theory

### Kafka Architecture

- **Broker**: A Kafka server that stores data and serves clients.
- **Topic**: A category/feed to which messages are published.
- **Partition**: A topic is split into partitions for parallelism and scaling.
- **Producer**: Publishes messages to topics.
- **Consumer**: Subscribes to topics and processes messages.
- **Consumer Group**: A group of consumers that jointly consume a topic.
- **Offset**: A unique identifier for each message within a partition.
- **ZooKeeper/KRaft**: Coordinates the cluster (ZooKeeper legacy, KRaft newer).

### Key Concepts

```
Topic
  |-- Partition 0 (ordered, immutable sequence of records)
  |       |-- Record 0 (offset 0)
  |       |-- Record 1 (offset 1)
  |       |-- Record 2 (offset 2)
  |-- Partition 1
  |       |-- Record 0
  |       |-- Record 1
  |-- Partition 2 (replicated to other brokers)
```

### Retention Policies

- **Time-based**: Delete messages older than X days/hours.
- **Size-based**: Delete oldest messages when topic exceeds Y bytes.
- **Compact**: Keep the latest message for each key (log compaction).

### Consumer Group Semantics

```
Topic (4 partitions)
  Consumer Group "orders-group"
    Consumer-1: partitions 0, 1
    Consumer-2: partitions 2, 3

Each message goes to exactly one consumer in the group.
If a consumer fails, partitions are rebalanced to remaining consumers.
```

## 3. Under-the-Hood Deep Dive

### Storage Architecture

Kafka's performance comes from its efficient storage design:

- **Commit Log**: Appending-only, sequential I/O (very fast on disks).
- **Zero Copy**: Data is sent from disk to network buffer without copying to application memory.
- **Batching**: Producers batch messages for efficiency.
- **Compression**: Messages are compressed (gzip, snappy, lz4, zstd).
- **Index Files**: Separate index files for fast offset lookups.

### Partition Leader and Follower

Each partition has one leader and N followers:
- **Leader**: Handles all read/write requests for the partition.
- **Followers**: Replicate data from the leader (in-sync replicas - ISR).
- If the leader fails, an ISR follower becomes the new leader.

### Producer Request Flow

1. Producer sends batch to partition leader.
2. Leader appends to its local log.
3. Based on `acks` setting:
   - `acks=0`: Fire and forget (fastest, may lose data).
   - `acks=1`: Leader acknowledges (default, moderate durability).
   - `acks=all` or `acks=-1`: All ISR replicas acknowledge (strongest durability).
4. Leader sends response with offset.

### Consumer Poll Loop

```java
while (true) {
    ConsumerRecords<String, Order> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, Order> record : records) {
        processOrder(record.value());
    }
    consumer.commitAsync();  // or commitSync()
}
```

## 4. Production Code Examples

### Spring Boot Kafka Configuration

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

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
      retries: 3
    consumer:
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      group-id: order-service
      auto-offset-reset: earliest
      enable-auto-commit: false
      max-poll-records: 500
      properties:
        spring.json.trusted.packages: "*"
    listener:
      concurrency: 3
      ack-mode: manual_immediate
```

### Producer

```java
@Service
public class OrderEventProducer {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public OrderEventProducer(KafkaTemplate<String, OrderEvent> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void sendOrderCreated(OrderEvent event) {
        String key = event.getOrderId();
        ListenableFuture<SendResult<String, OrderEvent>> future =
            kafkaTemplate.send("order.created", key, event);

        future.whenComplete((result, ex) -> {
            if (ex == null) {
                log.info("Order event sent: topic={}, partition={}, offset={}",
                    result.getRecordMetadata().topic(),
                    result.getRecordMetadata().partition(),
                    result.getRecordMetadata().offset());
            } else {
                log.error("Failed to send order event: {}", event.getOrderId(), ex);
                // Implement retry or DLQ logic
                sendToDeadLetter(event, ex);
            }
        });
    }

    public void sendBulkOrders(List<OrderEvent> events) {
        List<ListenableFuture<SendResult<String, OrderEvent>>> futures = events.stream()
            .map(event -> kafkaTemplate.send("order.bulk", event.getOrderId(), event))
            .collect(Collectors.toList());

        CompletableFuture.allOf(
            futures.stream()
                .map(ListenableFuture::completable)
                .toArray(CompletableFuture[]::new)
        ).join();
    }
}
```

### Consumer with Manual Acknowledgment

```java
@Component
public class OrderEventConsumer {

    private final OrderService orderService;

    public OrderEventConsumer(OrderService orderService) {
        this.orderService = orderService;
    }

    @KafkaListener(topics = "order.created", concurrency = "3")
    public void handleOrderCreated(
            ConsumerRecord<String, OrderEvent> record,
            Acknowledgment acknowledgment) {

        OrderEvent event = record.value();
        log.info("Received order event: key={}, partition={}, offset={}",
            record.key(), record.partition(), record.offset());

        try {
            orderService.processNewOrder(event);
            acknowledgment.acknowledge();
        } catch (Exception e) {
            log.error("Failed to process order: {}", event.getOrderId(), e);
            // Depending on strategy, skip or retry
            if (record.headers().lastHeader("retry-count") != null) {
                int retryCount = Integer.parseInt(
                    new String(record.headers().lastHeader("retry-count").value()));
                if (retryCount >= 3) {
                    // Send to DLQ
                    deadLetterProducer.send(event, e);
                    acknowledgment.acknowledge(); // Ack to skip
                    return;
                }
            }
            // Nack to retry
            acknowledgment.nack(1000); // retry after 1 second
        }
    }
}
```

### Batch Consumer

```java
@Component
public class BatchOrderConsumer {

    @KafkaListener(topics = "order.bulk", batch = "true")
    public void handleBatch(List<ConsumerRecord<String, OrderEvent>> records,
                           Acknowledgment acknowledgment) {

        List<OrderEvent> events = records.stream()
            .map(ConsumerRecord::value)
            .collect(Collectors.toList());

        log.info("Processing batch of {} orders", events.size());

        try {
            orderService.batchProcessOrders(events);
            acknowledgment.acknowledge();
        } catch (Exception e) {
            log.error("Batch processing failed, nacking all", e);
            acknowledgment.nack(5000);
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
    public KStream<String, OrderEvent> orderStream(
            StreamsBuilder builder) {

        KStream<String, OrderEvent> source = builder
            .stream("order.created",
                Consumed.with(Serdes.String(), new JsonSerde<>(OrderEvent.class)));

        // Filter high-value orders
        KStream<String, OrderEvent> highValueOrders = source
            .filter((key, event) -> event.getAmount().compareTo(new BigDecimal("1000")) > 0);

        highValueOrders.to("order.high-value",
            Produced.with(Serdes.String(), new JsonSerde<>(OrderEvent.class)));

        // Count orders by status
        KTable<String, Long> orderCounts = source
            .groupBy((key, event) -> event.getStatus())
            .count(Materialized.as("order-status-counts"));

        orderCounts.toStream()
            .to("order.status-counts",
                Produced.with(Serdes.String(), Serdes.Long()));

        // Enrich orders with user data
        KTable<String, User> userTable = builder
            .table("users",
                Consumed.with(Serdes.String(), new JsonSerde<>(User.class)));

        KStream<String, EnrichedOrder> enrichedOrders = source
            .join(userTable,
                (order, user) -> new EnrichedOrder(order, user),
                Joined.with(Serdes.String(),
                    new JsonSerde<>(OrderEvent.class),
                    new JsonSerde<>(User.class)));

        enrichedOrders.to("order.enriched",
            Produced.with(Serdes.String(), new JsonSerde<>(EnrichedOrder.class)));

        return source;
    }
}
```

### Transactional Producer

```java
@Service
public class TransactionalOrderProducer {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @Transactional
    public void sendOrderWithTransaction(OrderEvent event) {
        kafkaTemplate.send("order.validated", event.getOrderId(), event);
        kafkaTemplate.send("order.audit", event.getOrderId(), event);
        kafkaTemplate.send("notification.trigger", event.getOrderId(),
            new NotificationEvent(event.getOrderId(), "order_created"));
        // All messages are committed atomically
    }
}
```

### Idempotent Producer

```java
@Configuration
public class IdempotentProducerConfig {

    @Bean
    public ProducerFactory<String, OrderEvent> idempotentProducerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        props.put(ProducerConfig.ACKS_CONFIG, "all");
        props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
        return new DefaultKafkaProducerFactory<>(props);
    }
}
```

### Custom Partitioner

```java
@Component
public class OrderPartitioner implements Partitioner {

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);

        // Route orders from the same customer to the same partition
        String orderKey = (String) key;
        int customerHash = orderKey.split("-")[0].hashCode();
        return Math.abs(customerHash) % partitions.size();
    }

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> configs) {}
}
```

### Kafka Admin Client

```java
@Component
public class KafkaAdminService {

    private final KafkaAdmin kafkaAdmin;

    public KafkaAdminService(KafkaAdmin kafkaAdmin) {
        this.kafkaAdmin = kafkaAdmin;
    }

    public void createTopic(String name, int partitions, short replicationFactor) {
        AdminClient admin = AdminClient.create(kafkaAdmin.getConfigurationProperties());

        NewTopic topic = new NewTopic(name, partitions, replicationFactor);
        topic.configs(Map.of(
            "retention.ms", "604800000",        // 7 days
            "cleanup.policy", "delete",
            "compression.type", "snappy"
        ));

        CreateTopicsResult result = admin.createTopics(List.of(topic));
        result.all().whenComplete((v, ex) -> {
            if (ex == null) {
                log.info("Topic created: {}", name);
            } else {
                log.error("Failed to create topic: {}", name, ex);
            }
        });
    }

    public void describeTopic(String name) {
        AdminClient admin = AdminClient.create(kafkaAdmin.getConfigurationProperties());

        DescribeTopicsResult result = admin.describeTopics(List.of(name));
        result.all().whenComplete((topics, ex) -> {
            if (ex == null) {
                TopicDescription description = topics.get(name);
                log.info("Topic: {}, partitions: {}, replication: {}",
                    name,
                    description.partitions().size(),
                    description.partitions().get(0).replicas().size());
            }
        });
    }
}
```

### Error Handling with Dead Letter Topic

```java
@Component
public class DeadLetterProducer {

    private final KafkaTemplate<String, DeadLetterRecord> kafkaTemplate;

    public DeadLetterProducer(KafkaTemplate<String, DeadLetterRecord> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void send(String originalTopic, Object failedMessage, Exception cause) {
        DeadLetterRecord record = new DeadLetterRecord(
            originalTopic,
            failedMessage,
            cause.getMessage(),
            Instant.now().toString()
        );
        kafkaTemplate.send("dead-letter", record);
    }
}

public record DeadLetterRecord(
    String originalTopic,
    Object originalMessage,
    String errorMessage,
    String timestamp
) {}
```

## 5. Real-World Scenarios

### Scenario 1: Event-Driven Microservices

```
Order Service --(order.created)--> Kafka --> Payment Service
                                         --> Inventory Service
                                         --> Notification Service
                                         --> Analytics Service
                                         --> Shipping Service
```

### Scenario 2: CDC (Change Data Capture) with Debezium

```yaml
debezium:
  connector:
    database:
      hostname: localhost
      port: 5432
      user: debezium
      password: debezium
      dbname: orders_db
      server.name: orders-server
      schema.include.list: public
      table.include.list: public.orders,public.order_items
      plugin.name: pgoutput
  topic:
    prefix: cdc
```

### Scenario 3: Clickstream Analytics Pipeline

```
Web App --(page.views)--> Kafka --> Flink/KSQL --> Real-Time Dashboard
                                  --> Hadoop --> Historical Analytics
                                  --> Alerting --> Anomaly Detection
```

### Scenario 4: Log Aggregation

```java
@Component
public class LogProducer {

    private final KafkaTemplate<String, String> kafkaTemplate;

    public void sendLog(String serviceName, String level, String message) {
        String key = serviceName;
        String value = String.format(
            "{\"timestamp\":\"%s\",\"service\":\"%s\",\"level\":\"%s\",\"message\":\"%s\"}",
            Instant.now(), serviceName, level, message);
        kafkaTemplate.send("application-logs", key, value);
    }
}
```

## 6. Performance

### Performance Tuning

```yaml
# Producer tuning
spring.kafka.producer:
  batch-size: 65536        # 64KB batch size
  linger-ms: 10            # Wait up to 10ms to batch
  compression-type: zstd   # Best compression ratio
  acks: all                # Strongest durability
  buffer-memory: 33554432  # 32MB buffer
  max-request-size: 1048576 # 1MB max request

# Consumer tuning
spring.kafka.consumer:
  fetch-min-bytes: 1024      # Minimum bytes per fetch
  fetch-max-wait-ms: 500     # Max wait for fetch
  max-poll-records: 1000     # Max records per poll
  max-partition-fetch-bytes: 1048576 # 1MB per partition

# Broker tuning (server.properties)
num.network.threads: 8
num.io.threads: 16
log.segment.bytes: 1073741824   # 1GB segments
log.retention.hours: 168        # 7 days
log.cleaner.enable: true
```

### Benchmarking

```
Single Broker:
  Producer: ~1M messages/sec (small messages)
  Consumer: ~3M messages/sec

3 Broker Cluster:
  Producer: ~3M messages/sec
  Consumer: ~9M messages/sec
  (Linearly scalable with partitions and brokers)
```

## 7. Security

### SSL/TLS Configuration

```yaml
spring:
  kafka:
    ssl:
      trust-store-location: classpath:kafka.truststore.jks
      trust-store-password: changeit
      key-store-location: classpath:kafka.keystore.jks
      key-store-password: changeit
    properties:
      security.protocol: SSL
      ssl.endpoint.identification.algorithm: https
```

### SASL Authentication

```yaml
spring:
  kafka:
    properties:
      security.protocol: SASL_SSL
      sasl.mechanism: SCRAM-SHA-512
      sasl.jaas.config: >
        org.apache.kafka.common.security.scram.ScramLoginModule required
        username="admin"
        password="admin-secret";
```

### ACL Management

```java
@Component
public class KafkaAclService {

    public void addReadAcl(String principal, String topic) {
        try (AdminClient admin = AdminClient.create(config)) {
            AclBinding aclBinding = new AclBinding(
                new ResourcePattern(ResourceType.TOPIC, topic, PatternType.LITERAL),
                new AccessControlEntry(principal, "*",
                    AclOperation.READ, AclPermissionType.ALLOW));
            admin.createAcls(List.of(aclBinding));
        }
    }
}
```

## 8. Common Mistakes

- **Too few partitions**: Limits parallelism and throughput.
- **Too many partitions**: Increases overhead and rebalance time.
- **Consumer lag not monitored**: Can silently lose data processing capability.
- **No idempotent producer**: Duplicate messages in failure scenarios.
- **Auto offset commit enabled**: Risk of missing messages on crash.
- **Synchronous production in hot path**: Blocks and reduces throughput.
- **Ignoring message size limits**: Kafka has a default 1MB message limit.
- **No dead letter topic**: Failed messages are lost if processing fails.
- **Incorrect partitioning strategy**: Uneven load across consumers.
- **Rebalance storms**: Frequent rebalances can halt processing.

## 9. Senior Engineer Perspective

### Kafka in Production

- **Cluster Sizing**: 3 brokers minimum, 5 for production, odd number for quorum.
- **Topic Design**: 3x replication factor, partitions = desired throughput / partition throughput.
- **Partition Count**: Aim for 10-50 partitions per broker.
- **Monitoring**: Consumer lag, request rates, disk usage, network throughput.
- **Disaster Recovery**: MirrorMaker 2 for cross-cluster replication.

### Exactly-Once Semantics

Kafka provides exactly-once semantics (EOS) through:
1. Idempotent producer (enable.idempotence=true).
2. Transactional producer/consumer.
3. Kafka Streams exactly-once processing.
4. But exactly-once output to external systems requires idempotent sinks.

### Schema Registry

```yaml
spring:
  kafka:
    properties:
      schema.registry.url: http://localhost:8081
      value.subject.name.strategy: io.confluent.kafka.serializers.subject.TopicRecordNameStrategy
```

## 10. Interview Questions (Easy)

1. What is Apache Kafka and what is it used for?
2. What is a Kafka topic and partition?
3. What is a Kafka broker?
4. What is a consumer group?
5. What is the role of ZooKeeper/KRaft in Kafka?
6. What is a Kafka offset?
7. What is the difference between a topic and a partition?
8. What are Kafka producers and consumers?
9. What is message retention in Kafka?
10. What is a Kafka cluster?

## Medium

1. How does Kafka achieve high throughput?
2. What is the difference between Kafka and traditional message queues?
3. How does consumer group rebalancing work?
4. What are the different acks settings in Kafka producers?
5. What is log compaction in Kafka?
6. How does partitioning affect parallelism?
7. What is the difference between at-least-once and exactly-once semantics?
8. How do you handle consumer lag?
9. What is Kafka Streams?
10. How do you choose the number of partitions for a topic?

## 11. Advanced Interview Questions (Hard)

1. How would you implement exactly-once semantics across Kafka and an external database?
2. Design a Kafka-based saga orchestration for distributed transactions.
3. How do you handle large messages (>1MB) in Kafka?
4. Implement a custom partitioner for geographic routing.
5. How do you perform multi-datacenter replication with Kafka?
6. Design a Kafka Connect connector for a custom data source.
7. How do you handle schema evolution with Avro and Schema Registry?
8. Implement a dead letter topic with automatic retry and escalation.
9. How do you perform stateful stream processing with Kafka Streams?
10. Design a system for exactly-once semantics with idempotent Kafka consumers.

## System Design

1. Design a real-time fraud detection system using Kafka Streams.
2. Design a Kafka-based event sourcing system for a banking application.
3. Design a real-time analytics pipeline using Kafka and Flink.
4. Design a change data capture (CDC) pipeline using Kafka Connect and Debezium.
5. Design a multi-region Kafka deployment with disaster recovery.
6. Design a Kafka-based microservices event bus.
7. Design a real-time notification system using Kafka.
8. Design a log aggregation and monitoring system with Kafka.
9. Design a clickstream data pipeline for a large e-commerce platform.
10. Design a Kafka-based data lake ingestion pipeline.

## 12. Expert-Level Interview Questions (Architect-Level)

1. Design a global-scale Kafka platform serving 10,000+ topics across 200+ microservices with multi-tenant isolation, quota management, and SLA guarantees.
2. How would you implement a Kafka cluster with automatic partition rebalancing that minimizes downtime during broker failures across multiple availability zones?
3. Design a schema evolution system for Kafka that supports both Avro and Protobuf, with automatic compatibility checking and field-level data masking.
4. How would you build a Kafka-based command and event store for a CQRS system with support for event sourcing, snapshots, and event replay at scale?
5. Design a Kafka monitoring and auto-remediation system that detects anomalies (consumer lag spikes, broker failures, disk space) and takes automated action.
6. How would you implement a Kafka tiered storage system that transparently moves older data to cheaper object storage while maintaining queryability?
7. Design a multi-tenant Kafka platform where each tenant has guaranteed throughput, storage isolation, and independent consumer group management.
8. How would you build a Kafka-based distributed transaction coordinator that supports the saga pattern with automatic compensation and recovery?
9. Design a Kafka migration strategy from an on-premise cluster to a managed cloud service with zero downtime and exactly-once guarantees.
10. How would you implement a Kafka-powered real-time data mesh that enables domain teams to publish and consume data products autonomously?

## 13. Debugging & Troubleshooting

### Common Issues

- **Consumer lag high**: Increase partitions, add consumers, optimize processing.
- **Rebalance storms**: Check session.timeout.ms, heartbeat.interval.ms settings.
- **Leader not available**: Check broker health, ZooKeeper/KRaft connectivity.
- **OffsetOutOfRangeException**: Reset auto.offset.reset or seek to valid offset.
- **RecordTooLargeException**: Increase max.message.bytes and max.request.size.
- **Network timeouts**: Check firewall, broker reachability, DNS resolution.

### Debugging Commands

```bash
# List topics
kafka-topics --bootstrap-server localhost:9092 --list

# Describe topic
kafka-topics --bootstrap-server localhost:9092 --describe --topic order.created

# Get offset details
kafka-get-offsets --bootstrap-server localhost:9092 --topic order.created

# Consume messages (debugging)
kafka-console-consumer --bootstrap-server localhost:9092 --topic order.created --from-beginning

# Check consumer group status
kafka-consumer-groups --bootstrap-server localhost:9092 --group order-service --describe
```

### Consumer Lag Monitoring

```java
@Component
public class LagMonitor {

    private final KafkaAdmin kafkaAdmin;

    @Scheduled(fixedRate = 30000)
    public void checkLag() {
        try (AdminClient admin = AdminClient.create(
                kafkaAdmin.getConfigurationProperties())) {

            ListConsumerGroupOffsetsResult offsets = admin
                .listConsumerGroupOffsets("order-service");

            offsets.partitionsToOffsetAndMetadata().get()
                .forEach((partition, metadata) -> {
                    long lag = metadata.offset();
                    log.info("Partition {} lag: {}", partition, lag);
                });
        } catch (Exception e) {
            log.error("Failed to check consumer lag", e);
        }
    }
}
```

## 14. Comparison Section

### Kafka vs RabbitMQ

| Aspect | Kafka | RabbitMQ |
|--------|-------|----------|
| Architecture | Distributed commit log | AMQP broker with exchanges |
| Message Model | Pull-based, replayable | Push/Pull, destructive read |
| Performance | ~1M msg/sec | ~50K msg/sec |
| Persistence | Durable by default (disk) | Optional persistence |
| Message Ordering | Ordered per partition | Ordered per queue |
| Routing | Topic-based | Flexible (headers, topic, direct, fanout) |
| Retention | Time/size/compact based | Deleted after ACK |
| Use Case | Event streaming, analytics | Task queues, RPC |

### Kafka vs Pulsar

| Aspect | Kafka | Pulsar |
|--------|-------|--------|
| Storage | Coupled (broker + storage) | Decoupled (broker + bookies) |
| Geo-Replication | MirrorMaker | Built-in |
| Multi-Tenancy | Topic naming convention | Native multi-tenant |
| Message TTL | Topic-level only | Per-message TTL |
| Performance | Higher for local | Better for geo-distributed |

## 15. Revision Notes

- Kafka = distributed commit log, pub-sub, event streaming
- Topic -> Partitions -> Segments -> Records
- Producer: key-based partitioning, acks (0, 1, all), batching, compression
- Consumer: poll-based, manual commit, consumer groups, rebalance
- Retention: time, size, compact
- Exactly-once: idempotent producer + transactions + idempotent consumer
- Kafka Streams: stream processing within application
- Connect: source/sink connectors for external systems
- Schema Registry: Avro/Protobuf schema management
- KRaft: ZooKeeper elimination (KIP-500)

## 16. Cheat Sheet

```
+------------------------------------------------------------------+
| APACHE KAFKA CHEAT SHEET                                         |
+------------------------------------------------------------------+
| CORE CONCEPTS                                                    |
|   Broker    -> Server storing data                                |
|   Topic     -> Category of messages                               |
|   Partition -> Ordered, immutable sequence                        |
|   Offset    -> Position in partition                              |
|   Producer  -> Publishes to topic                                 |
|   Consumer  -> Reads from topic/partitions                        |
|   Group     -> Load balancing consumers                           |
+------------------------------------------------------------------+
| PRODUCER SETTINGS                                                |
|   acks=0        -> Fire and forget                               |
|   acks=1        -> Leader ACK (default)                           |
|   acks=all      -> All ISR ACK                                   |
|   compression: none, gzip, snappy, lz4, zstd                    |
|   batch.size    -> Bytes to batch before sending                  |
|   linger.ms     -> Max wait time for batching                     |
+------------------------------------------------------------------+
| CONSUMER SETTINGS                                                |
|   auto.offset.reset: earliest, latest, none                      |
|   enable.auto.commit: true/false                                 |
|   max.poll.records: 500                                          |
|   session.timeout.ms: 45000                                      |
|   heartbeat.interval.ms: 3000                                    |
+------------------------------------------------------------------+
| RETENTION POLICIES                                               |
|   time-based:  log.retention.hours=168 (7 days)                  |
|   size-based:  log.retention.bytes=1073741824 (1 GB)             |
|   compact:     log.cleanup.policy=compact                        |
+------------------------------------------------------------------+
| SPRING BOOT ANNOTATIONS                                          |
|   @KafkaListener(topics="...")                                   |
|   @KafkaHandler                                                  |
|   @EnableKafka                                                   |
|   @EnableKafkaStreams                                            |
+------------------------------------------------------------------+
| COMMON CLI COMMANDS                                              |
|   kafka-topics.sh --list                                         |
|   kafka-topics.sh --describe --topic <topic>                     |
|   kafka-console-producer.sh --topic <topic>                      |
|   kafka-console-consumer.sh --topic <topic> --from-beginning     |
|   kafka-consumer-groups.sh --describe --group <group>            |
+------------------------------------------------------------------+
```
