# Spring Boot Validation

---

## 1. Executive Summary

### What Is It?
Spring Boot Validation provides declarative input validation using Jakarta Bean Validation API (formerly JSR 380 / Jakarta Validation). It integrates with Spring MVC to automatically validate request bodies, parameters, and path variables.

### Core Dependencies
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

---

## 2. Core Theory

### Bean Validation Annotations

| Annotation | Purpose | Example |
|-----------|---------|---------|
| `@NotNull` | Value must not be null | `@NotNull String email` |
| `@NotEmpty` | String/Collection not null and not empty | `@NotEmpty String name` |
| `@NotBlank` | String not null and has at least one non-whitespace char | `@NotBlank String password` |
| `@Size` | Length/size within bounds | `@Size(min=3, max=50) String name` |
| `@Min` / `@Max` | Minimum/maximum value | `@Min(0) @Max(150) int age` |
| `@Positive` / `@PositiveOrZero` | Positive value check | `@Positive BigDecimal price` |
| `@Negative` / `@NegativeOrZero` | Negative value check | `@Negative int adjustment` |
| `@Email` | Valid email format | `@Email String contactEmail` |
| `@Pattern` | Regex match | `@Pattern(regexp = "^[A-Z].*") String code` |
| `@Past` / `@PastOrPresent` | Date in the past | `@Past LocalDate birthDate` |
| `@Future` / `@FutureOrPresent` | Date in the future | `@Future LocalDate eventDate` |
| `@Digits` | Digit count constraint | `@Digits(integer=5, fraction=2) BigDecimal price` |
| `@AssertTrue` / `@AssertFalse` | Boolean check | `@AssertTrue boolean termsAccepted` |

### Validation Groups

```java
public interface Create {}
public interface Update {}

public class UserRequest {
    
    @Null(groups = Create.class)         // Must be null for create
    @NotNull(groups = Update.class)      // Must be provided for update
    private Long id;
    
    @NotBlank(groups = {Create.class, Update.class})
    private String name;
    
    @Email(groups = Create.class)        // Only validate on create
    private String email;
}

// Usage
@PostMapping
public User create(@Validated(Create.class) @RequestBody UserRequest request) { ... }

@PutMapping("/{id}")
public User update(@Validated(Update.class) @RequestBody UserRequest request) { ... }
```

### Custom Validator

```java
@Target({FIELD})
@Retention(RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@Component
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {
    private final UserRepository userRepository;
    
    public UniqueEmailValidator(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
    
    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        return email != null && !userRepository.existsByEmail(email);
    }
}

// Usage
public class CreateUserRequest {
    @Email
    @UniqueEmail
    private String email;
}
```

---

## 3. Production Code

### 3.1 Request Validation

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public User create(@Valid @RequestBody CreateUserRequest request) {
        return userService.create(request);
    }
}

public record CreateUserRequest(
    @NotBlank @Size(min = 2, max = 100)
    String fullName,
    
    @NotBlank @Email
    String email,
    
    @NotBlank @Size(min = 8, max = 100)
    @Pattern(regexp = "^(?=.*[0-9])(?=.*[a-z])(?=.*[A-Z])(?=.*[@#$%^&+=]).*$",
             message = "Password must contain digit, uppercase, lowercase, and special character")
    String password,
    
    @NotNull @Min(18) @Max(150)
    Integer age
) {}
```

### 3.2 Validation Error Handling

```java
@RestControllerAdvice
public class ValidationExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        Map<String, List<String>> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.groupingBy(
                FieldError::getField,
                Collectors.mapping(FieldError::getDefaultMessage, Collectors.toList())
            ));
        
        return new ErrorResponse("VALIDATION_FAILED", "Input validation failed", errors);
    }
    
    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleConstraintViolation(ConstraintViolationException ex) {
        Map<String, String> errors = ex.getConstraintViolations().stream()
            .collect(Collectors.toMap(
                v -> v.getPropertyPath().toString(),
                ConstraintViolation::getMessage
            ));
        return new ErrorResponse("VALIDATION_FAILED", "Parameter validation failed", errors);
    }
}
```

---

## 4. Cheat Sheet

```
═══ SPRING BOOT VALIDATION ════════════════════════════════════

┌─ ANNOTATIONS ──────────────────────────────────────────────┐
│ @NotBlank @NotEmpty @NotNull @Size @Min @Max @Email        │
│ @Pattern @Past @Future @Positive @Negative @Digits          │
│ @AssertTrue @AssertFalse                                    │
└─────────────────────────────────────────────────────────────┘

┌─ ACTIVATION ───────────────────────────────────────────────┐
│ @Valid       — JSR-303 (jakarta.validation-api)            │
│ @Validated   — Spring variant (supports groups)            │
│ Both trigger validation on @RequestBody, @RequestParam     │
└─────────────────────────────────────────────────────────────┘

┌─ CUSTOM VALIDATION ────────────────────────────────────────┐
│ 1. Create annotation with @Constraint(validatedBy = ...)    │
│ 2. Implement ConstraintValidator<A, T>                     │
│ 3. Inject Spring beans into validator                      │
│ 4. Use annotation on fields/methods                        │
└─────────────────────────────────────────────────────────────┘

┌─ BEST PRACTICES ───────────────────────────────────────────┐
│ • Validate at controller boundary (don't trust clients)     │
│ • Use DTOs/records for request/response (not entities)      │
│ • Use validation groups for create vs update                │
│ • Custom validators for business rules (unique email)       │
│ • Return structured error responses (not just 400)          │
│ • Log validation failures for monitoring                    │
└─────────────────────────────────────────────────────────────┘
```
