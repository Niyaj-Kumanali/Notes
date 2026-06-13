# Service Discovery

## Overview

- **Definition** — Service discovery enables microservices to dynamically locate the network addresses of other services without hardcoded configuration.
- **Why It Exists** — In dynamic environments where instances are constantly created, destroyed, and scaled, static IP addresses and ports are impractical; service discovery provides a registry where services register their location and consumers query at runtime.
- **Historical Context** — Early service discovery relied on DNS round-robin with low TTLs, but this lacked health checking and instant failover; modern solutions like Eureka and Consul emerged alongside the rise of container orchestration and microservices architectures.
- **Key Concepts** — **Registry** — central store of service instances; **Registration** — process of a service adding itself to the registry; **Heartbeat** — periodic signal proving instance liveness; **Health Check** — deeper validation beyond liveness; **Service Instance** — a single running copy of a service with IP/port metadata; **Client-side discovery** — consumer queries registry directly and picks an instance; **Server-side discovery** — consumer sends request to a load balancer/router that queries the registry.

## Core Concepts

- **Client-Side Discovery**
  - The service consumer directly queries the registry (e.g., Eureka, Consul) to obtain available instances.
  - The consumer applies load-balancing logic (e.g., round-robin, weighted response time) using a client-side library like Netflix Ribbon or Spring Cloud LoadBalancer.
  - Benefits: Fewer moving parts (no intermediate router), lower latency (direct connections), native awareness of instance health.
  - Drawbacks: Every language/runtime needs a compatible client library; registry changes must propagate to all consumers.

- **Server-Side Discovery**
  - The consumer sends requests to a well-known router or load balancer (e.g., AWS ALB, Kubernetes Service, Kong) which queries the registry and forwards to a healthy instance.
  - Benefits: Language-agnostic; clients are simpler (just a fixed DNS/hostname); centralised control of routing policies.
  - Drawbacks: Additional network hop; the router becomes a potential bottleneck and single point of failure.

- **Eureka (Netflix)**
  - AP-oriented (Availability and Partition tolerance) system — favours availability over consistency.
  - Each instance registers with a `POST /eureka/apps/{appName}` and sends heartbeats every 30 seconds.
  - Instances that fail to heartbeat within 90 seconds are evicted.
  - Self-preservation mode: If fewer than 85% of heartbeats arrive in the last minute, Eureka stops evicting instances — protects against network partitions incorrectly removing healthy instances but can keep dead instances registered.
  - Clients cache the full registry locally and refresh periodically; stale caches can cause calls to dead instances.

- **Consul (HashiCorp)**
  - CP-oriented (Consistency and Partition tolerance) system — uses Raft consensus to maintain a strongly consistent catalog.
  - Stores all service metadata, health check definitions, and KV configuration in a single key-value store.
  - Agent-based architecture: each node runs a Consul agent that handles health checks and communicates with the server cluster.
  - Supports multiple health check types (script, HTTP, TCP, gRPC, TTL).
  - DNS interface: services are discoverable via `<service>.service.consul` DNS queries, enabling existing applications to use discovery with zero code changes.
  - mTLS support via Connect: automatic TLS certificate rotation and authorization between services.

- **Kubernetes DNS (CoreDNS/kube-dns)**
  - Each Service object receives a DNS name (e.g., `my-svc.my-namespace.svc.cluster.local`).
  - kube-proxy watches the API server for Endpoint changes and programs iptables or IPVS rules to route Service cluster IPs to healthy pod IPs.
  - Readiness probes determine which pods receive traffic; failed readiness probes remove the pod from Endpoints.
  - IPVS mode provides better scalability and more sophisticated load-balancing algorithms (least connections, locality, etc.) than iptables.
  - Services of type `Headless` bypass cluster IP and return pod IPs directly via DNS for client-side discovery.

- **ZooKeeper (Apache)**
  - CP-oriented — uses ZAB (ZooKeeper Atomic Broadcast) consensus; sacrifices availability during leader elections.
  - Services register as ephemeral znodes; the znode is automatically removed when the session expires or the client disconnects.
  - Watchers can be set on znodes to notify consumers of membership changes.
  - Simpler than Consul (no built-in health check engine or KV UI), but provides strong ordering guarantees.
  - Unavailability window: during leader election (ZAB), the cluster is read-only; no registration or discovery changes can be made.

## Common Mistakes

- **Misunderstanding Eureka self-preservation**
  - During a network partition, Eureka stops evicting instances and the registry becomes stale; clients may route to dead instances for minutes.
  - **Why it looks correct:** Self-preservation is designed to prevent a flapping registry, but engineers often disable it without understanding the trade-off, causing cascading failures during real partitions.
  - The fix: Design clients to fail fast (circuit breakers, retries) rather than relying on registry accuracy alone. Consider using a separate health-check sidecar.

- **Shared ZooKeeper cluster for unrelated services**
  - Multiple microservices teams using the same ZooKeeper ensemble for discovery, leader election, and distributed locks, causing contention and slowdowns.
  - **Why it looks correct:** One ZooKeeper cluster is easier to operate than many, but the CP nature means a slow leader election stalls all dependent services.
  - The fix: Dedicate ZooKeeper ensembles per critical domain or use a lighter AP discovery system (Eureka/Consul) for registration and reserve ZooKeeper for coordination tasks.

- **Ignoring client-side caching**
  - Clients cache registry data indefinitely, causing traffic to be sent to terminated instances for hours.
  - **Why it looks correct:** Reducing registry lookups improves latency, but stale caches are worse than a small overhead per request.
  - The fix: Configure aggressive cache TTLs and implement background refresh; use circuit breakers to detect and skip dead instances.

- **DNS-based discovery without health checks**
  - Using standard DNS with short TTLs for discovery but relying only on TCP liveness (port open), not application-level health.
  - **Why it looks correct:** DNS is universally supported, but standard DNS lacks the ability to report application-level health (e.g., database connectivity failure).
  - The fix: Use a DNS server that integrates with health checks (Consul DNS, CoreDNS with EndpointSlice) or layer a client-side load balancer on top.

## Real-World Scenarios

### Netflix's Dependency on Eureka Self-Preservation

- During a regional AWS outage, Eureka self-preservation prevented 90% of healthy instances from being evicted due to heartbeat failures crossing the partition.
- Services that implemented retry-with-backoff and Hystrix circuit breakers survived; those that relied purely on registry accuracy suffered elevated error rates.
- Netflix tuned self-preservation thresholds over years based on production failure patterns; off-the-shelf defaults are not universally optimal.

### Kubernetes Service Mesh Migration for Discovery

- A large fintech migrated from Eureka to Kubernetes-native DNS + Istio service discovery.
- During the transition, they ran both registries in parallel with a Sidecar Proxy that maintained an internal service table from both sources.
- The migration took six months; the biggest challenge was ensuring readiness probes accurately reflected application health (not just process liveness).

### Consul mTLS Rollout at Scale

- An e-commerce platform enabled Consul Connect for all inter-service communication.
- Initial rollout caused 3x CPU increase on proxy sidecars due to per-request mTLS handshakes.
- They mitigated by enabling connection pooling, increasing persistent connection idle timeout, and using SPIFFE-based short-lived certificates.

## Scenario-Based Questions

**Q: Your team notices that after a network partition heals, some services continue to route to instances that were terminated during the partition. What is the most likely cause?**

- Eureka self-preservation mode prevented eviction of those instances during the partition. Clients cached the registry and did not refresh aggressively enough. The fix involves reducing the client-side registry cache TTL and implementing circuit breakers that mark instances as dead on connection failure.
- **Interview follow-up:** How would you design a circuit breaker that distinguishes between a dead instance and a slow instance?

**Q: A financial trading system uses ZooKeeper for both leader election and service discovery. During a ZooKeeper leader election, the entire trading platform becomes unresponsive. Why?**

- ZooKeeper is a CP system; during leader election (ZAB protocol), it enters a read-only state where no writes (registering or unregistering services) are possible. Services that cannot start without registration will block, and those needing to discover new instances will see stale data.
- **Interview follow-up:** How would you redesign the system to remain available during ZooKeeper leader elections without sacrificing consistency guarantees?

**Q: A Kubernetes cluster runs 500 microservices. After a rolling deployment, traffic still reaches old pod IPs for several minutes. What went wrong?**

- The readiness probe is not properly configured or is too lenient. kube-proxy watches Endpoints but only removes pods once the readiness probe fails. If the probe only checks process liveness (not application readiness), old pods remain in the Service Endpoints until the next sync cycle.
- **Interview follow-up:** How would you implement a gradual traffic drain for a pod being terminated during a rolling update?

## Interview Questions

- **Explain the difference between client-side and server-side service discovery. When would you choose one over the other?**
  - Client-side: the consumer queries the registry directly and selects an instance (e.g., Eureka + Ribbon). Server-side: a router/load balancer handles registry queries and forwards requests (e.g., Kubernetes Service + kube-proxy). Choose client-side when latency is critical and you control the client stack; choose server-side when polyglot environments or simplicity at the client matter.

- **What problem does Eureka's self-preservation mode solve, and what risk does it introduce?**
  - It prevents cascading registry flapping during network partitions by pausing instance eviction when heartbeat success rate drops below 85%. The risk is stale entries that cause traffic to dead instances after the partition heals.

- **How does ZooKeeper achieve consistency for service discovery?**
  - ZooKeeper uses the ZAB consensus protocol: a single leader processes all writes; writes commit to a majority before being acknowledged. Ephemeral znodes are automatically removed on session expiration, ensuring the registry stays consistent. However, during leader election, the cluster is unavailable for writes.

- **Compare Consul and Eureka for service discovery in a multi-datacenter deployment.**
  - Consul supports WAN gossip and federated datacenters natively with strong consistency via Raft across datacenters. Eureka is AP-focused; each datacenter runs its own Eureka cluster and peers across datacenters with asynchronous replication, which can lead to inconsistency but survives datacenter-level partitions.

- **How does Kubernetes DNS-based service discovery differ from a dedicated service registry like Consul?**
  - Kubernetes DNS (CoreDNS) returns the cluster IP of a Service, which kube-proxy load balances to healthy pods. Consul DNS returns individual instance IPs and supports richer health checks, KV metadata, and mTLS (Connect). Kubernetes DNS is simpler and deeply integrated with the platform but less flexible for advanced routing policies.

## Developer Recommendations

- **Use AP discovery for most microservices and CP discovery for critical coordination**
  - Most services benefit from the availability guarantees of AP systems like Eureka. Reserve CP systems like ZooKeeper or etcd for leader election, distributed locks, and configuration that requires strong consistency.
  - Implementation: Run Eureka or Consul in AP mode (Consul in non-strict-consistency mode for registration queries) as the primary registry; use a separate ZooKeeper ensemble solely for coordination work.

- **Implement health-check-aware load balancing on the client side**
  - Even with server-side discovery, client-side logic should account for connection failures, timeouts, and circuit breakers to handle stale registry entries.
  - Implementation: Use Spring Cloud LoadBalancer with a custom `ServiceInstanceListSupplier` that filters instances based on recent failures; or use gRPC's built-in health checking protocol.
  - **Production story:** At one streaming company, client-side circuit breakers reduced error rates during Eureka self-preservation events by 97% by dropping unresponsive instances from the local cache within 500ms.

- **Always configure readiness probes to reflect application-level health**
  - A readiness probe that only checks TCP port liveness is insufficient; it should validate that dependencies (database, cache, upstream services) are reachable.
  - Implementation: Expose a `/health/readiness` endpoint that checks database connections, message queue connectivity, and cache reachability. Use a short failure threshold so unhealthy instances are removed from the load-balancing pool quickly.

- **Monitor registry health separately from service health**
  - The service registry is a critical infrastructure component; its own health (leader status, replication lag, heartbeat success rate) should be monitored independently.
  - Implementation: Set up Prometheus alerts on Eureka's `renewal-per-minute` vs `threshold`, Consul's raft leadership and election metrics, or ZooKeeper's outstanding requests queue depth.
