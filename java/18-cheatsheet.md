# 18 — Core Java Cheat Sheet

One-page review for the night before an interview.

## Types & literals
```java
byte 8b  short 16b  int 32b  long 64b (10L)  float 32b (1.5f)  double 64b  char 16b ('A')  boolean
Integer.MAX_VALUE  Long.MIN_VALUE  1_000_000  0x1F  0b1010  'A' + 1 == 66
Integer cache: -128..127  → compare wrappers with equals()
Widening: byte→short→int→long→float→double (char→int)       Narrowing needs (cast)
0.1 + 0.2 != 0.3 → BigDecimal("0.1")     1/0 → ArithmeticException   1.0/0 → Infinity
Java is ALWAYS pass-by-value (references are copied)
```

## Strings
```java
"a" == "a" (true, pooled)   new String("a") == "a" (false)   use equals()
Immutable · final class · pooled · cached hashCode
StringBuilder (fast, not thread-safe)  StringBuffer (synchronized)
s.strip() isBlank() repeat(n) lines() chars() split(regex) join() formatted() substring(b, e)
```

## OOP quick facts
| Topic | Rule |
|---|---|
| Overloading | Same name, different params; compile time; return type alone isn't enough |
| Overriding | Same signature; runtime; covariant return; access same or wider; no broader checked exceptions |
| Not overridable | `static` (hidden), `private`, `final`, constructors |
| Fields | Not polymorphic (resolved by reference type) |
| Abstract class | State + constructors + any methods; single `extends` |
| Interface | Constants + abstract + `default`/`static`/`private` methods; multiple `implements` |
| Diamond (defaults) | Class wins → more specific interface → must override (`X.super.m()`) |
| Init order | Static (parent → child, once) → parent instance init + ctor → child instance init + ctor |
| Access | `private` < default (package) < `protected` (+ subclasses) < `public` |
| `final` | var: no reassign · method: no override · class: no subclass |
| `this()`/`super()` | Before Java 25 must be the first statement |

## equals / hashCode / compare
```java
equal objects ⇒ equal hashCodes (not vice versa)       override BOTH, same fields
equals: reflexive, symmetric, transitive, consistent, x.equals(null) == false
Objects.equals(a, b)  Objects.hash(f1, f2)
Comparable.compareTo (natural, inside class)   Comparator.compare (external)
Comparator.comparing(P::age).thenComparing(P::name).reversed()     never "a - b"
record Point(int x, int y) {}   → ctor, accessors x(), equals, hashCode, toString; final; shallow-immutable
```

## Exceptions
```
Throwable → Error (don't catch) | Exception → RuntimeException (unchecked) | others (checked)
try-with-resources closes in reverse order; close() errors → suppressed
finally always runs (except System.exit / JVM death); return in finally overrides & swallows
throw (statement) vs throws (signature)    multi-catch: catch (A | B e)
```

## Collections
| Interface | Main impls | Notes |
|---|---|---|
| `List` | `ArrayList` (O(1) get, grows 1.5×), `LinkedList` | `ArrayList` almost always |
| `Set` | `HashSet` (O(1)), `LinkedHashSet` (insertion order), `TreeSet` (sorted, O(log n)) | `HashSet` = `HashMap` keys |
| `Map` | `HashMap`, `LinkedHashMap` (LRU), `TreeMap`, `ConcurrentHashMap`, `EnumMap` | |
| `Queue`/`Deque` | `ArrayDeque` (stack + queue), `PriorityQueue` (heap) | No `Stack`, no `Vector` |
| Concurrent | `ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue` | Weakly consistent iterators |
| Immutable | `List.of`, `Map.of`, `List.copyOf`, `stream.toList()` | No nulls in `of`/`copyOf` |

```
HashMap: table of 16 buckets → index = (n-1) & (h ^ h>>>16) → list → tree at 8 (if cap ≥ 64)
         resize ×2 when size > 0.75 × capacity · 1 null key · not thread-safe · fail-fast
ConcurrentHashMap: CAS + per-bucket synchronized · lock-free reads · no nulls
offer/poll/peek (return special values) vs add/remove/element (throw)
Remove while iterating: iterator.remove() or removeIf()
```

## Generics
```java
<T extends Number & Comparable<T>>   bounded
List<? extends T> → read (producer)  List<? super T> → write (consumer)   PECS
Invariant: List<Integer> is NOT List<Number>
Type erasure: no new T(), no T[], no instanceof List<String>, no primitives
```

## Functional & streams
```java
Supplier<T> ()→T   Consumer<T> T→void   Function<T,R> T→R   Predicate<T> T→boolean
UnaryOperator<T> T→T   BinaryOperator<T> (T,T)→T   Runnable ()→void   Callable<V> ()→V
Method refs: Integer::parseInt  System.out::println  String::length  ArrayList::new

list.stream().filter(p).map(f).flatMap(g).distinct().sorted(c).limit(n).skip(n).peek(c)
terminal: collect toList() forEach reduce count min max findFirst anyMatch allMatch noneMatch
Collectors: toList toSet toMap(k, v, merge) joining groupingBy(f, downstream) partitioningBy
            counting summingInt averagingDouble mapping maxBy collectingAndThen teeing
Lazy · single-use · short-circuiting · parallelStream uses common ForkJoinPool
Optional: map flatMap filter orElse (eager) orElseGet (lazy) orElseThrow ifPresentOrElse
```

## Concurrency
```
Create: Runnable/Callable + ExecutorService · Thread.ofVirtual().start(r) · extends Thread
start() = new thread · run() = same thread
States: NEW RUNNABLE BLOCKED WAITING TIMED_WAITING TERMINATED
synchronized = mutual exclusion + visibility (reentrant)   volatile = visibility + ordering, NOT atomic
wait() releases lock (inside synchronized, in a while loop) · sleep() keeps lock
Atomics: AtomicInteger.incrementAndGet() (CAS) · LongAdder for hot counters
ReentrantLock: tryLock(timeout), lockInterruptibly, fairness, Conditions → unlock in finally
Executors: fixed, cached, single, scheduled, workStealing, newVirtualThreadPerTaskExecutor
CompletableFuture: supplyAsync thenApply thenCompose thenCombine allOf anyOf exceptionally handle
CountDownLatch (one-shot) · CyclicBarrier (reusable) · Semaphore (permits) · BlockingQueue (prod/cons)
Deadlock fix: global lock ordering · tryLock timeout · fewer nested locks
Virtual threads: I/O-bound, millions, don't pool, Semaphore to limit; ScopedValue over ThreadLocal
```

## Memory & GC
```
Stack: per thread, frames, locals/references → StackOverflowError
Heap: objects (Young: Eden + S0/S1 → Old) → OutOfMemoryError
Metaspace (native): class metadata (replaced PermGen in Java 8)
GC roots: stack locals, statics, active threads, JNI refs   reachability, not ref counting
Collectors: Serial · Parallel · G1 (default) · ZGC (sub-ms) · Shenandoah · Epsilon
References: strong > soft (cache) > weak (WeakHashMap) > phantom (Cleaner)
System.gc() = hint · finalize() deprecated · leaks = unintended reachability
```

## Big-O
| Op | ArrayList | LinkedList | HashMap/Set | TreeMap/Set | PriorityQueue | ArrayDeque |
|---|---|---|---|---|---|---|
| get/contains | O(1) idx / O(n) | O(n) | O(1) | O(log n) | O(n) | O(n) |
| add | O(1)* end | O(1) ends | O(1) | O(log n) | O(log n) | O(1)* ends |
| remove | O(n) | O(1) ends | O(1) | O(log n) | O(log n) poll | O(1) ends |

## Modern Java by version
```
8  lambdas streams Optional default methods java.time CompletableFuture
9  modules jshell List.of private interface methods     10 var
11 HttpClient String.isBlank/strip/repeat Files.readString     14 switch expressions, helpful NPE
15 text blocks    16 records, instanceof patterns, Stream.toList()    17 sealed classes
21 virtual threads, record patterns, switch patterns, sequenced collections
22 unnamed _ , FFM API    24 stream gatherers, no synchronized pinning
25 void main(), flexible constructors, import module, ScopedValue, compact headers
```

## Common one-liners
```java
Collections.reverse(list);                       new StringBuilder(s).reverse().toString();
Arrays.sort(arr); Arrays.toString(arr);          Arrays.stream(arr).max().getAsInt();
map.merge(word, 1, Integer::sum);                map.computeIfAbsent(k, x -> new ArrayList<>()).add(v);
list.stream().collect(groupingBy(identity(), counting()));
IntStream.rangeClosed(1, n).sum();               String.join(",", list);
Character.isDigit(c); Character.isLetter(c);     Integer.parseInt(s); String.valueOf(n);
new PriorityQueue<>(Comparator.reverseOrder());  // max-heap
Deque<Integer> stack = new ArrayDeque<>();       // push/pop/peek
```
