# REST vs Messaging (Sync vs Async)

---

## Overview

- **Definition:** REST (synchronous HTTP request-response) and messaging (asynchronous event-driven broker) are the two fundamental communication patterns in distributed systems.
- **Why It Exists:** Choosing between them determines coupling, resilience, scalability, and complexity. REST is simple and direct; messaging provides temporal decoupling, load leveling, and reliable delivery.
- **Key Concepts:** **Temporal Coupling** (both parties must be online for REST), **Event-Driven** (producer publishes, broker stores, consumer processes asynchronously), **At-Least-Once/Exactly-Once** delivery guarantees, **DLQ** (dead-letter queue for failed messages), **Consumer Groups** (scalable parallel processing).

---

## Core Concepts

- **REST (Request-Response):** Client sends request → server processes → returns response. Protocols: HTTP/HTTPS. Best for immediate responses, CRUD operations, and simple workflows. Temporal coupling means both parties must be available.
- **Messaging (Event-Driven):** Producer publishes event → broker stores → consumer processes. Protocols: AMQP, Kafka, JMS. Best for decoupling subsystems, load leveling, broadcasting, and long-running workflows. Producer and consumer need not be online simultaneously.
- **Hybrid Approach:** Use REST for commands (synchronous validation + persistence), then publish async events for downstream processing. CQRS pattern: commands via messaging, queries via REST.

```java
// Hybrid: REST for validation + persistence, then async event
@Transactional
public OrderResponse createOrder(CreateOrderRequest request) {
    validateInventory(request.getItems());
    Order order = orderRepository.save(Order.from(request));
    eventPublisher.orderCreated(order); // Kafka event
    return OrderResponse.from(order, "Order submitted for processing");
}
```

---

## Common Mistakes

- **Using REST for Long-Running Operations** — Causes timeouts and thread exhaustion. Return 202 Accepted with polling/callback instead. This *looks correct* because the operation completes in 2 seconds during testing — the timeout only becomes a problem under load when queueing delays push completion beyond the HTTP timeout.
- **Using Messaging for Queries** — Adds unnecessary complexity. Use REST for queries, events for commands. This *looks correct* because messaging seems like "the modern approach" for all communication — the added latency and complexity of request-response messaging only becomes frustrating when debugging a simple "get me the user's name" flow.
- **Ignoring Idempotency** — Leads to duplicate processing. Use idempotency keys in messages. This *looks correct* because the message arrives exactly once during development — duplicates only appear in production due to consumer crashes, network retries, and broker failovers that are rare in test environments.
- **Tight Coupling on Event Schemas** — Creates brittle consumers. Use schema registry + versioning. This *looks correct* because the producer and consumer are deployed together initially — the coupling pain only emerges months later when one team needs to evolve the schema without coordinating with 10 downstream consumers.
- **No Dead-Letter Queue** — Lost messages become silent failures. Configure DLQ for all consumers. This *looks correct* because exceptions during processing are caught and logged — the developer assumes the error is visible in logs, not realizing the failed message is simply gone and nobody will look for it.

---

## Key Design Considerations

- **Default to REST** for simple request-response. Use messaging when you need resilience, broadcasting, load leveling, or temporal decoupling.
- **Start with a single approach** and migrate to hybrid as complexity demands.
- **Version event schemas** independently of API versions.
- **Monitor both sync latency (p99)** and async consumer lag.
- **SAGA Pattern:** Choreography (messaging with events) vs Orchestration (REST with a coordinator). Choreography is more decoupled; orchestration is easier to manage.
- **Error Handling:** REST — client retries. Messaging — broker retries with DLQ.

---

## Real-World Scenarios

### Scenario 1: Order Processing — The Wrong Way and The Right Way
**Context:** A startup initially builds order processing with REST: `POST /api/orders` synchronously calls Inventory, Payment, Shipping, and Email services. Each call adds 500ms latency — total 2 seconds for the user. If any service is down, the order fails.

**Resolution (Wrong):** Users wait 2 seconds for order confirmation. Payment failures cause abandoned carts. Inventory service outages block order placement entirely.

**Resolution (Right):** `POST /api/orders` validates the request (synchronous, 50ms), saves the order to the database, publishes an `OrderPlaced` event to a message queue, and returns 202 Accepted. Downstream services consume the event asynchronously. The user sees "Order received!" immediately. Backend processing happens reliably via the queue.

```java
// REST endpoint — fast synchronous validation
@PostMapping("/api/orders")
public ResponseEntity<OrderResponse> placeOrder(@Valid @RequestBody CreateOrderRequest request) {
    Order order = orderService.createPending(request);  // 50ms: validate + persist
    eventPublisher.orderPlaced(order);                   // 5ms: publish to queue
    return ResponseEntity.accepted()
        .body(new OrderResponse(order.getId(), "Order received! You'll get a confirmation shortly."));
}

// Async consumer — reliable processing
@Component
public class OrderProcessingConsumer {
    @KafkaListener(topics = "order.events", groupId = "order-processor")
    public void process(OrderPlacedEvent event) {
        inventoryService.reserve(event.items());   // 200ms
        paymentService.charge(event.total());       // 500ms
        shippingService.createShipment(event);      // 100ms
        // Total: 800ms, but user already got response
    }
}
```

### Scenario 2: User Registration with Hybrid Pattern
**Context:** A user registration flow needs to validate email uniqueness (sync), create the user (sync), send a welcome email (async), notify CRM (async), and create defaults (async).

**Resolution:** Use the hybrid pattern. `POST /api/users` synchronously validates and creates the user, then publishes a `UserCreated` event. Email, CRM, and defaults are async consumers. The user gets an immediate response while heavy lifting happens in the background.

### Scenario 3: Query Performance Dashboard
**Context:** A real-time dashboard needs to display orders per second, revenue, and error rates. Using REST to query each service for every dashboard refresh (every 5 seconds) creates massive load.

**Resolution:** Use event-driven CQRS. Each service publishes events for every state change (order placed, payment completed, error occurred). A dashboard projection service consumes all events and maintains materialized views of current metrics. The dashboard queries these pre-computed views via REST with sub-10ms response times.

---

## Scenario-Based Questions

1. **Q: You are building a food delivery app. A customer places an order. You must: validate the order, charge the payment, notify the restaurant, assign a driver, and track the delivery. Which parts are synchronous (REST) and which are asynchronous (messaging)?**
    - A: Validate the order and charge payment synchronously — the customer needs immediate confirmation that the order is valid and funds are captured. Notify restaurant, assign driver, and track delivery are async — they happen in the background. Publish an `OrderPlaced` event after validation+payment. Restaurant notification, driver assignment, and tracking consumers process independently. The customer gets a "Order confirmed!" response in <1 second while backend processing happens asynchronously.
    > **Interview follow-up:** The payment gateway responds in 200ms during testing but can take up to 5 seconds during peak hours. If the synchronous payment call blocks the HTTP response, all API threads are consumed waiting for payment — how long would you wait synchronously before offloading to async processing with a polling status endpoint?

2. **Q: Your inventory check takes 500ms (looking up stock across warehouses). If you do this synchronously in the order API, response time is 500ms. If you move it to async, the customer might order items that are actually out of stock. How do you balance this?**
   - A: Use a two-phase approach. Phase 1 (sync, 50ms): check a Redis cache of current stock for a quick yes/no. Phase 2 (async): reserve the inventory via messaging. If phase 2 fails (actually out of stock), the customer gets a notification and can choose a substitute or refund. This keeps the API fast while maintaining accuracy. The risk window is the cache refresh delay (typically 1-5 seconds).

3. **Q: Your startup has grown and the monolith's REST API is slow because every request makes 5-10 synchronous calls to internal services. Users wait seconds for simple operations. How do you migrate to async without a complete rewrite?**
    - A: The strangler fig pattern for communication. Identify the slowest synchronous call chain (e.g., order → inventory + payment + shipping). Replace it with a hybrid approach: the REST API validates and persists synchronously, then publishes an event. Keep other synchronous calls intact initially. Gradually move each downstream integration to async event consumers. Each step improves latency independently.
    > **Interview follow-up:** During the migration, some requests go through the old sync path and some through the new async path. If a customer's order takes the async path but the inventory reservation fails, the customer already received a "success" response. How do you handle the UX of "success response followed by failure notification" during the transition period?

4. **Q: Your system uses messaging for everything, including simple queries. Developers complain that debugging is hard because you can't trace a request's flow. The product manager wants to see immediate search results, not "search results will arrive via message." What went wrong?**
   - A: Using messaging for queries is the wrong choice. Queries need immediate responses — use REST for reads, messaging for writes. CQRS is the right pattern: commands (writes) via messaging for decoupling, queries (reads) via REST/gRPC for immediate responses. Mixing them adds unnecessary complexity. The principle: if the client needs an answer now, use REST; if processing can happen later, use messaging.

5. **Q: Your payment service needs to return a URL for 3D Secure authentication. This URL is needed immediately for the user's browser redirect. But payment processing is done via messaging. How do you handle this synchronous handoff?**
   - A: Hybrid approach. The synchronous REST call initiates the payment and returns the 3DS URL immediately — this part is inherently synchronous because the user's browser needs the redirect URL. The actual payment confirmation (after 3DS completes) happens via a webhook callback. The webhook handler publishes a `PaymentCompleted` event for downstream processing. The synchronous part is minimized to just what the user's browser needs.

6. **Q: Your messaging consumer processes payments. A network partition isolates the consumer from the broker for 5 minutes. When connectivity restores, 10,000 pending messages flood the consumer. The payment gateway rate-limits to 100 requests/second. How do you handle this?**
   - A: Implement backpressure in the consumer. Use a rate limiter (Token Bucket) in the consumer that limits calls to the payment gateway to 100/second. The consumer reads messages but queues them internally before calling the gateway. Monitor the internal queue depth and adjust consumer prefetch. For extreme cases, implement circuit breaker on the consumer — if the local queue exceeds a threshold, pause consumption until it drains.

7. **Q: Your team is debating whether to use REST or messaging for a new notification service. The notification must reach the user in <100ms. What do you choose?**
   - A: For <100ms delivery, use messaging for resilience but WebSocket/SSE for the actual push. The flow: REST API receives notification request → publishes to a message queue → consumer processes and pushes via WebSocket to the user. The queue provides reliability (the notification is persisted) and buffering (if the WebSocket server is busy). The end-to-end latency is dominated by the WebSocket push (~50ms), not the queue (<5ms).

8. **Q: Your system has 15 microservices communicating via REST. A single request traces through 5 services — if any is slow, the entire request is slow. P99 latency is 5 seconds. How do you reduce this?**
   - A: Analyze the call chain and identify which calls need synchronous responses and which can be async. Often, 3 of the 5 calls are fire-and-forget actions (logging, analytics, notifications) that can be moved to async messaging. The remaining 2 synchronous calls might be optimizable with caching, circuit breakers, or parallel execution. Aim to reduce the synchronous chain to the minimum needed for the response.

9. **Q: You're designing a payment system that must report to the user "Payment successful" before returning the response. Does this require synchronous REST, or can you do it with messaging?**
   - A: If the user needs confirmation in the HTTP response, use REST synchronously for that specific operation. The payment controller calls the payment gateway directly (with proper timeouts and retries), gets the result, and returns the response. Messaging is inappropriate here because the user is waiting. After the response, you can still publish events for downstream processing (receipt email, accounting).

10. **Q: Your CTO says "All inter-service communication should be async via Kafka." Your team protests that some operations need immediate responses. How do you resolve this debate?**
    - A: Distinguish between commands and queries. Commands (writes): use async messaging because they don't need immediate responses after the initial validation. Queries (reads): use synchronous REST/gRPC because they need immediate answers. The CTO is right for commands but wrong for queries. The hybrid approach (CQRS) gives the best of both — the async flow for writes ensures resilience and decoupling; the sync flow for reads provides fast, reliable queries.

---

## Interview Questions

1. **What is the main difference between REST and messaging?**
   - A: REST is synchronous — the client sends a request and waits for a response. Messaging is asynchronous — the producer publishes a message and continues immediately; the consumer processes later. REST has temporal coupling (both parties must be online); messaging decouples sender and receiver in time and space.

2. **When would you choose REST over messaging?**
   - A: When immediate response is required (queries, user-facing actions), for CRUD operations, or when the client needs confirmation before proceeding. Examples: querying order status, user registration validation, login authentication.

3. **When would you choose messaging over REST?**
   - A: When decoupling subsystems, buffering traffic spikes, broadcasting events to multiple consumers, handling long-running workflows, or providing reliable delivery. Examples: order placed → notify inventory + payment + shipping + email.

4. **What is a dead-letter queue?**
   - A: A DLQ stores messages that failed processing after all retry attempts. Prevents poison messages from blocking the main queue. Allows manual inspection, bug fixing, and replay of failed messages.

5. **How do you handle idempotency in messaging?**
   - A: Include a unique idempotency key (message ID or correlation ID) in each message. The consumer checks a dedup store (Redis with TTL, DB unique constraint) before processing. If the ID was already processed, skip the message.

6. **What is the difference between at-least-once and exactly-once delivery?**
   - A: At-least-once guarantees delivery but may duplicate — requires idempotent consumers. Exactly-once avoids duplicates but requires transactional brokers and idempotent sinks — higher complexity and cost.

7. **How do you handle long-running operations in REST?**
   - A: Return 202 Accepted with a `Location` header pointing to a status endpoint that the client polls. Or provide a callback/webhook URL that the server calls when the operation completes. The client doesn't wait synchronously for long operations.

8. **What is the hybrid REST + messaging pattern?**
   - A: Use REST for synchronous validation and persistence (fast, immediate response). Then publish an async event for downstream processing. The API returns quickly while background consumers handle side effects. Best of both worlds.

9. **How do you version event schemas?**
   - A: Use a schema registry (Avro, Protobuf) with compatibility checks. Include a schema version in the event envelope. New fields are optional with defaults; never remove fields. Backward compatibility allows new consumers to read old events.

10. **What is CQRS and how does it relate to REST vs messaging?**
    - A: CQRS separates commands (writes) from queries (reads). Commands are typically sent via messaging for decoupling and reliability. Queries use REST for immediate responses. This leverages the strengths of both patterns.

---

## Developer Recommendations

- **Default to REST for queries, messaging for commands** — The simplest decision framework: if the client needs data (read), use REST for an immediate synchronous response. If the client is changing state (write), use messaging for decoupling and reliability. This follows CQRS principles and gives clear guidance to developers. The gray area is when a command needs to return data (e.g., create user and return ID) — use REST for the command with a synchronous response, then async for side effects. A startup built their entire platform on event-driven messaging, including user profile queries — every "get profile" request required publishing an event, waiting for a consumer to process it, and polling for the result. The 500ms query latency made the app feel sluggish, and the engineering team spent a quarter migrating to REST for reads.

- **Use the hybrid pattern for writes that need validation** — Pure async commands can't return validation errors to the caller. The hybrid approach solves this: synchronously validate (REST), then publish the async event. The user sees validation errors immediately and doesn't wait for downstream processing. This pattern works for order placement, user registration, payment initiation — any flow where validation must be immediate but processing can be deferred.

- **Never use messaging for queries that need immediate responses** — Request-response with messaging (correlation IDs, reply queues) is possible but adds latency, complexity, and coupling that negates messaging's benefits. If you need a response now, use REST/gRPC. Reserve request-response messaging for cases where the response can take seconds (saga coordination, long-running operations) and the client isn't waiting on the HTTP connection.

- **Design for eventual consistency when using messaging** — Messaging means the consumer may process the message seconds, minutes, or (in failure cases) hours after publication. Your system must handle this: stale data in reads, duplicate processing, and out-of-order delivery. Use TTLs on cached data, idempotent consumers, and version numbers on entities to detect conflicts. Accept that consistency is eventual, not immediate.

- **Monitor both synchronous latency and async consumer lag** — Two dimensions of system health: REST latency (P50/P95/P99) tells you if synchronous APIs are healthy. Consumer lag tells you if async processing is keeping up. A fast REST API with growing consumer lag means users get quick responses but backend processing is falling behind, and they'll see stale data until the lag is cleared.

- **Start simple (REST-only) and add messaging as complexity demands** — A new system should start with REST for simplicity. Add messaging when you hit specific pain points: a service needs to communicate with multiple others (broadcast), traffic spikes overwhelm synchronous processing (buffering), or a consumer needs reliable delivery. Premature messaging adds complexity without benefit.
