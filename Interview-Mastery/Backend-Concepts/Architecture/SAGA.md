# SAGA Pattern

## 1. Executive Summary

The Saga pattern is a design pattern for managing distributed transactions across multiple services in a microservices architecture. Instead of using a monolithic distributed transaction (2PC/XA), a Saga breaks the transaction into a sequence of local transactions, each with a compensating action that undoes its effects if a subsequent step fails. Sagas ensure data consistency in distributed systems without the scalability and availability sacrifices of distributed transactions.

## 2. Core Theory

### Problem Statement
In a monolithic application, a single database transaction can span multiple operations (deduct inventory, charge payment, create shipment). In microservices, each service has its own database, making distributed transactions impractical.

### Saga Solution
A saga is a sequence of local transactions where each transaction updates data within a single service. If a step fails, the saga runs compensating transactions in reverse order to undo previous steps.

### Saga Types

**Choreography-based Saga:** Services communicate via events. Each service performs its local transaction and publishes an event that triggers the next step. On failure, services publish failure events and execute compensating actions.

**Orchestration-based Saga:** A central orchestrator tells each service what to do. The orchestrator manages the saga state, sequential execution, and compensation.

### Local Transaction and Compensation
Each saga step consists of:
- **Tx(i):** The forward transaction (e.g., reserve inventory).
- **CC(i):** The compensating transaction (e.g., release inventory), which semantically undoes Tx(i).

## 3. Under-the-Hood Deep Dive

### Saga Execution Coordinator (Orchestration)

```
[Orchestrator]
    |
    |-- Step 1: ReserveInventory --> [Inventory Service]
    |       |-- Success --> Step 2: ProcessPayment --> [Payment Service]
    |       |                              |-- Success --> Step 3: ConfirmOrder --> [Order Service]
    |       |                              |-- Failure --> Compensate 1: ReleaseInventory
    |       |-- Failure --> Saga fails, no compensation needed
    |
    Saga completes when all steps succeed.
    Saga compensates when any step fails (rollback in reverse order).
```

### Saga State Machine

```
                    +-----------+
                    | STARTED   |
                    +-----+-----+
                          |
                    +-----v-----+
                    | STEP_1_TX |  Reserve Inventory
                    +-----+-----+
                          |
              +-----------+-----------+
              |                       |
        +-----v-----+          +-----v-----+
        | STEP_1_OK  |          | STEP_1_CC  |  Release Inventory
        +-----+-----+          +-----+------+
              |                       |
        +-----v-----+                 |
        | STEP_2_TX |  Charge Payment  |
        +-----+-----+                 |
              |                       |
              |                  +----v----+
              |                  | FAILED  |
              |                  +---------+
        +-----v-----+
        | STEP_2_OK  |
        +-----+-----+
              |
        +-----v-----+
        | COMPLETED |
        +-----------+
```

## 4. Production Code Examples

### Choreography-based Saga Example

```java
// Order Service - publishes event when order created
@Service
public class OrderSagaParticipant {

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = new Order();
        order.setId(UUID.randomUUID().toString());
        order.setCustomerId(request.getCustomerId());
        order.setAmount(request.getAmount());
        order.setStatus(OrderStatus.PENDING);
        orderRepository.save(order);

        // Publish event to start saga
        OrderCreatedEvent event = OrderCreatedEvent.builder()
            .orderId(order.getId())
            .customerId(order.getCustomerId())
            .amount(order.getAmount())
            .build();

        kafkaTemplate.send("order.events", order.getId(), event);
        return order;
    }

    // Handle compensation - order cancelled
    @KafkaListener(topics = "saga.events", groupId = "order-service")
    public void handleSagaEvent(SagaEvent event) {
        if (event.getType().equals("ORDER_CANCELLED")) {
            orderRepository.findById(event.getOrderId()).ifPresent(order -> {
                order.setStatus(OrderStatus.CANCELLED);
                orderRepository.save(order);
            });
        }
    }
}

// Inventory Service - reserves on OrderCreated, compensates on failure
@Service
public class InventorySagaParticipant {

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @KafkaListener(topics = "order.events", groupId = "inventory-service")
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            // Reserve inventory
            inventoryService.reserve(event.getOrderId(), event.getItems());

            // Success - trigger next step
            InventoryReservedEvent reserved = InventoryReservedEvent.builder()
                .orderId(event.getOrderId())
                .build();
            kafkaTemplate.send("inventory.events", event.getOrderId(), reserved);
        } catch (Exception e) {
            // Failure - trigger compensation
            InventoryFailedEvent failed = InventoryFailedEvent.builder()
                .orderId(event.getOrderId())
                .reason(e.getMessage())
                .build();
            kafkaTemplate.send("inventory.events", event.getOrderId(), failed);
        }
    }

    @KafkaListener(topics = "compensation.events", groupId = "inventory-service")
    public void handleInventoryRelease(ReleaseInventoryEvent event) {
        inventoryService.release(event.getOrderId());
    }
}
```

### Orchestration-based Saga with State Machine

```java
// Saga Orchestrator
@Component
public class OrderSagaOrchestrator {

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    @Autowired
    private SagaStateRepository sagaStateRepository;

    @Transactional
    public void startSaga(CreateOrderCommand command) {
        SagaState state = SagaState.builder()
            .sagaId(UUID.randomUUID().toString())
            .orderId(UUID.randomUUID().toString())
            .currentStep(SagaStep.RESERVE_INVENTORY)
            .status(SagaStatus.STARTED)
            .build();

        sagaStateRepository.save(state);

        // Step 1: Reserve Inventory
        ReserveInventoryCommand cmd = ReserveInventoryCommand.builder()
            .sagaId(state.getSagaId())
            .orderId(state.getOrderId())
            .items(command.getItems())
            .build();

        kafkaTemplate.send("inventory.commands", cmd);
    }

    @KafkaListener(topics = "saga.responses", groupId = "saga-orchestrator")
    public void handleResponse(SagaResponse response) {
        SagaState state = sagaStateRepository.findById(response.getSagaId())
            .orElseThrow(() -> new SagaNotFoundException(response.getSagaId()));

        if (response.isSuccess()) {
            handleSuccess(state, response);
        } else {
            handleFailure(state, response);
        }
    }

    private void handleSuccess(SagaState state, SagaResponse response) {
        switch (state.getCurrentStep()) {
            case RESERVE_INVENTORY -> {
                // Move to payment step
                state.setCurrentStep(SagaStep.PROCESS_PAYMENT);
                sagaStateRepository.save(state);

                ProcessPaymentCommand cmd = ProcessPaymentCommand.builder()
                    .sagaId(state.getSagaId())
                    .orderId(state.getOrderId())
                    .amount(response.getAmount())
                    .build();
                kafkaTemplate.send("payment.commands", cmd);
            }
            case PROCESS_PAYMENT -> {
                // Move to confirmation
                state.setCurrentStep(SagaStep.CONFIRM_ORDER);
                sagaStateRepository.save(state);

                ConfirmOrderCommand cmd = ConfirmOrderCommand.builder()
                    .sagaId(state.getSagaId())
                    .orderId(state.getOrderId())
                    .build();
                kafkaTemplate.send("order.commands", cmd);
            }
            case CONFIRM_ORDER -> {
                // Saga complete
                state.setStatus(SagaStatus.COMPLETED);
                sagaStateRepository.save(state);
                log.info("Saga {} completed successfully", state.getSagaId());
            }
        }
    }

    private void handleFailure(SagaState state, SagaResponse response) {
        log.error("Saga {} failed at step {}: {}", state.getSagaId(),
            state.getCurrentStep(), response.getError());

        state.setStatus(SagaStatus.COMPENSATING);
        sagaStateRepository.save(state);

        // Execute compensations in reverse order
        compensate(state);
    }

    private void compensate(SagaState state) {
        // Reverse order of completed steps
        List<SagaStep> completedSteps = getCompletedSteps(state);

        for (int i = completedSteps.size() - 1; i >= 0; i--) {
            SagaStep step = completedSteps.get(i);
            switch (step) {
                case CONFIRM_ORDER -> cancelOrder(state);
                case PROCESS_PAYMENT -> refundPayment(state);
                case RESERVE_INVENTORY -> releaseInventory(state);
            }
        }

        state.setStatus(SagaStatus.COMPENSATED);
        sagaStateRepository.save(state);
    }
}
```

### Saga State Persistence

```java
@Entity
@Table(name = "saga_states")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class SagaState {
    @Id
    private String sagaId;
    private String orderId;

    @Enumerated(EnumType.STRING)
    private SagaStep currentStep;

    @Enumerated(EnumType.STRING)
    private SagaStatus status;

    @Version
    private Long version;

    @OneToMany(cascade = ALL)
    @JoinColumn(name = "saga_id")
    private List<SagaStepLog> stepLogs;

    @Column(columnDefinition = "TEXT")
    private String payload;
}

@Entity
@Table(name = "saga_step_logs")
@Data
@Builder
public class SagaStepLog {
    @Id
    @GeneratedValue(strategy = UUID)
    private String id;
    private String sagaId;

    @Enumerated(EnumType.STRING)
    private SagaStep step;

    @Enumerated(EnumType.STRING)
    private StepStatus status;

    private String responseData;
    private String errorMessage;
    private Instant executedAt;
}
```

### Idempotent Command Handling in Saga

```java
@Component
public class IdempotentCommandHandler {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    private static final Duration COMMAND_TTL = Duration.ofHours(24);

    public boolean isAlreadyProcessed(String commandId) {
        String key = "saga:cmd:" + commandId;
        Boolean absent = redisTemplate.opsForValue()
            .setIfAbsent(key, "PROCESSED", COMMAND_TTL);
        return Boolean.FALSE.equals(absent);
    }

    @KafkaListener(topics = "inventory.commands")
    public void handleReserveInventory(ReserveInventoryCommand command) {
        if (isAlreadyProcessed(command.getCommandId())) {
            log.warn("Duplicate command: {}", command.getCommandId());
            return;
        }

        // Process the command
        try {
            inventoryService.reserve(command.getOrderId(), command.getItems());
            sendSuccess(command);
        } catch (Exception e) {
            sendFailure(command, e);
        }
    }
}
```

## 5. Real-World Scenarios

### Order Processing Saga

```
Steps:
1. Order Service: Create order (PENDING)
2. Inventory Service: Reserve items
3. Payment Service: Charge customer
4. Order Service: Confirm order
5. Shipping Service: Create shipment

Compensations:
- Cancel shipment
- Refund payment
- Release inventory
- Cancel order
```

### Travel Booking Saga

```
Steps:
1. Flight Service: Book flight
2. Hotel Service: Book room
3. Car Rental Service: Book car
4. Payment Service: Charge total

Compensations:
- Cancel car booking
- Cancel hotel booking
- Cancel flight booking
- Refund payment
```

## 6. Performance

### Saga Latency Breakdown

```
Step 1: Reserve Inventory   - 50ms
Step 2: Process Payment     - 200ms (external gateway)
Step 3: Confirm Order       - 20ms
Step 4: Create Shipment     - 50ms
Total Saga Time: ~320ms (successful)
Total Compensation: Variable (reverse order)
```

### Optimization Strategies
- Parallelize independent steps (book flight + hotel simultaneously).
- Use async messaging instead of synchronous HTTP.
- Cache saga state in Redis for fast access.
- Timeout per step with configurable limits.

## 7. Security

### Idempotency Keys
Every saga command carries a unique idempotency key to prevent duplicate execution.

```java
public class SagaCommand {
    private String commandId; // UUID - unique per command attempt
    private String sagaId;
    // ... other fields
}
```

### Authorization in Saga
Each saga step must authenticate and authorize the action, even if triggered by an event.

### Audit Trail
All saga steps and compensations must be logged for auditability.

## 8. Common Mistakes

### Mistake 1: Non-Idempotent Compensations
Running compensation twice should be safe. Use idempotency keys.

### Mistake 2: Compensations That Fail
Compensations can also fail. Implement retry logic and manual intervention for failed compensations.

### Mistake 3: Long-Running Sagas Without Timeout
```java
// WRONG - saga can run forever
@Saga
public void processOrder() {
    reserveInventory();
    processPayment(); // Blocks indefinitely if payment gateway hangs
}

// RIGHT - timeout per step
@KafkaListener(topics = "saga.commands")
public void handleCommand(SagaCommand cmd) {
    // Timeout handled at orchestrator level
    orchestrator.processWithTimeout(cmd, Duration.ofSeconds(30));
}
```

### Mistake 4: Mixing Orchestration and Choreography
Choose one pattern per saga. Mixing leads to confusion about who owns the saga logic.

### Mistake 5: Saga State Lost on Crash
Persist saga state to durable storage (database), not just in memory.

## 9. Senior Engineer Perspective

### Saga Design Guidelines

1. **Each step should be idempotent**: Replaying a step should not cause issues.
2. **Compensations must succeed**: If a compensation fails, the system enters an inconsistent state.
3. **Saga state must be persistent**: Crashes should not lose saga progress.
4. **Timeout handling**: Each step should have a timeout. Timeouts trigger compensation.
5. **Manual intervention**: Some sagas may require human intervention to resolve.

### When to Use Orchestration vs Choreography

| Criteria | Orchestration | Choreography |
|----------|--------------|--------------|
| Number of steps | 3-5+ | 2-3 |
| Complexity | Complex | Simple |
| Centralized control | Needed | Not needed |
| Coupling | Tighter | Looser |
| Visibility | High | Low |
| When to choose | Complex business process | Simple linear flow |

### Handling Compensations at Scale

```java
@Component
public class CompensationManager {

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    private static final int MAX_RETRIES = 3;
    private static final Duration RETRY_DELAY = Duration.ofSeconds(30);

    public void compensate(SagaState state) {
        List<SagaStep> completedSteps = getCompletedSteps(state);

        for (int i = completedSteps.size() - 1; i >= 0; i--) {
            SagaStep step = completedSteps.get(i);
            boolean compensated = compensateWithRetry(state, step);

            if (!compensated) {
                // Log critical error - requires manual intervention
                log.error("Failed to compensate step {} for saga {}",
                    step, state.getSagaId());
                alertOperations(state, step);
            }
        }
    }

    private boolean compensateWithRetry(SagaState state, SagaStep step) {
        for (int attempt = 1; attempt <= MAX_RETRIES; attempt++) {
            try {
                sendCompensationCommand(state, step);
                return true;
            } catch (Exception e) {
                log.warn("Compensation attempt {} failed for saga {} step {}",
                    attempt, state.getSagaId(), step, e);
                if (attempt < MAX_RETRIES) {
                    Thread.sleep(RETRY_DELAY.toMillis());
                }
            }
        }
        return false;
    }
}
```

## 10. Interview Questions (20: 10 easy + 10 medium)

### Easy

1. **Q:** What is the Saga pattern?
   **A:** A distributed transaction pattern that coordinates multiple local transactions with compensating actions on failure.

2. **Q:** Why do we need Sagas in microservices?
   **A:** Because distributed transactions (2PC) don't scale well. Each microservice has its own database, making ACID transactions across services impractical.

3. **Q:** What is the difference between orchestration and choreography in Sagas?
   **A:** Orchestration: central coordinator directs steps. Choreography: services react to events without central control.

4. **Q:** What is a compensating transaction?
   **A:** An action that semantically undoes a previous transaction (e.g., release reserved inventory).

5. **Q:** Can a Saga span multiple hours?
   **A:** Yes, Sagas are designed for long-running transactions, potentially spanning hours or days.

6. **Q:** What happens if a compensating transaction fails?
   **A:** The system enters an inconsistent state requiring manual intervention or automated retry.

7. **Q:** Is a Saga an ACID transaction?
   **A:** No. Sagas provide eventual consistency, not ACID guarantees. Each step is a separate local ACID transaction.

8. **Q:** What is a step in a Saga?
   **A:** A local transaction within a single service, part of the overall saga workflow.

9. **Q:** How do you handle idempotency in Sagas?
   **A:** Each command has a unique idempotency key. Services check if they've already processed the key before executing.

10. **Q:** What is the difference between Saga and 2PC?
    **A:** 2PC is synchronous and holds locks. Saga is asynchronous and releases locks after each step.

### Medium

11. **Q:** How do you ensure a saga state survives a system crash?
    **A:** Persist saga state in a database. On recovery, reconstruct saga from persisted state and continue or compensate.

12. **Q:** How do you handle timeouts in Sagas?
    **A:** Each step has a deadline. If not completed within deadline, trigger compensation for previous steps.

13. **Q:** What is the difference between choreography and orchestration for failure handling?
    **A:** Choreography: each service publishes failure event, triggering compensation. Orchestration: central coordinator detects failure and sends compensation commands.

14. **Q:** How do you test Sagas?
    **A:** Unit test individual steps, integration test with messaging, test compensating flows, chaos testing for failure scenarios.

15. **Q:** Can a Saga have parallel steps?
    **A:** Yes. Independent steps can execute in parallel (e.g., book hotel and flight simultaneously).

16. **Q:** How do you monitor Saga progress?
    **A:** Track saga state (started, step 1/3, step 2/3, completed, compensating, compensated). Metric: saga success rate, duration.

17. **Q:** What is the role of the outbox pattern in Sagas?
    **A:** Ensures event publication is atomic with local database transaction. Prevents inconsistent state where DB is updated but event isn't published.

18. **Q:** How do you avoid cascading compensations?
    **A:** Design compensations to be idempotent and safe to run multiple times. Use circuit breakers to prevent compensating healthy services.

19. **Q:** What is a saga participant?
    **A:** A service that executes one or more saga steps and their compensations.

20. **Q:** How does Saga differ from event-driven architecture?
    **A:** Saga is a specific pattern for distributed transactions. EDA is a broader communication pattern. Sagas often use events for choreography.

## 11. Advanced Interview Questions (20: 10 hard + 10 system design)

### Hard

1. **Q:** Design a saga that handles partial failures where some compensations succeed and others fail.
    **A:** Track each compensation's status. If a compensation fails, log a critical error, trigger alert, and save to manual intervention queue. Continue compensating remaining steps. Periodic retry for failed compensations.

2. **Q:** How do you implement a saga that involves human approval steps (e.g., loan approval)?
    **A:** Saga pauses at approval step. Timer service sends timeout. Approval service consumes event, responds with approved/rejected. Saga resumes or compensates.

3. **Q:** Design a saga for money transfer between banks (slow, external systems).
    **A:** Steps: 1) Debit sender (internal), 2) Send wire to external bank (async, callback), 3) Credit receiver. Compensation: 1) Reverse debit, 2) Recall wire (may not be possible = manual intervention). Use idempotent commands with external confirmation.

4. **Q:** How do you handle deadlock in saga step execution?
    **A:** Deadlock is rare in sagas because each step releases locks after completion. Use lock ordering, timeouts, and retries. Detect via monitoring (steps not progressing).

5. **Q:** Design a saga monitoring system that provides real-time visibility into all active sagas.
    **A:** Saga state persisted to time-series DB. Dashboard: active sagas count, per-step duration, success/failure rate, stuck sagas. Alerts: sagas stuck > 5 min, compensation failures, high failure rate.

6. **Q:** How do you implement saga isolation levels?
    **A:** Sagas don't provide isolation like ACID. Mitigate with: 1) Semantic locks (reserve inventory), 2) Commutative updates (add/subtract rather than set), 3) Pending state (preview data), 4) Saga log queries for visibility.

7. **Q:** Design a saga that must maintain strict ordering across steps.
    **A:** Orchestration-based saga: central coordinator enforces ordering. Kafka partition by saga ID for ordering. Step output includes expected next step. Validate step sequence before execution.

8. **Q:** How do you handle cross-saga dependencies?
    **A:** Avoid if possible. If needed, use hierarchical sagas: parent saga coordinates child sagas. Each child saga is independent. Parent handles both child success and failure.

9. **Q:** Design a saga retry strategy for transient failures.
    **A:** Exponential backoff: 1s, 2s, 4s, 8s, 16s, 32s. Max retries: 5. After max: send to DLQ and trigger manual intervention. Classify failures: transient (retry), permanent (compensate immediately).

10. **Q:** How do you version sagas to support long-running instances across deployments?
    **A:** Saga version in state payload. Backward-compatible saga changes during rolling deployment. Old sagas continue with old version. New sagas use new version.

### System Design

11. **Q:** Design an order processing saga for an e-commerce platform.
    **A:** Orchestration-based. Steps: Create Order -> Reserve Inventory -> Charge Payment -> Confirm Order -> Schedule Delivery. Compensations: Cancel Delivery -> Refund -> Release Inventory -> Cancel Order. Kafka for commands/events. PostgreSQL for saga state.

12. **Q:** Design a travel booking saga (flight + hotel + car).
    **A:** Choreography-based. Independent steps can run in parallel (flight + hotel + car). Payment step after all bookings confirmed. Compensations: cancel each booking independently. External cancellations may have fees.

13. **Q:** Design a banking transfer saga between two banks.
    **A:** Orchestration-based. Step 1: Debit from source bank. Step 2: Send to external banking system (ISO 20022). Step 3: Poll for confirmation (timeout 24h). Step 4: Credit to destination bank. Compensations: reverse debit, cancel wire transfer.

14. **Q:** Design a user registration saga across multiple services.
    **A:** Steps: 1) Create user profile, 2) Send verification email, 3) Create default workspace, 4) Set up billing account, 5) Send welcome email. Compensations: delete billing, delete workspace, delete profile.

15. **Q:** Design a content publishing saga (draft -> review -> publish).
    **A:** Steps: 1) Save draft, 2) Submit for review, 3) Reviewer approves, 4) Schedule publish, 5) Publish. Compensations: unpublish, cancel schedule, revert to draft.

16. **Q:** Design a multiplayer game match-making saga.
    **A:** Steps: 1) Find players (wait for N), 2) Select map, 3) Assign teams, 4) Initialize game state, 5) Start game. Compensations: cancel match, release players back to queue.

17. **Q:** Design a subscription upgrade saga.
    **A:** Steps: 1) Validate new plan, 2) Calculate prorated charge, 3) Charge customer, 4) Update subscription, 5) Notify customer. Compensations: revert subscription, refund charge.

18. **Q:** Design a cloud resource provisioning saga.
    **A:** Steps: 1) Create VM, 2) Attach storage, 3) Configure networking, 4) Install software, 5) Add to load balancer. Compensations: remove from LB, delete software, delete network, detach storage, delete VM.

19. **Q:** Design a loan application saga.
    **A:** Steps: 1) Check credit score, 2) Verify income, 3) Appraise property, 4) Underwriting approval, 5) Disburse funds. Compensations: recall funds, reverse approval. Human-in-the-loop for steps 3-4.

20. **Q:** Design a multi-step notification saga (email -> SMS -> push).
    **A:** Steps run in parallel with fallback: 1a) Send email, 1b) if email fails, send SMS, 1c) if SMS fails, send push notification. No compensation (notifications are fire-and-forget).

## 12. Expert-Level Interview Questions (10: architect-level)

1. **Q:** Design a saga framework that supports both orchestration and choreography from a single definition.
    **A:** DSL-based saga definition (YAML/JSON). Compiler generates both orchestration and choreography implementations. Orchestration mode: saga coordinator service. Choreography mode: event schemas and handler skeletons.

2. **Q:** How do you implement a saga with nested compensating transactions (compensations that themselves fail)?
    **A:** Compensation failure handling is a nested saga. Each compensation is a local transaction with its own compensation. If release-inventory fails, the compensation for that is "mark inventory as released manually". Chain of compensation guarantees eventual consistency.

3. **Q:** Design a saga system that guarantees exactly-once execution across all participants.
    **A:** Each step: idempotent command processing + outbox pattern. Saga coordinator: write-ahead log with at-least-once delivery. Consumer: idempotent processing with dedup store. Exactly-once achieved by combining exactly-once delivery + idempotent processor.

4. **Q:** How do you implement a saga for a multi-region deployment?
    **A:** Saga state replicated across regions (CRDT or last-writer-wins). Coordinator runs in all regions (active-active) with leader election. Compensations must run in correct region (affinity to original data). Cross-region saga events via global event bus.

5. **Q:** Design a governance system for managing sagas across 100+ microservices.
    **A:** Central saga registry: all saga definitions registered with metadata (participants, steps, timeouts). Automated dependency graph. Saga monitoring dashboard. Approval required for saga modifications. Schema validation for saga events. Deprecation process for old sagas.

6. **Q:** How do you ensure data consistency when a saga participant fails before recording the compensation status?
    **A:** Two-phase compensation: Phase 1 - mark compensation as pending in saga state. Phase 2 - execute compensation and mark as complete. If crash between phases, recovery process finds pending compensations and retries. Idempotent compensations ensure safe retry.

7. **Q:** Design a saga that dynamically discovers participants based on runtime conditions.
    **A:** Saga step is an interface. Registry maps step names to implementations. Dynamic discovery via service registry (Consul/Etcd). Step implementation selected based on runtime context (A/B test, feature flag, tenant configuration).

8. **Q:** How do you implement a saga that handles competing actions (auction scenario)?
    **A:** Bid saga: Place Bid -> Hold Funds -> Notify Outbid (previous bidder). Competing bids create concurrent sagas for the same item. Use optimistic concurrency: version check on item. Only first bid wins; others fail with retry.

9. **Q:** Design a saga debugger that can step through saga execution in production.
    **A:** Saga state checkpointing at each step. Pause/resume saga capability via admin API. Replay single saga step for debugging without affecting data. Trace visualization: timeline of saga events across services. Correlation ID links all saga logs.

10. **Q:** How do you implement a saga with dynamic compensation based on the exact state at failure time?
    **A:** Store payload snapshots at each step. Compensation logic reads snapshot to determine exact reverse action. For example: if inventory reserved 5 units on step 2, but pricing discount applied on step 3, compensation for step 2 releases 5 units, compensation for step 3 reverses the discount.

## 13. Debugging & Troubleshooting

### Common Saga Issues

**Issue: Saga stuck in "compensating" state**
- Check if compensation service is down.
- Check DLQ for compensation failures.
- Check logs for compensation errors.
- Manually trigger compensation if automated retry exhausted.

**Issue: Duplicate compensation executed**
- Verify idempotency of compensation logic.
- Check dedup key TTL.
- Check if saga recovery doubled compensation.

**Issue: Event not delivered to saga participant**
- Check Kafka consumer lag.
- Check Kafka topic configuration.
- Verify consumer group is active.
- Check for serialization errors.

**Issue: Compensating transaction order wrong**
- Verify compensation order in saga definition.
- Check saga state tracking.

### Debugging Commands
```bash
# Check saga state in DB
SELECT * FROM saga_states WHERE saga_id = 'saga-123';
SELECT * FROM saga_step_logs WHERE saga_id = 'saga-123' ORDER BY executed_at;

# Check Kafka consumer lag for saga participants
kafka-consumer-groups --bootstrap-server kafka:9092 \
  --group saga-orchestrator --describe

# Check DLQ for failed saga steps
kafka-console-consumer --bootstrap-server kafka:9092 \
  --topic saga.dlq --from-beginning --max-messages 10

# Check saga metrics
curl http://saga-monitor/metrics | grep saga
```

## 14. Comparison Section

### Saga vs 2PC (XA Transactions)

| Aspect | Saga | 2PC |
|--------|------|-----|
| Consistency | Eventual | Strong |
| Lock Duration | Short (per step) | Long (entire transaction) |
| Scalability | High | Low |
| Availability | High (async) | Lower (sync) |
| Complexity | Medium | Low (but harder to operate) |
| Use Case | Long-running, multi-service | Short, same-DB transactions |

### Orchestration vs Choreography

| Aspect | Orchestration | Choreography |
|--------|--------------|--------------|
| Coordinator | Central service | None |
| Coupling | Tighter (services know coordinator) | Loose (services know events) |
| State Management | Centralized | Distributed |
| Visibility | Easy (single flow) | Harder (follow events) |
| Complexity | Higher in coordinator | Higher in each service |
| Failure Handling | Centralized | Distributed |
| Best For | Complex workflows (>3 steps) | Simple linear flows |

### Saga vs Event Sourcing

| Aspect | Saga | Event Sourcing |
|--------|------|---------------|
| Purpose | Distributed transactions | State persistence |
| Events | Command events | State change events |
| State | Current step + compensation | Full event history |
| Duration | Until complete | Forever |
| Replay | Recovery | Full state rebuild |

## 15. Revision Notes

### Quick Recap
- **Saga**: Sequence of local transactions + compensation for distributed consistency.
- **Orchestration**: Central coordinator controls the flow.
- **Choreography**: Services react to events without coordinator.
- **Compensation**: Semantically undoes a completed step.
- **Idempotency**: Required for safe retries in all steps and compensations.
- **State Persistence**: Saga state must survive crashes.
- **Timeout**: Each step should have a deadline.

### Key Design Rules
1. Compensations must be idempotent.
2. Persist saga state to durable storage.
3. Use idempotency keys for all commands.
4. Set timeouts for each saga step.
5. Monitor saga progress and alert on failures.
6. Test compensation flows as thoroughly as forward flows.

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|                      SAGA PATTERN CHEAT SHEET                      |
+-------------------------------------------------------------------+
| TYPE         | COORDINATION | COUPLING | VISIBILITY | BEST FOR     |
+--------------+--------------+----------+------------+--------------+
| Orchestration| Central      | Tighter  | High       | Complex      |
|              | coordinator  |          |            | workflows    |
| Choreography | Events       | Loose    | Lower      | Simple flows |
+--------------+--------------+----------+------------+--------------+
| SAGA STRUCTURE                                                     |
+-------------------------------------------------------------------+
| Step 1: Service A (Tx 1)   | Compensation: Service A (CC 1)       |
| Step 2: Service B (Tx 2)   | Compensation: Service B (CC 2)       |
| Step 3: Service C (Tx 3)   | Compensation: Service C (CC 3)       |
|                                                                |
| SUCCESS: Tx1 -> Tx2 -> Tx3 -> COMPLETE                             |
| FAILURE at Tx2: CC1 (reverse order) -> COMPENSATED                |
+-------------------------------------------------------------------+
| ORCHESTRATION FLOW                                                 |
+-------------------------------------------------------------------+
| [Orchestrator] -> Command -> [Service A]                           |
| [Orchestrator] <- Response <- [Service A]                          |
| [Orchestrator] -> Command -> [Service B]                           |
| [Orchestrator] <- Response <- [Service B] (failure!)               |
| [Orchestrator] -> Compensate -> [Service A] (rollback)             |
+-------------------------------------------------------------------+
| CHOREOGRAPHY FLOW                                                  |
+-------------------------------------------------------------------+
| [Service A] publishes Event_A                                      |
| [Service B] consumes Event_A, processes, publishes Event_B         |
| [Service C] consumes Event_B, processes, publishes Event_C         |
| If Service_B fails: publishes Event_B_Failed                       |
| [Service A] consumes Event_B_Failed, compensates                   |
+-------------------------------------------------------------------+
| IMPLEMENTATION REQUIREMENTS                                        |
+-------------------------------------------------------------------+
| [x] Idempotent commands (idempotency key in every command)         |
| [x] Persistent saga state (database)                              |
| [x] Timeout handling per step                                     |
| [x] Retry mechanism for transient failures                        |
| [x] Dead letter queue for permanent failures                      |
| [x] Monitoring: saga status, step duration, failure rate          |
| [x] Compensating transactions for every forward transaction        |
| [x] Exactly-once or at-least-once delivery + idempotent consumer  |
+-------------------------------------------------------------------+
| COMMON PITFOLDS                                                    |
+-------------------------------------------------------------------+
| [ ] Non-idempotent compensations (double-compensation causes harm)|
| [ ] Missing timeout handling (saga stuck forever)                 |
| [ ] In-memory saga state (lost on crash)                          |
| [ ] Sync communication (HTTP instead of async messaging)          |
| [ ] Ignoring compensation failures (partial inconsistency)        |
+-------------------------------------------------------------------+
```
