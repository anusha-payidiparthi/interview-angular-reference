# 08 — `Object` Class, equals/hashCode, Immutability & Records

## 1. Methods of `java.lang.Object`

Every class inherits these:

| Method | Purpose | Override? |
|---|---|---|
| `boolean equals(Object o)` | Logical equality (default: `this == o`) | Often |
| `int hashCode()` | Hash for hash-based collections (default: identity-based) | Whenever you override `equals` |
| `String toString()` | Text form (default: `ClassName@hexHash`) | Usually |
| `Class<?> getClass()` | Runtime class | `final` |
| `protected Object clone()` | Field-by-field copy | Rarely (prefer copy constructors) |
| `protected void finalize()` | Called before GC | **Deprecated for removal** (Java 9/18). Don't use. |
| `wait()`, `notify()`, `notifyAll()` | Thread coordination on the object's monitor | `final` (chapter 13) |

## 2. `==` vs `equals()`

```java
String a = new String("x"), b = new String("x");
a == b;          // false: different objects (reference comparison)
a.equals(b);     // true:  String overrides equals to compare content

Point p1 = new Point(1, 2), p2 = new Point(1, 2);
p1.equals(p2);   // false unless Point overrides equals (default is ==)
```

## 3. The `equals()` contract

For non-null references, `equals` must be:
1. **Reflexive**: `x.equals(x)` is `true`.
2. **Symmetric**: `x.equals(y)` ⇔ `y.equals(x)`.
3. **Transitive**: `x.equals(y)` and `y.equals(z)` ⇒ `x.equals(z)`.
4. **Consistent**: repeated calls return the same result if nothing changed.
5. **Non-null**: `x.equals(null)` is `false`.

## 4. The `hashCode()` contract

1. Consistent within one execution if the fields used by `equals` don't change.
2. **If `a.equals(b)` then `a.hashCode() == b.hashCode()`**. This is mandatory.
3. Unequal objects *may* share a hash code (a collision). Fewer collisions means better
   performance.

**What breaks if you override `equals` but not `hashCode`?** Hash-based collections look in the
wrong bucket:

```java
class Emp {
    String id;
    Emp(String id) { this.id = id; }
    @Override public boolean equals(Object o) {
        return o instanceof Emp e && id.equals(e.id);
    }
    // no hashCode!
}

Set<Emp> set = new HashSet<>();
set.add(new Emp("1"));
set.contains(new Emp("1"));   // false (almost always): different identity hash → different bucket
set.add(new Emp("1"));        // a "duplicate" gets added
```

## 5. Writing `equals` and `hashCode` correctly

```java
public final class Employee {
    private final String id;
    private final String name;
    private final int age;

    public Employee(String id, String name, int age) { this.id = id; this.name = name; this.age = age; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;                         // fast path
        if (!(o instanceof Employee other)) return false;   // handles null too
        return age == other.age
            && Objects.equals(id, other.id)                 // null-safe
            && Objects.equals(name, other.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, name, age);                 // uses the SAME fields as equals
    }

    @Override
    public String toString() {
        return "Employee[id=%s, name=%s, age=%d]".formatted(id, name, age);
    }
}
```

**`instanceof` vs `getClass() != o.getClass()`:**
- `getClass()` gives strict equality: a subclass is never equal to its parent. This keeps symmetry
  when subclasses add fields.
- `instanceof` allows subclass instances to be equal. Fine for `final` classes; risky for
  extensible ones.

**Mutable keys are dangerous:** if a field used in `hashCode` changes after the object is put in a
`HashMap`/`HashSet`, you can't find it again. Use immutable keys.

## 6. `Comparable` vs `Comparator`

| | `Comparable<T>` | `Comparator<T>` |
|---|---|---|
| Package | `java.lang` | `java.util` |
| Method | `int compareTo(T o)` | `int compare(T a, T b)` |
| Defines | The **natural order**, inside the class | **External** orders, as many as you like |
| Used by | `Collections.sort(list)`, `TreeMap`, `TreeSet` (default) | `list.sort(cmp)`, `new TreeMap<>(cmp)` |

```java
public record Student(String name, double gpa, int age) implements Comparable<Student> {
    @Override public int compareTo(Student o) {
        return name.compareTo(o.name);           // natural order: by name
    }
}

List<Student> students = new ArrayList<>(List.of(
    new Student("Ana", 3.9, 21), new Student("Raj", 3.5, 22), new Student("Bo", 3.9, 20)));

Collections.sort(students);                                       // by name (Comparable)
students.sort(Comparator.comparingDouble(Student::gpa).reversed() // GPA desc
                        .thenComparing(Student::age));            // then age asc
students.sort(Comparator.comparing(Student::name, String.CASE_INSENSITIVE_ORDER));
students.sort(Comparator.comparing(Student::name, Comparator.nullsLast(Comparator.naturalOrder())));
```

Return value: negative if `a < b`, zero if equal, positive if `a > b`.

**Never** use `return a.age - b.age;`: it overflows for large or negative values. Use
`Integer.compare(a.age, b.age)` or `Comparator.comparingInt`.

Keep `compareTo` **consistent with `equals`** (`compareTo == 0` ⇔ `equals`), otherwise
`TreeSet`/`TreeMap` (which use `compareTo`) and `HashSet` (which uses `equals`) disagree about
duplicates. `BigDecimal` is the famous exception: `new BigDecimal("1.0")` and `"1.00"` are equal by
`compareTo` but not `equals`.

## 7. `clone()`, shallow vs deep copy

```java
class Team implements Cloneable {
    String name;
    List<String> members = new ArrayList<>();

    @Override
    public Team clone() {
        try {
            Team copy = (Team) super.clone();          // shallow: copy.members is the SAME list
            copy.members = new ArrayList<>(members);   // deep-copy mutable fields manually
            return copy;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(e);               // can't happen: we implement Cloneable
        }
    }
}
```
- Without `implements Cloneable`, `Object.clone()` throws `CloneNotSupportedException`.
- **Shallow copy**: copies field values, so references are shared. **Deep copy**: also copies the
  referenced objects.
- `clone()` is widely considered broken (a marker interface that changes the behavior of a protected
  method, no constructor call, trouble with `final` fields). Prefer a **copy constructor** or a
  **static factory** (`Team.copyOf(team)`).

## 8. Immutable classes

An immutable object's state can't change after construction. Examples: `String`, wrappers,
`LocalDate`, `BigDecimal`, records (shallowly).

Recipe:
1. Declare the class `final` (or use a private constructor with factories).
2. Make all fields `private final`.
3. No setters or other mutators.
4. **Defensive copies** of mutable inputs in the constructor and of mutable outputs in getters.
5. Don't let `this` escape during construction.

```java
public final class Order {
    private final String id;
    private final List<String> items;
    private final Date createdAt;            // mutable legacy type

    public Order(String id, List<String> items, Date createdAt) {
        this.id = id;
        this.items = List.copyOf(items);                       // immutable copy
        this.createdAt = new Date(createdAt.getTime());        // defensive copy
    }

    public String getId() { return id; }
    public List<String> getItems() { return items; }           // already unmodifiable
    public Date getCreatedAt() { return new Date(createdAt.getTime()); }   // copy out

    public Order withItem(String item) {                       // "wither" returns a new object
        List<String> next = new ArrayList<>(items);
        next.add(item);
        return new Order(id, next, createdAt);
    }
}
```
Benefits: thread-safe without locks, safe as map keys, hash can be cached, easy to reason about,
safe to share. Cost: more allocations (usually negligible).

`Collections.unmodifiableList(list)` is a **read-only view**: changes to the underlying list still
show through. `List.copyOf` / `List.of` are truly immutable.

## 9. Records (Java 16)

A **record** is a transparent, shallowly immutable data carrier. The compiler generates the
canonical constructor, accessors, `equals`, `hashCode` and `toString`.

```java
public record Point(int x, int y) { }

Point p = new Point(1, 2);
p.x();                              // accessor: x(), not getX()
p.equals(new Point(1, 2));          // true
p.toString();                       // "Point[x=1, y=2]"
```

Customizing:
```java
public record Range(int start, int end) {
    public Range {                                   // compact canonical constructor: validation
        if (start > end) throw new IllegalArgumentException("start > end");
    }

    public Range(int end) { this(0, end); }          // extra constructors must delegate

    public int length() { return end - start; }      // instance methods OK

    public static Range empty() { return new Range(0, 0); }   // static members OK
}

public record Team(String name, List<String> members) {
    public Team {
        members = List.copyOf(members);              // defensive copy → really immutable
    }
}
```

Rules:
- Implicitly `final`; fields are `private final`; can't extend another class (implicitly extends
  `java.lang.Record`); **can** implement interfaces.
- No extra instance fields beyond the components (static fields are fine).
- Great for DTOs, value objects, map keys, multiple return values, and pattern matching:

```java
if (obj instanceof Point(int x, int y)) {            // record pattern (Java 21)
    System.out.println(x + y);
}
```

Records vs Lombok `@Data`: records are immutable and built into the language; `@Data` generates
mutable JavaBeans. JPA entities can't be records (they need a no-arg constructor and mutability),
but records are perfect for DTOs and projections.

## 10. Gotchas

1. Overriding `equals(Employee e)` instead of `equals(Object o)` overloads it, and collections
   ignore it.
2. Overriding `equals` without `hashCode`.
3. Using mutable fields in `hashCode` for objects stored in hash collections.
4. Subtraction-based comparators overflow.
5. `clone()` being shallow by default.
6. Record components holding mutable lists: the record is only *shallowly* immutable unless you
   copy.

## 11. Interview questions

1. **Name the methods of `Object`.** `equals`, `hashCode`, `toString`, `getClass`, `clone`,
   `finalize` (deprecated), `wait`, `notify`, `notifyAll`.
2. **`==` vs `equals`?** Identity vs logical equality.
3. **What is the equals/hashCode contract?** Equal objects must have equal hash codes; equals must
   be reflexive, symmetric, transitive, consistent and non-null.
4. **What happens if you override only `equals`?** Hash collections break: duplicates appear and
   lookups fail.
5. **Can two unequal objects have the same hash code?** Yes, that's a collision. The reverse isn't
   allowed.
6. **`Comparable` vs `Comparator`?** Natural order inside the class vs external, pluggable orders.
7. **Shallow vs deep copy?** Copying references vs recursively copying the referenced objects.
8. **Why avoid `clone()`?** Fragile design. Copy constructors or factories are clearer and work with
   `final` fields.
9. **How do you create an immutable class?** `final` class, `private final` fields, no setters,
   defensive copies in and out.
10. **What is a record? Limitations?** A compiler-generated immutable data carrier; can't extend
    classes or add instance fields; shallowly immutable.
11. **Is a record a good `HashMap` key?** Yes, as long as its components are immutable.
12. **`unmodifiableList` vs `List.copyOf`?** A read-only *view* (reflects source changes) vs a true
    immutable copy.
