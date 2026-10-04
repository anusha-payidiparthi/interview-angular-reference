# 11 — Generics

## 1. Why generics?

Before Java 5, collections held `Object`, so casts were everywhere and type errors surfaced at
runtime:
```java
List list = new ArrayList();          // raw type
list.add("hello");
list.add(42);                         // compiles
String s = (String) list.get(1);      // ClassCastException at runtime
```
With generics the compiler catches it:
```java
List<String> list = new ArrayList<>();   // <> diamond operator (Java 7)
list.add(42);                            // compile error
String s = list.get(0);                  // no cast
```
Benefits: **compile-time type safety**, **no casts**, **reusable generic algorithms**.

## 2. Generic classes, interfaces & methods

```java
public class Box<T> {                        // T = type parameter
    private T value;
    public Box(T value) { this.value = value; }
    public T get() { return value; }
    public <R> Box<R> map(Function<? super T, ? extends R> f) {   // generic method
        return new Box<>(f.apply(value));
    }
}

Box<Integer> b = new Box<>(5);
Box<String> s = b.map(i -> "#" + i);

public interface Repository<T, ID> {
    Optional<T> findById(ID id);
    List<T> findAll();
    T save(T entity);
}

public class Pair<K, V> {
    private final K key; private final V value;
    public Pair(K key, V value) { this.key = key; this.value = value; }
    public K key() { return key; }
    public V value() { return value; }
}

// Generic static method: the type parameter goes before the return type
public static <T> void swap(T[] arr, int i, int j) {
    T tmp = arr[i]; arr[i] = arr[j]; arr[j] = tmp;
}
Util.<String>swap(names, 0, 1);    // explicit type witness (rarely needed)
```

Naming conventions: `T` type, `E` element, `K`/`V` key/value, `R` result, `N` number, `S, U` extra
types.

## 3. Bounded type parameters

```java
// Upper bound: T must be Number or a subclass
public static <T extends Number> double sum(List<T> nums) {
    double total = 0;
    for (T n : nums) total += n.doubleValue();     // can call Number methods
    return total;
}

// Multiple bounds: class first, then interfaces
public static <T extends Number & Comparable<T>> T max(List<T> list) {
    T best = list.get(0);
    for (T t : list) if (t.compareTo(best) > 0) best = t;
    return best;
}

// Recursive bound: the classic signature for "comparable to itself"
public static <T extends Comparable<? super T>> void sort(List<T> list) { ... }
```

## 4. Wildcards: `?`, `? extends`, `? super`

Generics are **invariant**: `List<Integer>` is **not** a `List<Number>`, even though `Integer` is a
`Number`.
```java
List<Integer> ints = new ArrayList<>();
// List<Number> nums = ints;     // compile error. If allowed:
// nums.add(3.14);               // ...a Double would end up inside a List<Integer>
```
Wildcards restore flexibility:

| Wildcard | Accepts | Read as | Can add? |
|---|---|---|---|
| `List<?>` | List of anything | `Object` | Only `null` |
| `List<? extends Number>` | `List<Number>`, `List<Integer>`, `List<Double>` … | `Number` | ❌ (only `null`) |
| `List<? super Integer>` | `List<Integer>`, `List<Number>`, `List<Object>` | `Object` | ✅ `Integer` (and subtypes) |

### PECS: Producer Extends, Consumer Super
- If the collection **produces** values you read → `? extends T`.
- If the collection **consumes** values you write → `? super T`.

```java
public static double sumAll(Collection<? extends Number> source) {     // producer: we read
    double s = 0;
    for (Number n : source) s += n.doubleValue();
    return s;
}

public static void fillWithInts(List<? super Integer> target) {         // consumer: we write
    for (int i = 0; i < 3; i++) target.add(i);
}

// java.util.Collections uses PECS:
public static <T> void copy(List<? super T> dest, List<? extends T> src) { ... }

sumAll(List.of(1, 2, 3));             // List<Integer> ✅
sumAll(List.of(1.5, 2.5));            // List<Double> ✅
fillWithInts(new ArrayList<Number>()); // ✅
fillWithInts(new ArrayList<Object>()); // ✅
```

## 5. Type erasure

Generics exist **only at compile time**. The compiler checks types, inserts casts, and then
**erases** type parameters to their bound (`Object` if unbounded). That's how Java 5 stayed
backward-compatible with pre-generics bytecode.

```java
// You write:
public class Box<T> { T value; T get() { return value; } }
// Bytecode is roughly:
public class Box { Object value; Object get() { return value; } }
// And at call sites:  String s = (String) box.get();

// <T extends Number> erases to Number
```

Consequences (frequent interview follow-ups):
```java
new ArrayList<String>().getClass() == new ArrayList<Integer>().getClass();  // true: both ArrayList

// None of these compile:
// if (list instanceof List<String>) { }   // can't check a parameterized type at runtime
//                                          // (List<?> is OK; Java 16+ allows it if provably safe)
// T obj = new T();                         // T's class is unknown at runtime
// T[] arr = new T[10];                     // generic array creation
// static T field;                          // static is shared across all parameterizations
// List<int> nums;                          // primitives can't be type arguments
// void m(List<String> l) {} void m(List<Integer> l) {}   // same erasure → clash

// Workaround: pass a Class<T> token
public static <T> T create(Class<T> type) throws ReflectiveOperationException {
    return type.getDeclaredConstructor().newInstance();
}
```
The compiler generates **bridge methods** so that overriding still works after erasure (e.g. a
`compareTo(Object)` that casts and calls your `compareTo(Person)`).

**Heap pollution:** a variable of a parameterized type refers to an object of a different
parameterization, which happens with raw types or generic varargs. That's why `@SafeVarargs`
exists:
```java
@SafeVarargs
public static <T> List<T> listOf(T... items) { return List.of(items); }
```

## 6. Raw types

`List list = new ArrayList();` is a **raw type**. It only exists for legacy compatibility; it turns
off generic checks and produces "unchecked" warnings. Never use raw types in new code. Use `List<?>`
if you really don't care about the type.

## 7. Generics and inheritance

```java
class Animal {}  class Dog extends Animal {}

Dog d = new Dog();  Animal a = d;            // ✅ ordinary subtyping
List<Dog> dogs = new ArrayList<>();
// List<Animal> animals = dogs;              // ❌ invariant
List<? extends Animal> animals = dogs;       // ✅ covariant view (read-only)
ArrayList<Dog> al = new ArrayList<>();
List<Dog> l = al;                            // ✅ same type argument, subtype container
```

## 8. Gotchas

1. Expecting `List<Integer>` to be assignable to `List<Number>`.
2. Trying to `add` into a `List<? extends T>`.
3. Overloading methods that differ only in generic parameters (erasure clash).
4. Mixing raw types and generics, which leads to `ClassCastException` far from the cause.
5. `var list = new ArrayList<>();` infers `ArrayList<Object>`.

## 9. Interview questions

1. **What are generics and why use them?** Parameterized types for compile-time safety, no casts,
   and reusable code.
2. **What is type erasure?** The compiler removes type parameters (replacing them with their bounds)
   after type-checking, so generic type info isn't available at runtime.
3. **Why can't you create `new T()` or `new T[]`?** `T`'s actual class isn't known at runtime due to
   erasure.
4. **What is the diamond operator?** `<>` (Java 7) infers type arguments from the context.
5. **`? extends` vs `? super`?** Upper-bounded (read as T, can't add) vs lower-bounded (can add T,
   read as Object).
6. **Explain PECS.** Producer Extends, Consumer Super, e.g. `Collections.copy(dest super, src
   extends)`.
7. **Is `List<String>` a subtype of `List<Object>`?** No; generics are invariant. `List<String>` is a
   subtype of `List<?>` and `List<? extends Object>`.
8. **Can you use primitives as type arguments?** No; use wrappers (boxing cost). Project Valhalla
   aims to change this in the future.
9. **What is a raw type?** A generic type used without type arguments; legacy only.
10. **What are bounded type parameters?** `<T extends X & Y>` restricts `T` and lets you call `X`'s
    methods.
11. **What are bridge methods?** Synthetic methods the compiler generates to keep polymorphism
    working after erasure.
12. **Why does `List<String>.class` not compile?** There's only one `Class` object, `List.class`;
    parameterized class literals don't exist.
