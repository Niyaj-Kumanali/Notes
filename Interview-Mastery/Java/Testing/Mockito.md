# Mockito

---

## Overview

- **Definition:** Mockito is the most popular mocking framework for Java unit tests. It creates mock objects to isolate the class under test from its dependencies.

- **Why Mockito?**
  - Clean, readable syntax with `when()`/`thenReturn()` and `verify()`
  - Support for spies (partial mocks)
  - Argument matchers for flexible stubbing
  - Integration with JUnit 5 via `@ExtendWith(MockitoExtension.class)`
  - Spring Boot integration via `@MockBean`

---

## Creating Mocks

```java
// Field annotation (requires MockitoExtension or MockitoAnnotations.openMocks())
@Mock
private OrderRepository orderRepository;

@InjectMocks  // Creates instance + injects mocks
private OrderService orderService;

// Programmatic (no annotation needed)
OrderRepository repo = mock(OrderRepository.class);
OrderService service = new OrderService(repo);
```

- **`@InjectMocks`** tries constructor injection first, then setter, then field. Constructor injection is preferred.

---

## Stubbing

- **Definition:** Defining what a mock should return when a specific method is called.

```java
// Basic stubbing
when(orderRepository.findById(1L)).thenReturn(Optional.of(order));
when(orderRepository.findById(99L)).thenReturn(Optional.empty());

// Void methods
doNothing().when(orderRepository).delete(any());

// Exception stubbing
when(orderRepository.save(any())).thenThrow(
    new DataIntegrityViolationException("Constraint violation"));

// Multiple calls — different values each time
when(orderRepository.findById(1L))
    .thenReturn(Optional.of(order1))
    .thenReturn(Optional.of(order2));

// Answer — dynamic response based on invocation
when(orderRepository.save(any())).thenAnswer(invocation -> {
    Order order = invocation.getArgument(0);
    order.setId(1L);
    return order;
});
```

---

## Verification

- **Definition:** Checking that specific interactions happened with the mock.

```java
// Verify method was called
verify(orderRepository).save(order);
verify(orderRepository, times(1)).save(order);
verify(orderRepository, never()).delete(any());
verify(orderRepository, atLeastOnce()).findById(any());
verify(orderRepository, atMost(3)).findAll();

// Verify order of calls
InOrder inOrder = inOrder(orderRepository, emailService);
inOrder.verify(orderRepository).save(order);
inOrder.verify(emailService).sendOrderConfirmation(order);

// Verify with timeout (async tests)
verify(orderRepository, timeout(1000)).save(any());

// No interactions at all
verifyNoInteractions(orderRepository);

// No more unverified interactions
verifyNoMoreInteractions(orderRepository);
```

---

## Argument Matchers

- **Definition:** Flexible matching of method arguments for stubbing and verification.

```java
// Any value
when(repo.findById(anyLong())).thenReturn(Optional.of(order));
when(repo.save(any(Order.class))).thenReturn(order);

// Specific matchers
when(repo.findByStatusAndDate(
    eq(OrderStatus.ACTIVE),
    any(LocalDateTime.class))
).thenReturn(List.of(order));

// Custom matcher
when(repo.save(argThat(o -> o.getTotal() > 100))).thenReturn(order);
```

- **Important:** If you use matchers for any argument in a method, you must use matchers for ALL arguments. Mixing literal values and matchers causes `InvalidUseOfMatchersException`.

---

## Spying

- **Definition:** Creating a partial mock — real object with some methods stubbed.

```java
Order realOrder = new Order(100, "PENDING");
Order spyOrder = spy(realOrder);

when(spyOrder.getStatus()).thenReturn("SHIPPED");  // Override real method
// Non-stubbed methods call the real implementation

// Alternative with annotation
@Spy
private OrderService orderService;
```

- **Use sparingly:** Spies indicate a design issue — prefer mocks over spies. Spies use the real object, so tests can have side effects.

---

## BDD Style

- **Definition:** Given/When/Then syntax using `BDDMockito` for behavior-driven development style.

```java
import static org.mockito.BDDMockito.*;

// Given
given(orderRepository.findById(1L)).willReturn(Optional.of(order));

// When
Order result = orderService.findById(1L);

// Then
then(orderRepository).should().findById(1L);
then(orderRepository).should(never()).save(any());
```

---

## Mockito with Spring Boot

```java
@SpringBootTest
@AutoConfigureMockMvc
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean  // Replaces the real bean with a mock in the Spring context
    private OrderService orderService;

    @Test
    void shouldReturnOrder() throws Exception {
        Order order = new Order(1L, "test@test.com", 150.00);
        when(orderService.findById(1L)).thenReturn(Optional.of(order));

        mockMvc.perform(get("/api/v1/orders/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1));
    }
}
```

- **`@MockBean` vs `@Mock`:** `@MockBean` (Spring Boot test) replaces the bean in the ApplicationContext. `@Mock` (JUnit) creates a plain mock without Spring context.

---

## Common Mistakes

- **Mocking the class under test** — unit tests should test real logic, not mock it. This *looks correct* because the mock compiles and the test passes — but it tests Mockito's default return values, not the actual class behavior. Always test real logic, not mock it.
- **Not using `@InjectMocks` properly** — constructor injection is most reliable; field/setter injection is fragile. This *looks correct* because the test runs without errors — `@InjectMocks` silently skips mismatched dependencies, leaving fields null until the tested code path triggers an NPE. Constructor injection is most reliable; field/setter injection is fragile.
- **Over-stubbing** — only stub what's needed for the specific test scenario. This *looks correct* because more stubbing makes the test feel more thorough — but unnecessary stubs make tests brittle, breaking when production code legitimately changes. Only stub what's needed for the specific test scenario.
- **Mixing matchers and literals** — must use matchers for ALL args if using matchers for ANY arg. This *looks correct* because the code looks reasonable at a glance — the `InvalidUseOfMatchersException` at runtime is confusing, and the developer may not connect it to the mixed usage pattern.
- **Not verifying interactions** — verify that the expected interactions actually occurred, especially for void methods. This *looks correct* because the test checks the return value, which is sufficient to make the assertion pass — the missing side effect (e.g., an email that wasn't sent) is invisible in the test result.
- **Using `@Spy` when `@Mock` would do** — spies should be rare. This *looks correct* because `@Spy` works and the test passes — the design smell (too many responsibilities) is invisible when tests are green. Always prefer extracting the stubbed method into its own class.
- **Stubbing `equals()`/`hashCode()`** — don't stub these. This *looks correct* because the IDE autocomplete suggests `equals()` and stubbing it compiles — the developer doesn't realize that Mockito uses these internally for argument matching. Use `refEq()` or `argThat()` for comparison.
- **Ignoring `verifyNoMoreInteractions()`** — helps detect unexpected calls that may hide bugs. This *looks correct* because the test passes without it, and adding the check feels like over-specification — the extra method call only matters if it has a real side effect that changes behavior.

---

## Real-World Scenarios

### Scenario 1: Testing a Service with External Payment Gateway

A checkout service calls a payment gateway, then updates the order, then sends a confirmation email. The service must handle success, declined card, and network timeout from the gateway. Each path has different side effects.

```java
@ExtendWith(MockitoExtension.class)
class CheckoutServiceTest {
    @Mock private PaymentGateway paymentGateway;
    @Mock private OrderRepository orderRepository;
    @Mock private EmailService emailService;
    @InjectMocks private CheckoutService checkoutService;

    @Test
    void shouldCompleteCheckoutOnSuccessfulPayment() {
        Order order = new Order("test@test.com", 100.00);
        when(paymentGateway.charge(100.00)).thenReturn(new ChargeResult(true, "auth_123"));
        when(orderRepository.save(any())).thenAnswer(inv -> { inv.getArgument(0, Order.class).setId(1L); return inv.getArgument(0); });

        Order result = checkoutService.checkout(order);

        assertAll(
            () -> assertEquals(OrderStatus.CONFIRMED, result.getStatus()),
            () -> assertEquals("auth_123", result.getAuthCode()),
            () -> verify(paymentGateway).charge(100.00),
            () -> verify(orderRepository).save(order),
            () -> verify(emailService).sendConfirmation("test@test.com")
        );
    }

    @Test
    void shouldMarkOrderAsFailedOnDeclinedCard() {
        Order order = new Order("test@test.com", 100.00);
        when(paymentGateway.charge(100.00)).thenThrow(new CardDeclinedException("insufficient funds"));

        assertThrows(CardDeclinedException.class, () -> checkoutService.checkout(order));

        assertAll(
            () -> verify(orderRepository).save(argThat(o -> o.getStatus() == OrderStatus.FAILED)),
            () -> verify(emailService, never()).sendConfirmation(any())
        );
    }
}
```

Each test method covers one path through the service. `assertAll` groups all assertions for that path — you immediately know if the state, auth code, or side effects are wrong. `verify(..., never())` explicitly checks that side effects don't happen on failure paths. `thenAnswer` simulates the database ID generation. The pattern: one test per user-visible outcome, asserting both return value and side effects.

### Scenario 2: Testing a REST Controller with Spring MockMvc

A REST API endpoint `POST /api/orders` validates input, delegates to a service, and returns the created resource. The controller has error handling for validation errors, service exceptions, and unexpected errors.

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired private MockMvc mockMvc;
    @MockBean private OrderService orderService;

    @Test
    void shouldReturn201OnValidOrder() throws Exception {
        when(orderService.createOrder(any())).thenReturn(new Order(1L, "test@test.com", 100.00));

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"email\":\"test@test.com\",\"total\":100.00}"))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.email").value("test@test.com"));
    }

    @Test
    void shouldReturn400OnInvalidEmail() throws Exception {
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"email\":\"invalid\",\"total\":100.00}"))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors[0].field").value("email"));
    }

    @Test
    void shouldReturn503OnServiceTimeout() throws Exception {
        when(orderService.createOrder(any())).thenThrow(new ServiceUnavailableException("DB timeout"));

        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"email\":\"test@test.com\",\"total\":100.00}"))
            .andExpect(status().isServiceUnavailable())
            .andExpect(jsonPath("$.error").value("service_unavailable"));
    }
}
```

`@MockBean` replaces `OrderService` in the Spring context with a mock. Each test covers one HTTP response: 201 (success), 400 (validation error — no service call needed), 503 (service failure — still returns structured JSON). `@WebMvcTest` only loads the web layer (no full Spring context), making tests fast (~100ms each). The controller's error handling is tested without starting the full application.

### Scenario 3: Testing Asynchronous Event Processing

A service publishes domain events after processing an order. An async listener sends emails and updates analytics. The test must verify these side effects happen asynchronously without arbitrary `Thread.sleep()`.

```java
@ExtendWith(MockitoExtension.class)
class OrderEventTest {
    @Mock private EmailService emailService;
    @Mock private AnalyticsService analyticsService;
    @InjectMocks private OrderEventListener eventListener;

    @Test
    void shouldProcessEventAsynchronously() {
        OrderCreatedEvent event = new OrderCreatedEvent("order-1", "test@test.com");

        eventListener.onOrderCreated(event);

        verify(emailService, timeout(2000)).sendConfirmation("order-1", "test@test.com");
        verify(analyticsService, timeout(2000)).trackEvent("order.created", "order-1");
    }

    @Test
    void shouldNotSendEmailOnFailedEvent() {
        doThrow(new RuntimeException("Analytics down"))
            .when(analyticsService).trackEvent(any(), any());

        doThrow(new AnalyticsException("Service unavailable"))
            .when(analyticsService).trackEvent("order.created", "order-1");

        assertThrows(AnalyticsException.class,
            () -> eventListener.onOrderCreated(new OrderCreatedEvent("order-1", "test@test.com")));

        verify(emailService, timeout(1000).times(0)).sendConfirmation(any(), any());
    }
}
```

Mockito's `timeout()` verifier polls the mock until the verification passes or the timeout expires. This is more reliable than `Thread.sleep()` which either waits too long (slow tests) or not long enough (flaky tests). The `timeout(2000)` means "wait up to 2 seconds, but return as soon as the method is called." For negative verification, `timeout(1000).times(0)` waits up to 1 second to confirm the method was NEVER called.

---

## Scenario-Based Questions

1. **Q: You have a service with 6 injected dependencies. `@InjectMocks` creates the instance but silently leaves some dependencies null because constructor parameter types don't match the mock types. The test passes until a code path exercises the null dependency, causing NPE. How do you make injection failures visible immediately?**
   A: Switch from `@InjectMocks` to explicit constructor injection — it fails at compile time if the constructor changes:
   ```java
   // Brittle — silent null injection
   @Mock private PaymentGateway paymentGateway;
   @Mock private EmailService emailService;
   @Mock private OrderRepository orderRepository;
   @InjectMocks private OrderService orderService; // May silently leave null

   // Robust — compile-time safety
   @BeforeEach
   void setUp() {
       orderService = new OrderService(paymentGateway, emailService, orderRepository);
   }
   ```
   Explicit construction fails at compile time when the constructor signature changes (adding/removing/reordering parameters). `@InjectMocks` fails at runtime — it tries constructor, setter, then field injection, and silently skips mismatches. The constructor is already a compile-time contract; the test constructor call should also be compile-time checked. Only use `@InjectMocks` when the constructor has many parameters (6+) and the test is stable.

2. **Q: A test mocks `orderRepository.findById(1L)` to return an order. Another developer changes the production code to call `findById(1L, LOCK)` (adding a lock mode parameter). The test still passes because the old stub is not called and `MockitoExtension` doesn't flag unused stubs by default. How do you make the test fail when production code changes the method signature or arguments?**
   A: Enable strict stubbing with `@MockitoExtension` which reports unused stubs:
   ```java
   @ExtendWith(MockitoExtension.class) // strict by default
   class OrderServiceTest {
       @Mock OrderRepository orderRepository;
       @InjectMocks OrderService orderService;

       @Test
       void shouldReturnOrder() {
           when(orderRepository.findById(1L)).thenReturn(Optional.of(order));
           // If production code changes to findById(1L, LOCK), this stub is never called
           // MockitoExtension throws UnnecessaryStubbingException
       }
   }
   ```
   `@MockitoExtension` enables strict stubbing by default — any stub that's never used during the test causes `UnnecessaryStubbingException`. This catches: (1) renamed methods, (2) changed parameters, (3) conditions that no longer trigger the stubbed call. If stubbing is shared across tests (common setup), use `lenient().when(...)` or `@MockitoSettings(strictness = LENIENT)` on the setup method.

3. **Q: You have a legacy class with 10 `@Autowired` fields and no constructor injection. You want to write a unit test for it. `@InjectMocks` doesn't reliably inject 10 dependencies. Manual `setField()` calls are fragile. How do you test this legacy code without refactoring it?**
   A: Use Mockito's `ReflectionTestUtils` (from Spring) or plain reflection to set private fields:

   > **Interview follow-up:** The candidate used `ReflectionTestUtils.setField()` to test 10 `@Autowired` fields. After 3 sprints of adding features, the class now has 15 `@Autowired` fields. The `setField()` calls in the test take up 30 lines, and every time a new field is added, the test silently uses a null dependency until a code path exercises it. How would you use this test as leverage to refactor the production class toward constructor injection incrementally, without rewriting the entire class at once?
   ```java
   // Legacy class with field injection
   public class LegacyService {
       @Autowired private PaymentGateway paymentGateway;
       @Autowired private InventoryService inventoryService;
       // ... 8 more fields
   }

   @Test
   void testLegacyService() {
       LegacyService service = new LegacyService();
       ReflectionTestUtils.setField(service, "paymentGateway", mockPaymentGateway);
       ReflectionTestUtils.setField(service, "inventoryService", mockInventoryService);
       // ... set remaining 8 fields
   }
   ```
   This is a testing workaround, not a design endorsement. The goal is to get the legacy code under test so you can refactor it safely. Once tested, migrate to constructor injection: `@InjectMocks` works reliably with constructor injection (explicit parameter matching vs field name matching). Each field you convert to constructor injection removes one `ReflectionTestUtils.setField()` call.

4. **Q: A test verifies that `emailService.send(any(Email.class))` is called. The test passes. In production, `send()` throws a `MailSendException` which the service catches and logs. The mock doesn't throw, so the exception handling code is never tested. How do you test both the happy path and the exception handling path?**
   A: Test the exception handling in a separate test method:
   ```java
   @Test
   void shouldSendEmail() {
       orderService.completeOrder(order);
       verify(emailService).send(any(Email.class));
   }

   @Test
   void shouldHandleEmailFailure() {
       doThrow(new MailSendException("SMTP timeout"))
           .when(emailService).send(any(Email.class));

       // Should log error but not fail the transaction
       assertDoesNotThrow(() -> orderService.completeOrder(order));

       verify(orderRepository).save(argThat(o -> o.getStatus() == CONFIRMED)); // order still confirmed
       verify(logService).logError(contains("Email failed"));
   }
   ```
   Each code path gets its own test method: the first tests that the email is called; the second tests the exception handler. The mock throws the real exception type, exercising the exact catch block that runs in production. The assertions verify the fallback behavior (order is still confirmed) and the error logging.

5. **Q: A service calls `restTemplate.getForObject(url, Order.class)` which makes an HTTP call. You mock it with `when(restTemplate.getForObject(anyString(), eq(Order.class))).thenReturn(order)`. The test passes locally. In CI, tests fail because the URL contains environment-specific values (ports, hostnames). How do you mock URL-dependent calls robustly?**
   A: Don't match the exact URL — match by the order ID or path pattern:
   ```java
   // Brittle — matches exact URL with port
   when(restTemplate.getForObject("http://localhost:8080/api/orders/1", Order.class))
       .thenReturn(order);

   // Robust — match by path pattern using argThat
   when(restTemplate.getForObject(argThat(url -> url.contains("/api/orders/1")), eq(Order.class)))
       .thenReturn(order);

   // Or separate URL building from HTTP call — makes the whole thing testable
   @Test
   void shouldBuildCorrectUrl() {
       String url = orderService.buildOrderUrl(1L);
       assertEquals("http://orderservice/api/orders/1", url);
   }
   ```
   For `RestTemplate`, better to wrap it in a `WebClient` or use `MockRestServiceServer` (Spring's HTTP mock server) which matches by request matchers:
   ```java
   MockRestServiceServer mockServer = MockRestServiceServer.bindTo(restTemplate).build();
   mockServer.expect(requestTo(contains("/api/orders/1")))
       .andRespond(withSuccess("{\"id\":1}", MediaType.APPLICATION_JSON));
   ```
   `MockRestServiceServer` validates the HTTP method, headers, query parameters, and body — not just the URL. It also verifies that exactly the expected calls were made (no extra, no missing).

6. **Q: You have 50 test classes, each with 5-10 test methods. Every test class mocks a `UserRepository` and sets up a default user in `@BeforeEach`. When the `UserRepository` interface changes (e.g., adding a new method), all 50 test classes' setup needs updating. How do you reduce this maintenance burden?**
   A: Create a test fixture factory — a reusable builder for common mocks:
   ```java
   public class TestFixture {
       public static OrderServiceFixture createOrderServiceFixture() {
           OrderRepository orderRepo = mock(OrderRepository.class);
           PaymentGateway paymentGateway = mock(PaymentGateway.class);
           EmailService emailService = mock(EmailService.class);
           OrderService orderService = new OrderService(orderRepo, paymentGateway, emailService);

           // Default stubbings
           when(orderRepo.findById(any())).thenReturn(Optional.of(new Order(1L, "test@test.com")));
           when(paymentGateway.charge(any())).thenReturn(new ChargeResult(true, "auth_123"));

           return new OrderServiceFixture(orderService, orderRepo, paymentGateway, emailService);
       }
   }

   record OrderServiceFixture(OrderService service, OrderRepository repo, PaymentGateway payment, EmailService email) {}

   class CheckoutTest {
       private OrderServiceFixture fixture;

       @BeforeEach
       void setUp() { fixture = TestFixture.createOrderServiceFixture(); }

       @Test
       void shouldCheckout() {
           fixture.service().checkout(new Order("test@test.com", 100.00));
           verify(fixture.repo()).save(any());
       }
   }
   ```
   When `UserRepository` changes, update the fixture in one place. Each test overrides specific stubbings for its scenario: `when(fixture.repo().findById(any())).thenReturn(Optional.empty())`. The tuple return avoids the `@InjectMocks` fragility. This pattern reduces mock boilerplate by 80% and makes test maintenance proportional to production changes, not test count.

7. **Q: A test uses `@MockBean` to mock a Redis cache in a Spring Boot test. The test starts the full Spring context with embedded Redis (Testcontainers). The `@MockBean` doesn't replace the Redis ConnectionFactory bean because it's created by auto-configuration. The test connects to the real Redis instead of the mock. How do you mock infrastructure beans in Spring Boot tests?**
   A: `@MockBean` replaces beans in the context — but auto-configuration may recreate the bean after the mock is created. Use `@MockBean` on the bean type that the application code injects:
   ```java
   // Don't mock the RedisConnectionFactory — mock your own wrapper
   @MockBean
   private CacheService cacheService; // Your application's cache abstraction

   // OR mock at the Spring auto-configuration boundary
   @TestConfiguration
   static class TestConfig {
       @Bean
       @Primary
       RedisConnectionFactory mockConnectionFactory() {
           return mock(RedisConnectionFactory.class);
       }
   }
   ```
   The issue is that `@MockBean` replaces a bean AFTER auto-configuration has already created the real one. For Spring Boot infrastructure beans, the safest approach is to mock at your application's abstraction boundary (your CacheService class) rather than the Spring infrastructure bean (RedisConnectionFactory). This also makes tests faster (no Redis container needed). For true integration tests, use Testcontainers with `@DynamicPropertySource`.

8. **Q: A test verifies that `orderService.processOrder(order)` calls `orderRepository.save(order)`. After a refactoring, `processOrder()` now saves the order inside a `@Transactional` method which proxies `OrderService`. The `@InjectMocks` creates a plain (non-proxied) instance, so the test doesn't exercise the transaction behavior. How do you test Spring-proxied behavior?**
   A: Use Spring's `@ExtendWith(SpringExtension.class)` with a minimal context:

   > **Interview follow-up:** The candidate added a Spring integration test specifically for transaction behavior. The team now has 100 unit tests using `@InjectMocks` across 20 service classes. After the team adds `@Transactional` to 5 more methods, 15 of those 100 tests pass despite not testing the actual transaction boundaries. A production incident occurs when a `@Transactional` rollback doesn't work because the catch block swallows the exception. Which of the 100 tests would have caught this bug, and how would you design a policy for deciding when a `@InjectMocks` unit test is sufficient versus when a Spring context test is required?
   ```java
   // Unit test — doesn't test transactional behavior
   @ExtendWith(MockitoExtension.class)
   class OrderServiceTest {
       @Mock OrderRepository orderRepository;
       @InjectMocks OrderService orderService;
   }

   // Integration test — tests transactional behavior
   @ExtendWith(SpringExtension.class)
   @ContextConfiguration(classes = TestConfig.class)
   class OrderServiceIntegrationTest {
       @Autowired
       private OrderService orderService; // Spring proxy with @Transactional

       @Test
       void shouldRollbackOnFailure() {
           assertThrows(RuntimeException.class, () -> orderService.processOrder(order));
           assertTrue(countRowsInTable(jdbcTemplate, "orders") == 0); // rolled back
       }
   }
   ```
   Use `@ContextConfiguration` to create a minimal Spring context with the service and its dependencies (real or mocked). The `@Autowired OrderService` is a Spring proxy that applies `@Transactional`. For most tests, `@MockitoExtension` is sufficient (transactional behavior is tested separately). Only add the Spring context when you specifically need to verify AOP behavior (transactions, caching, security).

9. **Q: A test uses `@Spy` on a real `OrderService`, stubs one method, and calls the real implementation for others. The test becomes fragile — changes to the real implementation break the test even though the stubbed method behavior hasn't changed. How do you restrict spies to minimal usage?**
   A: Spies should only be used for legacy code where extracting an interface is too expensive:
   ```java
   // Fragile — spies couple test to real implementation details
   @Spy
   private OrderService orderService;

   @Test
   void shouldApplyDiscount() {
       doReturn(BigDecimal.valueOf(10)).when(orderService).calculateDiscount(any());
       // rest uses real orderService methods
   }

   // Better — extract the discount logic into its own class
   @Mock
   private DiscountCalculator discountCalculator;

   @InjectMocks
   private OrderService orderService;

   @Test
   void shouldApplyDiscount() {
       when(discountCalculator.calculate(any())).thenReturn(BigDecimal.valueOf(10));
       // OrderService uses the real injected DiscountCalculator mock
   }
   ```
   Spies indicate a design smell: the class has too many responsibilities. Extract the stubbed behavior into a separate, injectable class. Then mock the extracted dependency instead of spying on the whole class. The rule: if you need to stub a method on a spy, that method should probably be in its own class. Spies are acceptable for testing legacy code during refactoring but should be removed as part of the refactoring.

10. **Q: A test creates a mock with `Mockito.mock(List.class)`. Calling `list.add("item")` doesn't throw but also doesn't actually add the item. Calling `list.get(0)` returns `null`. Developers are confused why the mock List doesn't behave like a List. How do you prevent mock behavior confusion in the team?**
    A: Mockito's mock of `List` is a complete shell — no methods work unless stubbed. Developers should mock interfaces, not concrete collection classes:
    ```java
    // Confusing — mocks List, no real list behavior
    List<String> mockList = mock(List.class);
    mockList.add("item");
    assertTrue(mockList.isEmpty()); // true! add() is a no-op

    // Correct — use a real list or a spy
    List<String> realList = new ArrayList<>();
    realList.add("item");
    assertFalse(realList.isEmpty()); // true

    // If you must mock a collection for specific behavior:
    List<String> stubbedList = mock(List.class);
    when(stubbedList.get(0)).thenReturn("item");
    ```
    The key rule: mock types, not implementations. Don't mock `ArrayList`, `HashMap`, or `StringBuilder` — use the real thing. If you need a collection with specific data, create a real collection with test data. Mocking collection classes confuses because the mock has none of the real behavior, and it makes tests harder to read and maintain. Use `List.of("item1", "item2")` instead.

---

## Interview Questions

1. **What is Mockito and why is it used?**
   A: Mockito is the most popular Java mocking framework for unit testing. It creates mock objects to isolate the class under test from its dependencies. Key features: stubbing (`when()`/`thenReturn()`), verification (`verify()`), argument matchers (`any()`, `eq()`), spies for partial mocking, and integration with JUnit 5 via `@ExtendWith(MockitoExtension.class)`.

2. **What is the difference between `@Mock` and `@InjectMocks`?**
   A: `@Mock` creates a mock instance of a dependency. `@InjectMocks` creates an instance of the class under test and injects the mocks into it. `@InjectMocks` tries constructor injection first, then setter, then field injection. Constructor injection is preferred because it's explicit and fails at compile time if the constructor signature changes.

3. **What is the difference between `when()` and `doThrow()`?**
   A: `when(mock.method()).thenReturn(value)` is used for methods that return a value. `doThrow(exception).when(mock).voidMethod()` is used for void methods — `when()` doesn't compile because `void` cannot be the argument to `when()`. `doThrow()` syntax is also useful for stubbing methods on spies (to avoid calling the real method during stubbing setup).

4. **What are argument matchers and what is the `any()` vs `eq()` rule?**
   A: Argument matchers (`any()`, `anyString()`, `eq(value)`, `argThat(predicate)`) provide flexible matching of method arguments. If you use a matcher for one argument, you MUST use matchers for ALL arguments in that method call — mixing literals and matchers throws `InvalidUseOfMatchersException`. Use `eq(value)` to wrap a specific value when other arguments use matchers.

5. **What is the difference between `verify()` and `then()` (BDDMockito)?**
   A: `verify(mock).method()` is the standard Mockito syntax for checking interactions. `then(mock).should().method()` is the BDDMockito equivalent (Given/When/Then style). BDDMockito uses `given()` instead of `when()` for stubbing and `then().should()` instead of `verify()`. Both are functionally identical — choose based on team preference for BDD vs traditional style.

6. **What is `ArgumentCaptor` and when would you use it?**
   A: `ArgumentCaptor<T>` captures method arguments during verification for detailed assertions. Use it when you need to assert properties of the argument that can't be expressed with matchers alone:
   ```java
   @Captor ArgumentCaptor<Email> emailCaptor;
   verify(emailService).send(emailCaptor.capture());
   assertEquals("user@test.com", emailCaptor.getValue().getTo());
   assertTrue(emailCaptor.getValue().getBody().contains("order-123"));
   ```
   Without `ArgumentCaptor`, you'd need custom `argThat()` matchers. For multiple calls, use `emailCaptor.getAllValues()`.

7. **What is the difference between `@Mock` and `@MockBean`?**
   A: `@Mock` (Mockito) creates a plain mock — no Spring context involved. `@MockBean` (Spring Boot test) replaces a bean in the Spring `ApplicationContext` with a mock. Use `@Mock` in unit tests with `@ExtendWith(MockitoExtension.class)`. Use `@MockBean` in Spring Boot tests (`@SpringBootTest`, `@WebMvcTest`) when you need a mock in the Spring context. `@MockBean` is slower (starts Spring context) but enables testing Spring-specific features.

8. **What is a spy (`@Spy`) and when should you use it?**
   A: A spy is a partial mock — a real object where some methods are stubbed and the rest call the real implementation. Use sparingly — spies indicate a design issue (class has too many responsibilities). Acceptable uses: testing legacy code during refactoring, calling real implementations for most methods while stubbing expensive or side-effecting ones. Prefer extracting the stubbed behavior into a separate injectable class.

9. **What is `lenient()` stubbing?**
   A: `lenient().when(mock.method()).thenReturn(value)` tells Mockito to skip the strict stubbing check. Without it, Mockito's strict mode (default with `@MockitoExtension`) throws `UnnecessaryStubbingException` if a stub is never used during the test. Use `lenient()` when: (1) stubbing is defined in shared setup but not used in every test, (2) the stub is for defensive code paths that are hard to trigger.

10. **What is the difference between `RETURNS_DEFAULTS` and `RETURNS_DEEP_STUBS`?**
    A: `RETURNS_DEFAULTS` (default) returns default values for unstubbed methods — null for objects, 0 for numbers, false for booleans, empty collections. `RETURNS_DEEP_STUBS` automatically creates mocks for chained calls: `when(service.doA().doB().doC()).thenReturn(result)` creates nested mocks for `doA()` and `doB()`. Deep stubs make tests brittle (any chain change requires updating stubs) and are harder to debug. Prefer refactoring the fluent API over deep stubs.

---

## Developer Recommendations

- **Use constructor injection over field injection** — Constructor injection makes dependencies explicit, enables immutability (`final` fields), and fails at compile time if a required bean is missing. Field injection hides dependencies and fails at runtime with a `NullPointerException`. In tests, constructor injection with explicit `new Service(a, b, c)` gives compile-time safety against constructor signature changes, unlike `@InjectMocks`.

- **Use `@MockitoExtension` for strict stubbing** — By default, `@MockitoExtension` reports unused stubs as `UnnecessaryStubbingException`. This catches: renamed methods, changed parameters, and conditions that no longer trigger the stubbed call. Without it, tests can pass even though production code has changed and the mock is never exercised. Only use `lenient()` when stubbing shared setup across tests.

- **Stub specifically, verify broadly** — Stub only what the test scenario needs (specific values over `any()` matchers). Verify that the expected interactions happened, but avoid `verifyNoMoreInteractions()` which over-specifies and makes tests brittle to legitimate future changes. Each test should stub the minimum necessary and verify the essential outcomes.

- **Use `thenAnswer()` over `thenReturn()` for dynamic responses** — `thenReturn(order)` always returns the same instance. `thenAnswer(invocation -> { Order o = invocation.getArgument(0); o.setId(1L); return o; })` simulates real behavior like ID generation. This catches bugs where the code modifies the returned object and expects the changes to persist.

- **Prefer `assertThrows` with `verify(never())` over empty catch blocks** — When testing error paths, don't use try-catch with `fail()`. Use `assertThrows(ExType.class, () -> code())` and then verify that side effects did NOT happen: `verify(mock, never()).method()`. This makes the error path test explicit and ensures no hidden behavior on failure.

- **Use `@Captor` over custom `argThat()` matchers** — `@Captor ArgumentCaptor<Order> captor` captures the actual argument passed to a mock method for detailed assertions. This is cleaner than writing a custom `argThat(o -> o.getTotal() > 100)` matcher which produces poor failure messages. Use `captor.getValue()` for single calls and `captor.getAllValues()` for multiple calls.

- **Mock types, not implementations** — Don't mock `ArrayList`, `HashMap`, or `StringBuilder`. Use real instances: `List.of("a", "b")` instead of `mock(List.class)`. Mocking concrete collection classes creates confusing behavior (no real list semantics) and makes tests harder to read. The mock boundary should be at your application's abstraction layer, not at the JDK collection level.
