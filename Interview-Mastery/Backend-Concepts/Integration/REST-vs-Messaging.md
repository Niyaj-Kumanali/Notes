# REST vs Messaging (Sync vs Async)

---

## 1. Executive Summary

REST (synchronous HTTP) and messaging (asynchronous event-driven) are the two fundamental communication patterns in distributed systems. Choosing between them determines your system's coupling, resilience, scalability, and complexity.

---

## 2. Core Theory

### REST (Request-Response)
- **Pattern**: Client sends request → Server processes → Returns response
- **Protocol**: HTTP/HTTPS
- **Coupling**: Temporal (both parties must be available)
- **Guarantee**: Best-effort (unless retried)

### Messaging (Event-Driven)
- **Pattern**: Producer publishes event → Broker stores → Consumer processes
- **Protocols**: AMQP, Kafka protocol, JMS
- **Coupling**: Temporal decoupling (producer and consumer need not be online simultaneously)
- **Guarantee**: Configurable (at-most-once, at-least-once, exactly-once)

| Aspect | REST | Messaging |
|--------|------|-----------|
| Communication | Synchronous | Asynchronous |
| Coupling | Temporal + spatial | Temporal decoupled |
| Error handling | Client retries | Broker retries / DLQ |
| Scalability | Horizontal (stateless) | Consumer groups |
| Visibility | Request/response pairs | Event log |
| Latency | Low (direct) | Medium (broker hop) |
| Complexity | Low | High |

---

## 3. When to Use What

### Use REST when:
- Immediate response is required (e.g., querying order status)
- CRUD operations over resources
- Simple request-response workflows
- Client needs confirmation before proceeding

### Use Messaging when:
- Decoupling subsystems (e.g., after order placed, send email + update inventory)
- Load leveling (buffer sudden spikes)
- Event broadcasting (one event → multiple consumers)
- Reliable delivery across service boundaries
- Long-running workflows (order fulfillment pipeline)

---

## 4. Production Code

### 4.1 REST Controller

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    
    private final OrderService orderService;
    
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(@RequestBody CreateOrderRequest request) {
        OrderResponse response = orderService.createOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<OrderResponse> getOrder(@PathVariable Long id) {
        return orderService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}
```

### 4.2 Event Publisher (Messaging)

```java
@Component
public class OrderEventPublisher {
    
    private final KafkaTemplate<String, Object> kafkaTemplate;
    
    public void orderCreated(Order order) {
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(), 
            order.getCustomerEmail(),
            order.getTotal(),
            Instant.now()
        );
        
        kafkaTemplate.send("order-events", order.getId().toString(), event);
    }
}
```

### 4.3 Event Consumer

```java
@Component
public class InventoryEventHandler {
    
    private final InventoryService inventoryService;
    
    @KafkaListener(topics = "order-events", groupId = "inventory-group")
    public void handleOrderCreated(OrderCreatedEvent event) {
        inventoryService.reserveItems(event.getOrderId());
    }
}
```

### 4.4 Hybrid Approach

```java
@Service
public class OrderFacade {
    
    private final OrderRepository orderRepository;
    private final OrderEventPublisher eventPublisher;
    
    @Transactional
    public OrderResponse createOrder(CreateOrderRequest request) {
        // 1. Synchronous validation & persistence
        validateInventory(request.getItems());
        Order order = orderRepository.save(Order.from(request));
        
        // 2. Publish async event for downstream processing
        eventPublisher.orderCreated(order);
        
        // 3. Return response immediately (eventual consistency)
        return OrderResponse.from(order, "Order submitted for processing");
    }
}
```

---

## 5. Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Using REST for long-running operations | Timeouts, thread exhaustion | Return 202 Accepted + polling/callback |
| Using messaging for queries | Unnecessary complexity | Use REST for queries, events for commands |
| Ignoring idempotency | Duplicate processing | Idempotency keys in messages |
| Tight coupling on event schemas | Brittle consumers | Schema registry + versioning |
| No dead-letter queue | Lost messages | Configure DLQ for all consumers |

---

## 6. Cheat Sheet

```
═══ REST vs MESSAGING ════════════════════════════════════════

┌─ DECISION TREE ────────────────────────────────────────────┐
│ Need immediate response?                                    │
│   ├─ YES → Need ACID transactions?                         │
│   │        ├─ YES → REST (synchronous)                     │
│   │        └─ NO  → gRPC (faster than REST)                │
│   └─ NO  → Need to decouple services?                      │
│            ├─ YES → Messaging (Kafka/RabbitMQ)              │
│            └─ NO  → REST is fine                           │
└─────────────────────────────────────────────────────────────┘

┌─ PATTERNS ─────────────────────────────────────────────────┐
│ REST:  Request → [Service] → Response                      │
│ Messaging: Producer → [Broker] → Consumer                  │
│ Hybrid: REST for commands, events for reactions             │
│ CQRS: Commands via messaging, queries via REST              │
│ SAGA: Choreography (messaging) vs Orchestration (REST)     │
└─────────────────────────────────────────────────────────────┘

┌─ BEST PRACTICES ───────────────────────────────────────────┐
│ • Default to REST for simple request-response               │
│ • Use messaging when you need: resilience, broadcasting,    │
│   load leveling, or temporal decoupling                     │
│ • Start with a single approach; migrate to hybrid as needed │
│ • Monitor both sync latency (p99) and async lag             │
│ • Version event schemas independently of API versions       │
└─────────────────────────────────────────────────────────────┘
```
