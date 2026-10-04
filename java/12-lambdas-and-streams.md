# 12 — Lambdas, Functional Interfaces & Streams

Java 8's headline features and still the most-asked "modern Java" interview topic.

## 1. Lambda expressions

A lambda is a concise way to implement a **functional interface** (an interface with exactly one
abstract method).

```java
// Before Java 8: anonymous class
Comparator<String> byLength = new Comparator<String>() {
    @Override public int compare(String a, String b) { return Integer.compare(a.length(), b.length()); }
};

// Lambda
Comparator<String> byLength2 = (a, b) -> Integer.compare(a.length(), b.length());

// Even shorter with a factory + method reference
Comparator<String> byLength3 = Comparator.comparingInt(String::length);
```

Syntax forms:
```java
() -> 42                                  // no params
x -> x * 2                                // one param, parentheses optional
(x, y) -> x + y                           // several params
(int x, int y) -> x + y                   // explicit types
(var x, var y) -> x + y                   // Java 11, allows annotations on params
(x, y) -> {                               // block body needs return
    int sum = x + y;
    return sum;
}
(_, v) -> v                               // Java 22: unnamed parameter
```

**Variable capture:** lambdas can read local variables that are **effectively final** (never
reassigned). They can freely read and modify fields.
```java
int base = 10;
Function<Integer, Integer> add = x -> x + base;   // ✅
// base++;                                         // ❌ makes base not effectively final
```

## 2. Built-in functional interfaces (`java.util.function`)

| Interface | Method | Signature | Example |
|---|---|---|---|
| `Supplier<T>` | `get()` | `() → T` | `() -> new ArrayList<>()` |
| `Consumer<T>` | `accept(t)` | `T → void` | `s -> System.out.println(s)` |
| `BiConsumer<T,U>` | `accept(t,u)` | `(T,U) → void` | `map.forEach((k, v) -> ...)` |
| `Function<T,R>` | `apply(t)` | `T → R` | `String::length` |
| `BiFunction<T,U,R>` | `apply(t,u)` | `(T,U) → R` | `(a, b) -> a + b` |
| `UnaryOperator<T>` | `apply(t)` | `T → T` | `String::toUpperCase` |
| `BinaryOperator<T>` | `apply(t1,t2)` | `(T,T) → T` | `Integer::sum` |
| `Predicate<T>` | `test(t)` | `T → boolean` | `String::isEmpty` |
| `BiPredicate<T,U>` | `test(t,u)` | `(T,U) → boolean` | `String::equalsIgnoreCase` |

Primitive specializations avoid boxing: `IntPredicate`, `IntFunction<R>`, `ToIntFunction<T>`,
`IntUnaryOperator`, `IntBinaryOperator`, `IntSupplier`, `LongXxx`, `DoubleXxx`, …

Older functional interfaces: `Runnable` (`() → void`), `Callable<V>` (`() → V`, can throw),
`Comparator<T>`.

### Composition
```java
Predicate<String> notEmpty = Predicate.not(String::isEmpty);     // Java 11
Predicate<String> longWord = s -> s.length() > 5;
Predicate<String> both = notEmpty.and(longWord).or(s -> s.startsWith("#")).negate();

Function<Integer, Integer> times2 = x -> x * 2;
Function<Integer, Integer> plus3 = x -> x + 3;
times2.andThen(plus3).apply(5);    // (5*2)+3 = 13
times2.compose(plus3).apply(5);    // (5+3)*2 = 16
Function.identity();               // x -> x
```

### Custom functional interface
```java
@FunctionalInterface                      // compiler error if it doesn't have exactly one abstract method
public interface TriFunction<A, B, C, R> {
    R apply(A a, B b, C c);
    default TriFunction<A, B, C, R> log() { return (a, b, c) -> { System.out.println(a); return apply(a, b, c); }; }
}
```
Default and static methods, and methods from `Object` (`equals`, `toString`), don't count toward
the "one abstract method".

## 3. Method references

| Kind | Syntax | Lambda equivalent |
|---|---|---|
| Static method | `Integer::parseInt` | `s -> Integer.parseInt(s)` |
| Instance method of a particular object | `System.out::println` | `x -> System.out.println(x)` |
| Instance method of an arbitrary object of a type | `String::toUpperCase` | `s -> s.toUpperCase()` |
| Constructor | `ArrayList::new` | `() -> new ArrayList<>()` |
| Array constructor | `int[]::new` | `n -> new int[n]` |

## 4. `Optional<T>`

A container that may or may not hold a value. It's designed as a **return type** that makes
"maybe absent" explicit and avoids `null`.

```java
Optional<User> found = repo.findById(id);

// Creating
Optional.of(value);          // NPE if value is null
Optional.ofNullable(maybe);  // empty if null
Optional.empty();

// Consuming (prefer these over isPresent() + get())
String name = found.map(User::name).orElse("guest");
String name2 = found.map(User::name).orElseGet(() -> expensiveDefault());   // lazy
User u = found.orElseThrow(() -> new UserNotFoundException(id));
User u2 = found.orElseThrow();                    // Java 10: NoSuchElementException
found.ifPresent(user -> send(user));
found.ifPresentOrElse(this::send, () -> log.warn("missing"));   // Java 9
Optional<String> email = found.flatMap(User::email);  // when the mapper returns an Optional
found.filter(User::isActive);
found.or(() -> backupRepo.findById(id));          // Java 9
found.stream();                                   // Java 9: 0 or 1 element stream
```

**`orElse` vs `orElseGet`:** `orElse(x)` *always evaluates* `x`, even when the value is present.
`orElseGet(supplier)` only calls the supplier when empty. Use `orElseGet` for anything expensive or
side-effecting.

Best practices:
- ✅ Return type for "might not find it" lookups.
- ❌ Don't use it for fields, method parameters, or in collections (`List<Optional<X>>`).
- ❌ Never return `null` from a method declared to return `Optional`.
- ❌ Avoid `opt.get()` without checking; prefer `orElseThrow()`.
- Use `OptionalInt`/`OptionalDouble` for primitives.

## 5. Streams

A **stream** is a lazy pipeline of operations over a sequence of elements. It doesn't store data
and doesn't modify its source.

```
source ──▶ intermediate ops (lazy, return Stream) ──▶ terminal op (triggers execution)
list.stream()   .filter(...)  .map(...)  .sorted()      .collect(...) / .forEach / .reduce
```

```java
record Employee(String name, String dept, double salary, int age) {}

List<Employee> emps = List.of(
    new Employee("Ana", "ENG", 120_000, 30),
    new Employee("Raj", "ENG", 95_000, 25),
    new Employee("Li",  "HR",  70_000, 40),
    new Employee("Bo",  "SALES", 80_000, 35));

List<String> highEarners = emps.stream()
    .filter(e -> e.salary() > 90_000)        // intermediate
    .map(Employee::name)                     // intermediate
    .sorted()                                // intermediate (stateful)
    .toList();                               // terminal → [Ana, Raj]
```

### Creating streams
```java
list.stream(); list.parallelStream();
Stream.of(1, 2, 3);
Arrays.stream(array);
IntStream.range(0, 5);          // 0..4
IntStream.rangeClosed(1, 5);    // 1..5
Stream.iterate(1, x -> x * 2).limit(10);             // infinite → limit
Stream.iterate(1, x -> x < 1000, x -> x * 2);        // Java 9 with predicate
Stream.generate(Math::random).limit(5);
"hello".chars();                // IntStream
Files.lines(path);              // close it! use try-with-resources
map.entrySet().stream();
Stream.concat(s1, s2);
```

### Intermediate operations (lazy)

| Op | What it does |
|---|---|
| `filter(Predicate)` | Keep matching elements |
| `map(Function)` | Transform each element 1 → 1 |
| `flatMap(Function<T, Stream<R>>)` | Transform each element 1 → many, then flatten |
| `mapToInt/Long/Double` | To a primitive stream (`sum`, `average`, `max`) |
| `mapMulti` (Java 16) | Imperative 1 → many alternative to flatMap |
| `distinct()` | Remove duplicates (uses `equals`) |
| `sorted()` / `sorted(Comparator)` | Sort (stateful) |
| `peek(Consumer)` | Look at elements as they pass (debugging) |
| `limit(n)` / `skip(n)` | Truncate / skip |
| `takeWhile` / `dropWhile` (Java 9) | Take/drop while predicate holds |
| `boxed()` | `IntStream` → `Stream<Integer>` |
| `gather(Gatherer)` (Java 24) | Custom intermediate ops (windows, scans, …) |

### Terminal operations (eager, end the stream)

| Op | Returns |
|---|---|
| `collect(Collector)` | Collection/Map/String … |
| `toList()` (Java 16) | Unmodifiable `List` |
| `forEach(Consumer)` | `void` |
| `reduce(identity, BinaryOperator)` | Single value |
| `count()` | `long` |
| `min/max(Comparator)` | `Optional<T>` |
| `findFirst()` / `findAny()` | `Optional<T>` |
| `anyMatch/allMatch/noneMatch` | `boolean` (short-circuit) |
| `sum/average/summaryStatistics` | Primitive streams only |
| `toArray()` | Array |

### `map` vs `flatMap`
```java
List<List<Integer>> nested = List.of(List.of(1, 2), List.of(3), List.of());
nested.stream().map(List::size).toList();                 // [2, 1, 0]
nested.stream().flatMap(List::stream).toList();           // [1, 2, 3]

List<String> sentences = List.of("hello world", "java streams");
sentences.stream()
    .flatMap(s -> Arrays.stream(s.split(" ")))
    .toList();                                            // [hello, world, java, streams]
```

### `reduce`
```java
int sum = Stream.of(1, 2, 3, 4).reduce(0, Integer::sum);           // 10
Optional<Integer> product = Stream.of(1, 2, 3, 4).reduce((a, b) -> a * b);   // Optional[24]
double total = emps.stream().mapToDouble(Employee::salary).sum();  // better for numbers
String longest = words.stream().reduce("", (a, b) -> a.length() >= b.length() ? a : b);
```

## 6. Collectors

```java
import static java.util.stream.Collectors.*;

// To collections
emps.stream().map(Employee::name).collect(toList());
emps.stream().map(Employee::dept).collect(toSet());
emps.stream().map(Employee::name).collect(toCollection(TreeSet::new));

// To map (key collision → IllegalStateException unless you pass a merge function)
Map<String, Double> salaryByName = emps.stream()
    .collect(toMap(Employee::name, Employee::salary));
Map<String, Double> maxSalaryByDept = emps.stream()
    .collect(toMap(Employee::dept, Employee::salary, Math::max));        // merge function
Map<String, Employee> ordered = emps.stream()
    .collect(toMap(Employee::name, e -> e, (a, b) -> a, LinkedHashMap::new));

// Joining
String csv = emps.stream().map(Employee::name).collect(joining(", ", "[", "]"));   // "[Ana, Raj, Li, Bo]"

// groupingBy: the SQL GROUP BY of streams
Map<String, List<Employee>> byDept = emps.stream().collect(groupingBy(Employee::dept));
Map<String, Long> countByDept = emps.stream().collect(groupingBy(Employee::dept, counting()));
Map<String, Double> avgSalaryByDept = emps.stream()
    .collect(groupingBy(Employee::dept, averagingDouble(Employee::salary)));
Map<String, List<String>> namesByDept = emps.stream()
    .collect(groupingBy(Employee::dept, mapping(Employee::name, toList())));
Map<String, Optional<Employee>> topPaidByDept = emps.stream()
    .collect(groupingBy(Employee::dept, maxBy(Comparator.comparingDouble(Employee::salary))));
Map<String, Long> sorted = emps.stream()
    .collect(groupingBy(Employee::dept, TreeMap::new, counting()));    // sorted keys

// partitioningBy: always two keys, true and false
Map<Boolean, List<Employee>> seniors = emps.stream()
    .collect(partitioningBy(e -> e.age() >= 35));

// Statistics
DoubleSummaryStatistics stats = emps.stream().collect(summarizingDouble(Employee::salary));
stats.getAverage(); stats.getMax(); stats.getCount();

// Collect then transform
List<String> unmodifiable = emps.stream().map(Employee::name)
    .collect(collectingAndThen(toList(), Collections::unmodifiableList));

// teeing (Java 12): two collectors, then combine
double range = emps.stream().map(Employee::salary)
    .collect(teeing(maxBy(Double::compare), minBy(Double::compare),
                    (max, min) -> max.get() - min.get()));
```

## 7. Classic stream interview tasks

```java
// Second-highest salary
Optional<Double> second = emps.stream().map(Employee::salary).distinct()
    .sorted(Comparator.reverseOrder()).skip(1).findFirst();

// Highest-paid employee per department (name only)
Map<String, String> topNameByDept = emps.stream().collect(groupingBy(Employee::dept,
    collectingAndThen(maxBy(Comparator.comparingDouble(Employee::salary)),
                      opt -> opt.map(Employee::name).orElse(""))));

// Word frequency, sorted by count desc
Map<String, Long> freq = Arrays.stream(text.toLowerCase().split("\\W+"))
    .collect(groupingBy(Function.identity(), counting()));
freq.entrySet().stream()
    .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
    .limit(3)
    .forEach(e -> System.out.println(e.getKey() + " " + e.getValue()));

// Find duplicates in a list
Set<Integer> seen = new HashSet<>();
Set<Integer> dups = nums.stream().filter(n -> !seen.add(n)).collect(toSet());
// or: groupingBy(identity, counting()) then filter count > 1

// First non-repeated character
Character firstUnique = s.chars().mapToObj(c -> (char) c)
    .collect(groupingBy(Function.identity(), LinkedHashMap::new, counting()))
    .entrySet().stream().filter(e -> e.getValue() == 1)
    .map(Map.Entry::getKey).findFirst().orElse(null);

// Sum of squares of even numbers
int result = IntStream.rangeClosed(1, 10).filter(n -> n % 2 == 0).map(n -> n * n).sum();

// Join names of employees older than 30, upper-case, comma-separated
String names = emps.stream().filter(e -> e.age() > 30).map(e -> e.name().toUpperCase())
    .collect(joining(","));

// Flatten and dedupe
List<String> allSkills = people.stream().flatMap(p -> p.skills().stream()).distinct().sorted().toList();
```
More in [20 — Coding problems](./20-coding-problems.md).

## 8. How streams execute: laziness & short-circuiting

Nothing happens until a terminal operation runs. Elements then flow through the pipeline **one at a
time** (vertically), not stage by stage.

```java
Stream.of("a", "bb", "ccc", "dddd")
    .filter(s -> { System.out.println("filter " + s); return s.length() > 1; })
    .map(s -> { System.out.println("map " + s); return s.toUpperCase(); })
    .findFirst();
// filter a
// filter bb
// map bb          ← stops here: findFirst short-circuits; "ccc" is never touched
```

Stateful operations (`sorted`, `distinct`) must see all elements before passing any on.

**Streams can't be reused:**
```java
Stream<String> s = list.stream();
s.count();
s.count();   // IllegalStateException: stream has already been operated upon or closed
```

## 9. Parallel streams

```java
long count = bigList.parallelStream().filter(this::isPrime).count();
```
- Uses the common `ForkJoinPool` (size = CPU cores − 1).
- Helps for **large, CPU-bound, stateless** work on easily splittable sources (`ArrayList`, arrays,
  `IntStream.range`).
- Hurts for small data, I/O-bound work (it blocks the shared pool), `LinkedList`/`iterate` sources,
  order-dependent ops (`findFirst`, `limit` on ordered streams), or shared mutable state.

```java
// ❌ Race condition: ArrayList isn't thread-safe
List<Integer> out = new ArrayList<>();
IntStream.range(0, 1000).parallel().forEach(out::add);
// ✅
List<Integer> safe = IntStream.range(0, 1000).parallel().boxed().toList();
```

## 10. Stream gatherers (Java 24, final)

Gatherers are to intermediate operations what collectors are to terminal ones: you can write your
own `window`, `scan`, `fold` and so on.
```java
Stream.of(1, 2, 3, 4, 5).gather(Gatherers.windowFixed(2)).toList();    // [[1, 2], [3, 4], [5]]
Stream.of(1, 2, 3, 4).gather(Gatherers.windowSliding(2)).toList();     // [[1, 2], [2, 3], [3, 4]]
Stream.of(1, 2, 3, 4).gather(Gatherers.scan(() -> 0, Integer::sum)).toList();   // [1, 3, 6, 10]
```

## 11. Gotchas

1. Side effects inside `map`/`filter` (mutating external state) break with parallel streams and
   make code hard to follow. Use collectors.
2. `toMap` throws on duplicate keys; pass a merge function.
3. `Collectors.toList()` is mutable (in practice) while `Stream.toList()` is unmodifiable.
4. `peek` is for debugging; it may not run at all if the terminal op doesn't need the elements
   (e.g. `count()` on a sized stream since Java 9).
5. `forEach` on a parallel stream doesn't preserve order; use `forEachOrdered`.
6. Boxing overhead: use `mapToInt(...).sum()` rather than `map(...).reduce(0, Integer::sum)`.
7. Unclosed `Files.lines` streams leak file handles.
8. Checked exceptions don't propagate out of lambdas (chapter 09).

## 12. Interview questions

1. **What is a functional interface?** One abstract method; target type for lambdas and method
   references.
2. **Name the core functional interfaces.** `Supplier`, `Consumer`, `Function`, `Predicate`,
   `UnaryOperator`, `BinaryOperator`, and the `Bi*` variants.
3. **What is a lambda?** An anonymous function implementing a functional interface, compiled with
   `invokedynamic`.
4. **What is "effectively final"?** A variable that's never reassigned after initialization;
   required for lambda capture.
5. **What are the kinds of method reference?** Static, bound instance, unbound instance,
   constructor.
6. **Collection vs stream?** A collection stores data and is eager and reusable. A stream computes
   on demand, is lazy, single-use, and can be infinite.
7. **Intermediate vs terminal operations?** Lazy and return a stream vs trigger execution and return
   a result or side effect.
8. **`map` vs `flatMap`?** 1 → 1 vs 1 → many, flattened.
9. **`findFirst` vs `findAny`?** `findAny` may return any element (faster in parallel).
10. **What is a short-circuiting operation?** One that can finish without processing everything:
    `findFirst`, `anyMatch`, `limit`.
11. **Are streams lazy? Prove it.** Without a terminal op nothing executes; a `peek`/print in the
    pipeline shows element-by-element processing.
12. **When should you use parallel streams?** Large, CPU-bound, stateless operations on splittable
    sources; not for I/O or small data.
13. **`Optional.orElse` vs `orElseGet`?** Eager vs lazy evaluation of the default.
14. **Why `Optional` and where not to use it?** To make absence explicit in return types; not for
    fields, parameters or collections.
15. **`groupingBy` vs `partitioningBy`?** Any number of keys from a classifier vs exactly
    `true`/`false`.
16. **Can a stream be reused?** No; it throws `IllegalStateException`.
17. **What does `reduce` do?** Combines elements into one value with an identity and an associative
    accumulator.
18. **`Stream.toList()` vs `collect(Collectors.toList())`?** Unmodifiable (Java 16) vs currently a
    mutable `ArrayList`.
