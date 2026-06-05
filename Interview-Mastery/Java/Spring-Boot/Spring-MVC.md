# Spring MVC

---

## 1. Executive Summary

### What Is It?
Spring MVC is a web framework built on the Servlet API that implements the Model-View-Controller pattern. It provides a clean separation between presentation logic, business logic, and data.

### Request Lifecycle

```
HTTP Request
    ↓
DispatcherServlet (Front Controller)
    ↓
HandlerMapping → determines which controller handles the request
    ↓
Controller (handles request, returns ModelAndView / @ResponseBody)
    ↓
Interceptors → pre/post processing
    ↓
HandlerAdapter → executes the controller method
    ↓
ViewResolver (if returning view name) → resolves to View
    ↓
View → renders response (JSP, Thymeleaf, JSON, etc.)
    ↓
HTTP Response
```

### Core Components

| Component | Purpose |
|-----------|---------|
| `DispatcherServlet` | Front controller — entry point for all requests |
| `HandlerMapping` | Maps requests to handlers |
| `Controller` | Handles the request, returns model data |
| `HandlerAdapter` | Adapts handler types to the framework |
| `ViewResolver` | Resolves view name to View implementation |
| `View` | Renders the response |
| `Model` | Holds data to be rendered by the view |
| `ModelAndView` | Container for both model data and view name |

---

## 2. Core Theory

### Annotations

| Annotation | Usage |
|------------|-------|
| `@Controller` | Class — marks as MVC controller |
| `@RestController` | Class — @Controller + @ResponseBody on all methods |
| `@RequestMapping` | Class/method — maps HTTP method + path |
| `@GetMapping` | Method — GET shortcut |
| `@PostMapping` | Method — POST shortcut |
| `@PutMapping` | Method — PUT shortcut |
| `@DeleteMapping` | Method — DELETE shortcut |
| `@PatchMapping` | Method — PATCH shortcut |
| `@RequestParam` | Method param — query parameter |
| `@PathVariable` | Method param — path segment |
| `@RequestBody` | Method param — request body |
| `@RequestHeader` | Method param — HTTP header |
| `@CookieValue` | Method param — cookie value |
| `@ModelAttribute` | Method/param — model attribute |
| `@SessionAttributes` | Class — store model in HTTP session |
| `@ResponseBody` | Method — write directly to response body |
| `@ResponseStatus` | Method — set HTTP status code |
| `@ExceptionHandler` | Method — handle specific exceptions |
| `@ControllerAdvice` | Class — global exception handling |
| `@InitBinder` | Method — customize data binding |
| `@CrossOrigin` | Method/class — CORS configuration |

### Content Negotiation

```java
@GetMapping(value = "/users/{id}", produces = MediaType.APPLICATION_JSON_VALUE)
public User getUser(@PathVariable Long id) {
    return userService.findById(id);
}
```

Spring automatically negotiates JSON, XML, etc. based on:
- `Accept` header
- URL suffix (`.json`, `.xml`)
- Query parameter (`?format=json`)

### RESTful Controller Example

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    private final OrderService orderService;
    
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }
    
    @GetMapping
    public ResponseEntity<List<Order>> getAll(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        PageResult<Order> result = orderService.findAll(page, size);
        return ResponseEntity.ok()
            .header("X-Total-Count", String.valueOf(result.total()))
            .body(result.items());
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<Order> getById(@PathVariable Long id) {
        return orderService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Order create(@Valid @RequestBody CreateOrderRequest request) {
        return orderService.create(request);
    }
    
    @PutMapping("/{id}")
    public Order update(@PathVariable Long id, @Valid @RequestBody UpdateOrderRequest request) {
        return orderService.update(id, request);
    }
    
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        orderService.delete(id);
    }
}
```

---

## 3. Under-the-Hood

### DispatcherServlet Initialization

```
1. DispatcherServlet extends FrameworkServlet extends HttpServletBean
2. On init(): reads config → sets up WebApplicationContext
3. On refresh(): strategy initialization
   - LocaleResolver
   - ThemeResolver
   - HandlerMappings
   - HandlerAdapters
   - HandlerExceptionResolvers
   - RequestToViewNameTranslator
   - ViewResolvers
   - FlashMapManager
```

### Request Processing

```
doService() → doDispatch() →
    1. MultipartContent resolution
    2. mappedHandler = getHandler(processedRequest)  ← finds HandlerExecutionChain
    3. HandlerAdapter adapter = getHandlerAdapter(mappedHandler.getHandler())
    4. mappedHandler.applyPreHandle()  ← interceptors before
    5. ModelAndView mv = adapter.handle(processedRequest, response, mappedHandler.getHandler())
    6. mappedHandler.applyPostHandle()  ← interceptors after
    7. processDispatchResult(processedRequest, response, mappedHandler, mv)
       → render() if view exists
```

### Async Request Processing (DeferredResult / Callable)

```java
@GetMapping("/async")
public DeferredResult<String> asyncProcess() {
    DeferredResult<String> result = new DeferredResult<>(5000L); // 5s timeout
    
    executor.submit(() -> {
        try {
            Thread.sleep(2000); // Simulate long operation
            result.setResult("Done!");
        } catch (Exception e) {
            result.setErrorResult("Failed");
        }
    });
    
    result.onTimeout(() -> result.setErrorResult("Timeout"));
    return result;
}
```

---

## 4. Production Code Examples

### 4.1 Global Exception Handler

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
        List<FieldError> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> new FieldError(e.getField(), e.getDefaultMessage()))
            .toList();
        return new ErrorResponse("VALIDATION_ERROR", "Validation failed", errors);
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneral(Exception ex) {
        log.error("Unhandled exception", ex);
        return new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred");
    }
}
```

### 4.2 CORS Configuration

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://frontend.example.com")
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

### 4.3 Interceptor

```java
@Component
public class RequestLoggingInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, 
                             Object handler) {
        long startTime = System.currentTimeMillis();
        request.setAttribute("startTime", startTime);
        MDC.put("requestId", UUID.randomUUID().toString());
        return true;
    }
    
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response,
                           Object handler, ModelAndView modelAndView) {
        long startTime = (Long) request.getAttribute("startTime");
        long duration = System.currentTimeMillis() - startTime;
        log.info("{} {} completed in {}ms with status {}",
            request.getMethod(), request.getRequestURI(), duration, response.getStatus());
    }
}
```

---

## 5. Cheat Sheet

```
═══ SPRING MVC ═══════════════════════════════════════════════

┌─ REQUEST LIFECYCLE ────────────────────────────────────────┐
│ DispatcherServlet → HandlerMapping → Controller → View      │
└─────────────────────────────────────────────────────────────┘

┌─ CONTROLLER ANNOTATIONS ───────────────────────────────────┐
│ @RestController    — REST controller (JSON by default)      │
│ @RequestMapping    — root mapping on class                  │
│ @GetMapping("/{id}") — GET with path variable               │
│ @PostMapping       — POST request                           │
│ @RequestBody       — deserialize request body               │
│ @PathVariable      — extract path segment                   │
│ @RequestParam      — extract query parameter                │
│ @Valid / @Validated — trigger Bean Validation               │
│ @ResponseStatus    — set HTTP status code                   │
└─────────────────────────────────────────────────────────────┘

┌─ RESPONSE ─────────────────────────────────────────────────┐
│ @ResponseBody / @RestController — JSON/XML response         │
│ ResponseEntity<T> — full control (status + headers + body)  │
│ DeferredResult<T> — async response                          │
│ StreamingResponseBody — stream large data                   │
└─────────────────────────────────────────────────────────────┘

┌─ CONFIGURATION ────────────────────────────────────────────┐
│ WebMvcConfigurer — customize MVC (CORS, interceptors, etc)  │
│ @ControllerAdvice — global exception handler                │
│ @InitBinder — customize data binding                        │
│ @CrossOrigin — per-Controller CORS                          │
└─────────────────────────────────────────────────────────────┘
```
