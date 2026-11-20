# Module 4: Collections Framework

## 1. Java Collections Framework Overview

### Core Framework Hierarchy & Implementations
The Java Collections Framework (`java.util`) provides a unified architecture for storing, manipulating, and accessing data collections. The core root interfaces are `Collection` (which branches into `List`, `Set`, `Queue`, and `Deque`) and `Map` (a standalone key-value mapping interface).

#### Summary of Core Interfaces & Implementations

```
java.util.Collection
 ├── List
 │    ├── ArrayList (Dynamic resizable array)
 │    ├── LinkedList (Doubly-linked list, also implements Deque)
 │    └── CopyOnWriteArrayList (java.util.concurrent: Snapshot copy-on-write array)
 ├── Set
 │    ├── HashSet (Backed by HashMap, O(1), unordered)
 │    ├── LinkedHashSet (HashMap + doubly-linked list, insertion-ordered)
 │    └── TreeSet (NavigableSet/SortedSet, backed by Red-Black Tree, O(log n))
 └── Queue & Deque
      ├── PriorityQueue (Binary heap priority queue)
      └── ArrayDeque (Resizable array double-ended queue, LIFO/FIFO)

java.util.Map
 ├── HashMap (Hash table of buckets, O(1), unordered)
 ├── LinkedHashMap (Hash table + doubly-linked list, insertion/access order)
 ├── TreeMap (NavigableMap/SortedMap, Red-Black Tree, O(log n) sorted keys)
 ├── ConcurrentHashMap (java.util.concurrent: Lock-free reads, CAS + node-level locks)
 └── WeakHashMap (Hash table with WeakReference keys for memory-sensitive caching)
```

#### Detailed Breakdown of When to Use and Avoid

| Implementation | Underlying Data Structure | When to Use | When to Avoid |
| :--- | :--- | :--- | :--- |
| **`ArrayList`** | Resizable array | Default choice for 95% of List use cases; fast random access ($O(1)$) and appending. | High-frequency insertions/deletions in the middle of massive lists ($O(n)$ element shifting). |
| **`LinkedList`** | Doubly-linked list | High-frequency insertions/removals strictly at head/tail when used as a queue/deque. | Almost always for random access; poor CPU cache locality makes it slower than `ArrayList` in practice. |
| **`CopyOnWriteArrayList`** | Snapshot cloned array | Multi-threaded read-heavy workloads (e.g., event listeners, 99% reads, 1% writes). | Write-heavy environments; copying the entire array on every write creates massive GC pressure. |
| **`HashSet`** | Hash table (`HashMap`) | Default Set choice; stores unique elements with blazing $O(1)$ add, remove, and lookup. | When element iteration order must be predictable or sorted. |
| **`LinkedHashSet`** | Hash table + Doubly-linked list | Uniqueness required while strictly preserving insertion order. | When insertion-order tracking memory overhead is unacceptable and order is irrelevant. |
| **`TreeSet`** | Red-Black Tree (`TreeMap`) | Elements must be maintained in natural sorted order or sorted via a custom `Comparator`. | Simple uniqueness checks where sorting is unneeded; operations are $O(\log n)$ instead of $O(1)$. |
| **`PriorityQueue`** | Binary Min-Heap | Priority processing where highest/lowest priority element must be dequeued first ($O(\log n)$). | General FIFO queue or random-access indexed lookups. |
| **`ArrayDeque`** | Resizable circular array | Default modern choice for Stacks (LIFO) and Queues (FIFO); faster than `Stack` and `LinkedList`. | Concurrent access without external synchronization, or random index access. |
| **`HashMap`** | Hash table (array + bins) | Default Map for general-purpose key-value lookups with $O(1)$ average time complexity. | Multi-threaded environments (thread-unsafe) or when key order must be maintained. |
| **`LinkedHashMap`** | Hash table + Doubly-linked list | Key-value mapping that preserves insertion order or access order (ideal for LRU caches). | Memory-constrained systems where pointer overhead per entry cannot be justified. |
| **`TreeMap`** | Red-Black Tree | Key-value dictionary requiring keys to be persistently sorted or navigable. | Performance-critical lookups where $O(\log n)$ tree traversal overhead is prohibitive. |
| **`ConcurrentHashMap`** | Lock-free reads + CAS / Node locks | High-concurrency enterprise applications requiring thread-safe, high-throughput key-value access. | Applications needing null keys or null values, or simple single-threaded scripts. |

---

## 2. Lists & Sequential Structures

### ArrayList vs LinkedList Architecture & CPU Cache Locality

#### Architecture & Complexity Comparison
| Feature | `ArrayList` | `LinkedList` |
| :--- | :--- | :--- |
| **Memory Allocation** | Contiguous blocks of heap memory | Scattered memory (nodes allocated independently across heap) |
| **Random Access (`get`)** | $O(1)$ instant access via direct index pointer arithmetic | $O(n)$ must traverse node links from head or tail |
| **Insert / Delete at End** | $O(1)$ amortized (occasional $O(n)$ array resize) | $O(1)$ updates tail pointers directly |
| **Insert / Delete at Middle** | $O(n)$ requires shifting subsequent elements | $O(1)$ pointer update once node is located (finding node is $O(n)$) |
| **Memory Overhead** | Low (single array holding object references) | High (every element creates a `Node` object with 2 pointers) |

#### The Senior Edge: Mechanical Sympathy & CPU Cache Locality
Modern hardware performance is heavily dominated by CPU L1/L2/L3 cache architectures.
1. **Contiguous Memory Prefetching:** Because `ArrayList` stores references in a contiguous block of memory, the CPU hardware prefetcher loads chunks of the array into L1/L2 cache lines automatically. Iterating through an `ArrayList` hits blazing-fast cache lines with almost zero CPU stall cycles.
2. **Heap Fragmentation & Cache Misses:** `LinkedList` elements reside in distinct `Node` objects scattered across heap memory. Traversing a `LinkedList` forces the CPU into continuous cache misses, stalling execution while fetching addresses from main RAM (which is hundreds of times slower than L1 cache).
3. **Garbage Collection Pressure:** Every insertion in `LinkedList` allocates a new `Node` wrapper (object header + previous pointer + next pointer + data pointer). In high-throughput systems, this churns the young generation heap, causes heap fragmentation, and forces frequent Garbage Collector stop-the-world pauses.

---

### List to Set Conversion & Deduplication
Converting a `List` with duplicates to a `Set` eliminates duplicates automatically according to the set implementation's equality mechanics.

```java
List<String> list = Arrays.asList("A", "B", "A", "C", "B");

// 1. Using HashSet: Removes duplicates, no order guarantees (fastest O(1))
Set<String> hashSet = new HashSet<>(list);
System.out.println(hashSet); // [A, B, C] (order unpredictable)

// 2. Using LinkedHashSet: Removes duplicates while strictly preserving insertion order
Set<String> linkedHashSet = new LinkedHashSet<>(list);
System.out.println(linkedHashSet); // [A, B, C] (preserves insertion sequence)

// 3. Using TreeSet: Removes duplicates and sorts elements in natural or comparator order
Set<String> treeSet = new TreeSet<>(list);
System.out.println(treeSet); // [A, B, C] (alphabetically/numerically sorted)

// 4. Using Java 8 Streams:
Set<String> streamSet = list.stream().collect(Collectors.toSet()); // Default HashSet

// Custom collection target via Stream:
Set<String> orderedStreamSet = list.stream()
    .collect(Collectors.toCollection(LinkedHashSet::new));
```

---

### Real-World Applications of Doubly & Circular Linked Lists

#### 1. Doubly Linked List (DLL)
A DLL allows bidirectional traversal (`prev` and `next`) and $O(1)$ node removal once a node's reference is known:
- **Browser History Navigation:** Current page is a node; clicking "Back" navigates `node.prev`; clicking "Forward" navigates `node.next`.
- **LRU (Least Recently Used) Cache:** Industry systems (e.g., Redis, internal caches) pair a `HashMap` (for $O(1)$ key lookup) with a Doubly Linked List (for $O(1)$ eviction and reordering). When an element is accessed, its node is unlinked via its `prev` and `next` pointers in $O(1)$ time and moved to the head (most recently used).
- **Undo / Redo Buffers:** Text editors store state changes in a DLL where Ctrl+Z traverses backwards and Ctrl+Y traverses forward.
- **Media Player Playlists:** Seamless track advancement and previous-track playback.

#### 2. Circular Linked List (CLL) & Ring Buffers
In a CLL, the `tail.next` pointer references the `head` node, forming an infinite loop:
- **OS CPU Round-Robin Scheduling:** The operating system places active threads/processes in a circular queue. The scheduler grants a time quantum (e.g., 10ms) to the current process, advances `node.next`, and loops back to the start without checking for `null` bounds or resetting pointers.
- **Turn-Based Systems & Multiplayer Games:** Managing player turns in sequence ($P_1 \rightarrow P_2 \rightarrow P_3 \rightarrow P_1$). If a player disconnects, updating pointers maintains a continuous ring.
- **Media Streaming Ring Buffers:** Live video/audio buffering allocates a fixed-size circular buffer. Incoming frames continuously overwrite the oldest frames at the head once full, enforcing bounded memory usage.

---

## 3. Map Internals & Custom Key Contracts

### Map Implementations Comparison (HashMap vs Hashtable vs ConcurrentHashMap)

| Feature | `HashMap` | `Hashtable` | `ConcurrentHashMap` |
| :--- | :--- | :--- | :--- |
| **Thread Safety** | ❌ Not thread-safe | ✅ Thread-safe (coarse synchronized methods) | ✅ Thread-safe (lock-free reads, CAS + node-level locks) |
| **Performance** | High in single-threaded scenarios | Poor; entire map is locked on every operation | High; locks are isolated to individual bucket nodes |
| **Null Keys / Values** | 1 `null` key allowed; multiple `null` values allowed | Prohibits `null` keys and `null` values (throws NPE) | Prohibits `null` keys and `null` values (throws NPE) |
| **Iteration Policy** | Fail-fast (`ConcurrentModificationException`) | Fail-fast (`ConcurrentModificationException`) | Weakly consistent (fail-safe; never throws CME) |
| **Framework Era** | Java 1.2 Collections Framework | Java 1.0 Legacy class (obsolete) | Java 5 (`java.util.concurrent`), revamped in Java 8 |
| **Primary Use Case** | Single-threaded applications, general caching | Deprecated historical codebases | High-throughput, multi-threaded enterprise services |

---

### HashMap Internal Working in Java 8+
A `HashMap` stores key-value pairs in an internal array of nodes: `Node<K,V>[] table`.

```
Index calculation:
int hash = (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16); // Supplemental hash spreading
int bucketIndex = (n - 1) & hash;                                  // Bitwise modulo where n is power of 2
```

#### Collision Resolution & Treeification (Java 8+)
- **Bucket Chaining:** When different keys produce the same bucket index, entries are chained.
- **Java 7 vs Java 8:** In Java 7, buckets were purely singly linked lists ($O(n)$ search). In Java 8+, once a bucket reaches `TREEIFY_THRESHOLD = 8` entries and total map capacity $\ge 64$, the linked list transforms into a balanced **Red-Black Tree** (`TreeNode<K,V>`).
- **DoS Mitigation:** Treeification guarantees worst-case lookup improves from $O(n)$ to $O(\log n)$, mitigating Denial of Service (DoS) attacks where malicious actors flood a service with colliding hash keys.
- **Untreeify:** When bucket items drop below `UNTREEIFY_THRESHOLD = 6` during resizing, the tree converts back into a singly linked list.

#### Resizing & Dynamic Rehashing
- **Capacity & Load Factor:** Default initial capacity is `16`, and default load factor is `0.75`.
- **Threshold:** When item count exceeds `capacity * loadFactor` ($16 \times 0.75 = 12$), `resize()` executes.
- **Doubling & Bitwise Split:** Capacity doubles (e.g., from 16 to 32). In Java 8, rehashing does not recompute full hash calculations; it checks the additional high bit (`hash & oldCap`). Nodes either remain at the same index or move to `index + oldCap`.

#### Architectural Hazards & Senior Best Practices
- **Pre-sizing Optimization:** If an application expects to store 10,000 items, constructing `new HashMap<>()` causes ~10 expensive resize operations. Pre-size the map:
  $$\text{Target Capacity} = \left\lceil \frac{\text{Expected Items}}{\text{Load Factor}} \right\rceil + 1 = \left\lceil \frac{10000}{0.75} \right\rceil + 1 = 13335 \rightarrow 16384 \text{ (next power of 2)}$$
- **Concurrency Danger:** `HashMap` is inherently thread-unsafe. In Java 7, concurrent resize operations caused circular linked lists leading to infinite CPU loops (100% CPU lockup). In Java 8+, race conditions cause silent data overwrites and node corruption.

---

### HashMap vs LinkedHashMap

| Feature | `HashMap` | `LinkedHashMap` |
| :--- | :--- | :--- |
| **Iteration Order** | Unpredictable; order can shift during bucket rehashing | Predictable; strictly maintains **Insertion Order** or **Access Order** |
| **Internal Architecture** | Hash table (array of bucket nodes) | Hash table + Doubly-linked list running through all entries |
| **Performance** | Slightly faster for put/get (no pointer management) | Slightly slower due to updating `before` and `after` pointers |
| **Memory Consumption** | Lower memory footprint per entry | Higher memory footprint (two additional reference pointers per node) |
| **Primary Use Cases** | General lookups, high-speed key-value caches | LRU (Least Recently Used) cache, ordered serialization |

#### LRU Cache Implementation with LinkedHashMap
`LinkedHashMap` provides a built-in eviction hook via `removeEldestEntry()`. When initialized in access-order mode (`new LinkedHashMap<>(capacity, 0.75f, true)`), accessing an entry moves it to the tail. When capacity is exceeded, the eldest entry at the head is automatically evicted.

---

### Custom Objects as Map Keys & The Contract Rules
To safely use a domain object as a `HashMap` key, strict adherence to `equals()` and `hashCode()` contracts is mandatory.

#### The Fundamental Contract
1. **Consistency:** If `k1.equals(k2)`, then `k1.hashCode() == k2.hashCode()` must be strictly true.
2. **Default Object Behavior Hazard:** If methods are unoverridden, the class inherits identity comparison from `Object`. Two distinct instances with identical business data will produce different hash codes and fail `equals()`, making retrieval with a new instance return `null`.

#### The Immutability Trap
All fields utilized in `hashCode()` and `equals()` calculations must be strictly **immutable (`final`)**.
- **The Catastrophe:** If an object is inserted with `id = 5` into Bucket 2, and another thread mutates `object.setId(10)`, calling `map.get(object)` recalculates the hash code based on `10`, routing to Bucket 8.
- **The Result:** Bucket 8 is empty, returning `null`. The original entry remains permanently trapped in Bucket 2. It can never be fetched or removed, causing an undetected permanent **memory leak**.

---

## 4. Concurrent Collections Architecture

### ConcurrentHashMap Internals (Java 8+ CAS & Node-Level Locking)
In modern Java (Java 8+), `ConcurrentHashMap` completely replaced the Java 7 "Segment Striping" model (which locked 1 of 16 segments) with lock-free reads, hardware-level CAS operations, and node-level locking.

#### Internal Mechanics of Read/Write Operations
1. **Lock-Free Reads (Zero Blocking):** `get()` operations never acquire a lock. The internal `Node` class marks its value (`val`) and next pointer (`next`) as `volatile`. Hardware cache coherence ensures reading threads immediately observe memory updates from concurrent writes.
2. **Lock-Free Writes via CAS (Empty Buckets):** When calling `put()`, the map calculates the bucket index. If the bucket is empty (`tabAt(i) == null`), it does not synchronize. It attempts to insert the node using CPU-level **Compare-And-Swap (`casTabAt`)**. If multiple threads compete for the empty bucket, only one CAS succeeds; other threads retry gracefully without thread suspension.
3. **Fine-Grained Node-Level Locking (Populated Buckets):** If the bucket already contains data (linked list or red-black tree collision), CAS cannot safely update the structure. The map synchronizes **only on the head node** of that specific bucket (`synchronized (f)`). All other buckets remain unlocked, allowing concurrent writes across distinct buckets without contention.

---

### ConcurrentHashMap vs Collections.synchronizedMap()

#### Coarse-Grained Locking Bottleneck
- `Collections.synchronizedMap()` is a legacy decorator that wraps a standard `HashMap` with `synchronized(this)` on every method (`get()`, `put()`, `remove()`).
- In high-throughput multi-threaded environments, reads serialize behind writes, and reads block other reads, causing severe thread queuing and CPU bottlenecks.

#### Iteration Semantics (Fail-Fast vs Weakly Consistent)
- **`synchronizedMap` (Fail-Fast):** Iterators are fail-fast. If another thread modifies the map during iteration, it throws `ConcurrentModificationException`. To safely iterate, the caller must manually lock the entire map object using `synchronized(map)`.
- **`ConcurrentHashMap` (Weakly Consistent):** Iterators are weakly consistent (fail-safe). They operate on a live snapshot of the bucket chain without locking. Modifications occurring during iteration do not throw `ConcurrentModificationException`; new elements may or may not be reflected in the current loop.

#### The Null Value Trap & The Ambiguity Problem
`ConcurrentHashMap` strictly prohibits `null` keys and `null` values (attempting to insert `null` throws `NullPointerException`), whereas `HashMap` permits one `null` key and multiple `null` values.
- **The Ambiguity:** In single-threaded `HashMap`, if `map.get(key)` returns `null`, it can mean:
  1. The key does not exist.
  2. The key exists and its value is explicitly set to `null`.
- In a single-threaded program, `map.containsKey(key)` resolves this ambiguity safely.
- **The Concurrency Race Condition:** In multi-threaded code, two-step checks (`get()` followed by `containsKey()`) are not atomic. Between `get(key)` returning `null` and the subsequent `containsKey(key)` check, another thread could insert, mutate, or remove the key.
- Doug Lea explicitly banned `null` in concurrent collections to prevent this fatal ambiguity race condition.

---

### Cooperative Resizing in ConcurrentHashMap
In standard `HashMap`, resizing is single-threaded and susceptible to race conditions. `ConcurrentHashMap` introduces **Cooperative Resizing** (Concurrent Migration) to rehash without freezing map operations.

#### Step-by-Step Mechanics
1. **Stride Partitioning:** When a thread triggers an array resize (doubling capacity), it allocates the new array and divides the old array into logical chunks called **strides** (minimum 16 buckets per stride).
2. **ForwardingNode Signposts:** When a thread finishes migrating a bucket's nodes to the new array, it places a `ForwardingNode` in the old bucket header with a fixed negative hash (`MOVED = -1`). This node contains a pointer to the new table.
3. **The Helper Swarm:** When another writer thread attempts to insert into a bucket containing a `ForwardingNode`, it recognizes that a resize is underway. Instead of blocking, the thread actively joins the migration swarm, claims an unmigrated stride, and helps move buckets to the new table.
4. **Lock-Free Reader Routing:** Readers encountering an unmigrated bucket read from the old table lock-free. Readers encountering a `ForwardingNode` follow the pointer to read directly from the new array. Readers are never blocked and never forced to participate in resizing.

---

### CopyOnWriteArrayList Mechanics & Trade-offs
`java.util.concurrent.CopyOnWriteArrayList` is a thread-safe `List` implementation designed for read-intensive architectures.

#### Write Operation Mechanics
1. The writer thread acquires an internal mutual exclusion lock (`ReentrantLock`).
2. It allocates a brand-new array with capacity `current.length + 1`.
3. It copies all existing elements from the old array into the new array.
4. It appends the new element into the final slot.
5. It atomically swaps the internal `volatile Object[] array` reference to the new array.
6. The lock is released.

#### Read Operation Mechanics (100% Lock-Free)
Readers access the current `volatile` array reference directly without acquiring locks. Iterators traverse the immutable snapshot that existed at the exact moment the iterator was instantiated. Mutations during iteration never trigger `ConcurrentModificationException`.

#### Catastrophic Anti-Patterns & Optimal Use Case
- **Terrible Scenario: Write-Heavy Workloads:** If a list contains 100,000 items and multiple threads perform frequent writes, every write allocates a 100,001-element array and copies 100,000 references. This causes extreme CPU spikes, Young-Gen exhaustion, and severe GC stop-the-world pauses.
- **Terrible Scenario: Strict Read Consistency:** Provides **eventual consistency**. Iterators read the historical snapshot and may process stale data while writes occur concurrently.
- **Optimal Enterprise Use Case:** Read-heavy, low-write, small-sized collections such as **Observer/Listener registries** (e.g., 5 UI listeners registered at boot, read millions of times per day).

---

### Simultaneous Read-Write Scenarios (HashMap vs ConcurrentHashMap)

```
Scenario: Thread A writes to Key X while Thread B reads from Key X simultaneously
```

#### Standard HashMap Behavior (The Disaster)
- **Visibility Failure:** Because `HashMap` fields are not `volatile`, Thread B may read stale cached values or partially updated memory due to CPU caching and instruction reordering.
- **Structural Corruption:** If Thread A's write triggers a resize while Thread B is iterating, pointers are corrupted. In Java 7, this produced infinite circular loops locking CPUs at 100%. In Java 8+, it causes data corruption.
- **Iterator Crash:** Thread B's iterator detects `modCount` changes and crashes with `ConcurrentModificationException`.

#### ConcurrentHashMap Behavior (The Enterprise Standard)
- **Lock-Free Immediate Read Visibility:** Bucket node `val` and `next` pointers are declared `volatile`. Thread B's read executes lock-free and observes Thread A's completed write immediately across CPU caches.
- **Granular Write Isolation:** Thread A only acquires a lock on the specific bucket head node (or executes CAS if empty). All other map buckets remain fully accessible.
- **Safe Weakly Consistent Iteration:** If Thread B is iterating, it traverses the snapshot of the bucket chain and finishes safely without throwing exceptions.

---

## 5. Specialized Collections & Memory Management

### WeakHashMap Architecture & GC Mechanics
`java.util.WeakHashMap` is a hash table implementation that stores its keys using **Weak References** (`java.lang.ref.WeakReference`).

```
Standard HashMap:
Map Entry ────[Strong Reference]────> Key Object (Cannot be GC'd while Map exists)

WeakHashMap:
Map Entry ────[Weak Reference]──────> Key Object (GC reclaims when no strong references remain)
```

#### Key Characteristics
- **Key Reference Model:** Keys are encapsulated in `WeakReference` instances. When an object is no longer referenced by active strong references in the application, the JVM Garbage Collector automatically reclaims the key memory.
- **ReferenceQueue & Stale Entry Expunging:** When the GC reclaims a weak key, it enqueues the `WeakReference` in an internal `ReferenceQueue`. When subsequent operations (`get()`, `put()`, `size()`) are executed on `WeakHashMap`, the map polls this queue and silently deletes the stale entries and their associated values.
- **Thread Safety:** `WeakHashMap` is not synchronized. Wrap with `Collections.synchronizedMap()` for multi-threaded environments.
- **Primary Use Case:** Memory-sensitive caches (e.g., caching metadata or expensive computations tied to short-lived domain entities), preventing long-running services from accumulating memory leaks and throwing `OutOfMemoryError`.

---

## 6. Performance Utilities & Iteration Mechanics

### Native Memory Copying with System.arraycopy()
`System.arraycopy()` is a core utility method for bulk array manipulation:
```java
public static native void arraycopy(Object src, int srcPos, Object dest, int destPos, int length);
```

#### Why Preferred Over Manual Bytecode Iteration
1. **Hardware-Level Native Execution:** Marked with the `native` keyword, `System.arraycopy()` is implemented in C/C++ via JNI. It bypasses JVM bytecode interpretation and executes bulk memory block transfers (analogous to `memmove` in C) directly at the CPU hardware level.
2. **Elimination of Loop Overhead:** A manual Java `for` loop executing over 1,000,000 elements requires the JVM to perform 1,000,000 individual array bounds checks and element assignments, generating massive CPU overhead.
3. **Safe Handling of Overlapping Memory Regions:** When shifting elements within the same array (e.g., right-shifting for array insertion), source and destination memory regions overlap. A naive manual forward loop overwrites source elements before they are read, corrupting data. `System.arraycopy()` detects overlap and copies memory in reverse or buffers safely.

---

### Fail-Fast vs Fail-Safe (Weakly Consistent) Iterators

#### 1. Fail-Fast Iterators (Standard Collections)
- **Concept:** Operates directly on the live memory structure of the underlying collection (`ArrayList`, `HashSet`, `HashMap`). If the collection is structurally modified (elements added or removed) during iteration by any thread, it immediately fails.
- **The Mechanism (`modCount`):** Collections track structural modifications using an internal integer `modCount`. When an iterator is initialized, it records `expectedModCount = modCount`. On each `iterator.next()` call, it asserts `modCount == expectedModCount`. Any discrepancy throws `ConcurrentModificationException`.
- **Safe Element Removal:** Calling the collection's `list.remove()` causes `modCount` mismatch and crashes. Safe removal requires using `iterator.remove()`, which updates both `modCount` and `expectedModCount` synchronously.

```java
List<String> cities = new ArrayList<>(Arrays.asList("New York", "London", "Tokyo"));

for (String city : cities) {
    if (city.equals("London")) {
        cities.add("Paris"); // Throws ConcurrentModificationException on next iteration!
    }
}
```

#### 2. Fail-Safe / Weakly Consistent Iterators (Concurrent Collections)
- **Nomenclature:** The official Java specification terms these **Weakly Consistent** iterators rather than "fail-safe".
- **Concept:** Iterates over a snapshot or concurrent view of the collection (`CopyOnWriteArrayList`, `ConcurrentHashMap`). Modifications made to the collection during iteration do not throw exceptions.
- **The Mechanism:** Operates on an immutable snapshot or traverses volatile bucket pointers without relying on `modCount`. Modifications update the primary collection while leaving active iterator traversal unaffected.
- **The Trade-off:** Sacrifices real-time consistency for non-blocking execution; newly added elements may not be observed during the current traversal.

```java
List<String> cities = new CopyOnWriteArrayList<>(Arrays.asList("New York", "London", "Tokyo"));

for (String city : cities) {
    if (city.equals("London")) {
        cities.add("Paris"); // Safe: Adds to underlying array without crashing iterator
    }
}
```
