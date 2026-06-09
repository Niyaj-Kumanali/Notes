# Interview-Mastery — Learning Roadmap

**Goal:** Transform existing notes into deep engineering references that explain *What*, *Why*, *How*, *When*, and *Why Not*.

**How to use this roadmap:**
- Files are ordered by **dependency** — complete each file before moving to the next in the phase.
- Within each file, add the supplementary sections (Why does it exist?, Real-World Analogy, How does it work internally?, etc.) **below** the existing content.
- Do **not** remove, replace, or shorten existing content.
- Mark files as `[x]` when done.

---

## Phase 1: Core Foundations (5 files)

These concepts underpin every other topic. Master them first.

- [ ] `Core-Concepts/OOPs.md`
- [ ] `Backend-Concepts/Architecture/SOLID.md`
- [ ] `Core-Concepts/Coupling-and-Cohesion.md`
- [ ] `Backend-Concepts/Architecture/Design-Patterns.md`
- [ ] `Core-Concepts/DSA.md`

---

## Phase 2: Language Deep-Dive — Java Core (6 files)

Build Java fundamentals before touching frameworks.

- [ ] `Java/Core/Collections-Framework.md`
- [ ] `Java/Core/Lambda-Expressions.md`
- [ ] `Java/Core/Stream-API.md`
- [ ] `Java/Core/Functional-Interfaces.md`
- [ ] `Java/Core/IO-Streams.md`
- [ ] `Java/Concurrency/Multithreading.md`
- [ ] `Java/Concurrency/Concurrency.md`

---

## Phase 3: Java Testing (3 files)

Learn to verify code before layering on frameworks.

- [ ] `Java/Testing/JUnit.md`
- [ ] `Java/Testing/Mockito.md`
- [ ] `Java/Testing/Testcontainers.md`

---

## Phase 4: Spring Boot (14 files)

Framework depth in logical dependency order.

- [ ] `Java/Spring-Boot/Dependency-Injection.md`
- [ ] `Java/Spring-Boot/Spring-Core.md`
- [ ] `Java/Spring-Boot/Bean-Lifecycle.md`
- [ ] `Java/Spring-Boot/Spring-MVC.md`
- [ ] `Java/Spring-Boot/Spring-Data-JPA.md`
- [ ] `Java/Spring-Boot/Transactions.md`
- [ ] `Java/Spring-Boot/Exception-Handling.md`
- [ ] `Java/Spring-Boot/Validation.md`
- [ ] `Java/Spring-Boot/Caching.md`
- [ ] `Java/Spring-Boot/Async-Processing.md`
- [ ] `Java/Spring-Boot/Scheduling.md`
- [ ] `Java/Spring-Boot/Spring-Security.md`
- [ ] `Java/Spring-Boot/Actuator.md`
- [ ] `Java/Spring-Boot/Performance-Optimization.md`

---

## Phase 5: Database (10 files)

The most common interview weak spot. Go deep here.

- [ ] `Database/SQL.md`
- [ ] `Database/Normalization.md`
- [ ] `Database/Denormalization.md`
- [ ] `Database/Indexing.md`
- [ ] `Database/Transactions.md`
- [ ] `Database/Isolation-Levels.md`
- [ ] `Database/Locking.md`
- [ ] `Database/Execution-Plans.md`
- [ ] `Database/Query-Optimization.md`
- [ ] `Database/Stored-Procedures.md`

---

## Phase 6: Backend — API Layer (5 files)

How systems expose and exchange data.

- [ ] `Backend-Concepts/API/REST-API.md`
- [ ] `Backend-Concepts/API/API-Security.md`
- [ ] `Backend-Concepts/API/API-Versioning.md`
- [ ] `Backend-Concepts/API/GraphQL.md`
- [ ] `Backend-Concepts/API/gRPC.md`

---

## Phase 7: Backend — Architecture & Design (8 files)

System design patterns and distributed architecture.

- [ ] `Backend-Concepts/Architecture/Microservices.md`
- [ ] `Backend-Concepts/Architecture/Event-Driven-Architecture.md`
- [ ] `Backend-Concepts/Architecture/Domain-Driven-Design.md`
- [ ] `Backend-Concepts/Architecture/CQRS.md`
- [ ] `Backend-Concepts/Architecture/SAGA.md`
- [ ] `Backend-Concepts/Architecture/Circuit-Breaker.md`
- [ ] `Backend-Concepts/Integration/REST-vs-Messaging.md`

---

## Phase 8: Backend — Communication & Caching (7 files)

Inter-service communication and performance optimisation.

- [ ] `Backend-Concepts/Communication/Message-Queues.md`
- [ ] `Backend-Concepts/Communication/Kafka.md`
- [ ] `Backend-Concepts/Communication/RabbitMQ.md`
- [ ] `Backend-Concepts/Communication/WebSockets.md`
- [ ] `Backend-Concepts/Caching/Caching-Strategies.md`
- [ ] `Backend-Concepts/Caching/Redis.md`
- [ ] `Backend-Concepts/Caching/Content-Delivery-Networks.md`

---

## Phase 9: Observability & Performance (4 files)

What matters in production.

- [ ] `Backend-Concepts/Observability/Logging.md`
- [ ] `Backend-Concepts/Observability/Metrics-Monitoring.md`
- [ ] `Backend-Concepts/Observability/Distributed-Tracing.md`
- [ ] `Backend-Concepts/Performance/Latency-vs-Throughput.md`
- [ ] `Backend-Concepts/Performance/Bottlenecks.md`

---

## Phase 10: Security (2 files)

Authentication, authorization, and token-based security.

- [ ] `Backend-Concepts/Security/OAuth2.md`
- [ ] `Backend-Concepts/Security/JWT.md`

---

## Phase 11: Cloud & DevOps (7 files)

Deployment, containerisation, and infrastructure.

- [ ] `Cloud-DevOps/Docker.md`
- [ ] `Cloud-DevOps/CI-CD.md`
- [ ] `Cloud-DevOps/GitHub-Actions.md`
- [ ] `Cloud-DevOps/Jenkins.md`
- [ ] `Cloud-DevOps/AWS.md`
- [ ] `Cloud-DevOps/Azure.md`
- [ ] `Cloud-DevOps/Azure-DevOps.md`

---

## Phase 12: Process & Additional Testing (6 files)

Methodology and cross-language testing.

- [ ] `Methodology/Agile.md`
- [ ] `Methodology/Scrum.md`
- [ ] `Testing/API-Testing.md`
- [ ] `Testing/Integration-Testing.md`
- [ ] `Testing/Jest.md`
- [ ] `Testing/Vitest.md`

---

## Phase 13 (Optional): C# Track

Complete only if targeting .NET roles.

- [ ] `CSharp/Core/Collections.md`
- [ ] `CSharp/Core/Lambda-Expressions.md`
- [ ] `CSharp/Core/LINQ.md`
- [ ] `CSharp/Core/IO-Streams.md`
- [ ] `CSharp/Concurrency/Multithreading.md`
- [ ] `CSharp/Concurrency/Async-Await.md`
- [ ] `CSharp/Frameworks/Entity-Framework.md`
- [ ] `CSharp/Frameworks/Dapper.md`

---

## Tips

- **One file at a time.** Open the file, read the existing content, then add the supplementary sections below it.
- **If a file is empty or missing**, create it with your own notes first, then expand it.
- **The C# track is parallel** — complete it after Phase 1 if you are a .NET developer, or skip it entirely for pure Java roles.
- **Revisit Phase 5 (Database)** before any system-design interview. It's the most common differentiator between junior and senior candidates.

---

*Created: 2026-06-09*
