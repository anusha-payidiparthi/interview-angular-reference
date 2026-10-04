# 01 — JVM, JDK, JRE & Program Structure

## 1. Anatomy of a Java program

```java
package com.example.app;               // namespace; must match folder com/example/app/

import java.util.List;                 // bring a type into scope (java.lang.* is imported automatically)

public class Greeter {                 // a public class must live in Greeter.java

    private final String greeting;     // field (instance state)

    public Greeter(String greeting) {  // constructor
        this.greeting = greeting;
    }

    public String greet(String name) { // instance method
        return greeting + ", " + name;
    }

    public static void main(String[] args) {   // entry point
        Greeter g = new Greeter("Hello");
        for (String name : List.of("Ana", "Raj")) {
            System.out.println(g.greet(name));
        }
    }
}
```

### Why is `main` declared `public static void main(String[] args)`?

| Part | Reason |
|---|---|
| `public` | The JVM calls it from outside the class. |
| `static` | The JVM calls it without creating an object first. |
| `void` | Nothing is returned to the JVM. Use `System.exit(code)` to return an exit status. |
| `String[] args` | Command-line arguments (`java Greeter a b` → `args = ["a", "b"]`). `String... args` also works. |

You can overload `main`, but the JVM only calls the `String[]` version (in classic launch mode).

**Java 25 (JEP 512, compact source files & instance main methods):** for single-file programs the
launcher also accepts an instance `void main()` with no class declaration. It's aimed at learning
and scripting. Production code still uses the classic form.

```java
// Hello.java (Java 25)
void main() {
    String name = IO.readln("Name? ");
    IO.println("Hi " + name);
}
```

## 2. Compilation and execution

```bash
javac -d out src/com/example/app/Greeter.java     # → out/com/example/app/Greeter.class
java -cp out com.example.app.Greeter              # run using the classpath
java src/com/example/app/Greeter.java             # single-file source launcher (Java 11+)
jar cfe app.jar com.example.app.Greeter -C out .  # package into an executable jar
java -jar app.jar
javap -c -p out/com/example/app/Greeter.class     # disassemble bytecode
```

## 3. JVM architecture

```
                ┌──────────────────────────────┐
 .class files ─▶│      Class Loader Subsystem  │  Loading → Linking (verify, prepare, resolve) → Initialization
                └──────────────┬───────────────┘
                               ▼
┌──────────────────────── Runtime Data Areas ───────────────────────────┐
│  Shared by all threads:                 Per thread:                    │
│   • Heap (objects, arrays)               • JVM Stack (frames: locals,  │
│   • Method Area / Metaspace                operand stack, return addr) │
│     (class metadata, static fields*,     • PC register                 │
│      runtime constant pool)              • Native method stack         │
└───────────────────────────────────────────────────────────────────────┘
                               ▼
                ┌──────────────────────────────┐
                │      Execution Engine        │  Interpreter · JIT compiler (C1, C2) · Garbage Collector
                └──────────────┬───────────────┘
                               ▼
                 JNI / FFM API ─▶ Native libraries
```
\* Since Java 8, static variables live on the heap (in the `Class` object); class metadata lives in
**Metaspace** (native memory), which replaced **PermGen**.

### Class loading in three phases
1. **Loading**: find the `.class` bytes and create a `Class` object.
2. **Linking**:
   - *Verification*: is the bytecode well-formed and type-safe?
   - *Preparation*: allocate static fields with default values (`0`, `null`, `false`).
   - *Resolution*: replace symbolic references with direct ones (can be lazy).
3. **Initialization**: run static initializers and assign static field values, in source order.
   This happens on **first active use** (creating an instance, calling a static method, accessing a
   non-constant static field), not at startup.

### Class loader hierarchy & delegation

```
Bootstrap ClassLoader     (native; loads java.base: java.lang, java.util ...)
   └─ Platform ClassLoader  (other JDK modules: java.sql, java.xml ...)   [was "Extension" before Java 9]
        └─ Application (System) ClassLoader   (your classpath / module path)
             └─ Custom ClassLoaders (app servers, plugins, hot reload)
```

**Parent-delegation model:** a loader first asks its parent; it loads the class itself only if the
parent can't. This stops anyone from replacing `java.lang.String` with their own version, and
makes sure each core class is loaded once.

A class's identity at runtime is **(fully qualified name + class loader)**. The same class loaded by
two loaders gives two distinct types, which is the cause of the classic
`ClassCastException: com.x.Foo cannot be cast to com.x.Foo` in app servers.

### `ClassNotFoundException` vs `NoClassDefFoundError`
| | `ClassNotFoundException` | `NoClassDefFoundError` |
|---|---|---|
| Type | Checked exception | Error (`LinkageError`) |
| When | Loading by name at runtime: `Class.forName("x")`, `loadClass` | The class was there at compile time but is missing or failed to initialize at runtime |
| Typical cause | Typo, missing driver jar | Missing jar on runtime classpath, or the static initializer threw an exception earlier |

## 4. Interpreter vs JIT

- The **interpreter** executes bytecode instruction by instruction. It starts fast but runs slowly.
- The **JIT (Just-In-Time) compiler** watches for *hot* methods/loops and compiles them to native
  machine code.
  - **C1** (client): quick compile, light optimizations.
  - **C2** (server): slower compile, aggressive optimizations (inlining, escape analysis, loop
    unrolling, lock elision).
  - **Tiered compilation** (default): interpret → C1 → C2 as code gets hotter.
- Startup work is being cut further by **AOT class loading & linking / Project Leyden** (Java 24+,
  extended in 25 and 26): a training run records which classes load, and later runs reuse that
  cache to start faster.

**Why Java is "compiled and interpreted":** `javac` compiles source to bytecode, then the JVM
interprets and JIT-compiles that bytecode.

## 5. Bytecode peek

```java
int add(int a, int b) { return a + b; }
```
`javap -c` shows:
```
iload_1      // push a
iload_2      // push b
iadd         // pop two ints, push sum
ireturn
```
The JVM is a **stack-based** machine: instructions operate on an operand stack inside each frame.

## 6. Packages, imports & access

- `package` groups related classes and provides a namespace and access boundary.
- `import java.util.*;` imports *types* from one package (not sub-packages).
- `import static java.lang.Math.*;` imports static members (`sqrt(2)` instead of `Math.sqrt(2)`).
- **Java 25 module imports (JEP 511):** `import module java.base;` imports every exported package
  of a module.
- **Modules (Java 9, JPMS):** `module-info.java` declares what a module `requires` and `exports`.
  This gives strong encapsulation (internals like `sun.misc` aren't reachable by default). Most app
  code still uses the plain classpath, but you should know the term.

```java
// module-info.java
module com.example.orders {
    requires java.sql;
    exports com.example.orders.api;      // only this package is visible to other modules
}
```

## 7. Gotchas

1. File name must match the **public** class name; only one public top-level class per file.
2. `java Greeter.class` is wrong. Use `java Greeter` (class name, not file name).
3. Static initializers that throw leave the class unusable: the first access throws
   `ExceptionInInitializerError`, later ones throw `NoClassDefFoundError`.
4. `System.exit()` inside `try` skips `finally`.

## 8. Interview questions

1. **JDK vs JRE vs JVM?** JVM runs bytecode; JRE = JVM + libraries (to run); JDK = JRE + tools like
   `javac` (to develop).
2. **Is the JVM platform independent?** No. Bytecode is platform independent; each OS has its own
   JVM implementation.
3. **What is bytecode?** The instruction set of the JVM, stored in `.class` files and produced by
   `javac`.
4. **Explain class loading.** Loading → linking (verify, prepare, resolve) → initialization, done
   lazily by Bootstrap, Platform and Application loaders using parent delegation.
5. **Why parent delegation?** Security (core classes can't be replaced) and consistency (each class
   is loaded once by the right loader).
6. **What is the JIT compiler?** It compiles hot bytecode to native code at runtime, using profiling
   data for optimizations like inlining and escape analysis.
7. **What replaced PermGen?** Metaspace (Java 8), which lives in native memory and grows
   automatically (limit it with `-XX:MaxMetaspaceSize`).
8. **Can you run a Java program without `main`?** Not as an application since Java 7 (the old
   static-block trick no longer works). Java 25 allows an instance `void main()` in compact source
   files, but there's still a `main`.
9. **`ClassNotFoundException` vs `NoClassDefFoundError`?** See the table in section 3.
10. **What are the runtime data areas?** Heap and Method Area/Metaspace (shared); stack, PC
    register and native stack (per thread).
