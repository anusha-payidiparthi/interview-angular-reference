# 17 — What's New in Spring Boot 3 & 4

Interviewers ask "what changed in Boot 3?" (most codebases migrated from 2.x recently) and,
increasingly, "what's new in Boot 4?". Know the headline items and *why* they matter.

## 1. Version timeline

| Version | Released | Spring Framework | Java baseline | Highlights |
|---|---|---|---|---|
| 2.7 | May 2022 | 5.3 | 8 | Last 2.x; new auto-config `.imports` file; now EOL |
| **3.0** | Nov 2022 | 6.0 | **17** | Jakarta EE, native images, observability |
| 3.1 | May 2023 | 6.0 | 17 | `@ServiceConnection`, Testcontainers at dev time, Docker Compose, SSL bundles |
| 3.2 | Nov 2023 | 6.1 | 17 | **Virtual threads**, `RestClient`, `JdbcClient`, CRaC |
| 3.3 | May 2024 | 6.1 | 17 | CDS support, SBOM endpoint |
| 3.4 | Nov 2024 | 6.2 | 17 | Structured logging, `@MockitoBean`, graceful shutdown default, `MockMvcTester` |
| 3.5 | May 2025 | 6.2 | 17 | Last 3.x line |
| **4.0** | Nov 2025 | **7.0** | 17 (21/25 recommended) | Modularization, Jackson 3, API versioning, HTTP service clients, resilience |

Boot releases a minor version every **May and November**; each minor gets about 13 months of open
source support.

## 2. Spring Boot 3.0: the big migration

### `javax.*` → `jakarta.*`
Java EE moved to the Eclipse Foundation as **Jakarta EE**, and the package namespace changed in
Jakarta EE 9.
```java
// Boot 2                                  // Boot 3+
import javax.persistence.Entity;           import jakarta.persistence.Entity;
import javax.validation.constraints.*;     import jakarta.validation.constraints.*;
import javax.servlet.http.*;               import jakarta.servlet.http.*;
import javax.annotation.PostConstruct;     import jakarta.annotation.PostConstruct;
```
Every library using these APIs needed a Jakarta-compatible version (Hibernate 6, Tomcat 10,
etc.). Tools: OpenRewrite recipes, IntelliJ migration, Spring Boot Migrator.

### Other 3.0 changes
- **Java 17 minimum** (records, text blocks, sealed classes, pattern matching usable everywhere).
- **Spring Security 6:** `WebSecurityConfigurerAdapter` removed → `SecurityFilterChain` beans;
  `authorizeRequests` → `authorizeHttpRequests`; `antMatchers` → `requestMatchers`;
  `@EnableGlobalMethodSecurity` → `@EnableMethodSecurity`.
- **Auto-config registration:** `spring.factories` no longer used for auto-configurations; use
  `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
- **Observability:** Micrometer Observation API + **Micrometer Tracing** replace Spring Cloud Sleuth.
- **AOT + GraalVM native image** support built in.
- **`ProblemDetail`** (RFC 7807, now 9457) error responses.
- **HTTP interface clients** (`@HttpExchange`).
- **Trailing slash matching disabled** (`/users/` no longer matches `/users`).
- **Hibernate 6** (new type system, changed ID generator defaults).
- Property renames, e.g. `spring.redis.*` → `spring.data.redis.*`. Use
  `spring-boot-properties-migrator` to report renamed properties at startup.

### Migration path from 2.x
1. Upgrade to the latest 2.7 and fix deprecations.
2. Move to Java 17.
3. Upgrade to 3.x; run OpenRewrite (`UpgradeSpringBoot_3_0`) for `jakarta` imports and API changes.
4. Update third-party libraries to Jakarta-compatible versions.
5. Fix Security configuration, renamed properties (properties migrator), Hibernate 6 query
   differences.
6. Run the full test suite, including integration tests against real infrastructure.

## 3. Highlights from 3.1 – 3.5

- **3.1:** `@ServiceConnection` for Testcontainers; `spring-boot-docker-compose` starts
  `compose.yaml` services automatically; **SSL bundles** (`spring.ssl.bundle.*`) to configure
  TLS uniformly.
- **3.2:** `spring.threads.virtual.enabled=true`; `RestClient` (modern sync HTTP client);
  `JdbcClient`; CRaC support; nested JAR loader rewrite.
- **3.3:** Class Data Sharing support for faster startup; `/actuator/sbom`; Base64 resources.
- **3.4:** **structured JSON logging** (`logging.structured.format.console=ecs`); `@MockitoBean`
  and `@MockitoSpyBean` (Spring Framework 6.2) replacing deprecated `@MockBean`/`@SpyBean`;
  **graceful shutdown on by default**; AssertJ-based `MockMvcTester`; auto-configured HTTP client
  settings for `RestClient`/`RestTemplate`.
- **3.5:** final 3.x feature release; many polish items and deprecations preparing for 4.0.

## 4. Spring Boot 4.0 / Spring Framework 7

**Platform baseline:** Java 17+ (first-class Java 25 support), **Jakarta EE 11** (Servlet 6.1,
JPA 3.2, Bean Validation 3.1), Kotlin 2.2, Hibernate 7, Spring Security 7, Spring Data 2025.1.

**Modular codebase.** The single `spring-boot-autoconfigure` JAR was split into focused modules per
technology, each with its own starter and test starter (e.g. `spring-boot-starter-webmvc`,
`spring-boot-starter-webmvc-test`). Smaller, more precise classpaths; better for AOT/native.
Old starter names are deprecated, and "classic" starters ease migration.

**Jackson 3** is the default JSON library (package `tools.jackson`, immutable `JsonMapper`).
Jackson 2 support is deprecated.

**API versioning** built into Spring MVC and WebFlux:
```java
@GetMapping(path = "/orders/{id}", version = "2.0")
```
with header, query param, path segment or media type strategies configured via
`ApiVersionConfigurer` / properties.

**HTTP service clients:** `@ImportHttpServices` registers groups of `@HttpExchange` interfaces as
beans with per-group configuration (base URL, timeouts) from properties. No more hand-written
`HttpServiceProxyFactory` boilerplate.

**Core resilience:** `@Retryable`, `@ConcurrencyLimit` and `@EnableResilientMethods` in Spring
Framework itself, plus `RetryTemplate` in core (previously required Spring Retry).

**Null safety:** the Spring portfolio adopted **JSpecify** annotations (`@Nullable`, `@NullMarked`)
for better Kotlin interop and static analysis (NullAway).

**Testing:** `@MockBean`/`@SpyBean` removed (use `@MockitoBean`/`@MockitoSpyBean`); new
`RestTestClient`; JUnit 6 support.

**Observability:** an OpenTelemetry starter for metrics/traces/logs export via OTLP.

**Removed / changed:**
- **Undertow** support removed (no Servlet 6.1 support).
- `spring-boot-starter-aop` renamed to `spring-boot-starter-aspectj`.
- Long-deprecated APIs removed; `RestTemplate` heading toward deprecation in favor of `RestClient`.
- Security 7 removes the `.and()` DSL chaining; lambda DSL only.

### Migrating 3.x → 4.0
1. Upgrade to the latest 3.5 and fix all deprecation warnings.
2. Swap `@MockBean` → `@MockitoBean`; move off Undertow; replace Jackson 2-specific code
   (`ObjectMapper` customizers → `JsonMapper` builders), or temporarily use Jackson 2 compatibility.
3. Switch to the new starter names (or use classic starters as a stepping stone).
4. Upgrade Spring Cloud to the matching release train.
5. Use OpenRewrite recipes and the official migration guide.

## 5. Interview questions

1. **What are the major changes in Spring Boot 3?**
   Java 17 baseline, Jakarta EE (`javax` → `jakarta`), Spring Framework 6, Security 6 configuration
   changes, native image/AOT support, Micrometer Observation and Tracing, `ProblemDetail`, HTTP
   interfaces, auto-configuration registration moved to `AutoConfiguration.imports`.

2. **Why the `javax` → `jakarta` rename?**
   Oracle transferred Java EE to the Eclipse Foundation but kept the `javax` trademark, so Jakarta EE 9
   moved all APIs to the `jakarta.*` namespace.

3. **How would you migrate a Boot 2 app to Boot 3?**
   Upgrade to 2.7, move to Java 17, run OpenRewrite migration recipes, update Jakarta-compatible
   dependencies, rewrite security config, use the properties migrator, run thorough tests.

4. **What replaced `WebSecurityConfigurerAdapter`?**
   Declaring `SecurityFilterChain` (and `WebSecurityCustomizer`, `UserDetailsService`, etc.) beans.

5. **What replaced Spring Cloud Sleuth?**
   Micrometer Tracing with OpenTelemetry or Brave bridges.

6. **What is new in Spring Boot 4?**
   Spring Framework 7 and Jakarta EE 11, modularized auto-configuration and starters, Jackson 3,
   built-in API versioning, `@ImportHttpServices` for HTTP clients, `@Retryable`/`@ConcurrencyLimit`
   in core, JSpecify null safety, removal of Undertow and `@MockBean`.

7. **When were virtual threads supported in Boot?**
   Boot 3.2 with Java 21 via `spring.threads.virtual.enabled=true`.

8. **What replaced `@MockBean`?**
   `@MockitoBean` (and `@MockitoSpyBean` for `@SpyBean`) from Spring Framework 6.2; `@MockBean` was
   deprecated in Boot 3.4 and removed in 4.0.

9. **What is `@ServiceConnection`?**
   A Boot 3.1+ annotation that derives connection details (URL, credentials) from a Testcontainers
   container or Docker Compose service instead of hard-coding properties.
