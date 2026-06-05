# Spring Boot Exception Handling

---

## 1. Executive Summary

### What Is It?
Spring Boot provides a structured approach to handling exceptions in web applications through `@ControllerAdvice` / `@RestControllerAdvice`, `ErrorController`, and automatic error responses (Whitelabel Error Page).

### Why Does It Exist?
Without structured exception handling:
- Stack traces leak to production responses (security risk)
- Inconsistent error format across endpoints
- Mixed HTTP status codes for the same error type
- No centralized logging of errors

---

## 2. Core Theory

### Exception Handling Approaches

| Approach | Scope | When to Use |
|----------|-------|-------------|
| `@ExceptionHandler` in Controller | Per controller | Endpoint-specific handling |
| `@ControllerAdvice` | Global | Across all controllers |
| `HandlerExceptionResolver` | Framework-level | Customizing framework error handling |
| `ErrorController` / `ErrorAttributes` | All unhandled errors | Catch-all (404, 500, etc.) |
| `ResponseStatusException` | Inline in code | One-off status code + reason |

### @ControllerAdvice

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex) {
        return new ErrorResponse("NOT_FOUND", ex.getMessage());
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        return new ErrorResponse("VALIDATION_FAILED", "Input validation failed");
    }
    
    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public ErrorResponse handleAccessDenied(AccessDeniedException ex) {
        return new ErrorResponse("FORBIDDEN", "Insufficient permissions");
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleUnhandled(Exception ex) {
        log.error("Unhandled exception", ex);
        return new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred");
    }
}
```

### ResponseStatusException (Preferred for Service Layer)

```java
@Service
public class OrderService {
    
    public Order findById(Long id) {
        return orderRepository.findById(id)
            .orElseThrow(() -> new ResponseStatusException(
                HttpStatus.NOT_FOUND, 
                "Order not found: " + id
            ));
    }
    
    public Order create(OrderRequest request) {
        if (!userRepository.existsById(request.getCustomerId())) {
            throw new ResponseStatusException(
                HttpStatus.BAD_REQUEST,
                "Customer does not exist: " + request.getCustomerId()
            );
        }
        // ...
    }
}
```

### Business Exception Classes

```java
// Abstract business exception
public abstract class BusinessException extends RuntimeException {
    private final String code;
    private final HttpStatus status;
    
    public BusinessException(String code, HttpStatus status, String message) {
        super(message);
        this.code = code;
        this.status = status;
    }
    
    public String getCode() { return code; }
    public HttpStatus getStatus() { return status; }
}

// Concrete exceptions
public class InsufficientFundsException extends BusinessException {
    public InsufficientFundsException(BigDecimal balance, BigDecimal required) {
        super("INSUFFICIENT_FUNDS", HttpStatus.CONFLICT,
            String.format("Insufficient funds: available %s, required %s", balance, required));
    }
}

public class OrderAlreadyShippedException extends BusinessException {
    public OrderAlreadyShippedException(Long orderId) {
        super("ORDER_ALREADY_SHIPPED", HttpStatus.CONFLICT,
            "Order " + orderId + " has already been shipped");
    }
}
```

### Structured Error Response

```java
public record ErrorResponse(
    String error,
    String message,
    Instant timestamp,
    String path,
    Map<String, List<String>> details  // field-level errors
) {
    public static ErrorResponse of(String error, String message, 
                                   HttpServletRequest request) {
        return new ErrorResponse(error, message, Instant.now(), 
            request.getRequestURI(), null);
    }
    
    public static ErrorResponse withDetails(String error, String message,
                                            HttpServletRequest request,
                                            Map<String, List<String>> details) {
        return new ErrorResponse(error, message, Instant.now(),
            request.getRequestURI(), details);
    }
}
```

---

## 3. Common Mistakes

| # | Mistake | Consequence | Fix |
|---|---------|-------------|-----|
| 1 | Returning stack trace in response | Security exposure | Log stack trace, return sanitized message |
| 2 | Catching Exception and doing nothing | Silent failures | Log and re-throw or handle explicitly |
| 3 | One @ExceptionHandler for Exception.class | Catches everything, same response | Multiple specific handlers |
| 4 | No global handler for unhandled exceptions | Whitelabel error / 500 | Fallback handler |
| 5 | Not logging exception in @ExceptionHandler | Lost debugging info | Always log in catch-all handlers |
| 6 | Using generic RuntimeException | No structured error info | Specific business exceptions |
| 7 | Inconsistent error response format | Hard for clients to parse | Standard ErrorResponse DTO |
| 8 | Not handling binding errors | Field-level errors lost | Handle MethodArgumentNotValidException |

---

## 4. Cheat Sheet

```
═══ EXCEPTION HANDLING ════════════════════════════════════════

┌─ GLOBAL HANDLER ───────────────────────────────────────────┐
│ @RestControllerAdvice                                       │
│ @ExceptionHandler(MyException.class)                        │
│ @ResponseStatus(HttpStatus.BAD_REQUEST)                     │
│ public ErrorResponse handle(MyException ex) { ... }         │
└─────────────────────────────────────────────────────────────┘

┌─ SERVICE LAYER ────────────────────────────────────────────┐
│ throw new ResponseStatusException(                          │
│     HttpStatus.NOT_FOUND, "Order not found");              │
│ throw new InsufficientFundsException(balance, required);    │
└─────────────────────────────────────────────────────────────┘

┌─ BEST PRACTICES ───────────────────────────────────────────┐
│ • Always log exceptions in global handler                   │
│ • Never expose stack traces to API consumers                │
│ • Use structured ErrorResponse DTO                          │
│ • Return appropriate HTTP status codes                      │
│ • Use @ResponseStatusException in service layer             │
│ • Extend RuntimeException for business exceptions           │
└─────────────────────────────────────────────────────────────┘
```
