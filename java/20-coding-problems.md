# 20 — Java Coding Problems (with Solutions)

The hands-on questions that show up in Core Java rounds: string/array manipulation, collections,
streams, and concurrency. Each solution notes its time and space complexity. Try each one yourself
before reading the answer.

All snippets assume `import java.util.*; import java.util.function.*; import java.util.stream.*;`
and, for the concurrency section, `import java.util.concurrent.*;`.

---

## Part 1: Strings

### 1. Reverse a string (three ways)
```java
static String reverse1(String s) { return new StringBuilder(s).reverse().toString(); }

static String reverse2(String s) {                 // two pointers: what interviewers usually want
    char[] c = s.toCharArray();
    for (int i = 0, j = c.length - 1; i < j; i++, j--) {
        char t = c[i]; c[i] = c[j]; c[j] = t;
    }
    return new String(c);
}

static String reverse3(String s) {                 // recursion (O(n²) due to substring; mention it)
    return s.isEmpty() ? s : reverse3(s.substring(1)) + s.charAt(0);
}
```
O(n) time, O(n) space (strings are immutable, so a copy is unavoidable).

### 2. Palindrome check (ignore case and non-alphanumerics)
```java
static boolean isPalindrome(String s) {
    int i = 0, j = s.length() - 1;
    while (i < j) {
        while (i < j && !Character.isLetterOrDigit(s.charAt(i))) i++;
        while (i < j && !Character.isLetterOrDigit(s.charAt(j))) j--;
        if (Character.toLowerCase(s.charAt(i++)) != Character.toLowerCase(s.charAt(j--))) return false;
    }
    return true;
}
// isPalindrome("A man, a plan, a canal: Panama") → true
```
O(n) time, O(1) space.

### 3. Anagram check
```java
static boolean isAnagram(String a, String b) {
    if (a.length() != b.length()) return false;
    int[] counts = new int[26];                     // assumes lowercase a–z
    for (int i = 0; i < a.length(); i++) {
        counts[a.charAt(i) - 'a']++;
        counts[b.charAt(i) - 'a']--;
    }
    for (int c : counts) if (c != 0) return false;
    return true;
}
// For any Unicode: use a Map<Character, Integer>, or sort both char arrays (O(n log n)).
```
O(n) time, O(1) space.

### 4. Character frequency
```java
// Imperative
Map<Character, Integer> freq = new LinkedHashMap<>();
for (char c : s.toCharArray()) freq.merge(c, 1, Integer::sum);

// Streams
Map<Character, Long> freq2 = s.chars().mapToObj(c -> (char) c)
    .collect(Collectors.groupingBy(Function.identity(), LinkedHashMap::new, Collectors.counting()));
```

### 5. First non-repeating character
```java
static Character firstUnique(String s) {
    Map<Character, Integer> counts = new LinkedHashMap<>();   // keeps insertion order
    for (char c : s.toCharArray()) counts.merge(c, 1, Integer::sum);
    for (var e : counts.entrySet()) if (e.getValue() == 1) return e.getKey();
    return null;
}
// firstUnique("swiss") → 'w'
```
O(n) time.

### 6. Reverse the words in a sentence
```java
static String reverseWords(String s) {
    String[] words = s.trim().split("\\s+");
    Collections.reverse(Arrays.asList(words));      // asList is a view, so this reverses the array
    return String.join(" ", words);
}
// reverseWords("  Java is   fun ") → "fun is Java"
```

### 7. Count vowels and consonants
```java
static int[] vowelsConsonants(String s) {
    int v = 0, c = 0;
    for (char ch : s.toLowerCase().toCharArray()) {
        if (ch >= 'a' && ch <= 'z') {
            if ("aeiou".indexOf(ch) >= 0) v++; else c++;
        }
    }
    return new int[]{v, c};
}
```

### 8. Remove duplicate characters, keep order
```java
static String dedupe(String s) {
    return s.chars().distinct()
            .collect(StringBuilder::new, StringBuilder::appendCodePoint, StringBuilder::append)
            .toString();
}
// dedupe("programming") → "progamin"
```

### 9. Check whether a string contains only digits
```java
s.chars().allMatch(Character::isDigit);
s.matches("\\d+");
```

### 10. Longest substring without repeating characters (sliding window)
```java
static int longestUniqueSubstring(String s) {
    Map<Character, Integer> lastSeen = new HashMap<>();
    int best = 0;
    for (int left = 0, right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (lastSeen.containsKey(c)) left = Math.max(left, lastSeen.get(c) + 1);
        lastSeen.put(c, right);
        best = Math.max(best, right - left + 1);
    }
    return best;
}
// "abcabcbb" → 3 ("abc")
```
O(n) time, O(k) space for k distinct characters.

---

## Part 2: Arrays & numbers

### 11. Find duplicates in an array
```java
static Set<Integer> duplicates(int[] nums) {
    Set<Integer> seen = new HashSet<>(), dups = new HashSet<>();
    for (int n : nums) if (!seen.add(n)) dups.add(n);     // add returns false if already present
    return dups;
}
```
O(n) time, O(n) space.

### 12. Second largest element (single pass)
```java
static int secondLargest(int[] nums) {
    int first = Integer.MIN_VALUE, second = Integer.MIN_VALUE;
    for (int n : nums) {
        if (n > first) { second = first; first = n; }
        else if (n > second && n != first) second = n;
    }
    if (second == Integer.MIN_VALUE) throw new IllegalArgumentException("no second largest");
    return second;
}
```
O(n) time, O(1) space. (Edge case: if `Integer.MIN_VALUE` itself is a legitimate answer, track it
with a boolean or use `Integer` with `null`.)

### 13. Two Sum: indices of two numbers adding up to a target
```java
static int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> indexOf = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        Integer j = indexOf.get(target - nums[i]);
        if (j != null) return new int[]{j, i};
        indexOf.put(nums[i], i);
    }
    return new int[0];
}
```
O(n) time vs O(n²) brute force.

### 14. Missing number in 1..n
```java
static int missing(int[] nums, int n) {       // nums has n-1 distinct values from 1..n
    int xor = 0;
    for (int i = 1; i <= n; i++) xor ^= i;
    for (int x : nums) xor ^= x;
    return xor;                               // XOR avoids overflow of the n(n+1)/2 sum approach
}
```

### 15. Move all zeros to the end (keep order)
```java
static void moveZeros(int[] a) {
    int write = 0;
    for (int x : a) if (x != 0) a[write++] = x;
    while (write < a.length) a[write++] = 0;
}
```
O(n) time, O(1) space.

### 16. Rotate an array right by k
```java
static void rotate(int[] a, int k) {
    k %= a.length;
    reverse(a, 0, a.length - 1);
    reverse(a, 0, k - 1);
    reverse(a, k, a.length - 1);
}
static void reverse(int[] a, int i, int j) {
    while (i < j) { int t = a[i]; a[i++] = a[j]; a[j--] = t; }
}
```

### 17. Fibonacci (iterative, memoized, stream)
```java
static long fib(int n) {                      // O(n) time, O(1) space
    long a = 0, b = 1;
    for (int i = 0; i < n; i++) { long t = a + b; a = b; b = t; }
    return a;
}

static final Map<Integer, Long> memo = new HashMap<>();
static long fibMemo(int n) {                  // naive recursion is O(2^n); memoization makes it O(n)
    if (n < 2) return n;
    Long cached = memo.get(n);
    if (cached != null) return cached;
    long v = fibMemo(n - 1) + fibMemo(n - 2);
    memo.put(n, v);
    return v;
}
// Note: memo.computeIfAbsent(n, k -> fibMemo(k-1) + fibMemo(k-2)) throws
// ConcurrentModificationException on HashMap (Java 9+): recursive modification inside compute.

Stream.iterate(new long[]{0, 1}, f -> new long[]{f[1], f[0] + f[1]})
      .limit(10).map(f -> f[0]).toList();     // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
```

### 18. Prime check & primes up to n (Sieve of Eratosthenes)
```java
static boolean isPrime(int n) {
    if (n < 2) return false;
    for (int i = 2; (long) i * i <= n; i++) if (n % i == 0) return false;
    return true;
}

static List<Integer> primesUpTo(int n) {
    boolean[] composite = new boolean[n + 1];
    List<Integer> primes = new ArrayList<>();
    for (int i = 2; i <= n; i++) {
        if (composite[i]) continue;
        primes.add(i);
        for (long j = (long) i * i; j <= n; j += i) composite[(int) j] = true;
    }
    return primes;
}
```
`isPrime`: O(√n). Sieve: O(n log log n).

### 19. Factorial (watch for overflow)
```java
static long factorial(int n) {                // overflows long after 20!
    long r = 1;
    for (int i = 2; i <= n; i++) r = Math.multiplyExact(r, i);
    return r;
}
static BigInteger bigFactorial(int n) {
    return IntStream.rangeClosed(2, n).mapToObj(BigInteger::valueOf)
                    .reduce(BigInteger.ONE, BigInteger::multiply);
}
```

### 20. Swap two numbers without a temp variable
```java
a = a + b; b = a - b; a = a - b;              // can overflow (though it still works with wrap-around)
a = a ^ b; b = a ^ b; a = a ^ b;              // XOR swap (breaks if a and b are the same variable/slot)
```

### 21. Armstrong number & reverse a number
```java
static boolean isArmstrong(int n) {           // 153 = 1³ + 5³ + 3³
    int digits = String.valueOf(n).length(), sum = 0;
    for (int x = n; x > 0; x /= 10) sum += (int) Math.pow(x % 10, digits);
    return sum == n;
}
static int reverseNumber(int n) {
    int r = 0;
    while (n != 0) { r = Math.addExact(Math.multiplyExact(r, 10), n % 10); n /= 10; }
    return r;
}
```

---

## Part 3: Collections & data structures

### 22. Balanced brackets
```java
static boolean isBalanced(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    Map<Character, Character> pairs = Map.of(')', '(', ']', '[', '}', '{');
    for (char c : s.toCharArray()) {
        if (pairs.containsValue(c)) stack.push(c);
        else if (pairs.containsKey(c)) {
            if (stack.isEmpty() || stack.pop() != pairs.get(c)) return false;
        }
    }
    return stack.isEmpty();
}
// isBalanced("{[()]}") → true, isBalanced("([)]") → false
```
O(n) time, O(n) space.

### 23. LRU cache from scratch (HashMap + doubly linked list)
Interviewers often forbid `LinkedHashMap` so you show the underlying structure.
```java
public class LRUCache<K, V> {
    private final class Node {
        K key; V value; Node prev, next;
        Node(K k, V v) { key = k; value = v; }
    }

    private final int capacity;
    private final Map<K, Node> map = new HashMap<>();
    private final Node head = new Node(null, null);   // sentinel: most recent after head
    private final Node tail = new Node(null, null);   // sentinel: least recent before tail

    public LRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail; tail.prev = head;
    }

    public V get(K key) {
        Node n = map.get(key);
        if (n == null) return null;
        moveToFront(n);
        return n.value;
    }

    public void put(K key, V value) {
        Node n = map.get(key);
        if (n != null) { n.value = value; moveToFront(n); return; }
        if (map.size() == capacity) {
            Node lru = tail.prev;
            unlink(lru);
            map.remove(lru.key);
        }
        n = new Node(key, value);
        map.put(key, n);
        addFront(n);
    }

    private void moveToFront(Node n) { unlink(n); addFront(n); }
    private void unlink(Node n) { n.prev.next = n.next; n.next.prev = n.prev; }
    private void addFront(Node n) {
        n.next = head.next; n.prev = head;
        head.next.prev = n; head.next = n;
    }
}
```
O(1) `get` and `put`. Follow-up "make it thread-safe": wrap methods in `synchronized` or a
`ReentrantLock` (both `get` and `put` mutate the list), or use Caffeine in production.

### 24. Generic stack backed by an array
```java
public class ArrayStack<T> implements Iterable<T> {
    private Object[] items = new Object[8];
    private int size;

    public void push(T item) {
        if (size == items.length) items = Arrays.copyOf(items, size * 2);
        items[size++] = item;
    }

    @SuppressWarnings("unchecked")
    public T pop() {
        if (size == 0) throw new NoSuchElementException("stack is empty");
        T item = (T) items[--size];
        items[size] = null;                           // avoid loitering (memory leak)
        return item;
    }

    @SuppressWarnings("unchecked")
    public T peek() {
        if (size == 0) throw new NoSuchElementException("stack is empty");
        return (T) items[size - 1];
    }

    public boolean isEmpty() { return size == 0; }

    @Override public Iterator<T> iterator() {
        return new Iterator<>() {
            int i = size;
            public boolean hasNext() { return i > 0; }
            @SuppressWarnings("unchecked") public T next() {
                if (i == 0) throw new NoSuchElementException();
                return (T) items[--i];
            }
        };
    }
}
```
Talking points: why `Object[]` (no generic arrays), why null out popped slots, amortized O(1)
`push`.

### 25. Simplified HashMap (show you understand buckets)
```java
public class SimpleHashMap<K, V> {
    private static final class Entry<K, V> {
        final K key; V value; Entry<K, V> next;
        Entry(K key, V value, Entry<K, V> next) { this.key = key; this.value = value; this.next = next; }
    }

    private Entry<K, V>[] table = newTable(16);
    private int size;

    @SuppressWarnings("unchecked")
    private static <K, V> Entry<K, V>[] newTable(int n) { return (Entry<K, V>[]) new Entry[n]; }

    private int index(Object key, int length) {
        int h = (key == null) ? 0 : key.hashCode();
        h ^= (h >>> 16);
        return h & (length - 1);
    }

    public V get(K key) {
        for (Entry<K, V> e = table[index(key, table.length)]; e != null; e = e.next)
            if (Objects.equals(e.key, key)) return e.value;
        return null;
    }

    public V put(K key, V value) {
        int i = index(key, table.length);
        for (Entry<K, V> e = table[i]; e != null; e = e.next) {
            if (Objects.equals(e.key, key)) { V old = e.value; e.value = value; return old; }
        }
        table[i] = new Entry<>(key, value, table[i]);
        if (++size > table.length * 3 / 4) resize();
        return null;
    }

    private void resize() {
        Entry<K, V>[] old = table;
        table = newTable(old.length * 2);
        for (Entry<K, V> head : old) {
            for (Entry<K, V> e = head; e != null; ) {
                Entry<K, V> next = e.next;
                int i = index(e.key, table.length);
                e.next = table[i];
                table[i] = e;
                e = next;
            }
        }
    }

    public int size() { return size; }
}
```

### 26. Sort a map by value
```java
Map<String, Integer> scores = Map.of("ana", 90, "raj", 75, "li", 95);

LinkedHashMap<String, Integer> sorted = scores.entrySet().stream()
    .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
    .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue,
                              (a, b) -> a, LinkedHashMap::new));      // keep sorted order
// {li=95, ana=90, raj=75}
```

### 27. Top-K frequent words
```java
static List<String> topK(String[] words, int k) {
    Map<String, Integer> count = new HashMap<>();
    for (String w : words) count.merge(w, 1, Integer::sum);

    PriorityQueue<String> heap = new PriorityQueue<>(           // min-heap by frequency
        Comparator.<String>comparingInt(count::get).thenComparing(Comparator.reverseOrder()));
    for (String w : count.keySet()) {
        heap.offer(w);
        if (heap.size() > k) heap.poll();
    }
    List<String> result = new ArrayList<>(heap);
    result.sort(Comparator.<String>comparingInt(count::get).reversed().thenComparing(Comparator.naturalOrder()));
    return result;
}
```
O(n log k).

---

## Part 4: Streams (Employee dataset)

```java
record Employee(int id, String name, String dept, double salary, int age, String gender) {}

List<Employee> emps = List.of(
    new Employee(1, "Ana",   "ENG",   120_000, 30, "F"),
    new Employee(2, "Raj",   "ENG",    95_000, 25, "M"),
    new Employee(3, "Li",    "HR",     70_000, 40, "F"),
    new Employee(4, "Bo",    "SALES",  80_000, 35, "M"),
    new Employee(5, "Maria", "SALES",  88_000, 29, "F"));
```

### 28. Count employees by gender / by department
```java
Map<String, Long> byGender = emps.stream().collect(Collectors.groupingBy(Employee::gender, Collectors.counting()));
Map<String, Long> byDept   = emps.stream().collect(Collectors.groupingBy(Employee::dept, Collectors.counting()));
```

### 29. Average salary per department
```java
Map<String, Double> avg = emps.stream()
    .collect(Collectors.groupingBy(Employee::dept, Collectors.averagingDouble(Employee::salary)));
```

### 30. Highest-paid employee overall and per department
```java
Optional<Employee> top = emps.stream().max(Comparator.comparingDouble(Employee::salary));

Map<String, Employee> topByDept = emps.stream().collect(Collectors.toMap(
    Employee::dept, Function.identity(),
    BinaryOperator.maxBy(Comparator.comparingDouble(Employee::salary))));
```

### 31. Nth highest salary
```java
static Optional<Double> nthHighest(List<Employee> emps, int n) {
    return emps.stream().map(Employee::salary).distinct()
               .sorted(Comparator.reverseOrder()).skip(n - 1).findFirst();
}
```

### 32. Names of employees older than 28, sorted by salary desc, joined
```java
String names = emps.stream()
    .filter(e -> e.age() > 28)
    .sorted(Comparator.comparingDouble(Employee::salary).reversed())
    .map(Employee::name)
    .collect(Collectors.joining(", "));     // "Ana, Maria, Bo, Li"
```

### 33. Partition into salary > 85k and ≤ 85k
```java
Map<Boolean, List<String>> split = emps.stream().collect(Collectors.partitioningBy(
    e -> e.salary() > 85_000, Collectors.mapping(Employee::name, Collectors.toList())));
```

### 34. Youngest employee in each department
```java
Map<String, Optional<Employee>> youngest = emps.stream().collect(Collectors.groupingBy(
    Employee::dept, Collectors.minBy(Comparator.comparingInt(Employee::age))));
```

### 35. Total, max, min, average salary in one pass
```java
DoubleSummaryStatistics s = emps.stream().mapToDouble(Employee::salary).summaryStatistics();
// s.getSum(), s.getMax(), s.getMin(), s.getAverage(), s.getCount()
```

### 36. Misc stream one-liners
```java
List<Integer> nums = List.of(5, 3, 8, 3, 1, 8, 9);

nums.stream().filter(n -> n % 2 == 0).toList();                       // evens
nums.stream().distinct().sorted().toList();                           // unique, sorted
nums.stream().sorted(Comparator.reverseOrder()).limit(3).toList();    // top 3
nums.stream().mapToInt(Integer::intValue).max().orElseThrow();        // max
nums.stream().collect(Collectors.groupingBy(Function.identity(), Collectors.counting()))
    .entrySet().stream().filter(e -> e.getValue() > 1).map(Map.Entry::getKey).toList();   // duplicates
nums.stream().map(String::valueOf).collect(Collectors.joining("-"));  // "5-3-8-3-1-8-9"
nums.stream().reduce(Integer::sum).orElse(0);                         // sum
Stream.of("apple", "kiwi", "banana").collect(Collectors.toMap(Function.identity(), String::length));
Stream.of("apple", "kiwi", "banana").max(Comparator.comparingInt(String::length)).get();   // longest
List.of(List.of(1, 2), List.of(3, 4)).stream().flatMap(List::stream).toList();            // flatten
IntStream.rangeClosed(1, 5).boxed().collect(Collectors.toList());                         // [1..5]
nums.stream().anyMatch(n -> n > 8);                                   // true
nums.stream().skip(nums.size() - 1).findFirst();                      // last element
```

---

## Part 5: Concurrency

### 37. Print odd and even numbers with two threads, in order
```java
public class OddEvenPrinter {
    private final int max;
    private int current = 1;
    private final Object lock = new Object();

    OddEvenPrinter(int max) { this.max = max; }

    void print(boolean printOdd) {
        synchronized (lock) {
            while (current <= max) {
                if ((current % 2 == 1) != printOdd) {     // not my turn
                    try { lock.wait(); }
                    catch (InterruptedException e) { Thread.currentThread().interrupt(); return; }
                    continue;
                }
                System.out.println(Thread.currentThread().getName() + ": " + current++);
                lock.notifyAll();
            }
        }
    }

    public static void main(String[] args) {
        OddEvenPrinter p = new OddEvenPrinter(10);
        new Thread(() -> p.print(true), "odd").start();
        new Thread(() -> p.print(false), "even").start();
    }
}
```
Alternatives: two `Semaphore`s (odd starts with 1 permit, even with 0, each releases the other), or
`ReentrantLock` with two `Condition`s.

```java
// Semaphore version
Semaphore oddTurn = new Semaphore(1), evenTurn = new Semaphore(0);
Thread odd = new Thread(() -> {
    for (int i = 1; i <= 9; i += 2) {
        oddTurn.acquireUninterruptibly(); System.out.println(i); evenTurn.release();
    }
});
Thread even = new Thread(() -> {
    for (int i = 2; i <= 10; i += 2) {
        evenTurn.acquireUninterruptibly(); System.out.println(i); oddTurn.release();
    }
});
odd.start(); even.start();
```

### 38. Producer–consumer with `BlockingQueue`
```java
public class ProducerConsumer {
    private static final int POISON_PILL = -1;

    public static void main(String[] args) throws InterruptedException {
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(5);

        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) {
                    queue.put(i);                         // blocks when full
                    System.out.println("Produced " + i);
                }
                queue.put(POISON_PILL);                   // signal completion
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread consumer = new Thread(() -> {
            try {
                while (true) {
                    int item = queue.take();              // blocks when empty
                    if (item == POISON_PILL) break;
                    System.out.println("Consumed " + item);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start(); consumer.start();
        producer.join(); consumer.join();
    }
}
```
Follow-up "do it without `BlockingQueue`": see the `wait`/`notifyAll` `BoundedBuffer` in
[chapter 13](./13-concurrency.md), section 9.

### 39. Write code that deadlocks, then fix it
```java
Object a = new Object(), b = new Object();

Thread t1 = new Thread(() -> {
    synchronized (a) {
        sleep(50);
        synchronized (b) { System.out.println("t1 done"); }
    }
});
Thread t2 = new Thread(() -> {
    synchronized (b) {                       // opposite order → deadlock
        sleep(50);
        synchronized (a) { System.out.println("t2 done"); }
    }
});
// Fix: make t2 lock 'a' then 'b' as well (consistent global lock order).
// sleep(ms) is a helper wrapping Thread.sleep and handling InterruptedException.
```

Bank transfer with lock ordering:
```java
void transfer(Account from, Account to, BigDecimal amount) {
    Account first = from.id() < to.id() ? from : to;
    Account second = first == from ? to : from;
    synchronized (first) {
        synchronized (second) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}
```

### 40. Thread-safe counter: three ways
```java
class SyncCounter   { private int c;  synchronized void inc() { c++; } synchronized int get() { return c; } }
class AtomicCounter { private final AtomicInteger c = new AtomicInteger(); void inc() { c.incrementAndGet(); } int get() { return c.get(); } }
class AdderCounter  { private final LongAdder c = new LongAdder(); void inc() { c.increment(); } long get() { return c.sum(); } }

// Verify
var counter = new AtomicCounter();
try (var pool = Executors.newFixedThreadPool(8)) {
    for (int i = 0; i < 10_000; i++) pool.submit(counter::inc);
}                                            // close() waits for completion (Java 19+)
System.out.println(counter.get());           // 10000
```

### 41. Run tasks in parallel and combine results
```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    CompletableFuture<String> user  = CompletableFuture.supplyAsync(() -> fetch("user"), executor);
    CompletableFuture<String> order = CompletableFuture.supplyAsync(() -> fetch("orders"), executor);
    String combined = user.thenCombine(order, (u, o) -> u + " | " + o)
                          .orTimeout(2, TimeUnit.SECONDS)
                          .exceptionally(ex -> "fallback")
                          .join();
    System.out.println(combined);
}

// N tasks → list of results
List<CompletableFuture<Integer>> futures = ids.stream()
    .map(id -> CompletableFuture.supplyAsync(() -> load(id), executor)).toList();
List<Integer> results = CompletableFuture.allOf(futures.toArray(CompletableFuture[]::new))
    .thenApply(v -> futures.stream().map(CompletableFuture::join).toList())
    .join();
```

### 42. Wait for N workers to finish (`CountDownLatch`)
```java
int workers = 3;
CountDownLatch done = new CountDownLatch(workers);
ExecutorService pool = Executors.newFixedThreadPool(workers);
for (int i = 0; i < workers; i++) {
    int id = i;
    pool.submit(() -> {
        try { System.out.println("worker " + id + " working"); }
        finally { done.countDown(); }          // always count down, even on failure
    });
}
done.await();
System.out.println("all workers finished");
pool.shutdown();
```

### 43. Print numbers 1..N with three threads in round-robin (T1: 1, T2: 2, T3: 3, T1: 4 …)
```java
class RoundRobin {
    private final int n, threads;
    private int current = 1;
    RoundRobin(int n, int threads) { this.n = n; this.threads = threads; }

    synchronized void run(int id) {            // id = 0, 1, 2
        while (current <= n) {
            if ((current - 1) % threads != id) {
                try { wait(); } catch (InterruptedException e) { Thread.currentThread().interrupt(); return; }
                continue;
            }
            System.out.println("T" + (id + 1) + ": " + current++);
            notifyAll();
        }
        notifyAll();                           // release threads still waiting at the end
    }
}

RoundRobin rr = new RoundRobin(9, 3);
for (int i = 0; i < 3; i++) { int id = i; new Thread(() -> rr.run(id)).start(); }
```

---

## Part 6: Design questions you may be asked to code

| Ask | Where |
|---|---|
| Thread-safe singleton (all variants) | [Chapter 17](./17-design-patterns.md), section 2 |
| Immutable class | [Chapter 08](./08-object-equals-hashcode-records.md), section 8 |
| Builder pattern | [Chapter 17](./17-design-patterns.md), section 2 |
| `equals`/`hashCode` for an entity | [Chapter 08](./08-object-equals-hashcode-records.md), section 5 |
| Custom exception hierarchy | [Chapter 09](./09-exceptions.md), section 5 |
| Custom `AutoCloseable` resource | [Chapter 09](./09-exceptions.md), section 3 |
| Sort objects by several fields | [Chapter 08](./08-object-equals-hashcode-records.md), section 6 |
| Sealed type + exhaustive switch | [Chapter 16](./16-modern-java-features.md), section 4 |

## Practice plan

1. Day 1: Part 1 (strings) and Part 2 (arrays). Write each solution from memory in `jshell`.
2. Day 2: Part 3. Implement the LRU cache and `SimpleHashMap` without looking.
3. Day 3: Part 4. Rewrite each stream solution, then explain each collector out loud.
4. Day 4: Part 5. Run the concurrency programs, break them on purpose (remove `volatile`, swap lock
   order), and watch what happens.
