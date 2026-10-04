# 00 — Overview & Roadmap

## 1. What is Spring? What is Spring Boot?

**Spring Framework** is a Java framework whose core is an **IoC container**: you write plain
classes, and Spring creates them, wires their dependencies, and adds cross-cutting behavior
(transactions, security, caching) around them. On top of the core sit modules for web (MVC,
WebFlux), data access (JDBC, ORM, transactions), messaging, testing, and more.

**Spring Boot** is an opinionated layer on top of Spring that removes the setup work:

| Problem with plain Spring | What Spring Boot gives you |
|---|---|
| Lots of XML / Java config to wire a web app, a `DataSource`, Jackson, etc. | **Auto-configuration**: beans configured automatically based on the classpath and properties |
| Picking compatible versions of dozens of libraries | **Starters** + a managed **dependency BOM** (one version for Boot, everything else aligned) |
| Deploying a WAR into an external Tomcat | **Embedded server** (Tomcat/Jetty/Netty); run with `java -jar` |
| Config spread across files and environments | **Externalized configuration** (properties, YAML, env vars, profiles) |
| Building monitoring yourself | **Actuator**: health, metrics, info, tracing endpoints |

One-liner for interviews: *"Spring is the framework (DI, MVC, data, transactions). Spring Boot is
convention-over-configuration on top of it: starters, auto-configuration, an embedded server and
production-ready features, so you can build a runnable service in minutes."*

## 2. The Spring ecosystem (what each project is for)

| Project | Purpose |
|---|---|
| Spring Framework | Core container, AOP, MVC, WebFlux, JDBC, transactions, testing support |
| Spring Boot | Auto-config, starters, embedded server, Actuator |
| Spring Data | Repository abstraction for JPA, MongoDB, Redis, Cassandra, Elasticsearch, JDBC, R2DBC |
| Spring Security | Authentication, authorization, OAuth2/OIDC, protection against CSRF etc. |
| Spring Cloud | Microservice patterns: config server, service discovery, gateway, circuit breaker, OpenFeign |
| Spring Batch | Large-volume batch jobs (chunk processing, restartability) |
| Spring Integration | Enterprise integration patterns (channels, adapters) |
| Spring for Apache Kafka / AMQP | Kafka and RabbitMQ templates and listeners |
| Spring AI | LLM clients, embeddings, vector stores, RAG, tool calling |
| Spring Modulith | Structuring a monolith into verified modules with events |
| Spring Authorization Server | Build your own OAuth2 authorization server |

## 3. Anatomy of a typical Spring Boot service

```
com.example.orders
├── OrdersApplication.java          @SpringBootApplication + main()
├── order
│   ├── OrderController.java        @RestController   (HTTP layer: DTOs, validation, status codes)
│   ├── OrderService.java           @Service          (business logic, @Transactional)
│   ├── OrderRepository.java        interface extends JpaRepository (data access)
│   ├── Order.java                  @Entity
│   └── dto/CreateOrderRequest.java record + Bean Validation annotations
├── config
│   └── SecurityConfig.java         @Configuration with @Bean methods
└── common
    └── GlobalExceptionHandler.java @RestControllerAdvice
src/main/resources
├── application.yml                 default configuration
├── application-prod.yml            profile-specific overrides
└── db/migration/V1__init.sql       Flyway migrations
```

**Package by feature** (above) is preferred over package by layer (`controllers/`, `services/`)
because related code stays together and you can make classes package-private.

### Request flow end to end
```
HTTP request
  → Embedded Tomcat (thread from pool, or a virtual thread)
  → Servlet filter chain (Spring Security, CORS, logging filters)
  → DispatcherServlet
  → HandlerMapping finds @RestController method
  → HandlerInterceptors (preHandle)
  → Argument resolvers (@PathVariable, @RequestBody via Jackson, @Valid)
  → Controller → Service (transaction proxy) → Repository (JPA/Hibernate → JDBC → DB)
  → Return value → HttpMessageConverter (Jackson → JSON)
  → Exception? → @ExceptionHandler / @RestControllerAdvice
  → HTTP response
```

## 4. Topic map (everything interviewers ask)

| Area | Topics | Chapter |
|---|---|---|
| Core container | IoC, DI types, `ApplicationContext`, bean scopes, lifecycle, `@Qualifier`/`@Primary`, circular deps, `BeanPostProcessor` | [01](./01-ioc-di-beans.md) |
| Boot internals | `@SpringBootApplication`, auto-config, `@Conditional*`, starters, `SpringApplication.run` steps, custom starter | [02](./02-auto-configuration-and-starters.md) |
| Configuration | properties vs YAML, `@Value`, `@ConfigurationProperties`, profiles, precedence, secrets | [03](./03-configuration-and-profiles.md) |
| Web | `@RestController`, mappings, `ResponseEntity`, content negotiation, CORS, interceptors vs filters, file upload, API versioning | [04](./04-rest-apis-spring-mvc.md) |
| Errors & validation | `@ExceptionHandler`, `@RestControllerAdvice`, `ProblemDetail`, `@Valid`/`@Validated`, custom constraints | [05](./05-exception-handling-and-validation.md) |
| Data | Spring Data JPA, derived queries, `@Query`, paging, relationships, N+1, projections, auditing, Flyway | [06](./06-data-jpa.md) |
| Transactions | `@Transactional`, propagation, isolation, rollback rules, proxies, optimistic/pessimistic locking | [07](./07-transactions.md) |
| Security | filter chain, `SecurityFilterChain`, `UserDetailsService`, password encoding, JWT, OAuth2, method security, CSRF | [08](./08-spring-security.md) |
| AOP | aspects, advice types, pointcut expressions, proxy types | [09](./09-aop.md) |
| Testing | `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`, MockMvc, `@MockitoBean`, Testcontainers | [10](./10-testing.md) |
| Ops | Actuator, health groups, Micrometer, tracing, logging | [11](./11-actuator-and-observability.md) |
| Async & infra | caching, scheduling, `@Async`, application events, virtual threads | [12](./12-caching-scheduling-async-events.md) |
| Microservices | `RestClient`, `WebClient`, HTTP interfaces, Feign, gateway, discovery, config server, resilience | [13](./13-rest-clients-and-microservices.md) |
| Messaging | Kafka, RabbitMQ, retries, DLQ, idempotency, outbox | [14](./14-messaging-kafka-rabbitmq.md) |
| Reactive | WebFlux, `Mono`/`Flux`, backpressure, R2DBC | [15](./15-webflux-reactive.md) |
| Deployment | fat JAR, layered JARs, Docker, buildpacks, GraalVM native, CDS, graceful shutdown | [16](./16-deployment-and-performance.md) |
| Versions | Boot 3 and Boot 4 changes | [17](./17-whats-new-boot-3-and-4.md) |

## 5. Interview questions

1. **What is Spring Boot and why use it?**
   An opinionated extension of Spring that provides auto-configuration, starters, an embedded
   server, externalized configuration and Actuator. It removes boilerplate configuration and
   dependency-version management so you can ship a production-ready service quickly.

2. **Spring vs Spring Boot?**
   Spring is the framework (IoC, MVC, data, transactions); you configure everything yourself. Boot
   doesn't replace Spring, it *configures* Spring for you based on what's on the classpath, adds an
   embedded server, and gives production features. A Boot app *is* a Spring app.

3. **Spring Boot vs Spring MVC?**
   Spring MVC is the web module (DispatcherServlet, controllers). Spring Boot can auto-configure
   Spring MVC (via `spring-boot-starter-web` / `spring-boot-starter-webmvc` in Boot 4), plus
   everything else. They're not alternatives.

4. **What are the main features of Spring Boot?**
   Auto-configuration, starter dependencies, embedded servers, externalized config and profiles,
   Actuator, Spring Boot DevTools, testing support (slices, Testcontainers), packaging as an
   executable JAR, native image support.

5. **Advantages and disadvantages of Spring Boot?**
   Pros: fast setup, consistent dependency versions, production features built in, huge ecosystem.
   Cons: "magic" can hide what's configured (debug with `--debug` or `/actuator/conditions`),
   larger memory footprint and startup time than minimal frameworks (mitigated by CDS, native
   images), and pulling in starters can bring dependencies you don't need.

6. **What is the latest version and what are the minimum requirements?**
   Boot 4.x on Spring Framework 7: Java 17 minimum (21/25 recommended), Jakarta EE 11 (Servlet 6.1).
   Boot 3.x: Java 17, Jakarta EE 10. Boot 2.x used `javax.*` and Java 8.
