# Questions: Collections Framework

### What are the main Java Collections Framework interfaces, their common implementation classes, and when should each be used?

> **Direct Answer:**
> - **List (`ArrayList`, `LinkedList`, `CopyOnWriteArrayList`):** Ordered collections allowing duplicates; use `ArrayList` by default for fast $O(1)$ random access, `LinkedList` only for head/tail deque operations, and `CopyOnWriteArrayList` for multi-threaded read-heavy workloads.
> - **Set (`HashSet`, `LinkedHashSet`, `TreeSet`):** Unique collections; use `HashSet` for $O(1)$ lookup, `LinkedHashSet` to preserve insertion sequence, and `TreeSet` for sorted/navigable data ($O(\log n)$).
> - **Queue & Deque (`ArrayDeque`, `PriorityQueue`):** Use `ArrayDeque` as the modern standard for LIFO stacks and FIFO queues; use `PriorityQueue` for priority heap ordering.
> - **Map (`HashMap`, `LinkedHashMap`, `TreeMap`, `ConcurrentHashMap`):** Key-value associations; use `HashMap` for general lookups, `LinkedHashMap` for LRU caches, `TreeMap` for sorted keys, and `ConcurrentHashMap` for high-throughput concurrency.
>
> **Key Nuance / Trade-off:**
> - `Map` does not inherit from `Collection`, but belongs to the framework; modern CPU cache line prefetching makes `ArrayList` and `ArrayDeque` consistently outperform linked-node alternatives across practical workloads.
>
> **Reference Concept:** `Core Framework Hierarchy & Implementations`

---

### What’s the difference between ArrayList and LinkedList?

> **Direct Answer:**
> - `ArrayList` is backed by a contiguous, resizable array delivering $O(1)$ random access; `LinkedList` is backed by a doubly-linked list where random access is $O(n)$ due to node traversal.
> - Inserting or deleting in the middle is $O(n)$ in `ArrayList` (requires element shifting), but updating pointers in `LinkedList` is $O(1)$ once the node is reached (finding it takes $O(n)$).
>
> **Key Nuance / Trade-off:**
> - Mechanical Sympathy: `ArrayList` leverages CPU L1/L2 cache line prefetching due to contiguous memory, whereas `LinkedList` incurs constant CPU cache misses and heap fragmentation from allocating independent `Node` objects for every element.
>
> **Reference Concept:** `ArrayList vs LinkedList Architecture & CPU Cache Locality`

---

### How do you convert a List to a Set in Java while handling duplicates?

> **Direct Answer:**
> - Pass the list to a `Set` constructor: `new HashSet<>(list)` removes duplicates with no ordering guarantees.
> - Use `new LinkedHashSet<>(list)` to remove duplicates while strictly preserving the list's original insertion order.
> - Use `new TreeSet<>(list)` to remove duplicates and sort elements in natural or comparator-defined order.
> - In Java 8+, use Streams: `list.stream().collect(Collectors.toSet())` or `list.stream().collect(Collectors.toCollection(LinkedHashSet::new))`.
>
> **Key Nuance / Trade-off:**
> - `Collectors.toSet()` defaults to a `HashSet` with no order guarantees; custom collection collectors must be passed explicitly when ordering is required.
>
> **Reference Concept:** `List to Set Conversion & Deduplication`

---

### What are real usage of Doubly Linked List (DLL) and Circular Linked List (CLL)

> **Direct Answer:**
> - **Doubly Linked List (DLL):** Used in Browser History navigation (Back/Forward), Undo/Redo buffers in text editors, and LRU Caches (paired with `HashMap` for $O(1)$ node eviction and reordering).
> - **Circular Linked List (CLL):** Used for Ring Buffers and Circular Queues, such as OS CPU Round-Robin process scheduling, turn-based game turns, and fixed-size streaming media buffers.
>
> **Key Nuance / Trade-off:**
> - In a CLL, `tail.next` points to `head`, eliminating `null` edge cases and bounds resets during continuous cyclical operations.
>
> **Reference Concept:** `Real-World Applications of Doubly & Circular Linked Lists`

---

### Difference between HashMap, Hashtable and ConcurrentHashMap.

> **Direct Answer:**
> - `HashMap` is unsynchronized, fast in single-threaded environments, and allows one `null` key and multiple `null` values with fail-fast iteration.
> - `Hashtable` is an obsolete Java 1.0 legacy class that synchronizes every method with coarse-grained locks, blocking all concurrent access, and bans `null` keys/values.
> - `ConcurrentHashMap` provides enterprise thread safety without global locks, using lock-free reads, CAS on empty buckets, and node-level locking on collisions with weakly consistent iterators.
>
> **Key Nuance / Trade-off:**
> - `ConcurrentHashMap` strictly bans `null` keys and values to prevent race-condition ambiguities between key absence and explicit `null` mapping.
>
> **Reference Concept:** `Map Implementations Comparison (HashMap vs Hashtable vs ConcurrentHashMap)`

---

### Walk me through the internal working of a HashMap in Java 8+. How does it handle collisions, dynamic resizing, and what are the concurrency implications?

> **Direct Answer:**
> - **Internal Storage:** Stores `Node<K,V>` instances across an array of buckets, computing indices via `(n - 1) & ((h = key.hashCode()) ^ (h >>> 16))`.
> - **Collision Resolution:** Collisions chain nodes in a linked list; once a bucket reaches `TREEIFY_THRESHOLD = 8` entries and table capacity $\ge 64$, it transforms into a balanced Red-Black Tree, improving worst-case search from $O(n)$ to $O(\log n)$.
> - **Dynamic Resizing:** When entry count exceeds `capacity * 0.75` (default 12 for initial capacity 16), capacity doubles; Java 8 uses a high-bit check (`hash & oldCap`) to split chains without recalculating full hashes.
>
> **Key Nuance / Trade-off:**
> - `HashMap` is fundamentally thread-unsafe; concurrent resizing in Java 7 caused infinite circular loops (100% CPU lock), while Java 8 causes silent data corruption. Always pre-size maps for known volumes to eliminate expensive rehashes.
>
> **Reference Concept:** `HashMap Internal Working in Java 8+`

---

### What is the difference between LinkedHashMap and HashMap?

> **Direct Answer:**
> - `HashMap` provides no iteration order guarantees, and its order can shift unpredictably following a table resize.
> - `LinkedHashMap` maintains a doubly-linked list running through all its entries, guaranteeing predictable iteration matching either **Insertion Order** or **Access Order**.
>
> **Key Nuance / Trade-off:**
> - `LinkedHashMap` incurs a higher memory footprint due to maintaining `before` and `after` pointers per entry, but enables straightforward implementation of LRU caches via `removeEldestEntry()`.
>
> **Reference Concept:** `HashMap vs LinkedHashMap`

---

### Can you use a custom object as a key in a Java Map, and what strict contracts must you follow to prevent data loss and memory leaks?

> **Direct Answer:**
> - Yes, provided `equals()` and `hashCode()` are properly overridden so that logically equal objects yield identical hash codes and match on field equality.
> - Relying on default `Object` methods compares memory addresses; lookups with distinct equivalent instances will fail and return `null`.
> - All fields participating in hash code and equality evaluation must be strictly **immutable (`final`)**.
>
> **Key Nuance / Trade-off:**
> - Mutating a key's fields post-insertion causes its hash code to shift: subsequent lookups search a new bucket and return `null`, leaving the original entry permanently stranded in the old bucket and creating a silent memory leak.
>
> **Reference Concept:** `Custom Objects as Map Keys & The Contract Rules`

---

### How does ConcurrentHashMap work under the hood in modern Java, and why is it architecturally superior to using a Collections.synchronizedMap() or standard synchronized blocks?

> **Direct Answer:**
> - `ConcurrentHashMap` replaces coarse-grained global locks with fine-grained concurrency: reads are 100% lock-free via `volatile` field pointers.
> - Inserting into empty buckets uses CPU hardware-level Compare-And-Swap (CAS), requiring zero lock acquisition.
> - Inserting into populated buckets acquires a lock exclusively on the head node of that specific bucket (`synchronized (head)`), allowing concurrent threads to write to other buckets simultaneously.
>
> **Key Nuance / Trade-off:**
> - Java 8 abandoned Java 7's 16-segment striping architecture in favor of individual bucket node synchronization, scaling concurrency with table capacity.
>
> **Reference Concept:** `ConcurrentHashMap Internals (Java 8+ CAS & Node-Level Locking)`

---

### What is the exact difference between ConcurrentHashMap and Collections.synchronizedMap(), and why is the latter considered obsolete?

> **Direct Answer:**
> - `Collections.synchronizedMap()` places a global `synchronized(this)` lock over every method, forcing readers to wait behind writers and readers to block other readers.
> - `ConcurrentHashMap` allows unlimited concurrent reads without blocking and isolates write locks to individual bucket head nodes.
> - Iterators in `synchronizedMap` are fail-fast and throw `ConcurrentModificationException` unless manually synchronized on the map; iterators in `ConcurrentHashMap` are weakly consistent and safe from concurrent crashes.
>
> **Key Nuance / Trade-off:**
> - `synchronizedMap()` allows `null` keys and values, whereas `ConcurrentHashMap` strictly prohibits them to eliminate concurrency ambiguity.
>
> **Reference Concept:** `ConcurrentHashMap vs Collections.synchronizedMap()`

---

### Why does ConcurrentHashMap strictly forbid null keys and null values, whereas a standard HashMap allows them?

> **Direct Answer:**
> - In a single-threaded `HashMap`, a `null` return from `get(key)` can be disambiguated between "key not present" and "key mapped to null" using `containsKey(key)`.
> - In a multi-threaded environment, the two-step verification (`get()` followed by `containsKey()`) is not atomic; another thread can modify or delete the entry between the two calls, corrupting logic.
> - Banning `null` keys and values eliminates the ambiguity race condition entirely: returning `null` unambiguously signifies key absence.
>
> **Key Nuance / Trade-off:**
> - Doug Lea considered permitting `null` in original collections a fundamental design flaw and systematically eliminated it across modern concurrent utilities.
>
> **Reference Concept:** `ConcurrentHashMap vs Collections.synchronizedMap()`

---

### How does ConcurrentHashMap guarantee thread safety when the underlying array gets full and needs to resize?

> **Direct Answer:**
> - It utilizes **Cooperative Resizing** (Concurrent Migration) rather than freezing the entire map under a single massive lock.
> - The resizing thread divides the table into logical chunks called **strides** (minimum 16 buckets) and migrates them to the new table.
> - Migrated buckets are marked with a `ForwardingNode` (`hash = -1`) pointing to the new table; incoming writer threads encountering a forwarding node join the migration swarm and help transfer remaining strides.
>
> **Key Nuance / Trade-off:**
> - Readers are completely non-blocking during resize: they read from the old table for unmigrated buckets or follow forwarding pointers to read from the new table.
>
> **Reference Concept:** `Cooperative Resizing in ConcurrentHashMap`

---

### How does CopyOnWriteArrayList work internally, and in what specific scenarios is it a terrible idea to use?

> **Direct Answer:**
> - On every write operation (`add()`, `set()`, `remove()`), the thread acquires a lock, allocates a brand-new array, copies all existing elements over, appends the mutation, and atomically swaps the internal `volatile` array reference.
> - Reads are 100% lock-free and iterate over immutable point-in-time snapshots without ever throwing `ConcurrentModificationException`.
> - It is disastrous in write-heavy workloads (constant array re-allocation causes CPU spikes and GC pause storms) and when strict real-time read consistency is required (reads are eventually consistent).
>
> **Key Nuance / Trade-off:**
> - Optimal strictly for small collections where reads vastly outnumber writes (e.g., event listener/observer registries with 99% reads and 1% writes).
>
> **Reference Concept:** `CopyOnWriteArrayList Mechanics & Trade-offs`

---

### If Thread A is writing to a map and Thread B is reading from the exact same element simultaneously, how do HashMap and ConcurrentHashMap handle it, and how are lock-free reads achieved?

> **Direct Answer:**
> - **HashMap:** Lacks memory visibility guarantees; Thread B can read stale data, observe partially updated objects, or crash with `ConcurrentModificationException` if iterating during a structural modification.
> - **ConcurrentHashMap:** Node values (`val`) and next pointers (`next`) are declared `volatile`, ensuring Thread B immediately observes Thread A's completed write across CPU caches.
> - Thread A acquires a lock strictly on that bucket's head node (or uses CAS), while Thread B reads without locking or blocking.
>
> **Key Nuance / Trade-off:**
> - Iterators in `ConcurrentHashMap` are weakly consistent, reflecting the table state when created and continuing safely without throwing exceptions if concurrent writes occur.
>
> **Reference Concept:** `Simultaneous Read-Write Scenarios (HashMap vs ConcurrentHashMap)`

---

### What is WeakHashMap and when would you use it?

> **Direct Answer:**
> - `WeakHashMap` is a hash-table implementation where keys are stored using **Weak References** (`WeakReference`).
> - When a key object is no longer referenced strongly elsewhere in the application, the JVM Garbage Collector automatically reclaims it.
> - It is primarily used for implementing memory-sensitive caches and metadata registries that automatically purge entries when objects are dereferenced.
>
> **Key Nuance / Trade-off:**
> - `WeakHashMap` is not thread-safe; wrap with `Collections.synchronizedMap()` for multi-threaded environments.
>
> **Reference Concept:** `WeakHashMap Architecture & GC Mechanics`

---

### How does WeakHashMap differ from a standard HashMap, and what is its primary architectural use case?

> **Direct Answer:**
> - Standard `HashMap` retains strong references to keys, permanently preventing garbage collection and causing memory leaks if entries are not manually deleted.
> - `WeakHashMap` enqueues reclaimed weak keys into a `ReferenceQueue`; subsequent operations (`get()`, `put()`) poll the queue and expunge stale entries and associated values.
> - Its primary architectural use case is caching transient metadata tied to lifecycle-bound objects, guaranteeing cache memory shrinks gracefully without causing an `OutOfMemoryError`.
>
> **Key Nuance / Trade-off:**
> - Only the **keys** are held weakly; values are held by strong references, so values must not contain strong references back to their own keys or GC reclamation will be blocked.
>
> **Reference Concept:** `WeakHashMap Architecture & GC Mechanics`

---

### If you need to shift elements to the right within an array, why is System.arraycopy() preferred over manual iteration?

> **Direct Answer:**
> - `System.arraycopy()` is a `native` C/C++ method executing via JNI, performing bulk block memory transfers (like `memmove`) directly at the CPU/hardware level.
> - Manual Java `for` loops force the JVM to execute bytecode loop iterations, performing bounds checking and single-element assignments that incur heavy CPU overhead.
> - It automatically handles overlapping source and destination memory regions, avoiding data corruption during in-place right-shifting.
>
> **Key Nuance / Trade-off:**
> - A naive left-to-right manual shift in overlapping regions overwrites elements before they can be read; `System.arraycopy()` buffers or copies backwards safely.
>
> **Reference Concept:** `Native Memory Copying with System.arraycopy()`

---

### Explain fail-safe and fail-fast iterators with examples

> **Direct Answer:**
> - **Fail-Fast Iterators (`ArrayList`, `HashMap`):** Traverse live collection structures directly; tracks modifications using `modCount` and immediately throws `ConcurrentModificationException` on `next()` if concurrent modifications occur.
> - **Fail-Safe / Weakly Consistent Iterators (`CopyOnWriteArrayList`, `ConcurrentHashMap`):** Traverse a point-in-time snapshot or concurrent view; modifications update the primary collection without interrupting or crashing the iterator.
> - Elements can be safely removed from fail-fast iterators exclusively via `iterator.remove()`, which synchronizes `modCount` and `expectedModCount`.
>
> **Key Nuance / Trade-off:**
> - The term "fail-safe" is industry terminology; the official Java specification designates these as "weakly consistent" because iterators trade immediate view consistency for lock-free non-crashing execution.
>
> **Reference Concept:** `Fail-Fast vs Fail-Safe (Weakly Consistent) Iterators`

---
