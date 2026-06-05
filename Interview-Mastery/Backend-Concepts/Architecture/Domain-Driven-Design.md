# Domain-Driven Design (DDD)

## 1. Executive Summary

Domain-Driven Design is a software development approach introduced by Eric Evans in 2003 that emphasizes modeling software to closely reflect the business domain. DDD provides a set of strategic and tactical patterns for building complex systems by focusing on the core domain, establishing a ubiquitous language between developers and domain experts, and defining clear bounded contexts. It is particularly valuable for microservices decomposition and complex business logic implementation.

## 2. Core Theory

### Strategic Design

**Ubiquitous Language:** A shared language between developers and domain experts, used in code, discussions, and documentation. Every term has a precise, agreed-upon meaning.

**Bounded Context:** A logical boundary within which a particular domain model applies. Each bounded context has its own ubiquitous language and internal models. Microservices often correspond to bounded contexts.

**Context Map:** A diagram showing the relationships between bounded contexts (partnership, shared kernel, customer-supplier, conformist, anticorruption layer, open-host service, published language, separate ways, big ball of mud).

### Tactical Design (Building Blocks)

- **Entity**: An object with a distinct identity that runs through time and different states.
- **Value Object**: An immutable object defined by its attributes (no identity).
- **Aggregate**: A cluster of associated objects treated as a unit with a root entity.
- **Repository**: Provides access to aggregates, encapsulating storage and retrieval.
- **Domain Service**: Stateless service that holds domain logic that doesn't naturally fit in an entity or value object.
- **Domain Event**: Something that happened in the domain that domain experts care about.
- **Factory**: Encapsulates complex creation logic for aggregates.

## 3. Under-the-Hood Deep Dive

### Entity vs Value Object

```java
// Entity - has identity (id field)
@Entity
public class Order {
    @Id
    private String orderId; // Identity
    private String customerId;
    private Money totalAmount;
    private OrderStatus status;
    private List<OrderLine> items;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order)) return false;
        Order order = (Order) o;
        return Objects.equals(orderId, order.orderId);
    }

    @Override
    public int hashCode() {
        return Objects.hash(orderId);
    }
}

// Value Object - no identity, immutable
@Value
public class Money {
    BigDecimal amount;
    Currency currency;

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money multiply(int quantity) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(quantity)), this.currency);
    }
}

// Usage
Money price = new Money(new BigDecimal("29.99"), Currency.getInstance("USD"));
Money total = price.multiply(3); // Returns new Money instance
```

### Aggregate and Aggregate Root

```java
// Aggregate Root - Order is the root entity
@Entity
@Table(name = "orders")
public class Order {

    @Id
    private String orderId;

    @Version
    private Long version; // Optimistic concurrency

    @Embedded
    private Money totalAmount;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "order_id")
    private List<OrderLine> items = new ArrayList<>();

    // Aggregate invariant: total must match sum of line items
    public void addItem(Product product, int quantity, Money price) {
        // Business rule validation inside aggregate
        if (this.status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Cannot modify confirmed order");
        }
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }

        OrderLine line = new OrderLine(product, quantity, price);
        this.items.add(line);
        this.totalAmount = calculateTotal(); // Invariant maintained
    }

    public void submit() {
        if (this.items.isEmpty()) {
            throw new IllegalStateException("Cannot submit empty order");
        }
        // Additional business rules
        this.status = OrderStatus.SUBMITTED;
        // Register domain event
        registerEvent(new OrderSubmittedEvent(this.orderId, this.totalAmount));
    }

    private Money calculateTotal() {
        return items.stream()
            .map(OrderLine::getSubtotal)
            .reduce(Money.zero(Currency.getInstance("USD")), Money::add);
    }

    // Event registration
    @Transient
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    public List<DomainEvent> getDomainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }

    public void clearEvents() {
        domainEvents.clear();
    }

    protected void registerEvent(DomainEvent event) {
        domainEvents.add(event);
    }
}

// Entity inside aggregate
@Embeddable
public class OrderLine {
    private String productId;
    private String productName;
    private int quantity;

    @Embedded
    private Money unitPrice;

    public Money getSubtotal() {
        return unitPrice.multiply(quantity);
    }
}
```

### Repository

```java
// Repository interface - in domain layer
public interface OrderRepository {
    Order findById(OrderId id);
    void save(Order order);
    void delete(Order order);
}

// Repository implementation - in infrastructure layer
@Repository
public class JpaOrderRepository implements OrderRepository {

    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public Order findById(OrderId id) {
        return entityManager.find(Order.class, id.getValue());
    }

    @Override
    public void save(Order order) {
        // Domain events should be published after save
        entityManager.persist(order);
    }

    @Override
    public void delete(Order order) {
        entityManager.remove(order);
    }
}
```

### Domain Service

```java
// Domain Service - stateless, holds domain logic not fitting in entity
@DomainService
public class OrderPricingService {

    private final DiscountCalculator discountCalculator;
    private final TaxCalculator taxCalculator;

    public Money calculateTotal(Order order, Customer customer) {
        Money subtotal = order.calculateSubtotal();
        Money discount = discountCalculator.applyDiscount(customer, subtotal);
        Money afterDiscount = subtotal.subtract(discount);
        Money tax = taxCalculator.calculateTax(order.getItems(), customer.getAddress());
        return afterDiscount.add(tax);
    }
}
```

### Domain Event

```java
// Domain Event - immutable fact about the domain
@Value
public class OrderSubmittedEvent implements DomainEvent {
    String eventId;
    String orderId;
    Money totalAmount;
    Instant occurredOn;

    public OrderSubmittedEvent(String orderId, Money totalAmount) {
        this.eventId = UUID.randomUUID().toString();
        this.orderId = orderId;
        this.totalAmount = totalAmount;
        this.occurredOn = Instant.now();
    }

    @Override
    public Instant occurredOn() {
        return occurredOn;
    }
}

// Publishing events after aggregate save
@Component
public class DomainEventPublisher {

    @Autowired
    private ApplicationEventPublisher applicationEventPublisher;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void publishEvents(AggregateSavedEvent event) {
        AggregateRoot<?> aggregate = event.getAggregate();
        aggregate.getDomainEvents().forEach(domainEvent -> {
            applicationEventPublisher.publishEvent(domainEvent);
        });
        aggregate.clearEvents();
    }
}
```

## 4. Production Code Examples

### Complete DDD Order Module

```java
// --- Domain Layer ---

// Value Object: OrderId
@Value
public class OrderId {
    String value;
    
    public OrderId(String value) {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("OrderId must not be blank");
        }
        this.value = value;
    }
    
    public static OrderId generate() {
        return new OrderId(UUID.randomUUID().toString());
    }
}

// Value Object: CustomerId
@Value
public class CustomerId {
    String value;
}

// Aggregate Root
@Entity
public class Order {
    @EmbeddedId
    private OrderId id;
    private CustomerId customerId;
    @Embedded
    private ShippingAddress shippingAddress;
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "amount", column = @Column(name = "total_amount")),
        @AttributeOverride(name = "currency", column = @Column(name = "total_currency"))
    })
    private Money totalAmount;
    private OrderStatus status;
    @Version
    private long version;
    @ElementCollection
    @CollectionTable(name = "order_items", joinColumns = @JoinColumn(name = "order_id"))
    private List<OrderItem> items = new ArrayList<>();
    @Transient
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    protected Order() {} // JPA

    public Order(CustomerId customerId, ShippingAddress address) {
        this.id = OrderId.generate();
        this.customerId = customerId;
        this.shippingAddress = address;
        this.status = OrderStatus.DRAFT;
        this.totalAmount = Money.zero(Currency.getInstance("USD"));
    }

    public void addItem(String productId, String productName, Money unitPrice, int quantity) {
        if (status != OrderStatus.DRAFT) {
            throw new OrderAlreadyConfirmedException(id);
        }
        items.add(new OrderItem(productId, productName, unitPrice, quantity));
        recalculateTotal();
    }

    public void submit() {
        if (items.isEmpty()) {
            throw new CannotSubmitEmptyOrderException(id);
        }
        if (customerId == null) {
            throw new InvalidOrderStateException(id, "Customer not set");
        }
        this.status = OrderStatus.SUBMITTED;
        registerEvent(new OrderSubmittedEvent(id, customerId, totalAmount));
    }

    public void confirm() {
        if (status != OrderStatus.SUBMITTED) {
            throw new InvalidOrderStateException(id, "Can only confirm submitted orders");
        }
        this.status = OrderStatus.CONFIRMED;
        registerEvent(new OrderConfirmedEvent(id));
    }

    public void cancel(CancelReason reason) {
        if (status == OrderStatus.SHIPPED || status == OrderStatus.DELIVERED) {
            throw new CannotCancelOrderException(id, "Order already shipped");
        }
        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelledEvent(id, reason));
    }

    private void recalculateTotal() {
        this.totalAmount = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.zero(Currency.getInstance("USD")), Money::add);
    }

    // Domain events
    public List<DomainEvent> getDomainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }

    public void clearEvents() {
        domainEvents.clear();
    }

    protected void registerEvent(DomainEvent event) {
        domainEvents.add(event);
    }
}

// Repository interface
public interface OrderRepository {
    Optional<Order> findById(OrderId id);
    void save(Order order);
    void delete(Order order);
}

// Domain Service
@DomainService
public class OrderValidationService {
    public void validateForSubmission(Order order) {
        if (order.getItems().isEmpty()) {
            throw new ValidationException("Order must have items");
        }
        if (order.getTotalAmount().isNegativeOrZero()) {
            throw new ValidationException("Order total must be positive");
        }
    }
}

// --- Application Layer ---

@Service
@Transactional
public class OrderApplicationService {
    private final OrderRepository orderRepository;
    private final OrderValidationService validationService;
    private final DomainEventPublisher eventPublisher;

    public OrderId createOrder(CreateOrderCommand command) {
        CustomerId customerId = new CustomerId(command.getCustomerId());
        ShippingAddress address = new ShippingAddress(
            command.getStreet(), command.getCity(), command.getZipCode());
        
        Order order = new Order(customerId, address);
        orderRepository.save(order);
        return order.getId();
    }

    public void addItem(AddItemCommand command) {
        Order order = orderRepository.findById(new OrderId(command.getOrderId()))
            .orElseThrow(() -> new OrderNotFoundException(command.getOrderId()));
        
        order.addItem(
            command.getProductId(),
            command.getProductName(),
            new Money(command.getUnitPrice(), Currency.getInstance("USD")),
            command.getQuantity()
        );
        orderRepository.save(order);
    }

    public void submitOrder(SubmitOrderCommand command) {
        Order order = orderRepository.findById(new OrderId(command.getOrderId()))
            .orElseThrow(() -> new OrderNotFoundException(command.getOrderId()));
        
        validationService.validateForSubmission(order);
        order.submit();
        orderRepository.save(order);
    }
}

// --- Infrastructure Layer ---

@Repository
public class JpaOrderRepository implements OrderRepository {
    @PersistenceContext
    private EntityManager em;

    @Override
    public Optional<Order> findById(OrderId id) {
        return Optional.ofNullable(em.find(Order.class, id));
    }

    @Override
    public void save(Order order) {
        em.persist(order);
    }

    @Override
    public void delete(Order order) {
        em.remove(order);
    }
}
```

### Anti-Corruption Layer

```java
// Anti-corruption layer between legacy system and new domain
@Component
public class LegacyOrderAntiCorruptionLayer {

    private final LegacyOrderService legacyService;

    public Order toDomainOrder(LegacyOrder legacyOrder) {
        // Translate legacy model to domain model
        CustomerId customerId = new CustomerId(String.valueOf(legacyOrder.getCustNum()));
        ShippingAddress address = new ShippingAddress(
            legacyOrder.getAddrLine1(),
            legacyOrder.getAddrCity(),
            legacyOrder.getAddrZip()
        );

        Order order = new Order(customerId, address);
        for (LegacyItem item : legacyOrder.getItems()) {
            order.addItem(
                String.valueOf(item.getProdCode()),
                item.getProdDesc(),
                new Money(item.getPrice(), Currency.getInstance("USD")),
                item.getQty()
            );
        }
        return order;
    }

    public LegacyOrder toLegacyOrder(Order order) {
        // Translate domain model to legacy model
        LegacyOrder legacy = new LegacyOrder();
        legacy.setCustNum(Integer.parseInt(order.getCustomerId().getValue()));
        legacy.setTotal(order.getTotalAmount().getAmount().doubleValue());
        return legacy;
    }
}
```

## 5. Real-World Scenarios

### E-Commerce Domain

**Bounded Contexts:**
- **Ordering**: Order aggregate, returns, cancellations.
- **Catalog**: Product, Category, Inventory.
- **Billing**: Invoices, Payments, Refunds.
- **Shipping**: Shipments, Carriers, Tracking.
- **Customer**: Accounts, Addresses, Preferences.

**Ubiquitous Language:** "Order" means confirmed purchase intent (not cart). "Cart" is transient, "Order" is permanent.

### Banking Domain

**Bounded Contexts:**
- **Accounts**: Account aggregate, balance management.
- **Payments**: Transfers, Wire, ACH.
- **Compliance**: AML checks, KYC.
- **Risk**: Fraud detection, scoring.
- **Reporting**: Statements, ledgers.

## 6. Performance

### DDD Performance Considerations

| Concept | Performance Impact | Mitigation |
|---------|-------------------|------------|
| Aggregate loading | Loading entire aggregate | Keep aggregates small |
| Version locking | Optimistic concurrency | Retry on conflict |
| Event publishing | After-commit hook | Async event publishing |
| Repository abstraction | Mapping overhead | Use efficient ORM |
| Value Object immutability | Object creation | Shared flyweights |

### Guidelines
- Keep aggregates small (no more than 10-20 entities per aggregate).
- Use lazy loading for large collections.
- Consider CQRS for read-heavy workloads (bypass aggregate loading for queries).
- Use domain events asynchronously when immediate consistency isn't required.

## 7. Security

### Domain Security

- **Authorization**: Check permissions in application layer before calling domain methods.
- **Validation**: Domain invariants prevent invalid state (e.g., cannot submit empty order).
- **Encryption**: Sensitive value objects (SSN, credit card) should be encrypted.
- **Audit**: Domain events provide natural audit trail.

```java
// Application layer handles authorization before domain
@Service
public class OrderApplicationService {
    public void cancelOrder(CancelOrderCommand command, User currentUser) {
        // Authorization check in application layer
        if (!authService.canCancelOrder(currentUser, command.getOrderId())) {
            throw new AccessDeniedException("Cannot cancel order");
        }

        // Domain logic
        Order order = orderRepository.findById(command.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException(command.getOrderId()));
        order.cancel(command.getReason());
        orderRepository.save(order);
    }
}
```

## 8. Common Mistakes

### Mistake 1: Anemic Domain Model
Domain objects are just data containers (getters/setters) with no behavior. All logic in services.

```java
// WRONG - anemic domain model
@Entity
public class Order {
    private String status;

    public void setStatus(String status) { this.status = status; }
    public String getStatus() { return status; }
}

// Service does all the work
public void submitOrder(String orderId) {
    Order order = repo.findById(orderId);
    if (order.getItems().isEmpty()) throw ...;
    order.setStatus("SUBMITTED");
    repo.save(order);
}

// RIGHT - rich domain model
public class Order {
    private OrderStatus status;

    public void submit() {
        if (items.isEmpty()) throw new CannotSubmitEmptyOrderException(id);
        this.status = OrderStatus.SUBMITTED;
        registerEvent(new OrderSubmittedEvent(id));
    }
}
```

### Mistake 2: Exposing Internal State
Returning internal collections that can be modified externally.

### Mistake 3: Large Aggregates
Loading hundreds of entities per aggregate causes performance issues.

### Mistake 4: Ignoring Bounded Contexts
Using the same "User" model across all contexts instead of context-specific models.

### Mistake 5: Infrastructure Coupling in Domain
Domain layer depending on JPA, Spring, or database concerns.

## 9. Senior Engineer Perspective

### DDD and Microservices Decomposition

Each bounded context maps to a potential microservice:
1. Start with the business domain model.
2. Identify bounded contexts via communication patterns between domain experts.
3. Each context becomes a microservice candidate.
4. Define context map (which services communicate how).

### Strategic DDD Patterns

- **Core Domain**: The most important part of the system (competitive advantage). Invest heavily.
- **Supporting Subdomain**: Supports the core but not critical. Can use simpler solutions.
- **Generic Subdomain**: Common functionality (authentication, email). Buy or use open source.

### Event Storming
Workshop technique to discover domain events and bounded contexts:
1. Domain experts + developers identify domain events (past tense).
2. Group events into flows ("Happy path" vs "Exceptions").
3. Identify aggregates that produce/consume events.
4. Draw bounded context boundaries.
5. Result: Context map for microservice decomposition.

## 10. Interview Questions (20: 10 easy + 10 medium)

### Easy

1. **Q:** What is Domain-Driven Design?
   **A:** A software development approach that focuses on modeling software to match the business domain, using a shared language between developers and domain experts.

2. **Q:** Who wrote the DDD "Blue Book"?
   **A:** Eric Evans ("Domain-Driven Design: Tackling Complexity in the Heart of Software", 2003).

3. **Q:** What is ubiquitous language?
   **A:** A shared, precise language used by both domain experts and developers in code, conversations, and documentation.

4. **Q:** What is a bounded context?
   **A:** A logical boundary where a particular domain model applies. Each bounded context has its own ubiquitous language.

5. **Q:** What is the difference between Entity and Value Object?
   **A:** Entity has identity (equals by ID). Value Object has no identity, is immutable, and equals by attributes.

6. **Q:** What is an Aggregate?
   **A:** A cluster of domain objects treated as a unit, with a root entity that controls access to all objects within.

7. **Q:** What are domain events?
   **A:** Events that domain experts care about, representing something that happened in the domain.

8. **Q:** What is a repository in DDD?
   **A:** A pattern that provides access to aggregates, abstracting the underlying storage mechanism.

9. **Q:** What is the difference between domain service and application service?
   **A:** Domain service holds domain logic that doesn't fit in an entity. Application service orchestrates use cases, coordinates domain objects.

10. **Q:** What is an anti-corruption layer?
    **A:** A translation layer that prevents a legacy system's model from corrupting a new domain model.

### Medium

11. **Q:** How does DDD help with microservices decomposition?
    **A:** Each bounded context is a natural microservice boundary. Context maps define inter-service communication patterns.

12. **Q:** What is event storming?
    **A:** A collaborative workshop technique to discover domain events, aggregates, and bounded contexts with domain experts.

13. **Q:** What is the difference between core, supporting, and generic subdomains?
    **A:** Core domain (competitive advantage, invest heavily), supporting (necessary but not core, simpler), generic (common, buy/use open source).

14. **Q:** How do you keep aggregates small?
    **A:** Identify true transactional boundaries. If two entities can be updated independently, they should be separate aggregates.

15. **Q:** What is the rule of thumb for aggregate design?
    **A:** Make aggregates as small as possible while maintaining invariants. Reference other aggregates by ID, not by object reference.

16. **Q:** How do you handle transactions across aggregates?
    **A:** Use eventual consistency with domain events. One transaction per aggregate. Saga pattern for multi-aggregate workflows.

17. **Q:** What is the difference between tactical and strategic DDD?
    **A:** Strategic DDD (bounded contexts, context maps) handles large-scale structure. Tactical DDD (entities, value objects, aggregates) handles implementation details.

18. **Q:** How does DDD relate to CQRS?
    **A:** CQRS is often used with DDD: command side uses aggregates for writes; query side uses read models for queries.

19. **Q:** What is a factory in DDD?
    **A:** A pattern for encapsulating complex aggregate creation logic that doesn't belong in the aggregate itself.

20. **Q:** How do you validate DDD models?
    **A:** Continuous collaboration with domain experts. Model validation workshops. Test domain logic with unit tests using the ubiquitous language.

## 11. Advanced Interview Questions (20: 10 hard + 10 system design)

### Hard

1. **Q:** Design aggregate boundaries for an e-commerce order system.
    **A:** Order aggregate (Order + OrderLines). Customer aggregate (Customer + Addresses). Product aggregate (Product + Inventory). Order references Customer by ID, Product by ID. Separate aggregates because they have different transactional boundaries.

2. **Q:** How do you handle eventual consistency between aggregates in the same bounded context?
    **A:** Domain events published after aggregate save. Event handlers update other aggregates asynchronously. Example: OrderSubmitted event triggers Inventory aggregate to reserve stock.

3. **Q:** Design a value object that must be encrypted at rest.
    **A:** EncryptedValue wrapper in infrastructure layer. Domain layer uses the value object (e.g., CreditCardNumber). Repository implementation encrypts/decrypts when persisting. Domain never sees raw encryption.

4. **Q:** How do you version domain events?
    **A:** Event header with type name and version. Backward-compatible schema evolution. Schema registry for compatibility checks.

5. **Q:** Design a domain model for a subscription billing system.
    **A:** Subscription aggregate (status, plan, billing cycle). PaymentMethod value object. Invoice aggregate generated by domain service. Domain event: SubscriptionCharged, InvoiceGenerated, PaymentFailed.

6. **Q:** How do you refactor an anemic domain model to a rich one?
    **A:** Identify business rules in services, move them into domain entities as behavior (methods). Add validation in setters. Encapsulate internal state. Add domain events for side effects.

7. **Q:** Design aggregate for a banking account.
    **A:** Account aggregate (balance, transactions list). AccountOperationsService (deposit, withdraw, transfer). Transaction as value object (immutable event). Balance invariant: cannot go below zero (unless overdraft allowed).

8. **Q:** How do you handle cross-bounded-context authentication?
    **A:** Identity and Access Context handles authentication. Issues tokens. Other contexts validate tokens (anti-corruption layer translates user representation).

9. **Q:** Design a context map for an e-commerce platform.
    **A:** Ordering (partnership with Billing), Catalog (shared kernel with Inventory), Shipping (customer-supplier with Ordering), Recommendations (separate ways from Catalog).

10. **Q:** How does DDD apply to reporting and analytics?
    **A:** Reporting is a separate bounded context. Uses CQRS read models (materialized views) from other contexts. Anti-corruption layer translates domain events to reporting model.

### System Design

11. **Q:** Design an online food delivery platform using DDD.
    **A:** Bounded contexts: Restaurant (menu, hours, location), Ordering (cart, order, status), Delivery (rider, route, tracking), Payment (charges, refunds), Customer (profile, preferences). Context map with events connecting contexts.

12. **Q:** Design a healthcare system using DDD.
    **A:** Bounded contexts: Patient (records, demographics), Appointment (scheduling, availability), Billing (insurance, claims), Pharmacy (prescriptions, inventory). Aggregate: PatientRecord (medical history, appointments). Strong privacy boundaries.

13. **Q:** Design a flight booking system using DDD.
    **A:** Bounded contexts: Inventory (flights, seats), Booking (reservations, passengers), Pricing (fares, rules), Payment (transactions, refunds). Aggregate: Booking (passengers, flights, payment info). Saga for booking flow.

14. **Q:** Design a SaaS platform using DDD with multi-tenancy.
    **A:** Bounded contexts: Tenant (provisioning, config), Billing (subscription, usage), Core (domain per tenant type). Tenant context provides tenant identity; all other contexts use it for isolation.

15. **Q:** Design a content management system using DDD.
    **A:** Bounded contexts: Content (documents, versions, publishing), Author (writers, permissions), Media (images, videos), Workflow (review, approval). Aggregate: Document (content, metadata, version history).

16. **Q:** Design a ride-sharing application using DDD.
    **A:** Bounded contexts: Rider (profile, payment), Driver (profile, vehicle, status), Trip (ride, route, fare), Matching (dispatch, geolocation), Payment (charges, payouts). Aggregate: Trip (rider, driver, route, fare, status).

17. **Q:** Design an inventory management system using DDD.
    **A:** Bounded contexts: Stock (inventory, warehouse), Procurement (purchase orders, suppliers), Sales (orders, allocations), Forecasting (demand, trends). Aggregate: InventoryItem (SKU, quantity, location, reservation).

18. **Q:** Design a social media platform using DDD.
    **A:** Bounded contexts: User (profile, friends), Content (posts, media), Feed (timeline, ranking), Notification (alerts, digests), Moderation (reports, filters). Aggregate: Post (content, author, comments, likes).

19. **Q:** Design a project management tool using DDD.
    **A:** Bounded contexts: Project (tasks, milestones), Team (members, roles), Time (tracking, reports), Billing (invoicing, rates). Aggregate: Task (title, assignee, status, due date, comments).

20. **Q:** Design a hotel booking system using DDD.
    **A:** Bounded contexts: Inventory (rooms, availability), Booking (reservations, guests), Pricing (rates, seasons), Housekeeping (room status, cleaning). Aggregate: Reservation (guest, room, dates, rate, status).

## 12. Expert-Level Interview Questions (10: architect-level)

1. **Q:** How do you evolve a domain model over years as business requirements change?
    **A:** Version bounded contexts independently. Create new version of aggregate with new behavior. Run old + new versions in parallel during migration. Event versioning in event store. Anti-corruption layer between old and new. Eventually deprecate old version.

2. **Q:** Design a domain model that must support both synchronous and eventual consistency use cases.
    **A:** Core transactional boundaries use aggregates (strong consistency). Across aggregates, use domain events (eventual consistency). For reads needing strong consistency, use query service that loads aggregate directly (bypass read model cached data).

3. **Q:** How do you handle domain events in a microservices environment where each service owns its database?
    **A:** Outbox pattern: events stored in same DB transaction as aggregate. Outbox publisher sends events to message broker. Consumer services process events. Schema registry for event versioning.

4. **Q:** Design a strategy for introducing DDD to a team that uses CRUD/transaction script pattern.
    **A:** Start small: pick one complex domain (not CRUD). Model aggregates with domain experts. Extract domain logic from services into aggregates. Add unit tests. Prove benefits before expanding.

5. **Q:** How do you reconcile DDD aggregates with database normalization and query performance?
    **A:** CQRS: write model uses aggregates (normalized for consistency). Read model uses denormalized views (optimized for queries). Eventual consistency between them.

6. **Q:** Design a domain model for a financial trading platform with millisecond latency requirements.
    **A:** Aggregates must be very small (order book per symbol). In-memory state (event sourced). No database in critical path. Events persisted asynchronously after trade execution. Rigorous aggregate sizing for performance.

7. **Q:** How do you validate that bounded contexts are correctly identified?
    **A:** Check: each context has its own ubiquitous language. Contexts have clear relationships (partnership, shared kernel). Business experts agree on boundaries. Change frequency: if two parts change for different reasons, they may be separate contexts.

8. **Q:** Design a governance model for DDD adoption across a large organization.
    **A:** DDD Center of Excellence. Event Storming workshops for new domains. Architecture Decision Records. Code review with DDD principles checklist. Shared model repository. Regular model validation sessions with domain experts.

9. **Q:** How do you split a large aggregate that has become a performance bottleneck?
    **A:** Identify parts that don't need transactional consistency. Move them to separate aggregates. Reference by ID instead of by object. Use eventual consistency for updates. Add domain events for cross-aggregate notifications.

10. **Q:** Design a system that uses DDD for core domain but CRUD for generic subdomains.
    **A:** Core domain: full DDD (aggregates, events, rich model). Supporting subdomain: simplified DDD (entities only, no event sourcing). Generic subdomain: CRUD with services. Anti-corruption layer between DDD core and non-DDD contexts.

## 13. Debugging & Troubleshooting

### Common DDD Issues

**Issue: Aggregate loading is slow**
- Check if aggregate references too many objects.
- Consider lazy loading for large collections.
- Review aggregate boundaries (too large?).

**Issue: Domain events not publishing**
- Check transactional event listener.
- Verify save happens before event publication.
- Check outbox pattern if using async events.

**Issue: Ubiquitous language mismatch**
- Schedule regular sessions with domain experts.
- Review code together: does code match business terms?
- Maintain a glossary of domain terms.

**Issue: Anemic domain model returns**
- Monitor code reviews for logic leaking into services.
- Ensure new features add behavior to domain objects.

## 14. Comparison Section

### DDD vs Transaction Script

| Aspect | DDD | Transaction Script |
|--------|-----|-------------------|
| Logic location | Domain objects/aggregates | Services/procedures |
| Model | Rich domain model | Anemic data model |
| Complexity handling | Excellent | Poor for complex domains |
| When to use | Complex business rules | Simple CRUD |
| Learning curve | Steep | Shallow |
| Testability | High (isolated domain) | Medium |

### DDD vs Active Record

| Aspect | DDD | Active Record |
|--------|-----|---------------|
| Pattern | Aggregate with repositories | Object that wraps DB row |
| Business logic | Rich methods on aggregate | Mix of data access + logic |
| Persistence | Repository abstraction | Direct CRUD on object |
| Complexity | Better for complex | Simple CRUD |
| Framework | Axon, custom | Rails, Spring Data |

### Strategic vs Tactical DDD

| Aspect | Strategic | Tactical |
|--------|-----------|----------|
| Focus | Large-scale structure | Implementation patterns |
| Tools | Context map, bounded context | Entities, value objects, aggregates |
| Audience | Architects | Developers |
| Output | Bounded context map | Domain model code |
| When | System design phase | Implementation phase |

## 15. Revision Notes

### Quick Recap
- **Ubiquitous Language**: Shared language between devs and domain experts.
- **Bounded Context**: Boundary within which a model applies.
- **Entity**: Has identity, mutable, changes over time.
- **Value Object**: No identity, immutable, defined by attributes.
- **Aggregate**: Cluster of entities/value objects owned by root entity.
- **Domain Event**: Something the business cares about that happened.
- **Repository**: Persistence abstraction for aggregates.
- **Domain Service**: Stateless logic that doesn't fit in entity.
- **Application Service**: Orchestrates use cases, delegates to domain.
- **Anti-Corruption Layer**: Translation between bounded contexts.

### Aggregate Design Rules
1. Reference other aggregates by ID only.
2. Keep aggregates small - one transaction per aggregate.
3. Use eventual consistency across aggregates.
4. Apply invariants within aggregate boundaries.
5. Domain events for cross-aggregate communication.

## 16. Cheat Sheet

```
+-------------------------------------------------------------------+
|               DOMAIN-DRIVEN DESIGN CHEAT SHEET                     |
+-------------------------------------------------------------------+
| STRATEGIC PATTERNS           | TACTICAL PATTERNS                  |
+------------------------------+------------------------------------+
| Bounded Context              | Entity (has identity)               |
| Ubiquitous Language          | Value Object (no identity,          |
| Context Map                  |   immutable)                        |
| Core / Supporting / Generic  | Aggregate (cluster, root entity)   |
|   Subdomain                  | Repository (persistence)            |
| Anti-Corruption Layer        | Domain Service (stateless logic)    |
| Open-Host Service            | Domain Event (something happened)   |
| Published Language           | Factory (creation)                  |
+------------------------------+------------------------------------+
| LAYERED ARCHITECTURE                                              |
+-------------------------------------------------------------------+
| Application: orchestrates use cases (thin)                        |
| Domain: business logic, entities, value objects, aggregates (core)|
| Infrastructure: persistence, messaging, external APIs             |
| Presentation: REST controllers, DTOs, views                       |
+-------------------------------------------------------------------+
| UBIQUITOUS LANGUAGE EXAMPLE                                       |
+-------------------------------------------------------------------+
| Business Term | Code Representation                               |
+---------------+---------------------------------------------------+
| Order         | Order aggregate root                              |
| Order Line    | OrderLine value object                            |
| Submit Order  | order.submit() method                            |
| Order         | OrderSubmitted domain event                       |
| Submitted     |                                                     |
| Reserve       | inventoryService.reserve() domain service         |
| Inventory     |                                                     |
| Cancel Order  | order.cancel(reason) method                      |
+---------------+---------------------------------------------------+
| AGGREGATE DESIGN RULES                                            |
+-------------------------------------------------------------------+
| [x] Aggregate root is the only entry point                         |
| [x] External objects reference aggregate by ID, not object        |
| [x] One transaction creates/updates one aggregate                 |
| [x] Events for cross-aggregate consistency                       |
| [x] Aggregate invariants are always enforced                     |
| [x] Keep aggregates small (usually < 10 entities)                |
+-------------------------------------------------------------------+
```
