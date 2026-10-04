# Spring Boot: Concepts, Examples & Interview Prep

A self-paced guide to **Spring Boot** for backend interviews: every topic interviewers ask about,
explained with runnable examples, how it works under the hood, common gotchas, and interview
questions **with answers** at the end of each chapter.

Written for **Spring Boot 4.x / Spring Framework 7** (current line, first released Nov 2025), with
notes on **Spring Boot 3.x** (what most company codebases run today) and on **2.x → 3.x** migration
topics (`javax` → `jakarta`) that still come up in interviews.

> **Version landscape (Oct 2026):** Spring Boot 2.7 is end-of-life. 3.x requires Java 17+ and
> Jakarta EE 10. 4.x requires Java 17+ (Java 21/25 recommended), Jakarta EE 11, and brings
> modular auto-configuration, Jackson 3, built-in API versioning, HTTP service clients and
> core resilience annotations. See [17: What's new](./17-whats-new-boot-3-and-4.md).

**Prerequisite:** comfortable with Core Java (see [../java](../java/README.md)), especially
interfaces, annotations, generics, lambdas, exceptions and concurrency basics.

---

## How to use this guide

### Phase 1: Spring fundamentals (2–3 days)
Every Spring interview starts here. You must be able to explain these without notes.

| # | Chapter | What you'll be able to answer |
|---|---------|-------------------------------|
| 00 | [Overview & roadmap](./00-overview.md) | "Spring vs Spring Boot? Why Boot?" |
| 01 | [IoC, dependency injection & beans](./01-ioc-di-beans.md) | Constructor vs field injection, bean scopes, lifecycle, `@Qualifier` |
| 02 | [Auto-configuration, starters & startup](./02-auto-configuration-and-starters.md) | How auto-config works, `@Conditional*`, custom starter |
| 03 | [Configuration, properties & profiles](./03-configuration-and-profiles.md) | `@Value` vs `@ConfigurationProperties`, property precedence, profiles |

### Phase 2: Building REST services (3–4 days)

| # | Chapter | What you'll be able to answer |
|---|---------|-------------------------------|
| 04 | [REST APIs with Spring MVC](./04-rest-apis-spring-mvc.md) | `DispatcherServlet` flow, mappings, `ResponseEntity`, CORS, versioning |
| 05 | [Exception handling & validation](./05-exception-handling-and-validation.md) | `@RestControllerAdvice`, `ProblemDetail`, Bean Validation |
| 06 | [Spring Data JPA](./06-data-jpa.md) | Repositories, derived queries, N+1, lazy loading, pagination |
| 07 | [Transactions](./07-transactions.md) | `@Transactional` propagation/isolation, self-invocation trap, rollback rules |

### Phase 3: Production concerns (1 week)

| # | Chapter | What you'll be able to answer |
|---|---------|-------------------------------|
| 08 | [Spring Security](./08-spring-security.md) | Filter chain, JWT, OAuth2, method security, CSRF |
| 09 | [AOP](./09-aop.md) | Advice types, pointcuts, JDK vs CGLIB proxies |
| 10 | [Testing](./10-testing.md) | `@SpringBootTest` vs slices, MockMvc, `@MockitoBean`, Testcontainers |
| 11 | [Actuator, logging & observability](./11-actuator-and-observability.md) | Health checks, metrics, tracing, log levels |
| 12 | [Caching, scheduling, async & events](./12-caching-scheduling-async-events.md) | `@Cacheable`, `@Scheduled`, `@Async`, virtual threads |

### Phase 4: Distributed systems (3–4 days)

| # | Chapter | What you'll be able to answer |
|---|---------|-------------------------------|
| 13 | [REST clients & microservices (Spring Cloud)](./13-rest-clients-and-microservices.md) | `RestClient`, HTTP interfaces, gateway, discovery, circuit breakers |
| 14 | [Messaging: Kafka & RabbitMQ](./14-messaging-kafka-rabbitmq.md) | Producers/consumers, consumer groups, retries, DLQ, idempotency |
| 15 | [WebFlux & reactive programming](./15-webflux-reactive.md) | `Mono` vs `Flux`, MVC vs WebFlux, backpressure |
| 16 | [Packaging, deployment & performance](./16-deployment-and-performance.md) | Fat JAR, Docker, native images, graceful shutdown, tuning |
| 17 | [What's new in Spring Boot 3 & 4](./17-whats-new-boot-3-and-4.md) | Jakarta migration, Boot 4 changes |

### Phase 5: Consolidate (3–5 days)
- **[18: Cheat sheet](./18-cheatsheet.md)**: annotations, properties and CLI commands on one page.
- **[19: Interview questions & answers](./19-interview-questions.md)**: 150 questions grouped by
  topic, plus scenario/design questions ("your API is slow, what do you check?").

---

## Quick setup

```bash
# Generate a project (or use https://start.spring.io in the browser)
curl https://start.spring.io/starter.zip \
  -d type=maven-project -d language=java -d javaVersion=21 \
  -d dependencies=web,data-jpa,validation,h2,actuator \
  -d groupId=com.example -d artifactId=demo -o demo.zip

# Run
./mvnw spring-boot:run          # Maven
./gradlew bootRun               # Gradle

# Build an executable jar and run it
./mvnw clean package
java -jar target/demo-0.0.1-SNAPSHOT.jar
```

The smallest Spring Boot web app:
```java
@SpringBootApplication
@RestController
public class DemoApplication {

    @GetMapping("/hello")
    String hello(@RequestParam(defaultValue = "world") String name) {
        return "Hello, " + name;
    }

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```
`curl localhost:8080/hello?name=Spring` → `Hello, Spring`

## Official resources
- Spring Boot reference: <https://docs.spring.io/spring-boot/>
- Spring Framework reference: <https://docs.spring.io/spring-framework/reference/>
- Spring guides (short task-focused tutorials): <https://spring.io/guides>
- Release notes / migration guides: <https://github.com/spring-projects/spring-boot/wiki>
- Spring Initializr: <https://start.spring.io>
