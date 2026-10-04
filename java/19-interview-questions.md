# 19 — Core Java Interview Questions & Answers

120 questions grouped by topic, plus "predict the output" puzzles at the end. Answers are written
the way you'd say them in an interview: the core answer first, then a supporting detail. Each
section links to the chapter with the full explanation.

**How to practice:** cover the answer, say yours out loud, then compare. Mark the ones you
hesitated on and reread the linked chapter.

---

## A. Java basics & JVM ([01](./01-jvm-architecture.md), [02](./02-data-types-and-operators.md))

**1. What are the main features of Java?**
Platform independent (bytecode + JVM), object-oriented, statically and strongly typed, automatic
memory management (GC), robust (no pointers, checked exceptions), secure (bytecode verification),
multithreaded, high performance via JIT.

**2. JDK vs JRE vs JVM?**
The JVM executes bytecode. The JRE is the JVM plus the standard libraries, enough to run programs.
The JDK is the JRE plus development tools (`javac`, `jshell`, `jar`, `jcmd`). Since Java 11 there's
no separate JRE download; you use the JDK or build a custom runtime with `jlink`.

**3. Why is Java platform independent? Is the JVM platform independent?**
`javac` compiles to bytecode, which any JVM can run. The JVM itself is platform-*specific*; each
OS/CPU has its own implementation.

**4. Is Java compiled or interpreted?**
Both. Source is compiled to bytecode; the JVM interprets that bytecode and JIT-compiles hot code to
native machine code.

**5. What is the JIT compiler?**
Part of the JVM that compiles frequently executed bytecode into optimized native code at runtime,
using profiling information (inlining, escape analysis, loop optimizations). Tiered compilation
goes interpreter → C1 → C2.

**6. Explain class loading.**
Loading (find the bytes and create a `Class` object) → linking (verify, prepare static defaults,
resolve references) → initialization (run static initializers on first active use). Done by the
Bootstrap, Platform and Application class loaders using **parent delegation**.

**7. What is the parent-delegation model and why does it exist?**
A class loader asks its parent first and only loads the class itself if the parent can't. This
prevents core classes like `java.lang.String` from being replaced and ensures each class is loaded
once.

**8. What are the JVM memory areas?**
Shared: heap (objects) and method area/Metaspace (class metadata). Per thread: JVM stack (frames),
PC register, native method stack.

**9. Why is `main` `public static void`?**
`public` so the JVM can call it, `static` so no instance is needed, `void` because nothing is
returned (use `System.exit` for an exit code). Java 25 also allows an instance `void main()` in
compact source files.

**10. What are the primitive types and their sizes?**
`byte` 8, `short` 16, `int` 32, `long` 64, `float` 32, `double` 64, `char` 16 (unsigned UTF-16),
`boolean` (size is JVM-dependent).

**11. Is Java pass-by-value or pass-by-reference?**
Always pass-by-value. For objects, the value passed is a copy of the reference, so a method can
mutate the object but can't reassign the caller's variable.

**12. What is autoboxing, and what's the `Integer` cache?**
Automatic conversion between primitives and wrappers via `valueOf`/`xxxValue`. `Integer.valueOf`
caches −128 to 127, so `==` happens to work for small values and fails for larger ones. Always
compare wrappers with `equals`.

**13. What happens when you unbox `null`?**
`NullPointerException`, e.g. `int x = map.get("missing");`.

**14. Widening vs narrowing conversion?**
Widening (`int` → `long`) is implicit and safe. Narrowing (`double` → `int`) requires a cast and can
lose data (truncation, overflow).

**15. What does `var` do?**
Local variable type inference (Java 10). The compiler infers a static type from the initializer.
Only for locals, loop variables and lambda params; Java is still statically typed.

---

## B. Strings ([04](./04-strings.md))

**16. Why is `String` immutable?**
So literals can be safely shared in the string pool; for security (paths, URLs, credentials can't
change after validation); for thread safety; so the hash code can be cached (fast `HashMap` keys);
and because class loading relies on string names.

**17. What is the String Constant Pool?**
A special area of the heap where string literals are interned. Identical literals share one
instance.

**18. `==` vs `equals()` for strings?**
`==` compares references; `equals` compares characters. Always use `equals` (or
`"literal".equals(x)` to be null-safe).

**19. How many objects does `new String("hello")` create?**
Up to two: the `"hello"` literal in the pool (if not already there) and a new `String` object on the
heap.

**20. What does `intern()` do?**
Returns the pooled instance of the string, adding it to the pool if absent.

**21. `String` vs `StringBuilder` vs `StringBuffer`?**
`String` is immutable. `StringBuilder` is mutable and not synchronized (fastest, use in loops).
`StringBuffer` is mutable and synchronized (legacy, slower).

**22. Why shouldn't you concatenate strings with `+` in a loop?**
Each iteration creates a new `String` and copies all characters, which is O(n²). Use
`StringBuilder`.

**23. Why is `char[]` preferred over `String` for passwords?**
A `char[]` can be wiped after use; a `String` stays in memory (possibly pooled) until GC and can leak
into logs or heap dumps.

**24. Is `String` thread-safe?**
Yes, because it's immutable.

**25. `trim()` vs `strip()`; `isEmpty()` vs `isBlank()`?**
`strip` (Java 11) is Unicode-aware; `trim` only removes chars ≤ `'\u0020'`. `isEmpty` checks
length 0; `isBlank` also treats whitespace-only as blank.

---

## C. OOP ([05](./05-classes-and-objects.md), [06](./06-oop-pillars.md), [07](./07-interfaces-abstract-nested-enums.md))

**26. What are the four pillars of OOP?**
Encapsulation (hide state behind methods), inheritance (reuse via is-a), polymorphism (one
interface, many implementations), abstraction (expose what, hide how).

**27. Overloading vs overriding?**
Overloading: same name, different parameters, resolved at compile time. Overriding: same signature
in a subclass, resolved at runtime by the object's type.

**28. What are the rules for overriding?**
Same name and parameters; return type same or covariant; access same or wider; no new or broader
checked exceptions; can't override `final`, `static` or `private` methods.

**29. Can you override a static method?**
No. A static method with the same signature in a subclass **hides** the parent's; the call is
resolved by the reference type at compile time.

**30. Can you override a private method?**
No. Private methods aren't inherited; a same-named method in the subclass is a new method.

**31. Can you overload `main`?**
Yes, but the JVM only calls `main(String[])`.

**32. What is runtime polymorphism / dynamic dispatch?**
The JVM picks the overridden method to run based on the actual object type, not the reference type
(implemented with virtual method tables).

**33. Why doesn't Java support multiple inheritance of classes?**
The diamond problem: ambiguity about which inherited implementation and state to use. Java allows
multiple inheritance of *type* through interfaces, and forces explicit resolution for conflicting
default methods.

**34. Abstract class vs interface?**
An abstract class can have state, constructors and any methods, and a class extends only one. An
interface has no instance state, only constants and abstract/default/static/private methods, and a
class can implement many. Abstract class for shared code among related classes; interface for
capabilities and API contracts.

**35. Can an abstract class have a constructor? Can it be instantiated?**
It can have a constructor (called via `super()` from subclasses), but it can't be instantiated
directly.

**36. Why were default methods added to interfaces?**
To evolve interfaces (e.g. add `stream()`, `forEach` to `Collection` in Java 8) without breaking
existing implementations.

**37. What is a functional interface?**
An interface with exactly one abstract method, usable as a lambda target. Examples: `Runnable`,
`Comparator`, `Function`. `@FunctionalInterface` enforces it.

**38. What is a marker interface?**
An interface with no methods that tags a class for special treatment, e.g. `Serializable`,
`Cloneable`, `RandomAccess`.

**39. What is the difference between `this` and `super`?**
`this` refers to the current object (or calls another constructor with `this(...)`); `super`
refers to the parent's members (or calls a parent constructor with `super(...)`).

**40. What is the order of initialization when creating an object?**
Static blocks/fields (parent first, once per class), then for each object: parent instance
initializers and constructor, then child instance initializers and constructor.

**41. What does `static` mean?**
The member belongs to the class rather than to instances: one shared copy for fields, callable
without an object for methods. Static methods can't use `this` or access instance members directly.

**42. What does `final` mean on a variable, method, class?**
Variable: can't be reassigned (the object it references may still change). Method: can't be
overridden. Class: can't be extended.

**43. Static nested class vs inner class?**
An inner (non-static) class holds an implicit reference to an outer instance and needs one to be
created; a static nested class doesn't. Prefer static to avoid memory leaks.

**44. Composition vs inheritance?**
Inheritance is is-a and tightly couples you to the parent's implementation. Composition is has-a:
you delegate to contained objects. Prefer composition for flexibility and looser coupling.

**45. What are enums and what can they do?**
Type-safe sets of constants. They're full classes: fields, constructors (private), methods,
constant-specific bodies, can implement interfaces, work in `switch`, `EnumMap`, `EnumSet`. They
can't extend classes.

---

## D. `Object`, equals/hashCode, immutability ([08](./08-object-equals-hashcode-records.md))

**46. What methods does `Object` have?**
`equals`, `hashCode`, `toString`, `getClass`, `clone`, `finalize` (deprecated), `wait`, `notify`,
`notifyAll`.

**47. What is the contract between `equals` and `hashCode`?**
If two objects are equal by `equals`, they must have the same `hashCode`. Equal hash codes don't
imply equality. `equals` must be reflexive, symmetric, transitive, consistent and return false for
null.

**48. What happens if you override `equals` but not `hashCode`?**
Equal objects get different identity hash codes, so `HashMap`/`HashSet` look in different buckets:
lookups fail and duplicates appear.

**49. Can two different objects have the same hash code?**
Yes. That's a collision, handled by the hash table via `equals`.

**50. How do you write a good `equals`?**
Check `this == o`; check type with `instanceof` (or `getClass()` for strict equality); compare the
significant fields with `Objects.equals`/`==` for primitives; use the same fields in `hashCode`
(`Objects.hash`).

**51. `Comparable` vs `Comparator`?**
`Comparable.compareTo` defines the natural order inside the class (one order). `Comparator.compare`
defines external orders (many), composable with `comparing().thenComparing().reversed()`.

**52. Shallow copy vs deep copy?**
A shallow copy duplicates field values, so referenced objects are shared. A deep copy also copies
the referenced objects recursively.

**53. Why is `clone()` considered problematic?**
It relies on a marker interface (`Cloneable`) that changes the behavior of a protected method, is
shallow by default, bypasses constructors, and conflicts with `final` fields. Prefer copy
constructors or static factories.

**54. How do you create an immutable class?**
Make the class `final`, all fields `private final`, provide no setters, defensively copy mutable
constructor arguments and returned mutable fields, and don't let `this` escape during construction.

**55. What is a record?**
A Java 16 language feature for immutable data carriers: the compiler generates the canonical
constructor, accessors, `equals`, `hashCode` and `toString`. Records are final, can implement
interfaces, can't extend classes or add instance fields, and are only shallowly immutable.

---

## E. Exceptions ([09](./09-exceptions.md))

**56. Checked vs unchecked exceptions?**
Checked (`Exception` but not `RuntimeException`) must be caught or declared; they represent
recoverable external conditions (`IOException`). Unchecked (`RuntimeException`, `Error`) don't need
declaring; they usually indicate bugs (`NullPointerException`).

**57. Error vs Exception?**
`Error` is a serious JVM-level problem the app shouldn't try to handle (`OutOfMemoryError`,
`StackOverflowError`). `Exception` is a condition the app can handle.

**58. `throw` vs `throws`?**
`throw` is a statement that throws an exception object; `throws` in the signature declares the
checked exceptions a method may throw.

**59. `final` vs `finally` vs `finalize`?**
`final` is a modifier; `finally` is a block that always runs after `try`; `finalize` is the
deprecated method the GC used to call before reclaiming an object.

**60. Does `finally` always run?**
Yes, except when the JVM exits (`System.exit`), crashes or is killed, or the try never completes
(infinite loop, deadlock).

**61. What if both `try` and `finally` return a value?**
The `finally` return wins, and any exception from `try` is silently discarded. Never return from
`finally`.

**62. What is try-with-resources?**
A Java 7 construct that automatically closes `AutoCloseable` resources in reverse order; exceptions
from `close()` are added as suppressed exceptions on the primary one.

**63. Can you have `try` without `catch`?**
Yes: `try`-`finally`, or try-with-resources on its own.

**64. How do you create a custom exception, and checked or unchecked?**
Extend `Exception` (checked) or `RuntimeException` (unchecked) and provide constructors taking a
message and a cause. Modern codebases mostly prefer unchecked exceptions; use checked when callers
can realistically recover.

**65. What are exception-related best practices?**
Catch specific exceptions, never swallow them, preserve the cause when wrapping, don't use
exceptions for control flow, use try-with-resources, restore the interrupt flag on
`InterruptedException`, and log once at the boundary.

---

## F. Collections ([10](./10-collections.md))

**66. Describe the Collections hierarchy.**
`Iterable` → `Collection` → `List`, `Set`, `Queue`/`Deque`. `Map` is separate. Key implementations:
`ArrayList`, `LinkedList`, `HashSet`, `LinkedHashSet`, `TreeSet`, `ArrayDeque`, `PriorityQueue`,
`HashMap`, `LinkedHashMap`, `TreeMap`, `ConcurrentHashMap`.

**67. How does `HashMap` work internally?**
It's an array of buckets (default 16, always a power of two). `put` spreads the key's hash
(`h ^ h >>> 16`), computes the index `(n - 1) & hash`, and stores a node there. On collision it
walks the bucket comparing hash and `equals`: replace the value if found, otherwise append. A bucket
becomes a red-black tree at 8 entries (if capacity ≥ 64). When size exceeds capacity × 0.75 it
doubles and redistributes. `get` follows the same path.

**68. What changed in `HashMap` in Java 8?**
Treeification of long buckets (worst case O(log n) instead of O(n)), tail insertion instead of head
insertion, and a simplified hash function.

**69. What is the load factor, and what happens on resize?**
The fill ratio (default 0.75) that triggers resizing. The table doubles and each entry either stays
at index `i` or moves to `i + oldCapacity`.

**70. Why should `HashMap` keys be immutable?**
If a key's hash code changes after insertion, the entry sits in a bucket that lookups no longer
search, so it's effectively lost.

**71. `HashMap` vs `Hashtable`?**
`HashMap` is unsynchronized, allows one null key and null values, and is fast. `Hashtable` is
synchronized on every method, allows no nulls, and is legacy.

**72. `HashMap` vs `ConcurrentHashMap`?**
`ConcurrentHashMap` is thread-safe with lock-free reads and per-bucket locking/CAS for writes,
provides atomic compound methods (`computeIfAbsent`, `merge`), weakly consistent iterators, and
rejects nulls.

**73. Why doesn't `ConcurrentHashMap` allow null keys or values?**
In concurrent code `get(k) == null` would be ambiguous ("absent" or "mapped to null"), and you can't
safely follow up with `containsKey` because another thread may change the map in between.

**74. `ArrayList` vs `LinkedList`?**
`ArrayList`: dynamic array, O(1) random access, amortized O(1) append, O(n) middle inserts, compact
and cache-friendly. `LinkedList`: doubly linked, O(n) access, O(1) insert/remove at the ends, more
memory. `ArrayList` is the default; use `ArrayDeque` for queues.

**75. How does `ArrayList` grow?**
When full, it allocates a new array about 1.5× larger and copies the elements.

**76. `HashSet` vs `LinkedHashSet` vs `TreeSet`?**
No order with O(1) operations; insertion order with O(1); sorted with O(log n) (red-black tree,
uses `compareTo`/`Comparator`).

**77. How does `HashSet` guarantee uniqueness?**
It's backed by a `HashMap`: elements are stored as keys with a dummy value, so uniqueness comes from
`hashCode`/`equals`.

**78. Fail-fast vs fail-safe iterators?**
Fail-fast iterators (`ArrayList`, `HashMap`) throw `ConcurrentModificationException` if the
collection is structurally modified outside the iterator (detected via `modCount`). Fail-safe or
weakly consistent iterators (`CopyOnWriteArrayList`, `ConcurrentHashMap`) work on a snapshot or
tolerate concurrent changes.

**79. How do you remove elements while iterating?**
`Iterator.remove()`, or `Collection.removeIf(predicate)`. Never `list.remove()` inside a for-each.

**80. `Collection` vs `Collections`?**
`Collection` is the root interface; `Collections` is a utility class of static methods (`sort`,
`unmodifiableList`, `synchronizedMap`, …).

**81. `Arrays.asList` vs `List.of` vs `new ArrayList<>(…)`?**
`Arrays.asList`: fixed-size view backed by the array (set OK, add/remove throw, nulls OK).
`List.of`: immutable, no nulls. `new ArrayList<>(…)`: fully mutable copy.

**82. How would you implement an LRU cache?**
Extend `LinkedHashMap` with `accessOrder = true` and override `removeEldestEntry` to return
`size() > capacity`. Or by hand: a `HashMap` of key → node plus a doubly linked list for recency,
O(1) get/put.

---

## G. Generics ([11](./11-generics.md))

**83. Why generics?**
Compile-time type safety, no explicit casts, reusable type-parameterized code.

**84. What is type erasure?**
After type checking, the compiler removes generic type parameters (replacing them with their bounds,
usually `Object`) and inserts casts. Generic type information isn't available at runtime.

**85. What can't you do because of type erasure?**
`new T()`, `new T[]`, `instanceof List<String>`, overload methods differing only by type argument,
use primitives as type arguments, declare static fields of type `T`.

**86. `? extends T` vs `? super T`?**
`? extends T` accepts T or subtypes; you can read Ts but not add. `? super T` accepts T or
supertypes; you can add Ts but only read `Object`.

**87. What is PECS?**
Producer Extends, Consumer Super. Use `extends` for sources you read from and `super` for
destinations you write to, e.g. `Collections.copy(List<? super T> dest, List<? extends T> src)`.

**88. Is `List<String>` a subtype of `List<Object>`?**
No; generics are invariant. Otherwise you could add an `Integer` to a list of strings through the
`List<Object>` reference. Use `List<?>` or `List<? extends Object>`.

---

## H. Java 8 & functional programming ([12](./12-lambdas-and-streams.md))

**89. What are the main Java 8 features?**
Lambdas, functional interfaces, method references, Streams API, `Optional`, default/static interface
methods, `java.time`, `CompletableFuture`, `Collectors`, Metaspace.

**90. What is a lambda expression?**
An anonymous function that implements a functional interface: `(params) -> expression or block`.
Compiled using `invokedynamic`, with no separate class file per lambda.

**91. What is "effectively final"?**
A local variable that's never reassigned after initialization. Lambdas and inner classes can only
capture such variables, because they capture a copy.

**92. What kinds of method references exist?**
Static (`Integer::parseInt`), bound instance (`System.out::println`), unbound instance
(`String::toLowerCase`), constructor (`ArrayList::new`).

**93. Collection vs Stream?**
A collection stores elements, is eagerly built and can be iterated many times. A stream is a lazy
pipeline of computations over a source, single-use, and possibly infinite.

**94. Intermediate vs terminal operations?**
Intermediate operations (`filter`, `map`, `sorted`) are lazy and return a new stream. Terminal
operations (`collect`, `forEach`, `reduce`, `count`) trigger processing and produce a result.

**95. `map` vs `flatMap`?**
`map` transforms each element one-to-one. `flatMap` maps each element to a stream and flattens the
results into one stream (e.g. a list of lists into a list).

**96. How do you group elements in a stream?**
`collect(Collectors.groupingBy(classifier, downstream))`, e.g. `groupingBy(Employee::dept,
counting())`. `partitioningBy(predicate)` splits into true/false.

**97. `findFirst` vs `findAny`?**
`findFirst` respects encounter order; `findAny` can return any element and is faster for parallel
streams.

**98. When should you use parallel streams?**
For large, CPU-intensive, stateless work on easily splittable sources (arrays, `ArrayList`,
ranges). Not for small data, I/O, order-dependent work or shared mutable state. They use the common
`ForkJoinPool`.

**99. What is `Optional` and how should you use it?**
A container for a possibly absent value, meant for return types. Use `map`, `flatMap`, `orElse`,
`orElseGet`, `orElseThrow`, `ifPresent`. Don't use it for fields, parameters or collections, and
avoid `get()` without checking.

**100. `orElse` vs `orElseGet`?**
`orElse(value)` always evaluates its argument; `orElseGet(supplier)` only calls the supplier when
the `Optional` is empty.

---

## I. Concurrency ([13](./13-concurrency.md))

**101. How do you create a thread?**
Implement `Runnable` or `Callable` and submit it to an `ExecutorService` (preferred), pass a
`Runnable` to `new Thread(...)`, extend `Thread`, or use `Thread.ofVirtual().start(...)` (Java 21).

**102. `start()` vs `run()`?**
`start()` creates a new thread that calls `run()`. Calling `run()` directly executes it
synchronously on the current thread.

**103. What are the thread states?**
NEW, RUNNABLE, BLOCKED (waiting for a monitor), WAITING (`wait`/`join`/`park`), TIMED_WAITING
(`sleep`, timed `wait`), TERMINATED.

**104. `wait()` vs `sleep()`?**
`wait()` is an `Object` method, must be called holding the monitor, releases the lock, and is woken
by `notify` or a timeout. `sleep()` is a static `Thread` method that pauses without releasing any
locks.

**105. Why must `wait()` be called inside a loop?**
Spurious wakeups can happen, and another thread may have changed the condition between the
`notify` and this thread reacquiring the lock.

**106. What does `synchronized` do?**
It acquires the object's (or class's) intrinsic lock, giving mutual exclusion and memory visibility
(changes are visible to the next thread acquiring the same lock). It's reentrant.

**107. What does `volatile` do?**
It guarantees visibility (reads see the latest write) and prevents reordering around the access. It
does not make compound operations like `count++` atomic.

**108. What is a race condition? Give an example.**
When correctness depends on thread timing. `count++` from two threads loses updates because
read-modify-write isn't atomic. Fix with `synchronized`, `AtomicInteger` or `LongAdder`.

**109. What is a deadlock and how do you prevent it?**
Threads waiting forever for locks held by each other (circular wait). Prevent it with consistent
lock ordering, `tryLock` with timeouts, fewer and shorter-held locks, and higher-level concurrency
utilities. Detect it with thread dumps (`jstack`).

**110. `synchronized` vs `ReentrantLock`?**
`ReentrantLock` adds `tryLock` with timeout, interruptible locking, a fairness option and multiple
`Condition`s, but you must `unlock()` in `finally`. `synchronized` is simpler and releases
automatically.

**111. What is the Executor framework? Why use thread pools?**
An abstraction for running tasks (`ExecutorService`) that decouples submission from execution.
Pools reuse threads (avoiding creation cost), bound concurrency and queue work.

**112. `Future` vs `CompletableFuture`?**
`Future` only lets you block on `get()` or cancel. `CompletableFuture` supports non-blocking
callbacks and composition (`thenApply`, `thenCompose`, `thenCombine`, `allOf`), error handling
(`exceptionally`, `handle`) and manual completion.

**113. `CountDownLatch` vs `CyclicBarrier` vs `Semaphore`?**
Latch: one-shot, threads wait until a count reaches zero. Barrier: reusable, N threads wait for each
other. Semaphore: limits concurrent access to N permits.

**114. What are virtual threads?**
Lightweight threads (Java 21) managed by the JVM and mounted on a few carrier OS threads. Blocking
I/O unmounts them, so you can run millions with simple blocking code. Ideal for I/O-bound
thread-per-request servers; don't pool them; no benefit for CPU-bound work.

**115. What is `ThreadLocal` and its main risk?**
A variable with a separate value per thread (user context, non-thread-safe formatters). In thread
pools, values leak between tasks and cause memory leaks unless you call `remove()` in `finally`.
Java 25's `ScopedValue` is a safer alternative.

---

## J. Memory & GC ([14](./14-memory-and-gc.md))

**116. Stack vs heap?**
The stack is per thread and holds method frames (local primitives and references); it's freed
automatically on return and overflows with `StackOverflowError`. The heap is shared and holds all
objects; it's managed by the GC and runs out with `OutOfMemoryError`.

**117. How does garbage collection decide what to collect?**
Reachability: anything not reachable through references from GC roots (thread stacks, static
fields, JNI refs, active threads) is garbage, including unreachable cycles.

**118. Explain the generational heap.**
New objects go to Eden. Minor GCs copy survivors between survivor spaces and promote long-lived
objects to the old generation. Old-gen collections are less frequent and more expensive. This works
because most objects die young.

**119. Which garbage collectors are available? Which is the default?**
Serial, Parallel, G1 (default since Java 9), ZGC (sub-millisecond pauses), Shenandoah, and Epsilon
(no-op). CMS was removed in Java 14.

**120. Can Java have memory leaks? Give examples.**
Yes, when objects stay reachable unintentionally: ever-growing static maps or caches, unremoved
listeners, `ThreadLocal`s in pools, inner classes holding outer instances, unclosed resources,
mutable `HashMap` keys.

---

## Bonus: predict the output

**P1.**
```java
String a = "hello";
String b = "hel" + "lo";
String c = new String("hello");
String d = "hel";
String e = d + "lo";
System.out.println((a == b) + " " + (a == c) + " " + (a == e) + " " + a.equals(c));
```
> `true false false true`. `b` is a compile-time constant (pooled); `c` is a new object; `e` is
> computed at runtime.

**P2.**
```java
Integer x = 100, y = 100, p = 1000, q = 1000;
System.out.println((x == y) + " " + (p == q));
```
> `true false` (Integer cache −128..127).

**P3.**
```java
static int f() {
    try { throw new RuntimeException(); }
    catch (Exception e) { return 1; }
    finally { return 2; }
}
```
> `2`. The `finally` return overrides.

**P4.**
```java
class A { void show() { System.out.println("A"); } }
class B extends A { @Override void show() { System.out.println("B"); } }
A obj = new B();
obj.show();
```
> `B` (runtime polymorphism).

**P5.**
```java
class A { static void show() { System.out.println("A"); } }
class B extends A { static void show() { System.out.println("B"); } }
A obj = new B();
obj.show();
```
> `A`. Static methods are hidden, not overridden; resolved by reference type.

**P6.**
```java
void print(Object o) { System.out.println("Object"); }
void print(String s) { System.out.println("String"); }
print(null);
```
> `String` (most specific overload). With `print(String)` and `print(Integer)` it would be an
> ambiguity compile error.

**P7.**
```java
List<Integer> list = new ArrayList<>(List.of(1, 2, 3));
list.remove(1);
System.out.println(list);
```
> `[1, 3]`. `remove(int index)` is chosen over `remove(Object)`.

**P8.**
```java
int i = 0;
i = i++ + ++i;
System.out.println(i);
```
> `2`. `i++` yields 0 (i becomes 1), `++i` yields 2, and 0 + 2 = 2.

**P9.**
```java
System.out.println(1 + 2 + "3" + 4 + 5);
```
> `3345`. Left to right: 1 + 2 = 3, then string concatenation.

**P10.**
```java
Stream.of("a", "b", "c").filter(s -> { System.out.print(s); return true; });
```
> Prints nothing. There's no terminal operation, so the lazy pipeline never executes.

**P11.**
```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
for (String s : list) if (s.equals("a")) list.remove(s);
```
> Throws `ConcurrentModificationException`. (Removing the *second-to-last* element, `"b"`, happens
> not to throw: `hasNext()` returns false before the check, a known quirk.)

**P12.**
```java
char c = 'A';
c += 1;
System.out.println(c);
System.out.println('A' + 1);
```
> `B` then `66`. Compound assignment casts back to `char`; `'A' + 1` is an `int`.

**P13.**
```java
System.out.println(0.1 + 0.2 == 0.3);
System.out.println(Math.round(-2.5) + " " + Math.round(2.5));
```
> `false`, then `-2 3`. `Math.round` rounds half up (toward positive infinity).

**P14.**
```java
Set<StringBuilder> set = new HashSet<>();
set.add(new StringBuilder("x"));
set.add(new StringBuilder("x"));
System.out.println(set.size());
```
> `2`. `StringBuilder` doesn't override `equals`/`hashCode`.

**P15.**
```java
try {
    System.out.println("try");
    System.exit(0);
} finally {
    System.out.println("finally");
}
```
> Prints only `try`. `System.exit` stops the JVM, so `finally` doesn't run.
