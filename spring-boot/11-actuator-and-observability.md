# 11 — Actuator, Logging & Observability

## 1. Spring Boot Actuator

`spring-boot-starter-actuator` adds production endpoints under `/actuator`.

| Endpoint | Shows |
|---|---|
| `health` | Up/down status, with details per component (DB, disk, Redis, Kafka…) |
| `info` | App info (build, git, custom) |
| `metrics` | Micrometer metrics (`/actuator/metrics/http.server.requests`) |
| `prometheus` | Metrics in Prometheus format (needs `micrometer-registry-prometheus`) |
| `env` | Environment properties (values masked) |
| `configprops` | `@ConfigurationProperties` beans |
| `beans` | All beans |
| `conditions` | Auto-configuration report |
| `mappings` | All request mappings |
| `loggers` | View and **change log levels at runtime** |
| `threaddump`, `heapdump` | JVM diagnostics |
| `httpexchanges` | Recent HTTP requests (needs a repository bean) |
| `scheduledtasks`, `caches`, `flyway`, `liquibase`, `startup`, `sbom` | |
| `shutdown` | Graceful shutdown (disabled by default) |

Only `health` is exposed over HTTP by default.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true            # liveness/readiness groups (auto-enabled on Kubernetes)
  server:
    port: 9091                   # serve actuator on a separate internal port
  info:
    git.mode: full
```
Secure it: expose only what you need, put it on an internal port or behind Spring Security
(`EndpointRequest.toAnyEndpoint()` matcher), never expose `heapdump`/`env` publicly.

## 2. Health checks and Kubernetes probes

```java
@Component
class PaymentProviderHealth implements HealthIndicator {
    private final PaymentClient client;
    PaymentProviderHealth(PaymentClient client) { this.client = client; }

    @Override
    public Health health() {
        try {
            client.ping();
            return Health.up().withDetail("provider", "stripe").build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```
Bean name `paymentProviderHealth` → appears as `paymentProvider` in `/actuator/health`.

- **Liveness** (`/actuator/health/liveness`): "Is the process healthy, or should it be restarted?"
  Don't include external dependencies here, or a DB outage will restart every pod.
- **Readiness** (`/actuator/health/readiness`): "Can it accept traffic right now?" May include
  critical dependencies; goes `OUT_OF_SERVICE` during startup and graceful shutdown.

```yaml
management.endpoint.health.group.readiness.include: readinessState,db,redis
```
```yaml
# Kubernetes
livenessProbe:  { httpGet: { path: /actuator/health/liveness,  port: 9091 } }
readinessProbe: { httpGet: { path: /actuator/health/readiness, port: 9091 } }
```

## 3. Custom endpoint and info

```java
@Component
@Endpoint(id = "features")
class FeaturesEndpoint {
    @ReadOperation  Map<String, Boolean> features() { return flags.all(); }
    @WriteOperation void toggle(@Selector String name, boolean enabled) { flags.set(name, enabled); }
}
```
```yaml
info:
  app:
    name: orders
    version: '@project.version@'      # Maven resource filtering
```
Add `build-info` goal to `spring-boot-maven-plugin` and `git-commit-id-maven-plugin` for build/git
info automatically.

## 4. Metrics with Micrometer

Micrometer is a vendor-neutral metrics facade ("SLF4J for metrics"): instrument once, export to
Prometheus, Datadog, New Relic, CloudWatch, OTLP, etc.

Built-in metrics: `http.server.requests` (latency, count by URI/status), `jvm.memory.used`,
`jvm.gc.pause`, `jvm.threads.live`, `hikaricp.connections.active`, `process.cpu.usage`,
`tomcat.threads.busy`, cache and Kafka metrics.

Meter types: **Counter** (only goes up), **Gauge** (current value), **Timer** (latency + count),
**DistributionSummary** (sizes), **LongTaskTimer** (in-flight durations).

```java
@Service
class CheckoutService {
    private final Counter ordersPlaced;
    private final Timer paymentTimer;

    CheckoutService(MeterRegistry registry) {
        this.ordersPlaced = Counter.builder("orders.placed")
                .description("Orders successfully placed")
                .tag("channel", "web")
                .register(registry);
        this.paymentTimer = registry.timer("payment.duration");
    }

    void checkout(Cart cart) {
        paymentTimer.record(() -> payments.charge(cart));
        ordersPlaced.increment();
    }
}

// Or declaratively (needs the aspect bean / AOP)
@Timed(value = "report.generate", percentiles = {0.5, 0.95, 0.99})
public Report generate() { ... }
```
**Avoid high-cardinality tags** (user IDs, order IDs, raw URLs): each unique tag combination is a
new time series and can overwhelm the metrics backend.

## 5. Distributed tracing

A **trace** follows a request across services; each unit of work is a **span**. IDs are
propagated in headers (W3C `traceparent`).

Boot 3+ uses **Micrometer Tracing** (replacing Spring Cloud Sleuth) with an OpenTelemetry or Brave
bridge and an exporter (OTLP, Zipkin). Boot 4 adds a `spring-boot-starter-opentelemetry`.

```yaml
management:
  tracing:
    sampling:
      probability: 0.1          # sample 10% in prod (default 0.1)
  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces
logging:
  pattern:
    correlation: "[${spring.application.name:},%X{traceId:-},%X{spanId:-}] "
```
Trace and span IDs are put in the logging MDC automatically, so every log line can be correlated.
`RestClient`, `WebClient`, `KafkaTemplate`, JDBC (with extra libraries) propagate context
automatically when built from the auto-configured builders.

**The Observation API** (`@Observed` or `Observation.createNotStarted(...).observe(...)`) creates a
timer metric **and** a span from one instrumentation.

### Three pillars of observability
**Logs** (what happened), **metrics** (how much / how fast, aggregated), **traces** (where time went
across services). Typical stack: Prometheus + Grafana, Loki/ELK, Tempo/Jaeger, or a vendor like
Datadog.

## 6. Logging

Boot uses **SLF4J** as the API and **Logback** as the default implementation (Log4j2 is an
alternative via `spring-boot-starter-log4j2`). Default output: console, `INFO` level.

```java
@Slf4j   // Lombok, or: private static final Logger log = LoggerFactory.getLogger(OrderService.class);
class OrderService {
    void place(Order o) {
        log.info("Placing order {} for customer {}", o.getId(), o.getCustomerId());   // parameterized
        log.debug("Order details: {}", o);                                           // not built if DEBUG off
        log.error("Payment failed for order {}", o.getId(), exception);              // exception last
    }
}
```
```yaml
logging:
  level:
    root: INFO
    com.example: DEBUG
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE      # SQL parameter values
  file:
    name: logs/app.log
  structured:
    format:
      console: ecs                         # JSON logs (Boot 3.4+): ecs, gelf, logstash
```
- Change levels at runtime: `POST /actuator/loggers/com.example {"configuredLevel":"DEBUG"}`.
- Custom config: `logback-spring.xml` (supports `<springProfile>` and `<springProperty>`).
- Use **structured JSON logs** in production so log platforms can index fields.
- Put request context (correlation ID, user ID) into the **MDC**.
- Never log passwords, tokens or personal data.

Log levels: `TRACE` < `DEBUG` < `INFO` < `WARN` < `ERROR`.

## 7. Gotchas

1. Exposing all actuator endpoints publicly (`include: "*"`) → leaks env/heap dumps.
2. Liveness checks that include the DB → cascading restarts during a DB blip.
3. High-cardinality metric tags.
4. String concatenation in log calls (`log.debug("x=" + x)`) → work done even when disabled.
5. Logging an exception's message only (`e.getMessage()`) loses the stack trace; pass `e` itself.
6. 100% trace sampling in production → cost and overhead.

## 8. Interview questions

1. **What is Spring Boot Actuator?**
   A module providing production-ready endpoints for health, metrics, info, environment, loggers,
   thread/heap dumps and more, over HTTP or JMX.

2. **How do you expose and secure actuator endpoints?**
   `management.endpoints.web.exposure.include`, a separate `management.server.port`, and Spring
   Security rules using `EndpointRequest`.

3. **How do you create a custom health check?**
   Implement `HealthIndicator` (or `ReactiveHealthIndicator`) as a bean returning
   `Health.up()/down()` with details.

4. **Liveness vs readiness?**
   Liveness: should the container be restarted? Readiness: should it receive traffic? Boot exposes
   both as health groups for Kubernetes probes.

5. **What is Micrometer?**
   A vendor-neutral metrics facade used by Boot to record and export metrics to Prometheus, Datadog,
   OTLP, etc.

6. **Counter vs gauge vs timer?**
   Counter: monotonically increasing count. Gauge: a current value that goes up and down. Timer:
   duration and count of events, with percentiles/histograms.

7. **How do you implement distributed tracing?**
   Micrometer Tracing with an OpenTelemetry bridge and OTLP/Zipkin exporter; trace context is
   propagated through headers and added to the log MDC.

8. **What replaced Spring Cloud Sleuth?**
   Micrometer Tracing (Boot 3+).

9. **What logging framework does Spring Boot use?**
   SLF4J facade with Logback by default; Log4j2 optional.

10. **How do you change the log level without restarting?**
    The `/actuator/loggers/{name}` endpoint (POST a `configuredLevel`).

11. **How do you correlate logs across services?**
    Propagate a trace/correlation ID (W3C `traceparent`), store it in the MDC, include it in the log
    pattern or structured JSON.

12. **What's `logback-spring.xml` vs `logback.xml`?**
    The `-spring` variant is loaded by Boot after the environment is ready, so it supports Spring
    profiles and properties.
