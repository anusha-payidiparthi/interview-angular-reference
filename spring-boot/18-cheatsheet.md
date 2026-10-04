# 18 — Spring Boot Cheat Sheet

## Core & configuration annotations

| Annotation | Purpose |
|---|---|
| `@SpringBootApplication` | `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan` |
| `@Component` / `@Service` / `@Repository` / `@Controller` / `@RestController` | Stereotypes picked up by scanning |
| `@Configuration` + `@Bean` | Java config / factory methods |
| `@Autowired`, `@Qualifier("x")`, `@Primary` | Injection and disambiguation |
| `@Value("${key:default}")` | Inject one property |
| `@ConfigurationProperties("prefix")` + `@ConfigurationPropertiesScan` | Type-safe config binding |
| `@Profile("prod")` / `@Profile("!prod")` | Environment-specific beans |
| `@ConditionalOnProperty`, `@ConditionalOnMissingBean`, `@ConditionalOnClass` | Conditional beans |
| `@Scope("prototype")`, `@RequestScope`, `@SessionScope` | Bean scopes |
| `@Lazy`, `@DependsOn`, `@Order` | Init timing and ordering |
| `@PostConstruct` / `@PreDestroy` | Lifecycle callbacks |
| `@Import`, `@ImportHttpServices` (Boot 4) | Import configuration / HTTP client interfaces |

## Web annotations

| Annotation | Purpose |
|---|---|
| `@RequestMapping("/api")` | Base path (class) or mapping (method) |
| `@GetMapping` `@PostMapping` `@PutMapping` `@PatchMapping` `@DeleteMapping` | HTTP method mappings |
| `@PathVariable`, `@RequestParam`, `@RequestBody`, `@RequestHeader`, `@CookieValue`, `@RequestPart` | Argument binding |
| `@ResponseStatus(HttpStatus.CREATED)` | Fixed response status |
| `@Valid` / `@Validated(Group.class)` | Trigger validation |
| `@ExceptionHandler`, `@RestControllerAdvice` | Error handling |
| `@CrossOrigin` | CORS |
| `@HttpExchange`, `@GetExchange`, `@PostExchange` | Declarative HTTP clients |

## Data & transactions

| Annotation | Purpose |
|---|---|
| `@Entity`, `@Table`, `@Id`, `@GeneratedValue`, `@Column`, `@Enumerated(STRING)` | Mapping |
| `@OneToMany(mappedBy=…)`, `@ManyToOne(fetch=LAZY)`, `@JoinColumn`, `@ManyToMany` | Relationships |
| `@Version` | Optimistic locking |
| `@CreatedDate`, `@LastModifiedDate`, `@EnableJpaAuditing` | Auditing |
| `@Query`, `@Modifying`, `@Param`, `@EntityGraph`, `@Lock` | Repository queries |
| `@Transactional(propagation, isolation, readOnly, rollbackFor, timeout)` | Transactions |
| `@TransactionalEventListener(phase = AFTER_COMMIT)` | After-commit hooks |

## Security, AOP, async, caching, testing

| Annotation | Purpose |
|---|---|
| `@EnableWebSecurity`, `@EnableMethodSecurity` | Security config |
| `@PreAuthorize("hasRole('ADMIN')")`, `@PostAuthorize`, `@Secured` | Method security |
| `@AuthenticationPrincipal` | Current user in controller |
| `@Aspect`, `@Pointcut`, `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`, `@Around` | AOP |
| `@EnableCaching`, `@Cacheable`, `@CachePut`, `@CacheEvict` | Caching |
| `@EnableScheduling`, `@Scheduled(fixedRate / fixedDelay / cron)` | Scheduling |
| `@EnableAsync`, `@Async` | Async methods |
| `@EventListener` | Application events |
| `@Retryable`, `@ConcurrencyLimit`, `@EnableResilientMethods` (Framework 7) | Resilience |
| `@KafkaListener`, `@RetryableTopic`, `@RabbitListener` | Messaging |
| `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`, `@JsonTest`, `@RestClientTest` | Test contexts |
| `@MockitoBean`, `@MockitoSpyBean`, `@TestBean` | Replace beans in tests |
| `@ActiveProfiles`, `@TestPropertySource`, `@DynamicPropertySource`, `@ServiceConnection` | Test config |
| `@WithMockUser` | Security tests |

## Common properties

```yaml
server:
  port: 8080
  servlet.context-path: /api
  shutdown: graceful
  compression.enabled: true
spring:
  application.name: orders
  profiles.active: dev
  threads.virtual.enabled: true
  datasource:
    url: jdbc:postgresql://localhost:5432/orders
    username: app
    password: ${DB_PASSWORD}
    hikari.maximum-pool-size: 10
  jpa:
    hibernate.ddl-auto: validate
    open-in-view: false
    properties.hibernate.default_batch_fetch_size: 50
  jackson.default-property-inclusion: non_null
  cache.type: caffeine
  kafka.bootstrap-servers: localhost:9092
  security.oauth2.resourceserver.jwt.issuer-uri: https://idp.example.com/realms/app
  mvc.problemdetails.enabled: true
management:
  endpoints.web.exposure.include: health,info,prometheus
  endpoint.health.probes.enabled: true
  tracing.sampling.probability: 0.1
logging:
  level:
    com.example: DEBUG
    org.hibernate.SQL: DEBUG
  structured.format.console: ecs
debug: false   # true prints the auto-configuration report
```

## Property precedence (high → low)
Test properties → command-line args → `SPRING_APPLICATION_JSON` → system properties → env vars →
external `application-{profile}` → external `application` → packaged `application-{profile}` →
packaged `application` → `@PropertySource` → defaults.

## Key decision tables

| Choose | When |
|---|---|
| Constructor injection | Always (required deps); setter for optional |
| `@ConfigurationProperties` over `@Value` | Any group of related settings |
| `RestClient` | Blocking HTTP calls; `WebClient` for reactive; HTTP interfaces for declarative |
| MVC + virtual threads | Default for new services; WebFlux for streaming/backpressure |
| `@WebMvcTest` / `@DataJpaTest` | Testing one layer; `@SpringBootTest` for integration |
| Optimistic locking | Low contention; pessimistic for hot rows |
| `REQUIRES_NEW` | Work that must commit independently (audit) |
| Kafka | Event streaming, replay, high throughput; RabbitMQ for task queues and routing |
| Caffeine | Local, per-instance; Redis for shared cache |

## HTTP status quick reference
200 OK · 201 Created · 202 Accepted · 204 No Content · 400 Bad Request · 401 Unauthorized ·
403 Forbidden · 404 Not Found · 405 Method Not Allowed · 409 Conflict · 415 Unsupported Media Type ·
422 Unprocessable Content · 429 Too Many Requests · 500 Internal Server Error · 502 Bad Gateway ·
503 Service Unavailable · 504 Gateway Timeout

## CLI

```bash
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
./mvnw spring-boot:test-run                      # run with test classpath (Testcontainers dev mode)
./mvnw clean package -DskipTests
java -jar target/app.jar --server.port=9090 --spring.profiles.active=prod
java -jar app.jar --debug                         # condition evaluation report
./mvnw spring-boot:build-image                    # OCI image via buildpacks
./mvnw -Pnative native:compile                    # GraalVM native executable
java -Djarmode=tools -jar app.jar extract --layers --launcher
curl localhost:8080/actuator/health
curl -X POST localhost:8080/actuator/loggers/com.example -H 'Content-Type: application/json' \
     -d '{"configuredLevel":"DEBUG"}'
```

## "Why isn't it working?" checklist

| Symptom | Likely cause |
|---|---|
| `@Transactional`/`@Async`/`@Cacheable` ignored | Self-invocation, private/final method, not a bean |
| `NoSuchBeanDefinitionException` | Class outside scanned package, missing `@Component`, slice test not loading it, condition not met |
| `NoUniqueBeanDefinitionException` | Two candidates; add `@Primary`/`@Qualifier` |
| `LazyInitializationException` | Lazy association accessed after transaction ended |
| Validation ignored | Missing `spring-boot-starter-validation` or `@Valid` |
| 403 on POST with session auth | CSRF token missing |
| CORS error only with Security | `http.cors()` not enabled |
| Fields null in `@RequestBody` | Missing `@RequestBody`, JSON names mismatch, no setters/constructor for Jackson |
| Checked exception didn't roll back | Default rollback only on unchecked; add `rollbackFor` |
| Slow API | N+1 queries, missing index, no pagination, no timeouts, small/huge pool, OSIV |
