# 03 — Control Flow & Arrays

## 1. Conditionals

```java
if (score >= 90) {
    grade = 'A';
} else if (score >= 80) {
    grade = 'B';
} else {
    grade = 'C';
}

String label = age >= 18 ? "adult" : "minor";   // ternary
```
Conditions must be `boolean`. `if (count)` doesn't compile (no truthy/falsy like JS).

## 2. `switch`: classic statement vs modern expression

### Classic switch statement (falls through!)
```java
switch (day) {
    case "SAT":
    case "SUN":
        System.out.println("weekend");
        break;                     // forget this and execution falls into the next case
    case "MON":
        System.out.println("ugh");
        break;
    default:
        System.out.println("weekday");
}
```
Supported selector types: `byte`, `short`, `char`, `int` and their wrappers, `String` (Java 7),
`enum`; since Java 21, **any reference type** with pattern matching.

### Switch expression (Java 14+): no fall-through, returns a value
```java
int letters = switch (day) {
    case "MON", "FRI", "SUN" -> 6;
    case "TUE"               -> 7;
    case "THU", "SAT"        -> 8;
    case "WED"               -> 9;
    default -> {
        System.out.println("unknown " + day);
        yield 0;                   // 'yield' returns a value from a block
    }
};
```
- `->` arms don't fall through, so there's no `break`.
- A switch *expression* must be **exhaustive** (every value covered, or `default`). For enums and
  sealed types the compiler checks this for you.

### Pattern matching for switch (Java 21)
```java
static String describe(Object o) {
    return switch (o) {
        case null                  -> "null!";              // explicit null handling
        case Integer i when i > 100 -> "big int " + i;      // guard with 'when'
        case Integer i             -> "int " + i;
        case String s              -> "string of length " + s.length();
        case int[] arr             -> "int array of " + arr.length;
        default                    -> "something else";
    };
}
```
Order matters: more specific cases must come first (the compiler rejects a "dominated" case).
Without `case null`, a `null` selector throws `NullPointerException`, as switch always has.

## 3. Loops

```java
for (int i = 0; i < 5; i++) { }                // classic for

for (String name : names) { }                  // enhanced for (arrays & any Iterable)

while (queue.size() > 0) { }                   // condition checked first

do {
    input = read();
} while (!input.equals("quit"));               // body runs at least once

names.forEach(n -> System.out.println(n));     // Iterable.forEach (Java 8)
```

### `break`, `continue` and labels
```java
outer:
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
        if (j == 1) continue;          // skip to next j
        if (i == 2) break outer;       // exit BOTH loops
        System.out.println(i + "," + j);
    }
}
// prints 0,0  0,2  1,0  1,2
```
Java has no `goto` (it's a reserved word but unused). Labeled `break`/`continue` are the
structured alternative.

### Don't modify a collection while iterating with for-each
```java
for (String s : list) {
    if (s.isEmpty()) list.remove(s);   // ConcurrentModificationException
}
list.removeIf(String::isEmpty);         // correct (Java 8)
// or use an explicit Iterator and it.remove()
```

## 4. Arrays

Arrays are **objects**, **fixed size**, hold primitives or references, and know their `length`.

```java
int[] a = new int[5];                 // [0, 0, 0, 0, 0]  (default values)
int[] b = {3, 1, 2};                  // array initializer
String[] names = new String[]{"x", "y"};
int len = b.length;                   // field, not a method (String uses length())

int[][] grid = new int[3][4];         // 3 rows × 4 cols
int[][] jagged = new int[3][];        // rows can have different lengths
jagged[0] = new int[]{1};
jagged[1] = new int[]{1, 2, 3};

a[5] = 1;                             // ArrayIndexOutOfBoundsException
```

### `java.util.Arrays` essentials
```java
int[] nums = {5, 3, 9, 1};
Arrays.sort(nums);                            // [1, 3, 5, 9]  dual-pivot quicksort for primitives
int idx = Arrays.binarySearch(nums, 5);       // 2  (array must be sorted)
System.out.println(Arrays.toString(nums));    // "[1, 3, 5, 9]"  (printing nums directly gives "[I@1b6d3586")
int[] copy = Arrays.copyOf(nums, 6);          // [1, 3, 5, 9, 0, 0]
int[] part = Arrays.copyOfRange(nums, 1, 3);  // [3, 5]
Arrays.fill(a, -1);
boolean same = Arrays.equals(nums, copy);     // content comparison (== compares references)
System.out.println(Arrays.deepToString(grid));// for 2D arrays
int sum = Arrays.stream(nums).sum();          // IntStream

String[] words = {"pear", "fig", "apple"};
Arrays.sort(words, Comparator.comparing(String::length));  // objects: stable TimSort
```

### Array ↔ List
```java
List<String> fixed = Arrays.asList(words);  // fixed-size view backed by the array: set() OK, add() throws
List<String> immutable = List.of(words);    // immutable copy (Java 9), no nulls allowed
List<String> mutable = new ArrayList<>(Arrays.asList(words));

String[] back = mutable.toArray(new String[0]);
String[] back2 = mutable.toArray(String[]::new);   // Java 11

int[] prim = {1, 2, 3};
List<int[]> oops = Arrays.asList(prim);            // a list with ONE element (the array)!
List<Integer> boxed = Arrays.stream(prim).boxed().toList();
```

### Arrays are covariant (a design wart)
```java
Object[] objs = new String[2];
objs[0] = 42;     // compiles, but throws ArrayStoreException at runtime
```
Generics are invariant (`List<Object> l = new ArrayList<String>()` doesn't compile), which is one
reason to prefer collections.

## 5. Gotchas

1. Classic `switch` fall-through when you forget `break`.
2. `switch` on a `null` String or enum throws NPE (unless you have `case null`, Java 21+).
3. `array.length` vs `string.length()` vs `list.size()`.
4. Printing an array prints its type and hash. Use `Arrays.toString`.
5. `Arrays.asList` returns a fixed-size list; `add`/`remove` throw `UnsupportedOperationException`.
6. Integer overflow in `(low + high) / 2` for huge arrays. Use `low + (high - low) / 2` or
   `(low + high) >>> 1`.

## 6. Interview questions

1. **Switch statement vs switch expression?** An expression returns a value, uses `->` without
   fall-through, and must be exhaustive; a statement doesn't need to be.
2. **What does `yield` do?** Returns a value from a block inside a switch expression.
3. **Can you switch on a `String`? A `long`?** `String` yes (Java 7+, uses `hashCode` then
   `equals`). `long`/`float`/`double`/`boolean` no in classic switch (primitive patterns in switch
   are still a preview feature).
4. **`while` vs `do-while`?** `do-while` runs the body at least once.
5. **How do you break out of nested loops?** Labeled `break`, or move the loops into a method and
   `return`.
6. **Are arrays objects?** Yes. They live on the heap, extend `Object`, and have a `length` field
   and `clone()`.
7. **How do you copy an array?** `clone()`, `Arrays.copyOf`, `System.arraycopy`. All are
   **shallow** for object arrays.
8. **`Arrays.asList` vs `List.of`?** `asList` is a fixed-size, writable view that allows `null`;
   `List.of` is immutable and rejects `null`.
9. **What algorithm does `Arrays.sort` use?** Dual-pivot quicksort for primitives (not stable,
   doesn't matter for primitives) and TimSort for objects (stable).
10. **What is `ArrayStoreException`?** Thrown when storing the wrong type into a covariant array.
