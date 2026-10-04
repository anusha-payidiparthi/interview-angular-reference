# 16 — Modern Java Features (Java 8 → 25)

What changed in each LTS, with examples of the features interviewers ask about. Features are listed
by the release where they became **final** (standard).

## 1. Release timeline at a glance

| Version | Year | LTS | Headline features |
|---|---|---|---|
| **8** | 2014 | ✅ | Lambdas, streams, `Optional`, default methods, `java.time`, `CompletableFuture` |
| 9 | 2017 | | Modules (JPMS), `jshell`, `List.of`/`Map.of`, private interface methods, `Optional`/stream additions |
| 10 | 2018 | | `var` |
| **11** | 2018 | ✅ | `HttpClient`, `String` methods (`isBlank`, `strip`, `lines`, `repeat`), `Files.readString`, single-file launch, `var` in lambdas |
| 12–13 | 2019 | | (Previews of switch expressions, text blocks) |
| 14 | 2020 | | Switch expressions, helpful NullPointerExceptions |
| 15 | 2020 | | Text blocks, ZGC production-ready |
| 16 | 2021 | | Records, pattern matching for `instanceof`, `Stream.toList()` |
| **17** | 2021 | ✅ | Sealed classes, strong encapsulation of JDK internals, `RandomGenerator` |
| 18 | 2022 | | UTF-8 by default, simple web server (`jwebserver`), `@snippet` in Javadoc |
| 19–20 | 2022–23 | | (Previews of virtual threads, record patterns) |
| **21** | 2023 | ✅ | **Virtual threads**, record patterns, pattern matching for `switch`, sequenced collections, generational ZGC |
| 22 | 2024 | | Unnamed variables & patterns (`_`), Foreign Function & Memory API, launching multi-file programs |
| 23 | 2024 | | Markdown doc comments, generational ZGC by default |
| 24 | 2025 | | Stream gatherers, virtual threads no longer pin on `synchronized`, class-file API, AOT class loading, quantum-resistant crypto (ML-KEM, ML-DSA) |
| **25** | 2025 | ✅ | Compact source files & instance `main`, flexible constructor bodies, module imports, scoped values, compact object headers, generational Shenandoah |
| 26 | 2026 | | HTTP/3 in `HttpClient`, more AOT/Leyden work, Applet API removed (plus new previews) |

**What companies run (2026):** Java 17 and 21 dominate, 25 adoption is growing, and plenty of legacy
apps are still on 8 or 11. Spring Boot 3.x requires Java 17+.

## 2. Java 8: the big one

- **Lambdas & functional interfaces** (chapter 12)
- **Streams API** (chapter 12)
- **`Optional`** (chapter 12)
- **Default and static interface methods** (chapter 07)
- **Method references**
- **`CompletableFuture`** (chapter 13)
- **`java.time`** (below)
- `ConcurrentHashMap` improvements, `HashMap` treeification, Metaspace replaces PermGen,
  `Base64`, `StringJoiner`, `Collectors`, `Map.merge`/`computeIfAbsent`

### `java.time` (replaces `Date`/`Calendar`)
`java.util.Date` is mutable, not thread-safe, has months starting at 0, and mixes date with time.
`java.time` is immutable and thread-safe.

```java
LocalDate today = LocalDate.now();                         // 2026-10-04 (no time, no zone)
LocalDate birthday = LocalDate.of(1995, Month.MARCH, 14);
LocalTime noon = LocalTime.of(12, 0);
LocalDateTime meeting = LocalDateTime.of(2026, 10, 5, 9, 30);
ZonedDateTime nyc = meeting.atZone(ZoneId.of("America/New_York"));
Instant now = Instant.now();                               // machine timestamp (UTC)

today.plusDays(10).minusMonths(1).withDayOfMonth(1);       // returns new objects
Period age = Period.between(birthday, today);              // years/months/days
Duration d = Duration.between(start, end);                 // hours/minutes/seconds/nanos
long days = ChronoUnit.DAYS.between(birthday, today);
today.isBefore(birthday); today.getDayOfWeek(); today.isLeapYear();

DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd-MM-yyyy");   // thread-safe
String s = today.format(fmt);
LocalDate parsed = LocalDate.parse("04-10-2026", fmt);
```

| Class | Represents |
|---|---|
| `LocalDate` / `LocalTime` / `LocalDateTime` | Date/time without a zone (birthdays, store opening hours) |
| `ZonedDateTime` / `OffsetDateTime` | Date-time in a zone/offset (scheduling across regions) |
| `Instant` | A point on the UTC timeline (timestamps, logs, DB) |
| `Duration` / `Period` | Time-based amount / date-based amount |

## 3. Java 9–11

```java
// Collection factories (immutable)
List<String> l = List.of("a", "b");
Set<Integer> s = Set.of(1, 2);
Map<String, Integer> m = Map.of("a", 1, "b", 2);

// var (10)
var users = new HashMap<String, List<User>>();
for (var entry : users.entrySet()) { }

// HttpClient (11): HTTP/1.1 and HTTP/2, sync and async (HTTP/3 support added in 26)
HttpClient client = HttpClient.newHttpClient();
HttpRequest req = HttpRequest.newBuilder(URI.create("https://api.github.com"))
    .header("Accept", "application/json").GET().build();
HttpResponse<String> res = client.send(req, HttpResponse.BodyHandlers.ofString());
client.sendAsync(req, HttpResponse.BodyHandlers.ofString())
      .thenApply(HttpResponse::body).thenAccept(System.out::println);

// String & Files (11)
"  ".isBlank(); " x ".strip(); "a\nb".lines(); "ab".repeat(3);
Files.readString(path); Files.writeString(path, "text");

// Optional / Stream additions (9–11)
opt.ifPresentOrElse(..., ...); opt.or(() -> other); opt.isEmpty();
stream.takeWhile(p); stream.dropWhile(p); Stream.ofNullable(x);
Predicate.not(String::isBlank);
```

## 4. Java 14–17: modern syntax

### Switch expressions (14)
See chapter 03.

### Helpful NullPointerExceptions (14)
```
Cannot invoke "String.length()" because "user.getAddress().city" is null
```

### Text blocks (15)
See chapter 04.

### Records (16)
See chapter 08.
```java
record Money(BigDecimal amount, Currency currency) {
    Money { Objects.requireNonNull(amount); }
    Money add(Money o) { return new Money(amount.add(o.amount), currency); }
}
```

### Pattern matching for `instanceof` (16)
```java
// Before
if (obj instanceof String) {
    String s = (String) obj;
    System.out.println(s.length());
}
// After
if (obj instanceof String s && !s.isEmpty()) {     // s in scope where the match is definitely true
    System.out.println(s.length());
}
if (!(obj instanceof String s)) return;
System.out.println(s.length());                    // s in scope after the negative early-return
```

### Sealed classes & interfaces (17)
Restrict **which classes may extend or implement** a type. Combined with records and pattern
matching, this gives you algebraic data types and exhaustive `switch`.

```java
public sealed interface Shape permits Circle, Square, Rectangle { }

public record Circle(double radius) implements Shape { }
public record Square(double side) implements Shape { }
public non-sealed class Rectangle implements Shape {      // reopens the hierarchy below it
    private final double width, height;
    public Rectangle(double width, double height) { this.width = width; this.height = height; }
    public double width() { return width; }
    public double height() { return height; }
}

// subclasses must be: final, sealed, or non-sealed (records are implicitly final)

static double area(Shape s) {
    return switch (s) {                                    // exhaustive: NO default needed
        case Circle c -> Math.PI * c.radius() * c.radius();
        case Square sq -> sq.side() * sq.side();
        case Rectangle r -> r.width() * r.height();
    };   // adding a new permitted subtype → compile errors at every non-exhaustive switch (a good thing)
}
```
Benefits: model closed domains (payment result = `Success | Declined | Error`), let the compiler
check exhaustiveness, and control your API's extension points.

## 5. Java 21: the big LTS after 17

### Virtual threads
See chapter 13, section 15.

### Record patterns (destructuring)
```java
record Point(int x, int y) {}
record Line(Point from, Point to) {}

if (obj instanceof Line(Point(var x1, var y1), Point(int x2, int y2))) {   // nested destructuring
    System.out.println(Math.hypot(x2 - x1, y2 - y1));
}
```

### Pattern matching for `switch`
```java
sealed interface Result<T> permits Ok, Err {}
record Ok<T>(T value) implements Result<T> {}
record Err<T>(String message, int code) implements Result<T> {}

static <T> String render(Result<T> r) {
    return switch (r) {
        case Ok<T>(var v)                      -> "OK: " + v;
        case Err<T>(var msg, var code) when code >= 500 -> "Server error: " + msg;
        case Err<T>(var msg, var code)         -> "Client error " + code + ": " + msg;
    };
}
```
This is **data-oriented programming**: data modeled with records and sealed types, behavior written
as pattern-matching functions over them (an alternative to the Visitor pattern).

### Sequenced collections
See chapter 10, section 10: `getFirst()`, `getLast()`, `reversed()`, `putFirst()`.

### Others in 21
Generational ZGC, `String.indexOf(ch, from, to)`, `StringBuilder.repeat`, `Character.isEmoji`,
`Math.clamp`, key encapsulation API.

## 6. Java 22–24

```java
// Unnamed variables & patterns (22): '_' for things you must declare but don't use
try { ... } catch (NumberFormatException _) { return 0; }
for (var _ : list) count++;
map.forEach((_, v) -> System.out.println(v));
if (obj instanceof Point(var x, _)) { }  // ignore y
```

- **Foreign Function & Memory API (22):** call native C libraries and manage off-heap memory safely,
  without JNI boilerplate (`Linker`, `MemorySegment`, `Arena`).
- **Launch multi-file source programs (22):** `java Main.java` compiles other source files it needs.
- **Markdown documentation comments (23):** `///` doc comments in Markdown.
- **Stream gatherers (24):** chapter 12, section 10.
- **Virtual threads without `synchronized` pinning (24, JEP 491).**
- **Class-File API (24):** a standard API to parse and generate bytecode (replacing ASM inside the
  JDK).
- **Ahead-of-time class loading & linking (24):** faster startup with a training-run cache
  (Project Leyden).
- **Security Manager permanently disabled (24).**

## 7. Java 25 (LTS, September 2025)

### Compact source files & instance main methods (JEP 512)
```java
// HelloWorld.java: no class, no static, no String[]
void main() {
    IO.println("Hello!");
}
```
The new `java.lang.IO` class has `println`, `print`, `readln`. Aimed at beginners and scripts.

### Flexible constructor bodies (JEP 513)
Code that doesn't use `this` may run before `super(...)` / `this(...)` (validation, computing
arguments). See chapter 05.

### Module import declarations (JEP 511)
```java
import module java.base;      // imports java.util.*, java.io.*, java.time.*, ... in one line
```

### Scoped values (JEP 506)
An immutable, bounded alternative to `ThreadLocal`, designed for virtual threads and structured
concurrency.
```java
static final ScopedValue<User> CURRENT_USER = ScopedValue.newInstance();

ScopedValue.where(CURRENT_USER, user).run(() -> handleRequest());   // bound only inside run()
// anywhere down the call stack:
User u = CURRENT_USER.get();
```
Advantages over `ThreadLocal`: immutable (no `set`), automatically unbound when the scope ends (no
leaks), cheap to inherit into child threads.

### Others in 25
- **Compact object headers (JEP 519):** object headers shrink from 12–16 bytes to 8 bytes
  (opt-in: `-XX:+UseCompactObjectHeaders`), saving memory for object-heavy apps.
- **Generational Shenandoah** (product feature).
- **AOT command-line ergonomics & method profiling** (faster warm-up).
- **Key derivation function API**, PEM encodings (preview).
- **Still in preview:** structured concurrency, primitive types in patterns, stable values
  (renamed "lazy constants" in 26), vector API (incubator).

## 8. Preview features: how they work
Preview features are complete but not final, and may change. They need flags:
```bash
javac --release 25 --enable-preview Main.java
java --enable-preview Main
```

## 9. "Modern Java" style checklist

- [ ] Records for DTOs and value objects
- [ ] Sealed types + switch pattern matching for closed hierarchies
- [ ] `var` where the type is obvious from the right-hand side
- [ ] Switch expressions instead of fall-through switch statements
- [ ] Text blocks for SQL/JSON/HTML
- [ ] `List.of`/`Map.of`/`Stream.toList()` for immutable data
- [ ] `java.time` instead of `Date`/`Calendar`
- [ ] `Optional` for "may be absent" return values
- [ ] Virtual threads for I/O-bound concurrency
- [ ] `HttpClient` instead of `HttpURLConnection`
- [ ] try-with-resources everywhere
- [ ] Pattern-matching `instanceof` instead of cast-after-check

## 10. Interview questions

1. **What are the main Java 8 features?** Lambdas, streams, functional interfaces, method
   references, `Optional`, default/static interface methods, `java.time`, `CompletableFuture`.
2. **Why was `java.time` introduced?** `Date`/`Calendar` were mutable, not thread-safe, and badly
   designed (0-based months, mixed concerns).
3. **`LocalDateTime` vs `ZonedDateTime` vs `Instant`?** No zone vs with a zone vs a UTC timeline
   point.
4. **What is `var`? Is Java dynamically typed now?** Local type inference; still statically typed.
5. **What is a record?** An immutable data carrier with generated constructor, accessors, `equals`,
   `hashCode`, `toString`.
6. **What are sealed classes and why use them?** They restrict permitted subtypes, which enables
   exhaustive pattern matching and controlled extension.
7. **`final` vs `sealed` vs `non-sealed`?** No subclasses vs only listed subclasses vs open again.
8. **What is pattern matching for `instanceof`/`switch`?** Test the type and bind a variable in one
   step; switch over types with guards (`when`) and record deconstruction.
9. **What are virtual threads?** Lightweight JVM-scheduled threads for massive blocking-I/O
   concurrency (Java 21).
10. **What are text blocks?** Multi-line string literals with `"""` (Java 15).
11. **What are sequenced collections?** Java 21 interfaces adding `getFirst`/`getLast`/`reversed`
    to ordered collections.
12. **What's new in Java 25?** Compact source files/instance main, flexible constructor bodies,
    module imports, scoped values, compact object headers.
13. **What are scoped values?** An immutable, scope-bound replacement for `ThreadLocal`.
14. **What does the `_` unnamed variable do?** Marks a variable or pattern component that's
    intentionally unused (Java 22).
15. **Which Java version would you pick for a new project in 2026, and why?** Java 25 (latest LTS:
    long support, virtual threads without pinning, modern syntax) unless a framework or platform
    constraint requires 21.
