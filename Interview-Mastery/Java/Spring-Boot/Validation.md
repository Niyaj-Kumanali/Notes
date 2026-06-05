# Spring Boot Validation

---

## What is Spring Boot Validation?

**Spring Boot Validation** provides declarative input validation using the **Jakarta Bean Validation API** (formerly JSR 380 / Jakarta Validation). It integrates with Spring MVC to automatically validate request bodies, request parameters, path variables, and nested objects, reducing boilerplate validation code.

### Key Concepts:

1. **Core Dependencies**:

   Spring Boot includes validation support through `spring-boot-starter-validation`:

   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-validation</artifactId>
   </dependency>
   ```

2. **Bean Validation Annotations**:

   - **`@NotNull`** — Value must not be null.
   - **`@NotEmpty`** — String, collection, map, or array must not be null and must have at least one element.
   - **`@NotBlank`** — String must not be null and must contain at least one non-whitespace character.
   - **`@Size(min, max)`** — Length of string or size of collection must be within bounds.
   - **`@Min` / `@Max`** — Numeric value must be at least / at most the specified value.
   - **`@Positive` / `@PositiveOrZero`** — Must be positive (optionally including zero).
   - **`@Negative` / `@NegativeOrZero`** — Must be negative (optionally including zero).
   - **`@Email`** — Valid email format.
   - **`@Pattern(regexp)`** — Must match the specified regular expression.
   - **`@Past` / `@PastOrPresent`** — Date must be in the past.
   - **`@Future` / `@FutureOrPresent`** — Date must be in the future.
   - **`@Digits(integer, fraction)`** — Numeric value with specified digit limits.
   - **`@AssertTrue` / `@AssertFalse`** — Boolean check.

3. **Validation Groups**:

   Group validation rules for different operations (e.g., create vs update):

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

---

## Core Concepts

### 1. Request Validation in Controller

   Annotate request bodies with `@Valid` or `@Validated` to trigger validation:

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

### 2. Validation Error Handling

   Handle `MethodArgumentNotValidException` to return structured error responses:

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

### 3. Custom Validator

   Create validators that can also inject Spring beans:

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

### 4. Path Variable and Query Parameter Validation

   Validate method parameters using `@Validated` at the class level:

   ```java
   @RestController
   @Validated
   @RequestMapping("/api/v1/products")
   public class ProductController {

       @GetMapping("/{id}")
       public Product getById(@PathVariable @Min(1) Long id) {
           return productService.findById(id);
       }

       @GetMapping
       public List<Product> search(
               @RequestParam @Size(min = 3, max = 100) String query,
               @RequestParam @Min(0) @Max(100) int page) {
           return productService.search(query, page);
       }
   }
   ```

---

## Common Mistakes

1. **Not adding `@Valid` or `@Validated` to request body parameters** — Validation annotations on the DTO are ignored if the controller parameter is not annotated. The method receives an invalid object without any error.

2. **Using entities as request/response DTOs** — Entities have JPA constraints and lazy-loaded associations that are inappropriate for API boundaries. Always use separate DTOs or records.

3. **No global validation exception handler** — Without a handler for `MethodArgumentNotValidException`, Spring returns a generic 400 with a default error message. Add a `@RestControllerAdvice` with structured error responses.

4. **Not validating nested objects** — Nested objects in a request DTO are not validated unless annotated with `@Valid` on the nested field:

```java
public class OrderRequest {
    @Valid  // Without this, Address validation is skipped
    @NotNull
    private Address shippingAddress;
}
```

5. **Ignoring validation groups** — Without groups, the same validation rules apply to both create and update. Use groups to differentiate (e.g., ID is null on create, not null on update).

6. **Over-messaging validation errors** — Exposing internal constraint details to the client. Define clear, user-friendly messages.

---

## Real-World Scenarios

### Scenario 1: User Registration with Cross-Field Validation

A registration form requires `password` and `confirmPassword` to match. The `email` must be unique (checked against the database). The `birthDate` must indicate age ≥ 18.

```java
@ValidDateRange
public record RegistrationRequest(
    @NotBlank @Email String email,
    @NotBlank @Size(min = 8, max = 100) String password,
    @NotBlank String confirmPassword,
    @NotNull @Past LocalDate birthDate,
    @NotNull boolean acceptTerms
) {
    @AssertTrue(message = "Passwords must match")
    boolean isPasswordMatch() {
        return password != null && password.equals(confirmPassword);
    }

    @AssertTrue(message = "You must be at least 18 years old")
    boolean isAdult() {
        return birthDate != null &&
            Period.between(birthDate, LocalDate.now()).getYears() >= 18;
    }
}
```

The custom `@ValidDateRange` annotation handles cross-field validation at the class level.

### Scenario 2: Service-Layer Validation for Database Constraints

A product import service receives bulk CSV data. Validation must check that category names exist in the database, SKUs are unique, and prices are non-negative. These constraints cannot be checked with simple annotations — they require database queries.

```java
@Service
@Validated
public class ProductImportService {
    public ProductImportResult importProducts(@Valid List<@Valid ProductRow> rows) {
        List<ProductImportResult.Error> errors = new ArrayList<>();
        for (int i = 0; i < rows.size(); i++) {
            ProductRow row = rows.get(i);
            try {
                if (categoryRepository.findByName(row.category()).isEmpty()) {
                    errors.add(new Error(i, "category", "Category not found: " + row.category()));
                    continue;
                }
                if (productRepository.existsBySku(row.sku())) {
                    errors.add(new Error(i, "sku", "Duplicate SKU: " + row.sku()));
                    continue;
                }
                productRepository.save(row.toEntity());
            } catch (Exception e) {
                errors.add(new Error(i, "general", e.getMessage()));
            }
        }
        return new ProductImportResult(rows.size() - errors.size(), errors);
    }
}
```

### Scenario 3: Nested Object Validation with Dynamic Error Collection

An order API accepts an order with a list of items. Each item has its own validation rules. All validation errors for all items must be collected and returned in a single response.

```java
public record CreateOrderRequest(
    @NotNull Long customerId,
    @NotEmpty List<@Valid OrderItemRequest> items,
    @NotNull @Future LocalDateTime deliveryDate
) {}

public record OrderItemRequest(
    @NotNull Long productId,
    @Min(1) @Max(100) int quantity,
    @NotBlank String unit
) {}

@RestControllerAdvice
public class ValidationHandler {
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        Map<String, List<String>> fieldErrors = ex.getBindingResult()
            .getFieldErrors().stream()
            .collect(Collectors.groupingBy(
                fe -> fe.getField(),
                Collectors.mapping(FieldError::getDefaultMessage, Collectors.toList())
            ));
        return new ErrorResponse("VALIDATION_FAILED", "Validation failed", fieldErrors);
    }
}
```

---

## Scenario-Based Questions

1. **Q: Your REST endpoint accepts a JSON payload. You add `@Valid @RequestBody` but validation errors are silently ignored — the method executes with an invalid object. What's wrong?**
   A: Most likely you forgot to handle `MethodArgumentNotValidException` in your `@RestControllerAdvice`. Without a handler, Spring uses its default error response (which may be a 400 with a generic body) but the controller method may still execute if `Errors` / `BindingResult` is the next parameter. Check: (a) Is there a `BindingResult` parameter after `@Valid` — if so, validation errors are stored there and the method executes. (b) Is there a `@RestControllerAdvice` that handles validation exceptions? Always add explicit field-level error handling.

2. **Q: Your DTO has 20 fields. Most are required for CREATE but optional for UPDATE. You don't want to create two separate DTOs. How do you handle this with a single class?**
   A: Use validation groups:
   ```java
   public interface Create {}
   public interface Update {}

   public class UserRequest {
       @Null(groups = Create.class)        // ID must be null for create
       @NotNull(groups = Update.class)     // ID must be present for update
       private Long id;
       @NotBlank(groups = Create.class)
       private String email;               // Optional on update — blank means no change
       @NotBlank(groups = {Create.class, Update.class})
       private String name;                // Required for both
   }

   @PostMapping
   public User create(@Validated(Create.class) @RequestBody UserRequest request) { ... }

   @PutMapping("/{id}")
   public User update(@Validated(Update.class) @RequestBody UserRequest request) { ... }
   ```
   This keeps a single DTO class with operation-specific validation rules.

3. **Q: You validate a `String` field with `@Email`. The field is optional — users can leave it blank. But `@Email` on a blank string returns validation error. How do you make the email optional yet validate format when provided?**
   A: Use `@Email` with `@Size(min = 0)` or handle blank values in the validator. The best approach is to not annotate optional fields with `@NotBlank` and use `@Email` only — the `@Email` validator by default returns `true` for null values but `false` for blank strings. Use `@Pattern` with a regex that allows empty:
   ```java
   @Pattern(regexp = "^$|^[\\w-.]+@([\\w-]+\\.)+[\\w-]{2,4}$",
            message = "Invalid email")
   private String email; // Optional — blank allowed, if provided must be valid email
   ```

4. **Q: Your custom `ConstraintValidator` injects a `UserRepository` to check email uniqueness. The validator works in the controller but throws a `NullPointerException` when used in a `@Validated` service method. Why?**
   A: In `@Validated` service methods, the validation is performed by a `MethodValidationPostProcessor`. The `ConstraintValidator` must be a Spring bean for injection to work. Verify: (a) The validator class is annotated with `@Component`. (b) The `MethodValidationPostProcessor` is configured (auto-configured in Spring Boot). (c) The service class has `@Validated` at the class level. If the validator is not a bean, Hibernate Validator creates it via `new` and dependencies are null.

5. **Q: Your JSON API returns 400 with `{"timestamp": "...", "status": 400, "error": "Bad Request", "path": "..."}` for validation errors. Your frontend team expects `{"field": "email", "message": "Invalid email"}`. How do you customize this?**
   A: Override Spring's default error handling by adding a proper handler in `@RestControllerAdvice`:
   ```java
   @ExceptionHandler(MethodArgumentNotValidException.class)
   @ResponseStatus(HttpStatus.BAD_REQUEST)
   public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
       Map<String, List<String>> fieldErrors = ex.getBindingResult()
           .getFieldErrors().stream()
           .collect(Collectors.groupingBy(
               FieldError::getField,
               Collectors.mapping(FieldError::getDefaultMessage, Collectors.toList())
           ));
       return new ErrorResponse("VALIDATION_FAILED", "Input validation failed", fieldErrors);
   }
   ```
   Set `server.error.include-message=never` to suppress Spring Boot's default error attributes and ensure only your custom handler returns error responses.

6. **Q: You validate a list of items with `List<@Valid OrderItem> items`. The validation works, but for an item with `null` fields, the error message is "must not be null" without indicating which item in the list failed. How do you include the item index?**
   A: The default error message doesn't include the list index. Customize by using a custom validator that adds the index to the property path, or extract it in the handler:
   ```java
   @ExceptionHandler(MethodArgumentNotValidException.class)
   public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
       Map<String, List<String>> errors = ex.getBindingResult()
           .getFieldErrors().stream()
           .collect(Collectors.groupingBy(
               fe -> enhanceFieldPath(fe),  // Convert "items[].name" to "items[3].name"
               Collectors.mapping(FieldError::getDefaultMessage, Collectors.toList())
           ));
       return new ErrorResponse("VALIDATION_FAILED", "Validation failed", errors);
   }

   private String enhanceFieldPath(FieldError fe) {
       // fe.getField() returns "items[0].name" including index
       return fe.getField();
   }
   ```
   The `FieldError.getField()` already includes the index in the format `items[0].name`.

7. **Q: Your Spring Boot application uses `spring-boot-starter-web` but validation annotations like `@NotBlank` are ignored — the application compiles but validation never runs. What's missing?**
   A: The `spring-boot-starter-web` does NOT include `spring-boot-starter-validation`. You must add it explicitly:
   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-validation</artifactId>
   </dependency>
   ```
   Without this dependency, Jakarta Bean Validation is not on the classpath, and `@Valid` effectively does nothing. Spring Boot 2.3+ made validation an opt-in dependency. Always check that `hibernate-validator` is on the classpath.

8. **Q: You have a DTO with 10 fields, each with multiple validation annotations. The error response is huge — every field has 3-4 error messages. You want Hibernate Validator to fail fast — stop validation at the first error. How?**
   A: Configure `FailFast` in the validator:
   ```java
   @Bean
   public Validator validator() {
       return Validation.byProvider(HibernateValidator.class)
           .configure()
           .failFast(true)  // Stop at first violation
           .buildValidatorFactory()
           .getValidator();
   }
   ```
   Or, for Spring Boot:
   ```yaml
   spring:
     jpa:
       properties:
         javax:
           validation:
             fail-fast: true
   ```
   Be aware: this returns only the first error, which may frustrate API clients that want all errors at once. Consider your API contract carefully.

9. **Q: You need to validate a request parameter that is a complex object. The validation errors should trigger a 400 with field-level details, but the default Spring behavior returns 500. How do you handle this?**
   A: For `@RequestParam` validation of simple types, add `@Validated` at the controller class level and use validation annotations on parameters. Handle `ConstraintViolationException` in `@RestControllerAdvice`:
   ```java
   @RestController
   @Validated
   public class SearchController {
       @GetMapping("/search")
       public List<Result> search(
               @RequestParam @Size(min = 3, max = 100) String query,
               @RequestParam @Min(0) int page,
               @RequestParam @Min(1) @Max(100) int size) { ... }
   }
   ```
   Handle `ConstraintViolationException` separately from `MethodArgumentNotValidException` — they have different structures.

10. **Q: Your microservice receives a request with a nested object that has its own validation. The nested object's validation passes, but a business rule requires a relationship between two nested objects (e.g., `shippingAddress.country` must match `billingAddress.country`). How do you validate this?**
    A: Use a class-level custom constraint on the root object:
    ```java
    @Target(TYPE)
    @Retention(RUNTIME)
    @Constraint(validatedBy = CountryMatchValidator.class)
    public @interface CountriesMatch {
        String message() default "Shipping and billing countries must match";
        Class<?>[] groups() default {};
        Class<? extends Payload>[] payload() default {};
    }

    @Component
    public class CountryMatchValidator
            implements ConstraintValidator<CountriesMatch, OrderRequest> {
        @Override
        public boolean isValid(OrderRequest request, ConstraintValidatorContext ctx) {
            if (request.shippingAddress() == null || request.billingAddress() == null) {
                return true; // Let @NotNull handle null checks
            }
            return Objects.equals(
                request.shippingAddress().country(),
                request.billingAddress().country());
        }
    }
    ```
    Class-level constraints have access to the entire object and can validate cross-field rules that span nested objects.

---

## Interview Questions

1. **What is the difference between `@Valid` and `@Validated`?** 
   A: `@Valid` is the Jakarta Bean Validation standard annotation that triggers validation. `@Validated` is Spring's variant that additionally supports validation groups. Use `@Valid` for simple cases and `@Validated` when you need to specify different validation rules for different operations (e.g., create vs update).

2. **What Bean Validation annotations does Spring Boot support?** 
   A: Core annotations: `@NotNull`, `@NotEmpty`, `@NotBlank`, `@Size`, `@Min`/`@Max`, `@Positive`/`@Negative`, `@Email`, `@Pattern`, `@Past`/`@Future`, `@Digits`, `@AssertTrue`/`@AssertFalse`. Custom validators can be created with `@Constraint`.

3. **What is the difference between `@NotNull`, `@NotEmpty`, and `@NotBlank`?** 
   A: `@NotNull` — value is not null. `@NotEmpty` — value is not null AND has at least one element (for strings: length > 0; for collections: size > 0). `@NotBlank` — value is not null AND contains at least one non-whitespace character (strings only). Use `@NotBlank` for string fields that must have meaningful content.

4. **How do you create a custom validator?** 
   A: (1) Create an annotation with `@Constraint(validatedBy = YourValidator.class)`. (2) Implement `ConstraintValidator<YourAnnotation, FieldType>`. (3) Annotate the implementation with `@Component` if it needs dependency injection. (4) Use the annotation on fields/classes.

5. **What are validation groups and when do you use them?** 
   A: Validation groups allow different validation rules for different operations. Define marker interfaces (`Create`, `Update`), annotate fields with `groups = {Create.class}`, and use `@Validated(Create.class)` at the controller. Use groups when the same DTO has different validation rules for create vs update.

6. **How do you validate path variables and request parameters?** 
   A: Add `@Validated` at the controller class level. Annotate method parameters with validation annotations (`@Min`, `@Size`). Handle `ConstraintViolationException` in `@RestControllerAdvice`. This is separate from `@RequestBody` validation.

7. **How do you validate nested objects in a request body?** 
   A: Annotate the nested field with `@Valid` to trigger validation of its fields:
   ```java
   public record OrderRequest(@Valid @NotNull Address shippingAddress) {}
   public record Address(@NotBlank String street, @NotBlank String city) {}
   ```
   Without `@Valid`, the nested object's validation annotations are ignored.

8. **How do you internationalize validation error messages?** 
   A: Create `ValidationMessages.properties` (and locale-specific variants like `ValidationMessages_fr.properties`) with keys like `javax.validation.constraints.NotBlank.message = {0} must not be blank`. Or use `message = "{my.custom.key}"` per annotation and define the key in a `messages.properties` file.

9. **How do you handle validation errors in `@RestControllerAdvice`?** 
   A: Handle `MethodArgumentNotValidException` for `@RequestBody` validation, `ConstraintViolationException` for parameter/path validation. Extract `FieldError` details and return a structured response with field-level error messages. Return HTTP 400.

10. **What is `@AssertTrue` and how do you use it for cross-field validation?** 
    A: `@AssertTrue` on a boolean method validates a custom condition. The method has access to all fields of the object, enabling cross-field validation (e.g., `password == confirmPassword`). Use it for simple cross-field rules within a single class. For complex rules, use class-level `@Constraint`.

---

## Developer Recommendations

- **Always add `spring-boot-starter-validation` as a dependency** — Since Spring Boot 2.3, validation is no longer included in `spring-boot-starter-web`. Without it, `@Valid` and validation annotations are silently ignored. Always verify the dependency is present.
- **Use DTOs for request/response, never entities** — JPA entities have lifecycle callbacks, lazy associations, and JPA constraints that are inappropriate for API boundaries. A separate DTO decouples the API contract from the database model and prevents `LazyInitializationException`.
- **Use validation groups for create vs update semantics** — A single DTO class with groups avoids duplication while supporting different rules: `Null` for ID on create, `NotNull` on update. Without groups, you'd need separate DTO classes or nullable fields everywhere.
- **Always handle `MethodArgumentNotValidException` in `@RestControllerAdvice`** — Without a custom handler, Spring returns a generic 400 with a default structure. API clients need field-level error details to fix invalid input. Return `field → [error messages]` mapping.
- **Use `@Validated` at the service layer for database-backed validation** — Annotations can check format but not existence (email uniqueness, category existence). Service-layer validation with `@Validated` and injected repositories provides database-aware validation while keeping controllers thin.
- **Use class-level `@Constraint` for cross-field validation** — `@AssertTrue` works for simple cases but produces unclear error paths. A class-level custom constraint can add contextual error messages on multiple fields simultaneously using `ConstraintValidatorContext.buildConstraintViolationWithTemplate()`.
- **Keep validation annotations on the DTO, not on entity fields** — Entity validation constraints (`@NotNull` on a database column) may differ from API validation (optional fields on update). Mixing them causes confusion. The DTO defines the API contract; the entity defines the database contract.
- **Use `@Pattern` with regex for format validation beyond what `@Email` provides** — `@Email` follows Jakarta's strict regex which rejects some valid email addresses. For custom format rules, use `@Pattern` with a regex that suits your needs. Document the regex pattern so clients can replicate it.
