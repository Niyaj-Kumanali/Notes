# Kafka and Messaging Questions

## Questions

1. What is Kafka?
2. Why use Kafka?
3. What is event streaming?
4. What is producer?
5. What is consumer?
6. What is topic?
7. What is partition?
8. What is broker?
9. What is consumer group?
10. How does Kafka scale?
11. Why are partitions important?
12. How do you decide partition count?
13. What is offset?
14. What is committed offset?
15. What is consumer lag?
16. How do you monitor consumer lag?
17. What is replication factor?
18. What is leader and follower replica?
19. What is ISR?
20. What happens when a broker fails?
21. What is acknowledgement in Kafka?
22. Difference between `acks=0`, `acks=1`, and `acks=all`.
23. What is at-most-once delivery?
24. What is at-least-once delivery?
25. What is exactly-once semantics?
26. How do you handle duplicate messages?
27. How do you make consumers idempotent?
28. What is dead-letter topic?
29. What is retry topic?
30. How do you handle poison messages?
31. What is message ordering?
32. How do you guarantee ordering in Kafka?
33. What is key in Kafka message?
34. What is schema registry?
35. What is Kafka retention?
36. Difference between Kafka and RabbitMQ.
37. Difference between Kafka and SQS.
38. Difference between Kafka and MQTT.
39. When would you not use Kafka?
40. How can Kafka be used in an IoT data pipeline?
41. Why is Kafka useful for IoT data?
42. How do you handle sensor data bursts in Kafka?
43. How do you scale Kafka consumers?
44. How do you secure Kafka?
45. What is Kafka Connect?
46. What is Kafka Streams?
47. What is backpressure?
48. How do you handle backpressure?
49. How do you test Kafka consumers?
50. How do you debug message loss?

---

## Answers

1. What is Kafka?
   - **Ans:**
      - Kafka is a distributed event-streaming platform used for high-throughput, fault-tolerant data pipelines
      - In IoT or real-time data architectures, Kafka can serve as the backbone that ingests real-time data from distributed sources and makes it available to multiple downstream consumers
      - Publish-subscribe model: producers publish to a topic and any number of consumers can independently subscribe and consume the same data, unlike traditional message queues where each message is consumed by exactly one consumer
      - Kafka's log-based storage retains messages on disk in append-only logs, allowing consumers to replay from any offset — critical for backfilling historical data without re-ingesting from source devices
2. Why use Kafka?
   - **Ans:**
      - Kafka handles high throughput, provides durability through disk-based logs, and allows multiple consumers to process the same stream independently
      - Kafka is well-suited for scenarios where high-volume event sources could burst thousands of readings per second and a buffer is needed to absorb the spikes without data loss
      - Replay capability is essential for debugging incidents — it is possible to re-consume the exact data that triggered an alert and trace it back to the root cause, something RabbitMQ and SQS cannot do since they delete messages after consumption
3. What is event streaming?
   - **Ans:**
      - Event streaming captures data in real-time as a sequence of events
      - In a real-time data pipeline, each sensor reading or system event can be published to Kafka, creating an immutable log that can be processed, analyzed, and visualized in real-time on dashboards
      - Unlike request-response where a client waits for a server reply, event streaming is fire-and-forget: events are published to a log and consumers process them independently without blocking the producer
      - Events are persisted to disk, can be replayed from any offset, and allow multiple independent consumer groups to process the same data without interfering with each other
4. What is producer?
   - **Ans:**
      - A producer publishes messages to Kafka topics
      - A typical producer might be a Lambda function or application that receives data from an external source and publishes it to a Kafka topic
      - Producer configuration matters: setting `acks=all` ensures no data loss by requiring all in-sync replicas to acknowledge, and tuning `linger.ms` (batch delay) and `batch.size` (max batch bytes) balances latency vs throughput
5. What is consumer?
   - **Ans:**
      - A consumer subscribes to topics and processes messages
      - A typical consumer reads messages from Kafka, performs validation or transformation, and stores results in a downstream system such as a time-series database
      - Consumer offset management determines when offsets are committed: auto-commit (`enable.auto.commit=true`) commits periodically in the background, while manual commit (`commitSync()`/`commitAsync()`) gives control over when offsets are saved. Manual commit after successful processing is recommended for at-least-once delivery with idempotent processing
6. What is topic?
   - **Ans:**
      - A topic is a logical channel where producers send messages and consumers read from
      - Topics can be organized by data type: for example, separate topics for raw sensor data, threshold alerts, and device health messages
      - A consistent naming convention such as `<domain>-<data-type>` (e.g., `sensor-readings`, `sensor-alerts`) is recommended. The number of topics is driven by consumer isolation — separating data that needs different processing logic or SLAs into distinct topics so one consumer's failure doesn't block another
7. What is partition?
   - **Ans:**
      - A partition is a unit of parallelism within a topic — messages are distributed across partitions, which are ordered and immutable
      - A topic might be configured with a number of partitions matching the number of source devices, using the device ID as the message key to ensure all readings from one device go to the same partition
      - Partition count directly caps consumer parallelism: each partition can only be read by one consumer within a group, so N partitions allow at most N consumers. Too many partitions increase metadata overhead in ZooKeeper/KRaft and lengthen rebalance time during scaling events
8. What is broker?
   - **Ans:**
      - A broker is a Kafka server that stores data and serves clients
      - A typical production deployment runs a 3-broker Kafka cluster, which provides fault tolerance — if one broker fails, the others continue serving with replicated data
      - Broker sizing is based on expected throughput and retention: disk is calculated as (msgs/sec × retention period × avg message size × replication factor) to ensure brokers never run out of storage. CPU and memory are sized to handle the producer/consumer request rate
9. What is consumer group?
   - **Ans:**
      - A consumer group is a set of consumers that collectively read from a topic, with each partition assigned to one consumer
      - Multiple consumer instances in the same group can parallelize processing of messages from different partitions
      - When a consumer joins or leaves the group, a rebalance redistributes partitions among remaining consumers. During GC pauses, a consumer might miss heartbeats and trigger an unnecessary rebalance — tuning `session.timeout.ms` to a higher value (e.g., 30 seconds) gives consumers breathing room during garbage collection
10. How does Kafka scale?
    - **Ans:**
      - Kafka scales horizontally by adding brokers to the cluster and partitions to topics
      - More partitions allow more consumers in a group to process in parallel
      - When additional sources are added, partition count can be increased and consumer instances added to handle the increased load
      - Scaling producers is straightforward — they can write to any partition via round-robin or key hashing. Consumers are bounded by partition count: adding more consumers than partitions leaves some idle. Kafka's built-in rebalancing automatically redistributes partitions when consumers join or leave the group
11. Why are partitions important?
    - **Ans:**
      - Partitions enable parallelism — multiple consumers can read different partitions concurrently
      - Without partitions, there would be a single sequential stream
      - Having multiple partitions means multiple consumers can process data in parallel, keeping the pipeline responsive even during bursts
      - Partitions provide ordering guarantees within a partition: messages with the same key always land in the same partition and are read in order. The partition count is the ceiling on consumer parallelism — there cannot be more active consumers in a group than partitions
12. How do you decide partition count?
    - **Ans:**
      - Partition count is based on expected throughput, number of consumers, and key cardinality
      - A reasonable starting point is one partition per expected data source or key, then monitoring consumer lag to determine if more are needed
      - The formula is: partitions ≥ max(target throughput / single partition throughput, desired consumer parallelism). For example, targeting 6000 msg/sec with each partition handling 1000 msg/sec requires at least 6 partitions
      - Too many partitions increase ZooKeeper/KRaft metadata overhead and extend rebalancing time during scaling, so the minimum that meets throughput and parallelism needs is preferred
13. What is offset?
    - **Ans:**
      - An offset is a sequential ID assigned to each message within a partition, representing its position
      - Offsets allow consumers to track how far they've read
      - If a consumer crashes, it can resume from its last committed offset without missing or duplicating messages
      - Offset commits can be automatic (`enable.auto.commit=true` with a configurable `auto.commit.interval.ms`) or manual (`commitSync()`/`commitAsync()`). Manual commits are recommended for critical processing because auto-commit could mark messages as consumed before processing finishes, risking data loss on crash
14. What is committed offset?
    - **Ans:**
      - A committed offset is the last offset a consumer has successfully processed and saved to Kafka's internal `__consumer_offsets` topic
      - When a consumer crashes during peak load, the committed offset ensures it resumes from where it left off rather than re-processing thousands of messages
      - Committing before processing is fast but risks message loss if the consumer crashes mid-processing. Committing after processing means a crash causes the message to be redelivered on restart, producing duplicates. Accepting duplicates in favor of zero data loss is a common trade-off when the consumer is idempotent
15. What is consumer lag?
    - **Ans:**
      - Consumer lag is the difference between the latest offset in a partition and the consumer's committed offset
      - If a downstream write path experiences a slowdown, lag can spike dramatically
      - This indicates the consumer is falling behind and cannot keep up with the producer rate
      - Lag is typically monitored using the `kafka-consumer-groups --describe` CLI command and publishing lag metrics to a monitoring system. An alarm at a threshold (e.g., 10,000 messages) can page the on-call engineer before the pipeline falls too far behind
16. How do you monitor consumer lag?
    - **Ans:**
      - The `kafka-consumer-groups --bootstrap-server --describe --group` command checks lag per partition
      - Lag metrics can be automated by publishing them to a monitoring system at regular intervals and setting a threshold alarm for immediate investigation
      - Burrow (LinkedIn's open-source lag monitor) provides lag trend analysis beyond raw numbers — an increasing lag trend means the consumer is falling further behind and needs immediate attention, while a stable or decreasing trend indicates the consumer is keeping pace or catching up
17. What is replication factor?
    - **Ans:**
      - Replication factor determines how many copies of each partition exist across brokers
      - A replication factor of 3 ensures that if one broker goes down, no data is lost and the cluster continues serving
      - The trade-off is durability vs resource cost: RF=3 stores each partition on 3 brokers, tripling disk usage and replication traffic. RF=1 uses minimal resources but risks data loss if that single broker fails
      - A common production standard is RF=3 with `min.insync.replicas=2`, which allows one broker to fail without data loss while still acknowledging writes after 2 replicas confirmed
18. What is leader and follower replica?
    - **Ans:**
      - For each partition, one broker is the leader that handles all reads and writes, and the rest are in-sync followers
      - When a leader broker goes down for maintenance, a follower automatically becomes the new leader with zero data loss
      - When a leader fails, Kafka triggers leader election and promotes the first in-sync follower to leader. The `preferred.replica.leader.election` config (or `auto.leader.rebalance.enable`) redistributes leadership back to preferred replicas once a failed broker recovers, preventing one broker from hoarding all leadership
19. What is ISR?
    - **Ans:**
      - ISR (In-Sync Replicas) are followers that are fully caught up with the leader
      - Setting `min.insync.replicas=2` ensures that at least 2 brokers acknowledge each write
      - This prevents data loss even if one broker fails after acknowledging
      - A follower falls out of ISR if it can't keep up with the leader (slow network, GC pause, disk bottleneck). Once it catches up, it re-enters the ISR. Kafka uses truncated replication: the follower fetches from the leader and truncates any divergent data before replicating the current log
20. What happens when a broker fails?
    - **Ans:**
      - When a broker fails, the controller detects it, and partition leaders on that broker are reassigned to ISR followers on other brokers
      - During an instance failure, the Kafka cluster can automatically reassign leadership within seconds, and producers/consumers continue with minimal interruption
      - A clean shutdown triggers controlled leader migration — followers are promoted in order and no data is lost. A hard failure with no ISR available may trigger unclean leader election, risking data loss. Using `min.insync.replicas=2` mitigates this by ensuring an ISR always exists even if one broker fails
21. What is acknowledgement in Kafka?
    - **Ans:**
      - Acknowledgement (acks) determines when the producer considers a write successful
      - For critical data where losing a message is unacceptable, `acks=all` is used — the producer waits for all in-sync replicas to acknowledge before moving on
      - `acks=all` increases latency because the producer waits for all ISRs to acknowledge, but guarantees no data loss. Setting `max.in.flight.requests.per.connection=1` prevents message reordering when producers retry, ensuring strict ordering within a partition
22. Difference between `acks=0`, `acks=1`, and `acks=all`.
    - **Ans:**
      - `acks=0` fires and forgets (fastest, highest risk)
      - `acks=1` waits for the leader only (balanced)
      - `acks=all` waits for all ISRs (safest, slower)
      - `acks=all` is used for critical data and `acks=1` for non-critical messages
      - `acks=all` requires all ISR members to acknowledge. With `min.insync.replicas=2`, if one of 3 brokers is down, only 2 ISRs exist and both must acknowledge — the write succeeds. If 2 brokers are down leaving only 1 ISR, the producer gets a NotEnoughReplicasException because `min.insync.replicas` isn't met
23. What is at-most-once delivery?
    - **Ans:**
      - At-most-once means a message is delivered zero or one time — if the consumer fails after fetching but before processing, the message is lost
      - At-most-once is generally avoided for critical data because losing a message could mean missing an important event
      - At-most-once is configured with `acks=0` on the producer (fire and forget, no acknowledgment) and `enable.auto.commit=true` on the consumer with auto-commit happening before processing completes. This is fast but risky — messages can be lost on both the producer and consumer side
24. What is at-least-once delivery?
    - **Ans:**
      - At-least-once means messages can be delivered more than once but never lost
      - At-least-once delivery is achieved by setting `enable.auto.commit=false` and manually committing offsets only after successful processing and storage in the downstream system
      - At-least-once requires idempotent consumers because duplicates can occur during rebalances or consumer restarts. Deduplication can be implemented by checking a unique event ID in the target store before inserting — if the record exists, the consumer skips it
25. What is exactly-once semantics?
    - **Ans:**
      - Exactly-once semantics (EOS) ensures each message is processed exactly once, with no duplicates and no losses
      - While Kafka supports EOS with transactions and idempotent producers, at-least-once with idempotent consumers is often preferred because the setup complexity is higher than the duplicate probability
      - EOS requires Kafka transactions with `transactional.id`, idempotent producers, and consumer isolation levels (`isolation.level=read_committed`), adding coordination overhead and latency. For financial systems where every dollar matters it's essential, but for sensor data where occasional duplicates are tolerable, at-least-once with idempotent consumers is simpler and sufficient
26. How do you handle duplicate messages?
    - **Ans:**
      - Duplicates are handled by making consumers idempotent — either by tracking processed event IDs in Redis or using database unique constraints
      - Each message can have a unique ID (e.g., device ID + timestamp), and a unique constraint on that in the target store prevents duplicate storage
      - Producer-side duplicates happen when the producer retries after a network timeout — the broker received the first write but the ack was lost. Consumer-side duplicates happen when a rebalance reassigns a partition before the consumer commits its offset. Both are handled by making the consumer idempotent: checking the event ID in the target store before inserting, which prevents duplicates regardless of their source
27. How do you make consumers idempotent?
    - **Ans:**
      - Idempotent consumers produce the same result regardless of how many times a message is processed
      - A typical approach is to check if the unique event ID already exists in the target table before inserting, and skip duplicates
      - This makes the entire pipeline safe to retry during failures
      - Kafka transactions offer true exactly-once but require a transactional producer, consumer isolation level config, and careful coordination. Idempotent consumers are simpler — they achieve the same result by checking if a record was already processed before applying it. Idempotency works across all duplicate sources without requiring Kafka-specific transaction setup
28. What is dead-letter topic?
    - **Ans:**
      - A dead-letter topic (DLT) stores messages that consumers cannot process after repeated retries
      - For an alert-processing topic, if a message fails validation after N retries, it can be moved to the DLT for manual inspection rather than blocking the consumer stream
      - The DLT should have a retention TTL so old failed messages don't consume storage indefinitely. A monitoring alert on the DLT topic's consumer lag can trigger notifications with error details, enabling rapid root-cause investigation
29. What is retry topic?
    - **Ans:**
      - A retry topic holds messages that failed temporarily so they can be reprocessed after a delay
      - When a downstream store is temporarily unavailable, the consumer can move the message to a retry topic with a delay before the next attempt, preventing a tight retry loop
      - The architecture flows: main topic → consumer attempts processing → on failure, message goes to retry topic with a delay header → retry consumer picks it up after the delay → if it still fails after N retries, the message is moved to the DLT. The retry delay is typically set using a timestamp header — the retry consumer checks if the delay has elapsed before processing
30. How do you handle poison messages?
    - **Ans:**
      - Poison messages are messages that always cause consumer failures
      - A malformed JSON payload from a faulty device can cause continuous deserialization errors
      - The recommended approach is to wrap deserialization in a try-catch and send the bad message to a DLT with the error details logged
      - Poison messages are detected by tracking repeated failures on the same offset — if the consumer fails N times on the same message, it's likely malformed. Automated DLT routing is critical because without it, the consumer keeps retrying the same bad message, gets stuck in a rebalance loop, and stops processing all other messages in the partition
31. What is message ordering?
    - **Ans:**
      - Message ordering guarantees that messages are processed in the order they were produced
      - Kafka guarantees order within a partition, not across partitions
      - Using the device ID as the message key ensures all readings from one device go to the same partition, preserving per-device ordering
      - Strong ordering requires all related messages to land in the same partition, which limits parallelism to one consumer per key. A common design pattern is to guarantee per-device ordering (device ID as key) while allowing different devices to be processed in parallel and out of order relative to each other
32. How do you guarantee ordering in Kafka?
    - **Ans:**
      - Use the same message key for all related messages — Kafka assigns messages with the same key to the same partition
      - Using a device or entity ID as the key ensures messages from each source are strictly ordered
      - Setting `max.in.flight.requests.per.connection=1` prevents reordering on producer retries
      - `enable.idempotence=true` enables the producer to assign sequence numbers to each batch, and Kafka's broker automatically deduplicates and reorders on the server side. This means `max.in.flight.requests.per.connection > 1` (up to 5) can be safely used without risking ordering, because out-of-order or duplicate batches are rejected by the broker
33. What is key in Kafka message?
    - **Ans:**
      - The message key determines which partition a message goes to — same key always goes to the same partition
      - Using a device or entity ID as the key ensures all data from one source is in order and also helps with debugging by filtering messages by source
      - Null keys use round-robin partition assignment, distributing messages evenly but with no ordering guarantee. The key should be chosen based on whether per-source ordering is needed; if only even distribution without ordering is required, a null key or random key can be used
34. What is schema registry?
    - **Ans:**
      - Schema Registry stores and validates message schemas (Avro/JSON Schema) to ensure compatibility between producers and consumers
      - While many teams use JSON with a shared contract, Schema Registry prevents issues like a producer sending an extra field without the consumer expecting it
      - Backward compatibility means new consumers can read old data; forward compatibility means old consumers can read new data. Schema Registry enforces these rules at produce time — if a producer tries to add a required field that breaks backward compatibility, the registry rejects the schema change, preventing silent downstream failures
35. What is Kafka retention?
    - **Ans:**
      - Retention controls how long Kafka retains messages
      - A typical retention policy is 7 days for critical data topics
      - This allows reprocessing data if a downstream system fails without needing to contact source devices again
      - Time-based retention deletes messages after a configured duration (e.g., 7 days). Size-based retention keeps the latest messages up to a total size limit per partition. Log compaction (`cleanup.policy=compact`) retains only the latest value per key, useful for stateful streams like changelog topics where only the current state is needed
36. Difference between Kafka and RabbitMQ.
    - **Ans:**
      - Kafka is a distributed log optimized for high-throughput event streaming with replay capability
      - RabbitMQ is a traditional message broker with complex routing and priority queues
      - Kafka is preferred when data is high-volume, needs replay, and has multiple consumers — scenarios where RabbitMQ handles things less efficiently
      - RabbitMQ excels at task distribution with complex routing (exchanges, bindings, priority queues) and deletes messages after acknowledgment — ideal for work queues and RPC. Kafka excels at event streaming with durable, replayable logs — ideal for high-throughput data pipelines. The choice hinges on whether message deletion after consumption is needed (RabbitMQ) or log retention with replay (Kafka)
37. Difference between Kafka and SQS.
    - **Ans:**
      - SQS is a fully managed AWS queue with at-least-once delivery and automatic scaling
      - Kafka requires manual cluster management but offers lower latency, higher throughput, and multi-consumer fan-out
      - Kafka is preferred on self-managed infrastructure when sub-100ms latency for real-time dashboards and multiple consumer groups are needed — SQS would add complexity for fan-out
      - SQS wins when zero infrastructure management, automatic scaling, and moderate throughput (under 10K msg/sec) are needed. It's a fully managed service — no brokers to patch, no cluster sizing, no partition planning. However, SQS lacks replay capability and multi-consumer fan-out is limited compared to Kafka
38. Difference between Kafka and MQTT.
    - **Ans:**
      - MQTT is a lightweight pub-sub protocol for IoT devices with limited bandwidth
      - Kafka is a distributed storage and streaming platform
      - In IoT architectures, they complement each other: MQTT carries data from edge devices to a cloud IoT broker, and Kafka handles the backend streaming from the broker to data stores
      - MQTT QoS 0 is at-most-once (fire and forget), QoS 1 is at-least-once (acknowledged delivery), and QoS 2 is exactly-once (four-part handshake). QoS 1 is commonly chosen for MQTT to guarantee device messages reached the broker, paired with Kafka's at-least-once delivery — achieving zero end-to-end message loss while avoiding the overhead of QoS 2
39. When would you not use Kafka?
    - **Ans:**
      - Kafka should not be used for simple task queues, low-throughput (< 100 msg/sec) scenarios, or when exactly-once delivery is needed without idempotent consumers
      - Also, if the team lacks operational experience with Kafka, the learning curve and operational overhead can be significant
      - For example, if a system only had 10 readings per minute and no replay requirement, a simple REST API from the source to a serverless function writing directly to a database would eliminate the need for a Kafka cluster entirely — saving infrastructure costs, operational complexity, and the team's maintenance burden
40. How can Kafka be used in an IoT data pipeline?
    - **Ans:**
      - Kafka can serve as the central message backbone in an IoT architecture
      - IoT gateways send data via MQTT to a cloud IoT broker, which forwards it to a processing function
      - The processing function publishes the data to a Kafka topic (e.g., `sensor-readings`)
      - A consumer application reads from Kafka, validates readings, and stores them in a time-series database
      - A separate consumer handles threshold-based alerting
      - Full architecture: IoT Gateways publish MQTT → Cloud IoT Core (MQTT broker) → Serverless function (Kafka producer) → Kafka cluster (3 brokers, `sensor-readings` topic with 6 partitions) → Consumer application (validates and writes to time-series DB) → Grafana reads from the DB for dashboards. A second consumer handles threshold-based alerting and sends notifications via a pub/sub service
41. Why is Kafka useful for IoT data?
    - **Ans:**
      - IoT data is high-volume, continuous, and comes from many sources
      - Kafka's log-based architecture absorbs bursty sensor data without backpressure, provides a durable buffer that prevents data loss during downstream outages, and allows multiple consumers (storage, alerting, analytics) to process the same stream independently
      - When a downstream database goes down for 30 minutes during a maintenance window, Kafka's retention policy buffers all sensor data. When the database comes back, the consumer simply resumes from its committed offset and catches up — zero data loss. With a point-to-point integration (like direct MQTT to database), those 30 minutes of data would have been permanently lost
42. How do you handle sensor data bursts in Kafka?
    - **Ans:**
      - Kafka naturally handles bursts because it's disk-based — producers can write faster than consumers can read
      - However, to prevent unbounded lag, partition count and consumer instances should be sized to handle peak throughput
      - Producer `max.block.ms` and consumer `fetch.max.bytes` should be set appropriately to avoid memory issues during bursts
      - Kafka producers experience no backpressure — they always write at full speed to the log. Consumers apply backpressure by calling `consumer.pause()` on assigned partitions when the downstream system is slow, preventing unbounded lag. When the downstream recovers, `consumer.resume()` restarts consumption
43. How do you scale Kafka consumers?
    - **Ans:**
      - Consumers are scaled by adding more consumer instances to the same consumer group
      - The maximum parallelism equals the number of partitions
      - When additional sources are added, partition count can be increased and more consumer instances deployed to distribute the load
      - Increasing partitions after topic creation adds new empty partitions — existing messages remain in the old partitions and aren't redistributed. This means partition count cannot be reduced later, and uneven message distribution can occur. Plan partition count upfront based on projected peak throughput and desired consumer parallelism
44. How do you secure Kafka?
    - **Ans:**
      - Kafka can be secured with SSL/TLS for encryption, SASL for authentication, and ACLs for authorization
      - When Kafka runs on private infrastructure within a VPC and is accessed only by internal services, security groups for network isolation may be used instead of enabling Kafka-level SSL
      - SASL/SCRAM provides username-password authentication: create JAAS config with credentials, configure `security.protocol=SASL_SSL`, and add ACLs to restrict topic access per user. Always encrypt data in transit with SSL/TLS if Kafka is accessible outside the VPC — without it, message payloads are sent in plaintext and vulnerable to interception
45. What is Kafka Connect?
    - **Ans:**
      - Kafka Connect is a framework for streaming data between Kafka and external systems using connectors
      - Kafka Connect can be evaluated as an alternative to custom consumer applications, but it may not support all validation or transformation logic required by a specific use case
      - Source connectors push data from external systems (databases, logs, APIs) into Kafka topics. Sink connectors pull data from Kafka topics and write to external systems (databases, search indices, file systems). Both run as distributed or standalone workers, eliminating the need to write custom producer/consumer code for common integrations
46. What is Kafka Streams?
    - **Ans:**
      - Kafka Streams is a Java library for building stream processing applications on top of Kafka
      - Kafka Streams can be used for operations like sliding-window average calculations, though custom consumers may be used depending on team familiarity
      - Stateful operations include windowed aggregations (tumbling, hopping, session windows) and stream-table joins. Kafka Streams stores local state in RocksDB on disk for fast reads, and backs up state changes to changelog topics in Kafka. On failure, the state is rebuilt by replaying the changelog topic, providing fault-tolerant state management without external databases
47. What is backpressure?
    - **Ans:**
      - Backpressure is a feedback mechanism where a slow consumer signals the producer to slow down
      - Kafka doesn't have traditional backpressure — producers always write at full speed, and consumers fall behind if they can't keep up
      - A slow downstream write path can create backpressure on the consumer, causing lag
      - A bounded queue between the Kafka poll loop and the processing thread pool can manage this. When the queue fills up (indicating the downstream is slow), `consumer.pause()` stops fetching new messages. When the queue drains below a threshold, `consumer.resume()` restarts consumption, preventing unbounded memory growth and consumer lag
48. How do you handle backpressure?
    - **Ans:**
      - In Kafka, backpressure is handled at the consumer level by pausing partition assignment when the processing pipeline is saturated
      - When a downstream store has a write slowdown, the consumer can pause all partitions, wait for pending writes to complete, and then resume consuming
      - An alternative approach uses a bounded queue between the poll loop and processing threads — the poll loop blocks when the queue is full, naturally throttling consumption. This mirrors reactive streams backpressure (like Reactive Kafka) where the consumer signals demand upstream, but Kafka's native approach uses partition pausing instead of pull-based demand signaling
49. How do you test Kafka consumers?
    - **Ans:**
      - Kafka consumers can be tested using embedded Kafka with Spring Kafka's `@EmbeddedKafka` annotation in integration tests
      - A typical test publishes sample data to an embedded topic, verifies the consumer processes it, and checks the data landed in a test database instance
      - Beyond happy-path tests, failure scenarios should be tested: simulating broker failures to verify consumer recovery, triggering rebalances by adding/removing consumer instances, sending malformed JSON to verify DLT routing, and asserting that offsets are committed only after successful processing to guarantee no data loss
50. How do you debug message loss?
    - **Ans:**
      - Start by checking consumer lag — if lag is zero, messages were consumed
      - Then check the consumer's committed offset against the latest offset
      - Enable producer-side logging with `errors.tolerance=all` and check the DLT
      - A common cause of message loss is a consumer crash between processing and offset commit
      - Systematic debugging approach: (1) check producer `acks` config and broker logs for write failures, (2) check consumer error logs for deserialization or processing exceptions, (3) compare consumer committed offset vs partition end offset to find gaps, (4) replay the topic from the suspected gap offset to verify the message exists in the log. The key insight is knowing the delivery semantic (at-least-once vs at-most-once) and checking each layer systematically
