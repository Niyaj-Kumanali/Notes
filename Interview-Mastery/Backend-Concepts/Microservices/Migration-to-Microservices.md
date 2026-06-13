# Migration to Microservices

## Overview

- **Definition** — Migration to microservices is the process of incrementally decomposing a monolithic application into independently deployable, loosely coupled services communicating over a network.
- **Why It Exists** — Monoliths become difficult to scale, deploy, and maintain as codebases grow; teams struggle with merge conflicts, long CI/CD cycles, and inability to scale individual components independently.
- **Historical Context** — The Strangler Fig pattern was first described by Martin Fowler in 2004; early adopters (Amazon, Netflix) proved the model in the late 2000s; by the 2010s, microservices became the default architectural direction, with many teams attempting migration during the "microservices hype" and learning hard lessons.
- **Key Concepts** — **Strangler Fig** — gradually replace monolith functionality with new services; **Bounded Context** — service boundaries aligned with domain-driven design subdomains; **Anti-Corruption Layer** — translation layer between old and new systems; **Feature Flag** — toggle that enables/disables new functionality at runtime; **Domain Event** — event published when business state changes, used for cross-service data sync.

## Core Concepts

- **Strangler Fig Pattern**
  - Identify a bounded piece of functionality (e.g., user authentication, product search) and extract it into a standalone service.
  - Route calls for that functionality to the new service while keeping the rest of the monolith unchanged.
  - The monolith is "strangled" incrementally until nothing remains — or the team decides to stop.
  - Key requirement: the monolith and new services must share the same front-end/client interface so that callers see no difference.
  - Proxy or gateway layer intercepts and routes requests to either the monolith or the new service based on URL, header, or feature flag.

- **Decomposition Strategies**
  - **Business Capability** — split based on what the business does (e.g., Orders, Payments, Inventory, Shipping). Aligns with organisational structure (Conway's Law).
  - **Subdomain (DDD)** — split based on domain-driven design subdomains: Core Domain (competitive advantage), Supporting Domain (necessary but not differentiating), Generic Domain (off-the-shelf solutions).
  - **Transaction Boundary** — each service owns its data and no transaction spans multiple services; if two entities are always updated together, they belong in the same service.
  - **Team Structure** — each service should be owned by a single team; avoid services that require coordinated changes across multiple teams.

- **Database Splitting**
  - Start with a shared database and move to separate databases per service gradually.
  - Phase 1: The monolith owns the shared database; the new service reads from a shared read-only replica.
  - Phase 2: The new service gets its own schema with its own tables; data is synced from the monolith via replication or dual writes.
  - Phase 3: The new service gets its own database instance; the monolith no longer accesses those tables.
  - Each phase requires data migration, backfill scripts, and validation that read/write paths are correct.

- **Feature Flags**
  - Toggle new service behaviour on/off at runtime without redeploying code.
  - Enable testing the new service with a subset of users or requests before full rollout.
  - Must be cleaned up after the migration is complete; lingering flags create dead code paths.
  - Use a feature flag management platform (LaunchDarkly, Split) for scaled rollouts and kill switches.

- **Domain Events for Data Sync**
  - When a monolith creates/updates data that the new service needs, the monolith publishes a domain event (e.g., `OrderPlaced`, `UserUpdated`).
  - The new service subscribes to these events and updates its own database accordingly.
  - Events are usually published to a message broker (Kafka, RabbitMQ) and should be idempotent.
  - This keeps services data-independent while the migration is in progress.

- **Anti-Corruption Layer (ACL)**
  - A translation layer between the monolith's legacy model and the new service's domain model.
  - The ACL translates calls, data formats, and protocols between old and new systems.
  - Prevents the legacy model's design choices (e.g., poorly named fields, inconsistent data formats) from leaking into the new service.
  - Can be a separate service, a facade, or a set of service interfaces.

- **Challenges**
  - **Distributed Transactions** — what was a single ACID transaction in the monolith now requires coordination across services; solved via Sagas or eventual consistency.
  - **Network Latency** — inter-service calls are slower than in-process calls; requires optimisation (caching, batching, asynchronous APIs).
  - **Testing** — integration tests require multiple services running, making test setup complex; contract testing (Pact) becomes essential.
  - **Deployment Coordination** — services with shared dependencies must be deployed in order; automate with CI/CD pipelines and backward-compatible APIs.

## Common Mistakes

- **Partial migration causing data inconsistency**
  - During migration, data is written by both the monolith and the new service without proper synchronisation, leading to conflicting or duplicated records.
  - **Why it looks correct:** Dual writes seem like a simple way to keep systems in sync, but one side often fails silently — the monolith writes to the old table, the new service writes to its own table, and no reconciliation process exists.
  - The fix: Choose a single source of truth for each data entity and route writes through that truth; use domain events or CDC (Change Data Capture) for one-way sync; implement reconciliation jobs that compare and resolve differences.

- **Feature flags accumulating indefinitely**
  - Feature flags added during migration are never removed after the migration completes, leaving dead code paths and conditional logic throughout the codebase.
  - **Why it looks correct:** Cleaning up flags feels like a low-priority technical debt item, but each flag adds maintenance cost and increases cognitive load for developers.
  - The fix: Enforce a feature flag lifecycle policy — every flag has an owner and an expiry date; flags older than 90 days are automatically flagged by CI; schedule a flag cleanup sprint after each major migration milestone.

- **Slow decomposition stalling the effort**
  - The team tries to perfect the service boundaries upfront, analysing for months and producing a perfect service design that never gets implemented.
  - **Why it looks correct:** Decomposition is a big design decision; but analysis paralysis means the monolith continues to grow while the team debates theoretical boundaries.
  - The fix: Use a "screaming architecture" approach — extract the first service based on a clear pain point (slowest endpoint, most frequent deployments) within 2 weeks; iterate on boundaries based on real-world feedback.

- **Big Bang rewrite instead of strangler fig**
  - The team decides to rewrite the entire monolith as microservices in one project, often taking 12–18 months with no value delivered until the end.
  - **Why it looks correct:** Starting from a clean slate feels more satisfying than gradually strangling the monolith; but the rewrite misses years of bug fixes and edge cases hidden in the monolith.
  - The fix: Always use the Strangler Fig pattern. Extract service-by-service, keeping the monolith running. Each extraction delivers immediate value (independent deployability, independent scaling).

## Real-World Scenarios

### Amazon's Monolith to Microservices Migration

- Amazon's retail monolith was strangulated over several years, starting with the product catalogue service and then the shopping cart.
- Each team extracted their service, built a BFF (Backend for Frontend) layer, and routed traffic via a home-grown API gateway.
- Key success factor: each service owned its data completely (separate database), and the monolith was eventually retired when it handled no real business logic.

### Uber's Trip Service Decomposition

- Uber's monolith started as a single Python application handling dispatch, pricing, and payments. As the company scaled globally, deployments became bottlenecked.
- They extracted the dispatch service first (the most latency-sensitive and rapidly changing component), then pricing, then payments.
- The extraction created a split-brain data problem for fares (dispatch stored estimated fares, billing stored actual fares) that took months to resolve with a reconciliation service.

### Fintech Strangulation with Anti-Corruption Layer

- A European fintech bank spent 3 years extracting services from a 15-year-old Java monolith.
- They built an ACL that wrapped the monolith's SOAP APIs into REST endpoints, allowing new microservices to call legacy functionality without depending on the monolith's internal model.
- The ACL became a bottleneck after 2 years because every new feature required ACL changes; they eventually split the ACL into per-domain ACL services.

## Scenario-Based Questions

**Q: Your team extracts the payment service from the monolith. After extraction, some orders are processed correctly but others show "payment pending" indefinitely. What is the likely cause?**

- The monolith and the payment service ended up in a dual-write inconsistency. The monolith writes `order.state = PAID` to its database, but the payment service writes to its own database and the events are not synchronised. The fix is to implement a reconciliation job that compares monolith orders with payment service transactions and marks inconsistencies for manual review.
- **Interview follow-up:** How would you design the reconciliation job to handle high volumes without creating a feedback loop?

**Q: A team plans to extract the search functionality from a monolith. After 6 months of design, they have not extracted any service. What advice do you give?**

- Analysis paralysis. Extract the search endpoint as a standalone service in 2 weeks — even if it calls back to the monolith's database. Then incrementally move data and logic over time. The first extraction should target the pain point: search is slow and uses too many resources; extracting it immediately reduces load on the monolith.
- **Interview follow-up:** How do you decide which slice of functionality to extract first when there is no obvious single pain point?

**Q: After extracting the inventory service, the monolith and the inventory service both write to the `inventory` table. Sometimes inventory counts differ. How do you fix this?**

- Assign the inventory service as the single source of truth for inventory data. Route all inventory writes through the inventory service via its API. The monolith should read inventory from the inventory service (or its read replica) and never write to inventory tables directly. Use a one-time data migration to sync the initial state.
- **Interview follow-up:** How would you handle a scenario where the monolith's old reporting queries still need direct database access for the next 6 months?

**Q: Your team migrates the user profile service out of the monolith. After the migration, the monolith's login endpoint still works, but users cannot update their profiles. What went wrong?**

- The login endpoint likely reads from the monolith's user table, but profile updates go to the new user service. The monolith's login code still validates credentials against the old table, which is no longer updated. Fix by routing all authentication flows through the new user service, even for the monolith. The monolith should call the user service's API for both read and write operations.
- **Interview follow-up:** How would you handle session management during the migration when the monolith and the new service both need access to the user's session data?

**Q: A manufacturing company wants to migrate from a 20-year-old monolith to microservices. The monolith has no tests, no CI/CD, and the only developer who understands the full codebase is leaving in 2 months. What is your approach?**

- Do not attempt a full migration. Focus on the highest-risk, highest-value extraction: the module that changes most frequently and causes the most production issues. Even one extracted service provides independent deployability and reduces risk. Apply the strangler fig pattern with an anti-corruption layer. Capture business knowledge from the leaving developer via architecture decision records (ADRs) and detailed documentation before they depart.
- **Interview follow-up:** The leaving developer has most of the business rules memorized but never documented them. How do you extract critical business logic before they leave without slowing down the migration?

**Q: Your team extracts the order service, but the monolith still needs to read order data for its legacy reporting dashboard. The reporting dashboard queries the database directly with complex SQL joins. How do you support this without coupling?**

- Create a read-only database replica of the order service's database for the monolith's reporting queries. The order service publishes data change events that a sync process applies to the reporting replica. The monolith's reporting queries run against this read-only replica, and the monolith never writes to the order service's tables directly. Eventually, replace the legacy reports with new ones that query the order service API.
- **Interview follow-up:** The reporting queries use JOINs across multiple tables that are now in different services. How would you redesign reporting without cross-service JOINs?

**Q: After extracting the notification service, the monolith crashes when the notification service is temporarily unavailable. How do you decouple the monolith from the new service's availability?**

- The monolith should communicate with the notification service asynchronously. Instead of making a synchronous HTTP call, the monolith publishes a `NotificationRequired` event to a message queue (Kafka, RabbitMQ). The notification service consumes events when available. This ensures the monolith remains operational even when the notification service is down. Implement a dead-letter queue for failed notifications.
- **Interview follow-up:** The business requires that some notifications (password reset, payment confirmation) are delivered within 30 seconds. How does async messaging meet this SLA?

**Q: Your team extracts the product catalog service. The new service performs well, but the monolith's API Gateway is still routing traffic to both the monolith and the new service inconsistently — some users see old product data, others see new. Why?**

- The feature flag or routing rule used to switch traffic between the monolith and the new service is not consistent for the same user across requests. If routing is based on a percentage (e.g., 50% to monolith, 50% to new service), the same user may hit different backends on subsequent requests. Fix by implementing user-consistent routing (hash the user ID to deterministically choose the backend) and ensure data is consistent between both backends during the transition.
- **Interview follow-up:** Once all users are routed to the new service, how do you validate that the monolith's product catalog code can be safely decommissioned without causing regressions?

**Q: A fintech company migrates the transaction processing service. The new service has a bug that duplicates transactions for 0.1% of users. The bug goes undetected for two weeks. How do you prevent this in future migrations?**

- Implement a data reconciliation pipeline that runs daily during migration and compares transaction records between the monolith and the new service. Use checksums or business-key-level comparison to detect discrepancies. In addition, deploy a canary extraction: route only internal testers to the new service for the first week, then 1% of users, gradually increasing while monitoring error rates and reconciliation results.
- **Interview follow-up:** How would you design a reconciliation pipeline that can detect and automatically fix data inconsistencies without requiring human intervention for every discrepancy?

**Q: Your team uses a shared database during migration. The DBA reports that the database CPU is at 95% because the new service and the monolith both run heavy queries simultaneously. How do you reduce the load?**

- The new service should not share the same database instance as the monolith during migration if both run production workloads. Use a read replica for the monolith's reporting queries, or schedule the migration work during off-peak hours. If sharing is unavoidable, implement database query optimization: add missing indexes, reduce query frequency, and batch writes. Prioritize the new service's queries over the monolith's queries for critical paths.
- **Interview follow-up:** The monolith's queries cannot be changed because the source code is too risky to modify. How do you optimize database performance without changing the monolith's queries?

## Interview Questions

- **Explain the Strangler Fig pattern and its key advantages over a Big Bang rewrite.**
  - The Strangler Fig pattern incrementally replaces monolith functionality with new services while the monolith continues to serve traffic. Advantages: continuous delivery of value (each extraction is deployable independently); risk reduction (small changes with immediate rollback); preservation of business logic (edge cases and bug fixes are carried over incrementally); organisational learning (team adapts to microservices gradually).

- **What is an Anti-Corruption Layer and when should you use one?**
  - An ACL translates between the monolith's legacy domain model and the new service's domain model. Use it when the legacy model has design flaws (inconsistent naming, denormalised data, tight coupling) that you do not want to propagate to the new service. The ACL typically lives as a separate service or a facade that provides clean APIs to the new service while translating calls back to the monolith.

- **How do you split a monolithic database into multiple service-specific databases without downtime?**
  - Use a phased approach: Phase 1 — create new tables in the shared database for the new service, with triggers to sync from old tables. Phase 2 — move the new service to its own schema in the shared database. Phase 3 — migrate to a separate database instance, with application-level dual writes and reconciliation jobs. Read operations always hit the new database; write operations go to both until Phase 3 is complete.

- **What role do feature flags play in microservices migration?**
  - Feature flags allow you to toggle between the monolith and the new service for specific user segments (e.g., internal testers, 1% of users, all users) without redeploying. They enable canary deployment of new services, instant kill-switches if the new service fails, and gradual rollout with automated rollback on error rate thresholds.

- **How do you handle distributed transactions during a migration?**
  - Avoid distributed transactions. Redesign the workflow to use eventual consistency with Sagas (choreography or orchestration). During the migration, use domain events to synchronise state between the monolith and the new service. Accept that there will be a window of inconsistency and design compensating actions (refunds, retries, notifications) for when inconsistency is detected.

- **What is Conway's Law and how does it affect microservices migration?**
  - Conway's Law: "Organisations design systems that mirror their communication structure." If your organisation has two teams (frontend and backend), you will end up with two microservices (frontend service and backend service). To migrate successfully, align service boundaries with team boundaries; avoid creating services that require cross-team coordination for every change.

- **How do you decide which service to extract first from a monolith?**
  - Extract the module that causes the most pain: the one with the most frequent deployments, the highest resource usage, the tightest database lock contention, or the most bugs. This generates immediate value and organizational momentum.
  - Avoid extracting the most complex module first — start with a well-understood, bounded module to prove the migration pattern before tackling the hardest parts.

- **What is the role of an Anti-Corruption Layer (ACL) in migration?**
  - An ACL is a translation layer between the monolith's legacy model and the new service's domain model. It translates calls, data formats, and protocols, preventing legacy design flaws from leaking into the new service.
  - The ACL can be a separate service, a facade, or a set of interfaces that live in the monolith's codebase but call the new service.

- **How do you handle data migration when splitting a database?**
  - Phase 1: The new service reads from a shared read replica while writes go to the monolith. Phase 2: Dual writes — both systems write to their own databases, with a reconciliation job to detect and fix inconsistencies. Phase 3: The new service owns its database exclusively.
  - Always have a rollback plan: keep the old tables intact until the migration is fully validated in production.

- **What are the risks of feature flags in migration?**
  - Feature flags can accumulate indefinitely, creating dead code paths and increasing cognitive load. They can introduce race conditions if not properly synchronized across distributed systems. They add testing complexity (need to test with flag on and off).
  - Mitigations: enforce flag lifecycle policies (owner, expiry date, automated cleanup), use a feature flag management platform, and schedule regular flag cleanup.

- **How do you handle contract changes during migration?**
  - Maintain backward compatibility: old API versions continue to work while clients migrate to new versions. Use versioned APIs and deprecation headers. For internal APIs between the monolith and new services, use an anti-corruption layer that translates between old and new contracts.
  - Coordinate contract changes via API review boards or consumer-driven contract testing.

- **What is the difference between a big-bang rewrite and strangler fig migration?**
  - Big-bang rewrite: rewrite the entire system from scratch and switch over at a single point in time. High risk, long time to value, likely to miss edge cases. Strangler fig: incrementally replace functionality while the old system continues running. Low risk, continuous value delivery, preserves business logic.
  - Strangler fig is almost always the better choice for production systems.

- **How do you handle reporting and analytics during migration?**
  - Create a reporting database that aggregates data from both the monolith and new services. Use change data capture (CDC) or scheduled ETL jobs to sync data from all sources into a unified reporting schema.
  - Avoid JOINs across service boundaries in the reporting layer. Each service should provide its own reporting API or export data to a shared data warehouse.

- **What is the role of containerization in microservices migration?**
  - Containerization (Docker) provides consistent environments for each service, simplifies deployment, and enables orchestration (Kubernetes). During migration, containers allow teams to deploy new services alongside the monolith without environment conflicts.
  - Containerization also enables independent scaling, rolling updates, and resource isolation per service.

- **How do you test a service extraction before cutting over production traffic?**
  - Run the new service in shadow mode: the monolith calls the new service in parallel and compares responses but does not use the new service's response for the actual operation. Use feature flags to enable the new service for internal testers first.
  - Validate data consistency between the monolith and the new service using reconciliation jobs. Monitor error rates, latency, and data integrity before routing production traffic.

- **How do you handle authentication and authorization during migration?**
  - The auth system must span both the monolith and new services. Centralize authentication at the API Gateway or use a shared auth service that both the monolith and new services call. Store user sessions in a shared store (Redis) or use JWT tokens.
  - During migration, the new services should use the same auth mechanism as the monolith to avoid a split auth state.

- **What is the impact of Conway's Law on service boundaries?**
  - Services will naturally align with team communication structures. If two teams must coordinate for every change, their services are likely too coupled. Organize services so that each team has end-to-end ownership, with well-defined APIs for cross-team communication.
  - During migration, this means extracting services based on team ownership boundaries, not just technical boundaries.

- **How do you handle logging during migration when requests span the monolith and new services?**
  - Implement distributed tracing from day one of the migration. Propagate a trace ID (W3C traceparent) from the API Gateway through the monolith to the new services. Both the monolith and new services log this trace ID.
  - Ship all logs to a centralized platform (ELK, Loki) so that a single trace ID can correlate logs across all components, even during the transition period.

- **What is the difference between event-driven and API-driven communication during migration?**
  - Event-driven: the monolith publishes events when state changes occur; new services consume these events and update their own state. This decouples the monolith from new services but introduces eventual consistency.
  - API-driven: the monolith makes synchronous API calls to new services. This is simpler to implement but creates tighter coupling and latency. Use event-driven for data synchronization and API-driven for operations requiring immediate consistency.

- **How do you know when the migration is complete?**
  - The migration is complete when all business functionality has been extracted from the monolith, the monolith is decommissioned (or reduced to a read-only legacy system), and all traffic routes directly to microservices.
  - Success criteria: zero traffic to the monolith, no shared database tables, independent deployability of all services, and feature flags related to migration have been removed.

## Developer Recommendations

- **Extract services based on pain points, not theoretical purity**
  - The first service to extract should be the one that solves the most immediate problem (slow endpoint, frequent deployments, tightest coupling to a database table). This generates organisational momentum and proves the model.
  - Implementation: Measure the top 5 slowest API endpoints, the top 5 most frequently changed modules, and the top 5 tables with the most locks/contention. Pick the one that scores highest across all three.
  - **Production story:** A retail company extracted their product search module first because it was causing 30% of CPU usage on the monolith. After extraction, the monolith's CPU dropped to 40% and search deployment frequency went from weekly to hourly.

- **Use a gateway/routing layer to decouple clients from the migration**
  - A proxy or gateway (e.g., Spring Cloud Gateway, Kong) allows you to route traffic to the monolith or the new service based on URL, header, or feature flag, without changing client code.
  - Implementation: Prefix all monolith routes with `/v1/` and new service routes with `/v1/migration/<service>`. The gateway checks a feature flag database to decide where to send each request. When the migration is complete, the gateway stops routing to the monolith for that prefix.

- **Invest in data reconciliation jobs**
  - During migration, data will fall out of sync between the monolith and new services. Reconciliation jobs that run periodically (hourly, daily) detect and fix inconsistencies.
  - Implementation: A reconciliation job reads from both the monolith database and the new service database, compares records by business key, and publishes discrepancies to a review queue. Automated fixes are applied for simple cases (missing records); complex conflicts require human review.

- **Never let feature flags accumulate beyond 90 days**
  - Feature flags have a half-life; if a flag is still active after 3 months, it either needs to be fully enabled and cleaned up, or the migration is stalled.
  - Implementation: Add a `remove-after` metadata field to every feature flag. CI enforces that flags with `remove-after` dates in the past cause a build warning. Schedule regular (monthly) flag cleanup tickets in the team's backlog.
