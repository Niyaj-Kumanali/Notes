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

## Use Cases

- **Choosing a database for global e-commerce** — balancing consistency of inventory counts with availability during network partitions
  - CP database (e.g., Spanner, MongoDB with majority write concern) guarantees consistent reads but may reject writes during a partition. AP database (Cassandra, DynamoDB) accepts writes everywhere but may show stale inventory.
  - **Avoid when:** your system runs on a single node — CAP only applies to distributed systems.

- **Designing cross-region replication** — multi-region deployments with different CAP trade-offs
  - Synchronous replication (CP) provides strong consistency but increases write latency and reduces availability during inter-region network issues. Asynchronous replication (AP) accepts writes locally and reconciles later, trading consistency for availability.
  - **Avoid when:** tolerance for stale reads is zero — CP is mandatory even if it means occasional unavailability.

- **Microservices data store decisions** — each service chooses its own CAP trade-off
  - Example: inventory service needs CP (you can't sell the same item twice), but analytics service can be AP (some data loss is acceptable). Choose per-service, not enterprise-wide.
  - **Avoid when:** business requirements don't distinguish between services — a uniform approach simplifies operations.

- **Configuring NoSQL consistency levels** — tuning read/write consistency in Cassandra (ONE, QUORUM, ALL)
  - Higher consistency reduces availability and increases latency during partitions. QUORUM balances both. Choose based on the criticality of the data being read.
  - **Avoid when:** all operations have the same consistency requirement — vary consistency per operation based on business context.

- **PACELC trade-off analysis** — considering CAP during partitions AND latency vs consistency during normal operation
  - Even without partitions, there's a trade-off: replicating synchronously (higher latency, stronger consistency) vs asynchronously (lower latency, eventual consistency).
  - **Avoid when:** your data store is not replicated — a single-node database has no PACELC trade-off.

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

**Q: Your social media feed system uses eventual consistency. During a network partition, some users see duplicate posts. How do you fix it?**

- Use idempotency keys on post creation — each post carries a unique ID, and the system deduplicates on write
- Implement vector clocks to track causal relationships and detect duplicates during read-repair
- Add a background deduplication job that scans for conflicting entries and merges them based on timestamp and causality
- **Interview follow-up:** How would you handle the case where two users concurrently post content that causally depends on each other's posts during a partition?

**Q: A financial trading system using CP (Paxos) suffers from high latency during leader re-election. How do you reduce the impact?**

- Pre-commit transactions to a write-ahead log before attempting consensus so that in-flight operations are not lost during leader election
- Use multi-leader replication per asset class to confine election scope — each asset class has its own Paxos group
- Optimize leader lease duration: longer leases reduce election frequency but increase failover time — tune based on cluster size and network reliability
- **Interview follow-up:** How do you handle split-brain scenarios where two nodes both believe they are the leader?

**Q: Your database uses quorum reads and writes for strong consistency. A network partition causes all writes to fail because quorum cannot be reached. How do you improve availability?**

- Reduce quorum size from ALL to QUORUM (N/2 + 1) — this allows minority partitions to still operate at the cost of consistency guarantees
- Implement a fallback mode: if quorum is unavailable after a timeout, degrade to read-only mode or serve stale data with a warning header
- Introduce a "quorum lease" that allows a partition with a recent lease to continue operating even without a full quorum
- **Interview follow-up:** How do you ensure safety when a stale partition is allowed to serve writes after lease expiration?

**Q: Your multi-region database uses synchronous replication. A cross-region latency spike increases write p99 from 5ms to 500ms. How do you fix it?**

- Switch to asynchronous replication for cross-region traffic while keeping synchronous within-region replication
- Use a committed transaction log that replicates asynchronously — the local region commits immediately, remote regions catch up
- Implement a latency-aware replication policy: if cross-region latency exceeds a threshold, automatically degrade from synchronous to asynchronous
- **Interview follow-up:** How do you handle the risk of data loss when switching from synchronous to asynchronous replication?

**Q: Your team implements an AP system for a ride-hailing platform. During a partition, two drivers accept the same ride request. How do you resolve the conflict?**

- Use a conflict resolution strategy: the first acceptance that reaches the central coordinator is honored; the second gets a compensation notification
- Implement a "claim token" system: the ride request is assigned a unique token; the driver who presents the token first gets the ride
- After partition recovery, run a reconciliation service that detects double-booked rides and reassigns one driver or cancels with a penalty-free notification
- **Interview follow-up:** How would you redesign the system to prevent double-acceptance entirely, even during partitions?

**Q: A configuration management service uses ZooKeeper (CP). During a network partition, the minority nodes stop serving reads. How do you ensure your application stays functional?**

- Cache ZooKeeper data locally on each application node so that reads can be served from cache even when ZooKeeper is unavailable
- Use a sidecar proxy that maintains a local copy of configuration and serves it with a staleness tolerance of a few seconds
- Implement a "last known good" configuration — if ZooKeeper is unavailable, continue operating with the last successfully read configuration
- **Interview follow-up:** How do you detect when ZooKeeper state has changed during the outage and invalidate the local cache?

**Q: Your IoT data pipeline ingests sensor readings from millions of devices using AP (Cassandra). Duplicate readings appear during partitions. How do you handle this?**

- Use a composite primary key (device_id, timestamp) to allow upserts — if the same reading arrives twice, the second write overwrites the first
- Design sensors to include a monotonically increasing sequence number; use it for deduplication during read-repair
- Accept duplicates at the storage layer and deduplicate at query time using window functions over time
- **Interview follow-up:** How do you handle out-of-order arrival of sensor readings due to network delays or device clock skew?

**Q: Your e-commerce platform uses eventual consistency for the product catalog. During a flash sale, a user sees an item as "in stock" but it is actually sold out. How do you prevent this?**

- Reduce the convergence window by tuning replica sync intervals from minutes to seconds
- Implement "reservation" semantics: when an item is added to cart, temporarily decrement the cached stock count and confirm from the authoritative inventory service asynchronously
- Use a hybrid approach: serve the catalog from an eventually consistent cache, but query the authoritative inventory service for stock checks on checkout
- **Interview follow-up:** How would you design the inventory system to handle both high-read throughput and strong consistency on stock checks?

## Interview Questions

- **What happens to a CP system during a network partition?**
  - The minority partition stops accepting writes to preserve consistency; the majority partition continues operating. Clients in the minority partition receive errors until the partition heals or they connect to the majority side.
- **Explain the PACELC extension and give a real-world example.**
  - PACELC states that when partitioned (P), trade off C vs A; when Else (E, normal operation), trade off Latency (L) vs Consistency (C). DynamoDB is PA/EL — it prefers availability during partition and low latency during normal operation.
- **How does eventual consistency differ from strong consistency in practice?**
  - Strong consistency guarantees that a read immediately following a write returns that write. Eventual consistency guarantees that if no new writes occur, all replicas will eventually converge. In practice, eventual consistency means stale reads are possible within a convergence window.
- **Why is CRDT preferred over last-write-wins for conflict resolution in some systems?**
  - LWW loses concurrent updates when timestamps tie or clocks skew. CRDTs (like grow-only counters or register sets) merge mathematically without data loss, making them suitable for offline-first and collaborative applications.
- **What is the difference between strong consistency and eventual consistency in practical terms?**
  - Strong consistency guarantees that once a write completes, all subsequent reads return that value. Eventual consistency guarantees that if no new writes occur, all replicas will converge. Strong consistency requires coordination (higher latency), while eventual consistency allows stale reads (lower latency, higher availability).
- **How do you implement read-after-write consistency?**
  - Route a client's reads to the same node that handled their most recent write, or have the write wait for acknowledgment from the read replica before returning. Session guarantees in DynamoDB and Cassandra read-your-writes consistency level provide this.
- **Explain the role of a consensus algorithm like Raft in CAP tradeoffs.**
  - Raft ensures strong consistency by having a single leader that orders all operations and replicates them to a majority of followers. If a partition separates the leader from the majority, the leader steps down and a new leader is elected in the majority partition. The minority partition cannot accept writes because it lacks a quorum. This makes Raft-based systems CP.
- **What happens to availability in a CP system using Raft when a follower's disk is full?**
  - The follower stops accepting log entries and falls behind. As long as a majority of nodes remain healthy, the system continues operating. The full-disk follower will eventually be removed from the cluster, and the leader does not block writes because only majority acknowledgment is needed.
- **How do you detect a network partition in a distributed system?**
  - Nodes use heartbeats with timeouts to detect failures. If node A does not receive a heartbeat from node B within a timeout period, A considers B down or partitioned. Gossip protocols (like SWIM) provide scalable failure detection. The challenge is distinguishing between a crash and a partition.
- **Explain the difference between a network partition and a node crash in CAP terms.**
  - In a node crash, remaining nodes form a majority and continue operating. In a network partition, two groups each think the other crashed, leading to split-brain. AP systems allow both sides to operate (divergence), while CP systems shut down the minority side to preserve consistency.
- **How would you design a system that needs strong consistency for some operations and eventual consistency for others?**
  - Use a polyglot architecture: store critical data (transactions, balances) in a CP system and non-critical data (profiles, feeds) in an AP system. Alternatively, use a single database with tunable consistency — Cassandra's QUORUM for critical reads, ONE for non-critical reads.
- **What is the role of a quorum in ensuring consistency?**
  - A quorum is the minimum number of nodes that must participate for an operation to be valid. For strong consistency, read-quorum + write-quorum > N ensures at least one node overlaps between read and write sets. With N=3, write QUORUM=2 and read QUORUM=2 ensures the read set overlaps with the write set.
- **How does Google Spanner achieve both strong consistency and high availability?**
  - Spanner uses TrueTime (GPS + atomic clocks) for globally synchronized timestamps, Paxos for synchronous within-region replication, and asynchronous cross-region replication. TrueTime enables lock-free read-only transactions and external consistency, while Paxos provides CP within each replica group.
- **What is the difference between linearizability and serializability?**
  - Linearizability means each operation appears to take effect atomically at some point between its start and end — it is about recency. Serializability means the result of concurrent transactions is equivalent to some sequential execution — it is about isolation. A system can be serializable without being linearizable.
- **How do you implement eventually consistent counters across data centers?**
  - Use CRDT counters (G-counter or PN-counter) that merge mathematically without conflict. Each data center maintains its own increment counters; periodic gossip merges them by taking the maximum of each counter. The total count is the sum of all per-DC counters, avoiding coordination while ensuring convergence.
- **What is the split-brain problem and how do consensus algorithms prevent it?**
  - Split-brain occurs when two nodes both believe they are the leader. Raft prevents this by requiring a candidate to receive votes from a majority. A partitioned node cannot get majority votes, so it cannot become leader. Raft also uses a "term" number to distinguish stale leaders.
- **How does MongoDB handle CAP tradeoffs in different configurations?**
  - Standalone MongoDB is CA (no distribution). Replica set with primary reads is CP — if the primary fails, writes are unavailable until election. Replica set with secondary reads is AP — reads may return stale data but availability is maintained.
- **Explain how hinted handoff works in Dynamo-style systems.**
  - When a write cannot reach its target replica due to a partition, the coordinator stores the write as a "hint" on a different healthy node. When the target replica recovers, the hint is replayed. Hinted handoff improves availability during partitions but risks data loss if the hint node also fails before replaying.
- **What is read repair and when is it triggered?**
  - Read repair compares responses from all replicas during a read. If replicas have different versions, the coordinator pushes the latest version to stale replicas before returning. It is triggered on every read in some systems (Cassandra) or probabilistically to reduce overhead.
- **How do you balance latency and consistency in a globally distributed database?**
  - Use a tiered approach: within-region strong consistency with synchronous replication; cross-region eventual consistency with asynchronous replication. Data needing global consistency can use Paxos across regions, accepting higher latency. Use a circuit breaker to degrade to local consistency if cross-region latency exceeds a threshold.

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
