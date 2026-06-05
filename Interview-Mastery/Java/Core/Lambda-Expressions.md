# Java Lambda Expressions

---

## 1. Executive Summary

### What Is It?
Lambda expressions introduce functional programming constructs to Java — anonymous functions that can be treated as values (passed as arguments, returned from methods, stored in variables).

### Why Does It Exist?
Before lambdas, Java required verbose anonymous inner classes for passing behavior. Lambdas enable:
- **Concise code** — focus on what, not boilerplate
- **Functional programming** — Stream API, Optional, CompletableFuture
- **Behavior parameterization** — pass logic as data
- **Parallel processing** — easier parallelization

### Syntax

```java
// Full syntax
(parameters) -> { body; return value; }

// Single parameter, no parentheses
param -> expression

// No parameters
() -> expression

// Single expression (returns automatically)
(a, b) -> a + b

// Multiple statements need braces and return
(a, b) -> {
    int sum = a + b;
    log.debug("Sum: {}", sum);
    return sum;
}
```

### Where Lambdas Are Used
- **Collections**: `list.sort((a, b) -> a.compareTo(b))`
- **Streams**: `list.stream().filter(x -> x > 5).map(x -> x * 2)`
- **Optional**: `optional.orElseGet(() -> expensiveDefault())`
- **CompletableFuture**: `future.thenApply(result -> transform(result))`
- **Event handlers**: `button.setOnAction(e -> handleClick(e))`
- **Threads**: `new Thread(() -> doWork()).start()`
- **Comparators**: `list.sort(Comparator.comparing(Person::getAge))`
- **Custom functional interfaces**: custom SAM interface implementations

---

## 2. Core Theory

### Lambda vs Anonymous Inner Class

```java
// Anonymous inner class
button.addActionListener(new ActionListener() {
    @Override
    public void actionPerformed(ActionEvent e) {
        System.out.println("Clicked at " + e.getWhen());
    }
});

// Lambda
button.addActionListener(e -> System.out.println("Clicked at " + e.getWhen()));
```

### Type Inference
The compiler infers the target type from context:

```java
// Explicit type
Function<String, Integer> f = (String s) -> Integer.parseInt(s);

// Inferred types (most common)
Function<String, Integer> f = s -> Integer.parseInt(s);

// Mixed: some explicit, some inferred
BiFunction<Integer, Integer, Integer> add = (Integer a, Integer b) -> a + b;
```

### Variable Capture
Lambdas can capture:
- **effectively final** local variables (not reassigned after initialization)
- **static fields** (anytime)
- **instance fields** (anytime — via captured `this`)

```java
public class Example {
    private int instanceField = 42;
    
    public Supplier<Integer> createSupplier() {
        int localVar = 10;
        // localVar = 20; // Would cause compile error — not effectively final
        
        return () -> instanceField + localVar;
        // Captures: this (for instanceField), localVar (copy)
    }
}
```

### Scope
Lambdas don't introduce a new scope — `this` refers to the enclosing class:

```java
public class ScopeExample {
    private String value = "class";
    
    public void test() {
        String value = "local";
        
        Runnable r = () -> {
            System.out.println(this.value); // "class" — refers to instance field
            System.out.println(value);      // "local" — captured local variable
            // String value = "";           // Compile error — variable already defined in scope
        };
    }
}
```

---

## 3. Under-the-Hood Deep Dive

### invokedynamic (indy)
JVM instruction that enables efficient lambda compilation:

```
Source: Supplier<String> s = () -> "hello";

javac generates:
invokedynamic #bootstrap, args:[]
    
Bootstrap method (LambdaMetafactory):
- Creates CallSite at link time
- Generates implementation class on-the-fly
- Links method handle to the lambda body
- Subsequent calls invoke linked handle directly (no reflection)
```

### Memory Characteristics

| Lambda Type | Instance | Cached | Captures |
|-------------|----------|--------|----------|
| Non-capturing | Single static instance | Yes (forever) | Nothing |
| Capturing (local vars) | New per call | No | Copies of variables |
| Capturing (this) | New per call | No | `this` reference |
| Method reference (static) | Single static | Yes | Nothing |
| Method reference (instance) | New per call | No | Instance reference |

### Performance Class Hierarchy
Lambda (non-capturing) ≈ method reference ≈ static call > anonymous class > lambda (capturing) ≈ inner class

---

## 4. Production Code Examples

### 4.1 Bad vs Good Lambda Usage

```java
// BAD: Lambda with complex body (needs extraction)
List<Order> processed = orders.stream()
    .filter(o -> {
        if (o.getStatus() == OrderStatus.PENDING) {
            BigDecimal discount = calculateDiscount(o.getCustomer().getTier());
            o.setDiscountedTotal(o.getTotal().multiply(discount));
            if (o.getDiscountedTotal().compareTo(MINIMUM_ORDER) < 0) {
                return false;
            }
            return true;
        }
        return false;
    })
    .collect(Collectors.toList());

// GOOD: Extract method reference
List<Order> processed = orders.stream()
    .filter(OrderFilters::isEligibleForProcessing)
    .map(OrderCalculations::applyDiscount)
    .collect(Collectors.toList());
```

### 4.2 Lambda with Builder Pattern

```java
// Functional builder
public class EmailBuilder {
    private String to;
    private String subject;
    private String body;
    
    public EmailBuilder with(Consumer<EmailBuilder> builder) {
        builder.accept(this);
        return this;
    }
    
    public Email build() { return new Email(to, subject, body); }
    
    // Setters return this for chaining
    public EmailBuilder to(String to) { this.to = to; return this; }
    public EmailBuilder subject(String subject) { this.subject = subject; return this; }
    public EmailBuilder body(String body) { this.body = body; return this; }
}

// Usage
Email email = new EmailBuilder()
    .with(b -> b.to("user@example.com"))
    .with(b -> b.subject("Hello"))
    .with(b -> b.body("Message body"))
    .build();
```

### 4.3 Lambda with Resource Management

```java
public class TransactionManager {
    public <T> T executeInTransaction(Supplier<T> operation) {
        beginTransaction();
        try {
            T result = operation.get();
            commit();
            return result;
        } catch (Exception e) {
            rollback();
            throw new TransactionException("Transaction failed", e);
        }
    }
}

// Usage
Order order = transactionManager.executeInTransaction(() -> {
    Order saved = orderRepository.save(newOrder);
    inventoryService.deduct(saved.getItems());
    return saved;
});
```

---

## 5-16. Summary

### Common Mistakes
1. **Too complex lambda bodies** — extract into methods
2. **Mutable captures** — not effectively final → compile error
3. **Serialization** — default lambdas not serializable
4. **Checked exceptions** — manual try-catch needed
5. **Ambiguous overloads** — cast to disambiguate
6. **this reference in inner classes vs lambdas** — different semantics
7. **Performance assumptions** — capturing lambdas allocate each invocation
8. **Debugging** — lambda stack traces are less readable
9. **Parallel side effects** — captured mutable state causes race conditions
10. **Overusing** — sometimes a for-each loop is clearer

### Cheat Sheet

```
═══ LAMBDA EXPRESSIONS ═══════════════════════════════════════

┌─ SYNTAX ───────────────────────────────────────────────────┐
│ (params) -> expression          // Single expression        │
│ (params) -> { statements }      // Block body               │
│ param -> expression             // Single param, no parens  │
│ () -> expression                // No params                │
└─────────────────────────────────────────────────────────────┘

┌─ CAPTURE RULES ────────────────────────────────────────────┐
│ ✅ Effectively final locals    ✅ Static fields             │
│ ✅ Instance fields (via this)  ❌ Mutable locals            │
└─────────────────────────────────────────────────────────────┘

┌─ METHOD REFERENCES ────────────────────────────────────────┐
│ Class::staticMethod     → Math::max        (a, b) → M.max  │
│ instance::method        → System.out::println → S.o.println │
│ Class::instanceMethod   → String::length    s → s.length()  │
│ Class::new              → ArrayList::new   () → new AL()   │
└─────────────────────────────────────────────────────────────┘

┌─ RULES ────────────────────────────────────────────────────┐
│ • Keep lambda bodies to 1-3 lines                          │
│ • Extract complex logic to methods                         │
│ • Prefer method references when possible                   │
│ • Non-capturing → cached (best performance)                │
│ • Handle checked exceptions inside lambda                  │
└─────────────────────────────────────────────────────────────┘
```
