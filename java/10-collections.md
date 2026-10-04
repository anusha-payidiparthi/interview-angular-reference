# 10 — Collections Framework

The single most important API area for interviews, especially **`HashMap` internals**.

## 1. The hierarchy

```
                         Iterable<E>
                             │
                       Collection<E>
          ┌──────────────┬───┴──────────┬────────────────┐
        List<E>        Set<E>        Queue<E>      (SequencedCollection, Java 21)
   ArrayList        HashSet          PriorityQueue
   LinkedList       LinkedHashSet    Deque<E>
   Vector (legacy)  SortedSet          ArrayDeque
   CopyOnWriteArrayList └ NavigableSet  LinkedList
                          └ TreeSet

                  Map<K,V>   (NOT a Collection)
   HashMap   LinkedHashMap   TreeMap (SortedMap → NavigableMap)   Hashtable (legacy)
   ConcurrentHashMap   EnumMap   WeakHashMap   IdentityHashMap
```

**`Collection` vs `Collections`:** `Collection` is the root interface; `Collections` is a utility
class (`sort`, `reverse`, `shuffle`, `unmodifiableList`, `synchronizedList`, `emptyList`, `max`,
`frequency`).

## 2. Choosing the right collection

| Need | Use |
|---|---|
| Ordered list with fast random access | `ArrayList` |
| Unique elements, no ordering | `HashSet` |
| Unique elements, insertion order | `LinkedHashSet` |
| Unique elements, sorted | `TreeSet` |
| Key → value, fastest | `HashMap` |
| Key → value, insertion (or access) order | `LinkedHashMap` |
| Key → value, sorted keys, range queries | `TreeMap` |
| Stack or queue (FIFO/LIFO) | `ArrayDeque` |
| Priority (min/max heap) | `PriorityQueue` |
| Thread-safe map | `ConcurrentHashMap` |
| Thread-safe list, read-heavy | `CopyOnWriteArrayList` |
| Producer–consumer | `BlockingQueue` (`ArrayBlockingQueue`, `LinkedBlockingQueue`) |
| Enum keys | `EnumMap` / `EnumSet` |
| Immutable | `List.of`, `Set.of`, `Map.of`, `List.copyOf` |

## 3. Big-O cheat sheet

| Collection | get/access | add | remove | contains | Ordering |
|---|---|---|---|---|---|
| `ArrayList` | O(1) by index | O(1) amortized at end; O(n) middle | O(n) | O(n) | Insertion |
| `LinkedList` | O(n) | O(1) at ends | O(1) at ends / via iterator; O(n) to find | O(n) | Insertion |
| `ArrayDeque` | n/a | O(1) amortized at ends | O(1) at ends | O(n) | Insertion |
| `HashSet`/`HashMap` | O(1) avg | O(1) avg | O(1) avg | O(1) avg (O(log n) worst, treeified) | None |
| `LinkedHashMap`/`Set` | O(1) | O(1) | O(1) | O(1) | Insertion/access |
| `TreeMap`/`TreeSet` | O(log n) | O(log n) | O(log n) | O(log n) | Sorted (red-black tree) |
| `PriorityQueue` | O(1) peek | O(log n) | O(log n) poll; O(n) arbitrary | O(n) | Heap order |

## 4. List

```java
List<String> list = new ArrayList<>();
list.add("a"); list.add(0, "z"); list.addAll(List.of("b", "c"));
list.get(1); list.set(1, "A"); list.remove("c"); list.remove(0);
list.indexOf("b"); list.contains("b"); list.size(); list.isEmpty();
list.subList(0, 1);                       // a view, not a copy
list.sort(Comparator.naturalOrder());
list.replaceAll(String::toUpperCase);
list.removeIf(s -> s.startsWith("x"));
Collections.reverse(list); Collections.shuffle(list);
```

### `ArrayList` internals
- Backed by an `Object[]`. Default capacity 10 (allocated lazily on the first add).
- When full it grows by **~50%** (`newCapacity = old + old >> 1`) and copies with
  `Arrays.copyOf`.
- `add` at the end is amortized O(1); inserting or removing in the middle shifts elements (O(n)).
- Pre-size when you know the count: `new ArrayList<>(10_000)`.

### `ArrayList` vs `LinkedList`

| | `ArrayList` | `LinkedList` |
|---|---|---|
| Structure | Dynamic array | Doubly linked list |
| Random access `get(i)` | **O(1)** | O(n) |
| Insert/remove at ends | O(1) at end, O(n) at front | **O(1)** |
| Insert/remove in middle | O(n) shift | O(n) to find, O(1) to relink |
| Memory | Compact | ~24 extra bytes per node (prev/next/object header) |
| CPU cache friendliness | Excellent | Poor (pointer chasing) |
| Implements | `List`, `RandomAccess` | `List`, `Deque` |

In practice `ArrayList` wins almost always; for queues use `ArrayDeque`.

### `ArrayList` vs `Vector`
`Vector` is synchronized on every method (slow) and doubles its capacity. It's legacy; use
`ArrayList`, or `Collections.synchronizedList` / `CopyOnWriteArrayList` if you need thread safety.

## 5. Set

```java
Set<String> hs = new HashSet<>(List.of("b", "a", "c"));   // no order
Set<String> lhs = new LinkedHashSet<>(List.of("b", "a")); // [b, a]
TreeSet<Integer> ts = new TreeSet<>(List.of(5, 1, 9, 3)); // [1, 3, 5, 9]
ts.first(); ts.last(); ts.floor(4); ts.ceiling(4); ts.headSet(5); ts.tailSet(5);
ts.descendingSet();

boolean added = hs.add("a");      // false: already present
// Set ops
Set<Integer> a = new HashSet<>(Set.of(1, 2, 3)), b = Set.of(2, 3, 4);
a.retainAll(b);   // intersection → [2, 3]
a.addAll(b);      // union
a.removeAll(b);   // difference
```

`HashSet` is internally a `HashMap` where the elements are keys and the value is a dummy constant
object. That's why element uniqueness depends on `equals`/`hashCode`, and `TreeSet` (backed by a
`TreeMap`) depends on `compareTo`/`Comparator`.

## 6. Map

```java
Map<String, Integer> m = new HashMap<>();
m.put("a", 1);
m.get("a");                    // 1
m.get("zzz");                  // null (ambiguous: missing key or null value?)
m.getOrDefault("zzz", 0);      // 0
m.containsKey("a"); m.containsValue(1);
m.putIfAbsent("b", 2);
m.remove("a");
m.merge("w", 1, Integer::sum);                       // word count idiom
m.computeIfAbsent("k", k -> new ArrayList<>());      // multimap idiom (Map<String, List<..>>)
m.compute("a", (k, v) -> v == null ? 1 : v + 1);
m.replaceAll((k, v) -> v * 10);

for (Map.Entry<String, Integer> e : m.entrySet()) {  // iterate entries (not keySet + get)
    System.out.println(e.getKey() + "=" + e.getValue());
}
m.forEach((k, v) -> System.out.println(k + "=" + v));

TreeMap<String, Integer> tm = new TreeMap<>(m);
tm.firstKey(); tm.headMap("m"); tm.ceilingEntry("c"); tm.descendingMap();

Map<String, Integer> im = Map.of("a", 1, "b", 2);    // immutable, ≤10 pairs
Map<String, Integer> im2 = Map.ofEntries(Map.entry("a", 1));
```

### How `HashMap` works internally (THE interview question)

```
table: Node<K,V>[]  (length is always a power of 2; default 16)

index = (n - 1) & hash          where hash = key.hashCode() ^ (key.hashCode() >>> 16)

 [0] → null
 [1] → Node(k1,v1) → Node(k9,v9)          ← collision: linked list (bucket)
 [2] → null
 [3] → TreeNode ...                        ← ≥ 8 nodes in a bucket (and table ≥ 64): red-black tree
 ...
[15] → Node(k4,v4)
```

**`put(key, value)`:**
1. If `key == null`, use hash 0 (bucket 0). HashMap allows **one null key**.
2. Compute `hash(key)`: `hashCode()` XOR its high 16 bits ("spreading"), so tables that only use
   the low bits still see the high bits.
3. Index = `(n - 1) & hash` (a fast modulo, which is why the capacity is a power of 2).
4. Bucket empty → store a new node.
5. Otherwise walk the bucket: if a node has the same hash **and** `equals` the key → **replace the
   value** (returns the old value). If none matches → append a new node.
6. If the bucket length reaches **8** (`TREEIFY_THRESHOLD`) and capacity ≥ **64**, convert the
   bucket to a **red-black tree** (Java 8), making the worst case O(log n) instead of O(n). If the
   capacity is < 64 it resizes instead. Trees turn back into lists at **6**.
7. If `size > capacity × loadFactor` (default **0.75**, so 12 for capacity 16) → **resize**: double
   the capacity and redistribute. Each node either stays at index `i` or moves to `i + oldCap`, a
   single bit check with no full rehash.

**`get(key)`:** compute hash → index → walk the bucket comparing `hash` then `equals` → return the
value or `null`.

Why it matters:
- Bad `hashCode` (e.g. always returns 1) → every entry in one bucket → O(n), or O(log n) once
  treeified.
- Mutable keys → the hash changes after insertion → entry lost.
- `equals` without `hashCode` → logically equal keys land in different buckets.

Java 7 vs 8: Java 7 inserted at the head of the bucket's list (resize under concurrency could
create a cycle, an infinite loop), and had no treeification. Java 8 appends at the tail and adds
trees.

### `HashMap` vs `Hashtable` vs `ConcurrentHashMap` vs `synchronizedMap`

| | `HashMap` | `Hashtable` | `Collections.synchronizedMap` | `ConcurrentHashMap` |
|---|---|---|---|---|
| Thread-safe | ❌ | ✅ (every method `synchronized`) | ✅ (one lock for the whole map) | ✅ (fine-grained) |
| Null keys/values | 1 null key, many null values | ❌ | Same as wrapped map | ❌ (null would be ambiguous with "absent" in concurrent code) |
| Performance under contention | n/a | Poor | Poor | Excellent |
| Iterator | Fail-fast | Fail-fast (Enumeration not) | Fail-fast (must sync manually when iterating) | **Weakly consistent**, never throws CME |
| Status | Default choice | Legacy | OK for low contention | Default for concurrency |

**How `ConcurrentHashMap` works (Java 8+):** reads are lock-free (volatile reads). Writes use **CAS**
to insert into an empty bucket, and otherwise `synchronized` on the **first node of that bucket
only**, so different buckets can be written in parallel. Resizing is done cooperatively by multiple
threads. (Java 7 used 16 "segments", each a lock; that's gone.) Atomic compound operations:
`putIfAbsent`, `computeIfAbsent`, `merge`, `compute`.

```java
ConcurrentHashMap<String, LongAdder> hits = new ConcurrentHashMap<>();
hits.computeIfAbsent(url, k -> new LongAdder()).increment();   // thread-safe counter

// NOT atomic: check-then-act race even on a ConcurrentHashMap
if (!map.containsKey(k)) map.put(k, v);   // ❌
map.putIfAbsent(k, v);                    // ✅
```

### `LinkedHashMap` as an LRU cache
```java
class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    LruCache(int capacity) {
        super(16, 0.75f, true);              // accessOrder = true: get() moves entry to the end
        this.capacity = capacity;
    }
    @Override protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;            // evict least recently used
    }
}
```

## 7. Queue, Deque, PriorityQueue

```java
Queue<Integer> q = new ArrayDeque<>();
q.offer(1); q.offer(2);         // add to tail (offer returns false if full; add throws)
q.peek();                       // 1 (null if empty; element() throws)
q.poll();                       // 1 (null if empty; remove() throws)

Deque<Integer> stack = new ArrayDeque<>();   // preferred over legacy Stack class
stack.push(1); stack.push(2);
stack.pop();                    // 2 (LIFO)
stack.peekFirst(); stack.offerLast(3); stack.pollLast();

PriorityQueue<Integer> minHeap = new PriorityQueue<>();
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
PriorityQueue<Task> byPriority = new PriorityQueue<>(Comparator.comparingInt(Task::priority));

// Top-K largest with a min-heap of size k: O(n log k)
PriorityQueue<Integer> heap = new PriorityQueue<>();
for (int n : nums) { heap.offer(n); if (heap.size() > k) heap.poll(); }
```
Iterating a `PriorityQueue` does **not** give sorted order; only `poll()` does.
`ArrayDeque` doesn't allow `null` elements.

| Operation | Throws exception | Returns special value |
|---|---|---|
| Insert | `add(e)` | `offer(e)` → `false` |
| Remove | `remove()` | `poll()` → `null` |
| Examine | `element()` | `peek()` → `null` |

## 8. Iteration, fail-fast vs fail-safe

```java
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().isEmpty()) it.remove();     // the ONLY safe way to remove while iterating (besides removeIf)
}

ListIterator<String> li = list.listIterator();   // bidirectional, can set() and add()
```

- **Fail-fast** iterators (`ArrayList`, `HashMap`, …) track a `modCount`. A structural change not
  made through the iterator → `ConcurrentModificationException` on the next `next()`. This is
  best-effort bug detection, not a thread-safety guarantee.
- **Fail-safe / weakly consistent** iterators (`ConcurrentHashMap`, `CopyOnWriteArrayList`) never
  throw CME. `CopyOnWriteArrayList` iterates over a **snapshot**; `ConcurrentHashMap` may or may
  not reflect concurrent updates.

## 9. Immutable & unmodifiable collections

```java
List<String> a = List.of("x", "y");             // immutable; nulls → NPE; add → UnsupportedOperationException
List<String> b = List.copyOf(source);           // immutable snapshot
List<String> c = Collections.unmodifiableList(source);   // read-only VIEW (source changes show through)
List<String> d = stream.toList();               // Java 16: unmodifiable
List<String> e = stream.collect(Collectors.toList());    // currently an ArrayList (no guarantee)
```

## 10. Sequenced collections (Java 21)

New interfaces `SequencedCollection`, `SequencedSet` and `SequencedMap` give a uniform API for
collections with a defined encounter order:
```java
list.getFirst(); list.getLast(); list.addFirst(x); list.removeLast();
list.reversed();                                 // reversed view
linkedHashSet.getFirst();
linkedHashMap.firstEntry(); linkedHashMap.pollLastEntry(); linkedHashMap.putFirst(k, v);
```
Before Java 21, getting the last element of a `LinkedHashSet` required iterating the whole thing.

## 11. Sorting & utilities

```java
Collections.sort(list);                          // natural order, stable TimSort
list.sort(Comparator.comparing(Person::age).thenComparing(Person::name));
Collections.max(list); Collections.min(list, cmp);
Collections.frequency(list, "a");
Collections.nCopies(3, "x");                     // [x, x, x]
Collections.swap(list, 0, 1);
Collections.emptyList(); Collections.singletonList(x);
Collections.synchronizedList(new ArrayList<>());
Collections.binarySearch(sortedList, key);
```

## 12. Gotchas

1. `List.remove(int)` vs `List.remove(Object)` with `List<Integer>`.
2. `Arrays.asList` is fixed-size; `List.of` is immutable and rejects nulls.
3. Modifying a collection inside for-each → `ConcurrentModificationException`.
4. Mutable objects as `HashMap` keys or `HashSet` elements.
5. `TreeMap`/`TreeSet` use `compareTo`, not `equals`, to detect duplicates.
6. `map.get(k)` returning `null` is ambiguous; use `containsKey` or `getOrDefault`.
7. `ConcurrentHashMap` makes single operations atomic, not sequences of operations.
8. `PriorityQueue` iteration order isn't sorted.
9. `subList` is a view; structurally modifying the parent invalidates it.
10. `HashMap` iteration order is not guaranteed and may change between Java versions.

## 13. Interview questions

1. **How does `HashMap` work internally?** Array of buckets; hash spreading; index = `(n-1) & hash`;
   collisions via linked list, treeified at 8; resize at 0.75 load factor by doubling.
2. **What happens when two keys have the same hash code?** Collision. Both live in the same bucket
   and are distinguished with `equals`.
3. **Why must `HashMap` capacity be a power of two?** So `(n-1) & hash` replaces the slower `%`, and
   resizing can split buckets with one bit check.
4. **What is the load factor? Why 0.75?** The ratio of entries to capacity that triggers a resize;
   0.75 balances memory against collision probability.
5. **What changed in `HashMap` in Java 8?** Treeification of long buckets, tail insertion, a simpler
   hash function.
6. **`HashMap` vs `Hashtable` vs `ConcurrentHashMap`?** See the table in section 6.
7. **Why doesn't `ConcurrentHashMap` allow nulls?** `get()` returning `null` couldn't distinguish
   "absent" from "null value", and you can't safely check `containsKey` afterwards in concurrent
   code.
8. **`ArrayList` vs `LinkedList`?** See the table in section 4.
9. **How does `ArrayList` grow?** By about 1.5× via array copy.
10. **`HashSet` vs `TreeSet` vs `LinkedHashSet`?** Unordered O(1) vs sorted O(log n) vs insertion
    order O(1).
11. **How does `HashSet` ensure uniqueness?** It's backed by a `HashMap`; elements are keys.
12. **Fail-fast vs fail-safe iterators?** `modCount` check that throws CME vs snapshot or weakly
    consistent iteration.
13. **How do you remove elements while iterating?** `Iterator.remove()` or `removeIf`.
14. **`Comparable` vs `Comparator`?** See chapter 08.
15. **How do you make a collection thread-safe?** Concurrent collections (`ConcurrentHashMap`,
    `CopyOnWriteArrayList`, `BlockingQueue`), `Collections.synchronizedX`, or immutable
    collections.
16. **How would you implement an LRU cache?** `LinkedHashMap` with `accessOrder = true` and
    `removeEldestEntry`, or a `HashMap` plus a doubly linked list (chapter 20).
17. **`Iterator` vs `ListIterator`?** `ListIterator` can go backward and supports `set`/`add`/index
    methods.
18. **Why prefer `ArrayDeque` over `Stack`?** `Stack` extends synchronized `Vector` (slow, exposes
    index operations); `ArrayDeque` is faster and has a clean `Deque` API.
