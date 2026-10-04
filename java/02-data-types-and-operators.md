# 02 — Data Types, Variables & Operators

## 1. Two kinds of types

| | Primitive types | Reference types |
|---|---|---|
| What the variable holds | The value itself | A reference (pointer) to an object on the heap |
| Examples | `int`, `double`, `boolean`, `char` … | `String`, `Integer`, arrays, any class/interface |
| Default (as a field) | `0`, `0.0`, `false`, `'\u0000'` | `null` |
| Can be `null`? | No | Yes |
| Generics allowed? | No (`List<int>` is illegal) | Yes |
| Compared with `==` | Compares values | Compares **references** (same object?) |

## 2. The 8 primitive types

| Type | Size | Range / notes | Default | Literal |
|---|---|---|---|---|
| `byte` | 8 bit | −128 … 127 | `0` | `(byte) 10` |
| `short` | 16 bit | −32,768 … 32,767 | `0` | `(short) 10` |
| `int` | 32 bit | ≈ ±2.1 billion (−2³¹ … 2³¹−1) | `0` | `10`, `0x1F`, `0b1010`, `1_000_000` |
| `long` | 64 bit | −2⁶³ … 2⁶³−1 | `0L` | `10L` |
| `float` | 32 bit | IEEE 754, ~7 decimal digits | `0.0f` | `3.14f` |
| `double` | 64 bit | IEEE 754, ~15–16 digits | `0.0` | `3.14`, `1e-9` |
| `char` | 16 bit | Unsigned UTF-16 code unit, 0 … 65,535 | `'\u0000'` | `'A'`, `'\n'`, `'\u0041'` |
| `boolean` | JVM-dependent | `true` / `false` (not 0/1!) | `false` | `true` |

```java
int max = Integer.MAX_VALUE;
int overflow = max + 1;           // -2147483648: silent wrap-around, no exception
int safe = Math.addExact(max, 1); // throws ArithmeticException

long big = 3_000_000_000L;        // without L: "integer number too large" compile error
float f = 1.5f;                   // without f: double can't be assigned to float
char c = 'A';
int code = c + 1;                 // 66: char is numeric
char next = (char) (c + 1);       // 'B'
```

### Floating point is not exact
```java
System.out.println(0.1 + 0.2);            // 0.30000000000000004
System.out.println(0.1 + 0.2 == 0.3);     // false

// Use BigDecimal for money (construct from String, not double!)
BigDecimal a = new BigDecimal("0.1");
BigDecimal b = new BigDecimal("0.2");
System.out.println(a.add(b));             // 0.3
new BigDecimal(0.1);                      // 0.1000000000000000055511151231257827... (avoid)

System.out.println(1.0 / 0);              // Infinity  (no exception for floating point)
System.out.println(0.0 / 0);              // NaN
System.out.println(Double.NaN == Double.NaN); // false; use Double.isNaN(x)
System.out.println(1 / 0);                // ArithmeticException: / by zero (integers only)
```

## 3. Variables

```java
int count = 0;                 // local variable: must be assigned before use (no default!)
final int LIMIT = 10;          // can't be reassigned
var names = new ArrayList<String>();   // Java 10: type inferred as ArrayList<String>
```

| Kind | Declared | Lives | Default value? |
|---|---|---|---|
| Local | Inside a method/block | Stack frame | **No**: compile error if read before assignment |
| Instance field | In class, no `static` | Inside the object (heap) | Yes |
| Static field | In class, `static` | One per class (heap, in the `Class` object) | Yes |
| Parameter | Method signature | Stack frame | Set by the caller |

### `var` rules (local variable type inference)
- Only for **local variables** with an initializer, `for` loops, and lambda params (Java 11).
- Not for fields, method params or return types. `var x;` and `var x = null;` don't compile.
- The type is still static: `var s = "hi"; s = 5;` is a compile error.
- `var list = new ArrayList<>();` infers `ArrayList<Object>`, which is usually not what you want.

## 4. Wrapper classes, autoboxing & caching

Each primitive has a wrapper in `java.lang`: `Byte`, `Short`, `Integer`, `Long`, `Float`,
`Double`, `Character`, `Boolean`. You need them for collections/generics, `null`, and utility
methods.

```java
Integer boxed = 42;            // autoboxing:   Integer.valueOf(42)
int unboxed = boxed;           // unboxing:     boxed.intValue()

List<Integer> nums = new ArrayList<>();
nums.add(5);                   // autoboxed

Integer missing = null;
int boom = missing;            // NullPointerException at unboxing!
```

### The Integer cache trap (very common interview question)
```java
Integer a = 127, b = 127;
System.out.println(a == b);        // true:  Integer.valueOf caches -128..127
Integer x = 128, y = 128;
System.out.println(x == y);        // false: two different objects
System.out.println(x.equals(y));   // true:  always compare wrappers with equals()
```
`Byte`, `Short`, `Long` and `Character` (0–127) cache too; `Boolean` has `TRUE`/`FALSE`. `Float`
and `Double` do not cache. `new Integer(5)` is deprecated (for removal); use `Integer.valueOf`.

### Useful wrapper methods
```java
Integer.parseInt("42");          // 42 (int);  NumberFormatException on bad input
Integer.valueOf("42");           // Integer
Integer.toBinaryString(10);      // "1010"
Integer.compare(a, b);           // safe comparison (no overflow, unlike a - b)
Character.isDigit('7');          // true
Character.isLetterOrDigit('_');  // false
Double.compare(0.0, -0.0);       // 1
```

## 5. Type conversion & casting

```
Widening (implicit, safe):  byte → short → int → long → float → double
                                    char ↗
Narrowing (explicit cast, may lose data): the reverse direction
```

```java
int i = 100;
long l = i;                 // widening, automatic
double d = l;               // widening, automatic
int back = (int) 3.99;      // 3: truncates toward zero (doesn't round)
byte b = (byte) 200;        // -56: keeps only the low 8 bits
long precise = 123456789123456789L;
float lossy = precise;      // compiles (widening), but loses precision!

// Integer arithmetic promotes to at least int
byte x = 10, y = 20;
// byte z = x + y;          // compile error: x + y is int
byte z = (byte) (x + y);
x += 5;                     // compiles: compound assignment includes an implicit cast
```

## 6. Operators

| Category | Operators | Notes |
|---|---|---|
| Arithmetic | `+ - * / %` | Integer `/` truncates: `7 / 2 == 3`; `-7 / 2 == -3`; `-7 % 3 == -1` |
| Unary | `++ -- + - !` | `i++` returns old value, `++i` returns new value |
| Assignment | `= += -= *= /= %= &= \|= ^= <<= >>= >>>=` | Compound ops cast implicitly |
| Relational | `== != < > <= >=` | `==` on references compares identity |
| Logical | `&& \|\| !` | Short-circuit |
| Bitwise | `& \| ^ ~` | On `boolean`, `&` and `\|` do **not** short-circuit |
| Shift | `<< >> >>>` | `>>` keeps the sign, `>>>` fills with zeros |
| Ternary | `cond ? a : b` | |
| Type | `instanceof` | Java 16+: `if (o instanceof String s)` |

```java
int i = 5;
int a = i++ + ++i;     // 5 + 7 = 12, i is now 7 (classic trick question)

int x = 0;
x = x++;               // x stays 0! (old value is saved, x incremented, then old value assigned back)

String s = null;
if (s != null && s.length() > 0) { }   // safe: && short-circuits
// if (s != null & s.length() > 0) { } // NPE: & evaluates both sides

System.out.println(-8 >> 1);    // -4
System.out.println(-8 >>> 28);  // 15 (zero-fill)
System.out.println(5 & 3);      // 1   (0101 & 0011)
System.out.println(5 ^ 3);      // 6   (XOR: swap without temp, check odd occurrences)
System.out.println((n & 1) == 0 ? "even" : "odd");   // parity check (n = some int)
System.out.println((n & (n - 1)) == 0);               // power of two (for n > 0)
```

String concatenation is evaluated left to right:
```java
System.out.println(1 + 2 + "3");   // "33"
System.out.println("1" + 2 + 3);   // "123"
System.out.println('a' + 'b');     // 195 (char + char = int)
System.out.println("" + 'a' + 'b');// "ab"
```

## 7. Java is always pass-by-value

Java passes **a copy of the value** into a method. For objects, that value *is a reference*, so the
method can mutate the object but can't make the caller's variable point somewhere else.

```java
static void reassign(StringBuilder sb) { sb = new StringBuilder("new"); }   // affects local copy only
static void mutate(StringBuilder sb)   { sb.append(" world"); }             // affects the shared object
static void bump(int n)                { n++; }                             // affects local copy only

StringBuilder sb = new StringBuilder("hello");
reassign(sb);   System.out.println(sb);   // hello
mutate(sb);     System.out.println(sb);   // hello world
int n = 1; bump(n); System.out.println(n);// 1
```

The classic `swap(a, b)` therefore can't be written in Java for object references.

## 8. Gotchas

1. Integer overflow is silent. Use `Math.addExact`/`multiplyExact` or `long`/`BigInteger`.
2. Comparing wrappers with `==` works by accident for small values only.
3. Unboxing `null` throws NPE, e.g. `int x = map.get("missing");`.
4. `Arrays.asList(1, 2, 3).contains(1L)` is `false` (`Long` vs `Integer`).
5. `list.remove(1)` on a `List<Integer>` removes **index 1**, not the value 1. Use
   `list.remove(Integer.valueOf(1))`.
6. Ternary with mixed boxed types can unbox unexpectedly:
   `Integer r = flag ? 1 : nullInteger;` throws NPE when `flag` is false.
7. `char` arithmetic gives an `int`.

## 9. Interview questions

1. **How many primitive types are there?** Eight: `byte`, `short`, `int`, `long`, `float`,
   `double`, `char`, `boolean`.
2. **Why isn't Java 100% object-oriented?** Primitives aren't objects (and static members aren't
   tied to an object).
3. **What is autoboxing?** Automatic conversion between primitives and wrappers, done by the
   compiler via `valueOf()` and `xxxValue()`.
4. **Why does `Integer a = 128, b = 128; a == b` give `false`?** Only −128..127 is cached by
   `Integer.valueOf`; outside that range each boxing creates a new object, and `==` compares
   references.
5. **Is Java pass-by-value or pass-by-reference?** Always pass-by-value. Object references are
   passed by value.
6. **Widening vs narrowing?** Widening goes to a larger type (implicit); narrowing needs an
   explicit cast and may lose data.
7. **Default value of local variables?** None. They must be definitely assigned before use.
8. **Why use `BigDecimal` for money?** `double` is binary floating point and can't represent most
   decimal fractions exactly.
9. **`>>` vs `>>>`?** Arithmetic (sign-extending) vs logical (zero-filling) right shift.
10. **What does `var` do?** Infers a local variable's static type at compile time. It is not
    dynamic typing.
11. **What's the size of `boolean`?** Not defined by the spec; typically 1 byte in arrays and an
    `int` slot on the operand stack.
12. **`&` vs `&&`?** `&&` short-circuits; `&` always evaluates both sides (and is also bitwise AND
    on integers).
