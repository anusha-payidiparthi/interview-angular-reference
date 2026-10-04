# Core Java: Concepts, Examples & Interview Prep

A self-paced guide to **Core Java** for interviews: every major concept explained, runnable
examples, how things work under the hood, common gotchas, and interview questions at the end of
each chapter.

Written for **Java 25 (current LTS, Sept 2025)**, with notes on **Java 17 and 21** (the versions
most company codebases run) and on **Java 8** idioms you will still see in older code.

> **Version landscape (Oct 2026):** LTS releases are 8, 11, 17, 21 and **25**. Non-LTS releases ship
> every March and September (26 came out in March 2026). Interviewers mostly ask about Java 8
> features (lambdas, streams, `Optional`) plus "modern Java" (records, sealed types, pattern
> matching, virtual threads).

---

## How to use this guide

### Phase 1: Foundations (2–3 days)
Make sure you can explain these without notes. Most interviews start here.

| # | Chapter | What you'll be able to answer |
|---|---------|-------------------------------|
| 00 | [Overview & roadmap](./00-overview.md) | "What is Java? JDK vs JRE vs JVM?" |
| 01 | [JVM, JDK, JRE & program structure](./01-jvm-architecture.md) | Class loading, bytecode, JIT, `main` |
| 02 | [Data types, variables & operators](./02-data-types-and-operators.md) | Primitives vs wrappers, casting, pass-by-value |
| 03 | [Control flow & arrays](./03-control-flow-and-arrays.md) | Switch expressions, loops, labeled break, arrays |
| 04 | [Strings](./04-strings.md) | String pool, immutability, `StringBuilder` |

### Phase 2: Object-oriented Java (3–4 days)

| # | Chapter | What you'll be able to answer |
|---|---------|-------------------------------|
| 05 | [Classes & objects](./05-classes-and-objects.md) | Constructors, `static`, `this`, init order, access modifiers |
| 06 | [The four OOP pillars](./06-oop-pillars.md) | Overloading vs overriding, polymorphism, `super` |
| 07 | [Interfaces, abstract classes, nested classes & enums](./07-interfaces-abstract-nested-enums.md) | Interface vs abstract class, inner classes, enums |
| 08 | [`Object` class, equals/hashCode, immutability & records](./08-object-equals-hashcode-records.md) | equals/hashCode contract, `Comparable` vs `Comparator` |
| 09 | [Exception handling](./09-exceptions.md) | Checked vs unchecked, try-with-resources, `finally` |

### Phase 3: The core APIs (1 week)

| # | Chapter | What you'll be able to answer |
|---|---------|-------------------------------|
| 10 | [Collections framework](./10-collections.md) | `HashMap` internals, `ArrayList` vs `LinkedList`, fail-fast |
| 11 | [Generics](./11-generics.md) | Type erasure, wildcards, PECS |
| 12 | [Lambdas, functional interfaces & streams](./12-lambdas-and-streams.md) | `map` vs `flatMap`, collectors, `Optional` |
| 13 | [Multithreading & concurrency](./13-concurrency.md) | `synchronized`, `volatile`, executors, virtual threads |
| 14 | [Memory management & garbage collection](./14-memory-and-gc.md) | Heap vs stack, GC algorithms, memory leaks |
| 15 | [I/O, NIO & serialization](./15-io-nio-serialization.md) | `Files`, streams vs readers, `transient` |

### Phase 4: Modern Java & design (2–3 days)

| # | Chapter | What you'll be able to answer |
|---|---------|-------------------------------|
| 16 | [Modern Java features (8 → 25)](./16-modern-java-features.md) | Records, sealed classes, pattern matching, what's new |
| 17 | [Design patterns in Java](./17-design-patterns.md) | Singleton (thread-safe), Builder, Factory, Strategy |

### Phase 5: Consolidate (3–5 days)
- **[18: Cheat sheet](./18-cheatsheet.md)**: one-page lookup of syntax, complexities and APIs.
- **[19: Interview questions & answers](./19-interview-questions.md)**: 120 questions grouped by topic.
- **[20: Coding problems](./20-coding-problems.md)**: the Java-specific coding questions interviewers
  actually ask (string tricks, streams, LRU cache, producer–consumer, odd/even threads).

---

## Quick setup

```bash
# Install a JDK 25 build (Temurin, Oracle, Corretto, Zulu are all fine)
# Windows:  winget install EclipseAdoptium.Temurin.25.JDK
java -version
javac -version

# Single-file programs run directly, no separate javac step (Java 11+)
java Hello.java

# Interactive shell, great for trying snippets from this guide
jshell
```

Since Java 25, a single-file program can be as small as:
```java
void main() {
    IO.println("Hello, Java 25");
}
```
The classic form (still required knowledge for interviews, and what every codebase uses):
```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello, Java");
    }
}
```

## Official resources
- Language spec & JVM spec: <https://docs.oracle.com/javase/specs/>
- API docs (Java 25): <https://docs.oracle.com/en/java/javase/25/docs/api/>
- JEP index (what changed in each release): <https://openjdk.org/jeps/0>
- Dev.java tutorials: <https://dev.java/learn/>
