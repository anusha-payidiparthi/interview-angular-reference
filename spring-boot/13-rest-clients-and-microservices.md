# 13 — REST Clients & Microservices (Spring Cloud)

## 1. Calling other services: which client?

| Client | Style | Status |
|---|---|---|
| `RestTemplate` | Synchronous, template methods | Maintenance mode; deprecation planned. Don't use in new code |
| **`RestClient`** (Spring 6.1+) | Synchronous, fluent | **Default choice for blocking apps** |
| `WebClient` | Reactive, non-blocking, fluent | For WebFlux / streaming / high fan-out |
| **HTTP interface** (`@HttpExchange`) | Declarative interface, backed by `RestClient` or `WebClient` | Recommended declarative option |
| OpenFeign (Spring Cloud) | Declarative interface | Feature-complete/maintenance mode; HTTP interfaces are the suggested successor |

### `RestClient`
```java
@Configuration
class ClientsConfig {
    @Bean
    RestClient inventoryRestClient(RestClient.Builder builder) {   // auto-configured builder: tracing, Jackson
        return builder
                .baseUrl("http://inventory-service")
                .defaultHeader("X-Client", "orders")
                .requestInterceptor(new BearerTokenInterceptor())
                .build();
    }
}

@Component
class InventoryClient {
    private final RestClient client;
    InventoryClient(RestClient inventoryRestClient) { this.client = inventoryRestClient; }

    Stock stock(String sku) {
        return client.get()
                .uri("/api/stock/{sku}", sku)
                .retrieve()
                .onStatus(status -> status.value() == 404,
                          (req, res) -> { throw new SkuNotFoundException(sku); })
                .body(Stock.class);
    }

    ResponseEntity<Void> reserve(Reservation r) {
        return client.post().uri("/api/reservations")
                .contentType(MediaType.APPLICATION_JSON)
                .body(r)
                .retrieve()
                .toBodilessEntity();
    }
}
```
Timeouts are essential (defaults can be infinite):
```yaml
spring.http.client:          # Boot 3.4+ names; Boot 4 reorganized these under spring.http.clients.*
  connect-timeout: 2s
  read-timeout: 5s
```

### HTTP interfaces (declarative)
```java
@HttpExchange("/api")
public interface InventoryApi {
    @GetExchange("/stock/{sku}")
    Stock stock(@PathVariable String sku);

    @PostExchange("/reservations")
    void reserve(@RequestBody Reservation r);
}

// Boot 3: create the proxy yourself
@Bean
InventoryApi inventoryApi(RestClient inventoryRestClient) {
    return HttpServiceProxyFactory
            .builderFor(RestClientAdapter.create(inventoryRestClient))
            .build()
            .createClient(InventoryApi.class);
}

// Boot 4 / Spring 7: register groups of clients declaratively
@Configuration
@ImportHttpServices(group = "inventory", types = InventoryApi.class)
class HttpClientsConfig { }
// spring.http.serviceclient.inventory.base-url=http://inventory-service
```

### `WebClient`
```java
Mono<Stock> stock = webClient.get().uri("/api/stock/{sku}", sku)
        .retrieve()
        .bodyToMono(Stock.class)
        .timeout(Duration.ofSeconds(3))
        .retryWhen(Retry.backoff(3, Duration.ofMillis(200)));

// Parallel fan-out: call three services concurrently
Mono.zip(userMono, ordersMono, recommendationsMono)
    .map(t -> new Dashboard(t.getT1(), t.getT2(), t.getT3()));
```
You can use `WebClient` in an MVC app (call `.block()` at the edge), but with virtual threads,
`RestClient` + `CompletableFuture`/structured concurrency covers most fan-out needs.

## 2. Microservices fundamentals

**Monolith vs microservices:**

| | Monolith | Microservices |
|---|---|---|
| Deploy | One unit | Independently per service |
| Scaling | Whole app | Per service |
| Data | One DB, ACID transactions | DB per service, eventual consistency |
| Complexity | In the code | In the network/ops (latency, partial failure, observability, distributed data) |
| Team fit | Small teams | Many teams owning services |

Good answer: *start with a well-modularized monolith (e.g. Spring Modulith) and extract services
when there's a clear reason (independent scaling, team autonomy, different release cadence).*

Key patterns: API gateway, service discovery, externalized config, circuit breaker, database per
service, saga, outbox, CQRS, event-driven communication, distributed tracing, strangler fig
(incremental migration from a monolith).

## 3. Spring Cloud components

| Concern | Spring Cloud project | Alternatives |
|---|---|---|
| API gateway | **Spring Cloud Gateway** (WebFlux or MVC variant) | Kong, NGINX, AWS API Gateway, Kubernetes Ingress/Gateway API |
| Service discovery | **Eureka** (Netflix), Consul, Zookeeper | Kubernetes Services (DNS) |
| Client-side load balancing | **Spring Cloud LoadBalancer** (`@LoadBalanced`) | Kubernetes/service mesh |
| Central configuration | **Spring Cloud Config** (Git-backed server) | Kubernetes ConfigMaps, Vault, AWS AppConfig |
| Resilience | **Spring Cloud Circuit Breaker** + Resilience4j | Service mesh (Istio) |
| Declarative clients | OpenFeign | HTTP interfaces |
| Messaging abstraction | Spring Cloud Stream (Kafka/Rabbit binders) | Spring Kafka directly |
| Tracing | (Sleuth, retired) → Micrometer Tracing | |
| Kubernetes | Spring Cloud Kubernetes | |

Hystrix and Ribbon (Netflix) are **retired**; replaced by Resilience4j and Spring Cloud
LoadBalancer.

### Service discovery (Eureka)
```java
@SpringBootApplication
@EnableEurekaServer                       // registry app (spring-cloud-starter-netflix-eureka-server)
class DiscoveryServer { }
```
```yaml
# each service (spring-cloud-starter-netflix-eureka-client)
spring.application.name: inventory-service
eureka.client.service-url.defaultZone: http://discovery:8761/eureka
```
```java
@Bean @LoadBalanced
RestClient.Builder loadBalancedBuilder() { return RestClient.builder(); }
// "http://inventory-service/api/stock" → resolved to a live instance, round-robin
```
On Kubernetes you usually skip Eureka: a `Service` gives a stable DNS name and load balancing.

### API Gateway (Spring Cloud Gateway)
```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: orders
          uri: lb://orders-service              # lb:// = via discovery + load balancer
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args: { name: orders, fallbackUri: forward:/fallback/orders }
            - name: RequestRateLimiter            # Redis-backed token bucket
              args: { redis-rate-limiter.replenishRate: 10, redis-rate-limiter.burstCapacity: 20 }
```
Gateway responsibilities: routing, authentication (validate JWT once at the edge), rate limiting,
CORS, request/response transformation, aggregation, TLS termination.

### Config Server
```java
@SpringBootApplication @EnableConfigServer
class ConfigServerApp { }
```
```yaml
# config server
spring.cloud.config.server.git.uri: https://github.com/acme/config-repo
# client
spring.config.import: "configserver:http://config-server:8888"
```
Clients fetch `{application}-{profile}.yml` at startup; `@RefreshScope` + `/actuator/refresh`
(or Spring Cloud Bus) reloads at runtime.

## 4. Resilience patterns

| Pattern | Purpose |
|---|---|
| **Timeout** | Never wait forever on a dependency |
| **Retry** (with exponential backoff + jitter) | Survive transient failures; only idempotent calls |
| **Circuit breaker** | Stop calling a failing dependency; fail fast; let it recover |
| **Bulkhead** | Isolate resources (separate pools/limits per dependency) so one slow service can't exhaust all threads |
| **Rate limiter** | Limit request rate (protect yourself or respect a quota) |
| **Fallback** | Degraded response (cached data, default value) |

### Circuit breaker states
```
CLOSED ──(failure rate ≥ threshold in sliding window)──▶ OPEN ──(wait duration)──▶ HALF_OPEN
  ▲                                                       │ calls fail fast              │
  └──────────────(trial calls succeed)────────────────────┴──────(trial calls fail)──────┘
```

```java
@Service
class PricingClient {
    @CircuitBreaker(name = "pricing", fallbackMethod = "cachedPrice")
    @Retry(name = "pricing")
    @TimeLimiter(name = "pricing")         // needs CompletableFuture return type
    public CompletableFuture<Price> price(String sku) { ... }

    CompletableFuture<Price> cachedPrice(String sku, Throwable t) {
        return CompletableFuture.completedFuture(priceCache.getOrDefault(sku, Price.UNKNOWN));
    }
}
```
```yaml
resilience4j:
  circuitbreaker:
    instances:
      pricing:
        sliding-window-size: 20
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        permitted-number-of-calls-in-half-open-state: 3
  retry:
    instances:
      pricing:
        max-attempts: 3
        wait-duration: 200ms
        enable-exponential-backoff: true
```
Resilience4j aspect order (outer → inner): Retry ( CircuitBreaker ( RateLimiter ( TimeLimiter (
Bulkhead ( call ))))).

## 5. Communication styles

| Sync (HTTP/gRPC) | Async (Kafka/RabbitMQ) |
|---|---|
| Simple request/response, immediate answer | Decoupled in time; producer doesn't wait |
| Temporal coupling: callee must be up | Callee can be down; messages wait |
| Cascading failures and latency chains | Eventual consistency; harder debugging |
| Queries, commands needing an answer now | Events ("OrderPlaced"), long workflows, fan-out |

## 6. Data in microservices

- **Database per service**: no shared tables; other services go through the API or events.
- **Saga** for multi-service business transactions:
  - *Choreography*: services react to each other's events (simple, but flow is implicit).
  - *Orchestration*: a coordinator tells each service what to do and runs compensations (explicit;
    tools: Temporal, Camunda, or a hand-written state machine).
- **Outbox pattern** for reliable "update DB + publish event" (see [14](./14-messaging-kafka-rabbitmq.md)).
- **CQRS**: separate write model and read model (read model built from events).
- **API composition** in a gateway/BFF for queries spanning services.

## 7. Gotchas

1. No timeouts on HTTP clients → threads hang when a dependency is slow.
2. Retrying non-idempotent POSTs → duplicate orders/payments.
3. Retries at every layer (client, gateway, mesh) → retry storms that multiply load.
4. Creating a new `RestClient`/`WebClient` per request instead of building from the
   auto-configured builder once (loses tracing/metrics, wastes connections).
5. A "distributed monolith": services that must be deployed together or share a database.
6. Chatty synchronous chains (A → B → C → D): latency adds up and availability multiplies down.

## 8. Interview questions

1. **`RestTemplate` vs `RestClient` vs `WebClient`?**
   `RestTemplate`: legacy synchronous template API. `RestClient`: modern fluent synchronous client
   (preferred for blocking apps). `WebClient`: reactive non-blocking client for WebFlux, streaming
   and massive concurrency.

2. **What are HTTP interface clients?**
   Java interfaces annotated with `@HttpExchange`/`@GetExchange` that Spring turns into client
   proxies backed by `RestClient` or `WebClient`. Boot 4 can register them with `@ImportHttpServices`.

3. **Feign vs HTTP interfaces?**
   Both are declarative. Feign is a Spring Cloud/Netflix-origin library now in maintenance mode; HTTP
   interfaces are built into Spring Framework and are the recommended choice for new code.

4. **What is service discovery? Client-side vs server-side?**
   Finding instance addresses dynamically. Client-side: the client queries the registry (Eureka) and
   load-balances itself. Server-side: a load balancer/router (Kubernetes Service, AWS ALB) does it.

5. **What does an API gateway do?**
   Single entry point: routing, authentication, rate limiting, CORS, TLS, request aggregation and
   transformation.

6. **What is a circuit breaker? Explain its states.**
   A wrapper that stops calling a failing dependency. CLOSED (normal), OPEN (fail fast after failure
   threshold), HALF_OPEN (trial calls decide whether to close or reopen).

7. **Retry vs circuit breaker?**
   Retry handles brief transient errors by trying again; circuit breaker stops trying when a
   dependency is consistently failing. Use both, with the circuit breaker preventing retry storms.

8. **What is a bulkhead?**
   Isolating resources per dependency (thread pools or semaphores) so one slow dependency can't
   consume everything.

9. **What is Spring Cloud Config?**
   A central server serving configuration from Git/Vault to services at startup, with refresh
   support.

10. **How do you handle a distributed transaction across services?**
    Saga (orchestration or choreography) with compensating actions, plus outbox for reliable event
    publishing and idempotent consumers.

11. **How do services communicate?**
    Synchronously (REST, gRPC) or asynchronously (Kafka, RabbitMQ). Prefer async events for
    decoupling and sync for queries that need an immediate answer.

12. **What are the downsides of microservices?**
    Operational complexity, network latency and failures, distributed data consistency, harder
    testing and debugging, need for strong observability and automation.

13. **How do you secure microservices?**
    Validate JWTs at the gateway and in each service (zero trust), OAuth2 client credentials for
    service-to-service calls, mTLS (often via service mesh), secrets management.

14. **What is the strangler fig pattern?**
    Migrate a monolith incrementally by routing specific endpoints to new services through a
    gateway until the monolith can be retired.
