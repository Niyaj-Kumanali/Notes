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
