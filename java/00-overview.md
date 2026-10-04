# 00 — Overview & Roadmap

The big picture before the deep dives. Read this once, then come back to it after Phase 3 to check
you can explain every line.

## 1. What is Java?

Java is a **statically typed, class-based, object-oriented** language that compiles to
**bytecode** and runs on the **Java Virtual Machine (JVM)**. Its key selling points, which you
should be able to list in an interview:

| Feature | What it means in practice |
|---|---|
| **Platform independent** | `javac` produces `.class` bytecode; any OS with a JVM runs it ("write once, run anywhere"). |
| **Object-oriented** | Everything lives in classes; supports encapsulation, inheritance, polymorphism, abstraction. Not *purely* OO because of primitives (`int`, `double`, …). |
| **Strongly & statically typed** | Types are checked at compile time; no implicit unsafe conversions. |
| **Automatic memory management** | Garbage collector frees unreachable objects. No `free`/`delete`. |
| **Robust** | No pointer arithmetic, array bounds checks, checked exceptions, strong typing. |
| **Secure** | Bytecode verifier, no direct memory access, class-loader isolation. |
| **Multithreaded** | Threads, `synchronized`, `java.util.concurrent`, virtual threads built in. |
| **High performance** | JIT compiler turns hot bytecode into optimized native code at runtime. |

**Coming from JS/TS:** Java looks like TypeScript with classes, but types are real at runtime
(except generics, see chapter 11), there's no `undefined`, every function lives in a class, and
there's true parallel multithreading instead of a single event loop.

## 2. JDK vs JRE vs JVM

```
┌──────────────────────────── JDK (Java Development Kit) ────────────────────────────┐
│  javac, jshell, jar, javadoc, jdb, jcmd, jfr, jlink ...  (development tools)       │
│  ┌──────────────────────── JRE (Java Runtime Environment) ──────────────────────┐  │
│  │  Class libraries (java.lang, java.util, java.io ...)                         │  │
│  │  ┌──────────────────────── JVM (Java Virtual Machine) ────────────────────┐  │  │
│  │  │  Class loader · Bytecode verifier · Interpreter · JIT · GC · Runtime   │  │  │
│  │  └────────────────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────────────┘
```

- **JVM**: the abstract machine that loads, verifies and executes bytecode. Platform-specific.
- **JRE**: JVM + standard libraries; enough to *run* Java. Since Java 11, Oracle no longer ships a
  separate JRE. You use the JDK, or build a trimmed runtime with `jlink`.
- **JDK**: JRE + compiler and tools; needed to *develop* Java.

## 3. How a Java program runs

```
Hello.java ──javac──▶ Hello.class (bytecode) ──java──▶ JVM
                                                       ├─ ClassLoader loads Hello.class
                                                       ├─ Verifier checks bytecode is safe
                                                       ├─ Interpreter executes bytecode
                                                       └─ JIT compiles hot methods to native code
```

## 4. The concept map (what the rest of the guide covers)

| Area | Key concepts | Chapter |
|---|---|---|
| Runtime | JVM architecture, class loaders, JIT, bytecode | 01 |
| Basics | 8 primitives, wrappers, autoboxing, casting, operators, pass-by-value | 02 |
| Flow | `if`, `switch` (classic + expression), loops, arrays | 03 |
| Strings | Immutability, string pool, `StringBuilder`, text blocks | 04 |
| OOP | Classes, constructors, `static`, access modifiers, 4 pillars | 05–06 |
| Abstraction | Interfaces, abstract classes, nested classes, enums | 07 |
| Object contract | `equals`/`hashCode`, `toString`, `Comparable`, records, immutability | 08 |
| Errors | Checked/unchecked, try-with-resources, custom exceptions | 09 |
| Data structures | `List`/`Set`/`Map`/`Queue`, `HashMap` internals, concurrent collections | 10 |
| Type safety | Generics, bounded types, wildcards, type erasure | 11 |
| Functional | Lambdas, method refs, functional interfaces, streams, `Optional` | 12 |
| Concurrency | Threads, locks, `volatile`, executors, `CompletableFuture`, virtual threads | 13 |
| Memory | Heap/stack/metaspace, GC algorithms, references, leaks | 14 |
| I/O | Byte vs char streams, NIO.2 `Files`/`Path`, serialization | 15 |
| Modern Java | `var`, records, sealed, pattern matching, text blocks, Java 21/25 features | 16 |
| Design | Singleton, Builder, Factory, Strategy, Observer, SOLID | 17 |

## 5. The ten questions you will almost certainly be asked

1. JDK vs JRE vs JVM?
2. Why is `String` immutable, and what is the string pool?
3. `==` vs `equals()`? What is the `equals`/`hashCode` contract?
4. How does `HashMap` work internally?
5. Checked vs unchecked exceptions?
6. Overloading vs overriding?
7. Interface vs abstract class (after Java 8)?
8. `ArrayList` vs `LinkedList`; `HashMap` vs `ConcurrentHashMap`?
9. `synchronized` vs `volatile`; how do you create a thread?
10. What's new in Java 8 (lambdas, streams, `Optional`) and in Java 17/21?

Every one of these is answered in detail in the chapters and collected in
[19 — Interview questions](./19-interview-questions.md).
