# SAGA Pattern

---

## Overview

- **Definition:** A design pattern for managing distributed transactions across multiple services by breaking the transaction into a sequence of local transactions, each with a compensating action that undoes its effects on failure.
- **Why It Exists:** Distributed transactions (2PC/XA) don't scale well in microservices because each service has its own database. Sagas provide eventual consistency without holding locks across services, enabling higher scalability and availability.
- **Key Concepts:** **Local Transaction** (atomic step within a single service), **Compensating Transaction** (semantically undoes a previous step), **Orchestration** (central coordinator controls the flow), **Choreography** (services react to events without coordinator), **Saga State** (persisted status of saga progress), **Idempotency Key** (unique identifier for safe retries)

---

## Core Concepts

- **Saga Structure:** Each step consists of a forward transaction Tx(i) and a compensating transaction CC(i). On success: Tx1 → Tx2 → Tx3 → Complete. On failure at Tx2: CC1 (reverse order) → Compensated. Compensations must semantically undo the forward action.
- **Choreography-based Saga:** Services communicate via events. Each service performs its local transaction and publishes an event that triggers the next step. On failure, services publish failure events and execute compensating actions. Best for simple linear flows with 2-3 steps.
- **Orchestration-based Saga:** A central orchestrator tells each service what to do, manages saga state, sequential execution, and compensation. Best for complex workflows with 3+ steps where centralized visibility and control are needed.
- **State Machine:** Saga progresses through states: STARTED → STEP_1_TX → STEP_1_OK → STEP_2_TX → ... → COMPLETED. On failure: COMPENSATING → individual compensations → COMPENSATED. State must be persisted to survive crashes.

```java
// Saga Orchestrator — central coordinator
@Component
public class OrderSagaOrchestrator {
    @Transactional
    public void startSaga(CreateOrderCommand command) {
        SagaState state = SagaState.builder()
            .sagaId(UUID.randomUUID().toString())
            .currentStep(SagaStep.RESERVE_INVENTORY)
            .status(SagaStatus.STARTED).build();
        sagaStateRepository.save(state);
        kafkaTemplate.send("inventory.commands", new ReserveInventoryCommand(state.getSagaId(), command.getItems()));
    }

    @KafkaListener(topics = "saga.responses", groupId = "saga-orchestrator")
    public void handleResponse(SagaResponse response) {
        SagaState state = sagaStateRepository.findById(response.getSagaId()).orElseThrow();
        if (response.isSuccess()) handleSuccess(state, response);
        else handleFailure(state, response);
    }

    private void compensate(SagaState state) {
        List<SagaStep> completed = getCompletedSteps(state);
        for (int i = completed.size() - 1; i >= 0; i--) {
            switch (completed.get(i)) {
                case CONFIRM_ORDER -> cancelOrder(state);
                case PROCESS_PAYMENT -> refundPayment(state);
                case RESERVE_INVENTORY -> releaseInventory(state);
            }
        }
        state.setStatus(SagaStatus.COMPENSATED);
        sagaStateRepository.save(state);
    }
}

// Idempotent command handling
@Component
public class IdempotentCommandHandler {
    @KafkaListener(topics = "inventory.commands")
    public void handleReserveInventory(ReserveInventoryCommand command) {
        String key = "saga:cmd:" + command.getCommandId();
        if (Boolean.FALSE.equals(redisTemplate.opsForValue().setIfAbsent(key, "PROCESSED", Duration.ofHours(24)))) {
            log.warn("Duplicate command: {}", command.getCommandId());
            return;
        }
        inventoryService.reserve(command.getOrderId(), command.getItems());
        sendSuccess(command);
    }
}
```

---

## Common Mistakes

- **Non-Idempotent Compensations** — running a compensation twice should be safe. Without idempotency, double-compensation causes data corruption.
- **Compensations That Fail** — compensations can also fail. Implement retry logic with exponential backoff and manual intervention when retries are exhausted.
- **Long-Running Sagas Without Timeout** — each step needs a timeout. Without timeouts, a saga can hang indefinitely waiting for a non-responsive service.
- **Mixing Orchestration and Choreography** — choosing one pattern per saga prevents confusion about who owns the saga logic.
- **Saga State Lost on Crash** — persist saga state to durable storage (database), not in-memory. Crashes should not lose saga progress.

---

## Key Design Considerations

- **Idempotency** — every saga command carries a unique idempotency key. Services check if they've already processed the key before executing. This enables safe retry of both forward and compensating transactions.
- **State Persistence** — saga state must be persisted to a database with optimistic concurrency control. On recovery, reconstruct saga from persisted state and continue or compensate. Use saga step logs for audit trails.
- **Timeout Handling** — each step should have a configurable deadline. If a step doesn't complete within the timeout, trigger compensation for all completed steps. Timeouts prevent sagas from hanging indefinitely.
- **Compensation Management** — compensations should have retry logic (3 attempts, exponential backoff). Failed compensations require manual intervention and operations alerts. Two-phase compensation (mark pending → execute → mark complete) prevents data loss on crash.
- **Orchestration vs Choreography Decision** — Orchestration for complex workflows (3+ steps), high visibility needs, and when centralized state management is required. Choreography for simple linear flows, loose coupling, and when services can react autonomously to events.

---

## Real-World Scenarios

### Scenario 1: E-Commerce Order SAGA
**Context:** An e-commerce order spans three services: Inventory (reserve items), Payment (charge customer), and Shipping (create shipment). Each has its own database. If payment fails after inventory is reserved, the reserved stock must be released or the system loses stock permanently.

**Resolution:** Implement an orchestrated saga. The Order Saga Orchestrator manages the workflow: (1) Send `ReserveInventory` command → Inventory service reserves stock, replies success. (2) Send `ProcessPayment` command → Payment service charges card, replies success. (3) Send `CreateShipment` command → Shipping service creates shipment, replies success → saga complete. If any step fails, the orchestrator executes compensating actions in reverse order: refund payment, release inventory, cancel order.

```java
@Component
public class OrderSagaOrchestrator {
    @Transactional
    public void start(PlaceOrderCommand cmd) {
        SagaState state = SagaState.create(UUID.randomUUID().toString(), cmd);
        sagaRepository.save(state);
        sendCommand(new ReserveInventory(state.sagaId(), cmd.items()));
    }

    public void onInventoryReserved(SagaResponse response) {
        SagaState state = sagaRepository.findById(response.sagaId()).orElseThrow();
        if (response.success()) {
            state.transitionTo(Step.PROCESS_PAYMENT);
            sagaRepository.save(state);
            sendCommand(new ProcessPayment(state.sagaId(), state.order().total()));
        } else {
            compensate(state);
        }
    }

    private void compensate(SagaState state) {
        for (SagaStep step : state.completedStepsInReverse()) {
            switch (step) {
                case RESERVE_INVENTORY -> sendCommand(new ReleaseInventory(state.sagaId(), state.order().items()));
                case PROCESS_PAYMENT -> sendCommand(new RefundPayment(state.sagaId(), state.order().total()));
            }
        }
        state.fail();
        sagaRepository.save(state);
    }
}
```

### Scenario 2: Travel Booking with Parallel Steps
**Context:** A travel booking system needs to book a flight, hotel, and car rental atomically for a vacation package. The flight and hotel bookings are independent (can happen in parallel), but the car rental depends on knowing the hotel location.

**Resolution:** Use a choreographed saga with parallel execution. The orchestrator sends `BookFlight` and `BookHotel` commands simultaneously. Both services process independently and reply. Once both succeed, the orchestrator sends `BookCar` with the hotel location. If any step fails, the orchestrator compensates the successful ones while the failed step needs no compensation.

### Scenario 3: Banking Transfer with Idempotency
**Context:** A banking system needs to transfer money between accounts in different services. Network retries cause duplicate transfer requests. Without idempotency, a single transfer could execute multiple times.

**Resolution:** Every saga command carries a unique `commandId`. Each service stores processed command IDs in a dedup table. If a duplicate command arrives (same `commandId`), the service returns the cached result instead of executing again. This enables safe retry of both forward actions and compensating actions.

---

## Scenario-Based Questions

1. **Q: You are building an order system where placing an order involves reserving inventory, processing payment, and creating a shipment. Each step has its own service and database. The payment is charged first, but if shipment creation fails, the customer has already been charged. How do you design this?**
   - A: Use a saga. The order should be: Reserve Inventory → Process Payment → Create Shipment. If shipment fails, trigger compensating actions in reverse: refund payment, release inventory. This ensures eventual consistency. Consider whether the order of operations matters — processing payment before shipment adds risk. Better: reserve inventory first (no financial impact), then create shipment, then process payment last. If payment fails, inventory is already released and shipment is cancelled — no financial harm.

2. **Q: Your orchestrated saga has 5 steps. After 3 successful steps, the 4th step fails, and the compensation for step 2 also fails. The system is now inconsistent. How do you recover?**
   - A: Implement a saga recovery process: (a) Retry the failed compensation with exponential backoff (3 attempts). (b) If retries exhausted, mark the saga as `COMPENSATION_FAILED` and alert operations. (c) Build an admin dashboard showing sagas in failed states with manual "retry compensation" and "force complete" buttons. (d) For critical cases, implement automated compensation retry with a background worker that retries every 5 minutes until success.

3. **Q: Your product manager says customers should be able to cancel an order within 30 minutes. The order has already triggered inventory reservation, payment, and shipping steps. How does the saga handle cancellation?**
   - A: The cancellation is itself a saga. Store the original saga's state. When the cancellation request arrives, check if the original saga is still within the cancel window. If so, execute the compensating saga: Release Inventory → Refund Payment → Cancel Shipment. Each compensation action must be idempotent. The cancellation saga has its own saga ID and state tracking, separate from the original order saga.

4. **Q: During a peak sales event, your saga orchestrator becomes a bottleneck and a single point of failure. How do you scale it?**
   - A: Make the orchestrator stateless — saga state is persisted in a database. Run multiple orchestrator instances behind a load balancer. Kafka topics partition by saga ID to ensure a single saga is always handled by the same instance (or use consistent hashing). The orchestrator's only stateful concern is the saga state DB, which must be highly available (replicated PostgreSQL or similar).

5. **Q: You're using a choreographed saga (event-driven) for a 3-step process. After a deployment, a bug causes step 2 to publish a failure event but step 2 actually succeeded. The saga compensates for a successful step. How do you prevent this?**
   - A: Use the outbox pattern — events are published only after the local transaction commits. If the transaction succeeded, the event is published. Additionally, implement a "confirmation window" between event publication and action. For critical sagas, add a read-back verification: step 2 publishes a `Pending` event first, and only publishes `Success` or `Failure` after verification. A timeout on the `Pending` state triggers investigation.

6. **Q: Your inventory service receives a "Release Inventory" compensation command. How do you ensure this is safe to call multiple times (e.g., during retries)?**
   - A: Idempotent compensation. Each inventory reservation creates a reservation record with a unique ID. The release operation checks if the reservation was already released before executing. Use `UPDATE inventory SET reserved = false WHERE reservation_id = ? AND reserved = true` — the WHERE clause makes it idempotent (second update affects zero rows). Store the compensation status in a dedup table with TTL.

7. **Q: Your saga has been running for 2 hours because a step is stuck waiting for a manual approval. The transaction timeout is 30 minutes. What happens?**
   - A: Implement saga timeouts. Each saga has a `timeout` and a `maxCompletionTime`. If the saga exceeds `maxCompletionTime` (e.g., 45 minutes for a 30-minute process), the orchestrator automatically initiates compensation, regardless of pending steps. For long-running approval workflows, model them differently — separate the approval from the saga, or use a waiting state that doesn't count toward the timeout.

8. **Q: You're building a bank transfer saga: Debit account A → Credit account B. The debit succeeds but the credit fails due to a bug. After fixing the bug, you replay the credit. How do you ensure the credit doesn't get applied twice?**
   - A: Idempotency keys on the credit operation. The saga sends a `CreditAccount` command with a unique `transferId`. The account service checks if it already processed `transferId`. If yes, return the existing result (success). If no, execute and mark as processed. This allows safe replay of any saga step. The debit side also needs idempotent replay in case the debit succeeded but the response was lost.

9. **Q: Your team is debating between orchestration and choreography for a 4-step saga. The steps are sequential with no parallelism. What factors drive the decision?**
   - A: Choose orchestration for: (a) complex workflows with 3+ steps, (b) need for centralized monitoring and management, (c) multiple teams owning different steps, (d) compliance requirements for audit trails. Choose choreography for: (a) simple linear flows with 2-3 steps, (b) maximum decoupling, (c) when each service can autonomously decide what to do next based on events. For 4+ sequential steps, orchestration is typically clearer.

10. **Q: A saga step modifies a resource, then the saga compensates. During the compensation window, another operation reads the modified resource. How do you prevent reads from seeing inconsistent intermediate states?**
    - A: Options: (a) Mark resources with `saga_id` and `saga_status` — reads filter out resources that are part of an in-progress saga. (b) Use a "pending" state for resources being modified — the UI shows a loading indicator instead of the inconsistent state. (c) In the read model, wait for the saga to complete before updating projections. (d) Accept the inconsistency for non-critical reads (eventual consistency).

---

## Interview Questions

1. **What is the Saga pattern?**
   - A: A design pattern for managing distributed transactions across multiple services. Each step is a local ACID transaction. If a step fails, compensating transactions undo the previously completed steps, achieving eventual consistency.

2. **Why do we need Sagas instead of distributed transactions (2PC)?**
   - A: Distributed transactions (2PC/XA) hold locks across services and databases, don't scale, reduce availability, and don't work well in microservices where each service has its own database. Sagas provide eventual consistency without global locks.

3. **What is the difference between orchestration and choreography in Sagas?**
   - A: Orchestration uses a central coordinator that commands each service and tracks state. Choreography uses events — services react to events published by other services, forming a chain. Orchestration is better for complex workflows; choreography is more decoupled.

4. **What is a compensating transaction?**
   - A: An action that semantically undoes a previous transaction. Not a rollback — a new transaction that reverses the effects. Example: if a payment was charged, the compensation is a refund. Compensations must be idempotent.

5. **What happens if a compensating transaction fails?**
   - A: Retry with exponential backoff (3-5 attempts). If all retries fail, the saga enters a `COMPENSATION_FAILED` state requiring manual intervention. Alert operations and provide an admin tool to retry or force-resolve.

6. **Is a Saga an ACID transaction?**
   - A: No. A saga provides eventual consistency, not ACID guarantees. Each individual step is a local ACID transaction. The saga as a whole is not atomic, isolated, or durable across steps — it's eventually consistent.

7. **How do you ensure saga state survives a crash?**
   - A: Persist saga state in a database. Use optimistic concurrency control for state updates (version field). On recovery, read all sagas in non-terminal states and either continue or compensate based on the current step.

8. **How do you handle timeouts in Sagas?**
   - A: Each saga step has a configurable timeout. If a step doesn't complete within the timeout, the orchestrator triggers compensation for all completed steps. Implement using scheduled timeout checks or TTL-based approaches.

9. **Can a Saga have parallel steps?**
   - A: Yes. Independent steps can execute in parallel (e.g., book hotel + book flight simultaneously). The orchestrator waits for all parallel steps to complete before proceeding. If any parallel step fails, compensate all completed steps.

10. **What is the role of idempotency in Sagas?**
    - A: Idempotency allows safe retry of both forward commands and compensating actions. Every command carries a unique ID. Services check if they've already processed an ID before executing. This prevents duplicate charges, duplicate inventory releases, and enables reliable recovery from network failures.

---

## Developer Recommendations

- **Prefer orchestration over choreography for complex workflows** — Orchestration with a central coordinator makes saga flows explicit, testable, and observable. You can see the entire saga state in one place, handle timeouts centrally, and implement recovery logic consistently. Choreography is more decoupled but makes the workflow implicit — you need to trace events across multiple services to understand the full flow. Rule of thumb: choreography for ≤3 simple steps; orchestration for 4+ steps or any steps with conditional branching.

- **Make every saga command and compensation idempotent** — Network failures, consumer crashes, and timeout retries mean duplicate commands are inevitable. Every command handler must check a dedup store before executing. Compensations must be safe to call multiple times. The cost of idempotency (a dedup table lookup) is negligible compared to the cost of inconsistent data.

- **Persist saga state in a database, not in memory** — In-memory saga state is lost on crash, leaving the system in an inconsistent state with no recovery path. Use a database (PostgreSQL, DynamoDB) with optimistic concurrency control. The saga state record should include: saga ID, current step, completed steps, saga status, and command payloads. Query non-terminal sagas on startup and resume or compensate them.

- **Implement saga timeouts with monitoring and alerting** — A saga that hangs indefinitely holds resources (reserved inventory, pending payments). Set a total saga timeout (e.g., 5 minutes for orders, 24 hours for travel bookings). If the timeout expires, automatically compensate. Monitor sagas approaching their timeout and alert when steps are slow.

- **Test saga failure scenarios with chaos engineering** — Don't just test the happy path. Test: step failure (what happens when payment fails?), compensation failure (what if refund fails?), saga timeout (does compensation trigger?), crash recovery (after orchestrator restart, does the saga resume?), duplicate commands (are they idempotent?). Use fault injection in staging and run chaos experiments in production during low traffic.
