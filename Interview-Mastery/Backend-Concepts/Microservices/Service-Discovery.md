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

**Q: Your service registry uses TTL-based health checks, but during a network partition, instances that are still healthy are incorrectly evicted. What is happening and how do you fix it?**

- The health check TTL is too short relative to the network partition detection latency. Eureka-style self-preservation should throttle eviction. Consul allows tuning of `check_update_interval` and deregistration thresholds. The fix involves increasing TTL values and implementing self-preservation or circuit-breaker-on-client logic.
- **Interview follow-up:** How would you design a health check that distinguishes between an unreachable instance and a truly dead instance?

**Q: A multi-region deployment uses Eureka for service discovery. When the network link between regions goes down, services fail to discover instances in the other region. Why?**

- Eureka clusters replicate asynchronously. During a network partition, replication fails, so each region's registry becomes stale for the other region's instances. Because Eureka is AP-oriented, it continues serving stale data. The fix is to use a multi-region discovery strategy (region-specific registries with local-first routing and cross-region fallback) rather than a single global registry.
- **Interview follow-up:** Should you use AP or CP discovery for cross-region communication? Under what circumstances would you choose one over the other?

**Q: Your Kubernetes services use DNS for discovery, but after a rolling update, some requests still go to terminated pods and fail. How do you fix this?**

- The readiness probe is not failing quickly enough. When a pod receives a SIGTERM during rolling update, it should immediately begin failing its readiness probe so kube-proxy removes it from Endpoints. Implement a pre-stop hook that sleeps for a few seconds (or calls a `/health/readiness` endpoint that returns non-200) before the container exits.
- **Interview follow-up:** Kubernetes uses iptables for load balancing. How would you handle the connection draining problem where in-flight requests are dropped when a pod terminates?

**Q: You deploy a new microservice that uses Finagle for client-side discovery against ZooKeeper. The service takes 30 seconds to start because it blocks on ZooKeeper connection. How do you improve startup time?**

- ZooKeeper client connection establishment includes a session negotiation timeout. Reduce the timeout or allow the service to start with cached registry data before the connection is fully established. Configure the service to retry registration asynchronously rather than blocking the startup sequence. Consider using a lighter-weight AP registry (Eureka/Consul) for discovery while reserving ZooKeeper for coordination.
- **Interview follow-up:** If you start the service with cached data and the cached data is stale, what safeguards do you implement to prevent routing to dead instances?

**Q: A microservice calls another service using DNS-based discovery, but during a traffic spike, the DNS server becomes overwhelmed and returns SERVFAIL. How do you make the client resilient to DNS failures?**

- Implement client-side DNS caching with a local resolver (e.g., `dnsjava`, `ndots:5` in Kubernetes). Cache successful DNS lookups with a reasonable TTL and fall back to the last known good IPs when DNS fails. Combine with a circuit breaker that treats DNS failures as upstream failures and opens the circuit to the affected service.
- **Interview follow-up:** What are the trade-offs of using a longer DNS cache TTL versus a shorter one for service discovery?

**Q: Your Consul cluster serves 200 services with 500 instances each. After adding 50 more services, Consul query latency triples. What is likely happening?**

- Consul uses Raft consensus, and each query may require a leader read. As the number of services grows, the catalog size increases and Consul's in-memory index lookups slow. The fix is to allow stale reads for service discovery queries (`?stale` parameter) and use prepared queries for frequently accessed services. Also consider scaling the Consul server pool or partitioning services into separate Consul datacenters.
- **Interview follow-up:** How would you decide between scaling the Consul cluster vertically (more resources) versus horizontally (more servers) when facing query latency issues?

**Q: In a hybrid cloud deployment (AWS + on-premise), on-premise services need to discover AWS-hosted services and vice versa. How do you design a unified discovery mechanism?**

- Run a Consul cluster that spans both environments with WAN gossip linking multiple datacenters. Each datacenter operates its own Consul servers, and WAN replication propagates service entries across datacenters. Alternatively, use a global registry backed by a strongly consistent store (etcd across regions) with local caches. For Kubernetes-based environments, use Submariner or similar multi-cluster service discovery.
- **Interview follow-up:** How do you handle the latency difference between local and remote service calls? How do you prevent the system from routing a remote call when a local instance is available?

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

- **What is the difference between Eureka and Consul for service discovery?**
  - Eureka is AP-oriented (favours availability over consistency), uses heartbeats and self-preservation. Consul is CP-oriented (uses Raft consensus), offers richer health checks, KV store, and DNS interface.
  - Choose Eureka for systems that must remain available during network partitions; choose Consul when strong consistency and richer health checking are required.

- **How does service registration work in Kubernetes?**
  - Pods are registered as Endpoints or EndpointSlices based on matching labels to a Service selector. kube-proxy watches the API server for Endpoint changes and programs iptables or IPVS rules to route traffic to healthy pod IPs.
  - Readiness probes determine which pods receive traffic; failed probes remove the pod from the Endpoint object.

- **What is the purpose of heartbeats in service discovery?**
  - Heartbeats signal that a service instance is still alive and capable of serving traffic. If a heartbeat is missed for a configurable threshold, the registry marks the instance as unhealthy and eventually evicts it.
  - Heartbeat frequency involves a trade-off: too frequent increases registry load; too infrequent delays detection of dead instances.

- **How do you handle service discovery in a polyglot microservices environment?**
  - Use a server-side discovery mechanism (e.g., Kubernetes Services, Kong, or a load balancer) so that clients in any language can discover services without needing a specific client library.
  - Alternatively, use DNS-based discovery (Consul DNS) which works with any language that supports standard DNS lookups.

- **What is the difference between a service registry and a load balancer?**
  - A service registry maintains a list of available service instances and their network locations. A load balancer distributes incoming traffic across those instances based on a policy (round-robin, least connections).
  - They often work together: the load balancer queries the registry to obtain the list of healthy instances to forward traffic to.

- **How does gRPC service discovery differ from REST service discovery?**
  - gRPC uses HTTP/2 persistent connections and typically relies on DNS-based discovery or a dedicated name resolver (e.g., `dns:///`, `consul:///`, Kubernetes DNS). The name resolver returns IP addresses, and the gRPC client maintains a long-lived connection pool.
  - gRPC also supports client-side load balancing (pick_first, round_robin) and can use a dedicated resolver for custom service registries.

- **What are the challenges of using DNS for service discovery in Kubernetes?**
  - DNS caching at multiple levels (kubelet, node, application) can cause stale lookups. Short TTLs increase DNS query load on CoreDNS. DNS-based discovery lacks rich health check information (only readiness probe status).
  - Solutions: use headless services with DNS SRV records for client-side discovery, or use a service mesh that bypasses DNS for inter-service communication.

- **How do you monitor the health of a service registry?**
  - Monitor metrics specific to the registry: Eureka — renewal rate vs threshold, evicted instance count; Consul — raft leadership status, election metrics, query latency; ZooKeeper — outstanding requests, session count, election time.
  - Set alerts on: renewal rate dropping below threshold, leader elections occurring too frequently, or query latency exceeding a baseline.

- **What is the trade-off between AP and CP service discovery?**
  - AP (Eureka): the registry remains available during partitions but may serve stale or inconsistent data. Clients may route to dead instances. CP (Consul, ZooKeeper): the registry is strongly consistent but may become unavailable during leader elections or partitions.
  - The right choice depends on whether your system prioritizes availability (no downtime) or consistency (no stale data) during failures.

- **How does a sidecar proxy participate in service discovery?**
  - In a service mesh (Istio, Linkerd), the sidecar proxy performs service discovery on behalf of the application pod. The proxy receives endpoint updates from the control plane via xDS APIs and load balances traffic to healthy instances.
  - The application connects to `localhost` and the sidecar handles all routing, discovery, and load balancing transparently.

- **What is the role of the `PreferSameZone` feature in client-side service discovery?**
  - It instructs the client to prefer instances in the same availability zone or region when multiple instances are available. This reduces cross-zone latency and data transfer costs.
  - Implemented in Netflix Ribbon and Spring Cloud LoadBalancer via the `ZonePreferenceServerListFilter`.

- **How do you test service discovery during development?**
  - Use a local registry instance (Eureka server, Consul agent) running in a Docker container. Services register against it using dev profiles. Use a static registry for unit tests and integration tests to avoid external dependencies.
  - For end-to-end tests, deploy a full environment with the real registry and multiple service instances to verify discovery and load balancing.

- **Explain the concept of "sticky sessions" in service discovery and when to avoid them.**
  - Sticky sessions (session affinity) route all requests from the same client to the same instance, typically using a cookie or source IP hash. They are useful for stateful services but cause load imbalance and complicate rolling updates.
  - Avoid sticky sessions in microservices — prefer stateless services that can be routed to any instance. If state is needed, use an external store (Redis, database) instead of instance-local state.

- **What happens when a service instance crashes without deregistering from the registry?**
  - The registry will not immediately know the instance is dead. It relies on heartbeat timeout or health check failure to eventually evict the instance. During this window, clients may attempt to call the dead instance and receive connection errors.
  - Mitigations: configure aggressive heartbeat failure thresholds, use client-side circuit breakers, and set up a grace period for deregistration on shutdown.

- **How does service discovery differ between virtual machines and containers?**
  - On VMs, instance IPs are relatively stable and services have fixed hostnames. In containers, pods are ephemeral with dynamic IPs, making service discovery essential. Containers also benefit from platform-native discovery (Kubernetes Services, DNS).
  - On VMs, registration typically happens at application startup; in containers, registration is more dynamic due to auto-scaling and rolling updates.

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
