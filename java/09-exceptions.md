# 09 — Exception Handling

## 1. The hierarchy

```
                    Throwable
                   /         \
              Error           Exception
   (JVM problems; don't       /        \
    catch)                   /          RuntimeException  ← UNCHECKED
  • OutOfMemoryError   IOException        • NullPointerException
  • StackOverflowError SQLException       • IllegalArgumentException
  • NoClassDefFoundError InterruptedException • IllegalStateException
  • AssertionError     ClassNotFoundException • ArithmeticException
                       ...                    • IndexOutOfBoundsException
                       ↑ CHECKED              • ClassCastException
                                              • NumberFormatException (extends IAE)
                                              • ConcurrentModificationException
                                              • UnsupportedOperationException
```

| | Checked exceptions | Unchecked exceptions |
|---|---|---|
| Classes | `Exception` and subclasses, **except** `RuntimeException` | `RuntimeException`, `Error` and their subclasses |
| Compiler | **Forces** you to catch or declare with `throws` | No enforcement |
| Meaning | Recoverable conditions outside the program's control (file missing, network down) | Programming bugs (null, bad index, bad argument) or fatal JVM errors |
| Examples | `IOException`, `SQLException`, `InterruptedException` | `NullPointerException`, `IllegalArgumentException` |

**Error vs Exception:** `Error`s are serious problems the application normally can't recover from
(out of memory, stack overflow). `Exception`s are conditions the application may handle.

## 2. try / catch / finally

```java
public int parse(String s) {
    try {
        return Integer.parseInt(s);
    } catch (NumberFormatException e) {
        log.warn("bad number: {}", s, e);
        return 0;
    } finally {
        System.out.println("always runs");   // runs after return value is computed
    }
}
```

### Multi-catch (Java 7)
```java
try {
    riskyIo();
    riskySql();
} catch (IOException | SQLException e) {     // types must not be subclasses of each other
    throw new DataAccessException("failed", e);
}
```

### Catch order: specific first
```java
try { ... }
catch (FileNotFoundException e) { ... }   // subclass first
catch (IOException e) { ... }             // then superclass
// reversed order → compile error: "exception already caught"
```

### `finally` rules and traps
- `finally` runs whether the try completes normally, returns, or throws.
- It does **not** run if the JVM exits (`System.exit`), the process is killed, or the thread dies
  abruptly.
- A `return` in `finally` **overrides** the try's return and **swallows** any exception. Never do
  this.

```java
static int trap() {
    try {
        return 1;
    } finally {
        return 2;      // method returns 2; an exception from try would also be silently lost
    }
}

static int trap2() {
    int x = 1;
    try {
        return x;      // value 1 is saved here
    } finally {
        x = 99;        // modifies the local, but the saved return value is still 1
    }
}   // returns 1
```

## 3. try-with-resources (Java 7)

Automatically closes any `AutoCloseable` resource, in **reverse order** of declaration, even when
an exception is thrown.

```java
try (var reader = Files.newBufferedReader(Path.of("in.txt"));
     var writer = Files.newBufferedWriter(Path.of("out.txt"))) {
    String line;
    while ((line = reader.readLine()) != null) {
        writer.write(line.toUpperCase());
        writer.newLine();
    }
}   // writer.close(), then reader.close()

// Java 9: effectively final resources declared outside
BufferedReader br = Files.newBufferedReader(path);
try (br) { ... }
```

**Suppressed exceptions:** if the body throws and `close()` also throws, the body's exception is
thrown and the close exception is attached via `e.getSuppressed()`. With old-style `finally`
closing, the close exception would *replace* the original one.

Your own resource:
```java
public class Timer implements AutoCloseable {
    private final long start = System.nanoTime();
    @Override public void close() {            // can narrow: no 'throws Exception'
        System.out.println("took " + (System.nanoTime() - start) / 1_000_000 + " ms");
    }
}
try (var t = new Timer()) { doWork(); }
```

## 4. `throw` vs `throws`

```java
public void withdraw(double amount) throws InsufficientFundsException {   // declares
    if (amount > balance) {
        throw new InsufficientFundsException(balance, amount);             // throws an instance
    }
    balance -= amount;
}
```

| `throw` | `throws` |
|---|---|
| A statement that throws one exception object | A clause in the method signature |
| Inside the method body | After the parameter list |
| `throw new X()` | `throws X, Y` (can list several) |

## 5. Custom exceptions

```java
// Checked: callers must handle it
public class InsufficientFundsException extends Exception {
    private final double balance, requested;

    public InsufficientFundsException(double balance, double requested) {
        super("Requested %.2f but balance is %.2f".formatted(requested, balance));
        this.balance = balance;
        this.requested = requested;
    }

    public double shortfall() { return requested - balance; }
}

// Unchecked: most modern code (Spring etc.) prefers these
public class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(String id) { super("Order not found: " + id); }
    public OrderNotFoundException(String id, Throwable cause) { super("Order not found: " + id, cause); }
}
```

### Exception chaining: keep the root cause
```java
try {
    repository.save(order);
} catch (SQLException e) {
    throw new OrderPersistenceException("Could not save order " + order.id(), e);   // pass 'e'!
}
```
Losing the cause (`throw new X(e.getMessage())`) throws away the original stack trace.

## 6. Exceptions and overriding

An overriding method:
- may throw the same checked exceptions, narrower ones, fewer ones, or none;
- may **not** throw new or broader checked exceptions;
- may throw any unchecked exception.

```java
class Parent { void read() throws IOException { } }
class OkChild extends Parent { @Override void read() throws FileNotFoundException { } }   // narrower ✅
class BadChild extends Parent { @Override void read() throws Exception { } }             // broader ❌
```

## 7. Common runtime exceptions and what causes them

| Exception | Typical cause |
|---|---|
| `NullPointerException` | Calling a method/field on `null`, unboxing `null`. Java 14+ "helpful NPE" messages say which variable was null. |
| `ArrayIndexOutOfBoundsException` | `arr[arr.length]` |
| `ClassCastException` | Bad downcast |
| `NumberFormatException` | `Integer.parseInt("abc")` |
| `IllegalArgumentException` | Method received an invalid argument (throw it yourself to validate) |
| `IllegalStateException` | Method called at the wrong time (e.g. iterator `remove()` twice) |
| `ConcurrentModificationException` | Modifying a collection while iterating it |
| `UnsupportedOperationException` | Modifying an immutable collection (`List.of(1).add(2)`) |
| `ArithmeticException` | Integer division by zero |
| `StackOverflowError` | Infinite or too-deep recursion |
| `OutOfMemoryError` | Heap (or metaspace, or native) exhausted |

## 8. Best practices

1. **Catch specific exceptions**, not `Exception` or `Throwable` (unless at a top-level boundary
   that logs and reports).
2. **Never swallow** exceptions: `catch (Exception e) {}` hides bugs. At minimum log, rethrow or
   wrap.
3. **Don't use exceptions for control flow.** They are slow (stack trace capture) and obscure the
   logic.
4. **Throw early, catch late**: validate inputs at the top of a method; handle exceptions where you
   can do something meaningful.
5. Use **try-with-resources** for anything closeable.
6. **Preserve the cause** when wrapping.
7. Don't log *and* rethrow the same exception at every layer (duplicate log noise).
8. Restore the interrupt flag when catching `InterruptedException`:
   ```java
   catch (InterruptedException e) {
       Thread.currentThread().interrupt();
       throw new RuntimeException(e);
   }
   ```
9. Prefer standard exceptions (`IllegalArgumentException`, `IllegalStateException`,
   `UnsupportedOperationException`) before inventing new ones.
10. Use `Objects.requireNonNull(arg, "arg must not be null")` for null checks.

## 9. Checked exceptions and lambdas

Functional interfaces like `Function` don't declare checked exceptions, so this doesn't compile:
```java
paths.stream().map(p -> Files.readString(p))   // ❌ unhandled IOException
```
Options: wrap inside the lambda, or use a helper.
```java
paths.stream().map(p -> {
    try { return Files.readString(p); }
    catch (IOException e) { throw new UncheckedIOException(e); }
}).toList();
```

## 10. Gotchas

1. `return` in `finally`.
2. Catching `Exception` also catches `RuntimeException`s you didn't mean to handle (and
   `InterruptedException`, losing the interrupt).
3. Catching `Throwable`/`Error`: you can't meaningfully recover from `OutOfMemoryError`.
4. `e.printStackTrace()` in production code. Use a logger.
5. Losing the root cause when wrapping.

## 11. Interview questions

1. **Checked vs unchecked exceptions?** Compiler-enforced, recoverable external conditions vs
   programming errors that don't need declaring.
2. **Error vs Exception?** Fatal JVM-level problems vs conditions the application can handle.
3. **`throw` vs `throws`?** A statement that throws vs a declaration in the signature.
4. **`final` vs `finally` vs `finalize`?** Modifier vs always-run block vs deprecated GC hook.
5. **Does `finally` always execute?** Except on `System.exit`, JVM crash/kill, or an infinite loop
   or deadlock in try.
6. **What if both try and finally return?** The finally value wins and any exception is lost.
7. **What is try-with-resources?** Auto-closes `AutoCloseable` resources in reverse order and
   records suppressed exceptions.
8. **What are suppressed exceptions?** Exceptions from `close()` attached to the primary exception.
9. **Can we have try without catch?** Yes: `try`-`finally`, or try-with-resources alone.
10. **How do you create a custom exception?** Extend `Exception` (checked) or `RuntimeException`
    (unchecked) and provide message and cause constructors.
11. **Overriding rules for exceptions?** No new or broader checked exceptions.
12. **Is it good practice to catch `Exception`?** Only at boundaries (top-level handlers); otherwise
    catch specific types.
13. **What is exception chaining?** Wrapping a low-level exception as the cause of a higher-level
    one (`new X(msg, cause)`).
14. **`ClassNotFoundException` vs `NoClassDefFoundError`?** See chapter 01.
