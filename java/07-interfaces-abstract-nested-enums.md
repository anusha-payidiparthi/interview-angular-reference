# 07 — Interfaces, Abstract Classes, Nested Classes & Enums

## 1. Abstract classes

A class declared `abstract` **can't be instantiated**. It can mix abstract methods (no body) with
concrete methods, fields and constructors. Use it when related classes share **state and code**.

```java
public abstract class Employee {
    private final String name;
    protected final double baseSalary;

    protected Employee(String name, double baseSalary) {   // constructors are allowed
        this.name = name;
        this.baseSalary = baseSalary;
    }

    public abstract double monthlyPay();                     // subclasses must implement

    public final String payslip() {                          // template method: fixed algorithm
        return name + ": " + String.format("%.2f", monthlyPay());
    }
}

public class FullTime extends Employee {
    public FullTime(String name, double salary) { super(name, salary); }
    @Override public double monthlyPay() { return baseSalary / 12; }
}

public class Contractor extends Employee {
    private final int hours;
    public Contractor(String name, double rate, int hours) { super(name, rate); this.hours = hours; }
    @Override public double monthlyPay() { return baseSalary * hours; }
}
```

Rules:
- A class with **any** abstract method must be declared abstract.
- An abstract class may have **zero** abstract methods (just to prevent instantiation).
- A concrete subclass must implement all inherited abstract methods, or be abstract itself.
- `abstract` can't be combined with `final`, `private` or `static` on a method.

## 2. Interfaces

An interface is a **contract**: a set of capabilities a class promises to provide. A class can
implement **many** interfaces.

```java
public interface Shape {
    double PI_APPROX = 3.14;                 // implicitly public static final (a constant)

    double area();                           // implicitly public abstract

    default String describe() {              // Java 8: default method with a body
        return "Shape with area " + area();
    }

    static Shape unitSquare() {              // Java 8: static method, called as Shape.unitSquare()
        return () -> 1.0;                    // lambda works because Shape has one abstract method
    }

    private void log(String msg) {           // Java 9: private helper for default methods
        System.out.println(msg);
    }
}

public class Square implements Shape, Comparable<Square> {
    private final double side;
    public Square(double side) { this.side = side; }
    @Override public double area() { return side * side; }
    @Override public int compareTo(Square o) { return Double.compare(area(), o.area()); }
}
```

What an interface can contain (by version):

| Member | Since | Modifiers |
|---|---|---|
| Constants | 1.0 | implicitly `public static final` |
| Abstract methods | 1.0 | implicitly `public abstract` |
| `default` methods | 8 | `public` |
| `static` methods | 8 | `public` (or `private` since 9) |
| `private` methods | 9 | `private` / `private static` |
| Nested types | 1.0 | implicitly `public static` |

Interfaces can't have instance fields (state) or constructors.

**Why were default methods added?** To evolve interfaces without breaking existing implementations.
Java 8 added `forEach`, `stream()`, `removeIf` to `Collection`/`Iterable` without breaking every
collection class in the world.

### The diamond problem with default methods
```java
interface Flyer  { default String move() { return "fly"; } }
interface Swimmer { default String move() { return "swim"; } }

class Duck implements Flyer, Swimmer {
    @Override public String move() {             // REQUIRED: compile error otherwise
        return Flyer.super.move() + " & " + Swimmer.super.move();
    }
}
```
Resolution rules:
1. **Class wins**: a method from the superclass chain beats any interface default.
2. **More specific interface wins**: if `B extends A` and both define the default, `B`'s is used.
3. Otherwise the class **must override** and may pick one with `X.super.method()`.

### Marker & functional interfaces
- **Marker interface**: no methods, just tags a type: `Serializable`, `Cloneable`, `RandomAccess`.
  (Annotations are the modern alternative.)
- **Functional interface**: exactly **one abstract method**, so it can be the target of a lambda:
  `Runnable`, `Comparator`, `Function<T,R>`. `@FunctionalInterface` makes the compiler enforce it.
  See chapter 12.

## 3. Abstract class vs interface (asked in almost every interview)

| | Abstract class | Interface |
|---|---|---|
| Keyword | `extends` (only **one**) | `implements` (**many**) |
| State (instance fields) | ✅ | ❌ (only constants) |
| Constructors | ✅ | ❌ |
| Method bodies | ✅ any | `default`, `static`, `private` only (Java 8/9+) |
| Access modifiers on methods | Any | `public` (or `private` helpers) |
| Relationship | "is-a" with shared implementation | "can-do" capability / contract |
| Use when | Related classes share code and state; you want a template method | Unrelated classes share a capability; you need multiple inheritance of type; you're defining an API |

Typical modern design: an **interface** for the contract, optionally with an **abstract skeletal
implementation** (e.g. `List` + `AbstractList`, `Map` + `AbstractMap`).

## 4. Nested classes

```
Nested classes
├── static nested class           — no reference to outer instance
└── inner classes (non-static)    — hold an implicit reference to the outer instance
    ├── member inner class
    ├── local class               — declared inside a method
    └── anonymous class           — declared and instantiated in one expression
```

```java
public class Outer {
    private int value = 10;
    private static int counter = 0;

    // 1. Static nested: like a top-level class namespaced inside Outer
    public static class Builder {
        int n;
        Builder n(int n) { this.n = n; return this; }
        // can access Outer's static members (counter) but not 'value'
    }

    // 2. Member inner class: tied to an Outer instance
    public class Inner {
        int read() { return value; }                // accesses outer instance field
        int shadow(int value) { return Outer.this.value + value; }   // explicit outer ref
    }

    void demo() {
        int local = 5;                              // must be effectively final to capture

        // 3. Local class
        class Helper { int calc() { return local * value; } }

        // 4. Anonymous class
        Runnable r = new Runnable() {
            @Override public void run() { System.out.println(local + value); }
        };

        Runnable lambda = () -> System.out.println(local + value);   // modern replacement
    }
}

Outer.Builder b = new Outer.Builder();      // no outer instance needed
Outer outer = new Outer();
Outer.Inner in = outer.new Inner();         // needs an outer instance
```

| | Static nested | Inner (member) | Local | Anonymous |
|---|---|---|---|---|
| Outer instance reference | No | Yes | Yes (if in instance method) | Yes (if in instance method) |
| Can have static members | Yes | Yes since Java 16 | Yes since Java 16 | Yes since Java 16 |
| Typical use | Builders, `Map.Entry`, helpers | Iterators over the outer's data | One-off helper inside a method | One-off implementation (pre-lambda callbacks) |

**Lambda vs anonymous class:** a lambda has no own `this` (it means the enclosing instance), creates
no new scope for variables, only works for functional interfaces, and doesn't produce a separate
`.class` file (it uses `invokedynamic`). An anonymous class can implement interfaces with several
methods, extend classes, and hold state.

**Memory leak warning:** inner and anonymous classes keep the outer object alive. Prefer `static`
nested classes unless you need the outer instance (*Effective Java* item 24).

**Effectively final:** local variables captured by lambdas and inner classes must not be reassigned
after initialization, because the class captures a *copy* of the value.

## 5. Enums

An `enum` is a class with a fixed set of instances. Enums are type-safe, can have fields,
constructors (implicitly private), methods, and per-constant behavior.

```java
public enum Planet {
    MERCURY(3.303e+23, 2.4397e6),
    EARTH(5.976e+24, 6.37814e6);

    private final double mass, radius;

    Planet(double mass, double radius) {      // implicitly private
        this.mass = mass;
        this.radius = radius;
    }

    public double surfaceGravity() { return 6.67300E-11 * mass / (radius * radius); }
}

public enum Operation {
    PLUS("+")  { public int apply(int a, int b) { return a + b; } },
    TIMES("*") { public int apply(int a, int b) { return a * b; } };

    private final String symbol;
    Operation(String symbol) { this.symbol = symbol; }
    public abstract int apply(int a, int b);  // constant-specific body
}

// Built-in methods
Planet.values();                  // [MERCURY, EARTH] (array, new copy every call)
Planet.valueOf("EARTH");          // EARTH; IllegalArgumentException if no match
Planet.EARTH.name();              // "EARTH"
Planet.EARTH.ordinal();           // 1 (don't persist this: reordering breaks it)
Planet.EARTH.compareTo(Planet.MERCURY);  // positive (by ordinal)

// Enums work great with switch, EnumMap and EnumSet
EnumSet<DayOfWeek> weekend = EnumSet.of(DayOfWeek.SATURDAY, DayOfWeek.SUNDAY);
EnumMap<Planet, String> notes = new EnumMap<>(Planet.class);   // array-backed, very fast

String type = switch (day) {
    case SATURDAY, SUNDAY -> "weekend";
    default -> "weekday";
};
```

Facts:
- Every enum implicitly extends `java.lang.Enum`, so it can't extend another class, but it can
  implement interfaces.
- Enum constants are `public static final` singletons, so comparing with `==` is safe and preferred.
- Enums are serialization-safe and reflection-safe singletons, which is why
  `enum Singleton { INSTANCE; }` is the recommended singleton (chapter 17).

## 6. Gotchas

1. Forgetting to resolve conflicting default methods (compile error).
2. Interface constants pollute namespaces (the "constant interface" anti-pattern). Use a `final`
   class with a private constructor or an enum instead.
3. Inner/anonymous classes leaking the outer object (e.g. a listener holding an Activity/Window).
4. Persisting `ordinal()`: reordering the constants corrupts data. Persist `name()` or a field.
5. `Enum.valueOf` is case-sensitive and throws for unknown names.

## 7. Interview questions

1. **Abstract class vs interface?** See the table in section 3. Mention Java 8 default/static and
   Java 9 private methods.
2. **Can an abstract class have a constructor?** Yes. It runs when a subclass is instantiated
   (`super(...)`).
3. **Can an interface have a constructor or instance fields?** No.
4. **Why default methods?** Backward-compatible interface evolution.
5. **How is the diamond problem handled with default methods?** Class wins, then the more specific
   interface; otherwise you must override and can call `X.super.m()`.
6. **What is a functional interface?** One abstract method; can be a lambda target. Examples:
   `Runnable`, `Callable`, `Comparator`, `Predicate`.
7. **What is a marker interface?** An empty interface that tags a type (`Serializable`,
   `Cloneable`).
8. **Static nested class vs inner class?** Inner classes hold a reference to an outer instance;
   static nested classes don't.
9. **Why must captured local variables be effectively final?** The lambda or inner class captures a
   copy of the value, so allowing reassignment would make the two copies diverge.
10. **Lambda vs anonymous class?** Meaning of `this`, scope, functional interfaces only, no extra
    class file. See section 4.
11. **Can an enum extend a class? Implement an interface?** Extend: no (it already extends `Enum`).
    Implement: yes.
12. **Can enums have constructors? Abstract methods?** Yes (private constructors); yes
    (constant-specific bodies).
13. **Why is an enum the best singleton?** The JVM guarantees one instance, with built-in
    protection against serialization and reflection attacks.
