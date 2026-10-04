# 04 — REST APIs with Spring MVC

## 1. How a request is processed: `DispatcherServlet`

Spring MVC is built around the **front controller** pattern: one servlet, `DispatcherServlet`,
receives every request and delegates.

```
Client
  │  GET /api/orders/42
  ▼
Filter chain (Security, CORS, request logging)            ← servlet level
  ▼
DispatcherServlet
  1. HandlerMapping           → which controller method? (RequestMappingHandlerMapping)
  2. HandlerInterceptor.preHandle()
  3. HandlerAdapter           → invokes the method
       - HandlerMethodArgumentResolvers build arguments
         (@PathVariable, @RequestParam, @RequestBody via HttpMessageConverter, @Valid)
  4. Controller method returns an object / ResponseEntity
  5. HttpMessageConverter (Jackson) writes JSON           ← @ResponseBody / @RestController
     (or a ViewResolver renders a template for @Controller + view name)
  6. HandlerInterceptor.postHandle() / afterCompletion()
  7. Exceptions → HandlerExceptionResolver → @ExceptionHandler / @ControllerAdvice
  ▼
Response
```

## 2. A complete REST controller

```java
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
class OrderController {

    private final OrderService service;

    @GetMapping                                          // GET /api/orders?status=NEW&page=0&size=20
    Page<OrderResponse> list(@RequestParam(required = false) OrderStatus status,
                             Pageable pageable) {
        return service.find(status, pageable);
    }

    @GetMapping("/{id}")                                 // GET /api/orders/42
    OrderResponse get(@PathVariable Long id) {
        return service.get(id);                          // throws OrderNotFoundException → 404
    }

    @PostMapping                                         // POST /api/orders
    ResponseEntity<OrderResponse> create(@Valid @RequestBody CreateOrderRequest req,
                                         UriComponentsBuilder uri) {
        OrderResponse created = service.create(req);
        URI location = uri.path("/api/orders/{id}").buildAndExpand(created.id()).toUri();
        return ResponseEntity.created(location).body(created);   // 201 + Location header
    }

    @PutMapping("/{id}")                                 // full replace
    OrderResponse update(@PathVariable Long id, @Valid @RequestBody UpdateOrderRequest req) {
        return service.update(id, req);
    }

    @PatchMapping("/{id}/status")                        // partial update
    OrderResponse changeStatus(@PathVariable Long id, @RequestBody StatusChange change) {
        return service.changeStatus(id, change.status());
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)               // 204
    void delete(@PathVariable Long id) {
        service.delete(id);
    }
}

record CreateOrderRequest(@NotNull Long customerId,
                          @NotEmpty List<@Valid OrderLine> lines) {}
record OrderLine(@NotBlank String sku, @Positive int quantity) {}
record OrderResponse(Long id, OrderStatus status, BigDecimal total, Instant createdAt) {}
```

**Use DTOs, not entities, in the API.** Exposing JPA entities leaks internal structure, causes
lazy-loading exceptions and infinite recursion during serialization, and couples API and schema.
Map with plain code, records, or MapStruct.

## 3. Request mapping annotations and argument binding

| Annotation | Binds |
|---|---|
| `@PathVariable` | `/orders/{id}` segment |
| `@RequestParam` | Query string / form field (`required`, `defaultValue`) |
| `@RequestBody` | Body deserialized by an `HttpMessageConverter` (Jackson for JSON) |
| `@RequestHeader` | A header (`@RequestHeader("X-Request-Id") String id`) |
| `@CookieValue` | A cookie |
| `@ModelAttribute` | Query/form params bound onto an object (good for search filters) |
| `@RequestPart` | A part of a multipart request |
| `@AuthenticationPrincipal` | The current user (Spring Security) |
| `Pageable`, `Sort` | `?page=0&size=20&sort=createdAt,desc` (Spring Data web support) |
| `HttpServletRequest`, `Principal`, `Locale` | Injected directly |

```java
@GetMapping(path = "/search",
            params = "q",                                 // only when ?q is present
            headers = "X-Client=mobile",
            produces = MediaType.APPLICATION_JSON_VALUE)
List<Product> search(@ModelAttribute ProductFilter filter) { ... }
```
`@GetMapping` = `@RequestMapping(method = GET)`; similarly `@PostMapping`, `@PutMapping`,
`@PatchMapping`, `@DeleteMapping`.

### `@RequestParam` vs `@PathVariable`
Path variables identify a **resource** (`/users/42`); query params **filter, sort or paginate**
(`/users?role=admin&page=2`).

## 4. `ResponseEntity` and status codes

```java
return ResponseEntity.ok(body);                                  // 200
return ResponseEntity.created(location).body(body);              // 201
return ResponseEntity.accepted().build();                        // 202 (async processing)
return ResponseEntity.noContent().build();                       // 204
return ResponseEntity.notFound().build();                        // 404
return ResponseEntity.status(HttpStatus.CONFLICT).body(error);   // 409
return ResponseEntity.ok()
        .eTag("\"v3\"")
        .cacheControl(CacheControl.maxAge(Duration.ofMinutes(10)))
        .header("X-Total-Count", "153")
        .body(body);
```
Use `ResponseEntity` when you need control over status/headers; otherwise return the body and use
`@ResponseStatus` for a fixed non-200 status.

### REST conventions interviewers expect
| Method | Semantics | Idempotent | Safe | Typical success |
|---|---|---|---|---|
| GET | Read | Yes | Yes | 200 |
| POST | Create / action | **No** | No | 201 + `Location` |
| PUT | Replace (or create at known URI) | Yes | No | 200 / 204 |
| PATCH | Partial update | Not necessarily | No | 200 |
| DELETE | Remove | Yes | No | 204 |

Status codes: 400 validation, 401 unauthenticated, 403 forbidden, 404 not found, 405 method not
allowed, 409 conflict, 415 unsupported media type, 422 semantic error, 429 rate limited, 500
server error, 503 unavailable.

## 5. `@Controller` vs `@RestController`

`@RestController` = `@Controller` + `@ResponseBody`: return values are written to the response
body (JSON). `@Controller` methods return **view names** resolved by a `ViewResolver` (Thymeleaf
templates) unless the method has `@ResponseBody`.

## 6. Content negotiation and Jackson

The `Accept` header chooses the response format among registered `HttpMessageConverter`s
(JSON by default; add `jackson-dataformat-xml` for XML). `Content-Type` chooses how to read the body;
an unsupported type → 415.

```yaml
spring:
  jackson:
    default-property-inclusion: non_null
    serialization:
      write-dates-as-timestamps: false   # ISO-8601 dates
    deserialization:
      fail-on-unknown-properties: false
```
```java
record UserResponse(
    Long id,
    @JsonProperty("full_name") String name,
    @JsonIgnore String passwordHash,
    @JsonFormat(pattern = "yyyy-MM-dd") LocalDate birthDate) {}
```
**Boot 4** uses **Jackson 3** by default (package `tools.jackson`, `JsonMapper` instead of
`ObjectMapper` as the main type); Jackson annotations keep the `com.fasterxml.jackson.annotation`
package. Boot 3 uses Jackson 2.

**Bidirectional JPA relations → infinite recursion** (`Order → lines → order → ...`). Fix with DTOs
(best), or `@JsonManagedReference`/`@JsonBackReference`, or `@JsonIgnore`.

## 7. Filters vs interceptors

| | Servlet `Filter` | `HandlerInterceptor` |
|---|---|---|
| Level | Servlet container, before `DispatcherServlet` | Inside Spring MVC, around handler |
| Sees | Every request (including static, errors) | Only requests mapped to handlers |
| Knows the handler method | No | Yes (`HandlerMethod`) |
| Can wrap/replace request/response | Yes | No |
| Use for | Security, CORS, compression, request logging, correlation IDs | Auth checks per controller, locale, timing, audit with handler info |

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
class CorrelationIdFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String id = Optional.ofNullable(req.getHeader("X-Correlation-Id"))
                            .orElse(UUID.randomUUID().toString());
        MDC.put("correlationId", id);
        res.setHeader("X-Correlation-Id", id);
        try { chain.doFilter(req, res); }
        finally { MDC.remove("correlationId"); }
    }
}

@Component
class TimingInterceptor implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        req.setAttribute("start", System.nanoTime());
        return true;                                   // false = stop processing
    }
    @Override
    public void afterCompletion(HttpServletRequest req, HttpServletResponse res,
                                Object handler, Exception ex) {
        long ms = (System.nanoTime() - (long) req.getAttribute("start")) / 1_000_000;
        log.info("{} {} took {} ms", req.getMethod(), req.getRequestURI(), ms);
    }
}

@Configuration
class WebConfig implements WebMvcConfigurer {
    private final TimingInterceptor timing;
    WebConfig(TimingInterceptor timing) { this.timing = timing; }

    @Override public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(timing).addPathPatterns("/api/**");
    }
}
```

## 8. CORS

```java
// Per controller / method
@CrossOrigin(origins = "https://app.example.com", maxAge = 3600)
@RestController class ProductController { ... }

// Global
@Configuration
class CorsConfig implements WebMvcConfigurer {
    @Override public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
                .allowedOrigins("https://app.example.com")
                .allowedMethods("GET", "POST", "PUT", "DELETE")
                .allowCredentials(true);
    }
}
```
With Spring Security, also call `http.cors(Customizer.withDefaults())` so the preflight `OPTIONS`
request isn't rejected before reaching MVC (security filters run first).

## 9. API versioning

Common strategies: URI (`/api/v1/orders`), header (`API-Version: 2`), query param (`?version=2`),
media type (`Accept: application/vnd.acme.v2+json`).

**Spring Framework 7 / Boot 4 has first-class support:**
```java
@Configuration
class ApiVersionConfig implements WebMvcConfigurer {
    @Override
    public void configureApiVersioning(ApiVersionConfigurer configurer) {
        configurer.useRequestHeader("API-Version")
                  .setDefaultVersion("1.0");
    }
}

@RestController
@RequestMapping("/api/orders")
class OrderController {
    @GetMapping(path = "/{id}", version = "1.0")
    OrderV1 getV1(@PathVariable Long id) { ... }

    @GetMapping(path = "/{id}", version = "2.0+")       // 2.0 and later
    OrderV2 getV2(@PathVariable Long id) { ... }
}
```
Before Boot 4 you'd do URI versioning with separate `@RequestMapping("/api/v1/...")` controllers or
`headers = "API-Version=2"` conditions.

## 10. File upload and download

```java
@PostMapping(path = "/files", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
ResponseEntity<String> upload(@RequestParam("file") MultipartFile file) throws IOException {
    Path target = storageDir.resolve(UUID.randomUUID() + "-" + file.getOriginalFilename());
    file.transferTo(target);
    return ResponseEntity.ok(target.getFileName().toString());
}

@GetMapping("/files/{name}")
ResponseEntity<Resource> download(@PathVariable String name) {
    Resource res = new FileSystemResource(storageDir.resolve(name));
    return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + name + "\"")
            .contentType(MediaType.APPLICATION_OCTET_STREAM)
            .body(res);
}
```
```yaml
spring.servlet.multipart:
  max-file-size: 10MB
  max-request-size: 20MB
```
Sanitize filenames (path traversal), validate content type, and stream large files instead of
loading them into memory.

## 11. Pagination and sorting

```java
public interface ProductRepository extends JpaRepository<Product, Long> {
    Page<Product> findByCategory(String category, Pageable pageable);
}

@GetMapping
Page<ProductDto> list(@RequestParam String category,
                      @PageableDefault(size = 20, sort = "name") Pageable pageable) {
    return repo.findByCategory(category, pageable).map(ProductDto::from);
}
// GET /products?category=books&page=2&size=50&sort=price,desc
```
`Page` runs an extra `COUNT` query; use `Slice` (just "has next") for infinite scroll, or
**keyset/cursor pagination** (`WHERE id > :lastId ORDER BY id LIMIT 20`) for large tables, since
deep `OFFSET` gets slow. Spring Data also has `Window`/`ScrollPosition` for keyset scrolling.
Since Spring Data 3.3, return `PagedModel` (or enable `@EnableSpringDataWebSupport(pageSerializationMode = VIA_DTO)`)
for a stable JSON shape.

## 12. Async request handling

```java
@GetMapping("/report")
CompletableFuture<Report> report() { return reportService.generateAsync(); }  // frees the Tomcat thread

@GetMapping(path = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
SseEmitter stream() {
    SseEmitter emitter = new SseEmitter(60_000L);
    executor.execute(() -> { /* emitter.send(...) ... emitter.complete(); */ });
    return emitter;
}
```
With virtual threads enabled (`spring.threads.virtual.enabled=true`, Java 21+), plain blocking
controllers scale to many concurrent requests, so `CompletableFuture`/`DeferredResult` are needed
less often.

## 13. HATEOAS and OpenAPI docs
- **Spring HATEOAS**: `EntityModel`, `linkTo(methodOn(...))` to add `_links` to responses.
- **springdoc-openapi** (`springdoc-openapi-starter-webmvc-ui`): generates OpenAPI 3 from
  controllers; Swagger UI at `/swagger-ui.html`. Annotate with `@Operation`, `@Schema`, `@Tag`.

## 14. Gotchas

1. Missing `@RequestBody` → fields are null (Spring tries to bind query params instead).
2. Returning entities → `LazyInitializationException` or infinite recursion during serialization.
3. Missing `@Valid` → constraints on the DTO are ignored.
4. Primitive `@RequestParam int page` without a default → 400 if absent; use `defaultValue` or `Integer`.
5. Trailing slash matching is **off** since Spring 6 (`/orders/` ≠ `/orders`).
6. CORS works in MVC but fails with Security → missing `http.cors()`.
7. `@PathVariable` name must match the template or use `@PathVariable("id")` (or compile with
   `-parameters`, which Boot's parent POM does).
8. Doing slow work on the request thread (with platform threads) exhausts the Tomcat pool.

## 15. Interview questions

1. **What is `DispatcherServlet`?**
   The front controller of Spring MVC. It receives all requests, finds the handler via
   `HandlerMapping`, invokes it through a `HandlerAdapter`, applies interceptors, converts the result
   with message converters or views, and routes exceptions to exception resolvers. Boot
   auto-registers it at `/`.

2. **Explain the Spring MVC request lifecycle.**
   Filters → `DispatcherServlet` → `HandlerMapping` → interceptor `preHandle` → argument resolution
   (`@RequestBody` via `HttpMessageConverter`, validation) → controller → return value handling
   (JSON via Jackson) → `postHandle`/`afterCompletion` → response. Exceptions go to
   `@ExceptionHandler`s.

3. **`@Controller` vs `@RestController`?**
   `@RestController` adds `@ResponseBody` to every method, so return values are serialized to the
   body. `@Controller` returns view names.

4. **`@RequestParam` vs `@PathVariable` vs `@RequestBody`?**
   Query/form parameter, URI template segment, and deserialized request body respectively.

5. **What is `ResponseEntity`?**
   A wrapper for the full HTTP response: status, headers and body. Used when you need non-default
   status or headers.

6. **How does Spring convert Java objects to JSON?**
   `HttpMessageConverter`s; for JSON, the Jackson converter (`MappingJackson2HttpMessageConverter`
   in Boot 3, the Jackson 3 converter in Boot 4) chosen by content negotiation on `Accept` /
   `Content-Type`.

7. **PUT vs PATCH vs POST?**
   PUT replaces a resource and is idempotent; PATCH partially updates; POST creates or triggers an
   action and isn't idempotent.

8. **What is idempotency and how do you make POST idempotent?**
   Repeating the request has the same effect as doing it once. For POST, have clients send an
   `Idempotency-Key` header; store the key with the result and return the stored result on retries.

9. **Filter vs interceptor?**
   Filters are servlet-level, run for every request before Spring MVC, and can wrap
   request/response. Interceptors are MVC-level, run around mapped handlers, and know which
   controller method is called.

10. **How do you enable CORS?**
    `@CrossOrigin`, a global `WebMvcConfigurer.addCorsMappings`, or a `CorsConfigurationSource` bean;
    with Spring Security also enable `http.cors()`.

11. **How do you version a REST API?**
    URI, header, query parameter or media-type versioning. Boot 4 supports these natively via
    `ApiVersionConfigurer` and the `version` attribute on mappings.

12. **How do you implement pagination?**
    Accept a `Pageable` parameter and return `Page`/`Slice` from Spring Data. For large data sets use
    keyset (cursor) pagination to avoid slow `OFFSET`.

13. **How do you upload files?**
    `MultipartFile` parameter with `consumes = multipart/form-data`; configure
    `spring.servlet.multipart.max-file-size`.

14. **Why use DTOs instead of entities in controllers?**
    Decouple API from schema, avoid exposing sensitive fields, avoid lazy-loading and recursion
    issues, and allow request-specific validation.

15. **How do you document APIs?**
    springdoc-openapi to generate an OpenAPI spec and Swagger UI from the controllers.

16. **What happens if two methods map to the same URL and method?**
    Startup fails with "Ambiguous mapping" unless they're distinguished by `params`, `headers`,
    `consumes`, `produces` or `version`.

17. **How do you handle long-running requests?**
    Return `202 Accepted` with a status URL and process asynchronously (queue/`@Async`), or use
    `CompletableFuture`/`DeferredResult`/SSE, or virtual threads for blocking I/O.
