# 13 — Multithreading & Concurrency

**Coming from JS:** JavaScript runs your code on one thread with an event loop. Java runs **many
threads truly in parallel** sharing the same heap, so you must think about *visibility*,
*atomicity* and *ordering* of memory access.

## 1. Process vs thread

| | Process | Thread |
|---|---|---|
| Memory | Own address space | Shares heap and metaspace with other threads in the process; has its own stack & PC |
| Creation cost | Heavy | Light (platform thread), very light (virtual thread) |
| Communication | IPC (sockets, pipes, files) | Shared memory (needs synchronization) |
| Crash impact | Isolated | An uncaught exception kills only that thread; but memory corruption affects all |

## 2. Creating threads

```java
// 1. Extend Thread (least flexible: uses up your one superclass)
class Worker extends Thread {
    @Override public void run() { System.out.println("running in " + getName()); }
}
new Worker().start();

// 2. Implement Runnable (preferred for the task itself)
Runnable task = () -> System.out.println("running in " + Thread.currentThread().getName());
Thread t = new Thread(task, "worker-1");
t.start();

// 3. Callable + ExecutorService: returns a result, can throw checked exceptions
try (ExecutorService pool = Executors.newFixedThreadPool(4)) {     // AutoCloseable since Java 19
    Future<Integer> f = pool.submit(() -> 6 * 7);
    System.out.println(f.get());       // blocks until done → 42
}

// 4. Builders (Java 21)
Thread platform = Thread.ofPlatform().name("p-1").start(task);
Thread virtual  = Thread.ofVirtual().name("v-1").start(task);
Thread.startVirtualThread(task);
```

**`start()` vs `run()`:** `start()` creates a new thread that then calls `run()`. Calling `run()`
directly just executes it on the **current** thread like a normal method. Calling `start()` twice
throws `IllegalThreadStateException`.

**`Runnable` vs `Callable`:**

| | `Runnable` | `Callable<V>` |
|---|---|---|
| Method | `void run()` | `V call() throws Exception` |
| Returns a value | No | Yes (via `Future`) |
| Checked exceptions | Can't throw | Can throw |

## 3. Thread lifecycle

```
          start()                    scheduler
  NEW ───────────▶ RUNNABLE ◀──────────────────▶ (running on a CPU)
                    │   ▲
  synchronized lock │   │ lock acquired
  not available     ▼   │
                  BLOCKED
                    │   ▲
  wait(), join(),   ▼   │ notify(), thread ends, unpark()
  park()          WAITING
                    │   ▲
  sleep(ms),        ▼   │ timeout / notify
  wait(ms), join(ms) TIMED_WAITING
                    │
  run() returns     ▼
  or throws       TERMINATED
```
`Thread.getState()` returns one of these six `Thread.State` values.

### Key `Thread` methods

| Method | Effect |
|---|---|
| `start()` | Begin execution in a new thread |
| `Thread.sleep(ms)` | Pause the current thread; **keeps** any locks held |
| `join()` | Wait for that thread to finish |
| `interrupt()` | Set the interrupt flag; wakes threads in `sleep`/`wait`/`join` with `InterruptedException` |
| `isInterrupted()` / `Thread.interrupted()` | Check the flag (the static version also clears it) |
| `setDaemon(true)` | Daemon threads don't keep the JVM alive (GC, background tasks); call before `start()` |
| `setPriority(1..10)` | A hint to the scheduler; don't rely on it |
| `Thread.yield()` | A hint to give up the CPU; rarely useful |
| `Thread.onSpinWait()` | A hint inside busy-wait loops |

`stop()`, `suspend()` and `resume()` are deprecated or removed because they're unsafe. Use
**interruption** for cooperative cancellation:
```java
Thread worker = new Thread(() -> {
    while (!Thread.currentThread().isInterrupted()) {
        try {
            doUnitOfWork();
            Thread.sleep(100);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();   // restore the flag and exit
            break;
        }
    }
});
worker.start();
worker.interrupt();
```

## 4. The three problems: race conditions, visibility, ordering

### Race condition (atomicity)
```java
class Counter {
    private int count = 0;
    void increment() { count++; }   // read → add → write: 3 steps, NOT atomic
    int get() { return count; }
}
// Two threads × 10,000 increments → usually prints less than 20,000
```

### Visibility
Each core may cache values. Without a *happens-before* relationship, one thread may **never see**
another thread's write:
```java
class Task implements Runnable {
    private boolean running = true;           // should be volatile
    public void run() { while (running) { } } // may loop forever: JIT can hoist the read
    public void stop() { running = false; }
}
```

### Reordering
The compiler and CPU may reorder instructions that look independent, which is why broken
double-checked locking (chapter 17) could hand out a half-constructed object.

## 5. The Java Memory Model (JMM) & happens-before

If action A **happens-before** action B, then A's effects are visible to B. Main rules:
1. **Program order**: within a thread, each action happens-before the later ones.
2. **Monitor lock**: unlocking a monitor happens-before every later lock of the same monitor.
3. **Volatile**: a write to a volatile field happens-before every later read of it.
4. **Thread start**: `t.start()` happens-before any action in `t`.
5. **Thread join**: all actions in `t` happen-before `t.join()` returns.
6. **Transitivity**: A hb B and B hb C ⇒ A hb C.
7. Concurrent utilities give similar guarantees (e.g. putting into a `BlockingQueue`
   happens-before taking it out; completing a `Future` happens-before `get()` returns).

## 6. `synchronized`

Every object has an **intrinsic lock (monitor)**. `synchronized` gives **mutual exclusion** (one
thread at a time) **and visibility** (changes are flushed on release and seen on acquire).

```java
class SafeCounter {
    private int count;
    private final Object lock = new Object();

    public synchronized void increment() { count++; }    // locks 'this'

    public static synchronized void util() { }           // locks SafeCounter.class (class-level)

    public void add(int n) {
        // ... non-critical work outside the lock ...
        synchronized (lock) {                            // synchronized block: finer-grained, private lock
            count += n;
        }
    }

    public synchronized int get() { return count; }      // reads need sync too (visibility)!
}
```
- **Reentrant**: a thread holding a lock can re-acquire it (a synchronized method calling another
  synchronized method on the same object doesn't deadlock).
- Object-level and class-level locks are **different** locks: a static synchronized method and an
  instance synchronized method can run at the same time.
- Prefer a `private final` lock object over `this` so outside code can't grab your lock.
- With virtual threads, `synchronized` no longer pins the carrier thread (fixed in Java 24, JEP
  491).

## 7. `volatile`

```java
private volatile boolean running = true;
```
- Guarantees **visibility** (reads always see the latest write) and prevents reordering around the
  access.
- Does **not** guarantee **atomicity** of compound actions: `volatile int c; c++` is still a race.
- Use it for flags, status fields, and safe publication of immutable objects (and the
  double-checked locking singleton).

### `synchronized` vs `volatile`

| | `synchronized` | `volatile` |
|---|---|---|
| Applies to | Methods, blocks | Fields |
| Mutual exclusion | ✅ | ❌ |
| Visibility | ✅ | ✅ |
| Atomic compound ops (`x++`) | ✅ | ❌ |
| Can block a thread | Yes | Never |
| Cost | Higher | Lower |

## 8. Atomic variables & CAS

`java.util.concurrent.atomic` gives lock-free thread-safe operations using **CAS
(compare-and-swap)**, a CPU instruction: "set to new value only if it still equals the expected
value; otherwise retry".

```java
AtomicInteger count = new AtomicInteger();
count.incrementAndGet();                         // atomic ++
count.addAndGet(5);
count.compareAndSet(10, 0);                      // CAS
count.updateAndGet(x -> x * 2);
count.accumulateAndGet(3, Math::max);

AtomicLong, AtomicBoolean, AtomicReference<V>, AtomicIntegerArray
LongAdder hits = new LongAdder();                // better than AtomicLong under heavy contention
hits.increment(); hits.sum();
```
**ABA problem:** a value changes A → B → A, and CAS can't tell it changed. Fix with
`AtomicStampedReference` (version stamp).

## 9. Inter-thread communication: `wait` / `notify`

Must be called while **holding the object's monitor** (inside `synchronized` on that object),
otherwise `IllegalMonitorStateException`.

```java
class BoundedBuffer<T> {
    private final Queue<T> queue = new LinkedList<>();
    private final int capacity;
    BoundedBuffer(int capacity) { this.capacity = capacity; }

    public synchronized void put(T item) throws InterruptedException {
        while (queue.size() == capacity) wait();   // ALWAYS loop: spurious wakeups & stolen conditions
        queue.add(item);
        notifyAll();                               // wake consumers
    }

    public synchronized T take() throws InterruptedException {
        while (queue.isEmpty()) wait();            // releases the lock while waiting
        T item = queue.poll();
        notifyAll();                               // wake producers
        return item;
    }
}
```
In real code use a `BlockingQueue` instead (section 12).

### `wait()` vs `sleep()`

| | `wait()` | `sleep()` |
|---|---|---|
| Defined in | `Object` | `Thread` (static) |
| Releases the lock | **Yes** | **No** |
| Must hold the monitor | Yes | No |
| Woken by | `notify`/`notifyAll`, timeout, interrupt | Timeout, interrupt |
| Purpose | Coordination between threads | Pause |

`notify()` wakes one arbitrary waiting thread; `notifyAll()` wakes all of them (safer, usually
preferred).

## 10. Explicit locks (`java.util.concurrent.locks`)

```java
private final ReentrantLock lock = new ReentrantLock();      // new ReentrantLock(true) = fair

public void transfer() {
    lock.lock();
    try {
        // critical section
    } finally {
        lock.unlock();                                       // ALWAYS in finally
    }
}

if (lock.tryLock(1, TimeUnit.SECONDS)) {                     // timeout: helps avoid deadlock
    try { ... } finally { lock.unlock(); }
}
lock.lockInterruptibly();                                    // can be interrupted while waiting

// Conditions: like wait/notify but with several wait-sets per lock
Condition notFull = lock.newCondition(), notEmpty = lock.newCondition();
notFull.await(); notEmpty.signal();

// Many readers OR one writer
ReadWriteLock rw = new ReentrantReadWriteLock();
rw.readLock().lock();  rw.writeLock().lock();

// StampedLock (Java 8): optimistic reads for read-mostly data
```

### `synchronized` vs `ReentrantLock`

| | `synchronized` | `ReentrantLock` |
|---|---|---|
| Acquire/release | Automatic (block scope) | Manual `lock()`/`unlock()` in `finally` |
| Try with timeout | ❌ | `tryLock(timeout)` |
| Interruptible wait | ❌ | `lockInterruptibly()` |
| Fairness option | ❌ | ✅ |
| Multiple conditions | One wait-set | Many `Condition`s |
| Simplicity | ✅ simpler, less error-prone | More control |

## 11. Executor framework & thread pools

Creating a thread per task is expensive and unbounded. **Thread pools** reuse a fixed set of
threads and queue tasks.

```java
ExecutorService fixed  = Executors.newFixedThreadPool(4);        // N threads, unbounded queue
ExecutorService cached = Executors.newCachedThreadPool();        // grows as needed, reuses idle threads (60s)
ExecutorService single = Executors.newSingleThreadExecutor();    // sequential execution
ScheduledExecutorService sched = Executors.newScheduledThreadPool(2);
ExecutorService steal  = Executors.newWorkStealingPool();        // ForkJoinPool
ExecutorService vts    = Executors.newVirtualThreadPerTaskExecutor();   // Java 21

sched.scheduleAtFixedRate(() -> poll(), 0, 5, TimeUnit.SECONDS);

// submit vs execute
pool.execute(runnable);                  // fire and forget; exceptions go to the thread's handler
Future<?> f = pool.submit(runnable);     // exceptions are captured in the Future (lost if you never call get()!)

List<Future<Integer>> results = pool.invokeAll(List.of(() -> 1, () -> 2));   // wait for all
Integer first = pool.invokeAny(List.of(() -> slow(), () -> fast()));         // first successful

// Shutdown (pre-Java 19 style)
pool.shutdown();                                   // stop accepting new tasks, finish queued ones
if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
    pool.shutdownNow();                            // interrupt running tasks, return queued ones
}
```

### `ThreadPoolExecutor`: what production code should configure
```java
ExecutorService pool = new ThreadPoolExecutor(
    4,                                   // corePoolSize
    8,                                   // maximumPoolSize
    60, TimeUnit.SECONDS,                // keep-alive for threads above core
    new ArrayBlockingQueue<>(100),       // BOUNDED queue (Executors.newFixedThreadPool's is unbounded → OOM risk)
    Thread.ofPlatform().name("orders-", 0).factory(),
    new ThreadPoolExecutor.CallerRunsPolicy());   // rejection policy: back-pressure
```
**Task submission order:** core threads → queue → extra threads up to max → rejection policy
(`AbortPolicy` default throws `RejectedExecutionException`; `CallerRunsPolicy`, `DiscardPolicy`,
`DiscardOldestPolicy`).

**Sizing rule of thumb:** CPU-bound → about the number of cores. I/O-bound → cores × (1 + wait
time / compute time), or simply use virtual threads.

## 12. Concurrent utilities

```java
// BlockingQueue: producer–consumer without wait/notify
BlockingQueue<Order> q = new ArrayBlockingQueue<>(100);
q.put(order);     // blocks if full
q.take();         // blocks if empty
q.offer(o, 1, TimeUnit.SECONDS); q.poll(1, TimeUnit.SECONDS);

// CountDownLatch: wait until N events happen (one-shot)
CountDownLatch ready = new CountDownLatch(3);
// each worker: ready.countDown();
ready.await();    // main waits for all 3

// CyclicBarrier: N threads wait for each other, reusable
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("all arrived"));
barrier.await();

// Semaphore: limit concurrent access to N permits
Semaphore permits = new Semaphore(5);
permits.acquire(); try { callRateLimitedApi(); } finally { permits.release(); }

// Phaser (flexible barrier), Exchanger (swap objects between two threads)
```

| | `CountDownLatch` | `CyclicBarrier` |
|---|---|---|
| Who waits | One or more threads wait for others to count down | All participating threads wait for each other |
| Reusable | No | Yes |
| Count changes via | `countDown()` (any thread, can be one thread many times) | `await()` by each party |

## 13. `CompletableFuture`: async pipelines (Java 8)

`Future.get()` blocks and can't be chained. `CompletableFuture` composes async steps, like JS
Promises.

```java
ExecutorService io = Executors.newVirtualThreadPerTaskExecutor();

CompletableFuture<User> userF = CompletableFuture.supplyAsync(() -> userApi.get(id), io);
CompletableFuture<List<Order>> ordersF = CompletableFuture.supplyAsync(() -> orderApi.forUser(id), io);

CompletableFuture<Dashboard> dash = userF
    .thenCombine(ordersF, Dashboard::new)               // like Promise.all for two
    .thenApply(d -> d.withBadge(rank(d)))               // map (sync transform)
    .thenCompose(d -> enrichAsync(d))                   // flatMap (async step returning a CF)
    .orTimeout(2, TimeUnit.SECONDS)                     // Java 9
    .exceptionally(ex -> Dashboard.empty());            // like .catch

dash.thenAccept(System.out::println);                   // consume
Dashboard d = dash.join();                              // block (unchecked exception), or get()

CompletableFuture.allOf(f1, f2, f3).join();             // wait for all
CompletableFuture.anyOf(f1, f2).join();                 // first to finish
f.handle((value, ex) -> ex == null ? value : fallback); // success or failure
f.whenComplete((v, ex) -> log(v, ex));                  // side effect, passes the result through
f.completeOnTimeout(defaultValue, 1, TimeUnit.SECONDS);
```

| JS Promise | CompletableFuture |
|---|---|
| `.then(x => y)` | `thenApply` |
| `.then(x => promise)` | `thenCompose` |
| `.then(x => { sideEffect })` | `thenAccept` / `thenRun` |
| `.catch` | `exceptionally` / `handle` |
| `Promise.all` | `allOf` (+ `join` each) / `thenCombine` |
| `Promise.race` / `any` | `anyOf` |

Without an executor argument, the `*Async` methods use `ForkJoinPool.commonPool()`. Don't run
blocking I/O there; pass your own executor.

## 14. Fork/Join framework

Divide and conquer for CPU-bound recursive work, using **work stealing**: idle threads steal tasks
from busy threads' deques.
```java
class SumTask extends RecursiveTask<Long> {
    private final int[] arr; private final int lo, hi;
    SumTask(int[] arr, int lo, int hi) { this.arr = arr; this.lo = lo; this.hi = hi; }
    @Override protected Long compute() {
        if (hi - lo <= 10_000) {
            long s = 0; for (int i = lo; i < hi; i++) s += arr[i]; return s;
        }
        int mid = (lo + hi) >>> 1;
        SumTask left = new SumTask(arr, lo, mid);
        left.fork();                                       // run asynchronously
        return new SumTask(arr, mid, hi).compute() + left.join();
    }
}
long total = ForkJoinPool.commonPool().invoke(new SumTask(arr, 0, arr.length));
```
Parallel streams use this framework under the hood.

## 15. Virtual threads (Java 21, Project Loom)

**Platform threads** map 1:1 to OS threads (about 1 MB of stack each, thousands max). **Virtual
threads** are lightweight threads managed by the JVM and mounted on a small pool of **carrier**
platform threads. When a virtual thread blocks on I/O, it **unmounts** and the carrier runs another
one. You can have **millions** of them.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 100_000).forEach(i ->
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));   // blocking is cheap now
            return i;
        }));
}   // close() waits for all tasks: 100k tasks in ~1 second
```

- Write simple **blocking, thread-per-request** code and get async-level scalability, with no
  reactive callbacks.
- Best for **I/O-bound** workloads (HTTP calls, DB queries). No benefit for CPU-bound work.
- **Don't pool virtual threads**; create one per task. Limit concurrency to scarce resources with a
  `Semaphore`.
- Be careful with `ThreadLocal` caches (millions of threads means millions of copies). Prefer
  **scoped values** (final in Java 25, JEP 506).
- **Pinning:** before Java 24, blocking inside `synchronized` pinned the virtual thread to its
  carrier. Java 24 (JEP 491) fixed this; native calls can still pin.
- Spring Boot 3.2+: `spring.threads.virtual.enabled=true`.

### Structured concurrency (preview in 21–26)
Treats a group of concurrent subtasks as one unit: if one fails, the others are cancelled, and the
scope doesn't exit until all are done. The API has changed between previews (Java 25 uses
`StructuredTaskScope.open()`), so check your version's docs. Recognize the concept in interviews.

```java
// Java 25 preview API shape
try (var scope = StructuredTaskScope.open()) {
    Subtask<User> user = scope.fork(() -> findUser(id));
    Subtask<List<Order>> orders = scope.fork(() -> findOrders(id));
    scope.join();                                   // throws if any subtask failed; others cancelled
    return new Dashboard(user.get(), orders.get());
}
```

## 16. Deadlock, livelock, starvation

**Deadlock:** two or more threads each hold a lock the other needs, so they wait forever.
```java
// Thread 1: synchronized(a) { synchronized(b) { ... } }
// Thread 2: synchronized(b) { synchronized(a) { ... } }    ← opposite order → deadlock
```
Four necessary conditions (Coffman): mutual exclusion, hold and wait, no preemption, circular wait.

Prevention:
1. **Consistent global lock ordering** (e.g. always lock the account with the lower id first).
2. `tryLock` with a timeout, then back off.
3. Hold locks for as short a time as possible; avoid calling unknown code (callbacks) while holding
   a lock.
4. Use higher-level concurrency utilities instead of nested locks.

Detection: `jstack <pid>` or a thread dump (`jcmd <pid> Thread.print`) reports "Found one Java-level
deadlock". `ThreadMXBean.findDeadlockedThreads()` does the same programmatically.

**Livelock:** threads keep reacting to each other and change state but make no progress (two people
stepping aside in a corridor). **Starvation:** a thread never gets CPU time or a lock because others
monopolize it (unfair locks, priorities).

## 17. `ThreadLocal`

Gives each thread its own copy of a variable. Used for per-request context (user, transaction,
`SimpleDateFormat`, which isn't thread-safe).
```java
private static final ThreadLocal<DateFormat> FMT =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
FMT.get().format(date);

// In thread pools, ALWAYS clean up, or values leak to the next task on that thread
try { CONTEXT.set(user); handle(); } finally { CONTEXT.remove(); }
```
(Use `DateTimeFormatter`, which is immutable and thread-safe, instead of `SimpleDateFormat` in new
code.)

## 18. Thread-safety strategies, in order of preference

1. **Immutability**: share only immutable objects.
2. **Confinement**: don't share (local variables, `ThreadLocal`, one thread owns the data).
3. **Concurrent collections and atomics**: `ConcurrentHashMap`, `AtomicInteger`, `BlockingQueue`.
4. **Locks**: `synchronized` / `ReentrantLock` around compound actions.

## 19. Gotchas

1. Calling `run()` instead of `start()`.
2. `volatile` for counters (`count++` is still a race).
3. Unsynchronized reads of a field that's written under a lock (visibility).
4. Swallowing `InterruptedException` without restoring the flag.
5. `wait()` in an `if` instead of a `while`.
6. Exceptions in `submit()`ted tasks silently disappear if you never call `Future.get()`.
7. Unbounded queues in `newFixedThreadPool` hide overload until you run out of memory.
8. Blocking I/O in `ForkJoinPool.commonPool()` (parallel streams, default `CompletableFuture`).
9. Forgetting `ThreadLocal.remove()` in pooled threads.
10. Synchronizing on a non-final field, a `String` literal, or a boxed `Integer` (shared or changing
    lock objects).

## 20. Interview questions

1. **Process vs thread?** See the table in section 1.
2. **Ways to create a thread?** Extend `Thread`, implement `Runnable`/`Callable` with an executor,
   `Thread.ofVirtual()`/`ofPlatform()` builders.
3. **`start()` vs `run()`?** A new thread vs a plain method call on the current thread.
4. **What are the thread states?** NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED.
5. **`wait()` vs `sleep()`?** `wait` releases the monitor and needs `synchronized`; `sleep` keeps
   locks.
6. **Why are `wait`/`notify` in `Object` and not `Thread`?** Because they operate on an object's
   monitor, and any object can be a lock.
7. **Why call `wait()` in a loop?** Spurious wakeups, and the condition may have changed again
   before the thread reacquired the lock.
8. **What does `volatile` guarantee?** Visibility and ordering, but not atomicity.
9. **`synchronized` vs `volatile`?** See the table in section 7.
10. **`synchronized` vs `ReentrantLock`?** See the table in section 10.
11. **What is a race condition? How do you fix it?** The outcome depends on thread timing. Fix with
    synchronization, atomics or immutability.
12. **What is a deadlock and how do you prevent it?** Circular waiting on locks; prevent with lock
    ordering, timeouts, fewer nested locks.
13. **What is CAS?** Atomic compare-and-swap CPU instruction behind atomics and lock-free algorithms.
14. **What is the Java Memory Model and happens-before?** Rules defining when one thread's writes
    are guaranteed visible to another.
15. **What is a thread pool and why use one?** Reuses threads, bounds concurrency, queues tasks.
16. **`submit()` vs `execute()`?** `submit` returns a `Future` and captures exceptions; `execute`
    returns nothing.
17. **`shutdown()` vs `shutdownNow()`?** Graceful vs interrupt running tasks and drain the queue.
18. **`Future` vs `CompletableFuture`?** Blocking `get` only vs composable, non-blocking callbacks
    with combinators and manual completion.
19. **`CountDownLatch` vs `CyclicBarrier`?** One-shot "wait for N events" vs reusable "N threads
    wait for each other".
20. **What is a `Semaphore`?** A counter of permits that limits concurrent access.
21. **What are virtual threads and when do you use them?** JVM-managed lightweight threads for
    high-concurrency blocking I/O; not for CPU-bound work; don't pool them.
22. **What is `ThreadLocal`? What's the risk?** A per-thread variable; leaks in thread pools if not
    removed.
23. **What is a daemon thread?** A background thread that doesn't prevent JVM exit.
24. **How does `ConcurrentHashMap` achieve thread safety?** CAS plus per-bucket `synchronized`,
    lock-free reads (chapter 10).
25. **How do you make a class thread-safe?** Immutability, confinement, concurrent utilities, then
    locking.
