# Distributed Transactions

## Overview

- **Definition** — A distributed transaction is a set of operations spanning multiple independent databases, services, or nodes that must be executed atomically, ensuring all participants commit or all roll back.
- **Why It Exists** — In a monolithic application, a single ACID transaction guarantees atomicity across tables; in microservices, each service owns its own database, so traditional database transactions cannot span services — distributed transaction protocols restore atomicity across process and network boundaries.
- **Historical Context** — X/Open XA standard (1992) defined two-phase commit for distributed databases; the rise of microservices in the 2010s led to the realisation that 2PC is too slow and blocking for high-throughput systems, popularising Sagas (Hector Garcia-Molina, 1987) and the Outbox pattern.
- **Key Concepts** — **Atomicity** — all-or-nothing execution across participants; **Coordinator** — entity managing the transaction lifecycle; **Prepare Phase** — each participant votes on whether it can commit; **Commit Phase** — coordinator instructs all participants to commit or abort; **Compensating Transaction** — action that semantically undoes a previous transaction; **Idempotency** — property ensuring repeated execution produces the same result.

## Core Concepts

- **Two-Phase Commit (2PC)**
  - **Phase 1 (Prepare):** The coordinator sends a `prepare` request to all participants. Each participant writes a prepare record to its transaction log, acquires locks on the affected resources, and replies with a vote (yes/abort).
  - **Phase 2 (Commit or Abort):** If all participants voted yes, the coordinator sends a `commit` request; each participant writes a commit record, releases locks, and acknowledges. If any participant voted abort (or timed out), the coordinator sends an `abort` request; each participant rolls back and releases locks.
  - **Blocking:** Participants that voted yes hold locks until they receive the commit/abort decision; if the coordinator crashes, participants remain blocked indefinitely (in-doubt transactions).
  - **Coordinator Failure:** If the coordinator crashes mid-prepare, participants remain locked until the coordinator recovers and queries its transaction log.
  - **XA Protocol** — the standard for distributed transaction coordination in JTA, supported by most relational databases (Oracle, PostgreSQL, MySQL) and JMS brokers.

- **Saga Pattern**
  - A saga is a sequence of local transactions where each step publishes an event or invokes the next step; if a step fails, compensating transactions undo the previously completed steps.
  - **Choreography:** Each service, after completing its local transaction, emits an event that triggers the next service. No central coordinator. Simple but debugging is difficult because the flow is distributed across services.
  - **Orchestration:** A central orchestrator (saga execution engine) tells each service what to do and handles compensating transactions. Easier to monitor and manage, but the orchestrator is a single point of failure and can become a bottleneck.
  - **Compensating Transactions:** Semantically opposite of the forward action (e.g., "refund payment" compensates "charge payment"). Compensations must be idempotent and may fail themselves — the saga must handle compensation failure (retry, alert, manual intervention).

- **TCC (Try-Confirm/Cancel)**
  - **Try:** Reserve resources; put the system in a pending state. The reservation is not yet committed but prevents other transactions from using the same resource.
  - **Confirm:** If all Try operations succeed, each participant converts its reservation into a confirmed state. This phase should not fail (all checks were done in Try).
  - **Cancel:** If any Try fails, each participant releases its reserved resources.
  - TCC is more performant than 2PC because the Confirm phase is expected to succeed and does not require coordination; but it requires participants to implement stateful reservation logic.

- **Outbox Pattern**
  - Instead of writing to the database and sending a message in separate operations (which risks inconsistency), the service writes both the domain entity and an outbox event record in a single local ACID transaction.
  - A separate message relay process reads the outbox table and publishes events to the message broker (Kafka, RabbitMQ).
  - The relay can poll the database periodically or use a database log tail (CDC) to capture changes.
  - Guarantees at-least-once delivery; consumers must be idempotent to handle duplicate messages.
  - Avoids the distributed transaction problem: the database transaction is local, and the message delivery is handled by a reliable relay.

- **Change Data Capture (CDC)**
  - Captures changes from a database transaction log (WAL, binlog, transaction log) and streams them to downstream consumers.
  - **Debezium** — open-source CDC platform that connects to PostgreSQL (pgoutput), MySQL (binlog), MongoDB (oplog), SQL Server (CDC tables), and others; outputs change events to Kafka.
  - **Kafka Connect** — framework for streaming data between Kafka and other systems; Debezium runs as a Kafka Connect source connector.
  - Each change event includes the before/after image of the row, the operation type (create/update/delete), and metadata (timestamp, transaction ID).
  - CDC is the backbone of the Outbox pattern when combined with a transactional outbox table — the relay reads outbox events from the database transaction log.

- **When to Avoid Distributed Transactions**
  - If the operations can be redesigned to be eventually consistent, avoid distributed transactions entirely.
  - If the data entities can be co-located in the same service and same database, do that instead.
  - If a saga can replace 2PC with acceptable consistency guarantees, prefer the saga.
  - Use distributed transactions only when strong atomicity is required across heterogeneous systems (e.g., a payment must hit both a ledger database and an external payment gateway).

## Common Mistakes

- **2PC coordinator crashing mid-prepare**
  - The coordinator crashes after sending prepare to some participants but before recording the decision. Participants that voted yes hold locks indefinitely, blocking other transactions against those resources.
  - **Why it looks correct:** 2PC guarantees atomicity — but it does not guarantee availability. The blocking nature is inherent, and teams often miss configuring a transaction timeout.
  - The fix: Use a distributed transaction manager with high availability (dedicated cluster, replicated transaction log). Configure a transaction timeout on all participants so in-doubt transactions are rolled back after a configurable period. Implement a recovery daemon that polls for stuck transactions.

- **Saga compensating transaction failure**
  - After the Saga orchestrator invokes a compensating transaction (e.g., refund payment), the compensating operation itself fails (e.g., the payment gateway is down).
  - **Why it looks correct:** Sagas are designed to handle failures via compensation, but the compensation can fail, leaving the system in an inconsistent state.
  - The fix: Compensating transactions must be idempotent and designed to eventually succeed. Implement retry with exponential backoff and dead-letter queues. When retries are exhausted, trigger an alert for manual intervention and log the full saga state.

- **Outbox relay duplication**
  - The outbox relay publishes the same event twice (e.g., due to a crash after sending to Kafka but before acknowledging the read), causing duplicate processing downstream.
  - **Why it looks correct:** The outbox pattern uses at-least-once delivery for reliability, but downstream services are often not idempotent.
  - The fix: Each outbox event should have a unique ID (UUID). Downstream consumers deduplicate by this ID (store processed IDs and ignore duplicates). The relay itself should use transactional offsets (Kafka Connect stores offsets in Kafka) to minimise duplicates.

- **2PC used where Saga would suffice**
  - Teams implement 2PC for all inter-service operations because they are used to ACID transactions, ignoring the performance and availability costs.
  - **Why it looks correct:** 2PC guarantees strong atomicity, which feels safer than eventual consistency. But it introduces blocking, reduces throughput, and increases latency.
  - The fix: Analyse the business requirement — does the operation need immediate atomicity, or is eventual consistency acceptable? Most business workflows (order placement, account transfer) can tolerate a short window of inconsistency.

## Real-World Scenarios

### Payment Gateway 2PC Failure

- An airline booking system used 2PC (XA) between the booking database and the payment gateway. During a flash sale, the coordinator crashed due to memory pressure.
- All in-flight transactions were stuck in the prepared state. The booking database had 2,000 locked rows, blocking new bookings for 15 minutes until the coordinator restarted and resolved the log.
- They migrated to a Saga pattern: book the seat, then process payment. If payment fails, cancel the booking (compensating transaction). This eliminated blocking and improved throughput by 10x.

### E-commerce Order Saga Outage

- A large e-commerce platform used an orchestrated Saga for order processing: Reserve Inventory → Charge Payment → Ship Order.
- The payment service was down for 3 minutes. The Saga orchestrator correctly invoked the compensating transaction (Release Inventory). But the inventory compensation service had a database deadlock, causing the inventory to remain reserved for orders that were not actually charged.
- The fix: Add retry with exponential backoff to the compensation step, and a periodic inventory reconciliation job that releases reservations older than 15 minutes.

### Outbox + CDC at a Fintech Startup

- A fintech startup used the Outbox pattern with Debezium and Kafka Connect to synchronise the `accounts` service (PostgreSQL) with the `transactions` service.
- Initially they implemented dual writes (write to accounts DB and publish to Kafka), which caused frequent inconsistencies (writes succeeded but publishes failed).
- Switching to the transactional outbox (write the outbox event in the same DB transaction as the account update) eliminated 99% of data inconsistency incidents. Debezium streamed the outbox events from the WAL to Kafka with sub-second latency.

## Use Cases

- Distributed transactions are necessary when atomicity must span services, but they come with significant trade-offs. These patterns guide when to use each approach and when to avoid distributed transactions altogether.

- **Financial transactions across services** — ensuring atomicity for money movement between account services
  - When to use: A funds transfer must debit one account service and credit another atomically. If one operation fails, both must roll back. Use 2PC for short-lived (seconds) high-value transactions where strong atomicity is non-negotiable. Example: a bank transfer from `checking-account-service` to `savings-account-service` coordinated by a 2PC transaction with both databases supporting XA.
  - **Avoid when:** The transaction can be redesigned as a saga — most financial transfers can use eventual consistency with compensating transactions, which scales better and avoids blocking locks.

- **Order management workflows** — coordinating inventory, payment, and fulfillment across services
  - When to use: An order must reserve inventory, capture payment, and create a shipment. If any step fails, the entire operation must be undone. Use the saga pattern with compensating transactions instead of 2PC, because locks on inventory would block other customers during the entire multi-step process. Example: orchestration-based saga where the order service coordinates `ReserveInventory` → `ProcessPayment` → `CreateShipment`, with compensating actions for each step on failure.
  - **Avoid when:** The order workflow is a single service responsibility — keep it as a local transaction.

- **Data consistency between bounded contexts** — synchronizing state across domain boundaries
  - When to use: Two bounded contexts need consistent state (e.g., an order status in the "Ordering" context must match the payment status in the "Payment" context). Use the outbox pattern with CDC (Change Data Capture) to ensure events are reliably published from the source context. Example: the `ordering` service writes both the order status change and an outbox event in the same database transaction; Debezium streams the outbox event to Kafka, which the `payment` service consumes.
  - **Avoid when:** Eventual consistency is acceptable without strict guarantees — a simple event publish after the transaction (with retry on failure) may be sufficient.

- **Migrating from 2PC to Saga for scalability** — replacing blocking distributed transactions with asynchronous compensation
  - When to use: Your 2PC-based system is hitting scalability limits because participants hold locks for too long under load. Migrate to a saga with compensating transactions to release locks quickly and improve throughput. Example: a travel booking system that originally used 2PC for hotel + flight + car reservations — migrated to orchestration-based sagas where each reservation is a local transaction, and cancellations serve as compensating actions.
  - **Avoid when:** The transaction must be strongly atomic (e.g., transferring money between accounts in the same bank) — 2PC guarantees atomicity that sagas cannot provide.

---

## Scenario-Based Questions

**Q: A 2PC transaction between Service A and Service B hangs for 5 minutes. You discover that the coordinator crashed in the middle of Phase 1. What is the state of Service A and Service B?**

- Both services that voted "yes" are in the prepared state — they have written prepare records to their transaction logs and acquired locks on the affected rows. They are blocking until the coordinator recovers (reads its transaction log) and sends the commit or abort decision. If a transaction timeout is configured, they will roll back after the timeout.
- **Interview follow-up:** How would you implement a "heuristic" resolution for stuck in-doubt transactions without a running coordinator?

**Q: A Saga for "Create Order" includes steps: Reserve Inventory → Charge Card → Confirm Order. If the card charge fails, the Saga runs a compensating transaction to release inventory. What happens if the release inventory compensation also fails?**

- The compensation is retried with exponential backoff. If retries are exhausted, the saga is marked as "failed with pending compensation" and sent to a dead-letter queue for manual review. A periodic reconciliation script should also detect reserved inventory that has no corresponding confirmed order and release it automatically.
- **Interview follow-up:** How would you design the dead-letter queue so that manual reviewers can resolve the inconsistency without writing custom scripts?

**Q: Your team uses the Outbox pattern: the service writes to its database and publishes an event. A consumer reads the event and processes it. You notice duplicate processing of some events. What went wrong?**

- The outbox relay uses at-least-once delivery — it may publish the same event twice if it crashes after sending to Kafka but before recording the offset. The consumer is not idempotent. Fix: Each outbox event has a unique ID; the consumer stores processed IDs in a deduplication table (or uses a Redis set) and skips events with already-processed IDs.
- **Interview follow-up:** How long should you retain the deduplication IDs? What happens if the deduplication table is lost?

## Interview Questions

- **Explain the difference between XA (2PC) and the Saga pattern.**
  - XA/2PC is a synchronous, blocking protocol that provides ACID guarantees across multiple resources using prepare and commit phases; participants hold locks during the prepare window. Sagas are an asynchronous, eventually-consistent pattern where each step is a local transaction with compensating transactions for rollback; no locks are held across services. Sagas provide BASE (Basically Available, Soft state, Eventual consistency) rather than ACID.

- **What is the difference between choreography and orchestration in the Saga pattern?**
  - Choreography: each service emits events that trigger the next service; no central coordinator. Simpler to implement but the flow is implicit (hard to trace). Orchestration: a central orchestrator invokes each service and handles compensation; easier to monitor, test, and manage, but the orchestrator is a single point of failure.

- **How does TCC differ from 2PC?**
  - TCC splits the transaction into Try (reserve), Confirm (commit), Cancel (rollback). The Try phase can reject the operation (like prepare). The Confirm phase is expected to always succeed and is independent (no coordinator-driven quorum). TCC is non-blocking and participants do not hold locks across phases; they hold "logical reservations" instead of physical locks.

- **What problem does the Outbox pattern solve?**
  - It solves the dual-write problem: writing to a database and sending a message/event in two separate operations risks inconsistency (write succeeds, message fails, or vice versa). The Outbox pattern writes the event as part of the same local database transaction, ensuring atomicity. A relay service (polling or CDC) reads the outbox and publishes events to the broker.

- **How does Change Data Capture (CDC) enable the Outbox pattern?**
  - CDC reads the database transaction log (WAL) and streams changes to a message broker. The outbox table is just another table in the database; when a row is inserted into the outbox table, CDC captures the insert event and publishes it to Kafka. This eliminates the need for a separate polling relay and provides sub-second latency. Debezium + Kafka Connect is the standard open-source CDC stack.

- **When would you accept using 2PC in a microservices architecture?**
  - 2PC might be acceptable when: (1) strong atomicity is a legal/regulatory requirement (e.g., financial settlement), (2) the transaction involves only a small number of participants (2–3), (3) the coordinator runs in a highly available cluster, and (4) throughput requirements are modest. In most other cases, Sagas or the Outbox pattern are preferred.

## Developer Recommendations

- **Prefer Sagas over 2PC for most microservices workflows**
  - 2PC blocks participants and introduces a single point of failure (the coordinator). Sagas are non-blocking, provide better throughput, and are more resilient to participant failures.
  - Implementation: Use an orchestration framework (e.g., Camunda, Temporal, Axon) or implement choreography via event-driven services. Ensure every step has a compensating transaction and every compensation is idempotent.
  - **Production story:** A SaaS company replaced a 2PC-based order system (5 tps max, frequent deadlocks) with a Temporal-based Saga; throughput increased to 200 tps and deadlocks were eliminated entirely.

- **Use the Outbox pattern for reliable event publication**
  - When a service must publish events as part of a business operation, use the transactional outbox to guarantee at-least-once delivery without distributed transactions.
  - Implementation: Add an `outbox` table with columns `id`, `aggregate_id`, `event_type`, `payload` (JSON), and `created_at`. In the same local transaction, write the domain entity and an outbox row. Use Debezium (CDC) to stream outbox rows to Kafka with sub-second latency.

- **Design all compensations and event consumers to be idempotent**
  - In distributed systems, messages and operations are delivered at least once; idempotency is the only safe assumption.
  - Implementation: Assign a unique ID to every event and operation. The consumer stores processed IDs in a deduplication table (or Redis with persistence) and skips duplicates. Compensating transactions check the current state before acting (e.g., "refund only if payment was actually charged").

- **Avoid distributed transactions where possible by redesigning the workflow**
  - Most business workflows can be redesigned to avoid distributed transactions. If a workflow spans services, ask: can this be done asynchronously? Can the data be owned by one service?
   - Implementation: Analyse the transaction rate, latency requirements, and consistency needs. If latency tolerance is > 1 second and eventual consistency is acceptable, use Sagas or async events. Only reach for 2PC when strong atomicity is truly non-negotiable.

## Scenario-Based Questions

**Q: Your team implements 2PC for a transaction involving three services: Inventory, Payment, and Shipping. The coordinator sends "prepare" to all three. Inventory and Payment vote "yes", but Shipping votes "abort" because its database is overloaded. What happens?**

- The coordinator sends "abort" to all participants. Inventory and Payment roll back their prepared transactions and release locks. Shipping already aborted, so no action needed. The transaction is fully rolled back.
- **Interview follow-up:** What happens if the coordinator's abort message to Inventory is lost due to a network partition — how does Inventory eventually learn about the abort?

**Q: You're designing an order placement flow using the Outbox pattern. The service writes the order to the `orders` table and an event to the `outbox` table in the same local transaction. A CDC pipeline crashes before publishing the outbox event. What happens?**

- Nothing permanent — the outbox event is already persisted in the database. When the CDC pipeline restarts, it resumes reading from the last committed WAL position and publishes the event. This is a key advantage of CDC-based outbox: the event is never lost because it's stored durably in the database transaction log.
- **Interview follow-up:** How does Debezium track its position in the WAL so that a crash doesn't cause it to skip or duplicate events?

**Q: In a TCC implementation for booking a hotel room, the Try phase reserves the room, but before the Confirm phase executes, the customer's credit card expires. How does your system handle this?**

- The Confirm phase should validate all preconditions before finalizing. If the card is expired, Confirm should fail and trigger the Cancel phase, which releases the room reservation. TCC assumes the Confirm phase can still fail (though it should be rare). The key is that the Try phase only reserves, so releasing via Cancel has no financial impact.
- **Interview follow-up:** What if the Cancel phase itself fails because the hotel inventory service is down — how do you prevent a permanently reserved room?

**Q: Your system uses both 2PC for financial settlements and Sagas for order workflows. An operator accidentally configures a Saga as a 2PC transaction. What problems would you expect to see?**

- The Saga would block participants because 2PC holds locks during the prepare phase. Long-running Saga steps would exacerbate blocking intervals, causing deadlocks and reduced throughput. The transaction coordinator would become a bottleneck. Eventually, participants would timeout and abort, causing frequent rollbacks.
- **Interview follow-up:** How would you detect such a misconfiguration in production before users report issues?

**Q: A microservice uses the Outbox pattern with a polling-based relay (not CDC). The relay polls every 100ms and publishes to Kafka. During a traffic spike, the outbox table grows to 100,000 rows before the relay catches up. What issues arise?**

- The relay may be overwhelmed by the backlog, causing increasing lag. If the relay crashes, it would need to paginate through 100,000+ rows on restart, potentially causing duplicate publication if it uses non-transactional offsets. The outbox table also grows the database, potentially impacting query performance on the main tables if they share the same storage.
- **Interview follow-up:** How would you modify the polling relay to handle a large backlog more efficiently without losing events?

**Q: You're implementing a distributed transaction across a relational database and a message queue (e.g., JMS). The database votes "yes" but the message queue cannot prepare because it's full. What does 2PC do?**

- The coordinator receives the abort vote from the message queue and sends an abort to all participants, including the database. The database rolls back its prepared transaction and releases locks. The transaction is fully aborted. This demonstrates 2PC's atomicity guarantee even when the decision is driven by a capacity issue in one participant.
- **Interview follow-up:** How would you instrument the system to alert operators that the message queue capacity was the root cause of the frequent transaction aborts?

**Q: Your team is building a payment reconciliation system that must have strong consistency. A senior engineer proposes using 2PC. The transaction involves a PostgreSQL database and an external REST API. Why is this problematic?**

- External REST APIs typically do not support 2PC's prepare/commit protocol (they have no transaction coordinator interface). The REST API would be treated as a "last resource" in a Last Resource Commit (LRC) optimization, which can lead to heuristic outcomes if the coordinator crashes after committing the database but before calling the REST API. This violates atomicity.
- **Interview follow-up:** What alternatives would you propose that still provide strong consistency guarantees for this scenario?

## Interview Questions

- **What is the difference between 2PC and TCC?**
  - 2PC uses a two-phase protocol (prepare, commit) with physical locks held during the prepare phase. TCC uses three phases (Try, Confirm, Cancel) where Try reserves logical resources without physical locks. TCC is non-blocking and more performant, but requires participants to implement reservation logic. 2PC provides stronger atomicity guarantees but reduces availability due to blocking.

- **What is the "dual-write problem" in microservices?**
  - The dual-write problem occurs when a service must write to its database and send a message/event in two separate operations. If the database write succeeds but the message send fails (or vice versa), the system becomes inconsistent. The Outbox pattern solves this by writing the event to a database table in the same local transaction as the domain entity.

- **How does the Outbox pattern differ from CDC?**
  - The Outbox pattern is a design approach: write events to a database table within the same transaction as the domain change. CDC is an implementation mechanism: read database transaction logs to capture changes. CDC can be used to implement the Outbox pattern (streaming outbox events from the WAL), but CDC is also useful for other use cases like data replication and audit logging.

- **What is idempotency and why is it critical in distributed transactions?**
  - Idempotency means that executing an operation multiple times produces the same result as executing it once. In distributed transactions, network failures, retries, and timeouts mean any operation may be executed multiple times. Idempotent operations (using unique keys, conditional updates, or dedup tables) ensure that retries do not cause unintended side effects like double charges or duplicate inventory releases.

- **What is the role of a transaction coordinator in 2PC?**
  - The transaction coordinator manages the lifecycle of a 2PC transaction. It sends prepare requests to all participants, collects votes, makes the commit/abort decision, and sends the final decision. It persists the transaction log to survive crashes. If the coordinator fails, participants remain blocked until it recovers and resolves in-doubt transactions.

- **What are heuristic decisions in distributed transactions?**
  - Heuristic decisions occur when a participant independently decides to commit or abort an in-doubt transaction without waiting for the coordinator's decision (usually due to a timeout or administrator intervention). Heuristic outcomes can violate atomicity if different participants make different heuristic decisions. Heuristic commits are particularly dangerous because they can leave the system in an inconsistent state.

- **What is the Last Resource Commit (LRC) optimization?**
  - LRC is an optimization for 2PC where one participant (typically a non-XA resource like a REST API or message queue) is treated as the "last resource". The coordinator commits all XA resources in phase 1, then commits the last resource. If the coordinator crashes between these steps, a heuristic outcome is possible. LRC improves performance but sacrifices some atomicity guarantees.

- **What is the "phantom inventory" problem in Sagas?**
  - Phantom inventory occurs when an inventory item is reserved by a Saga step, then released by a compensating transaction, but during the window between release and notification, another customer sees the item as available and places an order. The released inventory can appear as phantom stock to concurrent operations. Solutions include optimistic concurrency control, versioned inventory records, and the "pending" state pattern.

- **How do you handle timeouts in distributed transactions?**
  - In 2PC, each participant configures a transaction timeout. If the prepare or commit phase exceeds the timeout, the participant aborts unilaterally (heuristic abort). In Sagas, each step has a timeout; if a step doesn't complete within the timeout, the orchestrator triggers compensation for all completed steps. Both approaches require monitoring and alerting for timeout events.

- **What is the "outbox relay" and how does it ensure reliable delivery?**
  - The outbox relay is a process that reads events from the outbox table and publishes them to a message broker. It can be implemented as a polling process (periodically querying the outbox table) or a CDC pipeline (streaming from the WAL). The relay ensures at-least-once delivery by tracking its position (offset) and retrying failed publications. Consumers must be idempotent to handle duplicate deliveries.

- **What is the difference between at-least-once and exactly-once delivery?**
  - At-least-once delivery guarantees that every message is delivered at least once, but duplicates may occur. Exactly-once delivery guarantees no duplicates and no lost messages. In practice, exactly-once is extremely difficult to achieve in distributed systems. Most systems use at-least-once delivery combined with idempotent consumers to achieve exactly-once processing semantics.

- **How does database transaction log (WAL) replication relate to distributed transactions?**
  - WAL replication captures every change made to a database and can be used to replicate data to followers or stream changes to downstream systems. In the context of distributed transactions, WAL-based CDC enables the Outbox pattern without polling. However, WAL replication does not by itself solve distributed transaction coordination — it only propagates changes from a single database.

- **What is the "scheduling" approach as an alternative to distributed transactions?**
  - The scheduling approach breaks a distributed operation into small, idempotent tasks that are executed by a reliable scheduler (e.g., Quartz, Temporal, or a simple cron job). Each task is retried until success. This avoids distributed transactions entirely by relying on deterministic retry rather than atomic coordination. It works well for batch-oriented workflows but adds latency for real-time operations.

- **Can you use 2PC with NoSQL databases?**
  - Most NoSQL databases (MongoDB, Cassandra, DynamoDB) do not support XA/2PC. Some provide alternatives: MongoDB offers multi-document ACID transactions within a replica set (not across shards), Cassandra offers lightweight transactions and batch operations (not full 2PC), DynamoDB offers transactions within a single account/region. For cross-database coordination, Sagas are typically the only viable option with NoSQL stores.
