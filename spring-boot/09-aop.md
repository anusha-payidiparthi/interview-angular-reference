# 09 — Aspect-Oriented Programming (AOP)

## 1. What problem does AOP solve?

**Cross-cutting concerns** are behaviors needed in many places that aren't the business logic:
logging, transactions, security checks, caching, metrics, retries, auditing. Without AOP you
repeat them everywhere. AOP lets you define them once and apply them declaratively.

Spring itself uses AOP for `@Transactional`, `@Cacheable`, `@Async`, `@PreAuthorize`,
`@Retryable`, `@Validated` (method validation) and `@Observed`.

## 2. Terminology

| Term | Meaning | Example |
|---|---|---|
| **Aspect** | A module of cross-cutting logic | `LoggingAspect` |
| **Join point** | A point in execution where an aspect can apply. In Spring AOP: **method execution** only | `OrderService.place()` running |
| **Advice** | The action taken at a join point | "log duration" |
| **Pointcut** | Expression selecting join points | `execution(* com.example..service.*.*(..))` |
| **Target** | The original object being advised | `OrderService` instance |
| **Proxy** | The object Spring creates that wraps the target | `OrderService$$SpringCGLIB$$0` |
| **Weaving** | Linking aspects to targets | Spring: at runtime via proxies |

## 3. Advice types

| Advice | Runs | Can stop/modify? |
|---|---|---|
| `@Before` | Before the method | Only by throwing |
| `@AfterReturning` | After normal return | Can read the return value |
| `@AfterThrowing` | After an exception | Can read the exception (it still propagates) |
| `@After` | After either (like `finally`) | No |
| `@Around` | Wraps the call | **Yes**: decide whether to proceed, change args/result, catch exceptions |

## 4. Example: a timing + logging aspect

Add `spring-boot-starter-aop` (in Boot 4 the starter is `spring-boot-starter-aspectj`).

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogExecutionTime {}

@Aspect
@Component
@Slf4j
public class LoggingAspect {

    // reusable pointcut: all methods in any service package
    @Pointcut("execution(* com.example..service..*(..))")
    void serviceLayer() {}

    @Before("serviceLayer()")
    void logCall(JoinPoint jp) {
        log.debug("→ {}({})", jp.getSignature().toShortString(), Arrays.toString(jp.getArgs()));
    }

    @AfterReturning(pointcut = "serviceLayer()", returning = "result")
    void logResult(JoinPoint jp, Object result) {
        log.debug("← {} = {}", jp.getSignature().toShortString(), result);
    }

    @AfterThrowing(pointcut = "serviceLayer()", throwing = "ex")
    void logError(JoinPoint jp, Exception ex) {
        log.warn("✗ {} threw {}", jp.getSignature().toShortString(), ex.toString());
    }

    @Around("@annotation(LogExecutionTime)")          // methods annotated with @LogExecutionTime
    Object time(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();                       // call the real method
        } finally {
            log.info("{} took {} ms", pjp.getSignature().toShortString(),
                     (System.nanoTime() - start) / 1_000_000);
        }
    }
}

@Service
class ReportService {
    @LogExecutionTime
    public Report generate(Long id) { ... }
}
```

### A retry aspect (shows the power of `@Around`)
```java
@Around("@annotation(retry)")
Object retry(ProceedingJoinPoint pjp, Retry retry) throws Throwable {
    for (int attempt = 1; ; attempt++) {
        try { return pjp.proceed(); }
        catch (TransientException e) {
            if (attempt >= retry.maxAttempts()) throw e;
            Thread.sleep(retry.backoffMs() * attempt);
        }
    }
}
```
(Spring Framework 7 ships `@Retryable` and `@ConcurrencyLimit` in core; Boot 3 apps use Spring
Retry or Resilience4j.)

## 5. Pointcut expressions

| Expression | Matches |
|---|---|
| `execution(public * *(..))` | Any public method |
| `execution(* com.example.service.OrderService.*(..))` | Any method of `OrderService` |
| `execution(* com.example..*Service.find*(..))` | `find*` methods in `*Service` classes in any sub-package |
| `execution(* *(Long, ..))` | Methods whose first param is `Long` |
| `within(com.example.service..*)` | Any method in classes in that package tree |
| `@annotation(com.example.Audited)` | Methods annotated `@Audited` |
| `@within(org.springframework.stereotype.Service)` | Methods in classes annotated `@Service` |
| `bean(*Controller)` | Beans whose name ends in `Controller` (Spring-specific) |
| `args(id, ..)` | Bind the first argument to an advice parameter |

Combine with `&&`, `||`, `!`. `execution` syntax:
`execution(modifiers? returnType declaringType? methodName(params) throws?)`, where `*` is any
single element and `..` is any number of params or sub-packages.

Order multiple aspects with `@Order` (lower value = outer = runs first on the way in).

## 6. Proxies: JDK dynamic vs CGLIB

| | JDK dynamic proxy | CGLIB proxy |
|---|---|---|
| How | Implements the target's **interfaces** | Generates a **subclass** of the target |
| Requires | Target implements an interface | Class not `final`; methods not `final`/`private` |
| Inject by | Interface type only | Class or interface type |
| Spring Boot default | — | **CGLIB** (`spring.aop.proxy-target-class=true`) since Boot 2 |

Consequences of the proxy approach:
- **Self-invocation** (`this.method()`) bypasses the proxy → advice doesn't run.
- `final` classes/methods and `private` methods can't be advised.
- Only Spring beans are advised.
- With JDK proxies, injecting by implementation class fails (`BeanNotOfRequiredTypeException`).

## 7. Spring AOP vs AspectJ

| | Spring AOP | AspectJ |
|---|---|---|
| Weaving | Runtime proxies | Compile-time, post-compile, or load-time (bytecode) |
| Join points | Method execution on Spring beans | Methods, constructors, field access, static init… on any object |
| Self-invocation | Not intercepted | Intercepted |
| Setup | Zero (annotations only) | AspectJ compiler or agent |
| Performance | Proxy call overhead (small) | Near zero at runtime |

Spring AOP uses AspectJ's **annotations and pointcut language** but not its weaver.

## 8. Gotchas

1. Advice not running because of self-invocation or because the class isn't a bean.
2. `@Around` that forgets to `return pjp.proceed()` → method result lost (returns null).
3. Catching exceptions in `@Around` and not rethrowing hides failures (and breaks `@Transactional`
   rollback if the aspect is ordered inside the transaction).
4. Overly broad pointcuts (`execution(* *(..))`) advise Spring's own beans → slowdowns or cycles.
5. Logging arguments may leak sensitive data (passwords, tokens).

## 9. Interview questions

1. **What is AOP?**
   A paradigm for modularizing cross-cutting concerns (logging, transactions, security) into
   aspects applied declaratively instead of duplicating code.

2. **Explain aspect, advice, pointcut, join point.**
   Aspect: the module. Advice: the code to run. Pointcut: the expression choosing where. Join point:
   the specific execution point (a method call in Spring AOP).

3. **Types of advice?**
   `@Before`, `@AfterReturning`, `@AfterThrowing`, `@After` (finally), `@Around`.

4. **Which advice is the most powerful?**
   `@Around`: it controls whether the method runs, can change arguments and results, and handle
   exceptions.

5. **How does Spring implement AOP?**
   Runtime proxies created by a `BeanPostProcessor` after bean initialization: JDK dynamic proxies
   (interface-based) or CGLIB subclasses (Boot's default).

6. **JDK dynamic proxy vs CGLIB?**
   JDK proxies implement interfaces; CGLIB subclasses the class (can't proxy final classes/methods).

7. **Spring AOP vs AspectJ?**
   Spring AOP: proxy-based, method-execution join points on beans only, simple. AspectJ: bytecode
   weaving, all join point types, intercepts self-calls.

8. **Why doesn't my aspect run when calling a method from the same class?**
   The call doesn't go through the proxy. Move the method to another bean or use AspectJ weaving.

9. **Where does Spring itself use AOP?**
   `@Transactional`, `@Cacheable`, `@Async`, `@PreAuthorize`, `@Retryable`, `@Validated`,
   `@Observed`.

10. **How do you order multiple aspects?**
    `@Order` or implement `Ordered`; lower values have higher precedence (outermost).

11. **Write a pointcut for all public methods in the service package.**
    `execution(public * com.example.service..*(..))`.
