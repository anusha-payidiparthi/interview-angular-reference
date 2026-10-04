# 01 — IoC, Dependency Injection & Beans

## 1. Inversion of Control (IoC) and Dependency Injection (DI)

**IoC:** instead of your code creating and managing its dependencies, a container does it and
"calls you". **DI** is the way Spring implements IoC: dependencies are *passed in* (injected)
rather than constructed inside the class.

```java
// Without DI: tightly coupled, impossible to swap or mock the gateway
class OrderService {
    private final PaymentGateway gateway = new StripeGateway("sk_live_...");
}

// With DI: depends on an abstraction, the container supplies the implementation
@Service
class OrderService {
    private final PaymentGateway gateway;

    OrderService(PaymentGateway gateway) {     // constructor injection
        this.gateway = gateway;
    }
}
```
Benefits: loose coupling, easy unit testing (pass a mock), swap implementations by config,
centralized lifecycle management.

## 2. The container: `BeanFactory` vs `ApplicationContext`

A **bean** is simply an object whose lifecycle is managed by the Spring container.

| | `BeanFactory` | `ApplicationContext` |
|---|---|---|
| What | Basic container: create beans, inject dependencies | Superset of `BeanFactory` |
| Singleton creation | Lazy (on first `getBean`) | **Eager** at startup (fail fast) |
| Extras | — | Events, i18n (`MessageSource`), resource loading, environment/profiles, automatic `BeanPostProcessor` registration, AOP integration |
| Used | Almost never directly | Always (Boot creates one for you) |

Boot picks the implementation: `AnnotationConfigServletWebServerApplicationContext` for servlet
web apps, a reactive variant for WebFlux, and a plain `AnnotationConfigApplicationContext` for
non-web apps.

## 3. Declaring beans

### Stereotype annotations (component scanning)
```java
@Component   // generic bean
@Service     // business logic (semantic only)
@Repository  // data access: also translates persistence exceptions to DataAccessException
@Controller  // MVC controller returning views
@RestController // = @Controller + @ResponseBody on every method
@Configuration  // class containing @Bean methods
```
`@Service`, `@Repository`, `@Controller` and `@Configuration` are all meta-annotated with
`@Component`, so component scanning finds them. `@SpringBootApplication` scans the package of the
main class **and its sub-packages**. Classes outside that tree are not found unless you add
`@ComponentScan("other.pkg")` or import them.

### `@Bean` methods (for third-party classes or custom construction)
```java
@Configuration
public class AppConfig {

    @Bean
    public ObjectMapper objectMapper() {          // you can't put @Component on a library class
        return JsonMapper.builder().findAndAddModules().build();
    }

    @Bean(initMethod = "start", destroyMethod = "stop")
    public CacheClient cacheClient(CacheProperties props) {   // params are injected
        return new CacheClient(props.host(), props.port());
    }
}
```

### `@Component` vs `@Bean`

| `@Component` | `@Bean` |
|---|---|
| On a **class** | On a **method** in a `@Configuration` class |
| Detected by component scanning | Explicitly declared |
| Only for classes you own | Works for any class, including third-party ones |
| Construction logic is the constructor | Arbitrary construction logic, conditional creation |

### `@Configuration` vs `@Component` with `@Bean` methods (lite mode)
`@Configuration` classes are CGLIB-proxied so that calling one `@Bean` method from another returns
the **same singleton**:
```java
@Configuration
class Config {
    @Bean A a() { return new A(b()); }   // b() goes through the proxy → returns the singleton B
    @Bean B b() { return new B(); }
}
```
In a `@Component` class (or `@Configuration(proxyBeanMethods = false)`), `b()` is a plain Java
call and creates a **new** `B` every time. Boot's own auto-configurations use
`proxyBeanMethods = false` for faster startup and inject dependencies as method parameters instead.

## 4. Types of dependency injection

```java
// 1. Constructor injection (RECOMMENDED)
@Service
public class OrderService {
    private final OrderRepository repo;
    private final PaymentGateway gateway;

    public OrderService(OrderRepository repo, PaymentGateway gateway) {  // @Autowired optional
        this.repo = repo;                                                 // with one constructor
        this.gateway = gateway;
    }
}

// 2. Setter injection (optional dependencies)
@Service
public class ReportService {
    private Notifier notifier;

    @Autowired(required = false)
    public void setNotifier(Notifier notifier) { this.notifier = notifier; }
}

// 3. Field injection (avoid in production code)
@Service
public class LegacyService {
    @Autowired private OrderRepository repo;
}
```

**Why constructor injection is preferred:**
1. Dependencies can be `final` → immutable, thread-safe after construction.
2. The object can never exist half-initialized.
3. Easy to unit test without Spring: `new OrderService(mockRepo, mockGateway)`.
4. Too many constructor params makes a "class does too much" smell obvious.
5. Circular dependencies fail fast at startup instead of being hidden.

With Lombok: `@RequiredArgsConstructor` generates the constructor for all `final` fields.

## 5. Resolving ambiguity: `@Primary`, `@Qualifier`, collections

```java
public interface PaymentGateway { void charge(Money amount); }

@Component("stripe") @Primary
class StripeGateway implements PaymentGateway { ... }

@Component("paypal")
class PaypalGateway implements PaymentGateway { ... }

@Service
class CheckoutService {
    CheckoutService(PaymentGateway gateway,                              // → Stripe (@Primary)
                    @Qualifier("paypal") PaymentGateway fallback,        // → PayPal
                    List<PaymentGateway> all,                            // → both
                    Map<String, PaymentGateway> byName) { ... }          // → {"stripe":…, "paypal":…}
}
```
Resolution order for a single injection point: by **type** → if several, a `@Qualifier` match →
a `@Primary` bean → the parameter **name** matching a bean name → otherwise
`NoUniqueBeanDefinitionException`. Missing bean → `NoSuchBeanDefinitionException` (use
`Optional<T>`, `ObjectProvider<T>` or `required = false` for optional dependencies).

The **Map-of-beans** trick is a clean way to implement the Strategy pattern:
```java
@Service
class PaymentRouter {
    private final Map<String, PaymentGateway> gateways;
    PaymentRouter(Map<String, PaymentGateway> gateways) { this.gateways = gateways; }

    void pay(String provider, Money amount) {
        gateways.get(provider).charge(amount);    // "stripe" or "paypal"
    }
}
```

`@Autowired` vs `@Resource` vs `@Inject`: `@Autowired` (Spring) injects by type; `@Inject`
(Jakarta, JSR-330) is equivalent; `@Resource` (Jakarta) injects **by name** first.

## 6. Bean scopes

| Scope | One instance per… | Notes |
|---|---|---|
| `singleton` (default) | Spring container | Must be stateless or thread-safe |
| `prototype` | Every injection / `getBean()` call | Spring does **not** call destroy callbacks |
| `request` | HTTP request | Web apps only |
| `session` | HTTP session | Web apps only |
| `application` | `ServletContext` | Like singleton but per web app |
| `websocket` | WebSocket session | |

```java
@Component
@Scope("prototype")
class ReportBuilder { private final List<String> lines = new ArrayList<>(); ... }

@Component
@RequestScope                     // shorthand for @Scope(value="request", proxyMode=TARGET_CLASS)
class RequestContext { private String correlationId; ... }
```

**Spring singleton vs GoF Singleton:** a GoF singleton is one instance per class loader, enforced
by a private constructor. A Spring singleton is one instance **per container per bean
definition**; you can define two beans of the same class.

### Injecting a prototype (or request-scoped) bean into a singleton
The singleton is created once, so it receives **one** prototype instance forever. Fixes:
```java
@Service
class ReportService {
    private final ObjectProvider<ReportBuilder> builders;     // 1. ObjectProvider (preferred)
    ReportService(ObjectProvider<ReportBuilder> builders) { this.builders = builders; }

    Report create() {
        ReportBuilder b = builders.getObject();               // a new instance each call
        ...
    }
}

// 2. Scoped proxy: inject a proxy that looks up the real instance per call
@Component
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
class ReportBuilder { ... }

// 3. @Lookup method injection (Spring overrides the method via CGLIB)
@Service
abstract class ReportService2 {
    @Lookup abstract ReportBuilder newBuilder();
}
```
Request/session scoped beans injected into singletons use a scoped proxy automatically when you
use `@RequestScope` / `@SessionScope`.

## 7. Bean lifecycle

```
1. Instantiate (constructor)                    ← constructor injection happens here
2. Populate properties (setter/field injection)
3. Aware callbacks (BeanNameAware, ApplicationContextAware, ...)
4. BeanPostProcessor.postProcessBeforeInitialization
5. @PostConstruct → InitializingBean.afterPropertiesSet() → @Bean(initMethod)
6. BeanPostProcessor.postProcessAfterInitialization   ← AOP proxies are created here
7. Bean ready for use
   ... application runs ...
8. @PreDestroy → DisposableBean.destroy() → @Bean(destroyMethod)   (on context close)
```

```java
@Component
class ConnectionPool {

    ConnectionPool(PoolProperties props) { ... }      // dependencies available

    @PostConstruct                                    // jakarta.annotation.PostConstruct
    void warmUp() { /* open initial connections; all injection is done */ }

    @PreDestroy
    void shutdown() { /* close connections */ }
}
```
Why `@PostConstruct` instead of the constructor? With field/setter injection the dependencies
aren't set yet in the constructor, and some work (e.g. calling a proxied method on yourself)
needs the fully initialized bean. Prototype beans never get destroy callbacks.

### Extension points
| Interface | Runs | Used for |
|---|---|---|
| `BeanFactoryPostProcessor` | After bean **definitions** load, before any bean is created | Modifying definitions (e.g. `PropertySourcesPlaceholderConfigurer` resolves `${...}`) |
| `BeanDefinitionRegistryPostProcessor` | Same, can register new definitions | `ConfigurationClassPostProcessor` processes `@Configuration` classes |
| `BeanPostProcessor` | Around initialization of **each bean** | Wrapping beans in proxies (`@Transactional`, `@Async`), processing `@Autowired` |

```java
@Component
class TimingPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String name) {
        if (bean.getClass().isAnnotationPresent(Timed.class)) {
            return ProxyFactory.getProxy(...);       // return a wrapper instead of the bean
        }
        return bean;
    }
}
```

## 8. Lazy initialization
```java
@Component @Lazy
class HeavyReportEngine { ... }        // created on first use instead of at startup
```
`spring.main.lazy-initialization=true` makes all beans lazy: faster startup (useful in dev), but
configuration errors surface on first request instead of at startup, and the first request is
slower. Not recommended for production.

## 9. Circular dependencies

```java
@Service class A { A(B b) {} }
@Service class B { B(A a) {} }
// → BeanCurrentlyInCreationException: "The dependencies of some of the beans form a cycle"
```
Since Boot 2.6, circular references are **prohibited by default** (even with field injection).
Fixes, in order of preference:
1. **Redesign:** extract the shared logic into a third bean `C` that both depend on, or use events.
2. Use `@Lazy` on one constructor parameter: Spring injects a proxy and resolves the real bean later.
3. Setter/field injection plus `spring.main.allow-circular-references=true` (last resort).

## 10. Conditional and profile-based beans
```java
@Bean
@Profile("dev")
DataSeeder dataSeeder() { return new DataSeeder(); }

@Bean
@ConditionalOnProperty(name = "feature.audit.enabled", havingValue = "true")
AuditService auditService() { return new AuditService(); }

@Bean
@ConditionalOnMissingBean            // only if the user hasn't defined their own
Clock clock() { return Clock.systemUTC(); }
```
Covered in detail in [02](./02-auto-configuration-and-starters.md) and [03](./03-configuration-and-profiles.md).

## 11. Gotchas

1. Field injection makes classes untestable without Spring and hides dependencies.
2. Mutable state (e.g. a `List` field) in a singleton bean → race conditions across requests.
3. Injecting a prototype into a singleton gives you one instance, not one per use.
4. A class outside the main application package isn't scanned → `NoSuchBeanDefinitionException`.
5. Calling a `@Bean` method directly inside a `proxyBeanMethods = false` config creates a new object.
6. `new MyService()` yourself → it isn't a bean: no injection, no `@Transactional`, no AOP.
7. `@PostConstruct` doing slow I/O slows startup; failing in it aborts startup.
8. Two beans of the same type without `@Primary`/`@Qualifier` → `NoUniqueBeanDefinitionException`.

## 12. Interview questions

1. **What is IoC?**
   A principle where the framework, not your code, controls object creation and wiring. Your
   classes declare what they need; the container builds the object graph.

2. **What is dependency injection? What types exist?**
   Supplying an object's dependencies from outside. Constructor, setter and field injection.
   Constructor injection is recommended (immutability, testability, fail-fast).

3. **What is a Spring bean?**
   An object instantiated, configured and managed by the Spring IoC container, defined via
   component scanning (`@Component` and friends), `@Bean` methods, or programmatic registration.

4. **`BeanFactory` vs `ApplicationContext`?**
   `ApplicationContext` extends `BeanFactory` and adds eager singleton creation, events, i18n,
   environment/profiles, resource loading and automatic post-processor registration. You always use
   `ApplicationContext`.

5. **`@Component` vs `@Bean`?**
   `@Component` goes on your own class and is found by scanning. `@Bean` goes on a factory method in
   a `@Configuration` class and is used for third-party classes or when construction needs logic.

6. **`@Component` vs `@Service` vs `@Repository` vs `@Controller`?**
   All are components. `@Service` is purely semantic. `@Repository` adds translation of
   persistence exceptions into Spring's `DataAccessException` hierarchy. `@Controller` marks an MVC
   handler; `@RestController` adds `@ResponseBody`.

7. **What are the bean scopes? What's the default?**
   Singleton (default), prototype, request, session, application, websocket.

8. **Are singleton beans thread-safe?**
   No. Spring doesn't synchronize anything; a singleton is shared by all request threads. Keep beans
   stateless (only `final` dependencies), or use thread-safe structures / `ThreadLocal` /
   request-scoped beans for per-request state.

9. **How do you inject a prototype bean into a singleton?**
   `ObjectProvider<T>.getObject()`, a scoped proxy (`proxyMode = TARGET_CLASS`), or `@Lookup`
   method injection. Plain injection gives one instance forever.

10. **Explain the bean lifecycle.**
    Instantiate → inject dependencies → `Aware` callbacks → `BeanPostProcessor` before-init →
    `@PostConstruct` / `afterPropertiesSet` / init-method → `BeanPostProcessor` after-init (proxies
    created) → in use → `@PreDestroy` / `destroy` / destroy-method on shutdown.

11. **`BeanPostProcessor` vs `BeanFactoryPostProcessor`?**
    `BeanFactoryPostProcessor` modifies bean *definitions* before any bean is created (e.g. resolving
    `${}` placeholders). `BeanPostProcessor` intercepts each bean *instance* around initialization
    (e.g. wrapping it in an AOP proxy).

12. **Two beans implement the same interface. How does Spring choose?**
    `@Qualifier("name")` at the injection point, `@Primary` on one bean, or matching parameter name.
    Or inject `List<T>` / `Map<String, T>` to get all of them.

13. **`@Autowired` vs `@Inject` vs `@Resource`?**
    `@Autowired` and `@Inject` inject by type (`@Autowired` has `required`). `@Resource` injects by
    name first, then type.

14. **Is `@Autowired` required on constructors?**
    Not if the class has a single constructor (Spring 4.3+). With multiple constructors, annotate the
    one Spring should use.

15. **What happens with circular dependencies?**
    With constructor injection Spring can't create either bean → `BeanCurrentlyInCreationException`.
    Boot 2.6+ forbids cycles by default. Fix by redesigning, `@Lazy` on one side, or (last resort)
    setter injection plus `spring.main.allow-circular-references=true`.

16. **What is `@Lazy`?**
    Delays creating a bean until first use. On an injection point, it injects a lazy-resolution proxy.

17. **What's the difference between `@Configuration` and `@Component` for `@Bean` methods?**
    `@Configuration` classes are CGLIB-proxied so inter-bean method calls return the container's
    singleton. In "lite" mode (`@Component` or `proxyBeanMethods = false`) such calls create new
    instances.

18. **How do you get a bean programmatically?**
    Inject `ApplicationContext` and call `getBean(Type.class)`. Prefer injection; use
    `ObjectProvider` for lazy or optional lookups.

19. **What is `@DependsOn`?**
    Forces another bean to be initialized first when there's no direct injection relationship (e.g.
    a DB migration bean before a cache warmer).

20. **Spring singleton vs Singleton design pattern?**
    Spring: one instance per container per bean definition, managed by Spring. GoF: one instance per
    class loader enforced by the class itself.
