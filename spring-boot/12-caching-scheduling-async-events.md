# 12 — Caching, Scheduling, Async & Events

## 1. Caching

```java
@SpringBootApplication
@EnableCaching
class App { }

@Service
class ProductService {

    @Cacheable(value = "products", key = "#id", unless = "#result == null")
    public ProductDto get(Long id) { ... }               // DB hit only on cache miss

    @CachePut(value = "products", key = "#result.id")
    public ProductDto update(ProductUpdate u) { ... }    // always runs, refreshes the cache

    @CacheEvict(value = "products", key = "#id")
    public void delete(Long id) { ... }

    @CacheEvict(value = "products", allEntries = true)
    @Scheduled(fixedRate = 3_600_000)
    public void evictAll() { }

    @Cacheable(value = "prices", key = "#sku + ':' + #currency", condition = "#currency != 'XXX'")
    public Price price(String sku, String currency) { ... }
}
```

| Annotation | Behavior |
|---|---|
| `@Cacheable` | Return cached value if present, otherwise run and cache the result |
| `@CachePut` | Always run and update the cache |
| `@CacheEvict` | Remove entries (`allEntries`, `beforeInvocation`) |
| `@Caching` | Combine several of the above |
| `@CacheConfig` | Class-level defaults (cache names, key generator) |

Default key: method params (`SimpleKey`). `condition` is checked before the call, `unless` after
(can use `#result`).

### Cache providers
Boot auto-detects: JCache (Ehcache 3), Hazelcast, Infinispan, Couchbase, **Redis**, **Caffeine**,
and falls back to a simple `ConcurrentHashMap` (no eviction or TTL, so not for production).

```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 10m
  data:
    redis:
      host: localhost
```
```yaml
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=10000,expireAfterWrite=5m
```
**Local (Caffeine)** vs **distributed (Redis)**: local is fastest but each instance has its own copy
(stale data across instances); distributed is consistent across instances but costs a network hop
and needs serialization.

### Caching strategies
- **Cache-aside** (what `@Cacheable` does): read from cache, on miss load from DB and populate.
- **Write-through**: write to cache and DB together. **Write-behind**: write to cache, DB later.
- Invalidation is the hard part: TTLs, evict on update, or events.
- **Cache stampede**: many requests miss at once and hit the DB. Mitigate with `@Cacheable(sync = true)`
  (one thread loads per key per instance), jittered TTLs, or refresh-ahead.

## 2. Scheduling

```java
@SpringBootApplication
@EnableScheduling
class App { }

@Component
class Jobs {
    @Scheduled(fixedRate = 5000)                 // every 5 s from START of last run
    void poll() { }

    @Scheduled(fixedDelay = 5000, initialDelay = 10000)   // 5 s after END of last run
    void cleanup() { }

    @Scheduled(cron = "0 0 2 * * MON-FRI", zone = "America/Los_Angeles")   // 02:00 weekdays
    void nightlyReport() { }

    @Scheduled(fixedRateString = "${jobs.sync.rate:PT1M}")  // from config, ISO-8601 duration
    void sync() { }
}
```
Spring cron has **6 fields**: `second minute hour day-of-month month day-of-week`. Macros:
`@hourly`, `@daily`, `@weekly`.

**Gotchas:**
- By default the scheduler has **one thread**, so a slow job delays all others. Set
  `spring.task.scheduling.pool.size=5` (or enable virtual threads).
- **Every instance runs the job.** With 3 replicas, a nightly job runs 3 times. Use **ShedLock**
  (`@SchedulerLock(name = "nightlyReport")` with a DB/Redis lock), a leader election, a Kubernetes
  CronJob, or Quartz in clustered mode.
- Exceptions are logged and the next execution still happens.

**Quartz** (`spring-boot-starter-quartz`) adds persistent jobs, clustering, misfire handling and
dynamic scheduling. **Spring Batch** is for large chunk-oriented processing (read → process →
write with restartability), often triggered by a scheduler.

## 3. `@Async`

```java
@SpringBootApplication
@EnableAsync
class App { }

@Service
class NotificationService {
    @Async
    public void sendWelcomeEmail(User u) { ... }                 // fire and forget

    @Async("reportExecutor")                                     // a specific executor
    public CompletableFuture<Report> buildReport(Long id) {
        return CompletableFuture.completedFuture(generate(id));
    }
}

@Configuration
class AsyncConfig {
    @Bean
    ThreadPoolTaskExecutor reportExecutor() {
        var ex = new ThreadPoolTaskExecutor();
        ex.setCorePoolSize(4);
        ex.setMaxPoolSize(8);
        ex.setQueueCapacity(100);
        ex.setThreadNamePrefix("report-");
        ex.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        ex.initialize();
        return ex;
    }
}
```
Boot auto-configures a default `ThreadPoolTaskExecutor` (`applicationTaskExecutor`), tunable via
`spring.task.execution.pool.*`.

**How it works / gotchas:**
- Proxy-based → **self-invocation doesn't run async**; method must be public/non-final on a bean.
- Return type must be `void`, `Future` or `CompletableFuture`.
- Exceptions from `void` methods are lost unless you configure an `AsyncUncaughtExceptionHandler`
  (implement `AsyncConfigurer`).
- The async thread doesn't inherit the caller's **transaction**, **SecurityContext** or **MDC**
  (use a `TaskDecorator` to copy MDC/security context).
- ThreadPoolExecutor semantics: threads grow beyond core size only after the **queue is full**, so
  an unbounded queue means max size is never used.

## 4. Virtual threads (Java 21+)

```yaml
spring.threads.virtual.enabled: true
```
Boot then runs Tomcat/Jetty request handling, `@Async`, `@Scheduled`, Kafka/Rabbit listeners and
the default task executor on **virtual threads**. Blocking I/O no longer ties up a scarce platform
thread, so a simple blocking MVC app handles tens of thousands of concurrent requests.

Caveats: don't pool virtual threads; CPU-bound work gains nothing; still bound by downstream limits
(DB pool size!), so add limits like a semaphore or `@ConcurrencyLimit`. Pinning on `synchronized`
was largely fixed in Java 24 (JEP 491). `ThreadLocal`-heavy code multiplies memory per thread.

## 5. Application events

Decouple components in the same app: the publisher doesn't know who listens.

```java
public record OrderPlaced(Long orderId, Long customerId) {}       // any object can be an event

@Service
class OrderService {
    private final ApplicationEventPublisher events;
    OrderService(ApplicationEventPublisher events) { this.events = events; }

    @Transactional
    public Order place(CreateOrderRequest req) {
        Order o = repo.save(...);
        events.publishEvent(new OrderPlaced(o.getId(), o.getCustomerId()));
        return o;
    }
}

@Component
class LoyaltyListener {
    @EventListener                                   // synchronous, same thread & transaction
    void award(OrderPlaced e) { loyalty.addPoints(e.customerId()); }
}

@Component
class EmailListener {
    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)   // only after commit
    void send(OrderPlaced e) { email.confirm(e.orderId()); }
}
```
- `@EventListener` is **synchronous** by default: it runs in the publisher's thread and
  transaction, and an exception propagates to the publisher.
- `@TransactionalEventListener` defers to a transaction phase (`AFTER_COMMIT` default,
  `AFTER_ROLLBACK`, `AFTER_COMPLETION`, `BEFORE_COMMIT`). If there's no transaction, the event is
  dropped unless `fallbackExecution = true`.
- Add `@Async` for asynchronous handling; filter with `condition = "#e.amount > 100"`; order with `@Order`.
- In-memory events are **lost on crash**. For guaranteed delivery use Spring Modulith's event
  publication registry (persists events, republishes incomplete ones) or the outbox pattern with a
  broker.

## 6. Retry and resilience (in-process)

```java
// Spring Framework 7 (Boot 4)
@Configuration @EnableResilientMethods
class ResilienceConfig { }

@Service
class InventoryClient {
    @Retryable(includes = TransientException.class, maxRetries = 3, delay = 200, multiplier = 2)
    public Stock check(String sku) { ... }

    @ConcurrencyLimit(10)                  // at most 10 concurrent calls (great with virtual threads)
    public Stock reserve(String sku) { ... }
}
```
On Boot 3 use **Spring Retry** (`@EnableRetry`, `@Retryable`, `@Recover`) or **Resilience4j**
(`@Retry`, `@CircuitBreaker`, `@RateLimiter`, `@Bulkhead`; see [13](./13-rest-clients-and-microservices.md)).
Only retry **idempotent** operations and transient failures, with exponential backoff and jitter.

## 7. Interview questions

1. **How does caching work in Spring Boot?**
   `@EnableCaching` activates a proxy that intercepts `@Cacheable`/`@CachePut`/`@CacheEvict`
   methods and reads/writes a `CacheManager` (Caffeine, Redis, etc.) using a key derived from the
   arguments.

2. **`@Cacheable` vs `@CachePut`?**
   `@Cacheable` skips the method on a hit; `@CachePut` always executes and updates the cache.

3. **Local vs distributed cache?**
   Local (Caffeine): fastest, per instance, can be inconsistent across instances. Distributed
   (Redis): shared and consistent, network/serialization cost.

4. **How do you handle cache invalidation?**
   TTLs, `@CacheEvict` on writes, `@CachePut` on updates, event-driven eviction across instances.

5. **`fixedRate` vs `fixedDelay` vs `cron`?**
   `fixedRate`: interval from start to start. `fixedDelay`: interval from end to next start. `cron`:
   calendar-based expression (6 fields in Spring).

6. **How do you prevent a scheduled job from running on every instance?**
   Distributed lock (ShedLock), leader election, Quartz clustering, or an external scheduler
   (Kubernetes CronJob).

7. **How does `@Async` work? What are its pitfalls?**
   A proxy submits the method call to a `TaskExecutor`. Pitfalls: self-invocation, void methods
   swallow exceptions, no transaction/security/MDC propagation, misconfigured pool sizes.

8. **What executor does `@Async` use by default?**
   Boot's auto-configured `ThreadPoolTaskExecutor` (`applicationTaskExecutor`), or a virtual-thread
   executor when virtual threads are enabled.

9. **What are virtual threads and how do you enable them in Boot?**
   Lightweight JVM-managed threads (Java 21). `spring.threads.virtual.enabled=true` makes the web
   server, `@Async`, schedulers and listeners use them.

10. **How do application events work?**
    `ApplicationEventPublisher.publishEvent(obj)`; listeners annotated `@EventListener` receive
    events by type, synchronously by default.

11. **`@EventListener` vs `@TransactionalEventListener`?**
    The former runs immediately within the publisher's transaction; the latter runs at a transaction
    phase (by default after commit).

12. **How do you implement retries?**
    Framework 7 `@Retryable`, Spring Retry, or Resilience4j, with backoff, only for idempotent calls
    and transient errors.
