# Spring Boot Exception Handling

---

## Overview

- **Definition** — **Spring Boot Exception Handling** provides a structured approach to managing errors in web applications through **`@ControllerAdvice`** / **`@RestControllerAdvice`**, **`ErrorController`**, and automatic error responses. It ensures consistent, secure, and informative error responses across all endpoints.
- **Why It Exists** — Structured exception handling prevents stack traces from leaking to production responses (creating security risks), ensures consistent error formats across endpoints, and avoids generic Whitelabel Error Pages with no useful information.
- **Key Concepts**

### Why Structured Exception Handling Matters

- Without it, stack traces may leak to production responses, creating **security risks** that expose internal implementation details to potential attackers.
- Inconsistent error formats across endpoints make API clients harder to build and maintain, increasing integration time for frontend teams.
- Mixed HTTP status codes for the same error type confuse consumers and make error handling logic on the client side unnecessarily complex.
- Unhandled exceptions return generic **Whitelabel Error Pages** with no useful information, providing a poor developer experience for API consumers.

### Exception Handling Approaches

- **`@ExceptionHandler` in Controller** — Handles exceptions locally within a single controller. Best for endpoint-specific handling that differs from the global pattern.
- **`@ControllerAdvice` / `@RestControllerAdvice`** — Global exception handling across all controllers. This is the most common approach for consistent error responses.
- **`HandlerExceptionResolver`** — Framework-level customization for handling how Spring MVC resolves exceptions. Used for low-level framework integration.
- **`ErrorController` / `ErrorAttributes`** — Catch-all for all unhandled errors including 404s and 500s that reach the servlet container.
- **`ResponseStatusException`** — Thrown inline in service code to return a specific HTTP status and reason without creating custom exception classes.

---

## Core Concepts

### Global Exception Handler with `@RestControllerAdvice`

- The recommended way to handle exceptions centrally:

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

### `ResponseStatusException` in the Service Layer

- For one-off status codes without creating custom exception classes:

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
    }
}
```

### Business Exception Classes

- For domain-specific errors with consistent error codes:

```java
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

### Structured Error Response DTO

- Always return a consistent error format to API clients:

```java
public record ErrorResponse(
    String error,
    String message,
    Instant timestamp,
    String path,
    Map<String, List<String>> details
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

## Common Mistakes

- **Returning stack traces in API responses** — Exposes internal implementation details, file paths, and potentially sensitive data like database queries.
- **Why it looks correct**: During development seeing the full trace helps debugging — but in production, the same response leaks class names and SQL queries to attackers.
- **Fix**: Always log the full stack trace server-side and return a sanitized message with a correlation ID.
- **Catching `Exception` and doing nothing** — Silent failures make debugging nearly impossible and leave the system in an unknown state.
- **Why it looks correct**: The code compiles and the catch block prevents the user from seeing an error — the application returns a successful response despite having failed internally.
- **Fix**: Always log the exception or re-throw it wrapped in a meaningful business exception.
- **Single `@ExceptionHandler` for `Exception.class`** — Catches everything and returns the same generic response, losing specific error information.
- **Why it looks correct**: All errors are caught and return a 500 — the application never crashes — but validation errors that should be 400 with field details are indistinguishable from NPEs.
- **Fix**: Use multiple specific handlers for different exception types with appropriate status codes.
- **No global fallback handler** — Unhandled exceptions result in generic Whitelabel error pages with no structured information.
- **Why it looks correct**: During development with thorough testing, unhandled exceptions are rare — the catch-all only matters when an unexpected edge case reaches production.
- **Fix**: Always include a catch-all `@ExceptionHandler(Exception.class)` to ensure every error returns a consistent format.
- **Not logging in `@ExceptionHandler`** — Without logging, you lose debugging information for production issues and cannot trace the root cause.
- **Why it looks correct**: The exception handler returns a clean response and the application appears to work — the missing logs only become apparent when investigating a production incident with no trail.
- **Fix**: Always include logging in catch-all handlers, especially for 500 errors.
- **Using generic `RuntimeException`** — Provides no structured error information, forcing handlers to parse message strings.
- **Why it looks correct**: `throw new RuntimeException("Order not found")` is quick to write and works — the error message is human-readable. The problem only surfaces when the exception handler must return different HTTP statuses for different errors.
- **Fix**: Create specific business exception classes with meaningful error codes and contextual data.
- **Inconsistent error response format** — Makes it harder for API clients to parse errors uniformly across endpoints.
- **Why it looks correct**: Each endpoint returns errors that make sense for that endpoint — the inconsistency only becomes visible when a frontend developer tries to write a single error-handling function that works across all endpoints.
- **Fix**: Use a standard `ErrorResponse` DTO everywhere and enforce it with code reviews or ArchUnit tests.
- **Not handling validation/binding errors** — Field-level validation errors are lost without handling `MethodArgumentNotValidException`, leaving API clients guessing which field caused the failure.
- **Why it looks correct**: Spring's default 400 response includes a reason like "Bad Request" — the client sees an error, but without field-level details they cannot fix the input.
- **Fix**: Always handle `MethodArgumentNotValidException` explicitly.

---

## Real-World Scenarios

### Scenario 1: Payment Gateway with Structured Error Responses

- A payment processing system communicates with three external gateways (Stripe, PayPal, Braintree). Each returns errors in different formats. Your API clients need a consistent error format regardless of which gateway failed.

```java
@RestControllerAdvice
public class PaymentExceptionHandler {
    @ExceptionHandler(PaymentGatewayException.class)
    public ResponseEntity<ErrorResponse> handlePaymentError(PaymentGatewayException ex) {
        // Map external gateway error to internal standard format
        ErrorResponse error = ErrorResponse.builder()
            .code(ex.getGatewayCode())
            .message(ex.getUserMessage())
            .timestamp(Instant.now())
            .details(Map.of(
                "gateway", ex.getGatewayName(),
                "declineCode", ex.getDeclineCode(),
                "retryable", String.valueOf(ex.isRetryable())
            ))
            .build();
        return ResponseEntity.status(HttpStatus.PAYMENT_REQUIRED).body(error);
    }

    @ExceptionHandler(NetworkTimeoutException.class)
    public ResponseEntity<ErrorResponse> handleTimeout(NetworkTimeoutException ex) {
        return ResponseEntity.status(HttpStatus.GATEWAY_TIMEOUT)
            .body(ErrorResponse.of("GATEWAY_TIMEOUT", "Payment gateway did not respond in time"));
    }
}
```

### Scenario 2: Microservices with Correlation IDs for Error Tracking

- A distributed system has 10 microservices. When a customer reports an error, support needs to trace it across all services. Without correlation IDs, finding the root cause takes hours.

```java
@Component
public class CorrelationIdFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) {
        HttpServletRequest httpReq = (HttpServletRequest) request;
        String correlationId = httpReq.getHeader("X-Correlation-Id");
        if (correlationId == null || correlationId.isBlank()) {
            correlationId = UUID.randomUUID().toString();
        }
        MDC.put("correlationId", correlationId);
        ((HttpServletResponse) response).setHeader("X-Correlation-Id", correlationId);
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.remove("correlationId");
        }
    }
}

// In global exception handler:
@ExceptionHandler(Exception.class)
@ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
public ErrorResponse handleAll(Exception ex, HttpServletRequest request) {
    log.error("Unhandled exception [correlationId={}]", MDC.get("correlationId"), ex);
    return new ErrorResponse("INTERNAL_ERROR",
        "An error occurred. Reference: " + MDC.get("correlationId"),
        Instant.now(), request.getRequestURI(), null);
}
```

### Scenario 3: File Upload Service with Validation Exception Handling

- A document management system accepts file uploads. Users send invalid files constantly — wrong formats, oversized files, missing metadata. Each failure must return precise field-level errors.

```java
@RestControllerAdvice
public class FileUploadExceptionHandler {
    @ExceptionHandler(MultipartException.class)
    @ResponseStatus(HttpStatus.PAYLOAD_TOO_LARGE)
    public ErrorResponse handleMaxUploadSizeExceeded(MultipartException ex) {
        return ErrorResponse.of("FILE_TOO_LARGE",
            "File exceeds maximum upload size of 10MB");
    }

    @ExceptionHandler(InvalidFileFormatException.class)
    @ResponseStatus(HttpStatus.UNSUPPORTED_MEDIA_TYPE)
    public ErrorResponse handleInvalidFormat(InvalidFileFormatException ex) {
        return ErrorResponse.withDetails("INVALID_FILE_FORMAT", "Unsupported file type",
            null, Map.of("allowedFormats", List.of("PDF", "DOCX", "JPG"),
                         "receivedFormat", List.of(ex.getReceivedFormat())));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidationErrors(MethodArgumentNotValidException ex) {
        Map<String, List<String>> fieldErrors = ex.getBindingResult()
            .getFieldErrors().stream()
            .collect(Collectors.groupingBy(
                FieldError::getField,
                Collectors.mapping(FieldError::getDefaultMessage, Collectors.toList())
            ));
        return ErrorResponse.withDetails("VALIDATION_FAILED",
            "Request validation failed", null, fieldErrors);
    }
}
```

---

## Use Cases

- Structured exception handling is critical for production APIs that serve external clients or integrate with other services. These use cases show when to use each Spring error-handling mechanism.

- **Global consistent error responses via `@RestControllerAdvice`** — A public REST API must return errors in a uniform JSON format (`{error, message, status, timestamp, path}`) across all endpoints.
  - Create a single `@RestControllerAdvice` with `@ExceptionHandler` methods for each exception type. Return a `ResponseEntity<ErrorResponse>` with appropriate HTTP status codes.
  - **Avoid when:** Only one controller needs custom error handling — use an `@ExceptionHandler` directly in that controller instead.

- **Correlation ID propagation for microservice debugging** — A 10-microservice system makes it impossible to trace a single user request across service boundaries during errors.
  - Add a servlet `Filter` that generates or propagates an `X-Correlation-Id` header, stores it in `MDC`, and includes it in all error responses. Downstream services forward the header.
  - **Avoid when:** You operate a monolith — a single `@ControllerAdvice` with structured logging is sufficient.

- **Field-level validation error collection** — A file upload API returns generic "invalid request" when a CSV row has multiple validation failures, forcing clients to guess which field is wrong.
  - Handle `MethodArgumentNotValidException` in `@RestControllerAdvice`. Extract `BindingResult.getFieldErrors()` and group by field name in the error response.
  - **Avoid when:** You only validate at the service layer — use a custom exception that carries field-level errors and handle it in `@ControllerAdvice`.

- **Graceful handling of external API failures** — A payment gateway call fails with a timeout, and the API must return a `502 Bad Gateway` with a user-friendly message instead of a 500 with a stack trace.
  - Catch the gateway exception in `@ExceptionHandler` and map it to `HttpStatus.BAD_GATEWAY` with a redacted message. Include a retryable flag and correlation ID.
  - **Avoid when:** The external failure is transient — combine with Spring Retry to retry before surfacing the error to the client.

- **Catch-all for unexpected errors** — An unhandled `NullPointerException` in deep code reaches the servlet container and returns a generic Whitelabel Error Page with no details.
  - Register a fallback `@ExceptionHandler(Exception.class)` that logs the full stack trace and returns a sanitized 500 response with a reference ID for support.
  - **Avoid when:** You use a centralized error tracking service (Sentry, Datadog) — delegate to it and return a minimal error to the client.

---

## Scenario-Based Questions

- **Q: You ship a new API endpoint to production. A client integration team reports they get a generic 500 with no body for certain inputs, but your logs don't show any errors. How do you diagnose this?**
- A: The most likely cause is an exception thrown in a Spring Security filter or a servlet filter that runs before the `@RestControllerAdvice` — these exceptions never reach your controller advice. Check `AuthenticationEntryPoint` and `AccessDeniedHandler` configurations. Also check if the error is a 404 (wrong URL) where Spring's default `BasicErrorController` returns the Whitelabel error page. Add a custom `ErrorController` or configure `server.error.include-message=always` to see the actual error.
- **Interview follow-up:** The candidate identified the root cause as a Security filter exception. If you add a catch-all `try-catch` in a `OncePerRequestFilter` that writes the error response, but the `@RestControllerAdvice` also handles `Exception.class`, an exception thrown in the controller will be caught by both — the filter writes the response, then the advice tries to write again, causing an `IllegalStateException: response already committed`. How would you design a single mechanism that handles errors from both filters and controllers without duplication?

- **Q: Your `@RestControllerAdvice` catches `Exception.class` and returns a sanitized message. A developer complains that they can't debug production issues because the response always says "An unexpected error occurred" even for validation errors. How do you design a better strategy?**
- A: Implement a layered exception handling hierarchy. Handle specific exception types before the catch-all:

```java
@ExceptionHandler(MethodArgumentNotValidException.class) // 400 with field details
@ExceptionHandler(AccessDeniedException.class)            // 403
@ExceptionHandler(ResourceNotFoundException.class)        // 404
@ExceptionHandler(BusinessException.class)                // Dynamic status from exception
@ExceptionHandler(Exception.class)                        // 500 catch-all
```

- Log the full stack trace for `Exception.class` but include a correlation ID in the response so developers can find the log entry. Never expose stack traces to clients, but do expose field-level validation errors — they are not security-sensitive.

- **Q: Your service layer throws `InsufficientFundsException` which should return 409 Conflict. But the client always gets 500 Internal Server Error. You have a handler for `BusinessException` parent class. What's wrong?**
- A: Two possible causes: (a) The `@ExceptionHandler` for `Exception.class` is defined before the `BusinessException` handler in the code — Spring checks handlers in declaration order in some configurations. (b) The `InsufficientFundsException` might not extend `BusinessException` — it might extend `RuntimeException` directly. Fix by ensuring `BusinessException` handlers are defined first and the exception hierarchy is correct:

```java
public class InsufficientFundsException extends BusinessException {
    public InsufficientFundsException() {
        super("INSUFFICIENT_FUNDS", HttpStatus.CONFLICT, "Insufficient balance");
    }
}
```

- **Q: You want to return different error response formats for API clients (JSON) vs browser users (HTML). How do you configure Spring to handle both?**
- A: Use content negotiation. Configure `ErrorAttributes` for structured data and create a custom `ErrorController` that checks the `Accept` header:

```java
@RequestMapping("/error")
public ResponseEntity<Map<String, Object>> handleError(HttpServletRequest request) {
    Map<String, Object> body = errorAttributes.getErrorAttributes(
        request, ErrorAttributeOptions.defaults());
    HttpStatus status = getStatus(request);
    if (request.getHeader("Accept") != null &&
        request.getHeader("Accept").contains("text/html")) {
        return ResponseEntity.status(status)
            .contentType(MediaType.TEXT_HTML)
            .body(Map.of("error", body.get("error"), "status", status.value()));
    }
    return ResponseEntity.status(status).body(body);
}
```

- Alternatively, use separate `@ControllerAdvice` for `@RestController` (which returns JSON) and a custom error page for browsers.

- **Q: A REST client sends an invalid JSON payload — a missing closing brace. The client receives a 400 with a Jackson parse error message that includes the full request body. This is a security risk. How do you sanitize it?**
- A: Handle `HttpMessageNotReadableException` in your `@RestControllerAdvice`:

```java
@ExceptionHandler(HttpMessageNotReadableException.class)
@ResponseStatus(HttpStatus.BAD_REQUEST)
public ErrorResponse handleMalformedJson(HttpMessageNotReadableException ex) {
    log.warn("Malformed JSON received", ex);
    return ErrorResponse.of("MALFORMED_JSON", "Request body is not valid JSON");
}
```

- Never expose the original exception message to the client as it may contain the full payload. Always log the detail server-side and return a sanitized message.

- **Q: Your team has 15 microservices, each with its own exception handling. The frontend team is frustrated because every service returns errors in a different format. How do you standardize across all services?**
- A: Create a shared library (internal Maven/Gradle artifact) with: (a) A standard `ErrorResponse` DTO with fields: `code`, `message`, `timestamp`, `path`, `correlationId`, `details`. (b) A base `BusinessException` class with status code and error code. (c) A standard `@RestControllerAdvice` base class. Every microservice depends on this library. This ensures all 15 services return identical error structures. Add a compliance test that verifies the error format in CI/CD.
- **Interview follow-up:** The candidate proposed a shared library. One microservice has a legacy endpoint that returns errors in the old format and cannot be changed due to external client contracts. The shared library's `@RestControllerAdvice` replaces all error responses, breaking that legacy endpoint. How would you design the shared library to allow per-endpoint opt-out of the standard format?

- **Q: Your `@ExceptionHandler` for `ConstraintViolationException` works locally but not in production. The production setup uses a different validation framework. What could differ?**
- A: In production, the application might be using a different web server or have a different classpath ordering. Check if `hibernate-validator` is the only Bean Validation implementation on the classpath (production might have multiple). Ensure the `MethodValidationPostProcessor` is configured if using method-level validation on services. Also check if the production environment uses a different Spring Boot version where the exception type changed (e.g., from `ConstraintViolationException` to `MethodArgumentNotValidException`).

- **Q: A scheduled batch job processes 10K records. If one record fails, should the entire batch roll back or should individual records fail silently? How do you design this?**
- A: Use a combination: wrap the batch in a `@Transactional` for the outer job, but process each record in a separate inner transaction using `REQUIRES_NEW` or `TransactionTemplate`:

```java
public void processBatch(List<Record> records) {
    for (Record record : records) {
        try {
            transactionTemplate.execute(status -> {
                processSingleRecord(record);
                return null;
            });
        } catch (Exception e) {
            log.error("Failed to process record {}: {}", record.getId(), e.getMessage());
            errorCollector.record(record.getId(), e);
            // Continue with next record — batch is not rolled back
        }
    }
    if (!errorCollector.isEmpty()) {
        notificationService.alert("Batch completed with " + errorCollector.size() + " errors");
    }
}
```

- **Q: Your REST API exposes a `POST /api/orders` endpoint. When validation fails, you want to return field-level errors, but also want to include a business-level error if the order total exceeds the customer's credit limit. How do you combine both types?**
- A: First, let `@Valid` handle field-level validation and return `400 BAD_REQUEST` with field errors. In the service layer, throw a `CreditLimitExceededException` which is caught by a separate `@ExceptionHandler` returning `409 CONFLICT`. Use a custom `ErrorResponse` that includes both field-level and business-level details:

```java
public record ErrorResponse(
    String code, String message, Instant timestamp,
    Map<String, List<String>> fieldErrors,   // from validation
    List<BusinessError> businessErrors        // from business logic
) {}
```

- **Q: Your application uses `@Transactional` and the service layer throws an exception. The transaction rolls back, but the exception handler in `@RestControllerAdvice` also needs to read from the database to build the error response. The read returns empty because the transaction rolled back. How do you handle this?**
- A: The read fails because the `EntityManager` is closed after rollback. Build the error response from the exception data alone — don't query the database in the exception handler. If you need additional data for the error response, capture it before the transaction ends:

```java
@Transactional
public Order createOrder(CreateOrderRequest request) {
    try {
        // business logic
    } catch (InsufficientFundsException e) {
        throw new OrderCreationException(
            "INSUFFICIENT_FUNDS", e,
            Map.of("available", e.getBalance(), "required", e.getRequired()));
    }
}
```

- The exception handler uses the data from the exception, not from the database.

---

## Interview Questions

- **What is `@RestControllerAdvice` and how does it work?**
- A: `@RestControllerAdvice` is a specialization of `@ControllerAdvice` for REST APIs. It applies `@ExceptionHandler`, `@InitBinder`, and `@ModelAttribute` methods globally across all `@RestController` classes. Spring scans it at startup and routes exceptions to the matching handler based on the exception type hierarchy.

- **What is the difference between `@ExceptionHandler` in a controller vs a controller advice?**
- A: An `@ExceptionHandler` in a specific controller handles exceptions only for that controller. In a `@ControllerAdvice`, it handles exceptions globally across all controllers. Use controller-level for endpoint-specific behavior (e.g., different error format for file upload endpoints) and advice-level for consistent global error handling.

- **How do you handle exceptions thrown in Spring Security filters?**
- A: Exceptions in Spring Security filters occur before the controller, so `@RestControllerAdvice` does not catch them. Handle them with `AuthenticationEntryPoint` (for authentication failures), `AccessDeniedHandler` (for authorization failures), or a custom `OncePerRequestFilter` with try-catch that writes the error response directly.

- **What is `ResponseStatusException` and when should you use it?**
- A: `ResponseStatusException` is a runtime exception that combines an HTTP status code, a reason, and optional cause. Use it for simple, single-use error cases in service or controller code where creating a custom exception class is overkill. Example: `throw new ResponseStatusException(NOT_FOUND, "Order not found")`.

- **What is the difference between `BusinessException` (custom) and `ResponseStatusException`?**
- A: Custom `BusinessException` classes are reusable across multiple throw points, support structured error codes and additional fields, and enable `@ExceptionHandler` to match on specific types. `ResponseStatusException` is inline and cannot be matched by type in exception handlers. Use custom exceptions for domain errors; use `ResponseStatusException` for one-off cases.

- **How do you implement a global exception handler that returns different HTTP status codes for different exceptions?**
- A: Create `@ExceptionHandler` methods for each exception type in a `@RestControllerAdvice` class. Each method declares the response status with `@ResponseStatus` or returns `ResponseEntity` with the desired status. Always include a catch-all `@ExceptionHandler(Exception.class)` for unexpected errors.

- **What is the `ErrorController` and when would you customize it?**
- A: `ErrorController` is Spring's fallback for handling errors that reach the servlet container (404s, 500s not caught by `@RestControllerAdvice`). Customize it when you need to: (a) return a consistent JSON format for 404s, (b) change the default Whitelabel error page, or (c) add correlation IDs to all error responses.

- **How do you handle validation errors with field-level messages?**
- A: Handle `MethodArgumentNotValidException` in `@RestControllerAdvice`. Extract `BindingResult.getFieldErrors()` and group by field name. Return a structured response with field → [error messages] mapping. Always include the field name so clients can highlight the invalid input.

- **What is the order of `@ExceptionHandler` resolution in Spring?**
- A: Spring checks handlers in this order: (a) `@ExceptionHandler` in the controller class, (b) `@ExceptionHandler` in `@ControllerAdvice`, (c) `DefaultHandlerExceptionResolver` (Spring MVC internal), (d) `ErrorController` / Whitelabel page. Within an advice, handlers are checked in declaration order — define specific exceptions before the catch-all.

- **How does `@Transactional` interact with exception handling?**
- A: By default, `@Transactional` rolls back for `RuntimeException` and `Error`, but not for checked exceptions. Use `rollbackFor` to customize. If an exception is caught and swallowed in the service layer, the transaction does not roll back. Exception handlers in `@RestControllerAdvice` run after the transaction commits or rolls back, so they cannot access the database if a rollback occurred.

---

## Developer Recommendations

- **Use custom `BusinessException` classes over generic `RuntimeException`**
- Custom exceptions carry structured codes, HTTP statuses, and contextual data. A generic `RuntimeException` forces the handler to parse the message string, which is fragile and makes error responses inconsistent across the codebase.

- **Always include a correlation ID in every error response**
- In distributed systems, error responses without correlation IDs make debugging nearly impossible. Generate a UUID in a servlet filter, store it in MDC, include it in the error response, and log it with every exception. Support can then search for the correlation ID across all services.

- **Handle `MethodArgumentNotValidException` with field-level details**
- Returning a generic 400 without field-level errors forces API clients to guess which field is wrong. Extract `FieldError` details and return them grouped by field name. This reduces client-side debugging time and improves developer experience.

- **Never expose stack traces in production error responses**
- Stack traces reveal internal class names, file paths, and library versions — valuable information for attackers. Always log the full stack trace server-side and return a sanitized message with a reference ID for support.

- **Use specific handlers before the catch-all**
- Spring checks `@ExceptionHandler` methods in declaration order. If `Exception.class` handler comes first, all exceptions return 500 regardless of type. Define handlers from most specific to most general: validation → auth → not-found → business → catch-all.

- **Use `ResponseStatusException` sparingly and only for one-off cases**
- It is convenient but cannot be caught by type-specific `@ExceptionHandler` methods. If the same error (e.g., "Order not found") is thrown from 10 places, create a `OrderNotFoundException` class once and use it everywhere.

- **Handle `HttpMessageNotReadableException` and `HttpMediaTypeNotSupportedException` explicitly**
- These are thrown before the controller method is invoked, and the default Spring response includes the full request body in the error message. Always override these to return sanitized messages.

- **Log exceptions at the correct level**
- Validation errors are `WARN` (client mistakes), not `ERROR`. Business rule violations (insufficient funds) are `INFO` with context. Only truly unexpected exceptions (NPE, connection failures) should be `ERROR`. This keeps log monitoring actionable.
