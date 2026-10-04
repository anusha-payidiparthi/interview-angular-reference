# 06 — The Four OOP Pillars

| Pillar | One-liner | Java mechanism |
|---|---|---|
| **Encapsulation** | Hide state, expose behavior | `private` fields + public methods |
| **Inheritance** | Reuse and extend an existing type ("is-a") | `extends`, `implements` |
| **Polymorphism** | One interface, many implementations | Overloading (compile time), overriding (runtime) |
| **Abstraction** | Show *what*, hide *how* | Abstract classes, interfaces |

## 1. Encapsulation

Bundle data with the methods that operate on it, and restrict direct access so the object controls
its own invariants.

```java
public class Temperature {
    private double celsius;                         // hidden

    public double getCelsius() { return celsius; }

    public void setCelsius(double celsius) {
        if (celsius < -273.15) throw new IllegalArgumentException("below absolute zero");
        this.celsius = celsius;                     // validation protects the invariant
    }

    public double getFahrenheit() { return celsius * 9 / 5 + 32; }  // derived, no separate field
}
```
Benefits: validation, freedom to change internals (e.g. store Kelvin later), easier debugging,
read-only or write-only properties.

Encapsulation is not the same as "add getters and setters for every field". Expose behavior
(`account.withdraw(50)`) rather than raw state (`account.setBalance(account.getBalance() - 50)`).

## 2. Inheritance

A subclass **inherits** the non-private members of its superclass and can add or override them.

```java
public class Animal {
    protected final String name;
    public Animal(String name) { this.name = name; }
    public String sound() { return "..."; }
    public String describe() { return name + " says " + sound(); }
}

public class Dog extends Animal {
    public Dog(String name) { super(name); }       // must call a parent constructor
    @Override public String sound() { return "Woof"; }
    public void fetch() { System.out.println(name + " fetches"); }
}

new Dog("Rex").describe();   // "Rex says Woof": describe() calls the overridden sound()
```

### Types of inheritance in Java

| Type | Supported with classes? |
|---|---|
| Single (`B extends A`) | ✅ |
| Multilevel (`C extends B extends A`) | ✅ |
| Hierarchical (`B`, `C` both extend `A`) | ✅ |
| **Multiple** (`C extends A, B`) | ❌ with classes, ✅ with interfaces |
| Hybrid | Only via interfaces |

**Why no multiple inheritance of classes?** The **diamond problem**: if `B` and `C` both override
`A.foo()` and `D extends B, C`, which `foo()` does `D` get, and how many copies of `A`'s state? Java
avoids this for state entirely. For interface default methods it forces you to resolve the
conflict explicitly (chapter 07).

Every class implicitly extends `java.lang.Object`.

### `super` keyword
```java
class Cat extends Animal {
    Cat(String name) { super(name); }                   // parent constructor
    @Override public String sound() { return "Meow"; }
    @Override public String describe() {
        return super.describe() + " (purrs)";           // call parent's version
    }
}
```

### Composition over inheritance
Inheritance couples you tightly to the parent's implementation (the "fragile base class"
problem). Prefer **has-a** (composition) unless there's a true **is-a** relationship and the parent
was designed for extension.

```java
// Inheritance misuse: a Stack is not really a Vector (java.util.Stack makes this mistake)
// Composition:
public class Stack<T> {
    private final Deque<T> items = new ArrayDeque<>();   // has-a
    public void push(T t) { items.push(t); }
    public T pop() { return items.pop(); }
}
```

## 3. Polymorphism

"Many forms": the same call behaves differently depending on the object.

### Compile-time polymorphism: method overloading
Same name, **different parameter list** (number, type or order) in the same class (or inherited).
Return type alone is **not** enough.

```java
class Printer {
    void print(int x)            { System.out.println("int " + x); }
    void print(long x)           { System.out.println("long " + x); }
    void print(Integer x)        { System.out.println("Integer " + x); }
    void print(Object x)         { System.out.println("Object " + x); }
    void print(int... xs)        { System.out.println("varargs"); }
    // int print(int x) { }      // compile error: differs only by return type
}

Printer p = new Printer();
byte b = 1;
p.print(b);        // "int 1"   (widening byte → int beats boxing)
p.print(5);        // "int 5"   (exact match)
p.print(5L);       // "long 5"
p.print("hi");     // "Object hi"
p.print();         // "varargs"
```
**Overload resolution order:** exact match → widening → boxing/unboxing → varargs. The compiler
picks the method using the **declared (static) types** of the arguments.

```java
Object o = "hello";
p.print(o);        // "Object hello": decided at compile time, NOT by the runtime type
```

### Runtime polymorphism: method overriding (dynamic dispatch)
A subclass provides its own implementation of an inherited instance method. The JVM picks the
implementation based on the **actual object type** at runtime.

```java
Animal a = new Dog("Rex");      // reference type Animal, object type Dog (upcasting, implicit)
a.sound();                      // "Woof": runtime type decides
// a.fetch();                   // compile error: Animal has no fetch()
if (a instanceof Dog d) d.fetch();   // downcast with pattern matching (Java 16)

List<Animal> zoo = List.of(new Dog("Rex"), new Cat("Tom"));
for (Animal x : zoo) System.out.println(x.sound());   // Woof, Meow
```

### Overriding rules
1. Same name and parameter list.
2. Return type same or a **subtype** (covariant return, Java 5).
3. Access can't be more restrictive (can be wider).
4. Can't throw **new or broader checked** exceptions (can throw fewer or narrower, or any unchecked).
5. `final`, `static` and `private` methods can't be overridden (static is *hidden*, private is
   invisible).
6. Use `@Override`: the compiler then catches typos like `equals(Dog d)` instead of
   `equals(Object o)`.

### Overloading vs overriding

| | Overloading | Overriding |
|---|---|---|
| Where | Same class (or subclass) | Subclass |
| Parameters | Must differ | Must be the same |
| Return type | Anything | Same or covariant |
| Binding | Compile time (static) | Runtime (dynamic) |
| `static`/`private`/`final` methods | Can be overloaded | Can't be overridden |
| Exceptions | No restriction | No new/broader checked exceptions |

### What is NOT polymorphic
- **Fields**: resolved by reference type.
- **Static methods**: resolved by reference type.

```java
class A { String name = "A"; static String s() { return "A"; } String m() { return "A"; } }
class B extends A { String name = "B"; static String s() { return "B"; } @Override String m() { return "B"; } }

A obj = new B();
obj.name;   // "A"  (field: reference type)
obj.s();    // "A"  (static: reference type, method hiding)
obj.m();    // "B"  (instance method: object type)
```

## 4. Abstraction

Expose essential behavior, hide implementation details. Callers code against the abstraction.

```java
public abstract class Shape {
    public abstract double area();                 // what every shape must do
    public String summary() {                      // shared concrete behavior
        return getClass().getSimpleName() + " with area " + String.format("%.2f", area());
    }
}

public class Circle extends Shape {
    private final double r;
    public Circle(double r) { this.r = r; }
    @Override public double area() { return Math.PI * r * r; }
}

public interface PaymentGateway {                  // pure contract
    Receipt charge(Money amount, Card card);
}

class StripeGateway implements PaymentGateway { /* ... */ }
class MockGateway   implements PaymentGateway { /* ... for tests */ }

class Checkout {
    private final PaymentGateway gateway;          // depends on the abstraction, not Stripe
    Checkout(PaymentGateway gateway) { this.gateway = gateway; }
}
```

**Abstraction vs encapsulation:** abstraction is about *design*: what to expose (interfaces,
abstract types). Encapsulation is about *implementation*: how to hide and protect state (access
modifiers).

## 5. Association, aggregation, composition

| Relationship | Meaning | Lifetime | Example |
|---|---|---|---|
| Association | Uses / knows about | Independent | `Teacher` ↔ `Student` |
| Aggregation | Has-a (weak) | Part can exist alone | `Department` has `Professor`s |
| Composition | Owns (strong) | Part dies with whole | `House` has `Room`s |

## 6. Casting objects

```java
Animal a = new Dog("Rex");       // upcast: always safe, implicit
Dog d = (Dog) a;                 // downcast: explicit, checked at runtime
Cat c = (Cat) a;                 // compiles, throws ClassCastException at runtime

if (a instanceof Cat cat) {      // safe pattern (Java 16+)
    System.out.println(cat.sound());
}
```

## 7. Gotchas

1. Overloading is resolved at compile time from static types, so `print(Object)` gets called for
   `Object o = "x"`.
2. `equals(MyType other)` **overloads** `Object.equals` instead of overriding it, and collections
   ignore it. Always use `@Override`.
3. A parent constructor calling an overridable method (chapter 05).
4. Overloading with `null`: `print(null)` with `print(String)` and `print(Object)` picks the most
   specific (`String`); with `print(String)` and `print(Integer)` it's ambiguous (compile error).
5. Inheritance for code reuse alone tends to produce brittle hierarchies.

## 8. Interview questions

1. **What are the four OOP pillars?** Encapsulation, inheritance, polymorphism, abstraction (with
   one-line definitions plus an example of each).
2. **Overloading vs overriding?** See the table in section 3.
3. **Can we overload `main`?** Yes, but the JVM only calls `main(String[])`.
4. **Can we override a private or static method?** No. Private isn't inherited; static is hidden,
   not overridden.
5. **What is dynamic method dispatch?** The JVM chooses the overridden method based on the runtime
   object type (via the vtable).
6. **What is covariant return type?** An overriding method may return a subtype of the parent's
   return type.
7. **Why doesn't Java support multiple inheritance of classes?** The diamond problem (ambiguous
   method and duplicated state). Interfaces give multiple *type* inheritance instead.
8. **Abstraction vs encapsulation?** Hiding complexity behind a contract vs hiding state behind
   access control.
9. **Composition vs inheritance; which do you prefer?** Composition by default: looser coupling,
   behavior can change at runtime, no fragile base class.
10. **Are fields polymorphic?** No. Field access uses the reference type.
11. **What is upcasting and downcasting?** Treating a subclass as its parent type (implicit) vs
    casting back to the subclass (explicit, can throw `ClassCastException`).
12. **Can an overriding method throw a checked exception the parent doesn't declare?** No. It can
    throw narrower checked exceptions, fewer of them, or unchecked ones.
