# 10 — Testing

## 1. The testing pyramid in a Spring Boot app

| Level | Tool | Spring context? | Speed |
|---|---|---|---|
| Unit | JUnit 5/6 + Mockito + AssertJ | **No** (`new MyService(mock)`) | ms |
| Slice | `@WebMvcTest`, `@DataJpaTest`, `@JsonTest`, `@RestClientTest`… | Partial (one layer) | ~1 s |
| Integration | `@SpringBootTest` + Testcontainers | Full | seconds |
| End-to-end / contract | REST Assured, Spring Cloud Contract, Pact | Running app | slow |

`spring-boot-starter-test` brings JUnit Jupiter, Mockito, AssertJ, Hamcrest, JSONassert, JsonPath,
Awaitility and Spring Test. (Boot 4 also has per-technology test starters such as
`spring-boot-starter-webmvc-test`.)

## 2. Unit tests (no Spring)

Constructor injection makes this trivial:
```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock OrderRepository repo;
    @Mock PaymentGateway gateway;
    @InjectMocks OrderService service;

    @Test
    void placesOrderAndCharges() {
        when(repo.save(any(Order.class))).thenAnswer(inv -> inv.getArgument(0));

        Order order = service.place(new CreateOrderRequest(1L, List.of(new OrderLine("ABC-1234", 2))));

        assertThat(order.getStatus()).isEqualTo(OrderStatus.NEW);
        verify(gateway).charge(any());
    }

    @Test
    void failsWhenPaymentDeclined() {
        doThrow(new PaymentDeclinedException()).when(gateway).charge(any());
        assertThatThrownBy(() -> service.place(validRequest()))
            .isInstanceOf(PaymentDeclinedException.class);
        verify(repo, never()).save(any());
    }
}
```

## 3. `@SpringBootTest` (full context)

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class OrderApiIT {

    @Autowired TestRestTemplate rest;        // or RestTestClient (Boot 4) / WebTestClient
    @LocalServerPort int port;

    @Test
    void createsOrder() {
        var res = rest.postForEntity("/api/orders", validRequest(), OrderResponse.class);
        assertThat(res.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(res.getHeaders().getLocation()).isNotNull();
    }
}
```
`webEnvironment` options: `MOCK` (default: mock servlet environment, use MockMvc), `RANDOM_PORT`,
`DEFINED_PORT`, `NONE`.

`@SpringBootTest` + `@AutoConfigureMockMvc` = full context but requests via MockMvc (no real HTTP).

## 4. Slice tests

Slices load only the beans relevant to one layer (plus their auto-configuration), so they're fast
and focused.

| Annotation | Loads | Doesn't load |
|---|---|---|
| `@WebMvcTest(Controller.class)` | Controllers, advice, filters, converters, `MockMvc`, security | Services, repositories (mock them) |
| `@WebFluxTest` | WebFlux controllers, `WebTestClient` | |
| `@DataJpaTest` | Entities, repositories, `TestEntityManager`, Flyway; **transactional, rolled back after each test** | Web layer, services |
| `@DataJdbcTest`, `@JdbcTest`, `@DataMongoTest`, `@DataRedisTest` | Data layer for that store | |
| `@JsonTest` | Jackson config, `JacksonTester` | |
| `@RestClientTest` | `RestClient`/`RestTemplate` builder, `MockRestServiceServer` | |

### `@WebMvcTest` with MockMvc
```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired MockMvc mvc;
    @MockitoBean OrderService service;        // replaces the bean in the context with a mock

    @Test
    void returnsOrder() throws Exception {
        when(service.get(42L)).thenReturn(new OrderResponse(42L, OrderStatus.NEW, new BigDecimal("9.99"), Instant.now()));

        mvc.perform(get("/api/orders/42").accept(MediaType.APPLICATION_JSON))
           .andExpect(status().isOk())
           .andExpect(jsonPath("$.id").value(42))
           .andExpect(jsonPath("$.status").value("NEW"));
    }

    @Test
    void returns404WhenMissing() throws Exception {
        when(service.get(1L)).thenThrow(new OrderNotFoundException(1L));
        mvc.perform(get("/api/orders/1"))
           .andExpect(status().isNotFound())
           .andExpect(jsonPath("$.title").value("Order not found"));
    }

    @Test
    void validatesBody() throws Exception {
        mvc.perform(post("/api/orders").contentType(MediaType.APPLICATION_JSON).content("{}"))
           .andExpect(status().isBadRequest())
           .andExpect(jsonPath("$.errors.customerId").exists());
    }
}
```
AssertJ style (`MockMvcTester`, Spring 6.2+):
```java
@Autowired MockMvcTester mvc;
assertThat(mvc.get().uri("/api/orders/42")).hasStatusOk()
        .bodyJson().extractingPath("$.status").isEqualTo("NEW");
```

### `@DataJpaTest`
```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)   // use the real DB below
@Testcontainers
class OrderRepositoryTest {

    @Container @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:17");

    @Autowired OrderRepository repo;
    @Autowired TestEntityManager em;

    @Test
    void findsByStatus() {
        em.persist(new Order(OrderStatus.NEW));
        em.persist(new Order(OrderStatus.SHIPPED));
        em.flush();

        assertThat(repo.findByStatus(OrderStatus.NEW)).hasSize(1);
    }
}
```

## 5. `@MockitoBean` / `@MockitoSpyBean` (formerly `@MockBean` / `@SpyBean`)

- `@MockitoBean` **replaces** (or adds) a bean in the Spring context with a Mockito mock.
- `@MockitoSpyBean` wraps the real bean in a spy (real behavior, but verifiable/stubbable).
- `@Mock` (plain Mockito) creates a mock **outside** Spring, for unit tests.

`@MockBean`/`@SpyBean` were deprecated in Boot 3.4 and **removed in Boot 4**; the replacements
live in Spring Framework (`org.springframework.test.context.bean.override.mockito`). Also new:
`@TestBean` to replace a bean with one returned by a static factory method.

**Context caching:** Spring caches application contexts between test classes with the same
configuration. Each distinct set of `@MockitoBean`s, profiles or properties creates a **new
context**, which slows the suite. Keep test configurations consistent (e.g. a shared base class).
`@DirtiesContext` forces a new context, so use it sparingly.

## 6. Testcontainers and `@ServiceConnection`

Run real dependencies (Postgres, Kafka, Redis) in Docker instead of H2/embedded fakes, so tests
match production behavior (SQL dialect, JSON types, constraints).

```java
@TestConfiguration(proxyBeanMethods = false)
class TestcontainersConfig {
    @Bean @ServiceConnection
    PostgreSQLContainer<?> postgres() { return new PostgreSQLContainer<>("postgres:17"); }

    @Bean @ServiceConnection
    KafkaContainer kafka() { return new KafkaContainer("apache/kafka:3.9.0"); }
}

@SpringBootTest
@Import(TestcontainersConfig.class)
class CheckoutFlowIT { ... }
```
`@ServiceConnection` (Boot 3.1+) automatically sets `spring.datasource.url`, credentials,
`spring.kafka.bootstrap-servers`, etc. Before 3.1 you used `@DynamicPropertySource`:
```java
@DynamicPropertySource
static void props(DynamicPropertyRegistry r) {
    r.add("spring.datasource.url", postgres::getJdbcUrl);
}
```
**Local dev with containers:** a `TestApplication` main in `src/test` that calls
`SpringApplication.from(App::main).with(TestcontainersConfig.class).run(args)` starts the app with
the containers (`./mvnw spring-boot:test-run`).

## 7. Testing other things

```java
// Outgoing HTTP calls
@RestClientTest(PaymentClient.class)
class PaymentClientTest {
    @Autowired PaymentClient client;
    @Autowired MockRestServiceServer server;

    @Test void charges() {
        server.expect(requestTo("/charges")).andRespond(withSuccess("{\"id\":\"ch_1\"}", APPLICATION_JSON));
        assertThat(client.charge(...).id()).isEqualTo("ch_1");
    }
}
// Or WireMock for a real HTTP stub server.

// Async / eventual consistency
await().atMost(Duration.ofSeconds(10)).untilAsserted(() ->
        assertThat(repo.findByStatus(OrderStatus.PAID)).hasSize(1));

// Properties for one test class
@SpringBootTest(properties = "feature.audit.enabled=false")
@TestPropertySource(properties = "app.payment.timeout=1s")

// Output capture
@ExtendWith(OutputCaptureExtension.class)
void logs(CapturedOutput output) { ...; assertThat(output).contains("Order placed"); }
```

**Architecture tests:** ArchUnit (e.g. "controllers must not access repositories") and Spring
Modulith's `ApplicationModules.of(App.class).verify()`.

## 8. Gotchas

1. Using `@SpringBootTest` for everything → slow suites. Prefer unit tests and slices.
2. Many different `@MockitoBean` combinations → many contexts → slow.
3. H2 in tests but Postgres in prod → tests pass, prod fails (dialect differences).
4. `@DataJpaTest` rolls back each test → you never see commit-time behavior (e.g. constraint
   errors at flush, `@TransactionalEventListener`). Call `flush()` or use a non-transactional test.
5. `@Transactional` on a `RANDOM_PORT` test doesn't roll back the server's work (different thread).
6. Slices don't load your `@Service`s → `NoSuchBeanDefinitionException`; mock or `@Import` them.
7. Asserting with `Thread.sleep` instead of Awaitility → flaky tests.

## 9. Interview questions

1. **How do you test a Spring Boot application?**
   Unit tests with Mockito for business logic, slice tests (`@WebMvcTest`, `@DataJpaTest`) for
   layers, `@SpringBootTest` with Testcontainers for integration, plus contract tests between
   services.

2. **`@SpringBootTest` vs `@WebMvcTest`?**
   `@SpringBootTest` loads the full context; `@WebMvcTest` loads only the MVC layer for the given
   controllers, with services mocked. `@WebMvcTest` is much faster.

3. **What does `@DataJpaTest` do?**
   Configures JPA, repositories and an embedded/test database; each test is transactional and
   rolled back.

4. **`@Mock` vs `@MockitoBean` (`@MockBean`)?**
   `@Mock` is plain Mockito outside Spring. `@MockitoBean` replaces a bean inside the Spring
   context so other beans receive the mock.

5. **What is MockMvc?**
   A way to test Spring MVC controllers by sending mock HTTP requests through the
   `DispatcherServlet` without starting a server; assert status, headers and JSON.

6. **MockMvc vs `TestRestTemplate`/`WebTestClient`?**
   MockMvc: in-process, no network, fast. `TestRestTemplate`/`WebTestClient`/`RestTestClient`
   against a `RANDOM_PORT` server: real HTTP, tests the full stack including the servlet container.

7. **What is Testcontainers and why use it?**
   A library that starts real dependencies in Docker for tests, giving production-like behavior
   instead of in-memory fakes.

8. **What is `@ServiceConnection`?**
   Boot 3.1+ annotation on a container bean/field that auto-configures connection properties from
   the running container.

9. **How do you speed up a slow Spring test suite?**
   More unit/slice tests, consistent configuration to reuse cached contexts, avoid `@DirtiesContext`,
   reuse containers (singleton containers), parallelize.

10. **How do you test a REST client?**
    `@RestClientTest` with `MockRestServiceServer`, or WireMock.

11. **How do you test asynchronous code?**
    Awaitility to poll until an assertion passes; avoid sleeps.

12. **What is `@TestConfiguration`?**
    Extra configuration for tests that doesn't get picked up by component scanning of the main app;
    imported explicitly with `@Import`.

13. **How do you test secured endpoints?**
    `spring-security-test`: `@WithMockUser`, `with(jwt())`, `with(csrf())`.
