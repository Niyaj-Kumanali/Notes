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

## Interview Questions

- **Why does naive hash(key) % N fail for distributed systems?**
  - When N changes, the modulus result changes for almost all keys, requiring nearly every key to be remapped to a different node. This causes massive data migration and cache invalidation in proportion to K (total keys), not K/N.
- **How do virtual nodes improve consistent hashing?**
  - Each physical node is represented by multiple positions on the ring (virtual nodes). This creates a finer-grained distribution that reduces load imbalance caused by non-uniform node capacities or ring positions. Without vnodes, large gaps between nodes cause some nodes to handle disproportionately more keys.
- **Explain clockwise assignment in consistent hashing.**
  - Each key is hashed to a position on the ring. Moving clockwise from that position, the first node encountered owns the key. This ensures that when a node is added, only the keys in the arc between the new node and its predecessor change ownership.
- **How does Cassandra use consistent hashing for data distribution?**
  - Cassandra assigns each node a token range on a consistent hash ring. A key is hashed and mapped to a token value; the node responsible for that token range stores the key. With replication factor R, the next R-1 nodes clockwise also store replicas. Virtual nodes (num_tokens) allow each node to own multiple non-contiguous ranges.

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
