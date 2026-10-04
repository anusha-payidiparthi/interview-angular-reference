# 16 — Packaging, Deployment & Performance

## 1. The executable ("fat"/"uber") JAR

`spring-boot-maven-plugin` (`bootJar` in Gradle) repackages your app into a self-contained JAR:
```
app.jar
├── META-INF/MANIFEST.MF        Main-Class: org.springframework.boot.loader.launch.JarLauncher
│                               Start-Class: com.example.OrdersApplication
├── org/springframework/boot/loader/...   the Boot loader
└── BOOT-INF/
    ├── classes/                your compiled code + application.yml
    ├── lib/                    all dependency JARs (nested, not unpacked)
    ├── classpath.idx
    └── layers.idx
```
The JDK can't load JARs nested inside a JAR, so Boot's `JarLauncher` provides a custom class loader
that reads `BOOT-INF/lib/*.jar`, then calls your `main`. Run with `java -jar app.jar`.

**JAR vs WAR:** JAR embeds the server and runs standalone (the norm for containers). WAR deploys into
an external servlet container (extend `SpringBootServletInitializer`, mark Tomcat `provided`).

## 2. Containers

### Layered Dockerfile (better caching)
Dependencies change rarely, your code changes often; put them in separate image layers.
```dockerfile
FROM eclipse-temurin:21-jre AS builder
WORKDIR /app
COPY target/app.jar app.jar
RUN java -Djarmode=tools -jar app.jar extract --layers --launcher --destination extracted

FROM eclipse-temurin:21-jre
WORKDIR /app
RUN useradd --system app
USER app                                                    # don't run as root
COPY --from=builder /app/extracted/dependencies/ ./
COPY --from=builder /app/extracted/spring-boot-loader/ ./
COPY --from=builder /app/extracted/snapshot-dependencies/ ./
COPY --from=builder /app/extracted/application/ ./
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "org.springframework.boot.loader.launch.JarLauncher"]
```
(Older Boot versions used `-Djarmode=layertools extract`.)

### Buildpacks (no Dockerfile)
```bash
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=acme/orders:1.0
./gradlew bootBuildImage
```
Paketo buildpacks produce an optimized, layered, non-root image with a memory calculator for JVM
flags.

### JVM in containers
- Modern JVMs are container-aware; size the heap with `-XX:MaxRAMPercentage=75` rather than a fixed
  `-Xmx` that ignores the container limit.
- Set CPU/memory requests and limits in Kubernetes; too-low CPU limits slow startup and GC.

## 3. Kubernetes essentials

```yaml
spec:
  containers:
    - name: orders
      image: acme/orders:1.0
      ports: [{ containerPort: 8080 }]
      env:
        - { name: SPRING_PROFILES_ACTIVE, value: prod }
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom: { secretKeyRef: { name: orders-db, key: password } }
      resources:
        requests: { cpu: 500m, memory: 768Mi }
        limits:   { memory: 768Mi }
      livenessProbe:  { httpGet: { path: /actuator/health/liveness,  port: 8080 }, initialDelaySeconds: 20 }
      readinessProbe: { httpGet: { path: /actuator/health/readiness, port: 8080 } }
      startupProbe:   { httpGet: { path: /actuator/health/liveness,  port: 8080 }, failureThreshold: 30, periodSeconds: 2 }
      lifecycle:
        preStop: { sleep: { seconds: 10 } }    # let the load balancer stop routing first
```
Config via env vars/ConfigMaps, secrets via Secrets, scale with an HPA on CPU or custom metrics.

## 4. Graceful shutdown

```yaml
server.shutdown: graceful                       # default since Boot 3.4 (was immediate)
spring.lifecycle.timeout-per-shutdown-phase: 30s
```
On SIGTERM: readiness goes `OUT_OF_SERVICE`, the web server stops accepting new requests, in-flight
requests finish (up to the timeout), then beans are destroyed (`@PreDestroy`, pools closed,
Kafka consumers stopped). Combine with a `preStop` sleep so Kubernetes removes the pod from
endpoints before shutdown begins.

## 5. Startup time and memory

| Technique | Effect | Trade-off |
|---|---|---|
| **GraalVM native image** (`./mvnw -Pnative native:compile`) | Startup in tens of ms, low RSS memory | Long builds, closed-world: reflection/proxies need hints; no JIT peak optimizations (lower peak throughput w/o PGO); some libraries unsupported |
| **AOT processing** (Spring AOT) | Pre-computes bean definitions at build time (required for native, optional on JVM) | Fixed classpath/profiles at build time |
| **CDS / AOT cache** (Class Data Sharing; Java 24+ `-XX:AOTCache` from JEP 483) | 30–50%+ faster JVM startup, still full JIT | Extra training run during build |
| **CRaC** (Coordinated Restore at Checkpoint) | Restore a warmed-up JVM snapshot in ms | Linux only, needs CRaC JDK, care with secrets/connections in the snapshot |
| Lazy initialization | Faster startup | Errors surface later, first requests slower |
| Trim starters/auto-config | Fewer beans | — |

```bash
# CDS with Boot 3.3+
java -Djarmode=tools -jar app.jar extract --destination app
java -XX:ArchiveClassesAtExit=app.jsa -Dspring.context.exit=onRefresh -jar app/app.jar   # training run
java -XX:SharedArchiveFile=app.jsa -jar app/app.jar
```
Native images shine for serverless (scale to zero, fast cold start) and CLIs; long-running
high-throughput services often stay on the JVM.

## 6. Runtime performance checklist

**Database (usually the bottleneck):**
- Fix N+1 queries; use projections; add indexes; paginate (keyset for deep pages).
- Tune **HikariCP**: `maximum-pool-size` small (often 10–20). More connections than the DB can run
  in parallel just adds contention. Pool size × instances must fit the DB's `max_connections`.
- `spring.jpa.open-in-view=false`; short transactions; `readOnly = true` on reads; JDBC batching.

**Web/threads:**
- Enable **virtual threads** (Java 21+) for blocking I/O workloads, or tune
  `server.tomcat.threads.max` and `accept-count`.
- Timeouts on every outbound call; circuit breakers; bulkheads.
- Response compression: `server.compression.enabled=true`; HTTP/2: `server.http2.enabled=true`.

**Caching:** Caffeine/Redis for hot read data; HTTP caching (ETag, `Cache-Control`).

**JVM:** G1 (default) for most; ZGC (generational) for low-latency large heaps; right-size heap;
watch GC pauses (`jvm.gc.pause` metric).

**Measure first:** Actuator metrics + Prometheus/Grafana, tracing to find slow spans, async-profiler
or JFR (Java Flight Recorder) for CPU/allocation hotspots, load test with Gatling/k6/JMeter.

## 7. Gotchas

1. Fat JAR copied as one Docker layer → every build re-pushes 100+ MB.
2. Fixed `-Xmx` larger than the container limit → OOMKilled.
3. Liveness probe too aggressive during slow startup → restart loop (use a `startupProbe`).
4. No graceful shutdown → dropped requests on every deploy.
5. Huge connection pools "for performance" → DB overload.
6. Native image surprises: reflection-based libraries failing at runtime without hints; test the
   native build in CI.

## 8. Interview questions

1. **How does `java -jar` work for a Spring Boot fat JAR?**
   The manifest's `Main-Class` is Boot's `JarLauncher`, which creates a class loader able to read
   nested JARs in `BOOT-INF/lib` and then invokes the `Start-Class` main method.

2. **How do you containerize a Spring Boot app?**
   A layered Dockerfile (extract layers with jarmode tools), or `spring-boot:build-image` with
   Cloud Native Buildpacks. Run as non-root, size heap with `MaxRAMPercentage`.

3. **What are layered JARs?**
   A JAR layout that separates dependencies, loader, snapshot dependencies and application classes
   so Docker can cache unchanged layers.

4. **What is graceful shutdown?**
   On SIGTERM, stop accepting new requests, finish in-flight ones within a timeout, then close
   resources. `server.shutdown=graceful` (default since 3.4).

5. **What is GraalVM native image? Pros and cons?**
   Ahead-of-time compilation to a native executable. Pros: very fast startup, low memory. Cons: long
   builds, reflection configuration, reduced dynamic features, potentially lower peak throughput.

6. **How do you improve startup time?**
   CDS/AOT cache, Spring AOT, native image, CRaC, lazy init, removing unused auto-configuration.

7. **How do you tune a Spring Boot app's performance?**
   Measure with metrics/tracing/profiling; fix DB access (N+1, indexes, pool size); add caching; set
   timeouts; enable virtual threads or tune thread pools; right-size JVM/GC.

8. **How big should the connection pool be?**
   Small: roughly the number of concurrent queries the DB can execute efficiently (often 10–20 per
   instance), validated by load testing; total across instances under the DB connection limit.

9. **How do you deploy without downtime?**
   Rolling or blue-green/canary deployments, readiness probes, graceful shutdown, backward-compatible
   DB migrations (expand → migrate → contract).

10. **How do you pass configuration to a containerized app?**
    Env vars (`SPRING_DATASOURCE_URL`), mounted ConfigMaps/Secrets (`configtree`), or a config server.
