# 17 — Design Patterns & SOLID in Java

Core Java interviews usually ask about **Singleton** (with thread safety), **Factory**, **Builder**,
**Strategy**, **Observer**, and the **SOLID** principles. Know where the JDK itself uses each
pattern.

## 1. SOLID

| Principle | Meaning | Violation → fix |
|---|---|---|
| **S**ingle Responsibility | A class has one reason to change | `OrderService` that also formats emails and writes PDFs → split into `OrderService`, `EmailNotifier`, `InvoiceRenderer` |
| **O**pen/Closed | Open for extension, closed for modification | `if (type == CARD) … else if (type == UPI) …` → `PaymentMethod` interface with one class per type |
| **L**iskov Substitution | Subtypes must be usable wherever the parent is expected | `Square extends Rectangle` breaks `setWidth`/`setHeight` expectations → separate types |
| **I**nterface Segregation | Many small interfaces beat one fat one | `Worker { work(); eat(); }` forces robots to implement `eat` → `Workable`, `Eater` |
| **D**ependency Inversion | Depend on abstractions, not concretions | `new MySqlRepo()` inside a service → inject a `Repository` interface (what Spring DI does) |

## 2. Creational patterns

### Singleton: exactly one instance

```java
// 1. Eager: simple and thread-safe (instance created at class initialization)
public final class Config {
    private static final Config INSTANCE = new Config();
    private Config() {}
    public static Config getInstance() { return INSTANCE; }
}

// 2. Lazy, synchronized method: thread-safe but locks on every call
public final class LazyConfig {
    private static LazyConfig instance;
    private LazyConfig() {}
    public static synchronized LazyConfig getInstance() {
        if (instance == null) instance = new LazyConfig();
        return instance;
    }
}

// 3. Double-checked locking: lazy plus fast; 'volatile' is REQUIRED
public final class DclConfig {
    private static volatile DclConfig instance;
    private DclConfig() {}
    public static DclConfig getInstance() {
        DclConfig local = instance;                 // one volatile read on the fast path
        if (local == null) {
            synchronized (DclConfig.class) {
                local = instance;
                if (local == null) {
                    instance = local = new DclConfig();
                }
            }
        }
        return local;
    }
}

// 4. Bill Pugh / initialization-on-demand holder: lazy, thread-safe, no locking
public final class HolderConfig {
    private HolderConfig() {}
    private static class Holder {                   // loaded only when getInstance() is first called
        static final HolderConfig INSTANCE = new HolderConfig();
    }
    public static HolderConfig getInstance() { return Holder.INSTANCE; }
}

// 5. Enum: recommended by Effective Java; safe against reflection AND serialization
public enum Registry {
    INSTANCE;
    private final Map<String, String> data = new ConcurrentHashMap<>();
    public void put(String k, String v) { data.put(k, v); }
}
Registry.INSTANCE.put("a", "b");
```

**Why does DCL need `volatile`?** `instance = new DclConfig()` is three steps: allocate memory,
run the constructor, assign the reference. Without `volatile`, the JIT/CPU may reorder so the
reference is assigned *before* the constructor finishes, and another thread could see a non-null
but half-built object.

**Ways to break a singleton and the defenses:**

| Attack | Defense |
|---|---|
| Reflection (`setAccessible(true)` on the private constructor) | Throw from the constructor if the instance exists, or use an enum |
| Serialization (deserializing creates a new instance) | `readResolve()` returning `INSTANCE`, or an enum |
| Cloning | Don't implement `Cloneable`, or throw from `clone()` |
| Multiple class loaders | Load from a common parent loader |

JDK examples: `Runtime.getRuntime()`, `Desktop.getDesktop()`. In Spring, beans are singletons
**per container** by default (not JVM-wide singletons).

### Factory Method / Static factory

```java
public interface Notification { void send(String msg); }
class EmailNotification implements Notification { public void send(String m) { /* ... */ } }
class SmsNotification   implements Notification { public void send(String m) { /* ... */ } }
class PushNotification  implements Notification { public void send(String m) { /* ... */ } }

public final class NotificationFactory {
    private NotificationFactory() {}
    public static Notification create(Channel channel) {
        return switch (channel) {          // exhaustive over an enum
            case EMAIL -> new EmailNotification();
            case SMS   -> new SmsNotification();
            case PUSH  -> new PushNotification();
        };
    }
}
```
Callers depend on `Notification`, not on concrete classes. JDK examples: `List.of()`,
`Integer.valueOf()`, `Calendar.getInstance()`, `NumberFormat.getInstance()`, `Path.of()`.

**Static factory method vs constructor** (*Effective Java* item 1): factories have names
(`fromString`), can return cached instances (`Integer.valueOf`), can return subtypes, and can
choose the implementation (`EnumSet.of` returns `RegularEnumSet` or `JumboEnumSet`).

**Abstract Factory:** a factory of related factories (e.g. `UIFactory` → `WindowsButton` +
`WindowsCheckbox` vs `MacButton` + `MacCheckbox`). JDK: `DocumentBuilderFactory`.

### Builder: many optional parameters, readable construction, immutability

```java
public final class HttpRequestSpec {
    private final String url;            // required
    private final String method;
    private final Map<String, String> headers;
    private final Duration timeout;

    private HttpRequestSpec(Builder b) {
        this.url = b.url;
        this.method = b.method;
        this.headers = Map.copyOf(b.headers);
        this.timeout = b.timeout;
    }

    public static Builder builder(String url) { return new Builder(url); }

    public static final class Builder {
        private final String url;
        private String method = "GET";
        private final Map<String, String> headers = new HashMap<>();
        private Duration timeout = Duration.ofSeconds(30);

        private Builder(String url) { this.url = Objects.requireNonNull(url); }
        public Builder method(String m) { this.method = m; return this; }
        public Builder header(String k, String v) { headers.put(k, v); return this; }
        public Builder timeout(Duration t) { this.timeout = t; return this; }
        public HttpRequestSpec build() {
            if (timeout.isNegative()) throw new IllegalStateException("timeout < 0");
            return new HttpRequestSpec(this);
        }
    }
}

HttpRequestSpec req = HttpRequestSpec.builder("https://api.example.com")
    .method("POST").header("Auth", "token").timeout(Duration.ofSeconds(5)).build();
```
It avoids **telescoping constructors** (`new X(a)`, `new X(a, b)`, `new X(a, b, c)` …) and setter-
based half-initialized objects. JDK: `StringBuilder`, `HttpRequest.newBuilder()`,
`Stream.builder()`, `Thread.ofVirtual()`. Lombok: `@Builder`.

### Prototype
Create objects by copying an existing one (`clone()`/copy constructors). Useful when construction is
expensive.

## 3. Structural patterns

### Adapter: make an incompatible interface fit
```java
interface PaymentProcessor { void pay(BigDecimal amount); }

class LegacyBankApi { void makeTransfer(double amountInCents) { /* ... */ } }   // can't change it

class LegacyBankAdapter implements PaymentProcessor {
    private final LegacyBankApi api;
    LegacyBankAdapter(LegacyBankApi api) { this.api = api; }
    @Override public void pay(BigDecimal amount) {
        api.makeTransfer(amount.movePointRight(2).doubleValue());
    }
}
```
JDK: `Arrays.asList()` (array → List), `InputStreamReader` (bytes → chars).

### Decorator: add behavior by wrapping, without subclassing
```java
interface DataSource { String read(); }
class FileSource implements DataSource { public String read() { return "data"; } }

abstract class SourceDecorator implements DataSource {
    protected final DataSource inner;
    SourceDecorator(DataSource inner) { this.inner = inner; }
}
class Decrypting extends SourceDecorator {
    Decrypting(DataSource s) { super(s); }
    public String read() { return decrypt(inner.read()); }
}
class Decompressing extends SourceDecorator {
    Decompressing(DataSource s) { super(s); }
    public String read() { return decompress(inner.read()); }
}

DataSource src = new Decompressing(new Decrypting(new FileSource()));
```
JDK: `java.io` streams (`new BufferedReader(new InputStreamReader(...))`),
`Collections.unmodifiableList`, `Collections.synchronizedMap`.

### Proxy: a stand-in that controls access (lazy loading, security, logging, transactions)
```java
interface UserService { User find(long id); }

UserService proxy = (UserService) Proxy.newProxyInstance(
    UserService.class.getClassLoader(),
    new Class<?>[]{UserService.class},
    (p, method, args) -> {
        long t = System.nanoTime();
        try { return method.invoke(realService, args); }
        finally { System.out.println(method.getName() + " took " + (System.nanoTime() - t) + "ns"); }
    });
```
This is how Spring AOP (`@Transactional`, `@Cacheable`) and Hibernate lazy loading work: JDK
dynamic proxies for interfaces, or CGLIB/ByteBuddy subclasses for classes.

### Facade
A simple interface over a complex subsystem (`OrderFacade.placeOrder()` coordinating inventory,
payment, shipping). JDK: `javax.faces.context.FacesContext`; SLF4J is a logging facade.

### Composite
Treat individual objects and groups uniformly (a `File` and a `Directory` both implement
`FileSystemNode.size()`). JDK: AWT/Swing `Container` holding `Component`s.

## 4. Behavioral patterns

### Strategy: swap algorithms at runtime
```java
@FunctionalInterface
interface DiscountStrategy { BigDecimal apply(BigDecimal price); }

class Checkout {
    private DiscountStrategy discount = p -> p;                    // default: none
    void setDiscount(DiscountStrategy d) { this.discount = d; }
    BigDecimal total(BigDecimal price) { return discount.apply(price); }
}

checkout.setDiscount(p -> p.multiply(new BigDecimal("0.90")));     // 10% off
checkout.setDiscount(p -> p.subtract(BigDecimal.TEN).max(BigDecimal.ZERO));
```
With lambdas, many strategies are just functions. JDK: `Comparator` passed to `sort`,
`ThreadPoolExecutor`'s `RejectedExecutionHandler`.

### Observer: notify subscribers of changes
```java
class StockTicker {
    private final List<Consumer<BigDecimal>> listeners = new CopyOnWriteArrayList<>();
    public Runnable subscribe(Consumer<BigDecimal> l) {
        listeners.add(l);
        return () -> listeners.remove(l);                 // return an unsubscribe handle (avoid leaks)
    }
    public void update(BigDecimal price) { listeners.forEach(l -> l.accept(price)); }
}

Runnable unsubscribe = ticker.subscribe(p -> System.out.println("Price: " + p));
```
JDK: `PropertyChangeListener`, Swing event listeners, `java.util.concurrent.Flow` (reactive
streams). Spring: `ApplicationEventPublisher` + `@EventListener`.

### Template Method: fixed algorithm skeleton, customizable steps
```java
abstract class DataImporter {
    public final void importFile(Path p) {      // final: the order can't be changed
        var raw = read(p);
        var records = parse(raw);
        validate(records);
        save(records);
    }
    protected String read(Path p) throws IOException { return Files.readString(p); }
    protected abstract List<Row> parse(String raw);      // subclasses fill in
    protected void validate(List<Row> rows) {}           // hook with a default
    protected abstract void save(List<Row> rows);
}
```
JDK: `AbstractList` (implement `get` and `size`, get the rest), `HttpServlet.service()` calling
`doGet`/`doPost`. Spring: `JdbcTemplate`, `RestTemplate`.

### Iterator
Sequential access without exposing internals: `Iterable`/`Iterator`, which is what the for-each loop
uses.

### Chain of Responsibility
Pass a request along a chain of handlers until one handles it. Servlet `Filter` chains, Spring
Security filter chain, logging levels.

### Command
Encapsulate a request as an object (`Runnable`, `Callable` submitted to an executor; undo/redo
stacks).

## 5. JDK pattern lookup

| Pattern | JDK example |
|---|---|
| Singleton | `Runtime.getRuntime()` |
| Factory method | `List.of`, `Integer.valueOf`, `Calendar.getInstance` |
| Abstract factory | `DocumentBuilderFactory`, `TransformerFactory` |
| Builder | `StringBuilder`, `HttpRequest.newBuilder`, `Stream.builder` |
| Prototype | `Object.clone()` |
| Adapter | `Arrays.asList`, `InputStreamReader` |
| Decorator | `BufferedInputStream`, `Collections.unmodifiableList` |
| Proxy | `java.lang.reflect.Proxy` |
| Flyweight | `Integer` cache (−128..127), string pool |
| Iterator | `Iterator` |
| Observer | `PropertyChangeListener`, `Flow.Subscriber` |
| Strategy | `Comparator` |
| Template method | `AbstractList`, `InputStream.read(byte[])` |
| Command | `Runnable` |
| Chain of responsibility | `javax.servlet.Filter` |

## 6. Interview questions

1. **Explain SOLID with examples.** See the table in section 1.
2. **How do you write a thread-safe singleton?** Eager init, holder idiom, DCL with `volatile`, or
   an enum (best).
3. **Why `volatile` in double-checked locking?** It prevents reordering that could publish a
   partially constructed object.
4. **How can a singleton be broken?** Reflection, serialization, cloning, multiple class loaders.
   See the defenses table.
5. **Factory vs Abstract Factory?** One method creating one product family member vs a factory of
   families of related products.
6. **Why the Builder pattern?** Many optional params, immutability, readability; avoids telescoping
   constructors.
7. **Decorator vs Proxy?** Both wrap. Decorator adds behavior (often stacked); Proxy controls access
   (lazy, remote, security), usually one level.
8. **Strategy vs Template Method?** Composition (pass in an algorithm) vs inheritance (override
   steps of a fixed algorithm).
9. **Where does Java use the Observer pattern?** Event listeners, `Flow` API, Spring events.
10. **Which design patterns does Spring use?** Singleton (bean scope), Factory (`BeanFactory`),
    Proxy (AOP, `@Transactional`), Template (`JdbcTemplate`), Observer (events), Front Controller
    (`DispatcherServlet`), Dependency Injection.
11. **What is dependency injection, and how does it relate to the D in SOLID?** Supplying
    dependencies from outside so classes depend on abstractions; DI containers implement
    dependency inversion.
12. **What is the Flyweight pattern? Example?** Sharing immutable instances to save memory: the
    `Integer` cache, the string pool.
