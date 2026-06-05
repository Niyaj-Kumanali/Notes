# Mockito

---

## 1. Executive Summary

Mockito is the most popular mocking framework for Java unit tests. It creates mock objects to isolate the class under test from its dependencies.

---

## 2. Core Theory

### Creating Mocks

```java
// Field annotation
@Mock
private OrderRepository orderRepository;

@InjectMocks  // Creates instance + injects mocks
private OrderService orderService;

// Programmatic
OrderRepository repo = mock(OrderRepository.class);
OrderService service = new OrderService(repo);
```

### Stubbing

```java
// Basic stubbing
when(orderRepository.findById(1L)).thenReturn(Optional.of(order));
when(orderRepository.findById(99L)).thenReturn(Optional.empty());

// Void methods
doNothing().when(orderRepository).delete(any());

// Exception
when(orderRepository.save(any())).thenThrow(new DataIntegrityViolationException(""));

// Multiple calls
when(orderRepository.findById(1L))
    .thenReturn(Optional.of(order1))
    .thenReturn(Optional.of(order2));

// Answer (dynamic response)
when(orderRepository.save(any())).thenAnswer(invocation -> {
    Order order = invocation.getArgument(0);
    order.setId(1L);
    return order;
});
```

### Verification

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
```

### Argument Matchers

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

### Spying

```java
// Spy — real object with some methods mocked
Order realOrder = new Order(100, "PENDING");
Order spyOrder = spy(realOrder);

when(spyOrder.getStatus()).thenReturn("SHIPPED"); // Override real method
// Real methods called by default (not mocked)
```

### BDD Style

```java
// Given
given(orderRepository.findById(1L)).willReturn(Optional.of(order));

// When
Order result = orderService.findById(1L);

// Then
then(orderRepository).should().findById(1L);
then(orderRepository).should(never()).save(any());
```

---

## 3. Production Code

### 3.1 Complete Unit Test

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    
    @Mock
    private OrderRepository orderRepository;
    @Mock
    private InventoryClient inventoryClient;
    @Mock
    private PaymentGateway paymentGateway;
    @Mock
    private EmailService emailService;
    
    @InjectMocks
    private OrderService orderService;
    
    @Test
    void shouldCreateOrderSuccessfully() {
        // Arrange
        CreateOrderRequest request = new CreateOrderRequest(
            "customer@test.com", List.of("SKU-001", "SKU-002"));
        
        when(inventoryClient.checkAvailability(anyList())).thenReturn(true);
        when(paymentGateway.charge(anyString(), any(BigDecimal.class)))
            .thenReturn(new PaymentResult("tx-123", true));
        when(orderRepository.save(any(Order.class))).thenAnswer(invocation -> {
            Order saved = invocation.getArgument(0);
            saved.setId(1L);
            return saved;
        });
        
        // Act
        Order result = orderService.createOrder(request);
        
        // Assert
        assertNotNull(result.getId());
        assertEquals("CONFIRMED", result.getStatus());
        
        // Verify interactions
        verify(orderRepository).save(any(Order.class));
        verify(paymentGateway).charge(anyString(), any(BigDecimal.class));
        verify(emailService).sendOrderConfirmation("customer@test.com", 1L);
    }
    
    @Test
    void shouldThrowExceptionWhenInventoryUnavailable() {
        // Arrange
        when(inventoryClient.checkAvailability(anyList())).thenReturn(false);
        
        // Act & Assert
        assertThrows(OutOfStockException.class, 
            () -> orderService.createOrder(anyRequest()));
        
        // Verify no payment or save happened
        verify(paymentGateway, never()).charge(any(), any());
        verify(orderRepository, never()).save(any());
    }
}
```

### 3.2 Mockito with Spring Boot Test

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
        Order order = new Order(1L, "customer@test.com", 150.00);
        when(orderService.findById(1L)).thenReturn(Optional.of(order));
        
        mockMvc.perform(get("/api/v1/orders/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.email").value("customer@test.com"));
    }
}
```

---

## 4. Cheat Sheet

```
═══ MOCKITO ═══════════════════════════════════════════════════

┌─ ANNOTATIONS ──────────────────────────────────────────────┐
│ @Mock               — create mock                           │
│ @InjectMocks        — create instance + inject mocks        │
│ @Spy                — partial mock                          │
│ @Captor             — capture argument                      │
│ @MockBean           — Spring Boot mock (replaces bean)      │
└─────────────────────────────────────────────────────────────┘

┌─ STUBBING ─────────────────────────────────────────────────┐
│ when(method).thenReturn(value)          // return value     │
│ when(method).thenThrow(exception)       // throw exception  │
│ when(method).thenAnswer(answer)         // dynamic response │
│ doNothing().when(mock).voidMethod()     // void method      │
│ doThrow().when(mock).voidMethod()       // void throw       │
│ doReturn(value).when(mock).method()     // alternative      │
└─────────────────────────────────────────────────────────────┘

┌─ VERIFICATION ─────────────────────────────────────────────┐
│ verify(mock).method()                    // called once     │
│ verify(mock, times(3)).method()          // called 3 times  │
│ verify(mock, never()).method()           // never called    │
│ verify(mock, atLeastOnce()).method()     // at least once   │
│ verify(mock, timeout(1000)).method()     // async timeout   │
│ verifyNoInteractions(mock)               // no interactions │
│ verifyNoMoreInteractions(mock)           // no extra calls  │
└─────────────────────────────────────────────────────────────┘

┌─ RULES ────────────────────────────────────────────────────┐
│ • Mock dependencies, not the class under test               │
│ • Stub only what's needed for the test scenario             │
│ • Verify interactions for side-effect methods               │
│ • Use BDDMockito (given/willReturn) for BDD style          │
│ • Don't mock value objects (use real instances)             │
│ • Use @InjectMocks carefully — constructor injection best  │
│ • Reset mocks between tests (@MockitoExtension does this)  │
└─────────────────────────────────────────────────────────────┘
```
