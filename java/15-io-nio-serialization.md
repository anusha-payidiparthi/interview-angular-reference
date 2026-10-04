# 15 — I/O, NIO & Serialization

## 1. The two families of classic I/O (`java.io`)

| | Byte streams | Character streams |
|---|---|---|
| Base classes | `InputStream` / `OutputStream` | `Reader` / `Writer` |
| Unit | 8-bit bytes | 16-bit chars (decoded with a charset) |
| Use for | Binary data: images, zip, serialized objects | Text |
| File classes | `FileInputStream`, `FileOutputStream` | `FileReader`, `FileWriter` |
| Buffered | `BufferedInputStream`, `BufferedOutputStream` | `BufferedReader` (`readLine`), `BufferedWriter` |
| Bridge | `InputStreamReader` / `OutputStreamWriter` convert bytes ↔ chars with a `Charset` | |

**Decorator pattern:** streams wrap each other to add features.
```java
try (BufferedReader in = new BufferedReader(
         new InputStreamReader(new FileInputStream("data.txt"), StandardCharsets.UTF_8))) {
    String line;
    while ((line = in.readLine()) != null) {
        System.out.println(line);
    }
}
```
Always specify the charset (Java 18+ defaults to UTF-8, JEP 400; earlier versions used the platform
default, a classic source of bugs).

**Why buffer?** Each unbuffered `read()` can be a system call. Buffering reads big chunks (8 KB by
default) into memory and serves from there.

```java
// Copy a binary file the classic way
try (InputStream in = new BufferedInputStream(new FileInputStream("a.png"));
     OutputStream out = new BufferedOutputStream(new FileOutputStream("b.png"))) {
    in.transferTo(out);           // Java 9; replaces the manual byte[] loop
}
```

Other useful classes: `PrintWriter` (`println`, `printf`), `Scanner` (parse tokens/numbers from
input), `DataInputStream` (read primitives), `ByteArrayInputStream`/`ByteArrayOutputStream`
(in-memory streams), `Console`.

## 2. NIO.2: `Path` & `Files` (Java 7+): use these for file work

```java
Path p = Path.of("data", "users.csv");           // Java 11 (Paths.get before)
p.getFileName(); p.getParent(); p.toAbsolutePath(); p.resolve("child"); p.normalize();

// Small files: one-liners
String text = Files.readString(p);               // Java 11, UTF-8
List<String> lines = Files.readAllLines(p);
byte[] bytes = Files.readAllBytes(p);
Files.writeString(p, "hello\n", StandardOpenOption.CREATE, StandardOpenOption.APPEND);
Files.write(p, lines);

// Large files: stream lazily (close it!)
try (Stream<String> s = Files.lines(p)) {
    long errors = s.filter(l -> l.contains("ERROR")).count();
}
try (BufferedReader r = Files.newBufferedReader(p)) { ... }

// File system operations
Files.exists(p); Files.isDirectory(p); Files.size(p);
Files.createDirectories(Path.of("out/reports"));
Files.copy(src, dst, StandardCopyOption.REPLACE_EXISTING);
Files.move(src, dst, StandardCopyOption.ATOMIC_MOVE);
Files.delete(p); Files.deleteIfExists(p);
Path tmp = Files.createTempFile("pre", ".tmp");

// Walking directories
try (Stream<Path> tree = Files.walk(Path.of("src"))) {
    tree.filter(f -> f.toString().endsWith(".java")).forEach(System.out::println);
}
try (Stream<Path> list = Files.list(dir)) { ... }   // one level only
```

`java.io.File` is the legacy API (poor error reporting: `delete()` just returns `false`, limited
metadata). Convert with `file.toPath()` / `path.toFile()`.

## 3. NIO: buffers, channels, selectors (Java 1.4)

| Classic I/O | NIO |
|---|---|
| Stream-oriented (one byte at a time, one direction) | Buffer-oriented (read into a `ByteBuffer`, move back and forth) |
| Blocking | Blocking **or non-blocking** |
| One thread per connection | One thread can manage many channels with a **`Selector`** |

```java
try (FileChannel ch = FileChannel.open(path, StandardOpenOption.READ)) {
    ByteBuffer buf = ByteBuffer.allocate(1024);
    while (ch.read(buf) > 0) {
        buf.flip();                          // switch from writing-into to reading-from
        while (buf.hasRemaining()) System.out.print((char) buf.get());
        buf.clear();                         // ready for the next read
    }
}
```
Buffer state: `capacity` ≥ `limit` ≥ `position`. `flip()` sets limit = position and position = 0.

Non-blocking servers (`ServerSocketChannel` + `Selector`) are what Netty and frameworks built on it
(Spring WebFlux, Vert.x) use. Memory-mapped files (`FileChannel.map`) give very fast access to large
files. With virtual threads, plain blocking I/O scales again for most apps.

## 4. Serialization

**Serialization** turns an object graph into bytes; **deserialization** rebuilds it.

```java
public class User implements Serializable {             // marker interface
    @Serial
    private static final long serialVersionUID = 1L;    // version id; declare it explicitly!

    private String name;
    private transient String password;                   // NOT serialized → null after deserialization
    private static int count;                            // statics are never serialized (class state)

    public User(String name, String password) { this.name = name; this.password = password; }
}

// Write
try (var out = new ObjectOutputStream(new FileOutputStream("user.ser"))) {
    out.writeObject(new User("ana", "secret"));
}
// Read
try (var in = new ObjectInputStream(new FileInputStream("user.ser"))) {
    User u = (User) in.readObject();                     // password == null
}
```

Rules:
- Every non-transient field's type must also be `Serializable`, or you get
  `NotSerializableException`.
- On deserialization, **constructors of serializable classes are not called**. The constructor of
  the first *non-serializable* superclass (it needs a no-arg constructor) is called.
- **`serialVersionUID`**: if you don't declare one, it's computed from the class structure. Any
  change (adding a method) changes it, and old data fails with `InvalidClassException`.
- Customize with `private void writeObject(ObjectOutputStream)` / `readObject(ObjectInputStream)`,
  `readResolve()` (e.g. to keep singletons single), `writeReplace()`, or the `Externalizable`
  interface (full manual control; requires a public no-arg constructor).
- Records serialize through their canonical constructor, which is safer.

```java
// Keep a Serializable singleton a singleton
@Serial
private Object readResolve() { return INSTANCE; }
```

### Security warning
Java native deserialization of **untrusted data** is a well-known remote-code-execution vector
("gadget chains"). Prefer JSON (Jackson), Protobuf or Avro for data exchange. If you must
deserialize, use **serialization filters** (`ObjectInputFilter`, Java 9+) to allow-list classes.

### `Serializable` vs `Externalizable`

| | `Serializable` | `Externalizable` |
|---|---|---|
| Methods | None (marker) | `writeExternal`, `readExternal` |
| Control | Automatic (customizable) | Fully manual |
| Constructor on read | Not called | Public no-arg constructor called |
| Performance | Slower (reflection, metadata) | Can be faster |

## 5. Reading console input

```java
Scanner sc = new Scanner(System.in);
int n = sc.nextInt();
sc.nextLine();                // consume the leftover newline (classic bug!)
String name = sc.nextLine();

BufferedReader br = new BufferedReader(new InputStreamReader(System.in));   // faster for big input
String line = br.readLine();

String input = IO.readln("Name? ");                 // Java 25 java.lang.IO
```

## 6. Gotchas

1. Not closing streams (use try-with-resources).
2. Not specifying a charset (pre-Java 18).
3. `Files.readAllLines` on a multi-GB file → `OutOfMemoryError`; use `Files.lines` or a reader.
4. Forgetting `flush()` on buffered writers when not closing them.
5. `Scanner.nextInt()` followed by `nextLine()` returns an empty string.
6. Missing `serialVersionUID` breaks compatibility after harmless code changes.
7. `Files.lines`/`walk`/`list` streams hold file handles until closed.

## 7. Interview questions

1. **Byte streams vs character streams?** Raw bytes for binary data vs chars decoded with a charset
   for text.
2. **Why use buffered streams?** Fewer system calls; reading and writing in chunks.
3. **What pattern does `java.io` use?** Decorator (wrapping streams to add behavior).
4. **I/O vs NIO?** Stream-oriented and blocking vs buffer/channel-oriented, optionally non-blocking
   with selectors.
5. **`File` vs `Path`/`Files`?** Legacy vs NIO.2: better errors, symlinks, attributes, streams.
6. **What is serialization? Why `serialVersionUID`?** Converting objects to bytes. The UID is a
   version check between the serialized data and the class.
7. **What does `transient` do?** Excludes a field from serialization (it gets its default value
   back).
8. **Are static fields serialized?** No; they belong to the class.
9. **What happens if a field's type isn't serializable?** `NotSerializableException` (unless the
   field is `transient`).
10. **Is the constructor called during deserialization?** Not for serializable classes; only the
    no-arg constructor of the first non-serializable superclass.
11. **`Serializable` vs `Externalizable`?** See the table in section 4.
12. **How do you keep a singleton a singleton during deserialization?** Implement `readResolve()`,
    or use an enum singleton.
13. **Why is Java deserialization a security risk?** Untrusted byte streams can instantiate gadget
    classes that execute code. Use filters or avoid native serialization.
