# Consistent Hashing

## Overview

- **Definition** — A distributed hashing scheme that maps keys to nodes on a hash ring, requiring minimal key redistribution when nodes are added or removed
- **Why It Exists** — Naive hash(key) % N breaks completely when N changes — nearly all keys must be remapped, causing massive data movement and cache invalidation
- **Historical Context** — Introduced by David Karger et al. in 1997 for distributed caching (Akamai); popularized by Amazon Dynamo paper (2007) and Cassandra
- **Key Concepts** — **Hash ring** is a circular space (0 to 2^32-1); **Virtual nodes** replicate each physical node to multiple ring positions for better load balance; **Clockwise assignment** maps each key to the nearest node moving clockwise; **Minimal redistribution** means only K/N keys move on average when a node joins or leaves

## Core Concepts

- Naive partitioning uses hash(key) % N for node selection
  - When N changes (node add/remove), nearly all keys hash to different nodes
  - Causes massive cache misses, data migration, or repartitioning
- Consistent hashing maps both nodes and keys to positions on a hash ring
  - Each node is hashed (MD5, SHA-1, or a fast hash like MurmurHash) to one or more ring positions
  - Each key is assigned to the nearest node scanning clockwise from the key's position
- When a node is added, only keys in the arc between the new node and its predecessor need reassignment
  - The new node takes over those keys from the next clockwise node
- When a node is removed, only its keys redistribute to the next clockwise node
  - No other nodes in the ring are affected
- Minimum redistribution property: a node addition or removal moves only K/N keys
  - Where K is the total number of keys and N is the number of nodes
- Virtual nodes (vnodes) replicate each physical node to multiple positions on the ring
  - A physical node with 100 vnodes appears in 100 separate locations
  - Without vnodes, non-uniform node distribution or heterogeneous capacities cause hot spots
  - Increasing vnodes improves load balance but increases routing table size
- Applications include:
  - DynamoDB partitioning: consistent hashing enables seamless scaling with minimal data movement
  - Cassandra partitioning: each node owns a token range on the ring with configurable replication factor
  - Distributed caching: Memcached, Redis Cluster, and CDNs use consistent hashing for cache key distribution
  - Load balancing: hash-based request routing with minimal session disruption on server change

## Common Mistakes

- **Using too few virtual nodes leading to hotspots**
  - Without sufficient vnodes, large nodes dominate uneven ring segments; popular keys accumulate on a single physical node
  - **Why it looks correct:** One vnode per physical node is the simplest implementation
  - Use 100–200 virtual nodes per physical node; monitor per-node key distribution and increase vnodes if standard deviation exceeds 15%
- **Assuming uniform hashing eliminates all hotspots**
  - Even with many vnodes, popular keys (viral content, celebrity profiles) can overload a single node
  - **Why it looks correct:** The hash function distributes keys uniformly across the ring space
  - Combine consistent hashing with load shedding, replication of hot keys, or a dedicated hot-key cache layer
- **Neglecting to handle node heterogeneity through weight**
  - Allocating the same number of vnodes to a 2-core VM and a 32-core machine overloads the smaller node
  - **Why it looks correct:** All nodes look the same on the ring
  - Assign vnode count proportional to node capacity (CPU, memory, disk); smaller nodes get fewer vnodes

## Real-World Scenarios

### Cassandra Partitioning

- Each node is assigned a token range on the ring (default Murmur3Partitioner)
- Replication factor N means each key is stored on the next N nodes clockwise
- Virtual nodes (num_tokens) are enabled by default — each node owns multiple token ranges
- When a new node joins, it steals token ranges from neighbors; only affected ranges stream data

### DynamoDB Partitioning

- Table data is partitioned by partition key hash; consistent hashing maps partitions to physical storage nodes
- When throughput exceeds a partition's capacity, DynamoDB splits the partition using the hash ring
- Split affects only one partition — adjacent partitions are untouched
- Each storage node handles multiple partitions for load distribution

### Distributed CDN Caching

- CDN edge caches use consistent hashing to map content URLs to cache nodes
- When a cache node fails, only content mapped to that range is fetched from origin
- New nodes are gradually populated without a global cache flush

## Use Cases

- **Distributed caching** — sharding cache keys across a Memcached or Redis cluster
  - Adding or removing cache nodes only moves keys in the affected range (1/N fraction), not all keys. Minimizes cache stampede during scaling.
  - **Avoid when:** cache size is small enough to fit on one node — a single-node cache with replication is simpler.

- **Database sharding** — partitioning data across multiple database instances
  - Consistent hashing on shard key (user_id, tenant_id) distributes data evenly. Adding a new shard only remaps a fraction of data, not a full rehash.
  - **Avoid when:** data can be partitioned naturally (by region, by date range) — range-based sharding is simpler for time-series data.

- **CDN content routing** — mapping content URLs to edge cache servers
  - When a cache node fails, only content mapped to that node's range is fetched from origin. New nodes are gradually populated without a global cache flush.
  - **Avoid when:** your CDN is a managed service (CloudFront, CloudFlare) — the provider handles routing transparently.

- **Load-balanced task distribution** — distributing processing tasks across worker nodes
  - Workers claim responsibility for hash ranges. When a worker joins or leaves, only its tasks are redistributed. Minimal task reassignment.
  - **Avoid when:** tasks have heterogeneous processing times — consistent hashing doesn't account for load; use weighted or least-loaded distribution.

- **Avoiding hot spots with virtual nodes** — preventing a few physical nodes from handling disproportionate traffic
  - Each physical node is represented by multiple virtual nodes on the ring. This smooths out load distribution, especially with small clusters.
  - **Avoid when:** the cluster is large (>100 nodes) — the law of large numbers distributes load evenly without virtual nodes.

## Scenario-Based Questions

**Q: Your distributed cache experiences high latency because one node handles 40% more requests than others. How do you redistribute the load?**

- Increase the number of virtual nodes per physical node for finer granularity
- Check if a few keys dominate traffic — those hot keys should be replicated to multiple nodes or cached at the client
- Rebalance by reassigning vnodes: remove a portion from hot nodes and assign to cooler nodes
- **Interview follow-up:** How would you handle a "celebrity key" that gets 100x more traffic than any other key, even with perfect distribution?

**Q: You are adding a new node to a Cassandra cluster. During the bootstrap, the existing nodes show increased latency. What is happening and how do you fix it?**

- The new node streams data from neighbors, consuming network bandwidth and disk I/O
- Throttle streaming throughput using streaming_throughput_mb setting in Cassandra
- Add nodes one at a time rather than in batches to control the impact
- **Interview follow-up:** What happens if the new node fails mid-bootstrap — how do you recover the partially streamed data?

**Q: Your consistent hashing ring has 10 physical nodes with 1 vnode each. Node 5 handles 25% of traffic while Node 8 handles only 3%. How do you fix the imbalance?**

- Increase virtual nodes per physical node to 100–200 for finer granularity — this distributes keys more evenly across all nodes
- Rebalance the ring by reassigning some vnode ranges from the hot node to cooler nodes
- Monitor per-node key distribution and adjust vnode counts based on actual load
- **Interview follow-up:** How do you determine the optimal number of virtual nodes for a cluster with heterogeneous hardware?

**Q: You are adding a new data center with 5 nodes to an existing 20-node Cassandra cluster. What happens to the ring and how do you minimize data movement?**

- The new nodes are assigned token ranges on the ring; each new node takes over a portion of keys from existing nodes
- Data streaming happens only between neighbors of the new token ranges, not the entire cluster
- Throttle streaming bandwidth and add nodes one at a time to control the impact on existing traffic
- **Interview follow-up:** How does the replication factor affect data movement when adding nodes?

**Q: Your distributed cache uses consistent hashing. When a cache node fails, the next node clockwise receives all its keys and becomes overloaded. How do you prevent this?**

- Use virtual nodes so that a single physical node's keys are spread across many successors, not just one
- Configure a replication factor of 2–3 so each key is stored on multiple nodes — if one fails, the load is shared among replicas
- Add a "load shedding" mechanism: if a node's request rate exceeds a threshold, it returns a retry response to some clients
- **Interview follow-up:** How would you implement a "consistent hashing with bounded loads" strategy to prevent overload on takeover?

**Q: You use consistent hashing for a CDN edge cache. A new edge location comes online, but users see increased cache miss rates for 30 minutes. Is this expected?**

- Yes — the new node takes over a range of keys from its predecessor. Those keys were previously cached on the predecessor, but the new node has a cold cache
- Mitigate by gradually shifting traffic: use a "warmup" phase where the new node receives a fraction of its eventual traffic for several minutes
- Pre-populate the new node's cache by streaming the most popular keys from the predecessor before going live
- **Interview follow-up:** How would you implement cache warming without impacting the predecessor's ability to serve traffic?

**Q: Your key distribution across virtual nodes shows a standard deviation of 30% in per-node key counts. What is the likely cause and how do you fix it?**

- 100–200 vnodes per physical node should yield <15% standard deviation — 30% suggests too few vnodes or a poor hash function
- Increase the number of virtual nodes per physical node to improve distribution granularity
- If vnodes are already high, check the hash function for bias — MurmurHash or SHA-256 are preferred over simple hash codes
- **Interview follow-up:** How do you measure key distribution imbalance in production without scanning the entire key space?

**Q: Your application uses Redis Cluster with hash slots (16384 slots). You add a new node, but some keys are now missing from the cluster. What went wrong?**

- Redis Cluster requires resharding hash slots from existing nodes to the new node — adding a node does not automatically migrate data
- Use the CLUSTER SETSLOT command family to migrate slot ranges, or use redis-cli --cluster reshard
- Verify the cluster state with CLUSTER INFO and CLUSTER NODES after resharding
- **Interview follow-up:** How does Redis Cluster handle requests for keys in slots that are in the process of being migrated?

**Q: Your consistent hashing implementation uses SHA-256 for node hashing. Node additions cause temporary routing inconsistencies where some clients route to the wrong node. Why?**

- SHA-256 output is large — different implementations may handle byte ordering or truncation differently, leading to inconsistent ring positions
- Hash functions for consistent hashing should produce consistent 32-bit or 64-bit integers regardless of platform
- Fix: use a well-defined hash function with a canonical implementation (MurmurHash3_x86_128, or CityHash) with explicit byte ordering
- **Interview follow-up:** How do you handle a rolling upgrade of the hashing algorithm without causing data unavailability?

**Q: Your database uses consistent hashing with a replication factor of 3. A node failure causes data loss for some keys. How is this possible with replication?**

- With RF=3, each key is stored on 3 consecutive nodes on the ring. If all 3 nodes fail simultaneously, data for the keys in that range is lost
- The ring topology creates correlated failure risk — nodes adjacent on the ring may share physical infrastructure (same rack, power supply)
- Mitigate: ensure failure domains are decorrelated from ring positions — use rack-aware placement so replicas span different failure domains
- **Interview follow-up:** How would you design a consistent hashing scheme that explicitly places replicas in different availability zones?

## Interview Questions

- **Why does naive hash(key) % N fail for distributed systems?**
  - When N changes, the modulus result changes for almost all keys, requiring nearly every key to be remapped to a different node. This causes massive data migration and cache invalidation in proportion to K (total keys), not K/N.
- **How do virtual nodes improve consistent hashing?**
  - Each physical node is represented by multiple positions on the ring (virtual nodes). This creates a finer-grained distribution that reduces load imbalance caused by non-uniform node capacities or ring positions. Without vnodes, large gaps between nodes cause some nodes to handle disproportionately more keys.
- **Explain clockwise assignment in consistent hashing.**
  - Each key is hashed to a position on the ring. Moving clockwise from that position, the first node encountered owns the key. This ensures that when a node is added, only the keys in the arc between the new node and its predecessor change ownership.
- **How does Cassandra use consistent hashing for data distribution?**
  - Cassandra assigns each node a token range on a consistent hash ring. A key is hashed and mapped to a token value; the node responsible for that token range stores the key. With replication factor R, the next R-1 nodes clockwise also store replicas. Virtual nodes (num_tokens) allow each node to own multiple non-contiguous ranges.
- **What happens to consistent hashing when a node's performance degrades but it does not fail?**
  - The node remains on the ring and continues to receive its share of requests. This can cause long-tail latency. Mitigation: use load-based weight adjustment (reduce vnodes for slow nodes) or proactively remove the node and let it rejoin after recovery.
- **How do you handle key hot spots in a consistent hashing ring?**
  - Hot keys cannot be solved by hashing alone. Strategies: replicate hot keys to multiple nodes, cache them at the client or in a front cache (Redis), or use a dedicated hot-key detection system that dynamically increases replication for popular keys.
- **Explain the tradeoff between number of virtual nodes and routing table size.**
  - More vnodes improve load balance but increase the routing table size (each node tracks all vnode locations). With 1000 nodes × 200 vnodes = 200,000 entries per node, which is manageable in memory but increases update overhead when topology changes.
- **How does consistent hashing differ from range-based partitioning?**
  - Range-based partitioning divides the key space into contiguous ranges assigned to nodes. Consistent hashing interleaves ranges using hashing, providing better load balance and less redistribution on topology changes. Range partitioning makes range queries easy; consistent hashing makes them hard.
- **What is the "ring walking" problem and how do you mitigate it?**
  - When a node fails, its successor receives all its keys, potentially creating a cascading overload as that successor becomes slower and its own successor then receives even more load. Mitigation: use replication so the load is shared among multiple successors, and use virtual nodes so the load is distributed across many nodes.
- **How do you implement consistent hashing for a load balancer?**
  - Hash the client IP or session ID to determine which backend server handles the request. This provides session affinity without sticky cookies. When a server is added or removed, only a fraction of clients (1/N) are remapped to different servers.
- **How does DynamoDB's partition splitting work with consistent hashing?**
  - When a partition exceeds throughput capacity, DynamoDB splits it into two partitions on the hash ring. The split affects only that partition's range — adjacent partitions are unaffected. The new partitions may be moved to different physical storage nodes for load distribution.
- **What is the role of a "token" in Cassandra's consistent hashing?**
  - Each Cassandra node is assigned a token value that determines its position on the ring. With Murmur3Partitioner, tokens range from -2^63 to 2^63-1. A key is hashed to a token value, and the node whose token range contains that value owns the key.
- **How do you handle node heterogeneity in a consistent hashing cluster?**
  - Assign vnode count proportional to node capacity. A 32-core node with 256GB RAM gets more vnodes than a 4-core node with 32GB RAM. Some implementations support weighted consistent hashing where each vnode can have a weight multiplier.
- **Explain the difference between "random slicing" and "consistent hashing" for data partitioning.**
  - Random slicing divides key space into fixed slices and maps slices to nodes; adding a node requires remapping slices. Consistent hashing maps both keys and nodes to a ring, requiring only K/N keys to move on average when nodes change.
- **What happens to read repair in a consistent hashing system during node addition?**
  - When a new node joins, it initially has no data for its assigned keys. Read repair helps populate the node: when a client reads a key, the coordinator detects the new node is missing the value and writes the latest version to it during the read response.
- **How do you test consistent hashing correctness?**
  - Verify that every key maps to the same node before and after a single node addition (except keys in the affected range). Verify that key distribution is uniform (chi-squared test). Test with node removal and confirm only its keys move to successors.
- **What is the "consistent hashing skew" problem?**
  - Skew occurs when the hash function does not distribute keys uniformly across the ring, or when node positions are clustered. Virtual nodes reduce skew by giving each physical node many interleaved positions. Monitoring per-node key count variance detects remaining skew.
- **How does Akamai use consistent hashing for CDN content routing?**
  - Akamai maps content URLs to edge servers using consistent hashing on the URL hash. When an edge server fails, only content mapped to that server's range is fetched from origin. New servers are gradually populated without a global cache flush.
- **Compare consistent hashing with a distributed hash table (DHT).**
  - Consistent hashing maps keys to nodes on a ring with minimal redistribution. DHTs (like Chord) provide a more complete abstraction including lookup protocols, routing, and node discovery. Consistent hashing is simpler; DHTs are more feature-rich for peer-to-peer applications.
- **What is "rendezvous hashing" and how does it compare to consistent hashing?**
  - Rendezvous hashing (HRW) assigns each key to the node with the highest computed weight, using a hash of (node, key). It provides minimal redistribution on node changes like consistent hashing, but does not require a ring or virtual nodes. It is simpler to implement but has O(N) lookup cost per key compared to O(log N) for consistent hashing.

## Developer Recommendations

- **Always use virtual nodes in production**
  - Without vnodes, adding a node with different capacity than existing nodes creates severe imbalance
  - Set virtual node count high enough (100–200) for balanced distribution but low enough to keep routing table manageable
  - **Production story:** A Cassandra cluster with 1 vnode per node had 3x load variance — switching to 256 vnodes reduced variance to under 10%
- **Monitor for hot keys and plan mitigation**
  - Even perfect hash distribution cannot compensate for a single key receiving disproportionate traffic
  - Implement hot-key detection (request rate per key), replication (store hot key on multiple nodes), or local client caching
  - **Production story:** A celebrity tweet caused a single Cassandra node to hit 100% CPU — adding a front cache (Redis) for viral content resolved the hotspot
- **Use weighted consistent hashing for heterogeneous clusters**
  - Assign vnode count proportional to node capacity; smaller nodes get fewer vnodes, larger nodes get more
  - Rebalance gradually during low-traffic windows to avoid streaming storms
- **Test failure and recovery scenarios in staging**
  - Simulate node removal and observe redistribution impact on latency and memory
  - Verify that the ring stabilizes within acceptable time (seconds to minutes depending on data size)
