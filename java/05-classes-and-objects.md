# 05 — Classes & Objects

## 1. Class vs object

A **class** is a blueprint (fields + methods); an **object** is an instance of it created with
`new`, living on the heap.

```java
public class BankAccount {
    // fields (state)
    private final String id;
    private double balance;
    private static int accountCount = 0;          // shared by all instances

    // constructor
    public BankAccount(String id, double openingBalance) {
        this.id = id;
        this.balance = openingBalance;
        accountCount++;
    }

    // behavior
    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("amount must be positive");
        balance += amount;
    }

    public double getBalance() { return balance; }
    public static int getAccountCount() { return accountCount; }
}

BankAccount acc = new BankAccount("A-1", 100);   // reference 'acc' on stack → object on heap
acc.deposit(50);
BankAccount.getAccountCount();                   // static: called on the class
```

**What `new` does:** allocates memory on the heap → sets fields to default values → runs field
initializers and instance init blocks → runs the constructor body → returns the reference.

## 2. Constructors

- Same name as the class, **no return type** (not even `void`; adding one makes it a method!).
- If you write **no** constructor, the compiler adds a public no-arg **default constructor**. If
  you write *any* constructor, the default one is **not** generated.
- Constructors aren't inherited and can't be `static`, `final` or `abstract`.
- They can be `private` (singletons, factory methods, utility classes).

```java
public class Pizza {
    private final String size;
    private final List<String> toppings;

    public Pizza() {
        this("medium");                        // constructor chaining
    }

    public Pizza(String size) {
        this(size, List.of());
    }

    public Pizza(String size, List<String> toppings) {
        this.size = size;
        this.toppings = List.copyOf(toppings); // defensive copy
    }
}
```

`this(...)` calls another constructor of the same class; `super(...)` calls the parent's.
Traditionally one of them had to be the **first statement**. If you write neither, the compiler
inserts `super()`.

**Java 25 (JEP 513, flexible constructor bodies):** statements that don't touch `this` may now
appear *before* `this(...)`/`super(...)`, which is handy for validating arguments:
```java
public class PositiveBox extends Box {
    public PositiveBox(int value) {
        if (value <= 0) throw new IllegalArgumentException();  // allowed before super() in Java 25
        super(value);
    }
}
```

**Copy constructor** (Java's alternative to `clone()`):
```java
public Pizza(Pizza other) { this(other.size, other.toppings); }
```

## 3. `this` keyword

1. Disambiguate a field from a parameter: `this.balance = balance;`
2. Call another constructor: `this(...)`
3. Pass the current object: `registry.register(this);`
4. Return the current object for fluent APIs: `return this;`

Can't be used in a `static` context, because there is no current object there.

## 4. `static`

`static` members belong to the **class**, not to an instance.

```java
public class MathUtil {
    public static final double PI = 3.14159;    // constant: static final, UPPER_SNAKE_CASE
    private static int calls;                   // one copy shared by all

    static {                                    // static initializer: runs once at class init
        System.out.println("MathUtil loaded");
    }

    private MathUtil() {}                       // prevent instantiation of utility class

    public static int square(int x) { calls++; return x * x; }
}
```

| | Static method | Instance method |
|---|---|---|
| Called on | Class: `MathUtil.square(3)` | Object: `acc.deposit(5)` |
| Can access | Only static members directly | Static and instance members |
| `this`/`super` | Not available | Available |
| Overridable? | No: **method hiding** (resolved at compile time by reference type) | Yes (runtime polymorphism) |

**Static nested class, static import and static block** are the other uses (see chapters 07 and 01).

**Why is `main` static?** So the JVM can call it without creating an object.

## 5. Initialization order (classic trick question)

```java
class Parent {
    static { System.out.println("1. Parent static block"); }
    { System.out.println("3. Parent instance block"); }
    Parent() { System.out.println("4. Parent constructor"); }
}

class Child extends Parent {
    static { System.out.println("2. Child static block"); }
    { System.out.println("5. Child instance block"); }
    Child() { System.out.println("6. Child constructor"); }
}

new Child();
new Child();   // second time: only 3, 4, 5, 6 (static blocks run once per class)
```

Order:
1. Static fields & static blocks, **parent first**, in textual order (once, when the class is
   initialized).
2. On each `new`: parent instance fields/blocks → parent constructor → child instance fields/blocks
   → child constructor.

**Trap:** calling an overridable method from a parent constructor runs the child's override
*before* the child's fields are initialized:
```java
class Base { Base() { init(); } void init() {} }
class Sub extends Base {
    private String name = "sub";
    @Override void init() { System.out.println(name.length()); }  // NPE: name is still null
}
```
Don't call overridable methods from constructors.

## 6. Access modifiers

| Modifier | Same class | Same package | Subclass (other package) | Everywhere |
|---|---|---|---|---|
| `private` | ✅ | ❌ | ❌ | ❌ |
| *(default / package-private)* | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ (through inheritance) | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

- Top-level classes can only be `public` or package-private.
- An overriding method can't have **weaker** access (`public` → `protected` is an error).
- Interface methods are implicitly `public` (private interface methods exist since Java 9).

## 7. Non-access modifiers cheat list

| Modifier | On a class | On a method | On a variable |
|---|---|---|---|
| `final` | Can't be subclassed (`String`) | Can't be overridden | Can't be reassigned (a constant if also static) |
| `static` | Only nested classes | Belongs to class | One copy per class |
| `abstract` | Can't be instantiated | No body; subclass must implement | n/a |
| `synchronized` | n/a | Acquires the monitor lock | n/a |
| `transient` | n/a | n/a | Skipped by serialization |
| `volatile` | n/a | n/a | Reads/writes go to main memory; visibility guarantee |
| `native` | n/a | Implemented in native code (JNI) | n/a |
| `sealed`/`non-sealed` | Restricts subclasses (Java 17) | n/a | n/a |

`final` on a reference variable means the *reference* can't change; the object can still be
mutated:
```java
final List<String> list = new ArrayList<>();
list.add("ok");          // fine
// list = new ArrayList<>();   // compile error
```

## 8. Getters, setters & JavaBeans

A **JavaBean** has a public no-arg constructor, private fields, `getX()`/`isX()`/`setX()`
accessors, and is usually `Serializable`. Frameworks (Spring, Hibernate, Jackson) rely on these
conventions. For pure data carriers, prefer **records** (chapter 08).

## 9. Object creation: five ways

1. `new Foo()`
2. Reflection: `Foo.class.getDeclaredConstructor().newInstance()`
3. `clone()`
4. Deserialization (`ObjectInputStream.readObject()`, which doesn't call the class's constructor)
5. Factory / builder methods (`List.of()`, `LocalDate.of()`), which use `new` internally

## 10. Gotchas

1. Adding a parameterized constructor removes the default one, so `new Foo()` stops compiling (and
   so do subclasses that implicitly call `super()`).
2. `void Foo() {}` is a method, not a constructor.
3. Static fields hold state across tests and requests, a common source of leaks and flaky tests.
4. Calling a static method via an instance (`acc.getAccountCount()`) compiles but misleads, and it
   works even when `acc` is `null`.

## 11. Interview questions

1. **Class vs object?** Blueprint vs instance.
2. **What is a default constructor?** The no-arg constructor the compiler adds only when you
   declare none.
3. **Can a constructor be private? Final? Static?** Private yes; final/static/abstract no.
4. **Can constructors be inherited or overridden?** No. They can be overloaded and chained.
5. **`this()` vs `super()`?** Call another constructor in the same class vs the parent class.
   Before Java 25 one of them had to be the first statement.
6. **Static vs instance members?** Class-level and shared vs per-object.
7. **Can we override static methods?** No. A same-signature static method in a subclass *hides* the
   parent's; which one runs depends on the reference type at compile time.
8. **What runs first: static block, instance block or constructor?** Static blocks (once, parent
   first), then per object: instance blocks/field initializers, then the constructor (parent chain
   first).
9. **Can a top-level class be `private` or `protected`?** No. Only `public` or package-private.
10. **What does `final` mean for a variable, method and class?** No reassignment, no override, no
    subclass.
11. **Why shouldn't you call overridable methods in constructors?** The subclass override runs
    before the subclass is initialized.
12. **Ways to create an object?** `new`, reflection, `clone`, deserialization, factory methods.
