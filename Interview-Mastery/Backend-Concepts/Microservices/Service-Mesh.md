# Service Mesh

## Overview

- **Definition** — A service mesh is a dedicated infrastructure layer that handles inter-service communication, security, observability, and traffic management via sidecar proxies deployed alongside each service instance.
- **Why It Exists** — As the number of microservices grows, cross-cutting concerns like retries, timeouts, circuit breaking, mutual TLS, distributed tracing, and traffic splitting become too complex to implement in each service; a service mesh moves these concerns out of the application code and into the infrastructure layer.
- **Historical Context** — Early microservices relied on client libraries (Hystrix, Finagle) for resilience, but these required language-specific dependencies and upgrades. The sidecar proxy pattern emerged with Linkerd (2016) and Istio (2017), inspired by Twitter's Finagle and Google's internal infrastructure.
- **Key Concepts** — **Sidecar Proxy** — a separate container/process running alongside each service instance that intercepts all inbound/outbound traffic; **Control Plane** — centralised management component that configures proxies and enforces policies; **Data Plane** — all sidecar proxies collectively handling traffic; **mTLS** — mutual TLS encryption between proxies for service-to-service authentication; **xDS** — discovery APIs (LDS, RDS, CDS, EDS) for dynamic proxy configuration.

## Core Concepts

- **Sidecar Proxy Pattern**
  - An intermediary proxy (typically Envoy) is injected as a separate container in the same pod or spawned as a separate process.
  - All inbound and outbound traffic is redirected to the sidecar via iptables rules or eBPF programs.
  - The sidecar enforces routing rules, retries, timeouts, circuit breaking, and collects telemetry data.
  - The application code remains unaware of the mesh; it connects to localhost and the sidecar handles the rest.

- **Istio Architecture**
  - **Pilot** — translates high-level routing rules (VirtualService, DestinationRule) into Envoy configuration and distributes it via the xDS API; manages service discovery from the platform (Kubernetes, Consul, VM).
  - **Mixer** — (deprecated in newer Istio versions) was a centralised telemetry and policy enforcement component; replaced by per-proxy Envoy WASM extensions and the Telemetry API.
  - **Citadel** — manages certificate issuance and rotation for mTLS; automatically generates SPIFFE-compliant identities for each workload.
  - **Galley** — validates, configures, and distributes Istio configuration to other components; serves as the configuration ingestion layer.
  - **Ingress/Egress Gateways** — dedicated Envoy proxies for handling external traffic entering or leaving the mesh.

- **Envoy Proxy**
  - High-performance L4/L7 proxy written in C++ with a thread-per-core model and non-blocking event loop.
  - **xDS Protocol** — a set of APIs (Listener Discovery — LDS, Route Discovery — RDS, Cluster Discovery — CDS, Endpoint Discovery — EDS) that provide dynamic, incremental configuration.
  - **Hot Restart** — Envoy can be restarted without dropping connections by spawning a new process that inherits the listening sockets from the old process via Unix domain sockets.
  - Supports advanced features: HTTP/2 and gRPC natively, advanced load balancing (zone-aware, ring hash, least request), outlier detection, and Lua/WASM filter extensions.

- **mTLS Between Services**
  - Each sidecar proxy is issued a certificate by Citadel (or equivalent) containing the service identity (e.g., `spiffe://cluster.local/ns/default/sa/my-sa`).
  - All inter-service traffic is encrypted and authenticated at the proxy level; applications communicate over plain HTTP within the pod.
  - mTLS mode: STRICT (all traffic must use mTLS), PERMISSIVE (accept both mTLS and plaintext), DISABLE.
  - Certificate rotation is automatic (typically every 24 hours); old certificates are revoked gracefully.

- **Traffic Splitting**
  - **Canary Deployments** — route a small percentage of traffic to the new version (e.g., 5% v2, 95% v1) and gradually increase the percentage as confidence grows.
  - **Blue-Green Deployments** — route all traffic to the "blue" (current) version while deploying the "green" (new) version; switch all traffic at once via a route update.
  - Istio VirtualServices support weight-based routing, header-based routing, and mirroring.
  - Traffic splitting does not require changes to application code; only control-plane configuration is updated.

- **Observability**
  - **Telemetry** — Envoy generates detailed metrics (request count, latency, error codes, TCP stats) per service, per route.
  - **Tracing** — Envoy propagates trace headers (Zipkin, Jaeger, Datadog) and generates spans for each hop, enabling end-to-end distributed tracing.
  - **Kiali** — web-based UI for observing the mesh topology, service graphs, traffic flows, and configuration validation.
  - **Access Logs** — per-request logs with source, destination, response code, latency, and protocol metadata.

- **Linkerd (CNCF)**
  - Written in Rust for the data plane (Linkerd2-proxy) and Go for the control plane; known for being lightweight (under 10MB binary for the sidecar).
  - Uses the **tap** feature for live request-level observability without storing logs.
  - Simpler to install and operate than Istio; fewer CRDs and moving parts.
  - Does not support Envoy — uses its own custom proxy with a smaller feature set (no Lua/WASM filters, no advanced load balancing).
  - Supports mTLS, HTTP/1.x, HTTP/2, gRPC, traffic splitting, and retries.

- **Istio vs Linkerd Comparison**
  - **Performance** — Linkerd uses ~50% less CPU and memory per sidecar than Istio with Envoy; Istio offers richer feature set.
  - **Features** — Istio supports Envoy WASM extensions, TCP traffic management, external service mesh federation; Linkerd is more opinionated and limited to HTTP/gRPC traffic.
  - **Complexity** — Linkerd installs in minutes with fewer CRDs; Istio has a steeper learning curve and more moving parts (Pilot, Mixer/Citadel/Galley).
  - **Ecosystem** — Istio integrates with a wider range of third-party tools (Grafana, Prometheus, Kiali, Jaeger, SkyWalking); Linkerd integrates with Prometheus and its own dashboard.

## Common Mistakes

- **Sidecar overhead underestimated**
  - Each sidecar proxy adds CPU (per-request processing, TLS handshake) and memory (connection pools, filter chains) overhead; deploying hundreds of sidecars can cost significant cluster resources.
  - **Why it looks correct:** The sidecar is "just an extra container" but each Envoy instance reserves 50–128MB of memory minimum; in large clusters this amounts to tens of GB of overhead.
  - The fix: Set appropriate resource requests/limits on sidecars, tune buffer sizes, and avoid running sidecars for batch or cron job workloads.

- **mTLS performance impact ignored**
  - Enabling STRICT mTLS across all services adds per-connection TLS handshake overhead; while connection pooling amortises this, first requests (cold start) face significant latency.
  - **Why it looks correct:** Certificates rotate automatically and handshakes are fast, but in high-connection-churn environments the cost is non-trivial.
  - The fix: Use persistent connection pooling (HTTP/2 or gRPC) to minimise handshakes, configure TLS session resumption, and benchmark before enabling STRICT mode.

- **Debugging complexity ignored**
  - When a request fails in a mesh, the failure could be in the application, the sidecar, the control plane, or a combination; debugging requires understanding the full path through multiple layers.
  - **Why it looks correct:** The mesh makes "everything observable" but the sheer volume of data (access logs, metrics, traces) can obscure the root cause.
  - The fix: Use structured error propagation (HTTP `x-envoy-*` headers), set up service-level dashboards in Kiali, and teach the team how to read Envoy configuration dumps.

- **Istio control plane becomes a bottleneck**
  - The Istio control plane (Pilot + Galley) processes all configuration changes and pushes updates to every Envoy proxy; in large clusters, config updates can take minutes to propagate.
  - **Why it looks correct:** The control plane is designed for centralised management, but as the number of proxies grows, the update latency increases linearly.
  - The fix: Use Istio's shardable control plane (revision-based upgrades), limit the number of VirtualServices/DestinationRules, and use Istio 1.12+'s "delta xDS" for incremental updates.

## Real-World Scenarios

### Canary Deployment Gone Wrong with Istio

- A team deployed a canary (5% traffic) for a payment service. The new version had a memory leak; after 4 hours, the pod was OOM-killed, but Envoy continued to route 5% of traffic to the new pod during its crash loop.
- The symptom was increased 503 errors on 5% of payment requests, which initially seemed like a transient issue. The team had not set outlier detection (consecutive 5XX errors) to eject the failing pod from the mesh.
- They added outlier detection with a 30-second ejection time and configured a max ejection percentage of 50% to prevent cascading failures.

### Linkerd Migration at a Startup

- A 50-microservice startup migrated from manual client-side retry logic (Finagle) to Linkerd. The migration took 2 weeks and involved installing the control plane and injecting sidecars into all deployments.
- The immediate benefit was zero-code retries, timeouts, and mTLS. The team reduced their codebase by ~15,000 lines of resilience logic.
- They hit one issue: Linkerd's default 10-second timeout on all outbound HTTP requests broke a legacy batch processing service that made long-running queries. They excluded the batch service from the mesh.

### mTLS Performance Tuning at a Fintech

- A fintech company enabled ISTIO_MUTUAL (STRICT mTLS) across 200 services. After enabling, p99 latency for the authentication service increased from 50ms to 280ms.
- The root cause: the authentication service created a new gRPC connection for every request (no connection pooling). Each new connection required a full TLS handshake with certificate validation.
- They fixed by switching to gRPC persistent connections and enabling TLS session caching; p99 dropped back to 60ms.

## Scenario-Based Questions

**Q: After deploying Istio, your team notices that all service-to-service communication is encrypted (mTLS) and latency has increased by 30%. What common misconfiguration might cause this?**

- The services are likely creating new connections per request rather than using connection pooling. Each new HTTP/1.1 connection triggers a full TLS handshake. The fix is to switch to HTTP/2 or gRPC with persistent connections, enable TLS session resumption, and confirm that all services are using connection keep-alive.
- **Interview follow-up:** How would you verify that a specific service is correctly using connection pooling without modifying its code?

**Q: During a traffic split (10% canary), you notice that the canary service is receiving 15% of traffic. What is the likely cause?**

- The VirtualService weights are probabilistic, not perfectly precise. With low request volumes, the law of large numbers does not apply and the actual distribution can deviate significantly. The fix is to ensure sufficient request volume for precise splitting, or use header-based routing (e.g., "x-canary: true") for exact matching.
- **Interview follow-up:** How would you design a progressive delivery strategy that increases canary traffic based on automated success criteria (e.g., error rate, latency)?

**Q: A developer reports that after moving to a service mesh, debugging a failed request takes much longer because error messages are generic 503s. How can you improve debuggability?**

- Envoy returns 503 when the upstream is unhealthy or the circuit breaker is open. The team should inspect Envoy access logs for the actual response flags (e.g., `UF` — upstream failure, `UO` — upstream overflow). Enable the `x-envoy-*` response headers (x-envoy-upstream-service-time, x-envoy-decorator-operation) and create Kiali dashboards showing service-to-service traffic flow with error annotations.
- **Interview follow-up:** If Envoy access logs show "NR" (no route) for a specific request, what would you check in the mesh configuration?

## Interview Questions

- **What is the difference between a service mesh and an API gateway?**
  - A service mesh manages east-west traffic (inter-service communication) using sidecar proxies, handling concerns like mTLS, retries, circuit breaking, and observability. An API gateway manages north-south traffic (external-to-service) handling auth, rate limiting, and routing. They are complementary: a gateway sits at the edge, and a mesh operates inside the cluster.

- **How does Envoy's xDS protocol work?**
  - xDS (Discovery Service) is a set of gRPC streaming APIs: LDS (Listener config), RDS (Route config), CDS (Cluster config), EDS (Endpoint config). Envoy connects to the control plane (e.g., Istio Pilot) and receives configuration updates incrementally via long-lived gRPC streams. This allows dynamic routing changes without restarting Envoy.

- **Compare Istio and Linkerd. When would you choose Linkerd over Istio?**
  - Istio is feature-rich (Envoy WASM, TCP traffic management, external mesh federation, complex routing) but complex and resource-heavy. Linkerd is lightweight (Rust-based proxy, ~10MB binary), simple to install, and uses less CPU/memory. Choose Linkerd for small-to-medium clusters where simplicity and low overhead matter; choose Istio for large clusters requiring advanced traffic management, WASM extensions, or multi-cluster federation.

- **What problem does Citadel solve in Istio?**
  - Citadel (now integrated into istiod) manages certificate authorities and workload identity. It generates SPIFFE-compliant certificates for each service, automatically rotates them, and integrates with Kubernetes Service Accounts. Citadel ensures that mTLS between services uses verifiable, short-lived certificates without manual certificate management.

- **How does a sidecar proxy intercept traffic in Kubernetes?**
  - Using iptables rules injected by an init container (istio-init, linkerd-init). All inbound traffic to the pod's port and all outbound traffic from the pod is redirected to the sidecar proxy's port (e.g., 15006 for inbound, 15001 for outbound). The sidecar then applies routing rules before forwarding to the destination. eBPF-based approaches are emerging for lower overhead.

- **What is the purpose of a VirtualService in Istio?**
  - A VirtualService defines routing rules for a given host: which subsets (versions) of the destination service receive traffic, under what conditions (headers, weight), and with what retry/timeout/mirroring policies. It works with DestinationRule (which defines subsets, load balancer settings, and connection pool/outlier detection settings).

## Developer Recommendations

- **Start with a thin service mesh (Linkerd) and upgrade to Istio only if needed**
  - Most teams do not need the full Istio feature set initially. Linkerd's simpler operation, lower resource usage, and faster learning curve provide 80% of the value (mTLS, retries, timeouts, observability) with 20% of the complexity.
  - Implementation: Install Linkerd with `linkerd install | kubectl apply -f -`, inject sidecars with `linkerd inject deployment/<name>`, and verify with `linkerd check`.
  - **Production story:** A 30-microservice startup achieved zero-code retries and mTLS with Linkerd in 3 days; they spent 3 months trying to stabilise Istio before switching.

- **Perform a gradual mTLS rollout**
  - Enable mTLS in PERMISSIVE mode first to ensure all services accept both TLS and plaintext; switch to STRICT mode gradually after verifying that no plaintext-only clients exist.
  - Implementation: Enable PeerAuthentication with PERMISSIVE, monitor the mesh dashboard for plaintext connections, and switch to STRICT after a week of zero plaintext traffic.

- **Set resource limits on sidecar proxies**
  - Without limits, sidecars can consume excessive memory and CPU, especially during config updates or traffic spikes.
  - Implementation: Add a `sidecar.istio.io/proxyCPU` and `sidecar.istio.io/proxyMemory` annotation to deployments. Typical values: `100m CPU`, `128Mi memory` for Envoy; Linkerd requires `~50m CPU` and `~64Mi memory`.

- **Enable outlier detection early**
  - Outlier detection (consecutive 5XX, connection failures) ejects unhealthy pods from the mesh automatically, preventing cascading failures.
  - Implementation: Configure a DestinationRule with `outlierDetection` — `consecutive5xxErrors: 5`, `interval: 30s`, `baseEjectionTime: 30s`, `maxEjectionPercent: 50`.
