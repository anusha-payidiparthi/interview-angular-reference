# 19 — Spring Boot Interview Questions & Answers

150 questions grouped by topic, then scenario/design questions and "what happens here?" code
puzzles. Answers are phrased the way you'd say them in an interview: the core answer first, then a
supporting detail. Each section links to the chapter with full explanations and examples.

**How to practice:** cover the answer, say yours out loud, then compare. Mark the ones you
hesitated on and reread the linked chapter.

---

## A. Spring Boot basics ([00](./00-overview.md), [02](./02-auto-configuration-and-starters.md))

**1. What is Spring Boot?**
An opinionated layer on Spring that provides auto-configuration, starter dependencies, an embedded
server, externalized configuration and production features (Actuator), so you can build
stand-alone, production-ready apps with minimal setup.

**2. Spring vs Spring Boot?**
Spring is the framework (DI, MVC, data, transactions) that you configure yourself. Spring Boot
configures Spring automatically based on the classpath and properties and adds an embedded server
and ops features. A Boot app is still a Spring app.

**3. Key features of Spring Boot?**
Auto-configuration, starters, embedded Tomcat/Jetty/Netty, externalized config and profiles,
Actuator, DevTools, testing slices, executable JARs, native image support.

**4. What does `@SpringBootApplication` contain?**
`@SpringBootConfiguration` (a `@Configuration`), `@EnableAutoConfiguration` and `@ComponentScan`
of the main class's package and sub-packages.

**5. How does auto-configuration work?**
`@EnableAutoConfiguration` loads classes listed in
`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Each is guarded
by `@Conditional` annotations (`@ConditionalOnClass`, `@ConditionalOnMissingBean`,
`@ConditionalOnProperty`); only matching ones register beans, after user configuration, so user
beans win.

**6. How do you exclude an auto-configuration?**
`@SpringBootApplication(exclude = X.class)` or `spring.autoconfigure.exclude=...`.

**7. How do you see which auto-configurations are active?**
Run with `--debug` for the condition evaluation report or check `/actuator/conditions`.

**8. What is a starter?**
A dependency descriptor that brings a consistent set of libraries for one capability plus its
auto-configuration, e.g. `spring-boot-starter-data-jpa`.

**9. Why don't you specify versions for Boot dependencies?**
The Boot parent/BOM (`spring-boot-dependencies`) manages tested-compatible versions.

**10. What happens when you call `SpringApplication.run()`?**
It determines the app type, prepares the `Environment`, creates and refreshes the
`ApplicationContext` (bean definitions, post-processors, embedded server, singletons), calls
runners and publishes lifecycle events ending with `ApplicationReadyEvent`.

**11. `CommandLineRunner` vs `ApplicationRunner`?**
Both run once after startup; the former receives raw `String[]`, the latter parsed
`ApplicationArguments`.

**12. Which embedded servers are supported and how do you switch?**
Tomcat (default), Jetty, and Reactor Netty for WebFlux (Undertow was removed in Boot 4). Exclude
`spring-boot-starter-tomcat` and add `spring-boot-starter-jetty`.

**13. How do you change the port?**
`server.port` in properties, `--server.port` argument or `SERVER_PORT` env var; `0` for random.

**14. How do you create a custom starter?**
An autoconfigure module with an `@AutoConfiguration` class (conditions + `@ConfigurationProperties`)
listed in the `AutoConfiguration.imports` file, and a starter POM that pulls it in with its
dependencies.

**15. What is DevTools?**
A dev-only dependency with automatic restart, LiveReload and development defaults; disabled in
packaged JARs.

---

## B. IoC, DI & beans ([01](./01-ioc-di-beans.md))

**16. What is IoC?**
The container, not your code, creates objects and wires dependencies.

**17. What is dependency injection and its types?**
Supplying dependencies from outside: constructor, setter and field injection.

**18. Why is constructor injection recommended?**
`final` (immutable) fields, no partially constructed objects, easy unit testing with `new`,
explicit dependencies, and circular dependencies fail fast.

**19. `BeanFactory` vs `ApplicationContext`?**
`ApplicationContext` extends `BeanFactory` with eager singleton creation, events, i18n, environment
and profiles, resource loading and automatic post-processor registration.

**20. `@Component` vs `@Bean`?**
`@Component` annotates your class for scanning; `@Bean` annotates a factory method in a
`@Configuration` class and is used for third-party classes or custom construction.

**21. `@Service` vs `@Repository` vs `@Controller`?**
All components. `@Repository` adds persistence exception translation, `@Controller` marks MVC
handlers, `@Service` is semantic.

**22. Bean scopes?**
Singleton (default), prototype, request, session, application, websocket.

**23. Are singletons thread-safe?**
No. Keep them stateless or protect shared mutable state.

**24. How do you inject a prototype bean into a singleton?**
`ObjectProvider<T>`, a scoped proxy, or `@Lookup`.

**25. Explain the bean lifecycle.**
Instantiate → populate dependencies → `Aware` callbacks → `BeanPostProcessor` before-init →
`@PostConstruct`/`afterPropertiesSet`/init-method → `BeanPostProcessor` after-init (proxies) →
ready → `@PreDestroy`/`destroy`/destroy-method on shutdown.

**26. `BeanPostProcessor` vs `BeanFactoryPostProcessor`?**
The latter modifies bean definitions before instantiation; the former wraps or modifies bean
instances around initialization.

**27. How does Spring resolve multiple beans of the same type?**
`@Qualifier` at the injection point, then `@Primary`, then parameter name; otherwise
`NoUniqueBeanDefinitionException`. Or inject a `List`/`Map` of all candidates.

**28. `@Autowired` vs `@Resource` vs `@Inject`?**
`@Autowired`/`@Inject` by type; `@Resource` by name first.

**29. How are circular dependencies handled?**
Forbidden by default since Boot 2.6. Fix by redesigning, `@Lazy` on one injection point, or
(last resort) setter injection with `spring.main.allow-circular-references=true`.

**30. `@Configuration` vs `@Component` for `@Bean` methods?**
`@Configuration` is CGLIB-proxied so inter-bean calls return the singleton; lite mode creates a
new instance per call.

**31. What is `@Lazy`?**
Defers bean creation to first use; on an injection point injects a lazy proxy.

**32. What is `@Primary`?**
Marks the default candidate when multiple beans of a type exist.

**33. What is `ApplicationContextAware`?**
A callback interface giving a bean the `ApplicationContext`; prefer injecting what you need.

**34. Can you have two beans of the same class?**
Yes, with different names (two `@Bean` methods); inject with `@Qualifier`.

**35. Spring singleton vs GoF singleton?**
Per container per bean definition vs per class loader enforced by the class.

---

## C. Configuration & profiles ([03](./03-configuration-and-profiles.md))

**36. `application.properties` vs `application.yml`?**
Same purpose; YAML is hierarchical and supports multiple documents. Properties wins on conflicts in
the same location.

**37. Property precedence order?**
Test properties > command-line > `SPRING_APPLICATION_JSON` > system properties > env vars > external
profile files > external files > packaged profile files > packaged files > `@PropertySource` >
defaults.

**38. `@Value` vs `@ConfigurationProperties`?**
Single values with SpEL vs type-safe binding of a prefix with validation, relaxed binding and IDE
metadata.

**39. What are profiles and how do you activate them?**
Named groups of config/beans; `spring.profiles.active` via property, argument, env var or
`@ActiveProfiles`.

**40. How do you validate configuration?**
`@Validated` + constraints on `@ConfigurationProperties`; invalid config fails startup.

**41. How do you manage secrets?**
Env vars, mounted secrets (`configtree`), Vault/cloud secret managers, never in Git.

**42. What is relaxed binding?**
Multiple naming forms (`my.service-url`, `MY_SERVICEURL`, `myServiceUrl`) bind to the same property.

**43. How do you refresh configuration without restart?**
`@RefreshScope` + `/actuator/refresh` (Spring Cloud), Cloud Bus, or Kubernetes ConfigMap reload.

**44. What does `spring.config.import` do?**
Imports additional config sources: files, config trees, Config Server, Vault.

**45. What is `@Profile("!prod")`?**
The bean is registered in every profile except `prod`.

---

## D. REST & Spring MVC ([04](./04-rest-apis-spring-mvc.md))

**46. What is `DispatcherServlet`?**
Spring MVC's front controller that routes requests to handlers and manages conversion, views and
exception resolution.

**47. Describe the request flow.**
Filters → `DispatcherServlet` → `HandlerMapping` → interceptors → argument resolvers/message
converters → controller → return value → message converter → response; exceptions → advice.

**48. `@Controller` vs `@RestController`?**
`@RestController` = `@Controller` + `@ResponseBody`.

**49. `@RequestParam` vs `@PathVariable`?**
Query/form parameters vs URI template segments. Path identifies a resource; query filters it.

**50. What does `@RequestBody` do?**
Deserializes the request body via an `HttpMessageConverter` (Jackson for JSON).

**51. What is `ResponseEntity`?**
Full control over status, headers and body.

**52. How is JSON produced?**
Jackson `HttpMessageConverter`, chosen via content negotiation (`Accept` header).

**53. PUT vs PATCH vs POST?**
Replace (idempotent) vs partial update vs create/action (not idempotent).

**54. Which HTTP methods are idempotent?**
GET, HEAD, PUT, DELETE, OPTIONS; POST is not; PATCH is not guaranteed.

**55. Filter vs interceptor?**
Servlet-level for all requests vs MVC-level around handlers with knowledge of the handler method.

**56. How do you enable CORS?**
`@CrossOrigin`, `WebMvcConfigurer.addCorsMappings`, or a `CorsConfigurationSource`; with Security also
`http.cors()`.

**57. How do you version APIs?**
URI, header, query param or media type. Boot 4 supports it natively with `version` on mappings.

**58. How do you implement pagination?**
`Pageable` parameter + `Page`/`Slice` from Spring Data; keyset pagination for large tables.

**59. How do you upload files?**
`MultipartFile` with `multipart/form-data`; configure size limits.

**60. Why DTOs instead of entities?**
Decoupling, security, avoiding lazy-loading/recursion problems, request-specific validation.

**61. How do you document APIs?**
springdoc-openapi (OpenAPI 3 + Swagger UI).

**62. How do you make POST idempotent?**
An `Idempotency-Key` header stored with the result; replays return the stored response.

**63. What is HATEOAS?**
Including hypermedia links in responses so clients discover actions; Spring HATEOAS supports it.

**64. How do you handle a long-running request?**
Return 202 with a status resource and process async, or use `CompletableFuture`/SSE; virtual threads
for blocking I/O.

**65. What is content negotiation?**
Selecting the response representation based on `Accept` (and request parsing on `Content-Type`).

---

## E. Exception handling & validation ([05](./05-exception-handling-and-validation.md))

**66. How do you handle exceptions globally?**
`@RestControllerAdvice` with `@ExceptionHandler` methods returning `ProblemDetail`.

**67. What is `ProblemDetail`?**
RFC 9457 standard error body (`type`, `title`, `status`, `detail`, `instance`).

**68. Ways to map exceptions to status codes?**
`@ExceptionHandler`, `@ResponseStatus` on the exception, `ResponseStatusException`,
`ErrorResponseException`.

**69. What happens to an unhandled exception?**
Forwarded to `/error`; `BasicErrorController` returns a default JSON with 500.

**70. How does Bean Validation work in Spring?**
Hibernate Validator; `@Valid` on parameters triggers validation; failures → 400 via
`MethodArgumentNotValidException`.

**71. `@Valid` vs `@Validated`?**
Standard with cascading vs Spring's with groups and class-level method validation.

**72. `@NotNull` vs `@NotEmpty` vs `@NotBlank`?**
Not null / not null and non-empty / not null and non-whitespace (strings).

**73. How do you write a custom validator?**
Constraint annotation with `@Constraint(validatedBy = ...)` + `ConstraintValidator` implementation.

**74. What are validation groups?**
Marker interfaces selecting which constraints apply (`OnCreate`, `OnUpdate`) via `@Validated(Group.class)`.

**75. Why don't security exceptions reach `@ControllerAdvice`?**
They're thrown in the filter chain before `DispatcherServlet`; handle with
`AuthenticationEntryPoint`/`AccessDeniedHandler`.

---

## F. Data JPA ([06](./06-data-jpa.md))

**76. JPA vs Hibernate vs Spring Data JPA?**
Spec vs implementation vs repository abstraction.

**77. `CrudRepository` vs `JpaRepository`?**
`JpaRepository` adds paging/sorting, flush, batch operations and returns `List`s.

**78. How do derived queries work?**
Method names are parsed into JPQL at startup.

**79. `@Query` JPQL vs native?**
JPQL works on entities and is portable; native SQL for DB-specific features.

**80. What is the N+1 problem?**
One query for parents plus one per parent for a lazy association. Fix with `JOIN FETCH`,
`@EntityGraph`, batch fetching or projections.

**81. Lazy vs eager fetching? Defaults?**
`@ManyToOne`/`@OneToOne` eager, `@OneToMany`/`@ManyToMany` lazy. Make all lazy; fetch per use case.

**82. What causes `LazyInitializationException`?**
Accessing a lazy association after the session closed.

**83. What is Open Session In View?**
Session kept open for the whole request (default true); disable for APIs.

**84. Entity states?**
Transient, managed, detached, removed.

**85. What is dirty checking?**
Automatic UPDATE of changed managed entities at flush/commit.

**86. `save` vs `saveAndFlush`?**
Deferred SQL vs immediate flush.

**87. `findById` vs `getReferenceById`?**
Query returning `Optional` vs lazy proxy without a query.

**88. What are projections?**
Interface/DTO results selecting only needed columns.

**89. First- vs second-level cache?**
Per persistence context vs shared, opt-in cache across sessions.

**90. ID generation strategies?**
`IDENTITY`, `SEQUENCE`, `TABLE`, `AUTO`, `UUID`. `SEQUENCE` enables batching.

**91. What is the owning side of a relationship?**
The side with the foreign key (`@JoinColumn`); `mappedBy` marks the inverse side, which isn't
used for writes.

**92. Cascade vs orphanRemoval?**
Propagate operations to children vs delete children removed from the collection.

**93. How do you write dynamic queries?**
Specifications, Querydsl, Query by Example, or Criteria API.

**94. How do you manage schema changes?**
Flyway or Liquibase versioned migrations; `ddl-auto=validate`.

**95. How do you speed up bulk inserts?**
JDBC batching with sequence IDs, flush/clear periodically, or `JdbcTemplate.batchUpdate`.

---

## G. Transactions ([07](./07-transactions.md))

**96. How does `@Transactional` work?**
An AOP proxy begins/joins a transaction via the `PlatformTransactionManager`, binds it to the
thread, and commits or rolls back after the method.

**97. Why does `@Transactional` fail on self-invocation?**
The internal call bypasses the proxy.

**98. Default rollback behavior?**
Rollback on unchecked exceptions and errors; commit on checked exceptions unless `rollbackFor`.

**99. Propagation types?**
`REQUIRED`, `REQUIRES_NEW`, `NESTED`, `SUPPORTS`, `NOT_SUPPORTED`, `MANDATORY`, `NEVER`.

**100. `REQUIRED` vs `REQUIRES_NEW`?**
Join the existing transaction vs suspend it and run an independent one.

**101. Isolation levels?**
Read uncommitted, read committed, repeatable read, serializable.

**102. Dirty vs non-repeatable vs phantom reads?**
Reading uncommitted data / same row changes between reads / new rows appear between reads.

**103. What does `readOnly = true` do?**
Optimizes Hibernate (no dirty checking/flush) and can route to replicas.

**104. Optimistic vs pessimistic locking?**
`@Version` check at update (no DB locks) vs `SELECT ... FOR UPDATE` locks.

**105. What is `UnexpectedRollbackException`?**
Outer code tries to commit a transaction an inner participant marked rollback-only.

**106. How do you handle distributed transactions?**
Saga + outbox + idempotent consumers instead of 2PC.

**107. Where should `@Transactional` go?**
Service methods representing a use case.

---

## H. Security ([08](./08-spring-security.md))

**108. How does Spring Security work?**
`DelegatingFilterProxy` → `FilterChainProxy` → matching `SecurityFilterChain` filters;
authentication via `AuthenticationManager`/providers; result in `SecurityContextHolder`;
`AuthorizationFilter` enforces rules.

**109. Authentication vs authorization?**
Who you are vs what you may do.

**110. How do you configure security in Security 6+?**
`SecurityFilterChain` bean with the lambda DSL.

**111. What is `UserDetailsService`?**
Loads user details by username for `DaoAuthenticationProvider`.

**112. How are passwords stored?**
BCrypt/Argon2 via `PasswordEncoder` (`DelegatingPasswordEncoder`).

**113. How does JWT authentication work?**
Client sends `Authorization: Bearer <jwt>`; resource server verifies signature (JWKS), expiry,
issuer, audience; maps claims to authorities; stateless.

**114. JWT drawbacks?**
Hard revocation, size, readable payload; use short expiry + refresh tokens.

**115. OAuth2 vs OIDC?**
Authorization (access tokens) vs authentication layer (ID tokens) on top.

**116. OAuth2 grant types?**
Authorization Code + PKCE, Client Credentials, Refresh Token, Device Code.

**117. What is CSRF and when can you disable it?**
Forged requests using the victim's cookies; disable only for stateless token APIs.

**118. `hasRole` vs `hasAuthority`?**
`hasRole("X")` checks `ROLE_X`; `hasAuthority` checks the exact string.

**119. Method-level security?**
`@EnableMethodSecurity` + `@PreAuthorize`/`@PostAuthorize`/`@Secured`.

**120. 401 vs 403?**
Unauthenticated vs authenticated but forbidden.

---

## I. AOP ([09](./09-aop.md))

**121. What is AOP?**
Modularizing cross-cutting concerns into aspects applied declaratively.

**122. Advice types?**
`@Before`, `@AfterReturning`, `@AfterThrowing`, `@After`, `@Around`.

**123. JDK proxy vs CGLIB?**
Interface-based vs subclass-based; Boot defaults to CGLIB.

**124. Spring AOP vs AspectJ?**
Runtime proxies on bean methods vs bytecode weaving for all join points.

**125. Where does Spring use AOP?**
`@Transactional`, `@Cacheable`, `@Async`, `@PreAuthorize`, `@Retryable`, `@Validated`.

---

## J. Testing ([10](./10-testing.md))

**126. `@SpringBootTest` vs slice tests?**
Full context vs one layer (`@WebMvcTest`, `@DataJpaTest`) with faster startup.

**127. `@Mock` vs `@MockitoBean`?**
Plain Mockito object vs a mock registered in the Spring context. (`@MockBean` removed in Boot 4.)

**128. What is MockMvc?**
In-process testing of MVC controllers through the `DispatcherServlet` without a server.

**129. What is Testcontainers / `@ServiceConnection`?**
Real dependencies in Docker for tests; `@ServiceConnection` wires their connection properties.

**130. How do you speed up Spring tests?**
More unit/slice tests, consistent configs for context caching, avoid `@DirtiesContext`, reuse
containers.

---

## K. Actuator & observability ([11](./11-actuator-and-observability.md))

**131. What is Actuator?**
Production endpoints: health, metrics, info, loggers, env, thread dumps.

**132. Custom health indicator?**
Implement `HealthIndicator` returning `Health.up()/down()`.

**133. Liveness vs readiness?**
Restart-or-not vs route-traffic-or-not.

**134. What is Micrometer?**
Vendor-neutral metrics facade; exports to Prometheus, Datadog, OTLP.

**135. How is distributed tracing done in Boot 3+?**
Micrometer Tracing + OpenTelemetry/Brave; IDs propagated in headers and added to log MDC.

**136. How do you change log levels at runtime?**
`POST /actuator/loggers/{logger}` with `configuredLevel`.

---

## L. Caching, scheduling, async ([12](./12-caching-scheduling-async-events.md))

**137. `@Cacheable` vs `@CachePut` vs `@CacheEvict`?**
Skip on hit / always run and update / remove entries.

**138. `fixedRate` vs `fixedDelay`?**
Start-to-start vs end-to-start interval.

**139. How do you avoid duplicate scheduled jobs across instances?**
ShedLock, leader election, Quartz clustering, or Kubernetes CronJob.

**140. `@Async` pitfalls?**
Self-invocation, lost exceptions in `void` methods, no transaction/security/MDC propagation, pool
configuration.

**141. How do you enable virtual threads?**
`spring.threads.virtual.enabled=true` on Java 21+.

**142. `@EventListener` vs `@TransactionalEventListener`?**
Immediate in the publisher's transaction vs at a transaction phase (after commit by default).

---

## M. Microservices & messaging ([13](./13-rest-clients-and-microservices.md), [14](./14-messaging-kafka-rabbitmq.md))

**143. `RestTemplate` vs `RestClient` vs `WebClient`?**
Legacy sync vs modern fluent sync vs reactive non-blocking.

**144. What is a circuit breaker?**
Stops calls to a failing dependency (CLOSED → OPEN → HALF_OPEN) to fail fast and allow recovery.

**145. What does an API gateway do?**
Routing, auth, rate limiting, CORS, TLS, aggregation.

**146. Service discovery?**
Dynamic lookup of service instances (Eureka, Consul, or Kubernetes DNS).

**147. Kafka vs RabbitMQ?**
Partitioned log with replay and high throughput vs broker with flexible routing and per-message acks.

**148. How do you guarantee message processing?**
At-least-once delivery with retries and DLQ plus idempotent consumers; outbox for publishing.

---

## N. Reactive & deployment ([15](./15-webflux-reactive.md), [16](./16-deployment-and-performance.md))

**149. MVC vs WebFlux?**
Blocking thread-per-request vs non-blocking event loop; MVC (+ virtual threads) by default, WebFlux
for streaming/backpressure.

**150. How do you improve startup time and memory?**
CDS/AOT cache, Spring AOT, GraalVM native image, CRaC, lazy init, fewer starters.

---

## O. Scenario & design questions

**S1. Your endpoint takes 5 seconds. How do you find and fix the cause?**
Reproduce and measure: check the trace (which span is slow), Actuator metrics
(`http.server.requests`, `hikaricp.connections.pending`), and SQL logs. Typical culprits: N+1
queries (fix with fetch joins/projections), missing indexes (check `EXPLAIN`), fetching too much
(paginate, project), slow downstream calls (timeouts, caching, parallel calls), connection-pool
starvation (long transactions, OSIV), GC pressure. Fix the biggest contributor, add a regression
test or alert.

**S2. Two users update the same record simultaneously. How do you avoid lost updates?**
Add `@Version` for optimistic locking; the second update fails with
`ObjectOptimisticLockingFailureException` → return 409 so the client reloads, or retry if the
operation is safe. For heavy contention (inventory counters), use pessimistic locking or an atomic
SQL update (`UPDATE stock SET qty = qty - 1 WHERE id = ? AND qty > 0`).

**S3. Place an order: save to DB, charge payment, send email, publish an event. Design it.**
Save the order as `PENDING` in a transaction, together with an outbox event. Call payment
**outside** the DB transaction with timeouts, retries (with an idempotency key) and a circuit
breaker; update the status in a new short transaction. Send the email from an
`@TransactionalEventListener(AFTER_COMMIT)` or an async consumer of the event. The outbox relay
publishes `OrderPlaced` to Kafka; consumers are idempotent. If payment fails, mark the order failed
(compensation).

**S4. A scheduled job runs three times because you have three pods. Fix it.**
ShedLock with a DB/Redis lock, Kubernetes CronJob calling an endpoint or running a one-off
container, Quartz in clustered mode, or leader election.

**S5. How would you implement rate limiting?**
At the gateway (Spring Cloud Gateway `RequestRateLimiter` with Redis, or the cloud provider's
gateway). In-app with Bucket4j (token bucket) backed by Redis for multiple instances, or
Resilience4j `RateLimiter` for outbound calls. Return 429 with `Retry-After`.

**S6. A downstream service is slow and your service starts failing too. What do you do?**
Timeouts on the client, a circuit breaker with a fallback, a bulkhead to cap concurrent calls,
retries with backoff only for idempotent calls, caching of stable data, async processing where an
immediate answer isn't needed. Monitor with metrics and alerts.

**S7. How would you design a file-upload service for large files?**
Don't stream through the app if avoidable: issue pre-signed S3/Blob URLs so clients upload
directly. If proxied, stream (no in-memory buffering), set multipart limits, validate type/size,
scan for malware asynchronously, store metadata in DB, process via events.

**S8. How do you secure a public REST API?**
HTTPS; OAuth2/OIDC with JWT validation at the gateway and in the service; scopes/roles with
`@PreAuthorize`; input validation; rate limiting; CORS restricted to known origins; security
headers; no sensitive data in logs or errors; dependency scanning; secrets in a secret manager.

**S9. How do you handle a breaking change to an API with existing clients?**
Version the API (header/URI or Boot 4 native versioning), run both versions in parallel, announce
deprecation (`Deprecation`/`Sunset` headers), monitor usage of the old version, then remove it.
Prefer additive, backward-compatible changes when possible.

**S10. How do you migrate a database column without downtime?**
Expand–contract: add the new column (nullable), deploy code writing both, backfill data, switch
reads to the new column, stop writing the old one, then drop it in a later release. Each step is a
Flyway migration compatible with the previous app version.

**S11. Your app runs out of memory in production. How do you investigate?**
Check memory metrics (heap vs non-heap/metaspace vs native) and container limits. Capture a heap
dump (`-XX:+HeapDumpOnOutOfMemoryError` or `/actuator/heapdump` internally) and analyze with
Eclipse MAT for dominant objects. Common causes: unbounded caches/maps, loading whole tables
(`findAll`), `ThreadLocal` leaks in pools, huge responses buffered in memory, heap set larger than
the container limit.

**S12. How would you structure a large Spring Boot monolith?**
Package by feature/module with clear public APIs, internal packages package-private, communication
between modules via application events; verify boundaries with Spring Modulith or ArchUnit. This
keeps the option to extract modules into services later.

**S13. How do you make a consumer safe against duplicate Kafka messages?**
Store processed event IDs in the same transaction as the side effect, use conditional updates or
upserts, and key messages so related events stay ordered in one partition.

**S14. How would you cache a product catalog read by many instances?**
Redis as a shared cache with TTLs and `@Cacheable`, evict/update on writes (or publish change
events to evict), optionally a small Caffeine near-cache with short TTL in front for the hottest
keys; HTTP caching with ETags for clients.

**S15. A bean isn't being created in your app. How do you debug it?**
Check the class is in the scanned package tree and annotated, run with `--debug` to see condition
evaluation (for conditional/auto-config beans), check active profiles, look for `@ConditionalOn*`
mismatches, and in tests, check whether the slice loads that bean type.

---

## P. "What happens here?" puzzles

**P1.**
```java
@Service
class UserService {
    public void registerAll(List<User> users) { users.forEach(this::register); }

    @Transactional
    public void register(User u) { repo.save(u); audit.log(u); }
}
```
*Is `register` transactional when called from `registerAll`?*
**No.** Self-invocation bypasses the proxy. Each `repo.save` runs in its own repository-level
transaction, and an exception in `audit.log` won't roll back the save.

**P2.**
```java
@Transactional
public void importFile(Path p) throws IOException {
    repo.save(parseHeader(p));
    throw new IOException("bad line 42");
}
```
*Is the header saved?*
**Yes**, it's committed: `IOException` is checked and Spring doesn't roll back on checked
exceptions by default. Use `@Transactional(rollbackFor = Exception.class)`.

**P3.**
```java
@Component
@Scope("prototype")
class Cart { List<Item> items = new ArrayList<>(); }

@Service
class ShopService {
    @Autowired Cart cart;
}
```
*Does each call to `ShopService` get a new `Cart`?*
**No.** `ShopService` is a singleton and receives one `Cart` at creation, so all users share it.
Use `ObjectProvider<Cart>` or a scoped proxy (and really, carts belong in a session or DB).

**P4.**
```java
@RestController
class OrderController {
    @PostMapping("/orders")
    Order create(OrderRequest req) { return service.create(req); }
}
```
*Why are all fields of `req` null when posting JSON?*
`@RequestBody` is missing, so Spring binds query/form parameters instead of the JSON body.

**P5.**
```java
@Async
public void sendEmail(String to) { throw new IllegalStateException("SMTP down"); }
```
*What does the caller see?*
Nothing. The exception happens on another thread and is passed to the
`AsyncUncaughtExceptionHandler` (logged by default). Return `CompletableFuture` or configure a
handler if failures matter.

**P6.**
```java
List<Order> orders = orderRepo.findAll();     // 200 orders
orders.forEach(o -> System.out.println(o.getCustomer().getName()));
```
*How many queries run (with `@ManyToOne(fetch = LAZY) customer`)?*
Up to **201** (1 + one per distinct customer): the N+1 problem. With the default `EAGER` you also
get N extra selects. Use `join fetch` or `@EntityGraph`.

**P7.**
```java
@Bean
SecurityFilterChain chain(HttpSecurity http) throws Exception {
    return http.authorizeHttpRequests(a -> a
            .anyRequest().authenticated()
            .requestMatchers("/public/**").permitAll())
        .build();
}
```
*Is `/public/hello` accessible anonymously?*
The app doesn't even start: Spring Security throws `IllegalStateException: Can't configure
requestMatchers after anyRequest`. Rules are matched in order (first match wins), so specific
matchers go first and `anyRequest()` goes last.

**P8.**
```java
@Cacheable("prices")
public Price price(String sku) { ... }

public List<Price> prices(List<String> skus) {
    return skus.stream().map(this::price).toList();
}
```
*Is caching applied when `prices` is called?*
**No.** Self-invocation again; `this.price` bypasses the caching proxy.

**P9.**
```yaml
# application.yml
server.port: 8080
```
```bash
SERVER_PORT=9090 java -jar app.jar --server.port=7070
```
*Which port is used?*
**7070.** Command-line arguments beat environment variables, which beat `application.yml`.

**P10.**
```java
@Transactional
public void transfer(...) {
    try {
        accountService.debit(from, amount);   // REQUIRED, throws RuntimeException
    } catch (RuntimeException e) {
        log.warn("debit failed, continuing");
    }
    accountService.credit(to, amount);
}
```
*What happens at the end of `transfer`?*
`UnexpectedRollbackException`. The inner `REQUIRED` method's exception marked the shared transaction
rollback-only; catching it doesn't undo that, so the commit attempt fails and everything rolls
back. If `accountService` were the same class (self-invocation), there'd be no inner proxy and the
catch would truly swallow the error.
