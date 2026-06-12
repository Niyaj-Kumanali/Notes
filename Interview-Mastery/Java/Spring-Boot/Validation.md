# Spring Boot Validation

---

## What is Spring Boot Validation?

**Spring Boot Validation** provides declarative input validation using the **Jakarta Bean Validation API** (formerly JSR 380 / Jakarta Validation). It integrates with Spring MVC to automatically validate request bodies, request parameters, path variables, and nested objects, reducing boilerplate validation code.

### Key Concepts:

- **Core Dependencies:** Spring Boot includes validation support through `spring-boot-starter-validation`. Since Spring Boot 2.3, validation is no longer included in `spring-boot-starter-web`, so you must add this dependency explicitly.

  ```xml
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>
  ```

- **Bean Validation Annotations:** The Jakarta Bean Validation API provides a comprehensive set of annotations including `@NotNull` (value must not be null), `@NotEmpty` (must have at least one element), `@NotBlank` (must contain non-whitespace), `@Size(min, max)`, `@Min`/`@Max`, `@Positive`/`@Negative`, `@Email`, `@Pattern(regexp)`, `@Past`/`@Future`, `@Digits`, and `@AssertTrue`/`@AssertFalse`. These cover null checks, size constraints, numeric ranges, format validation, and temporal assertions.

- **Validation Groups:** Validation groups allow different validation rules for different operations (e.g., create vs update) using the same DTO class. Define marker interfaces like `Create` and `Update`, annotate fields with `groups = {Create.class}`, and use `@Validated(Create.class)` at the controller. This avoids duplicating DTOs for each operation while supporting operation-specific constraints.

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

### Request Validation in Controller

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

### Validation Error Handling

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

### Custom Validator

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

### Path Variable and Query Parameter Validation

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

- **Not adding `@Valid` or `@Validated` to request body parameters** — Validation annotations on the DTO are ignored if the controller parameter is not annotated, and the method receives an invalid object without any error. This *looks correct* because the DTO has `@NotNull` and `@Size` annotations, and the code compiles — there is no warning indicating the validation is disabled. Always annotate `@RequestBody` parameters with `@Valid` or `@Validated`.

- **Using entities as request/response DTOs** — JPA entities have lifecycle callbacks, lazy-loaded associations, and persistence constraints that are inappropriate for API boundaries. This *looks correct* because entity fields match the API fields one-to-one — creating a separate DTO feels like duplicate work, and the application works correctly with small datasets. Always use separate DTOs or records to decouple the API contract from the database model.

- **No global validation exception handler** — Without a handler for `MethodArgumentNotValidException`, Spring returns a generic 400 with a default error message. This *looks correct* because Spring Boot returns a valid 400 response — the error is conveyed, just without field-level detail. The missing field names only matter to API clients building rich error UIs. Add a `@RestControllerAdvice` with structured field-level error responses for consistent API error formatting.

- **Not validating nested objects** — Nested objects in a request DTO are not validated unless the field is annotated with `@Valid`. This *looks correct* because the nested object has `@NotNull` annotations on its fields — developers expect cascading validation to happen automatically, but Jakarta Bean Validation requires explicit `@Valid` on the parent field. Without this annotation, validation annotations on nested object fields are silently skipped.

- **Ignoring validation groups** — Without groups, the same validation rules apply to both create and update operations. This *looks correct* because a single set of validation rules works for both operations during development — the issue only appears when an ID field is required on update but must be null on create, or when optional fields on update are incorrectly required. Use groups to differentiate rules, such as ID being null on create but not null on update.

---

## Real-World Scenarios

### Scenario 1: User Registration with Cross-Field Validation

A registration form requires `password` and `confirmPassword` to match. The `email` must be unique (checked against the database). The `birthDate` must indicate age >= 18.

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

A product import service receives bulk CSV data. Validation must check that category names exist in the database, SKUs are unique, and prices are non-negative. These constraints require database queries.

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

- **Q: Your REST endpoint accepts a JSON payload. You add `@Valid @RequestBody` but validation errors are silently ignored — the method executes with an invalid object. What's wrong?**
  A: Most likely you forgot to handle `MethodArgumentNotValidException` in your `@RestControllerAdvice`, or there is a `BindingResult` parameter after `@Valid` that stores errors and allows the method to execute. Always add explicit field-level error handling in a `@RestControllerAdvice` and remove any `BindingResult` parameter if you want automatic error responses.

  > **Interview follow-up:** The candidate identified the `BindingResult` parameter as the cause. If the method signature is `create(@Valid @RequestBody UserRequest request, BindingResult result)` and 5 out of 10 fields fail validation, the method still executes with a partially valid object. The service layer persists the valid fields but silently ignores invalid ones. How would you detect this data integrity risk in code review or in CI?

- **Q: Your DTO has 20 fields. Most are required for CREATE but optional for UPDATE. You don't want to create two separate DTOs. How do you handle this with a single class?**
  A: Use validation groups with marker interfaces `Create` and `Update`. Annotate fields with `groups = Create.class` for create-only rules, `groups = Update.class` for update-only rules, and `groups = {Create.class, Update.class}` for fields required in both operations. Use `@Validated(Create.class)` and `@Validated(Update.class)` at the controller methods.

- **Q: You validate a `String` field with `@Email`. The field is optional — users can leave it blank. But `@Email` on a blank string returns validation error. How do you make the email optional yet validate format when provided?**
  A: The `@Email` validator returns `true` for null values but `false` for blank strings. Do not annotate optional fields with `@NotBlank` and rely on `@Email` alone for null values. Alternatively, use `@Pattern` with a regex that allows empty strings as valid input.

- **Q: Your custom `ConstraintValidator` injects a `UserRepository` to check email uniqueness. The validator works in the controller but throws a `NullPointerException` when used in a `@Validated` service method. Why?**
  A: In `@Validated` service methods, the `ConstraintValidator` must be a Spring bean for injection to work. Verify the validator class is annotated with `@Component`, the `MethodValidationPostProcessor` is configured (auto-configured in Spring Boot), and the service class has `@Validated` at the class level. Without `@Component`, Hibernate Validator creates the validator via `new` and dependencies are null.

- **Q: Your JSON API returns 400 with `{"timestamp": "...", "status": 400, "error": "Bad Request", "path": "..."}` for validation errors. Your frontend team expects `{"field": "email", "message": "Invalid email"}`. How do you customize this?**
  A: Override Spring's default error handling by adding a handler in `@RestControllerAdvice` for `MethodArgumentNotValidException` that extracts field errors from the `BindingResult` and returns a structured response with field-level error messages. Set `server.error.include-message=never` to suppress Spring Boot's default error attributes.

- **Q: You validate a list of items with `List<@Valid OrderItem> items`. The validation works, but for an item with `null` fields, the error message is "must not be null" without indicating which item in the list failed. How do you include the item index?**
  A: The `FieldError.getField()` method already includes the index in the format `items[0].name`. In your `@RestControllerAdvice` handler, use the field path as returned by `getField()` which includes the list index, making it clear which item in the collection has the validation error.

- **Q: Your Spring Boot application uses `spring-boot-starter-web` but validation annotations like `@NotBlank` are ignored — the application compiles but validation never runs. What's missing?**
  A: Since Spring Boot 2.3, `spring-boot-starter-web` does NOT include `spring-boot-starter-validation`. You must add it explicitly as a dependency. Without it, Jakarta Bean Validation is not on the classpath and `@Valid` effectively does nothing.

- **Q: You have a DTO with 10 fields, each with multiple validation annotations. The error response is huge — every field has 3-4 error messages. You want Hibernate Validator to fail fast — stop validation at the first error. How?**
  A: Configure `FailFast` by creating a `Validator` bean with `Validation.byProvider(HibernateValidator.class).configure().failFast(true)`. Be aware this returns only the first error, which may frustrate API clients that want all errors at once, so consider your API contract carefully before enabling this.

  > **Interview follow-up:** The candidate knows about `failFast`. If you enable `failFast`, the client fixes the first error and resubmits, only to discover the second error, then resubmits again to find the third. This round-trip cycle slows development. What alternative approach provides all errors in one response while still bounding the total number of error messages returned?

- **Q: You need to validate a request parameter that is a complex object. The validation errors should trigger a 400 with field-level details, but the default Spring behavior returns 500. How do you handle this?**
  A: For `@RequestParam` validation of simple types, add `@Validated` at the controller class level and use validation annotations on method parameters. Handle `ConstraintViolationException` in `@RestControllerAdvice` separately from `MethodArgumentNotValidException` since they have different exception structures.

- **Q: Your microservice receives a request with a nested object that has its own validation. The nested object's validation passes, but a business rule requires a relationship between two nested objects (e.g., `shippingAddress.country` must match `billingAddress.country`). How do you validate this?**
  A: Use a class-level custom constraint on the root object. Class-level constraints have access to the entire object and can validate cross-field rules that span nested objects. Implement `ConstraintValidator` with the root request type and access all fields to compare values across nested objects.

---

## Interview Questions

- **What is the difference between `@Valid` and `@Validated`?**
  A: `@Valid` is the Jakarta Bean Validation standard annotation that triggers validation. `@Validated` is Spring's variant that additionally supports validation groups. Use `@Valid` for simple cases and `@Validated` when you need different validation rules for different operations (e.g., create vs update).

- **What Bean Validation annotations does Spring Boot support?**
  A: Core annotations include `@NotNull`, `@NotEmpty`, `@NotBlank`, `@Size`, `@Min`/`@Max`, `@Positive`/`@Negative`, `@Email`, `@Pattern`, `@Past`/`@Future`, `@Digits`, and `@AssertTrue`/`@AssertFalse`. Custom validators can be created with `@Constraint` for application-specific rules.

- **What is the difference between `@NotNull`, `@NotEmpty`, and `@NotBlank`?**
  A: `@NotNull` checks the value is not null. `@NotEmpty` checks the value is not null AND has at least one element (for strings: length > 0; for collections: size > 0). `@NotBlank` checks the value is not null AND contains at least one non-whitespace character (strings only). Use `@NotBlank` for string fields that must have meaningful content.

- **How do you create a custom validator?**
  A: Create an annotation with `@Constraint(validatedBy = YourValidator.class)`, implement `ConstraintValidator<YourAnnotation, FieldType>`, annotate the implementation with `@Component` if it needs dependency injection, and use the annotation on fields or classes.

- **What are validation groups and when do you use them?**
  A: Validation groups allow different validation rules for different operations. Define marker interfaces (`Create`, `Update`), annotate fields with `groups = {Create.class}`, and use `@Validated(Create.class)` at the controller. Use groups when the same DTO has different validation rules for create vs update operations.

- **How do you validate path variables and request parameters?**
  A: Add `@Validated` at the controller class level and annotate method parameters with validation annotations (`@Min`, `@Size`). Handle `ConstraintViolationException` in `@RestControllerAdvice`. This validation is separate from `@RequestBody` validation.

- **How do you validate nested objects in a request body?**
  A: Annotate the nested field with `@Valid` to trigger validation of its fields. Without `@Valid`, the nested object's validation annotations are ignored even if the parent object is validated.

- **How do you internationalize validation error messages?**
  A: Create `ValidationMessages.properties` with locale-specific variants like `ValidationMessages_fr.properties` using keys such as `javax.validation.constraints.NotBlank.message`. Alternatively, define custom message keys per annotation and resolve them in a `messages.properties` file.

- **How do you handle validation errors in `@RestControllerAdvice`?**
  A: Handle `MethodArgumentNotValidException` for `@RequestBody` validation and `ConstraintViolationException` for parameter/path validation. Extract `FieldError` details from the binding result and return a structured response with field-level error messages and HTTP 400 status.

- **What is `@AssertTrue` and how do you use it for cross-field validation?**
  A: `@AssertTrue` on a boolean method validates a custom condition with access to all fields of the object, enabling cross-field validation like password confirmation matching. Use it for simple cross-field rules within a single class; for complex rules spanning multiple objects, use a class-level `@Constraint`.

---

## Developer Recommendations

- **Always add `spring-boot-starter-validation` as a dependency** — Since Spring Boot 2.3, validation is no longer included in `spring-boot-starter-web`. Without it, `@Valid` and validation annotations are silently ignored, so verify the dependency is present in your build file.

- **Use DTOs for request/response, never entities** — JPA entities have lifecycle callbacks, lazy associations, and JPA constraints that are inappropriate for API boundaries. A separate DTO decouples the API contract from the database model and prevents `LazyInitializationException` during serialization.

- **Use validation groups for create vs update semantics** — A single DTO class with groups avoids duplication while supporting different rules: `Null` for ID on create, `NotNull` on update. Without groups, you would need separate DTO classes or nullable fields everywhere.

- **Always handle `MethodArgumentNotValidException` in `@RestControllerAdvice`** — Without a custom handler, Spring returns a generic 400 with a default structure. API clients need field-level error details to fix invalid input, so return a `field` to `[error messages]` mapping.

- **Use `@Validated` at the service layer for database-backed validation** — Annotations can check format but not existence (email uniqueness, category existence). Service-layer validation with `@Validated` and injected repositories provides database-aware validation while keeping controllers thin.

- **Use class-level `@Constraint` for cross-field validation** — `@AssertTrue` works for simple cases but produces unclear error paths. A class-level custom constraint can add contextual error messages on multiple fields simultaneously using `ConstraintValidatorContext.buildConstraintViolationWithTemplate()`.

- **Keep validation annotations on the DTO, not on entity fields** — Entity validation constraints like `@NotNull` on a database column may differ from API validation where fields are optional on update. The DTO defines the API contract; the entity defines the database contract.

- **Use `@Pattern` with regex for format validation beyond what `@Email` provides** — The `@Email` annotation follows Jakarta's strict regex that rejects some valid email addresses. For custom format rules, use `@Pattern` with a regex that suits your needs and document the pattern so clients can replicate it.
