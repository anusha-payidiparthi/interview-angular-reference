# 03 — Configuration, Properties & Profiles

## 1. `application.properties` vs `application.yml`

```properties
# application.properties
server.port=8080
spring.datasource.url=jdbc:postgresql://localhost:5432/orders
spring.datasource.username=app
app.cors.allowed-origins[0]=https://example.com
app.cors.allowed-origins[1]=https://admin.example.com
```
```yaml
# application.yml — same thing, hierarchical
server:
  port: 8080
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/orders
    username: app
app:
  cors:
    allowed-origins:
      - https://example.com
      - https://admin.example.com
```
YAML is more readable for nested config and supports multiple documents in one file (`---`).
`.properties` is simpler, has no indentation pitfalls, and wins if both exist in the same
location. Pick one per project.

## 2. Externalized configuration and precedence

Spring Boot merges properties from many sources. **Higher in the list overrides lower** (simplified):

1. Test properties (`@TestPropertySource`, `@SpringBootTest(properties=...)`, `@DynamicPropertySource`)
2. **Command-line arguments** (`--server.port=9000`)
3. `SPRING_APPLICATION_JSON` (inline JSON)
4. Servlet config/context params
5. JNDI, Java system properties (`-Dserver.port=9000`)
6. **OS environment variables** (`SERVER_PORT=9000`)
7. `random.*` properties
8. **Config files**, in this order (later wins):
   - `application.yml` inside the JAR
   - `application-{profile}.yml` inside the JAR
   - `application.yml` outside the JAR (`./config/`, current dir)
   - `application-{profile}.yml` outside the JAR
9. `@PropertySource` on `@Configuration` classes
10. Defaults (`SpringApplication.setDefaultProperties`)

Rule of thumb: **command line > env vars > external files > packaged files**, and
**profile-specific > generic**. This lets you ship one artifact and override settings per
environment.

### Relaxed binding
`spring.datasource.url` can be supplied as `SPRING_DATASOURCE_URL` (env var: uppercase, dots → `_`,
remove dashes), `spring.datasource-url` or `springDatasourceUrl`. Use kebab-case in files.

### Importing more config
```yaml
spring:
  config:
    import:
      - optional:file:./secrets.yml
      - optional:configtree:/run/secrets/      # Kubernetes/Docker secrets mounted as files
      - optional:configserver:http://config:8888   # Spring Cloud Config
```

## 3. `@Value`

```java
@Service
class EmailService {
    @Value("${app.mail.from}")                    private String from;
    @Value("${app.mail.retries:3}")               private int retries;        // default 3
    @Value("${app.mail.enabled:true}")            private boolean enabled;
    @Value("#{'${app.mail.cc}'.split(',')}")      private List<String> cc;   // SpEL
    @Value("#{T(java.time.Duration).ofSeconds(${app.mail.timeout-seconds:5})}")
                                                  private Duration timeout;
}
```
`${...}` is a **property placeholder**; `#{...}` is a **SpEL expression**. A missing property
with no default fails startup (`Could not resolve placeholder`). `@Value` also works on
constructor parameters (preferred over fields).

## 4. `@ConfigurationProperties` (type-safe config)

```yaml
app:
  payment:
    base-url: https://api.stripe.com
    timeout: 5s               # Duration conversion: 5s, 500ms, 1m
    max-retries: 3
    supported-currencies: [USD, EUR]
    api-key: ${STRIPE_API_KEY}  # from an env var
```
```java
@ConfigurationProperties(prefix = "app.payment")
@Validated
public record PaymentProperties(
        @NotBlank String baseUrl,
        @NotNull Duration timeout,
        @Min(0) @DefaultValue("3") int maxRetries,
        List<Currency> supportedCurrencies,
        @NotBlank String apiKey) {}
```
Register it with **one** of:
```java
@SpringBootApplication
@ConfigurationPropertiesScan                     // scans for @ConfigurationProperties classes
public class App { ... }

// or
@EnableConfigurationProperties(PaymentProperties.class)
```
Inject like any bean:
```java
@Service
class PaymentClient {
    PaymentClient(PaymentProperties props) { ... props.timeout() ... }
}
```

### `@Value` vs `@ConfigurationProperties`

| | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| Granularity | One property | A group under a prefix |
| Type safety | Manual | Full binding: nested objects, lists, maps, `Duration`, `DataSize` |
| Relaxed binding | Limited | Full |
| Validation | No | `@Validated` + Bean Validation (fails startup on bad config) |
| SpEL | Yes | No |
| IDE autocompletion | No | Yes (with `spring-boot-configuration-processor`) |
| Best for | A single value, or SpEL | Any real configuration group |

## 5. Profiles

Profiles group configuration and beans per environment (`dev`, `test`, `prod`) or capability
(`kafka`, `local-mocks`).

```
application.yml           # shared defaults
application-dev.yml       # active when profile 'dev' is on
application-prod.yml
```
```yaml
# application.yml — multi-document alternative in one file
spring:
  application:
    name: orders
---
spring:
  config:
    activate:
      on-profile: dev
logging.level.com.example: DEBUG
---
spring:
  config:
    activate:
      on-profile: prod
server.tomcat.threads.max: 400
```

### Activating profiles
```bash
java -jar app.jar --spring.profiles.active=prod
SPRING_PROFILES_ACTIVE=prod,kafka java -jar app.jar
mvn spring-boot:run -Dspring-boot.run.profiles=dev
```
```yaml
spring.profiles.active: dev          # in application.yml (default for local dev; override in prod)
spring.profiles.group:
  production: [prod, monitoring, kafka]   # activating 'production' activates all three
```
In tests: `@ActiveProfiles("test")`. Without any active profile, the `default` profile is active.

### Profile-specific beans
```java
@Configuration
class StorageConfig {
    @Bean @Profile("prod")
    FileStorage s3Storage(S3Client s3) { return new S3FileStorage(s3); }

    @Bean @Profile("!prod")                  // NOT prod; also supports & and |
    FileStorage localStorage() { return new LocalFileStorage(Path.of("/tmp/files")); }
}
```

## 6. Reading the environment programmatically
```java
@Component
class Info {
    Info(Environment env) {
        env.getProperty("server.port", Integer.class, 8080);
        env.getActiveProfiles();
        env.acceptsProfiles(Profiles.of("prod"));
    }
}
```

## 7. Secrets management

Never commit secrets to `application.yml`. Options:
- Environment variables (`${DB_PASSWORD}`) injected by the platform.
- Kubernetes Secrets mounted as files + `spring.config.import=configtree:/etc/secrets/`.
- Vault (`spring-cloud-vault`), AWS Secrets Manager / Parameter Store, Azure Key Vault integrations.
- Spring Cloud Config Server with encrypted values (`{cipher}...`).

Actuator's `/env` and `/configprops` mask sensitive values; since Boot 3 they mask **all** values
by default (`management.endpoint.env.show-values=never`).

## 8. Refreshing config at runtime

Properties are bound at startup. To change config without a restart:
- Spring Cloud: `@RefreshScope` beans + `POST /actuator/refresh` (or Spring Cloud Bus to broadcast).
- Kubernetes: Spring Cloud Kubernetes reloads from ConfigMaps.
- Feature-flag systems (Unleash, LaunchDarkly, OpenFeature) for business toggles.

## 9. Gotchas

1. YAML indentation errors and tabs; values like `on`, `yes`, `no` may be parsed as booleans; quote
   strings with special characters (`"*"`, leading `0`).
2. `@Value` on a `static` field doesn't work (injected as null).
3. `@Value` in a class you created with `new` isn't resolved (not a bean).
4. Both `application.properties` and `.yml` present → both load, `.properties` wins on conflicts.
   Confusing; pick one.
5. `spring.profiles.active` inside a profile-specific file is invalid (Boot 2.4+).
6. `@ConfigurationProperties` class not registered (no scan/enable) → binding silently doesn't happen.
7. Leaking secrets in logs via `toString()` on a properties record. Override it or mask.

## 10. Interview questions

1. **`application.properties` vs `application.yml`?**
   Same content; YAML is hierarchical and supports multi-document files and lists more readably.
   Properties wins if both define a key in the same location.

2. **What is the order of property precedence?**
   Test properties > command-line args > `SPRING_APPLICATION_JSON` > system properties > env vars >
   external profile-specific files > external files > packaged profile-specific files > packaged
   files > `@PropertySource` > defaults.

3. **`@Value` vs `@ConfigurationProperties`?**
   `@Value` injects single values and supports SpEL. `@ConfigurationProperties` binds a whole prefix
   into a typed object with relaxed binding, validation and IDE metadata. Prefer
   `@ConfigurationProperties` for anything beyond one value.

4. **How do you give a default value to `@Value`?**
   `@Value("${key:default}")`. Empty default: `${key:}`.

5. **What are profiles? How do you activate one?**
   Named sets of config and beans. Activate with `spring.profiles.active` (property, `--` arg,
   `SPRING_PROFILES_ACTIVE` env var), `@ActiveProfiles` in tests, or programmatically.

6. **How do you load a bean only in a specific environment?**
   `@Profile("prod")` / `@Profile("!prod")`, or `@ConditionalOnProperty` for feature toggles.

7. **How do you validate configuration at startup?**
   `@Validated` on the `@ConfigurationProperties` class plus constraints (`@NotBlank`, `@Min`). Bad
   config fails startup with a clear message.

8. **How do you manage secrets?**
   Env vars or mounted secret files (`configtree`), a secrets manager (Vault, AWS/Azure), or Config
   Server encryption. Never in source control; mask in logs and Actuator.

9. **How does relaxed binding work?**
   Boot maps kebab-case, camelCase, underscore and upper-case env var forms to the same property.
   `my.service-url` ↔ `MY_SERVICEURL`/`MY_SERVICE_URL`.

10. **How do you change config without restarting?**
    Spring Cloud `@RefreshScope` + `/actuator/refresh` (or Cloud Bus), Kubernetes ConfigMap reload,
    or a feature-flag service.

11. **What is a profile group?**
    `spring.profiles.group.<name>` maps one profile name to several, so activating `production`
    activates `prod`, `monitoring`, etc.

12. **What's `@PropertySource`?**
    Adds a `.properties` file to the environment from a `@Configuration` class. Lower precedence
    than `application.*`; doesn't support YAML out of the box.
