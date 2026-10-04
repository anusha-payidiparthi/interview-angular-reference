# 02 — Auto-configuration, Starters & Startup

## 1. `@SpringBootApplication`

```java
@SpringBootApplication
public class OrdersApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrdersApplication.class, args);
    }
}
```
It's a composed annotation of three:

| Annotation | Does |
|---|---|
| `@SpringBootConfiguration` | A specialized `@Configuration`: this class can declare `@Bean`s (and tests find it as the root config) |
| `@EnableAutoConfiguration` | Loads Boot's auto-configuration classes |
| `@ComponentScan` | Scans this package and sub-packages for components |

Customize: `@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)`,
`scanBasePackages = "com.example"`.

## 2. What happens in `SpringApplication.run()`

1. Create a `SpringApplication`, deduce the app type (servlet / reactive / none) from the classpath.
2. Load `ApplicationContextInitializer`s and `ApplicationListener`s.
3. Start listeners, publish `ApplicationStartingEvent`.
4. **Prepare the `Environment`**: load properties from all sources (args, env vars,
   `application.yml`, profiles…). `ApplicationEnvironmentPreparedEvent`.
5. Print the banner.
6. **Create the `ApplicationContext`** (type depends on step 1).
7. Prepare the context: register the main class as a bean source. `ApplicationContextInitializedEvent`,
   `ApplicationPreparedEvent`.
8. **Refresh the context** (the core of Spring):
   - parse `@Configuration` classes, run component scanning, process auto-configuration imports,
     evaluate `@Conditional`s → bean definitions
   - run `BeanFactoryPostProcessor`s
   - register `BeanPostProcessor`s
   - **create the embedded web server** (Tomcat starts here)
   - instantiate all non-lazy singletons
   - `ContextRefreshedEvent`
9. Web server starts accepting requests; `ApplicationStartedEvent`.
10. Call `CommandLineRunner` and `ApplicationRunner` beans.
11. `ApplicationReadyEvent`: the app is ready to serve traffic. (Failure anywhere →
    `ApplicationFailedEvent`, and `FailureAnalyzer`s print a friendly error.)

### Running code at startup
```java
@Component
class DataLoader implements CommandLineRunner {
    @Override public void run(String... args) { /* raw String[] args */ }
}

@Component
class Warmup implements ApplicationRunner {
    @Override public void run(ApplicationArguments args) {
        args.getOptionValues("mode");        // parsed --mode=fast
    }
}

@Component
class ReadyListener {
    @EventListener(ApplicationReadyEvent.class)
    void onReady() { /* app is fully started and serving */ }
}
```
Order multiple runners with `@Order`. Use `ApplicationReadyEvent` for things that must happen after
everything (including runners) is done.

## 3. How auto-configuration works

1. `@EnableAutoConfiguration` imports `AutoConfigurationImportSelector`.
2. It reads every
   `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` file on the
   classpath (one fully qualified class name per line). In Boot 2.x this list lived in
   `META-INF/spring.factories`; that mechanism was removed for auto-configs in 3.0.
3. Each listed class is an `@AutoConfiguration` class guarded by `@Conditional` annotations.
4. Conditions are evaluated; only matching configurations contribute beans.
5. Auto-configurations are processed **after** your own beans, so `@ConditionalOnMissingBean` lets
   your definitions win ("back-off").

A simplified real example:
```java
@AutoConfiguration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })   // JDBC on classpath?
@EnableConfigurationProperties(DataSourceProperties.class)               // binds spring.datasource.*
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(DataSource.class)                          // user didn't define one
    @ConditionalOnProperty(name = "spring.datasource.url")
    DataSource dataSource(DataSourceProperties props) {
        return props.initializeDataSourceBuilder().build();              // HikariCP by default
    }
}
```
So adding `spring-boot-starter-data-jpa` + a DB driver + `spring.datasource.url` gives you a
pooled `DataSource`, an `EntityManagerFactory`, a `JpaTransactionManager` and repositories, with
zero configuration classes.

### Common `@Conditional` annotations

| Annotation | Matches when |
|---|---|
| `@ConditionalOnClass` / `@ConditionalOnMissingClass` | A class is / isn't on the classpath |
| `@ConditionalOnBean` / `@ConditionalOnMissingBean` | A bean of the type exists / doesn't exist |
| `@ConditionalOnProperty(name, havingValue, matchIfMissing)` | A property has a value |
| `@ConditionalOnWebApplication` / `@ConditionalOnNotWebApplication` | Servlet/reactive web app or not |
| `@ConditionalOnResource` | A resource exists (e.g. `classpath:logback.xml`) |
| `@ConditionalOnExpression` | A SpEL expression is true |
| `@ConditionalOnJava` | JVM version range |
| `@ConditionalOnCloudPlatform` | Running on Kubernetes, Cloud Foundry, etc. |
| `@ConditionalOnThreading(Threading.VIRTUAL)` | Virtual threads are enabled |
| `@Conditional(MyCondition.class)` | Your own `Condition.matches(...)` returns true |

```java
class OnLinuxCondition implements Condition {
    @Override
    public boolean matches(ConditionContext ctx, AnnotatedTypeMetadata md) {
        return ctx.getEnvironment().getProperty("os.name", "").toLowerCase().contains("linux");
    }
}
```

### Seeing what got configured (and why)
- Run with `--debug` (or `debug=true`) → prints the **condition evaluation report**: positive
  matches, negative matches and the reason for each.
- Actuator `/actuator/conditions` shows the same at runtime; `/actuator/beans` lists all beans.

### Overriding / disabling auto-configuration
1. Define your own bean of the same type → `@ConditionalOnMissingBean` backs off.
2. Tune via properties (`spring.datasource.hikari.maximum-pool-size=20`).
3. Exclude: `@SpringBootApplication(exclude = SecurityAutoConfiguration.class)` or
   `spring.autoconfigure.exclude=org.springframework.boot...SecurityAutoConfiguration`.

## 4. Starters

A starter is a dependency descriptor (an almost-empty JAR with a POM) that pulls in a coherent set
of libraries plus the matching auto-configuration.

| Starter | Brings |
|---|---|
| `spring-boot-starter-web` (`-webmvc` in Boot 4) | Spring MVC, embedded Tomcat, Jackson, validation hooks |
| `spring-boot-starter-webflux` | WebFlux, Reactor Netty |
| `spring-boot-starter-data-jpa` | Hibernate, Spring Data JPA, HikariCP, `spring-jdbc` |
| `spring-boot-starter-validation` | Hibernate Validator (Jakarta Bean Validation) |
| `spring-boot-starter-security` | Spring Security |
| `spring-boot-starter-oauth2-resource-server` | JWT/opaque token validation |
| `spring-boot-starter-actuator` | Actuator + Micrometer |
| `spring-boot-starter-test` | JUnit Jupiter, AssertJ, Mockito, Spring Test, JSONassert |
| `spring-boot-starter-cache`, `-data-redis`, `-amqp`, `-mail`, `-batch`... | |

**Boot 4 change:** auto-configuration was split from one big `spring-boot-autoconfigure` JAR into
focused modules (e.g. `spring-boot-webmvc`, `spring-boot-jdbc`), and each technology has its own
starter and test starter (e.g. `spring-boot-starter-webmvc-test`). Old names like
`spring-boot-starter-web` still work but are deprecated, and "classic" starters exist to ease
migration.

### Dependency management (why you never write versions)
```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.0.0</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webmvc</artifactId>   <!-- no version -->
    </dependency>
</dependencies>
```
The parent imports `spring-boot-dependencies`, a **BOM** listing tested-compatible versions of
hundreds of libraries. Override one with a property: `<jackson-bom.version>…</jackson-bom.version>`.
If you can't use the parent (corporate parent POM), import the BOM in `<dependencyManagement>`
with `<scope>import</scope>`. Gradle uses the `org.springframework.boot` plugin plus
`io.spring.dependency-management` or a `platform(...)` BOM.

## 5. Embedded servers

Tomcat is the default for servlet apps; Jetty is an alternative (Undertow was dropped in Boot 4
because it didn't support Servlet 6.1). WebFlux uses Reactor Netty by default.
```xml
<!-- Switch Tomcat → Jetty -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webmvc</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```
```yaml
server:
  port: 8081               # 0 = random port
  servlet.context-path: /api
  tomcat.threads.max: 200
  shutdown: graceful       # default since Boot 3.4
```

## 6. Writing a custom starter

Useful for sharing config across many services (e.g. a company-wide audit-logging client).

```
acme-audit-spring-boot-starter/           (just a POM depending on the autoconfigure module)
acme-audit-spring-boot-autoconfigure/
  └─ src/main/java/com/acme/audit/
       AuditProperties.java
       AuditAutoConfiguration.java
  └─ src/main/resources/META-INF/spring/
       org.springframework.boot.autoconfigure.AutoConfiguration.imports
```
```java
@ConfigurationProperties("acme.audit")
public record AuditProperties(boolean enabled, String endpoint, Duration timeout) {}

@AutoConfiguration
@ConditionalOnClass(AuditClient.class)
@ConditionalOnProperty(prefix = "acme.audit", name = "enabled", havingValue = "true",
                       matchIfMissing = true)
@EnableConfigurationProperties(AuditProperties.class)
public class AuditAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean          // let applications override it
    AuditClient auditClient(AuditProperties props) {
        return new AuditClient(props.endpoint(), props.timeout());
    }
}
```
```
# AutoConfiguration.imports
com.acme.audit.AuditAutoConfiguration
```
Naming: official starters are `spring-boot-starter-*`; third-party ones should be
`*-spring-boot-starter`. Order relative to others with `@AutoConfiguration(after = ...)`.
Add `spring-boot-configuration-processor` to generate metadata for IDE autocompletion.

## 7. DevTools

`spring-boot-devtools` (dev only, excluded from the packaged JAR): automatic restart when classes
change (two class loaders: one for libraries, one for your code, so restart is fast), LiveReload,
dev-friendly property defaults (template caching off). Boot also supports **Docker Compose**
(`spring-boot-docker-compose`): starts the services in `compose.yaml` on app startup and wires
connection properties automatically.

## 8. Gotchas

1. Main class in a sub-package → components in sibling packages aren't scanned.
2. Adding a starter "just in case" activates its auto-config (e.g. adding JPA without a DB URL →
   startup failure "Failed to configure a DataSource").
3. Adding `spring-boot-starter-security` instantly locks every endpoint behind a generated password.
4. Custom auto-config placed in your component-scan path is picked up as regular config, so its
   `@ConditionalOnMissingBean` ordering guarantees break. Keep auto-configs out of scanned packages.
5. Overriding a library version in the BOM can break compatibility; prefer the Boot-managed version.
6. Using `spring.factories` for auto-configs in Boot 3+ → silently ignored.

## 9. Interview questions

1. **What does `@SpringBootApplication` do?**
   Combines `@SpringBootConfiguration` (a `@Configuration`), `@EnableAutoConfiguration` and
   `@ComponentScan` of the current package and sub-packages.

2. **How does auto-configuration work internally?**
   `@EnableAutoConfiguration` imports `AutoConfigurationImportSelector`, which loads class names from
   `META-INF/spring/...AutoConfiguration.imports` in every JAR. Each is a configuration class
   guarded by `@Conditional`s (class present, bean missing, property set). Matching ones register
   beans; they run after user config so user beans take precedence via `@ConditionalOnMissingBean`.

3. **How do you disable a specific auto-configuration?**
   `exclude` on `@SpringBootApplication`/`@EnableAutoConfiguration`, or the
   `spring.autoconfigure.exclude` property. Or define your own bean so it backs off.

4. **How do you find out which auto-configurations were applied?**
   Start with `--debug` to print the condition evaluation report, or use `/actuator/conditions`.

5. **What is a starter? Name some.**
   A curated dependency bundle plus auto-config for one capability: `-web`/`-webmvc`, `-data-jpa`,
   `-security`, `-actuator`, `-test`, `-validation`, `-cache`, `-amqp`, `-data-redis`.

6. **What is `spring-boot-starter-parent`?**
   A parent POM that imports the Boot dependency BOM (versions), sets the Java version, encoding,
   plugin configuration (resource filtering, `spring-boot-maven-plugin`) and sensible defaults.

7. **What does `SpringApplication.run()` do?**
   Prepares the environment, creates and refreshes the application context (bean definitions,
   post-processors, embedded server, singletons), runs `CommandLineRunner`/`ApplicationRunner`s,
   publishes lifecycle events ending in `ApplicationReadyEvent`.

8. **`CommandLineRunner` vs `ApplicationRunner`?**
   Both run after the context starts. `CommandLineRunner` gets raw `String[]`;
   `ApplicationRunner` gets parsed `ApplicationArguments` (option vs non-option args).

9. **How do you create a custom starter?**
   An autoconfigure module with an `@AutoConfiguration` class (with conditions and
   `@ConfigurationProperties`), listed in `AutoConfiguration.imports`, plus a starter POM that
   depends on it and the required libraries.

10. **What embedded servers are supported? How do you switch?**
    Tomcat (default), Jetty, Reactor Netty (WebFlux). Exclude `spring-boot-starter-tomcat` and add
    `spring-boot-starter-jetty`.

11. **Can you deploy a Spring Boot app as a WAR?**
    Yes: packaging `war`, extend `SpringBootServletInitializer`, mark the embedded server dependency
    `provided`. Rare today; executable JARs/containers are the norm.

12. **What is `@ConditionalOnMissingBean` and why is it important?**
    Registers a bean only if none of that type exists. It's what lets auto-configuration provide
    defaults that your own beans override.

13. **What is Spring Boot DevTools?**
    A dev-only dependency providing fast automatic restart, LiveReload and dev defaults. It's
    disabled automatically when running a packaged JAR.

14. **How can you change the default port?**
    `server.port=9090` in properties, `--server.port=9090` on the command line, or `SERVER_PORT`
    env var. `server.port=0` picks a random free port (handy in tests).

15. **What is a `FailureAnalyzer`?**
    A component that turns startup exceptions into human-readable messages, e.g. "Port 8080 was
    already in use. Action: identify and stop the process…".
