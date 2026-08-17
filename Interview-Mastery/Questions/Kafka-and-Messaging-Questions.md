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
40. How did you use Kafka in your cold-chain project?
41. Why was Kafka useful for IoT data?
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
   - **Answer:**
      - Kafka is a distributed event-streaming platform used for high-throughput, fault-tolerant data pipelines
      - In the cold-chain project, Kafka was the backbone that ingested real-time temperature sensor data from IoT gateways and made it available to multiple downstream consumers
      - Publish-subscribe model: producers publish to a topic and any number of consumers can independently subscribe and consume the same data, unlike traditional message queues where each message is consumed by exactly one consumer
      - Kafka's log-based storage retains messages on disk in append-only logs, allowing consumers to replay from any offset — critical for backfilling historical temperature data without re-ingesting from gateways
2. Why use Kafka?
   - **Answer:**
      - Kafka handles high throughput, provides durability through disk-based logs, and allows multiple consumers to process the same stream independently
      - We chose Kafka in the cold-chain project because IoT gateways could burst thousands of sensor readings per second and we needed a buffer that could absorb the spikes without data loss
      - Replay capability was essential for debugging temperature excursion incidents — we could re-consume the exact data that triggered an alert and trace it back to the root cause, something RabbitMQ and SQS cannot do since they delete messages after consumption
3. What is event streaming?
   - **Answer:**
      - Event streaming captures data in real-time as a sequence of events
      - In the cold-chain system, every temperature reading from each sensor was an event published to Kafka, creating an immutable log that we could process, analyze, and visualize in real-time on the Grafana dashboard
      - Unlike request-response where a client waits for a server reply, event streaming is fire-and-forget: events are published to a log and consumers process them independently without blocking the producer
      - Events are persisted to disk, can be replayed from any offset, and allow multiple independent consumer groups to process the same data without interfering with each other
4. What is producer?
   - **Answer:**
      - A producer publishes messages to Kafka topics
      - In our cold-chain pipeline, an AWS Lambda function acted as the producer — it received MQTT messages from AWS IoT Core and published them to the `sensor-readings` Kafka topic on our EC2-hosted Kafka cluster
      - Producer configuration matters: I set `acks=all` to ensure no data loss by requiring all in-sync replicas to acknowledge, and tuned `linger.ms` (batch delay) and `batch.size` (max batch bytes) to balance latency vs throughput
5. What is consumer?
   - **Answer:**
      - A consumer subscribes to topics and processes messages
      - In the cold-chain project, our Spring Boot application was the consumer — it read temperature readings from Kafka, validated them, and stored them in InfluxDB for time-series analysis
      - Consumer offset management determines when offsets are committed: auto-commit (`enable.auto.commit=true`) commits periodically in the background, while manual commit (`commitSync()`/`commitAsync()`) gives control over when offsets are saved. I used manual commit after successful processing for at-least-once delivery with idempotent processing
6. What is topic?
   - **Answer:**
      - A topic is a logical channel where producers send messages and consumers read from
      - In the cold-chain system, I organized topics by data type: `sensor-readings` for temperature data, `sensor-alerts` for threshold violations, and `sensor-health` for gateway heartbeat messages
      - I used a consistent naming convention: `<domain>-<data-type>` (e.g., `sensor-readings`, `sensor-alerts`). The number of topics was driven by consumer isolation — separating data that needed different processing logic or SLAs into distinct topics so one consumer's failure wouldn't block another
7. What is partition?
   - **Answer:**
      - A partition is a unit of parallelism within a topic — messages are distributed across partitions, which are ordered and immutable
      - I configured the `sensor-readings` topic with 6 partitions to match the 6 IoT gateways, using the gateway ID as the message key to ensure all readings from one gateway went to the same partition
      - Partition count directly caps consumer parallelism: each partition can only be read by one consumer within a group, so 6 partitions allow at most 6 consumers. Too many partitions increase metadata overhead in ZooKeeper/KRaft and lengthen rebalance time during scaling events
8. What is broker?
   - **Answer:**
      - A broker is a Kafka server that stores data and serves clients
      - We ran a 3-broker Kafka cluster on EC2 instances, which gave us fault tolerance — if one broker failed, the others continued serving with replicated data
      - Broker sizing was based on expected throughput and retention: we calculated disk as (1000 msgs/sec × 7 days × avg message size × 3 for replication factor) to ensure brokers never ran out of storage. CPU and memory were sized to handle the producer/consumer request rate
9. What is consumer group?
   - **Answer:**
      - A consumer group is a set of consumers that collectively read from a topic, with each partition assigned to one consumer
      - In cold-chain, we had multiple consumer instances in the same group to parallelize processing of sensor data from different gateways
      - When a consumer joins or leaves the group, a rebalance redistributes partitions among remaining consumers. During GC pauses, a consumer might miss heartbeats and trigger an unnecessary rebalance — I tuned `session.timeout.ms` to 30 seconds to give consumers breathing room during garbage collection
10. How does Kafka scale?
    - **Answer:**
      - Kafka scales horizontally by adding brokers to the cluster and partitions to topics
      - More partitions allow more consumers in a group to process in parallel
      - In cold-chain, when we added two more gateways, I increased the partition count and added consumer instances to handle the increased load
      - Scaling producers is straightforward — they can write to any partition via round-robin or key hashing. Consumers are bounded by partition count: adding more consumers than partitions leaves some idle. Kafka's built-in rebalancing automatically redistributes partitions when consumers join or leave the group
11. Why are partitions important?
    - **Answer:**
      - Partitions enable parallelism — multiple consumers can read different partitions concurrently
      - Without partitions, we'd have a single sequential stream
      - In cold-chain, 6 partitions meant 6 consumers could process temperature data in parallel, keeping the pipeline responsive even during sensor bursts
      - Partitions provide ordering guarantees within a partition: messages with the same key always land in the same partition and are read in order. The partition count is the ceiling on consumer parallelism — you cannot have more active consumers in a group than partitions
12. How do you decide partition count?
    - **Answer:**
      - I base partition count on the expected throughput, number of consumers, and key cardinality
      - For cold-chain, I started with 6 partitions — one per gateway — and monitored consumer lag
      - I planned to increase to 12 if lag grew, but 6 was sufficient for our 1000 msg/sec peak
      - The formula is: partitions ≥ max(target throughput / single partition throughput, desired consumer parallelity). For example, targeting 6000 msg/sec with each partition handling 1000 msg/sec requires at least 6 partitions
      - Too many partitions increase ZooKeeper/KRaft metadata overhead and extend rebalancing time during scaling, so I aim for the minimum that meets throughput and parallelism needs
13. What is offset?
    - **Answer:**
      - An offset is a sequential ID assigned to each message within a partition, representing its position
      - Offsets allow consumers to track how far they've read
      - In the cold-chain pipeline, if a consumer crashed, it could resume from its last committed offset without missing or duplicating messages
      - Offset commits can be automatic (`enable.auto.commit=true` with a configurable `auto.commit.interval.ms`) or manual (`commitSync()`/`commitAsync()`). I switched to manual commits for critical alert processing because auto-commit could mark messages as consumed before processing finished, risking data loss on crash
14. What is committed offset?
    - **Answer:**
      - A committed offset is the last offset a consumer has successfully processed and saved to Kafka's internal `__consumer_offsets` topic
      - When the inventory consumer crashed during a peak load, the committed offset ensured it resumed from where it left off rather than re-processing thousands of messages
      - Committing before processing is fast but risks message loss if the consumer crashes mid-processing. Committing after processing means a crash causes the message to be redelivered on restart, producing duplicates. I accepted duplicates in favor of zero data loss because our consumer was idempotent
15. What is consumer lag?
    - **Answer:**
      - Consumer lag is the difference between the latest offset in a partition and the consumer's committed offset
      - When our InfluxDB write path had a slowdown, I saw lag spike to 50,000 messages
      - That told me the consumer was falling behind and couldn't keep up with the producer rate
      - I monitored lag using the `kafka-consumer-groups --describe` CLI command and published lag metrics to CloudWatch every minute. A CloudWatch alarm was configured at 10,000 messages to page the on-call engineer before the pipeline fell too far behind
16. How do you monitor consumer lag?
    - **Answer:**
      - I used the `kafka-consumer-groups --bootstrap-server --describe --group` command to check lag per partition
      - In cold-chain, I automated this by publishing lag metrics to CloudWatch every minute and set a threshold alarm at 10,000 messages lag for immediate investigation
      - Burrow (LinkedIn's open-source lag monitor) provides lag trend analysis beyond raw numbers — an increasing lag trend means the consumer is falling further behind and needs immediate attention, while a stable or decreasing trend indicates the consumer is keeping pace or catching up
17. What is replication factor?
    - **Answer:**
      - Replication factor determines how many copies of each partition exist across brokers
      - In cold-chain, I set replication factor to 3 for the sensor-readings topic so that if one EC2 broker instance went down, no data was lost and the cluster continued serving
      - The trade-off is durability vs resource cost: RF=3 stores each partition on 3 brokers, tripling disk usage and replication traffic. RF=1 uses minimal resources but risks data loss if that single broker fails
      - Our production standard was RF=3 with `min.insync.replicas=2`, which allowed one broker to fail without data loss while still acknowledging writes after 2 replicas confirmed
18. What is leader and follower replica?
    - **Answer:**
      - For each partition, one broker is the leader that handles all reads and writes, and the rest are in-sync followers
      - When the leader broker for our sensor-readings partition went down for maintenance, a follower automatically became the new leader with zero data loss
      - When a leader fails, Kafka triggers leader election and promotes the first in-sync follower to leader. The `preferred.replica.leader.election` config (or `auto.leader.rebalance.enable`) redistributes leadership back to preferred replicas once a failed broker recovers, preventing one broker from hoarding all leadership
19. What is ISR?
    - **Answer:**
      - ISR (In-Sync Replicas) are followers that are fully caught up with the leader
      - I set `min.insync.replicas=2` for cold-chain topics to ensure that at least 2 brokers acknowledged each write
      - This prevented data loss even if one broker failed after acknowledging
      - A follower falls out of ISR if it can't keep up with the leader (slow network, GC pause, disk bottleneck). Once it catches up, it re-enters the ISR. Kafka uses truncated replication: the follower fetches from the leader and truncates any divergent data before replicating the current log
20. What happens when a broker fails?
    - **Answer:**
      - When a broker fails, the controllers detect it, partition leaders on that broker are reassigned to ISR followers on other brokers
      - During one EC2 instance failure in cold-chain, the Kafka cluster automatically reassigned the leadership within seconds, and producers/consumers continued with minimal interruption
      - A clean shutdown triggers controlled leader migration — followers are promoted in order and no data is lost. A hard failure with no ISR available may trigger unclean leader election, risking data loss. We mitigated this by using `min.insync.replicas=2`, ensuring an ISR always existed even if one broker failed
21. What is acknowledgement in Kafka?
    - **Answer:**
      - Acknowledgement (acks) determines when the producer considers a write successful
      - In cold-chain, I used `acks=all` for the temperature topic because losing a reading was unacceptable — the producer waited for all in-sync replicas to acknowledge before moving on
      - `acks=all` increases latency because the producer waits for all ISRs to acknowledge, but guarantees no data loss. I set `max.in.flight.requests.per.connection=1` to prevent message reordering when producers retry, ensuring strict ordering within a partition
22. Difference between `acks=0`, `acks=1`, and `acks=all`.
    - **Answer:**
      - `acks=0` fires and forgets (fastest, highest risk)
      - `acks=1` waits for the leader only (balanced)
      - `acks=all` waits for all ISRs (safest, slower)
      - I used `acks=all` for sensor data and `acks=1` for non-critical health check messages
      - `acks=all` requires all ISR members to acknowledge. With `min.insync.replicas=2`, if one of 3 brokers is down, only 2 ISRs exist and both must acknowledge — the write succeeds. If 2 brokers are down leaving only 1 ISR, the producer gets a NotEnoughReplicasException because `min.insync.replicas` isn't met
23. What is at-most-once delivery?
    - **Answer:**
      - At-most-once means a message is delivered zero or one time — if the consumer fails after fetching but before processing, the message is lost
      - In cold-chain, I avoided this for temperature data because losing a reading could mean missing a temperature excursion
      - At-most-once is configured with `acks=0` on the producer (fire and forget, no acknowledgment) and `enable.auto.commit=true` on the consumer with auto-commit happening before processing completes. This is fast but risky — messages can be lost on both the producer and consumer side
24. What is at-least-once delivery?
    - **Answer:**
      - At-least-once means messages can be delivered more than once but never lost
      - In cold-chain, I used at-least-once by setting `enable.auto.commit=false` and manually committing offsets only after successful processing and storage in InfluxDB
      - At-least-once requires idempotent consumers because duplicates can occur during rebalances or consumer restarts. I implemented deduplication by checking the unique event ID (gatewayID + timestamp) in InfluxDB before inserting — if the record existed, the consumer skipped it
25. What is exactly-once semantics?
    - **Answer:**
      - Exactly-once semantics (EOS) ensures each message is processed exactly once, with no duplicates and no losses
      - While Kafka supports EOS with transactions and idempotent producers, I used at-least-once with idempotent consumers instead because the setup complexity was higher than the duplicate probability
      - EOS requires Kafka transactions with `transactional.id`, idempotent producers, and consumer isolation levels (`isolation.level=read_committed`), adding coordination overhead and latency. For financial systems where every dollar matters it's essential, but for IoT sensor data where occasional duplicates are tolerable, at-least-once with idempotent consumers is simpler and sufficient
26. How do you handle duplicate messages?
    - **Answer:**
      - I handle duplicates by making consumers idempotent — either by tracking processed event IDs in Redis or using database unique constraints
      - In cold-chain, each sensor reading had a unique ID (gatewayID + timestamp), and I used a unique constraint on that in InfluxDB to prevent duplicate storage
      - Producer-side duplicates happen when the producer retries after a network timeout — the broker received the first write but the ack was lost. Consumer-side duplicates happen when a rebalance reassigns a partition before the consumer commits its offset. I handled both by making the consumer idempotent: checking the event ID in InfluxDB before inserting, which prevented duplicates regardless of their source
27. How do you make consumers idempotent?
    - **Answer:**
      - Idempotent consumers produce the same result regardless of how many times a message is processed
      - In the cold-chain consumer, I checked if the unique event ID already existed in the target table before inserting, and skipped duplicates
      - This made the entire pipeline safe to retry during failures
      - Kafka transactions offer true exactly-once but require a transactional producer, consumer isolation level config, and careful coordination. Idempotent consumers are simpler — they achieve the same result by checking if a record was already processed before applying it. I chose idempotency because it worked across all duplicate sources without requiring Kafka-specific transaction setup
28. What is dead-letter topic?
    - **Answer:**
      - A dead-letter topic (DLT) stores messages that consumers cannot process after repeated retries
      - In cold-chain, I configured a DLT for the alert-processing topic — if a sensor reading failed validation after 3 retries, it was moved to the DLT for manual inspection rather than blocking the consumer stream
      - The DLT had a retention TTL so old failed messages didn't consume storage indefinitely. I set up a CloudWatch alarm on the DLT topic's consumer lag — any new message landing in the DLT triggered a Slack notification to the team with the error details, enabling rapid root-cause investigation
29. What is retry topic?
    - **Answer:**
      - A retry topic holds messages that failed temporarily so they can be reprocessed after a delay
      - In cold-chain, when InfluxDB was temporarily unavailable, the consumer moved the message to a retry topic with a 10-second delay before the next attempt, preventing a tight retry loop
      - The architecture flows: main topic → consumer attempts processing → on failure, message goes to retry topic with a delay header → retry consumer picks it up after the delay → if it still fails after N retries, the message is moved to the DLT. The retry delay was set using a timestamp header — the retry consumer checked if the delay had elapsed before processing
30. How do you handle poison messages?
    - **Answer:**
      - Poison messages are messages that always cause consumer failures
      - In cold-chain, a malformed JSON payload from a faulty gateway once caused continuous deserialization errors
      - I handled it by wrapping the deserialization in a try-catch and sending the bad message to a DLT with the error details logged
      - Poison messages are detected by tracking repeated failures on the same offset — if the consumer fails N times on the same message, it's likely malformed. Automated DLT routing is critical because without it, the consumer keeps retrying the same bad message, gets stuck in a rebalance loop, and stops processing all other messages in the partition
31. What is message ordering?
    - **Answer:**
      - Message ordering guarantees that messages are processed in the order they were produced
      - Kafka guarantees order within a partition, not across partitions
      - In cold-chain, I used the gateway ID as the message key so all readings from one gateway went to the same partition, preserving per-gateway ordering
      - Strong ordering requires all related messages to land in the same partition, which limits parallelism to one consumer per key. I designed around this by guaranteeing per-gateway ordering (gateway ID as key) while allowing different gateways to be processed in parallel and out of order relative to each other
32. How do you guarantee ordering in Kafka?
    - **Answer:**
      - Use the same message key for all related messages — Kafka assigns messages with the same key to the same partition
      - In cold-chain, `gatewayId` was the key, so temperature readings from each gateway were strictly ordered
      - I also set `max.in.flight.requests.per.connection=1` to prevent reordering on producer retries
      - `enable.idempotence=true` enables the producer to assign sequence numbers to each batch, and Kafka's broker automatically deduplicates and reorders on the server side. This means you can safely set `max.in.flight.requests.per.connection > 1` (up to 5) without risking ordering, because out-of-order or duplicate batches are rejected by the broker
33. What is key in Kafka message?
    - **Answer:**
      - The message key determines which partition a message goes to — same key always goes to the same partition
      - In cold-chain, the key was the gateway device ID, which ensured all sensor data from one gateway was in order and also helped with debugging by filtering messages by gateway
      - Null keys use round-robin partition assignment, distributing messages evenly but with no ordering guarantee. I chose gateway ID as the key because I needed per-gateway ordering; if I only needed even distribution without ordering, I would have used a null key or a random key
34. What is schema registry?
    - **Answer:**
      - Schema Registry stores and validates message schemas (Avro/JSON Schema) to ensure compatibility between producers and consumers
      - While we used JSON with a shared contract in cold-chain, Schema Registry would have prevented issues like a producer sending an extra field without the consumer expecting it
      - Backward compatibility means new consumers can read old data; forward compatibility means old consumers can read new data. Schema Registry enforces these rules at produce time — if a producer tries to add a required field that breaks backward compatibility, the registry rejects the schema change, preventing silent downstream failures
35. What is Kafka retention?
    - **Answer:**
      - Retention controls how long Kafka retains messages
      - In cold-chain, I set retention to 7 days for the sensor-readings topic
      - This allowed us to reprocess data if a downstream system failed over the weekend without needing to contact the gateways again
      - Time-based retention deletes messages after a configured duration (e.g., 7 days). Size-based retention keeps the latest messages up to a total size limit per partition. Log compaction (`cleanup.policy=compact`) retains only the latest value per key, useful for stateful streams like changelog topics where you only need the current state
36. Difference between Kafka and RabbitMQ.
    - **Answer:**
      - Kafka is a distributed log optimized for high-throughput event streaming with replay capability
      - RabbitMQ is a traditional message broker with complex routing and priority queues
      - I chose Kafka for cold-chain because the IoT data was high-volume, needed replay, and had multiple consumers, which RabbitMQ handles less efficiently
      - RabbitMQ excels at task distribution with complex routing (exchanges, bindings, priority queues) and deletes messages after acknowledgment — ideal for work queues and RPC. Kafka excels at event streaming with durable, replayable logs — ideal for high-throughput data pipelines. The choice hinges on whether you need message deletion after consumption (RabbitMQ) or log retention with replay (Kafka)
37. Difference between Kafka and SQS.
    - **Answer:**
      - SQS is a fully managed AWS queue with at-least-once delivery and automatic scaling
      - Kafka requires manual cluster management but offers lower latency, higher throughput, and multi-consumer fan-out
      - We used Kafka on EC2 because we needed sub-100ms latency for real-time dashboards and multiple consumer groups — SQS would have added complexity for fan-out
      - SQS wins when you need zero infrastructure management, automatic scaling, and your throughput is moderate (under 10K msg/sec). It's a fully managed service — no brokers to patch, no cluster sizing, no partition planning. However, SQS lacks replay capability and multi-consumer fan-out is limited compared to Kafka
38. Difference between Kafka and MQTT.
    - **Answer:**
      - MQTT is a lightweight pub-sub protocol for IoT devices with limited bandwidth
      - Kafka is a distributed storage and streaming platform
      - In cold-chain, they complemented each other: MQTT carried data from gateways to AWS IoT Core, and Kafka handled the backend streaming from IoT Core to the data stores
      - MQTT QoS 0 is at-most-once (fire and forget), QoS 1 is at-least-once (acknowledged delivery), and QoS 2 is exactly-once (four-part handshake). We chose QoS 1 for MQTT to guarantee gateway messages reached IoT Core, paired with Kafka's at-least-once delivery — the end-to-end pipeline lost zero messages while avoiding the overhead of QoS 2
39. When would you not use Kafka?
    - **Answer:**
      - I would not use Kafka for simple task queues, low-throughput (< 100 msg/sec) scenarios, or when you need exactly-once delivery without idempotent consumers
      - Also, if the team lacks operational experience with Kafka, the learning curve and operational overhead can be significant
      - For example, if the cold-chain system only had 10 temperature readings per minute and no replay requirement, a simple REST API from the gateway to a Lambda function writing to InfluxDB would eliminate the need for a Kafka cluster entirely — saving EC2 costs, operational complexity, and the team's maintenance burden
40. How did you use Kafka in your cold-chain project?
    - **Answer:**
      - Kafka was the central message backbone
      - IoT gateways sent temperature data via MQTT to AWS IoT Core, which forwarded it to Lambda
      - Lambda published the data to our Kafka topic (`sensor-readings`) running on EC2
      - A Spring Boot consumer read from Kafka, validated readings, and stored them in InfluxDB
      - A separate consumer handled threshold-based alerting
      - Full architecture: IoT Gateways publish MQTT → AWS IoT Core (MQTT broker) → Lambda function (Kafka producer) → Kafka cluster on EC2 (3 brokers, `sensor-readings` topic with 6 partitions) → Spring Boot consumer (validates and writes to InfluxDB) → Grafana reads from InfluxDB for dashboards. A second consumer handles threshold-based alerting and sends notifications via SNS
41. Why was Kafka useful for IoT data?
    - **Answer:**
      - IoT data is high-volume, continuous, and comes from many sources
      - Kafka's log-based architecture absorbed bursty sensor data without backpressure, provided a durable buffer that prevented data loss during downstream outages, and allowed multiple consumers (storage, alerting, analytics) to process the same stream independently
      - When InfluxDB went down for 30 minutes during a maintenance window, Kafka's 7-day retention buffered all sensor data. When InfluxDB came back, the consumer simply resumed from its committed offset and caught up — zero data loss. With a point-to-point integration (like direct MQTT to InfluxDB), those 30 minutes of data would have been permanently lost
42. How do you handle sensor data bursts in Kafka?
    - **Answer:**
      - Kafka naturally handles bursts because it's disk-based — producers can write faster than consumers can read
      - However, to prevent unbounded lag, I sized the partition count and consumer instances to handle peak throughput
      - I also set producer `max.block.ms` and consumer `fetch.max.bytes` appropriately to avoid memory issues during bursts
      - Kafka producers experience no backpressure — they always write at full speed to the log. Consumers apply backpressure by calling `consumer.pause()` on assigned partitions when the downstream system (e.g., InfluxDB) is slow, preventing unbounded lag. When the downstream recovers, `consumer.resume()` restarts consumption
43. How do you scale Kafka consumers?
    - **Answer:**
      - You scale consumers by adding more consumer instances to the same consumer group
      - The maximum parallelism equals the number of partitions
      - In cold-chain, when we increased from 3 to 6 gateways, I increased the partition count from 3 to 6 and added 3 more consumer instances, distributing the load
      - Increasing partitions after topic creation adds new empty partitions — existing messages remain in the old partitions and aren't redistributed. This means you can't reduce partition count later, and uneven message distribution can occur. Plan partition count upfront based on projected peak throughput and desired consumer parallelism
44. How do you secure Kafka?
    - **Answer:**
      - Kafka can be secured with SSL/TLS for encryption, SASL for authentication, and ACLs for authorization
      - In the cold-chain project, since Kafka ran on EC2 within a VPC and was accessed only by internal services, we used security groups for network isolation rather than enabling Kafka-level SSL
      - SASL/SCRAM provides username-password authentication: create JAAS config with credentials, configure `security.protocol=SASL_SSL`, and add ACLs to restrict topic access per user. Always encrypt data in transit with SSL/TLS if Kafka is accessible outside the VPC — without it, message payloads are sent in plaintext and vulnerable to interception
45. What is Kafka Connect?
    - **Answer:**
      - Kafka Connect is a framework for streaming data between Kafka and external systems using connectors
      - I evaluated the InfluxDB Sink Connector for cold-chain to replace the custom Spring Boot consumer, but the connector didn't support all our validation logic, so I kept the custom consumer
      - Source connectors push data from external systems (databases, logs, APIs) into Kafka topics. Sink connectors pull data from Kafka topics and write to external systems (databases, search indices, file systems). Both run as distributed or standalone workers, eliminating the need to write custom producer/consumer code for common integrations
46. What is Kafka Streams?
    - **Answer:**
      - Kafka Streams is a Java library for building stream processing applications on top of Kafka
      - In cold-chain, I considered using Kafka Streams for the sliding-window temperature average calculation, but implemented it in the Spring Boot consumer instead due to familiarity
      - Stateful operations include windowed aggregations (tumbling, hopping, session windows) and stream-table joins. Kafka Streams stores local state in RocksDB on disk for fast reads, and backs up state changes to changelog topics in Kafka. On failure, the state is rebuilt by replaying the changelog topic, providing fault-tolerant state management without external databases
47. What is backpressure?
    - **Answer:**
      - Backpressure is a feedback mechanism where a slow consumer signals the producer to slow down
      - Kafka doesn't have traditional backpressure — producers always write at full speed, and consumers fall behind if they can't keep up
      - In cold-chain, the downstream InfluxDB write path created backpressure on the consumer, causing lag
      - I added a bounded queue between the Kafka poll loop and the processing thread pool. When the queue filled up (indicating the downstream InfluxDB was slow), I called `consumer.pause()` to stop fetching new messages. When the queue drained below a threshold, I called `consumer.resume()` to restart consumption, preventing unbounded memory growth and consumer lag
48. How do you handle backpressure?
    - **Answer:**
      - In Kafka, you handle backpressure at the consumer level by pausing partition assignment when the processing pipeline is saturated
      - In cold-chain, when InfluxDB had a write slowdown, the consumer paused all partitions, waited for pending writes to complete, and then resumed consuming
      - An alternative approach uses a bounded queue between the poll loop and processing threads — the poll loop blocks when the queue is full, naturally throttling consumption. This mirrors reactive streams backpressure (like Reactive Kafka) where the consumer signals demand upstream, but Kafka's native approach uses partition pausing instead of pull-based demand signaling
49. How do you test Kafka consumers?
    - **Answer:**
      - I test Kafka consumers using embedded Kafka with Spring Kafka's `@EmbeddedKafka` annotation in integration tests
      - In cold-chain, I wrote tests that published sample sensor data to an embedded topic, verified the consumer processed it, and checked the data landed in a test InfluxDB instance
      - Beyond happy-path tests, I tested failure scenarios: simulating broker failures to verify consumer recovery, triggering rebalances by adding/removing consumer instances, sending malformed JSON to verify DLT routing, and asserting that offsets were committed only after successful processing to guarantee no data loss
50. How do you debug message loss?
    - **Answer:**
      - I start by checking consumer lag — if lag is zero, messages were consumed
      - Then I check the consumer's committed offset against the latest offset
      - I also enable producer-side logging with `errors.tolerance=all` and check the DLT
      - In cold-chain, one message loss was caused by a consumer crash between processing and offset commit
      - Systematic debugging approach: (1) check producer `acks` config and broker logs for write failures, (2) check consumer error logs for deserialization or processing exceptions, (3) compare consumer committed offset vs partition end offset to find gaps, (4) replay the topic from the suspected gap offset to verify the message exists in the log. The key insight is knowing the delivery semantic (at-least-once vs at-most-once) and checking each layer systematically
