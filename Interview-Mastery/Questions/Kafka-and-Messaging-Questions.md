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
   - **If asked more:**
      - I would explain the publish-subscribe model and contrast Kafka with traditional message queues
      - Kafka's log-based storage allows replay, which was critical for backfilling historical temperature data
2. Why use Kafka?
   - **Answer:**
      - Kafka handles high throughput, provides durability through disk-based logs, and allows multiple consumers to process the same stream independently
      - We chose Kafka in the cold-chain project because IoT gateways could burst thousands of sensor readings per second and we needed a buffer that could absorb the spikes without data loss
   - **If asked more:**
      - I would compare Kafka with alternatives (RabbitMQ, SQS) and explain why Kafka's replay capability was essential for debugging temperature excursion incidents
3. What is event streaming?
   - **Answer:**
      - Event streaming captures data in real-time as a sequence of events
      - In the cold-chain system, every temperature reading from each sensor was an event published to Kafka, creating an immutable log that we could process, analyze, and visualize in real-time on the Grafana dashboard
   - **If asked more:**
      - I would explain how event streaming differs from request-response
      - Events are persisted, can be replayed, and allow multiple independent consumers to process the same data
4. What is producer?
   - **Answer:**
      - A producer publishes messages to Kafka topics
      - In our cold-chain pipeline, an AWS Lambda function acted as the producer — it received MQTT messages from AWS IoT Core and published them to the `sensor-readings` Kafka topic on our EC2-hosted Kafka cluster
   - **If asked more:**
      - I would discuss producer configuration — how I set `acks=all` to ensure no data loss and how I tuned `linger.ms` and `batch.size` for throughput
5. What is consumer?
   - **Answer:**
      - A consumer subscribes to topics and processes messages
      - In the cold-chain project, our Spring Boot application was the consumer — it read temperature readings from Kafka, validated them, and stored them in InfluxDB for time-series analysis
   - **If asked more:**
      - I would explain consumer offset management, auto-commit vs manual commit, and how I configured the consumer for at-least-once delivery with idempotent processing
6. What is topic?
   - **Answer:**
      - A topic is a logical channel where producers send messages and consumers read from
      - In the cold-chain system, I organized topics by data type: `sensor-readings` for temperature data, `sensor-alerts` for threshold violations, and `sensor-health` for gateway heartbeat messages
   - **If asked more:**
      - I would explain how I used topic naming conventions and considered the number of topics based on data volume and consumer isolation requirements
7. What is partition?
   - **Answer:**
      - A partition is a unit of parallelism within a topic — messages are distributed across partitions, which are ordered and immutable
      - I configured the `sensor-readings` topic with 6 partitions to match the 6 IoT gateways, using the gateway ID as the message key to ensure all readings from one gateway went to the same partition
   - **If asked more:**
      - I would explain how partition count affects consumer parallelism — more partitions allow more consumers, but too many increase overhead and rebalancing time
8. What is broker?
   - **Answer:**
      - A broker is a Kafka server that stores data and serves clients
      - We ran a 3-broker Kafka cluster on EC2 instances, which gave us fault tolerance — if one broker failed, the others continued serving with replicated data
   - **If asked more:**
      - I would explain how we sized the brokers based on expected throughput and retention — we calculated disk space based on 7 days of sensor data at 1000 msg/sec with a replication factor of 3
9. What is consumer group?
   - **Answer:**
      - A consumer group is a set of consumers that collectively read from a topic, with each partition assigned to one consumer
      - In cold-chain, we had multiple consumer instances in the same group to parallelize processing of sensor data from different gateways
   - **If asked more:**
      - I would explain consumer group rebalancing — what happens when a consumer joins or leaves, and how we tuned `session.timeout.ms` to reduce unnecessary rebalances during GC pauses
10. How does Kafka scale?
    - **Answer:**
      - Kafka scales horizontally by adding brokers to the cluster and partitions to topics
      - More partitions allow more consumers in a group to process in parallel
      - In cold-chain, when we added two more gateways, I increased the partition count and added consumer instances to handle the increased load
    - **If asked more:**
      - I would explain the difference between scaling producers (they write to any partition) and consumers (limited by partition count), and how I used Kafka's built-in rebalancing to distribute load automatically
11. Why are partitions important?
    - **Answer:**
      - Partitions enable parallelism — multiple consumers can read different partitions concurrently
      - Without partitions, we'd have a single sequential stream
      - In cold-chain, 6 partitions meant 6 consumers could process temperature data in parallel, keeping the pipeline responsive even during sensor bursts
    - **If asked more:**
      - I would explain how partitions provide ordering guarantees (within a partition) and how the number of partitions is the maximum parallelism for consumers
12. How do you decide partition count?
    - **Answer:**
      - I base partition count on the expected throughput, number of consumers, and key cardinality
      - For cold-chain, I started with 6 partitions — one per gateway — and monitored consumer lag
      - I planned to increase to 12 if lag grew, but 6 was sufficient for our 1000 msg/sec peak
    - **If asked more:**
      - I would share the formula: partitions should be at least max(target throughput / partition throughput, number of consumers)
      - I also consider that too many partitions increase ZooKeeper overhead and rebalancing time
13. What is offset?
    - **Answer:**
      - An offset is a sequential ID assigned to each message within a partition, representing its position
      - Offsets allow consumers to track how far they've read
      - In the cold-chain pipeline, if a consumer crashed, it could resume from its last committed offset without missing or duplicating messages
    - **If asked more:**
      - I would explain how offset commits work — auto-commit at `enable.auto.commit=true` with `auto.commit.interval.ms`, and why I switched to manual commits for exactly-once semantics in critical alert processing
14. What is committed offset?
    - **Answer:**
      - A committed offset is the last offset a consumer has successfully processed and saved to Kafka's internal `__consumer_offsets` topic
      - When the inventory consumer crashed during a peak load, the committed offset ensured it resumed from where it left off rather than re-processing thousands of messages
    - **If asked more:**
      - I would explain the risk of committing before processing (message loss on crash) vs after processing (duplicate reprocessing), and how I accepted duplicates in favor of zero data loss
15. What is consumer lag?
    - **Answer:**
      - Consumer lag is the difference between the latest offset in a partition and the consumer's committed offset
      - When our InfluxDB write path had a slowdown, I saw lag spike to 50,000 messages
      - That told me the consumer was falling behind and couldn't keep up with the producer rate
    - **If asked more:**
      - I would explain how I monitored lag using `kafka-consumer-groups` CLI and set up CloudWatch alarms on lag metrics to get paged before the pipeline fell too far behind
16. How do you monitor consumer lag?
    - **Answer:**
      - I used the `kafka-consumer-groups --bootstrap-server --describe --group` command to check lag per partition
      - In cold-chain, I automated this by publishing lag metrics to CloudWatch every minute and set a threshold alarm at 10,000 messages lag for immediate investigation
    - **If asked more:**
      - I would also mention Burrow, a LinkedIn tool for lag monitoring, and how lag trends (increasing vs stable) tell you whether the consumer is catching up or falling further behind
17. What is replication factor?
    - **Answer:**
      - Replication factor determines how many copies of each partition exist across brokers
      - In cold-chain, I set replication factor to 3 for the sensor-readings topic so that if one EC2 broker instance went down, no data was lost and the cluster continued serving
    - **If asked more:**
      - I would explain the trade-off: higher replication provides better durability but uses more disk and network bandwidth
      - RF=3 with min.insync.replicas=2 was our standard for production
18. What is leader and follower replica?
    - **Answer:**
      - For each partition, one broker is the leader that handles all reads and writes, and the rest are in-sync followers
      - When the leader broker for our sensor-readings partition went down for maintenance, a follower automatically became the new leader with zero data loss
    - **If asked more:**
      - I would explain how Kafka handles leader election and how the preferred replica configuration helps redistribute leadership after a failed broker recovers
19. What is ISR?
    - **Answer:**
      - ISR (In-Sync Replicas) are followers that are fully caught up with the leader
      - I set `min.insync.replicas=2` for cold-chain topics to ensure that at least 2 brokers acknowledged each write
      - This prevented data loss even if one broker failed after acknowledging
    - **If asked more:**
      - I would explain what happens when a follower falls out of ISR (slow network, GC pause) and how it catches up using truncated replication
20. What happens when a broker fails?
    - **Answer:**
      - When a broker fails, the controllers detect it, partition leaders on that broker are reassigned to ISR followers on other brokers
      - During one EC2 instance failure in cold-chain, the Kafka cluster automatically reassigned the leadership within seconds, and producers/consumers continued with minimal interruption
    - **If asked more:**
      - I would explain the difference between a clean shutdown (controlled leader migration) vs a hard failure (unclean leader election if no ISR is available), and how we handled each case
21. What is acknowledgement in Kafka?
    - **Answer:**
      - Acknowledgement (acks) determines when the producer considers a write successful
      - In cold-chain, I used `acks=all` for the temperature topic because losing a reading was unacceptable — the producer waited for all in-sync replicas to acknowledge before moving on
    - **If asked more:**
      - I would explain the performance impact of `acks=all` (higher latency, guaranteed durability) and how we tuned `max.in.flight.requests.per.connection` to prevent ordering issues
22. Difference between `acks=0`, `acks=1`, and `acks=all`.
    - **Answer:**
      - `acks=0` fires and forgets (fastest, highest risk)
      - `acks=1` waits for the leader only (balanced)
      - `acks=all` waits for all ISRs (safest, slower)
      - I used `acks=all` for sensor data and `acks=1` for non-critical health check messages
    - **If asked more:**
      - I would explain how these interact with `min.insync.replicas` and why `acks=all` with `min.insync.replicas=2` still works even if one broker is down
23. What is at-most-once delivery?
    - **Answer:**
      - At-most-once means a message is delivered zero or one time — if the consumer fails after fetching but before processing, the message is lost
      - In cold-chain, I avoided this for temperature data because losing a reading could mean missing a temperature excursion
    - **If asked more:**
      - I would explain the configuration: `enable.auto.commit=true` with commits before processing, combined with `acks=0` on the producer side
24. What is at-least-once delivery?
    - **Answer:**
      - At-least-once means messages can be delivered more than once but never lost
      - In cold-chain, I used at-least-once by setting `enable.auto.commit=false` and manually committing offsets only after successful processing and storage in InfluxDB
    - **If asked more:**
      - I would explain that this requires idempotent consumers (since duplicates can happen), and how I implemented deduplication using event IDs in the consumer
25. What is exactly-once semantics?
    - **Answer:**
      - Exactly-once semantics (EOS) ensures each message is processed exactly once, with no duplicates and no losses
      - While Kafka supports EOS with transactions and idempotent producers, I used at-least-once with idempotent consumers instead because the setup complexity was higher than the duplicate probability
    - **If asked more:**
      - I would explain the trade-offs: EOS adds latency and requires transactional coordination, so it's worth it for financial systems but overkill for IoT sensor data where occasional duplicates are tolerable
26. How do you handle duplicate messages?
    - **Answer:**
      - I handle duplicates by making consumers idempotent — either by tracking processed event IDs in Redis or using database unique constraints
      - In cold-chain, each sensor reading had a unique ID (gatewayID + timestamp), and I used a unique constraint on that in InfluxDB to prevent duplicate storage
    - **If asked more:**
      - I would explain the difference between producer-side duplicates (from retries) and consumer-side duplicates (from rebalancing), and how I handled both
27. How do you make consumers idempotent?
    - **Answer:**
      - Idempotent consumers produce the same result regardless of how many times a message is processed
      - In the cold-chain consumer, I checked if the unique event ID already existed in the target table before inserting, and skipped duplicates
      - This made the entire pipeline safe to retry during failures
    - **If asked more:**
      - I would discuss the trade-off between exactly-once (Kafka transactions) and idempotent consumers, and why I chose idempotency for simplicity
28. What is dead-letter topic?
    - **Answer:**
      - A dead-letter topic (DLT) stores messages that consumers cannot process after repeated retries
      - In cold-chain, I configured a DLT for the alert-processing topic — if a sensor reading failed validation after 3 retries, it was moved to the DLT for manual inspection rather than blocking the consumer stream
    - **If asked more:**
      - I would explain how I set up the DLT with a TTL and automated alerting to notify the team when messages landed there, so we could investigate the root cause
29. What is retry topic?
    - **Answer:**
      - A retry topic holds messages that failed temporarily so they can be reprocessed after a delay
      - In cold-chain, when InfluxDB was temporarily unavailable, the consumer moved the message to a retry topic with a 10-second delay before the next attempt, preventing a tight retry loop
    - **If asked more:**
      - I would explain the retry topic architecture: main topic → consumer → retry topic → retry consumer → DLT (if still failing), and how to set the retry delay
30. How do you handle poison messages?
    - **Answer:**
      - Poison messages are messages that always cause consumer failures
      - In cold-chain, a malformed JSON payload from a faulty gateway once caused continuous deserialization errors
      - I handled it by wrapping the deserialization in a try-catch and sending the bad message to a DLT with the error details logged
    - **If asked more:**
      - I would explain how to detect poison messages — repeated failures on the same offset — and why automated DLT routing is critical to prevent the consumer from being stuck in a rebalance loop
31. What is message ordering?
    - **Answer:**
      - Message ordering guarantees that messages are processed in the order they were produced
      - Kafka guarantees order within a partition, not across partitions
      - In cold-chain, I used the gateway ID as the message key so all readings from one gateway went to the same partition, preserving per-gateway ordering
    - **If asked more:**
      - I would explain the trade-off: strong ordering limits parallelism (key → single partition), so I designed around it by keeping per-gateway ordering but allowing different gateways to be processed out of order
32. How do you guarantee ordering in Kafka?
    - **Answer:**
      - Use the same message key for all related messages — Kafka assigns messages with the same key to the same partition
      - In cold-chain, `gatewayId` was the key, so temperature readings from each gateway were strictly ordered
      - I also set `max.in.flight.requests.per.connection=1` to prevent reordering on producer retries
    - **If asked more:**
      - I would explain the impact of `enable.idempotence=true` on ordering — Kafka internally handles ordering with idempotent producers without `max.in.flight.requests=1`
33. What is key in Kafka message?
    - **Answer:**
      - The message key determines which partition a message goes to — same key always goes to the same partition
      - In cold-chain, the key was the gateway device ID, which ensured all sensor data from one gateway was in order and also helped with debugging by filtering messages by gateway
    - **If asked more:**
      - I would explain how null keys cause round-robin distribution across partitions (no ordering guarantee), and how I chose keys based on ordering vs. load-balancing needs
34. What is schema registry?
    - **Answer:**
      - Schema Registry stores and validates message schemas (Avro/JSON Schema) to ensure compatibility between producers and consumers
      - While we used JSON with a shared contract in cold-chain, Schema Registry would have prevented issues like a producer sending an extra field without the consumer expecting it
    - **If asked more:**
      - I would explain backward/forward compatibility and how Schema Registry helps with evolution — a producer can add optional fields without breaking existing consumers
35. What is Kafka retention?
    - **Answer:**
      - Retention controls how long Kafka retains messages
      - In cold-chain, I set retention to 7 days for the sensor-readings topic
      - This allowed us to reprocess data if a downstream system failed over the weekend without needing to contact the gateways again
    - **If asked more:**
      - I would explain the difference between time-based retention and size-based retention, and how log compaction retains only the latest message per key for stateful streams
36. Difference between Kafka and RabbitMQ.
    - **Answer:**
      - Kafka is a distributed log optimized for high-throughput event streaming with replay capability
      - RabbitMQ is a traditional message broker with complex routing and priority queues
      - I chose Kafka for cold-chain because the IoT data was high-volume, needed replay, and had multiple consumers, which RabbitMQ handles less efficiently
    - **If asked more:**
      - I would argue that RabbitMQ is better for task distribution (commands, work queues) while Kafka is better for event streaming, and the choice depends on whether you need message deletion after consumption or log retention
37. Difference between Kafka and SQS.
    - **Answer:**
      - SQS is a fully managed AWS queue with at-least-once delivery and automatic scaling
      - Kafka requires manual cluster management but offers lower latency, higher throughput, and multi-consumer fan-out
      - We used Kafka on EC2 because we needed sub-100ms latency for real-time dashboards and multiple consumer groups — SQS would have added complexity for fan-out
    - **If asked more:**
      - I would explain when SQS wins: simpler setup, no infrastructure management, automatic scaling, and when throughput requirements are moderate (under 10K msg/sec)
38. Difference between Kafka and MQTT.
    - **Answer:**
      - MQTT is a lightweight pub-sub protocol for IoT devices with limited bandwidth
      - Kafka is a distributed storage and streaming platform
      - In cold-chain, they complemented each other: MQTT carried data from gateways to AWS IoT Core, and Kafka handled the backend streaming from IoT Core to the data stores
    - **If asked more:**
      - I would explain how MQTT's QoS levels (0, 1, 2) map to Kafka's delivery semantics, and why we chose QoS 1 (at-least-once) for MQTT combined with Kafka's at-least-once for end-to-end reliability
39. When would you not use Kafka?
    - **Answer:**
      - I would not use Kafka for simple task queues, low-throughput (< 100 msg/sec) scenarios, or when you need exactly-once delivery without idempotent consumers
      - Also, if the team lacks operational experience with Kafka, the learning curve and operational overhead can be significant
    - **If asked more:**
      - I would give a concrete example: if we only had to process 10 temperature readings per minute and didn't need replay, a simple REST API call would be simpler than running a Kafka cluster on EC2
40. How did you use Kafka in your cold-chain project?
    - **Answer:**
      - Kafka was the central message backbone
      - IoT gateways sent temperature data via MQTT to AWS IoT Core, which forwarded it to Lambda
      - Lambda published the data to our Kafka topic (`sensor-readings`) running on EC2
      - A Spring Boot consumer read from Kafka, validated readings, and stored them in InfluxDB
      - A separate consumer handled threshold-based alerting
    - **If asked more:**
      - I would draw the complete architecture: Gateway → IoT Core → Lambda (producer) → Kafka (brokers on EC2) → Spring Boot (consumer) → InfluxDB + another consumer → Grafana alerts
41. Why was Kafka useful for IoT data?
    - **Answer:**
      - IoT data is high-volume, continuous, and comes from many sources
      - Kafka's log-based architecture absorbed bursty sensor data without backpressure, provided a durable buffer that prevented data loss during downstream outages, and allowed multiple consumers (storage, alerting, analytics) to process the same stream independently
    - **If asked more:**
      - I would mention that if InfluxDB went down for 30 minutes, Kafka's 7-day retention meant zero data loss — the consumer just caught up when InfluxDB was back, which wouldn't be possible with a point-to-point integration
42. How do you handle sensor data bursts in Kafka?
    - **Answer:**
      - Kafka naturally handles bursts because it's disk-based — producers can write faster than consumers can read
      - However, to prevent unbounded lag, I sized the partition count and consumer instances to handle peak throughput
      - I also set producer `max.block.ms` and consumer `fetch.max.bytes` appropriately to avoid memory issues during bursts
    - **If asked more:**
      - I would explain the concept of backpressure in Kafka — it doesn't apply to producers (they always write), but consumers can apply backpressure by pausing partitions via `consumer.pause()` when downstream systems are slow
43. How do you scale Kafka consumers?
    - **Answer:**
      - You scale consumers by adding more consumer instances to the same consumer group
      - The maximum parallelism equals the number of partitions
      - In cold-chain, when we increased from 3 to 6 gateways, I increased the partition count from 3 to 6 and added 3 more consumer instances, distributing the load
    - **If asked more:**
      - I would warn that increasing partitions after topic creation creates a new partition assignment, and existing messages aren't redistributed — so plan the partition count upfront based on projected growth
44. How do you secure Kafka?
    - **Answer:**
      - Kafka can be secured with SSL/TLS for encryption, SASL for authentication, and ACLs for authorization
      - In the cold-chain project, since Kafka ran on EC2 within a VPC and was accessed only by internal services, we used security groups for network isolation rather than enabling Kafka-level SSL
    - **If asked more:**
      - I would explain how to set up SASL/SCRAM for username-password auth, and why you should always encrypt data in transit if Kafka is accessible outside the VPC
45. What is Kafka Connect?
    - **Answer:**
      - Kafka Connect is a framework for streaming data between Kafka and external systems using connectors
      - I evaluated the InfluxDB Sink Connector for cold-chain to replace the custom Spring Boot consumer, but the connector didn't support all our validation logic, so I kept the custom consumer
    - **If asked more:**
      - I would explain the difference between source connectors (push data into Kafka) and sink connectors (pull data from Kafka), and how they simplify integration without writing producer/consumer code
46. What is Kafka Streams?
    - **Answer:**
      - Kafka Streams is a Java library for building stream processing applications on top of Kafka
      - In cold-chain, I considered using Kafka Streams for the sliding-window temperature average calculation, but implemented it in the Spring Boot consumer instead due to familiarity
    - **If asked more:**
      - I would explain stateful operations (windowed aggregations, joins) and how Kafka Streams handles state using RocksDB for local state stores with changelog topics for fault tolerance
47. What is backpressure?
    - **Answer:**
      - Backpressure is a feedback mechanism where a slow consumer signals the producer to slow down
      - Kafka doesn't have traditional backpressure — producers always write at full speed, and consumers fall behind if they can't keep up
      - In cold-chain, the downstream InfluxDB write path created backpressure on the consumer, causing lag
    - **If asked more:**
      - I would explain how I handled this by adding a processing buffer with bounded queues in the consumer and using `consumer.pause()` when the buffer was full, resuming when it drained
48. How do you handle backpressure?
    - **Answer:**
      - In Kafka, you handle backpressure at the consumer level by pausing partition assignment when the processing pipeline is saturated
      - In cold-chain, when InfluxDB had a write slowdown, the consumer paused all partitions, waited for pending writes to complete, and then resumed consuming
    - **If asked more:**
      - I would explain the alternative approach: using a bounded queue between the poll loop and the processing thread pool, and comparing Kafka's approach to reactive streams backpressure
49. How do you test Kafka consumers?
    - **Answer:**
      - I test Kafka consumers using embedded Kafka with Spring Kafka's `@EmbeddedKafka` annotation in integration tests
      - In cold-chain, I wrote tests that published sample sensor data to an embedded topic, verified the consumer processed it, and checked the data landed in a test InfluxDB instance
    - **If asked more:**
      - I would also mention testing failure scenarios: broker failure, consumer rebalancing, poison messages, and verifying that offsets are committed correctly after processing
50. How do you debug message loss?
    - **Answer:**
      - I start by checking consumer lag — if lag is zero, messages were consumed
      - Then I check the consumer's committed offset against the latest offset
      - I also enable producer-side logging with `errors.tolerance=all` and check the DLT
      - In cold-chain, one message loss was caused by a consumer crash between processing and offset commit
    - **If asked more:**
      - I would describe a systematic debugging approach: check producer acks, consumer error logs, offset commits, and then replay the topic to verify
      - The key insight is knowing the delivery semantic and checking each layer
