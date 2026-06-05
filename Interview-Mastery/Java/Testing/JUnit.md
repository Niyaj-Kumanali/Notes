# JUnit 5

---

## 1. Executive Summary

JUnit 5 is the latest version of the Java testing framework. It consists of three modules:
- **JUnit Platform** — launches tests on JVM
- **JUnit Jupiter** — programming model (annotations, assertions)
- **JUnit Vintage** — backward compatibility with JUnit 4

---

## 2. Core Theory

### Annotations

| Annotation | Purpose |
|-----------|---------|
| `@Test` | Marks a test method |
| `@ParameterizedTest` | Parameterized test method |
| `@RepeatedTest` | Repeat test N times |
| `@BeforeAll` | Static method, runs once before all tests |
| `@AfterAll` | Static method, runs once after all tests |
| `@BeforeEach` | Instance method, runs before each test |
| `@AfterEach` | Instance method, runs after each test |
| `@DisplayName` | Custom test name |
| `@Disabled` | Skip test |
| `@Tag` | Filter tests by tag |
| `@Nested` | Inner test class |
| `@Timeout` | Fail test if exceeds duration |

### Assertions

```java
import static org.junit.jupiter.api.Assertions.*;

@Test
void standardAssertions() {
    assertEquals(4, calculator.add(2, 2));
    assertNotEquals(5, calculator.add(2, 2));
    assertTrue(result.isValid());
    assertFalse(result.hasErrors());
    assertNull(nullValue);
    assertNotNull(nonNullValue);
    assertThrows(IllegalArgumentException.class, 
        () -> calculator.divide(1, 0));
    assertDoesNotThrow(() -> calculator.divide(1, 2));
}

@Test
void groupedAssertions() {
    assertAll("address",
        () -> assertEquals("John", address.getFirstName()),
        () -> assertEquals("Doe", address.getLastName()),
        () -> assertNotNull(address.getZipCode())
    );
}
```

### Parameterized Tests

```java
@ParameterizedTest
@ValueSource(strings = {"racecar", "radar", "level"})
void palindromes(String word) {
    assertTrue(StringUtils.isPalindrome(word));
}

@ParameterizedTest
@CsvSource({
    "1, 1, 2",
    "2, 3, 5",
    "10, 20, 30"
})
void addition(int a, int b, int expected) {
    assertEquals(expected, calculator.add(a, b));
}

@ParameterizedTest
@MethodSource("provideOrders")
void testOrderProcessing(Order order, boolean expected) {
    assertEquals(expected, orderService.process(order));
}

static Stream<Arguments> provideOrders() {
    return Stream.of(
        Arguments.of(new Order(100, "ACTIVE"), true),
        Arguments.of(new Order(0, "INACTIVE"), false)
    );
}
```

### Test Lifecycle

```java
class OrderServiceTest {
    
    private OrderService orderService;
    
    @BeforeAll
    static void setupDatabase() {
        // Initialize test database (once)
    }
    
    @BeforeEach
    void setUp() {
        orderService = new OrderService();
    }
    
    @Test
    @DisplayName("Should create order with valid request")
    void shouldCreateOrder() {
        CreateOrderRequest request = new CreateOrderRequest(
            "customer@test.com", List.of("item-1", "item-2"));
        
        Order order = orderService.createOrder(request);
        
        assertNotNull(order.getId());
        assertEquals(2, order.getItemCount());
    }
    
    @AfterEach
    void tearDown() {
        orderService = null;
    }
    
    @AfterAll
    static void cleanupDatabase() {
        // Clean up test database
    }
}
```

---

## 3. Cheat Sheet

```
═══ JUNIT 5 ═══════════════════════════════════════════════════

┌─ LIFECYCLE ────────────────────────────────────────────────┐
│ @BeforeAll    → @BeforeEach    → @Test    → @AfterEach    →│
│                                                            │
│ @BeforeAll    → @BeforeEach    → @Test    → @AfterEach    →│
│                                             → @AfterAll    │
└─────────────────────────────────────────────────────────────┘

┌─ ASSERTIONS ───────────────────────────────────────────────┐
│ assertEquals(expected, actual)      // ==                   │
│ assertNotEquals(expected, actual)                          │
│ assertTrue(condition)                                      │
│ assertFalse(condition)                                     │
│ assertNull(object) / assertNotNull(object)                 │
│ assertThrows(Exception.class, executable)                  │
│ assertDoesNotThrow(executable)                             │
│ assertAll("name", () -> ..., () -> ...)  // grouped        │
│ assertIterableEquals(iterable1, iterable2)                 │
│ assertTimeout(ofSeconds(2), executable)                    │
└─────────────────────────────────────────────────────────────┘

┌─ PARAMETERIZED ────────────────────────────────────────────┐
│ @ValueSource(strings = {"a", "b"})                         │
│ @CsvSource({"1, 2, 3", "4, 5, 9"})                        │
│ @MethodSource("staticMethod")                              │
│ @EnumSource(MyEnum.class)                                  │
│ @ArgumentsSource(CustomProvider.class)                     │
└─────────────────────────────────────────────────────────────┘

┌─ EXTENSIONS ───────────────────────────────────────────────┐
│ @ExtendWith(SpringExtension.class)    // Spring integration │
│ @ExtendWith(MockitoExtension.class)   // Mockito           │
│ @TempDir                               // Temp directory   │
│ @Timeout(5, unit = TimeUnit.SECONDS)   // Timeout          │
└─────────────────────────────────────────────────────────────┘
```
