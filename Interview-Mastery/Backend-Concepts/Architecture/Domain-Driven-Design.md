# Domain-Driven Design (DDD)

---

## Overview

- **Definition:** A software development approach introduced by Eric Evans that emphasizes modeling software to closely reflect the business domain using a shared language between developers and domain experts.
- **Why It Exists:** Complex business domains require a model that evolves with business understanding. DDD provides strategic patterns (bounded contexts, context maps) and tactical patterns (entities, value objects, aggregates) to manage complexity and align software with business goals.
- **Key Concepts:** **Ubiquitous Language** (shared language between devs and domain experts), **Bounded Context** (logical boundary for a domain model), **Entity** (object with identity), **Value Object** (immutable, defined by attributes), **Aggregate** (cluster of objects with a root entity), **Repository** (persistence abstraction), **Domain Event** (something the business cares about), **Domain Service** (stateless domain logic), **Anti-Corruption Layer** (translation between contexts)
- **DDD vs CRUD** — CRUD treats all operations as Create, Read, Update, Delete on data. DDD models behavior: `order.submit()`, not `order.setStatus(SUBMITTED)`. CRUD is appropriate for simple admin screens where the user directly manipulates data. DDD is essential for complex domains with business rules, invariants, and workflows (finance, insurance, logistics).
- **Context Map Relationships** — Partnership (two contexts cooperate), Shared Kernel (shared subset of model), Customer-Supplier (upstream/downstream dependency), Conformist (downstream conforms to upstream model), Anticorruption Layer (translation layer), Open Host Service (published API), Separate Ways (no relationship). Choose the relationship based on team dynamics and integration requirements.

---

## Core Concepts

- **Entity vs Value Object:** Entity has a distinct identity that persists across time and states — equals by ID. Value Object is immutable, has no identity, and equals by its attributes — e.g., `Money(amount, currency)`.
- **Aggregate and Aggregate Root:** A cluster of associated objects treated as a unit with a root entity that controls access. External objects reference the aggregate by ID only. One transaction per aggregate. Invariants are enforced within aggregate boundaries.
- **Repository:** Provides access to aggregates, encapsulating storage and retrieval. Defined in the domain layer, implemented in the infrastructure layer. Returns aggregates in a consistent state.
- **Domain Service:** Stateless service that holds domain logic that doesn't naturally fit in an entity or value object. Different from Application Service which orchestrates use cases.
- **Domain Event:** An immutable fact about something that happened in the domain that domain experts care about. Published after aggregate changes are persisted. Used for cross-aggregate and cross-context communication.
- **Strategic Design Patterns:** Core Domain (competitive advantage, invest heavily), Supporting Subdomain (necessary but not core), Generic Subdomain (common — buy or use open source). Bounded Contexts and Context Maps define inter-context relationships.
- **Hexagonal Architecture (Ports and Adapters)** — The domain layer is at the center, with ports (interfaces) that define how the domain interacts with the outside world. Adapters implement these ports for specific technologies: REST controllers, JPA repositories, Kafka producers. The domain has zero dependencies on infrastructure — it only knows about its own interfaces. This enforces DIP and keeps the domain pure.
- **Domain Events vs Integration Events** — Domain events represent something that happened within an aggregate (`OrderSubmitted`). Integration events communicate across bounded contexts (`PaymentProcessed`). Domain events are consumed within the same context; integration events cross context boundaries. The distinction prevents coupling between contexts — an integration event should not expose internal domain details.

```java
// Value Object — immutable, no identity
@Value
public class Money {
    BigDecimal amount;
    Currency currency;

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) throw new IllegalArgumentException("Currency mismatch");
        return new Money(this.amount.add(other.amount), this.currency);
    }
}

// Aggregate Root — enforces invariants
@Entity
public class Order {
    @Id private String orderId;
    @Version private Long version;
    private OrderStatus status;
    @OneToMany(cascade = ALL) private List<OrderLine> items = new ArrayList<>();

    public void addItem(Product product, int quantity, Money price) {
        if (this.status != OrderStatus.DRAFT) throw new IllegalStateException("Cannot modify confirmed order");
        if (quantity <= 0) throw new IllegalArgumentException("Quantity must be positive");
        this.items.add(new OrderLine(product, quantity, price));
    }

    public void submit() {
        if (this.items.isEmpty()) throw new IllegalStateException("Cannot submit empty order");
        this.status = OrderStatus.SUBMITTED;
        registerEvent(new OrderSubmittedEvent(this.orderId));
    }
}

// Repository — abstraction in domain layer
public interface OrderRepository {
    Order findById(OrderId id);
    void save(Order order);
    void delete(Order order);
}
```

---

## Common Mistakes

- **Anemic Domain Model** — domain objects are just data containers (getters/setters) with no behavior, while all logic lives in services. Instead, put behavior in the domain objects: `order.submit()` not `orderService.submitOrder(order)`. This *looks correct* because: keeping domain objects clean as data holders follows the principle of separation of concerns, and services seem like the right place for "logic" — the behavior that gets scattered across multiple service methods becomes hard to find and test.
- **Exposing Internal State** — returning internal collections that can be modified externally. Return unmodifiable copies or streams. This *looks correct* because: exposing the collection through a getter is the simplest way to provide access, and it works fine for reads — the external modification only becomes a problem when a caller accidentally clears the list in production.
- **Large Aggregates** — loading hundreds of entities per aggregate causes performance issues. Keep aggregates small (usually fewer than 10 entities). This *looks correct* because: loading more entities provides more data and ensures consistency, and the performance impact may not be noticeable in development — the cost of loading hundreds of entities only manifests in production under real traffic.
- **Ignoring Bounded Contexts** — using the same "User" model across all contexts instead of context-specific models tailored to each bounded context's needs. This *looks correct* because: DRY (Don't Repeat Yourself) is a fundamental principle, and having one "User" model eliminates duplication — the problem only surfaces when the billing context needs credit info and the support context needs only a name, forcing the same entity to serve conflicting requirements.
- **Infrastructure Coupling in Domain** — domain layer depending on JPA, Spring, or database concerns. Domain should be plain Java objects with no framework annotations. This *looks correct* because: framework annotations are convenient and standard practice in modern frameworks like Spring, and they seem harmless — coupling to infrastructure only becomes painful when you need to test domain logic outside the framework or migrate to a different persistence technology.
- **Using Events to Communicate Instead of Commands** — Publishing `SendEmail` (a command) instead of `OrderSubmitted` (an event) couples the producer to the consumer's behavior. Events should be past-tense business facts that multiple consumers can interpret independently. This *looks correct* because: the immediate need is to send an email when an order is submitted, so publishing a `SendEmail` event seems direct and efficient — the coupling only becomes apparent when a second consumer wants to react to the same order submission for a different purpose.
- **Over-engineering with Value Objects** — Replacing every `String` with a type-safe value object adds ceremony. Apply value objects to concepts that have validation rules, formatting, or behavior. `EmailAddress` (validates format) is justified. `FirstName` (no rules beyond not null) is over-engineering. This *looks correct* because: type safety is always good, and wrapping primitives in types prevents bugs — the ceremony outweighs the benefit when the value has no validation or behavior beyond being a string.

---

## Key Design Considerations

- **DDD and Microservices Decomposition** — each bounded context maps to a potential microservice. Start with the business domain model, identify bounded contexts via communication patterns, define context maps for inter-service relationships.
- **Event Storming** — a collaborative workshop technique where domain experts and developers identify domain events, group them into flows, identify aggregates, and draw bounded context boundaries.
- **Aggregate Design Rules** — reference other aggregates by ID only; keep aggregates small; one transaction per aggregate; use eventual consistency across aggregates; enforce invariants within aggregate boundaries; use domain events for cross-aggregate communication.
- **Anti-Corruption Layer** — a translation layer that prevents a legacy system's model from corrupting the new domain model. Translates between the legacy model and the domain model at the bounded context boundary.
- **Layered Architecture** — Application layer (orchestrates, thin), Domain layer (business logic, core), Infrastructure layer (persistence, messaging, external APIs), Presentation layer (REST controllers, DTOs).
- **Specification Pattern** — Encapsulates a business rule that can be evaluated against an entity. Example: `OrderIsOverdue`, `CustomerIsVip`. Specifications can be combined with AND/OR for complex rules. Useful for filtering, validation, and business rule extraction from domain objects.
- **Domain Primitive Pattern** — A value object that wraps a primitive type with domain-specific validation and behavior. `Email` ensures format validity, `PositiveAmount` ensures value > 0. Domain primitives push validation to the edges — invalid states are impossible to represent in the domain model.
- **Event Sourcing with DDD** — Stores aggregate state as a sequence of domain events. Current state is derived by replaying events. Benefits: complete audit trail, temporal queries (state at any point in time), and event-driven communication. Trade-offs: higher storage, eventual consistency, and the complexity of event schema evolution.

---

## Real-World Scenarios

### Scenario 1: E-Commerce Order Domain with Bounded Contexts
**Context:** An e-commerce company has a monolithic "Order" concept used everywhere — order management, inventory, shipping, billing, analytics. Each team has different definitions of what an "Order" is. The same entity is pulled in conflicting directions.

**Resolution:** Split into bounded contexts. The **Ordering** context has `Order` with items, prices, and status (DRAFT → SUBMITTED). The **Shipping** context has `Shipment` with addresses, carrier, and tracking. The **Billing** context has `Invoice` with amounts, payment status, and refunds. Each context has its own understanding of the order — the Shipping context doesn't need item prices, and the Billing context doesn't need the shipping carrier. Context maps define the relationships between these bounded contexts.

```java
// Ordering Bounded Context
@Entity
@Table(name = "orders")
public class Order { // Aggregate Root
    @Id private OrderId id;
    private OrderStatus status;
    @OneToMany(cascade = ALL) private List<OrderLine> items;
    private Money total;

    public void submit() {
        if (items.isEmpty()) throw new IllegalStateException("Empty order");
        this.status = OrderStatus.SUBMITTED;
        registerEvent(new OrderSubmittedEvent(id, total, items.stream()
            .map(OrderLine::getProductId).toList()));
    }
}

// Shipping Bounded Context — different concept of "order"
@Document
public class Shipment {
    @Id private String id;
    private String orderId; // References order by ID only
    private Address shippingAddress;
    private String carrier;
    private TrackingStatus status;
}
```

### Scenario 2: Banking Domain with Ubiquitous Language
**Context:** A banking team building a loan application system. Developers use technical terms ("Insert into loans table", "Update the status flag"), while domain experts use business terms ("Underwrite the application", "Disburse the funds"). Miscommunication causes constant rework.

**Resolution:** Establish a ubiquitous language. Business experts and developers agree that "Loan Application" goes through states: SUBMITTED → UNDERWRITING → APPROVED → DISBURSED → ACTIVE. The codebase uses exactly these terms. The `LoanApplication` entity has methods like `submit()`, `startUnderwriting()`, `approve()`, `disburse()`. Repository methods are named after business concepts: `findPendingUnderwriting()`. The language is used in code, database schemas, REST endpoints, and Jira tickets.

### Scenario 3: Legacy Integration with Anti-Corruption Layer
**Context:** A company is migrating from a 20-year-old legacy ERP system to a new microservices platform. The legacy system has a 50-column `ORDERS` table with cryptic column names (`ORD_ID`, `CUST_NUM`, `PRD_CD`, `AMT_BASE`, `AMT_TAX`).

**Resolution:** Build an Anti-Corruption Layer (ACL) between the new domain model and the legacy system. The ACL translates between the legacy schema and the new domain model. The new services work with clean domain objects (`Order`, `Customer`, `Product`). The ACL handles the ugly mapping, shielding the new system from the legacy model's complexity.

```java
// Anti-Corruption Layer
@Component
public class LegacyOrderTranslator {
    public Order toDomain(LegacyOrder legacy) {
        return Order.builder()
            .orderId(new OrderId(legacy.getOrdId()))
            .customerId(new CustomerId(legacy.getCustNum()))
            .items(parseItems(legacy.getPrdCd(), legacy.getAmtBase()))
            .total(new Money(legacy.getAmtBase().add(legacy.getAmtTax()), USD))
            .build();
    }
}
```

---

## Scenario-Based Questions

1. **Q: You're building a hotel booking system where the "Room" concept means different things to different teams: Front Desk cares about room number and cleanliness status, Housekeeping cares about supplies and maintenance, and Accounting cares about rate and occupancy. How do you model this?**
   - A: Use bounded contexts. Each team gets its own "Room" model tailored to its needs. Front Desk context: `Room(roomNumber, status: CLEAN/DIRTY/OCCUPIED)`. Housekeeping context: `MaintenanceTask(roomId, suppliesNeeded, lastService)`. Accounting context: `RoomRate(roomType, basePrice, seasonalAdjustment)`. They share a room ID but have different models. Context maps define the relationships between these bounded contexts.

> **Interview follow-up:** The Front Desk marks a room as DIRTY after checkout, but Housekeeping doesn't see the update for 5 minutes because they use separate bounded contexts with eventual consistency — what happens if a new guest checks into that room during those 5 minutes?

2. **Q: You're working on a payment system where domain experts talk about "Settling a transaction" but developers have implemented it as `updatePaymentStatus(transactionId, "SETTLED")`. What's wrong and how do you fix it?**
   - A: This violates the ubiquitous language principle. The code should express the business concept. Refactor to `payment.settle()` on a `Payment` aggregate. The method name matches the business language. Tests become readable: `assertThat(payment.isSettled()).isTrue()`. This prevents misinterpretation between domain experts and developers.

> **Interview follow-up:** The `payment.settle()` method now sounds right, but the settlement process involves calling an external clearing house API that can take 30 seconds — should `settle()` be synchronous in the aggregate, or does the aggregate's method just validate and then a domain service does the actual settlement? Where does the line belong?

3. **Q: Your aggregate loads 200+ entities for a single order, causing performance problems. How do you redesign it?**
   - A: Review the true transactional boundary. Does updating an order line item really require loading all 200 line items? Split into smaller aggregates: `Order(header)` + `OrderLineItem` as a separate aggregate referenced by ID. The `Order` aggregate holds invariants (total amount, status). Individual line items can be updated independently. Reference other aggregates by ID only.

> **Interview follow-up:** You split the order into separate aggregates, but now there's a business rule that says "an order cannot exceed $10,000 total" — if line items are added independently as separate aggregates, how do you enforce this invariant consistently?

4. **Q: Your team is doing Event Storming for a new insurance claims system. The whiteboard is chaos with 200+ sticky notes and everyone arguing. How do you bring structure?**
   - A: Start with the happy path — identify the core domain events in chronological order (Claim Filed → Claim Assessed → Claim Approved → Payment Sent). Then add alternate paths and exceptions. Group events by bounded context (Claims Assessment, Payment Processing, Fraud Detection). Timebox each phase: 30min for events, 30min for aggregates, 30min for bounded contexts. Use different colored stickies for events (orange), commands (blue), aggregates (yellow), and actors (green).

> **Interview follow-up:** After Event Storming, you discover 12 bounded contexts for a team of 6 developers — each context maps to a potential microservice, but you can't build and operate 12 services. How do you prioritize which contexts become independent services and which stay in a modular monolith?

5. **Q: You're introducing DDD to a team with a strong CRUD mindset. They keep creating getters/setters and putting all logic in services. How do you shift to a rich domain model?**
   - A: Start with one aggregate and enforce the behavioral style. Instead of `order.setStatus(OrderStatus.SUBMITTED)`, use `order.submit()`. Instead of `order.getItems().add(item)`, use `order.addItem(product, quantity)`. Review these in code review. Show how encapsulating behavior in the aggregate makes service code thinner and more testable. The anemic domain model is the most common DDD anti-pattern — fight it early.

6. **Q: Your order processing has a requirement: "Orders over $1000 require manager approval." Where does this logic belong — entity, service, or somewhere else?**
   - A: This is domain logic and belongs in the `Order` aggregate. The `submit()` method checks the total: `if (this.total.isGreaterThan(new Money(1000))) { this.status = PENDING_APPROVAL; }`. The application service calls `order.submit()` and handles the resulting domain event (`OrderWaitingForApproval`). This keeps the business rule explicit in the domain model, testable, and visible to domain experts.

> **Interview follow-up:** Six months later, the business changes the rule to "orders over $5000 need VP approval, orders $1000-$5000 need manager approval, and some VIP customers are exempt entirely" — how does this affect the aggregate design, and where do you draw the line between domain logic worth modeling in the aggregate vs externalizing to a rules engine?

7. **Q: Your team is implementing a new notification feature. A senior engineer says "just add an Email field to the Account entity and a sendEmail method." Why might this be problematic from a DDD perspective?**
   - A: Sending email is not a core domain concern — it's a generic subdomain. Adding it to the Account entity violates SRP and distracts from the core domain. Instead, publish a domain event (`AccountCreated`) and let an infrastructure service consume it and send the email. The domain model stays focused on business logic, not technical concerns.

8. **Q: Your company is acquiring another company with its own customer database. The legacy customer model has fields your new system doesn't need, and your model has fields the legacy system doesn't have. How do you integrate?**
   - A: Build an Anti-Corruption Layer between the two systems. The ACL translates between the legacy customer model and your domain model. Each system maintains its own bounded context. The ACL handles the mapping for shared operations (customer creation, address update) and prevents the legacy model's complexity from leaking into your clean domain.

9. **Q: A product manager asks for a feature: "When an order is shipped, notify the customer via email." Where should this logic live in a DDD architecture?**
   - A: The domain model publishes `OrderShippedEvent` after the `Order.ship()` method executes. An application-layer event handler subscribes to this event and calls `NotificationService.sendOrderShippedEmail(customerId, orderId)`. The email sending itself is infrastructure. The domain knows that shipping triggers something; it doesn't know or care about email protocols, templates, or delivery status.

10. **Q: Your team is struggling with microservices boundaries. Services keep growing and overlapping. How does DDD help you decompose?**
    - A: Use bounded contexts as microservice boundaries. Start with Event Storming to discover aggregates and bounded contexts. Each bounded context maps to a potential microservice. Define context maps to show relationships (partner, shared kernel, anti-corruption layer). If two services share too much data or need to be deployed together, they're likely the same bounded context and should stay as one service.

---

## Interview Questions

1. **What is Domain-Driven Design?**
   - A: A software development approach by Eric Evans that emphasizes modeling software to match the business domain using a shared language (ubiquitous language) between developers and domain experts.

2. **What is the difference between an Entity and a Value Object?**
   - A: Entity has a distinct identity (equals by ID, mutable state). Value Object has no identity, is immutable, and equals by its attributes — e.g., `Money(amount, currency)`.

3. **What is an Aggregate?**
   - A: A cluster of domain objects treated as a single unit with an aggregate root that controls access. External objects reference the aggregate by ID only. One transaction per aggregate. Invariants are enforced within the aggregate boundary.

4. **What is the difference between a Domain Service and an Application Service?**
   - A: Domain Service holds domain logic that doesn't naturally fit in an entity or value object (e.g., `TransferService.transferFunds(from, to, amount)`). Application Service orchestrates use cases — it's thin and delegates to domain objects.

5. **How does DDD help with microservices decomposition?**
   - A: Each bounded context maps naturally to a microservice. Context maps define inter-service communication (events, APIs). Subdomain analysis (core/supporting/generic) guides investment decisions.

6. **What is Event Storming?**
   - A: A collaborative workshop technique where domain experts and developers identify domain events, group them into flows, identify aggregates, and draw bounded context boundaries using sticky notes on a wall.

7. **What is the difference between Core, Supporting, and Generic subdomains?**
   - A: Core domain (competitive advantage — invest heavily, build in-house). Supporting (necessary but not differentiating — simpler solutions). Generic (common functionality — buy or use open source).

8. **What is an Anti-Corruption Layer?**
   - A: A translation layer that prevents a legacy or external system's model from corrupting your domain model. It translates between the two models, keeping your domain clean.

9. **What is the Ubiquitous Language?**
   - A: A shared language between developers and domain experts used in code, database schemas, REST endpoints, documentation, and conversations. It ensures that business concepts map directly to code constructs.

10. **What are the rules for aggregate design?**
    - A: Reference other aggregates by ID only, keep aggregates small (usually <10 entities), one transaction per aggregate, use eventual consistency across aggregates, enforce invariants within the aggregate boundary.

---

## Developer Recommendations

- **Start with Event Storming before writing code** — Event Storming sessions with domain experts reveal the true domain model in hours, not weeks. You'll discover aggregates, bounded contexts, and domain events before writing a single line of code. The cost of fixing a wrong model in Event Storming is zero; the cost of fixing it in code is exponential. A healthcare startup skipped Event Storming and built their patient intake system based on developer assumptions — 6 months in, domain experts pointed out that "Appointment" and "Visit" were distinct concepts that the code treated as one, requiring a 3-month rewrite.

- **Use anemic domain models only for simple CRUD, never for complex domains** — An anemic domain model (entities with only getters/setters, all logic in services) is the #1 DDD anti-pattern. For complex domains, put behavior in the domain objects: `order.submit()`, not `orderService.submitOrder(order)`. Rich domain models are more testable, more maintainable, and more aligned with business language.

- **Keep aggregates small** — The most common aggregate design mistake is making them too large. If an aggregate loads 50+ entities, it's too big. True transactional boundaries are smaller than you think. If two entities can be updated independently (different transactions, different times), they should be separate aggregates. Reference other aggregates by ID only — never by object reference. An insurance company modeled a policy as a single aggregate containing all claims, payments, and endorsements — a single policy load fetched 300+ entities, causing 5-second database queries and frequent deadlocks when two agents updated different claims on the same policy simultaneously.

- **Use Value Objects extensively for primitive obsession** — Replace `String email`, `String phone`, `BigDecimal amount` with `EmailAddress`, `PhoneNumber`, `Money`. Value Objects encapsulate validation (is this email valid?), formatting, and behavior (money addition). They eliminate scattered validation logic and make the domain model self-documenting.

- **Don't use DDD everywhere** — DDD is for complex business domains where the model provides competitive advantage. For simple CRUD screens, reporting dashboards, and generic functionality, DDD adds ceremony without benefit. An insurance team applied full DDD tactical patterns to their email template management module — aggregates, value objects, domain events for a system that was essentially a text editor with variables — doubling development time with zero business value gain while their competitors shipped the same feature in two weeks using a database-backed CRUD approach. Reserve tactical patterns (Entities, Value Objects, Aggregates) for core domains; use simpler approaches for supporting and generic subdomains.

- **Build an Anti-Corruption Layer when integrating with legacy systems** — Without an ACL, the legacy system's bad design choices, confusing terminology, and tangled relationships leak into your new domain model. The ACL is a one-time investment that preserves the integrity of your domain model for the lifetime of the system. A logistics company skipped the ACL and let their new order management system directly reference legacy `CUST_NUM` and `PRD_CD` fields for "speed" — within 6 months, the legacy's inconsistent terminology and null-handling conventions had spread to every new service, making the eventual migration harder than starting from scratch.
- **Invest heavily in the Core Domain, but not in Supporting or Generic subdomains** — The Core Domain is your competitive advantage — build it in-house with the best engineers, apply full DDD tactical patterns, and invest in rich domain models. Supporting subdomains (necessary but not differentiating) can use simpler approaches (CRUD, services). Generic subdomains (logging, email, payments) should use existing solutions (SaaS, open source) without custom domain models.
- **Use Event Storming to discover the domain model collaboratively** — Event Storming brings domain experts and developers together to model the business process using sticky notes. Start with domain events (orange), then add commands (blue), aggregates (yellow), and bounded context boundaries. A single workshop session can reveal the entire domain model in hours rather than weeks of document analysis.
