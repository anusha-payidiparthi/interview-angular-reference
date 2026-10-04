# 04 — Strings

`String` is the most-asked-about class in Java interviews. Know immutability, the pool, `==` vs
`equals`, and `StringBuilder` cold.

## 1. Creating strings & the String Pool

```java
String a = "java";               // literal → goes in the String Constant Pool (in the heap)
String b = "java";               // reuses the same pooled object
String c = new String("java");   // forces a NEW object on the heap (pool still has "java")
String d = c.intern();           // returns the pooled instance

System.out.println(a == b);       // true:  same pooled object
System.out.println(a == c);       // false: different objects
System.out.println(a.equals(c));  // true:  same characters
System.out.println(a == d);       // true
```

```
Heap
├── String Pool:  "java"  ◀── a, b, d
└── Other:        String("java") ◀── c
```

**How many objects does `new String("java")` create?** Up to two: the literal `"java"` in the pool
(if not already there, created when the class is loaded) and the new heap object.

### Compile-time constants are pooled too
```java
String s1 = "ja" + "va";          // folded at compile time → "java" (pooled)
System.out.println(s1 == a);      // true

String part = "ja";
String s2 = part + "va";          // computed at runtime → new object
System.out.println(s2 == a);      // false

final String fPart = "ja";
String s3 = fPart + "va";         // final + literal → compile-time constant
System.out.println(s3 == a);      // true
```

**Rule:** compare strings with `equals()` (or `equalsIgnoreCase()`), never `==`. Write
`"literal".equals(var)` or `Objects.equals(x, y)` to avoid NPE.

## 2. Why is `String` immutable?

Once created, a `String`'s contents never change. Methods like `toUpperCase()` return a *new*
string.

```java
String s = "hello";
s.toUpperCase();                 // result discarded!
System.out.println(s);           // hello
s = s.toUpperCase();             // reassign the variable to the new object
```

Reasons (classic interview answer):
1. **String pool**: sharing literals is only safe if nobody can modify them.
2. **Security**: class names, file paths, URLs, DB credentials passed as strings can't be altered
   after validation.
3. **Thread safety**: immutable objects can be shared across threads without synchronization.
4. **Hash caching**: `hashCode()` is computed once and cached, making strings fast `HashMap` keys.
5. **Class loading** relies on string names not changing.

How it's enforced: the class is `final` (no subclass can add mutability), the internal array is
`private final`, and no method exposes or modifies it.

Since Java 9, **compact strings** store Latin-1 text as a `byte[]` (1 byte per char) instead of
`char[]`, which roughly halves memory for ASCII-heavy apps.

## 3. Essential `String` methods

```java
String s = "  Hello, World  ";

s.length();                       // 16
s.charAt(2);                      // 'H'
s.trim();                         // "Hello, World"   (removes ASCII whitespace ≤ ' ')
s.strip();                        // "Hello, World"   (Java 11, Unicode-aware; also stripLeading/Trailing)
s.isEmpty();                      // false (length == 0)
"   ".isBlank();                  // true  (Java 11: empty or only whitespace)
s.toLowerCase(); s.toUpperCase();
s.contains("World");              // true
s.indexOf('o'); s.lastIndexOf('o');
s.substring(2, 7);                // "Hello"  (begin inclusive, end exclusive)
s.replace('l', 'L');              // replaces all occurrences (char or CharSequence)
s.replaceAll("\\s+", " ");        // regex
s.startsWith("  He"); s.endsWith("  ");
"a,b,,c".split(",");              // ["a", "b", "", "c"]
"a,b,,c,,".split(",");            // ["a", "b", "", "c"]  trailing empty strings are dropped!
"a,b,,c,,".split(",", -1);        // ["a", "b", "", "c", "", ""]
String.join("-", "a", "b", "c");  // "a-b-c"
String.join(", ", List.of("x", "y"));
"ab".repeat(3);                   // "ababab" (Java 11)
"Line1\nLine2".lines().count();   // 2 (Java 11)
"abc".compareTo("abd");           // negative (lexicographic, used for sorting)
"Java".equalsIgnoreCase("JAVA");  // true
String.valueOf(42);               // "42"  (safe for null objects: "null")
"Hi %s, you are %d".formatted("Ana", 30);   // Java 15 (same as String.format)
"hello".chars().filter(ch -> ch == 'l').count();  // 2
char[] arr = "hello".toCharArray();
new String(arr);
```

## 4. `String` vs `StringBuilder` vs `StringBuffer`

| | `String` | `StringBuilder` | `StringBuffer` |
|---|---|---|---|
| Mutable? | No | Yes | Yes |
| Thread-safe? | Yes (immutable) | **No** | Yes (`synchronized` methods) |
| Speed | Slow for repeated modification | Fastest | Slower than `StringBuilder` |
| Since | 1.0 | 1.5 | 1.0 |
| Use when | Values that don't change | Building strings in a single thread (99% of cases) | Legacy code; rarely needed |

```java
// BAD: O(n²); each += creates a new String and copies everything
String result = "";
for (int i = 0; i < 10_000; i++) result += i;

// GOOD: amortized O(n)
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 10_000; i++) sb.append(i);
String out = sb.toString();

// StringBuilder API
sb.append("x").append(1).append('c');
sb.insert(0, "start-");
sb.reverse();                   // common in "reverse a string" questions
sb.deleteCharAt(sb.length() - 1);
sb.setCharAt(0, 'S');
sb.replace(0, 5, "HELLO");
sb.setLength(0);                // clear and reuse
```

Note: a single expression like `"a" + x + "b"` is fine. The compiler optimizes it (since Java 9
via `invokedynamic`/`StringConcatFactory`). The problem is concatenation *inside loops*.

`StringBuilder` doesn't override `equals()`. Compare with `sb1.toString().equals(sb2.toString())`
or `sb1.compareTo(sb2) == 0` (Java 11).

## 5. Text blocks (Java 15)

```java
String json = """
    {
      "name": "Ana",
      "age": 30
    }
    """;                      // indentation is stripped relative to the closing """
String sql = """
    SELECT id, name \
    FROM users \
    WHERE active = true""";   // \ joins lines, no trailing newline here
```

## 6. Common string interview snippets

```java
// Reverse
new StringBuilder("hello").reverse().toString();

// Palindrome (ignoring case/non-letters)
String clean = s.replaceAll("[^A-Za-z0-9]", "").toLowerCase();
boolean isPal = new StringBuilder(clean).reverse().toString().equals(clean);

// Character frequency
Map<Character, Long> freq = s.chars()
    .mapToObj(ch -> (char) ch)
    .collect(Collectors.groupingBy(Function.identity(), LinkedHashMap::new, Collectors.counting()));

// Anagram check
char[] x = a.toCharArray(), y = b.toCharArray();
Arrays.sort(x); Arrays.sort(y);
boolean anagram = Arrays.equals(x, y);

// First non-repeating character
for (char ch : s.toCharArray())
    if (s.indexOf(ch) == s.lastIndexOf(ch)) { System.out.println(ch); break; }

// Count words
int words = s.isBlank() ? 0 : s.trim().split("\\s+").length;
```
More in [20 — Coding problems](./20-coding-problems.md).

## 7. Gotchas

1. `==` on strings: works for literals by accident, fails for runtime values.
2. Forgetting to reassign: `s.trim();` does nothing useful.
3. `split` takes a **regex**: `"a.b".split(".")` returns an empty array. Use `split("\\.")` or
   `Pattern.quote(".")`.
4. `substring` on very old JVMs (≤ Java 6) shared the parent array and could leak memory; since
   Java 7u6 it copies.
5. Passwords: prefer `char[]` over `String`. You can zero out an array after use, but a `String`
   stays in memory (and maybe the pool) until GC, and can show up in logs/heap dumps.
6. `toUpperCase()` without a `Locale` can surprise you (Turkish dotless i). Use
   `toUpperCase(Locale.ROOT)` for identifiers.

## 8. Interview questions

1. **Why is `String` immutable?** Pool safety, security, thread safety, cached hash code, class
   loading.
2. **What is the String Constant Pool?** A heap area where string literals are interned and shared.
3. **`==` vs `equals()` for strings?** Reference identity vs character content.
4. **How many objects does `String s = new String("abc")` create?** Up to two (pool literal plus
   heap object).
5. **What does `intern()` do?** Returns the canonical pooled instance, adding it to the pool if
   needed.
6. **`String` vs `StringBuilder` vs `StringBuffer`?** See the table in section 4.
7. **Why is `String` a good `HashMap` key?** Immutable (hash can't change after insertion) and the
   hash code is cached.
8. **Is `String` thread-safe?** Yes, because it's immutable.
9. **Why `char[]` for passwords?** It can be wiped explicitly; strings linger and may be pooled or
   logged.
10. **Can you make a class like `String` (immutable)?** Yes. See chapter 08: `final` class,
    `private final` fields, no setters, defensive copies.
11. **`trim()` vs `strip()`?** `strip()` (Java 11) understands Unicode whitespace; `trim()` only
    removes characters ≤ `'\u0020'`.
12. **What are compact strings?** Java 9 optimization: Latin-1 strings use 1 byte per character.
