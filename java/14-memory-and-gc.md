# 14 — Memory Management & Garbage Collection

## 1. Stack vs heap

```java
public class Demo {
    public static void main(String[] args) {
        int x = 10;                         // primitive local → stack (main's frame)
        Person p = new Person("Ana");       // reference 'p' → stack; Person object → heap
        greet(p);                           // new frame pushed for greet
    }
    static void greet(Person who) {         // 'who' (a copy of the reference) → greet's frame
        String msg = "Hi " + who.name();    // 'msg' ref → stack; the String object → heap
    }                                       // frame popped; the String becomes unreachable
}
```

```
 Thread stack (main)             Heap (shared)
 ┌─────────────────────┐        ┌───────────────────────────┐
 │ greet frame:        │        │  Person{name ─┐}  ◀────┐  │
 │   who ──────────────┼───────▶│               ▼        │  │
 │   msg ───────────┐  │        │          "Ana"         │  │
 ├──────────────────┼──┤        │  "Hi Ana" ◀─┘          │  │
 │ main frame:      │  │        └────────────────────────┼──┘
 │   x = 10         │  │                                 │
 │   p ─────────────┼──┼─────────────────────────────────┘
 └─────────────────────┘
```

| | Stack | Heap |
|---|---|---|
| Stores | Frames: local primitives, references, partial results | All objects and arrays |
| Scope | Per thread | Shared by all threads |
| Lifetime | Frame popped when the method returns | Until garbage collected |
| Size | Small (`-Xss`, e.g. 512 KB–1 MB per platform thread) | Large (`-Xms`/`-Xmx`) |
| Speed | Very fast (pointer bump) | Fast allocation (TLABs), GC cost |
| Error when exhausted | `StackOverflowError` | `OutOfMemoryError: Java heap space` |
| Thread-safety | Inherently thread-safe | Needs synchronization |

The JIT's **escape analysis** can avoid heap allocation for objects that never escape a method
(scalar replacement), so "objects always live on the heap" is the language model, not always the
physical reality.

## 2. JVM memory areas (recap from chapter 01)

| Area | Contents | Error |
|---|---|---|
| Heap | Objects, arrays, string pool, static fields | `OutOfMemoryError: Java heap space` |
| Metaspace (native) | Class metadata, method bytecode, runtime constant pool | `OutOfMemoryError: Metaspace` |
| Thread stacks | Frames | `StackOverflowError` / `OutOfMemoryError: unable to create native thread` |
| Code cache | JIT-compiled native code | JIT stops compiling (warning) |
| Direct memory | `ByteBuffer.allocateDirect`, NIO | `OutOfMemoryError: Direct buffer memory` |

## 3. Generational heap

Most objects die young (the **weak generational hypothesis**). So the heap is split:

```
┌──────────────────── Young Generation ────────────────────┬──────── Old Generation ────────┐
│   Eden          │  Survivor 0 (S0)  │  Survivor 1 (S1)   │  (Tenured)                      │
│ new objects     │  survivors ping-pong between S0 and S1 │  long-lived objects             │
└──────────────────────────────────────────────────────────┴─────────────────────────────────┘
```

1. New objects are allocated in **Eden** (very fast: each thread has a TLAB and bumps a pointer).
2. When Eden fills, a **minor GC** copies live objects to an empty survivor space; everything left
   in Eden is garbage, reclaimed in one go.
3. Survivors that live through enough minor GCs (age threshold, up to 15) are **promoted** to the
   **old generation**.
4. When the old gen fills → **major / full GC** (more expensive).

(G1 and ZGC still use generations logically but organize the heap into regions.)

## 4. How the GC decides what's garbage: reachability

An object is **eligible for GC** when no chain of references from a **GC root** reaches it.
Reference counting is **not** used (it can't handle cycles); two objects pointing to each other but
unreachable from roots are collected.

**GC roots:** local variables and parameters on thread stacks, active threads, static fields of
loaded classes, JNI references, monitors held by `synchronized`.

Ways an object becomes eligible:
```java
Person p = new Person();  p = null;              // 1. reference set to null
Person a = new Person();  a = new Person();      // 2. reference reassigned (first object orphaned)
void m() { Person local = new Person(); }        // 3. goes out of scope when the method returns
// 4. island of isolation: a ↔ b reference each other, nothing else references either
```

## 5. GC algorithms (phases)

- **Mark**: traverse from roots and mark reachable objects.
- **Sweep**: free unmarked objects (leaves fragmentation).
- **Compact**: move live objects together (no fragmentation; requires updating references).
- **Copy**: copy live objects to another space (used for the young gen; cost is proportional to
  *live* objects, not garbage).

**Stop-the-world (STW) pause:** all application threads are halted during some GC phases. Modern
collectors do most work **concurrently** to keep pauses short.

## 6. Garbage collectors in HotSpot

| Collector | Flag | Pauses | Best for |
|---|---|---|---|
| **Serial** | `-XX:+UseSerialGC` | STW, single thread | Tiny heaps, single-CPU containers, CLI tools |
| **Parallel** (Throughput) | `-XX:+UseParallelGC` | STW, multi-threaded | Batch jobs where throughput matters more than latency |
| **G1** (Garbage-First) | `-XX:+UseG1GC` | Short, mostly predictable (target `-XX:MaxGCPauseMillis=200`) | **Default since Java 9.** General-purpose server apps |
| **ZGC** | `-XX:+UseZGC` | Sub-millisecond, independent of heap size (up to TBs) | Low-latency services, huge heaps. Generational by default since Java 23 |
| **Shenandoah** | `-XX:+UseShenandoahGC` | Very short (concurrent compaction) | Low latency (OpenJDK builds; generational mode final in Java 25) |
| **Epsilon** | `-XX:+UseEpsilonGC` | No GC at all | Performance testing, very short-lived jobs |
| ~~CMS~~ | Removed in Java 14 | | Replaced by G1/ZGC |

**G1 in brief:** the heap is split into equal regions (1–32 MB), each tagged Eden/Survivor/Old/
Humongous. G1 tracks how much garbage each region holds and collects the regions with the **most
garbage first** (hence the name) within a pause-time target, using concurrent marking and
incremental compaction ("mixed collections").

## 7. Reference types (`java.lang.ref`)

| Type | Collected when | Use case |
|---|---|---|
| **Strong** (normal) | Never while reachable | Everything by default |
| **Soft** (`SoftReference`) | Only when memory is low | Memory-sensitive caches |
| **Weak** (`WeakReference`) | At the next GC once only weakly reachable | Canonicalizing maps, listeners, `WeakHashMap` keys |
| **Phantom** (`PhantomReference`) | After finalization; `get()` always returns `null` | Post-mortem cleanup (`Cleaner`, replaces `finalize`) |

```java
WeakHashMap<Key, Metadata> cache = new WeakHashMap<>();   // entry removed once the key is unreachable elsewhere

Cleaner cleaner = Cleaner.create();                         // modern replacement for finalize()
cleaner.register(resourceOwner, () -> releaseNativeHandle(handle));
```

## 8. `finalize()`, `System.gc()`

- `finalize()` is **deprecated for removal** (Java 9, JEP 421 in Java 18). It's unpredictable,
  slow, can resurrect objects, and might never run. Use try-with-resources or `Cleaner`.
- `System.gc()` / `Runtime.getRuntime().gc()` is only a *hint*; the JVM may ignore it (and
  `-XX:+DisableExplicitGC` turns it off). Don't call it in application code.

## 9. Memory leaks in Java

The GC can't free objects that are still **reachable**, even if you'll never use them again.

Common causes:
1. **Static collections** that only grow (caches without eviction).
   ```java
   private static final Map<String, Session> SESSIONS = new HashMap<>();   // never cleaned up
   ```
2. **Listeners/callbacks** registered and never removed.
3. **`ThreadLocal`** values in pooled threads not `remove()`d.
4. **Inner/anonymous classes** holding the outer instance.
5. **Unclosed resources** (streams, connections, `ResultSet`s): native memory and handle leaks.
6. **Mutable keys** in `HashMap` whose hash changed (entries become unreachable via lookup but are
   still referenced).
7. **Class loader leaks** in app servers (redeploys keep old classes in Metaspace).
8. Badly implemented custom collections that don't null out removed slots.

Fixes: bounded caches (Caffeine, `LinkedHashMap` LRU), weak references, try-with-resources,
removing listeners, `ThreadLocal.remove()` in `finally`.

## 10. Tuning & diagnostic tools

```bash
# Common flags
-Xms2g -Xmx2g                    # initial/max heap (equal values avoid resize pauses)
-Xss512k                         # thread stack size
-XX:MaxMetaspaceSize=256m
-XX:+UseG1GC -XX:MaxGCPauseMillis=100
-XX:MaxRAMPercentage=75          # containers: size the heap as % of the container memory limit
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps
-Xlog:gc*:file=gc.log            # unified GC logging (Java 9+)

# Tools
jps                              # list Java processes
jcmd <pid> GC.heap_info          # heap summary
jcmd <pid> GC.heap_dump /tmp/h.hprof
jcmd <pid> Thread.print          # thread dump (deadlocks!)
jstat -gcutil <pid> 1000         # GC stats every second
jcmd <pid> JFR.start duration=60s filename=rec.jfr   # Java Flight Recorder (low overhead profiling)
```
Analyze heap dumps with Eclipse MAT or VisualVM (look for the "dominator tree" and the biggest
retained sizes). JDK Mission Control reads JFR recordings.

### `OutOfMemoryError` flavors

| Message | Likely cause |
|---|---|
| `Java heap space` | Leak or heap too small |
| `GC overhead limit exceeded` | >98% of time spent in GC recovering <2% of heap (Parallel GC) |
| `Metaspace` | Too many classes / class loader leak |
| `unable to create native thread` | Too many platform threads (OS limit or memory) |
| `Direct buffer memory` | NIO direct buffers not released |
| `Requested array size exceeds VM limit` | Allocating an enormous array |

## 11. Gotchas

1. Setting fields to `null` "to help the GC" is usually pointless for locals; useful only for
   long-lived holders (custom stacks, caches).
2. A bigger heap means fewer GCs but potentially longer pauses with older collectors.
3. In containers, older JVMs ignored cgroup limits. Java 10+ (and 8u191+) are container-aware.
4. `StackOverflowError` usually means unbounded recursion, not a too-small `-Xss`.

## 12. Interview questions

1. **Stack vs heap?** See the table in section 1.
2. **What is garbage collection?** Automatic reclamation of memory used by objects no longer
   reachable from GC roots.
3. **What are GC roots?** Thread stack locals, static fields, active threads, JNI references.
4. **How does generational GC work?** Young gen (Eden plus survivors, frequent cheap copying minor
   GCs), promotion to old gen (less frequent major GCs).
5. **Minor vs major vs full GC?** Young gen vs old gen vs the whole heap (plus metaspace).
6. **Which GC is the default?** G1 since Java 9 (Serial on very small machines).
7. **G1 vs ZGC?** G1 balances throughput and pause targets (~tens to hundreds of ms); ZGC targets
   sub-ms pauses at any heap size with slightly more CPU overhead.
8. **Can you force garbage collection?** No. `System.gc()` is a hint.
9. **Can Java have memory leaks?** Yes: objects that stay reachable unintentionally (static maps,
   listeners, `ThreadLocal`, unclosed resources).
10. **Strong vs soft vs weak vs phantom references?** See the table in section 7.
11. **What is `WeakHashMap`?** A map whose entries vanish once their keys are only weakly reachable.
12. **Why is `finalize` deprecated?** Unpredictable timing, performance cost, resurrection bugs;
    replaced by `Cleaner` and try-with-resources.
13. **When is an object eligible for GC?** When it's unreachable from any GC root (including cycles
    of unreachable objects).
14. **PermGen vs Metaspace?** Fixed-size heap area (≤ Java 7) vs native memory that auto-grows
    (Java 8+).
15. **How do you investigate an `OutOfMemoryError`?** Enable heap dumps on OOM, analyze them in MAT
    for the biggest retainers, check GC logs, reproduce with JFR.
16. **What is stop-the-world?** A pause where all application threads stop for a GC phase.
