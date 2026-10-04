# 15 — WebFlux & Reactive Programming

## 1. Why reactive?

Traditional Spring MVC uses **one thread per request**. If a request waits 200 ms for a DB or
remote call, its thread sits blocked. With a 200-thread pool, you can't serve more than 200
concurrent slow requests.

**Spring WebFlux** is a non-blocking stack: a few event-loop threads (≈ CPU cores) handle many
connections; I/O is asynchronous and work resumes via callbacks when data arrives. Built on
**Project Reactor** and the **Reactive Streams** spec (with backpressure), running on Netty by
default.

## 2. Reactor: `Mono` and `Flux`

| Type | Emits |
|---|---|
| `Mono<T>` | 0 or 1 element, then complete (or error) |
| `Flux<T>` | 0..N elements, then complete (or error); can be infinite |

```java
Mono<User> user = userRepo.findById(id);                 // nothing happens yet
Flux<Order> orders = orderRepo.findByUserId(id);

Mono<UserDashboard> dashboard = user
        .switchIfEmpty(Mono.error(new UserNotFoundException(id)))
        .flatMap(u -> orders.collectList()
                .map(list -> new UserDashboard(u, list)))
        .timeout(Duration.ofSeconds(2))
        .onErrorResume(TimeoutException.class, e -> Mono.just(UserDashboard.empty()));

dashboard.subscribe(d -> log.info("Got {}", d));          // NOW it executes
```
**Nothing happens until you subscribe.** In WebFlux the framework subscribes to what your
controller returns.

### Key operators
| Operator | Purpose |
|---|---|
| `map` | Synchronous 1:1 transform |
| `flatMap` | Async transform returning a publisher; runs inner publishers **concurrently** (order not preserved) |
| `concatMap` | Like `flatMap` but sequential, preserves order |
| `filter`, `take`, `skip`, `distinct` | Filtering |
| `zip`, `zipWith` | Combine results of several publishers (parallel calls) |
| `merge`, `concat` | Interleave vs append streams |
| `switchIfEmpty`, `defaultIfEmpty` | Handle empty |
| `onErrorResume`, `onErrorReturn`, `onErrorMap`, `retryWhen` | Error handling |
| `timeout`, `delayElements`, `buffer`, `window` | Time and batching |
| `doOnNext`, `doOnError`, `log` | Side effects / debugging (don't put business logic here) |
| `subscribeOn`, `publishOn` | Switch the thread (scheduler) |

`map` vs `flatMap`: `map(u -> u.getName())` for plain values; `flatMap(u -> client.fetch(u.id()))`
when the function returns a `Mono`/`Flux`, otherwise you'd get `Mono<Mono<T>>`.

## 3. WebFlux controllers

### Annotated (same annotations as MVC)
```java
@RestController
@RequestMapping("/api/users")
class UserController {
    private final UserRepository repo;          // ReactiveCrudRepository (R2DBC/Mongo reactive)

    @GetMapping("/{id}")
    Mono<ResponseEntity<User>> get(@PathVariable Long id) {
        return repo.findById(id)
                   .map(ResponseEntity::ok)
                   .defaultIfEmpty(ResponseEntity.notFound().build());
    }

    @GetMapping
    Flux<User> all() { return repo.findAll(); }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    Mono<User> create(@Valid @RequestBody Mono<User> user) { return user.flatMap(repo::save); }

    @GetMapping(value = "/events", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    Flux<PriceTick> stream() {                  // server-sent events: an infinite stream
        return Flux.interval(Duration.ofSeconds(1)).map(i -> pricing.current());
    }
}
```

### Functional endpoints
```java
@Bean
RouterFunction<ServerResponse> routes(UserHandler h) {
    return RouterFunctions.route()
            .GET("/api/users/{id}", h::get)
            .POST("/api/users", h::create)
            .build();
}
```

## 4. Backpressure

The subscriber tells the publisher how many items it can handle (`request(n)`), so a fast producer
doesn't overwhelm a slow consumer. Strategies when the consumer can't keep up:
`onBackpressureBuffer`, `onBackpressureDrop`, `onBackpressureLatest`, `limitRate(n)`.

## 5. The golden rule: never block the event loop

```java
// WRONG: blocks a Netty event-loop thread → the whole server stalls
Mono<Report> r = Mono.just(jdbcTemplate.queryForObject(...));
someMono.block();

// If you must call blocking code, offload it
Mono<Report> r = Mono.fromCallable(() -> legacyBlockingClient.fetch())
                     .subscribeOn(Schedulers.boundedElastic());
```
Use reactive drivers end to end: **R2DBC** (SQL), reactive MongoDB/Redis/Cassandra, `WebClient`,
Reactor Kafka. JPA/JDBC are blocking. **BlockHound** detects blocking calls in tests.

## 6. MVC vs WebFlux

| | Spring MVC | Spring WebFlux |
|---|---|---|
| Model | Blocking, thread per request | Non-blocking, event loop |
| Server | Tomcat/Jetty (Servlet) | Netty (or Servlet containers in async mode) |
| Data access | JDBC, JPA | R2DBC, reactive NoSQL drivers |
| Code style | Imperative, easy to read/debug | Functional/reactive chains, steeper learning curve, harder stack traces |
| Context | `ThreadLocal` works (security, MDC, transactions) | Use Reactor `Context` instead of `ThreadLocal` |
| Best for | Most CRUD/business apps | Streaming, very high concurrency with I/O, gateways, many parallel remote calls |

**Virtual threads changed the trade-off.** With `spring.threads.virtual.enabled=true`, MVC with
blocking code scales to very high concurrency with simple imperative code. WebFlux remains valuable
for streaming (SSE, WebSockets), backpressure and fully non-blocking pipelines like Spring Cloud
Gateway. Good interview answer: *"Default to MVC (with virtual threads on Java 21+); choose WebFlux
when you need streaming/backpressure or the whole stack is already reactive."*

You can use `WebClient` in an MVC app without switching to WebFlux.

## 7. Error handling and testing

```java
// global error handling works with @RestControllerAdvice too
@ExceptionHandler(UserNotFoundException.class)
ProblemDetail notFound(UserNotFoundException e) { ... }

// StepVerifier for reactive streams
StepVerifier.create(service.findActiveUsers())
        .expectNextMatches(u -> u.isActive())
        .expectNextCount(2)
        .verifyComplete();

// WebTestClient for endpoints
@WebFluxTest(UserController.class)
class UserControllerTest {
    @Autowired WebTestClient client;
    @MockitoBean UserRepository repo;

    @Test void get() {
        when(repo.findById(1L)).thenReturn(Mono.just(new User(1L, "Ann")));
        client.get().uri("/api/users/1").exchange()
              .expectStatus().isOk()
              .expectBody().jsonPath("$.name").isEqualTo("Ann");
    }
}
```

## 8. Gotchas

1. Calling `.block()` or blocking I/O on event-loop threads.
2. Forgetting to return/subscribe: building a `Mono` and never subscribing → nothing happens.
3. Subscribing twice re-executes the pipeline (cold publishers): two HTTP calls. Use `cache()` if needed.
4. Using `ThreadLocal`-based libraries (MDC, `SecurityContextHolder`) without context propagation.
5. Using `flatMap` when order matters (use `concatMap`/`flatMapSequential`).
6. Mixing JPA with WebFlux and expecting non-blocking behavior.

## 9. Interview questions

1. **What is reactive programming?**
   Programming with asynchronous data streams and change propagation, using non-blocking I/O and
   backpressure (Reactive Streams: `Publisher`, `Subscriber`, `Subscription`, `Processor`).

2. **`Mono` vs `Flux`?**
   `Mono` emits at most one item; `Flux` emits zero to many.

3. **Spring MVC vs WebFlux? When would you use WebFlux?**
   MVC is blocking thread-per-request; WebFlux is non-blocking on an event loop. WebFlux fits
   streaming, high concurrency with I/O-bound work and fully reactive stacks; MVC (with virtual
   threads) fits most apps.

4. **What is backpressure?**
   A mechanism for consumers to signal demand so producers don't overwhelm them.

5. **`map` vs `flatMap` in Reactor?**
   `map` applies a synchronous function; `flatMap` applies an async function returning a publisher and
   flattens it (concurrently, unordered).

6. **What happens if you block in WebFlux?**
   You tie up one of the few event-loop threads, collapsing throughput. Offload to
   `Schedulers.boundedElastic()` if unavoidable.

7. **Can you use JPA with WebFlux?**
   Not non-blockingly; JPA/JDBC are blocking. Use R2DBC or wrap blocking calls on a bounded elastic
   scheduler.

8. **`subscribeOn` vs `publishOn`?**
   `subscribeOn` chooses the scheduler where the source subscription (and upstream) runs; `publishOn`
   switches the scheduler for downstream operators after that point.

9. **Hot vs cold publishers?**
   Cold publishers start producing per subscriber (an HTTP call per subscription). Hot publishers emit
   regardless of subscribers (a price ticker); late subscribers miss earlier items.

10. **Do virtual threads make WebFlux obsolete?**
    Not entirely. They remove the main scalability reason for many CRUD apps, but WebFlux still offers
    backpressure, streaming and composable async pipelines.

11. **How do you test reactive code?**
    `StepVerifier` for publishers, `WebTestClient` for endpoints, BlockHound to catch blocking calls.
