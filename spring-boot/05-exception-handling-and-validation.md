# 05 — Exception Handling & Validation

## 1. Default error handling

Without any handler, Boot's `BasicErrorController` (mapped to `/error`) returns:
```json
{ "timestamp": "2026-10-04T10:15:30Z", "status": 500, "error": "Internal Server Error", "path": "/api/orders/42" }
```
Stack traces and messages are hidden by default (`server.error.include-message=never`,
`include-stacktrace=never`). Don't turn those on in production.

## 2. `@ExceptionHandler` (one controller)

```java
@RestController
class OrderController {
    @GetMapping("/orders/{id}") Order get(@PathVariable Long id) { ... }

    @ExceptionHandler(OrderNotFoundException.class)
    ResponseEntity<String> handle(OrderNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(ex.getMessage());
    }
}
```

## 3. `@RestControllerAdvice` (global) with `ProblemDetail`

`ProblemDetail` (Spring 6+) implements **RFC 9457** "Problem Details for HTTP APIs", a standard
JSON error format (`application/problem+json`).

```java
public class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(Long id) { super("Order " + id + " not found"); }
}

@RestControllerAdvice                         // = @ControllerAdvice + @ResponseBody
class GlobalExceptionHandler extends ResponseEntityExceptionHandler {   // handles Spring MVC's own exceptions

    @ExceptionHandler(OrderNotFoundException.class)
    ProblemDetail handleNotFound(OrderNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("Order not found");
        pd.setType(URI.create("https://api.example.com/errors/order-not-found"));
        return pd;
    }

    @ExceptionHandler(OptimisticLockingFailureException.class)
    ProblemDetail handleConflict(OptimisticLockingFailureException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT,
                "The resource was modified by someone else. Reload and retry.");
    }

    @ExceptionHandler(Exception.class)          // last-resort catch-all
    ProblemDetail handleUnexpected(Exception ex) {
        log.error("Unhandled error", ex);       // log details, return a generic message
        return ProblemDetail.forStatusAndDetail(HttpStatus.INTERNAL_SERVER_ERROR,
                "Something went wrong");
    }

    @Override   // customize 400 for @Valid failures
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex, HttpHeaders headers,
            HttpStatusCode status, WebRequest request) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(status, "Validation failed");
        Map<String, String> errors = new LinkedHashMap<>();
        ex.getBindingResult().getFieldErrors()
          .forEach(fe -> errors.put(fe.getField(), fe.getDefaultMessage()));
        pd.setProperty("errors", errors);
        return ResponseEntity.status(status).body(pd);
    }
}
```
Response:
```json
{
  "type": "about:blank",
  "title": "Bad Request",
  "status": 400,
  "detail": "Validation failed",
  "instance": "/api/orders",
  "errors": { "customerId": "must not be null", "lines[0].quantity": "must be greater than 0" }
}
```
Enable `ProblemDetail` for Spring's built-in exceptions without writing an advice:
`spring.mvc.problemdetails.enabled=true`.

Alternatives for mapping exceptions to statuses:
```java
@ResponseStatus(HttpStatus.NOT_FOUND)                // on the exception class
class ProductNotFoundException extends RuntimeException { }

throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Product not found");   // ad hoc

class InsufficientStockException extends ErrorResponseException {   // carries its own ProblemDetail
    InsufficientStockException(String sku) {
        super(HttpStatus.UNPROCESSABLE_ENTITY,
              ProblemDetail.forStatusAndDetail(HttpStatus.UNPROCESSABLE_ENTITY, "Out of stock: " + sku),
              null);
    }
}
```

Scope advice: `@RestControllerAdvice(basePackages = "com.example.api")` or
`assignableTypes = {...}`. Order multiple advices with `@Order`. The **most specific** exception
handler wins.

### Exception handling design tips
- A small hierarchy of domain exceptions (`NotFoundException`, `ConflictException`,
  `BusinessRuleException`) mapped centrally.
- Never return stack traces or SQL errors to clients; log them with a correlation ID and return
  that ID.
- Exceptions thrown in **filters** (e.g. Spring Security) don't reach `@ControllerAdvice`. Handle
  them with `AuthenticationEntryPoint`/`AccessDeniedHandler` or a filter-level handler.

## 4. Bean Validation (Jakarta Validation)

Add `spring-boot-starter-validation` (Hibernate Validator). Not included in the web starter since
Boot 2.3.

### Common constraints
| Constraint | Meaning |
|---|---|
| `@NotNull` | Not null |
| `@NotEmpty` | Not null and size > 0 (strings, collections) |
| `@NotBlank` | Not null and at least one non-whitespace char (strings) |
| `@Size(min, max)` | Length / collection size |
| `@Min`, `@Max`, `@Positive`, `@PositiveOrZero`, `@DecimalMin` | Numbers |
| `@Email`, `@Pattern(regexp)` | Format |
| `@Past`, `@Future`, `@PastOrPresent` | Dates |
| `@Valid` | Cascade validation into a nested object / collection elements |

```java
public record CreateUserRequest(
        @NotBlank @Size(max = 100) String name,
        @NotBlank @Email String email,
        @NotNull @Past LocalDate birthDate,
        @Valid @NotNull Address address,
        @Size(max = 5) List<@NotBlank String> tags) {}

@PostMapping("/users")
UserResponse create(@Valid @RequestBody CreateUserRequest req) { ... }
// invalid → MethodArgumentNotValidException → 400
```

### Validating path variables, params and service methods
```java
@RestController
@Validated                                      // enables method-level validation
class UserController {
    @GetMapping("/users/{id}")
    UserResponse get(@PathVariable @Positive Long id) { ... }
}

@Service
@Validated
class TransferService {
    void transfer(@NotNull Long from, @NotNull Long to, @Positive BigDecimal amount) { ... }
    // violation → ConstraintViolationException (map it to 400 in your advice)
}
```
Since Spring 6.1, built-in method validation on controllers raises
`HandlerMethodValidationException` for constraints directly on parameters.

### `@Valid` vs `@Validated`
| `@Valid` (Jakarta) | `@Validated` (Spring) |
|---|---|
| Triggers validation of an argument / cascades into nested objects | Same on arguments, **plus** supports validation **groups** |
| Can be used on fields for nesting | On a class: enables AOP-based method validation |

### Validation groups
```java
interface OnCreate {}
interface OnUpdate {}

record ProductRequest(
        @Null(groups = OnCreate.class) @NotNull(groups = OnUpdate.class) Long id,
        @NotBlank(groups = {OnCreate.class, OnUpdate.class}) String name) {}

@PostMapping UserResponse create(@Validated(OnCreate.class) @RequestBody ProductRequest r) { ... }
@PutMapping  UserResponse update(@Validated(OnUpdate.class) @RequestBody ProductRequest r) { ... }
```

### Custom constraint
```java
@Target({ElementType.FIELD, ElementType.PARAMETER, ElementType.RECORD_COMPONENT})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = SkuValidator.class)
public @interface ValidSku {
    String message() default "invalid SKU format";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class SkuValidator implements ConstraintValidator<ValidSku, String> {
    private static final Pattern SKU = Pattern.compile("[A-Z]{3}-\\d{4}");
    @Override public boolean isValid(String value, ConstraintValidatorContext ctx) {
        return value == null || SKU.matcher(value).matches();   // null handled by @NotNull
    }
}
```
Validators are Spring beans, so they can inject repositories (e.g. "email must be unique"), but
uniqueness should still be enforced by a DB unique constraint (race conditions).

### Cross-field validation
Put a class-level constraint on the DTO (e.g. `@PasswordsMatch`) whose validator receives the
whole object, or use a `@AssertTrue boolean isDateRangeValid()` method on the record.

### Custom messages / i18n
```java
@NotBlank(message = "{user.name.required}") String name
```
`src/main/resources/ValidationMessages.properties` → `user.name.required=Name is required`.

## 5. Which exception for which failure?

| Situation | Exception | Default status |
|---|---|---|
| `@Valid @RequestBody` fails | `MethodArgumentNotValidException` | 400 |
| `@Validated` method param fails (class-level) | `ConstraintViolationException` | 500 unless handled |
| Parameter constraint in controller (6.1+) | `HandlerMethodValidationException` | 400 |
| Malformed JSON | `HttpMessageNotReadableException` | 400 |
| Wrong type (`/orders/abc`) | `MethodArgumentTypeMismatchException` | 400 |
| Missing required param | `MissingServletRequestParameterException` | 400 |
| Wrong HTTP method | `HttpRequestMethodNotSupportedException` | 405 |
| Unsupported `Content-Type` | `HttpMediaTypeNotSupportedException` | 415 |
| No handler | `NoResourceFoundException` (6.1+) | 404 |

## 6. Gotchas

1. Forgetting `spring-boot-starter-validation` → annotations are silently ignored.
2. Missing `@Valid` on nested objects/collections → nested fields aren't validated.
3. `ConstraintViolationException` from `@Validated` services becomes a 500 unless you map it.
4. `@NotNull` on a primitive `int` is meaningless (never null; defaults to 0).
5. Catch-all `@ExceptionHandler(Exception.class)` can swallow Spring's `ResponseStatusException`s
   (e.g. 404s) if you don't extend `ResponseEntityExceptionHandler` or handle them first.
6. Security exceptions don't reach `@ControllerAdvice`.

## 7. Interview questions

1. **How do you handle exceptions globally in Spring Boot?**
   A `@RestControllerAdvice` class with `@ExceptionHandler` methods mapping exceptions to responses
   (ideally `ProblemDetail`); optionally extend `ResponseEntityExceptionHandler` to customize Spring
   MVC's built-in exceptions.

2. **`@ControllerAdvice` vs `@RestControllerAdvice`?**
   `@RestControllerAdvice` adds `@ResponseBody`, so handler return values are serialized as the body.

3. **What is `ProblemDetail`?**
   Spring's implementation of RFC 9457: a standard error body with `type`, `title`, `status`,
   `detail`, `instance` and custom properties.

4. **Ways to map an exception to an HTTP status?**
   `@ExceptionHandler` in an advice, `@ResponseStatus` on the exception class, throwing
   `ResponseStatusException`, or extending `ErrorResponseException`.

5. **What happens if an exception is thrown and there's no handler?**
   It propagates to the servlet container, which forwards to `/error`; `BasicErrorController`
   returns the default error JSON with status 500 (or the status from `@ResponseStatus`).

6. **How does validation work in Spring Boot?**
   Hibernate Validator implements Jakarta Bean Validation. `@Valid`/`@Validated` on a controller
   argument triggers validation during argument resolution; failures throw
   `MethodArgumentNotValidException` → 400.

7. **`@Valid` vs `@Validated`?**
   `@Valid` is the standard annotation and supports cascading. `@Validated` is Spring's and supports
   groups and class-level method validation.

8. **`@NotNull` vs `@NotEmpty` vs `@NotBlank`?**
   Not null; not null and not empty; not null and contains non-whitespace (strings only).

9. **How do you create a custom validator?**
   Define a constraint annotation with `@Constraint(validatedBy = ...)` and implement
   `ConstraintValidator<A, T>.isValid`.

10. **How do you validate request params and path variables?**
    Put constraints on the parameters. In Spring 6.1+ controller method validation is built in;
    before that, add `@Validated` on the controller class and handle `ConstraintViolationException`.

11. **How do you validate differently for create vs update?**
    Validation groups with `@Validated(OnCreate.class)`.

12. **How do you handle exceptions thrown in filters?**
    They don't reach controller advice. Handle in the filter, or use Spring Security's
    `AuthenticationEntryPoint`/`AccessDeniedHandler`, or delegate to `HandlerExceptionResolver`.

13. **Should you return the exception message to the client?**
    Only for expected, safe business errors. For unexpected errors return a generic message plus a
    correlation ID, and log the details server-side.
