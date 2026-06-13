# CAP Theorem

## Overview

- **Definition** — In a distributed data store, you can only guarantee two of three properties: Consistency, Availability, and Partition Tolerance
- **Why It Exists** — Network partitions are unavoidable in distributed systems, forcing a fundamental tradeoff between consistency and availability
- **Historical Context** — Proposed by Eric Brewer in 2000 as the CAP conjecture, formalized as a theorem by Seth Gilbert and Nancy Lynch in 2002
- **Key Concepts** — **Consistency** means all nodes see the same data at the same time; **Availability** means every request gets a non-error response; **Partition Tolerance** means the system continues operating despite dropped or delayed network messages; **PACELC** extends CAP by adding tradeoffs when the system is running normally (Else)

## Core Concepts

- Consistency (C): every read receives the most recent write or an error
  - Strong consistency requires synchronous replication across nodes
  - Eventual consistency allows replicas to converge over time
- Availability (A): every request receives a non-error response, without guarantee it contains the latest write
  - High availability requires redundant nodes and automatic failover
- Partition Tolerance (P): the system continues to function despite network partitions that split nodes into groups
  - Partitions are inevitable — networks drop packets, switches fail, latency spikes
- CP systems sacrifice availability during a partition: HBase, ZooKeeper, MongoDB (with default settings)
  - When a partition occurs, CP systems may block writes or serve stale data from minority partitions
- AP systems sacrifice consistency during a partition: Cassandra, DynamoDB, CouchDB
  - When a partition heals, AP systems reconcile divergent writes using conflict resolution
- CA systems sacrifice partition tolerance, which means they cannot function across a network
  - Single-node RDBMS like a standalone PostgreSQL is CA but is not distributed
  - A distributed system claiming CA is misleading — if a partition happens, it must choose C or A
- PACELC extension: when Partitioned (P) trade off C vs A; when Else (E) trade off Latency (L) vs Consistency (C)
  - Example: DynamoDB prefers A during partition and L in normal operation (PA/EL)
  - Example: Google Spanner prefers C during partition and C in normal operation (PC/EC)
- Eventual consistency requires conflict resolution strategies
  - Last-write-wins (LWW) uses timestamps — simple but loses concurrent updates
  - CRDTs (Conflict-Free Replicated Data Types) merge automatically without coordination
- BASE (Basically Available, Soft state, Eventual consistency) vs ACID properties

## Common Mistakes

- **Choosing CA for a distributed system**
  - Assuming you can achieve both consistency and availability without partition tolerance in a multi-node deployment
  - **Why it looks correct:** A single-node RDBMS provides CA and works well; scaling vertically seems safer than distributing
  - For any system that spans multiple nodes over a network, design for P by choosing CP or AP based on business needs
- **Ignoring the recovery phase after a partition**
  - Designing only for partition detection without planning what happens when the network heals
  - **Why it looks correct:** Fixing the network seems like an ops concern, not an architecture concern
  - Implement anti-entropy mechanisms — Merkle trees in Cassandra, hinted handoff in DynamoDB, read-repair in Riak

## Real-World Scenarios

### HBase — CP System

- HBase uses HDFS and ZooKeeper for coordination
- During a region server failure, HBase blocks writes to that region until recovery completes
- Consistency is guaranteed at the cost of temporary unavailability

### Cassandra — AP System

- Cassandra uses gossip protocol and hinted handoff for eventual consistency
- During a partition, Cassandra accepts writes on both sides and reconciles later
- Tunable consistency (ONE, QUORUM, ALL) allows per-operation tradeoff

### Single-Node PostgreSQL — CA System

- No network distribution, so partition tolerance is irrelevant
- If you replicate PostgreSQL and a network split occurs, you must choose CP (synchronous) or AP (asynchronous)

## Scenario-Based Questions

**Q: Your e-commerce platform shows inconsistent product inventory across regions during a network partition. How do you resolve it?**

- Use read-repair: on each read, compare replicas and push the latest version to stale nodes
- Implement version vectors to track causal relationships between updates
- Add a reconciliation cron job that runs anti-entropy every few minutes
- **Interview follow-up:** How would you handle inventory decrements that go both directions during a split-brain scenario?

**Q: You are designing a global financial trading system. Which CAP tradeoff do you choose and why?**

- Choose CP with Paxos/Raft consensus to ensure all nodes agree on order and value
- Availability can be improved with redundant consensus groups and automatic leader election
- Compensating transactions or sagas handle the rare cases where a trade must be rolled back
- **Interview follow-up:** How do you maintain CP while keeping p99 latency under 10ms across regions?

## Interview Questions

- **What happens to a CP system during a network partition?**
  - The minority partition stops accepting writes to preserve consistency; the majority partition continues operating. Clients in the minority partition receive errors until the partition heals or they connect to the majority side.
- **Explain the PACELC extension and give a real-world example.**
  - PACELC states that when partitioned (P), trade off C vs A; when Else (E, normal operation), trade off Latency (L) vs Consistency (C). DynamoDB is PA/EL — it prefers availability during partition and low latency during normal operation.
- **How does eventual consistency differ from strong consistency in practice?**
  - Strong consistency guarantees that a read immediately following a write returns that write. Eventual consistency guarantees that if no new writes occur, all replicas will eventually converge. In practice, eventual consistency means stale reads are possible within a convergence window.
- **Why is CRDT preferred over last-write-wins for conflict resolution in some systems?**
  - LWW loses concurrent updates when timestamps tie or clocks skew. CRDTs (like grow-only counters or register sets) merge mathematically without data loss, making them suitable for offline-first and collaborative applications.

## Developer Recommendations

- **Default to CP for transactional workloads**
  - Financial systems, order processing, and inventory management need strict consistency to prevent double-spend or oversell
  - Use Raft or Paxos consensus, maintain quorum writes (QUORUM in Cassandra, majority in etcd/ZooKeeper)
  - **Production story:** A payment service used AP and double-charged users during a partition — switching to quorum writes eliminated the issue
- **Choose AP for high-volume, always-on systems**
  - Social feeds, analytics pipelines, and IoT data ingestion tolerate eventual consistency
  - Use conflict resolution with CRDTs or application-level merge logic
  - **Production story:** A chat system chose CP and dropped messages during partition — moving to AP with last-write-wins preserved availability
- **Design for P even when running in a single datacenter**
  - Network switches fail, rack power loss splits clusters, software bugs cause silent drops
  - Always test partition scenarios with chaos engineering (Chaos Monkey, Litmus)
  - **Production story:** A "single-region CA" system went down when a switch misconfiguration split the replica set — recovery took hours because there was no partition tolerance logic
