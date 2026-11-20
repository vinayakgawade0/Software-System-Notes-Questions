# Module 11: Concurrency, JVM, GC

## 1. Advanced Concurrency Foundations & Hazards

### Concurrency Fundamentals & Architectural Impact
Concurrency represents the capacity of an application to execute multiple distinct computational tasks in overlapping timeframes. In Java, concurrency is achieved by dispatching multiple threads that share process heap memory while maintaining independent execution stacks.

#### The Three Core Motivations for Concurrency
1. **Hardware Utilization:** Modern enterprise servers feature 8 to 64+ physical CPU cores. A single-threaded application is bound to a single core, wasting up to 95% of available hardware compute power. Concurrency distributes execution across all hardware cores.
2. **Server Throughput (Web Tier):** Application servers (e.g., Spring Boot) handle thousands of simultaneous client requests. Concurrent dispatching processes these requests concurrently; serial execution would force downstream clients into unacceptable queue delays.
3. **Application Responsiveness:** Delegating long-running computations, file downloads, or network calls to background threads prevents UI or primary request pipelines from freezing.

---

### Race Conditions, Deadlocks & Thread Safety Strategies

#### 1. Race Conditions (Data Corruption)
A race condition occurs when concurrent threads execute non-atomic read-modify-write operations on shared mutable state, with the final state determined by OS thread scheduling order.
- **The Classic Increment Flaw:** The expression `count++` consists of three distinct bytecode steps:
  1. Read `count` from memory into CPU register.
  2. Increment register value by 1.
  3. Write register value back to memory.
  If Thread A and Thread B read `count = 5` simultaneously, both calculate `6` and write back `6`. Two increments occurred, but the counter advanced by only one.
- **The Resolution:** Enforce atomicity using atomic hardware instructions (`AtomicInteger` via CAS), explicit locks (`ReentrantLock`), or `synchronized` blocks.

#### 2. Deadlocks (Circular Wait Freeze)
A deadlock is an unrecoverable state where two or more threads are permanently blocked, each holding a lock that another thread requires to proceed.
- **Circular Wait Condition:** Thread 1 acquires Lock A and requests Lock B; Thread 2 acquires Lock B and requests Lock A. Neither thread can proceed.
- **Prevention Strategies:**
  - **Strict Lock Ordering:** Mandate that all threads acquire locks in a globally defined, identical sequence (always acquire Lock A before Lock B).
  - **Timed Lock Acquisition:** Utilize `ReentrantLock.tryLock(timeout, unit)` to retreat and release held resources if a secondary lock cannot be acquired promptly.

#### 3. Strategies for Thread Safety
A component is **thread-safe** if it functions correctly when accessed concurrently by multiple threads without requiring caller synchronization.
- **Statelessness:** Eliminate mutable instance fields (e.g., standard Spring `@Service` beans). Local variables inside methods reside on isolated thread stacks.
- **Immutability:** Unmodifiable objects (Java `record`, `String`, classes with `private final` fields) require zero synchronization on read paths.
- **Thread-Confinement:** Isolate mutable state to individual threads using `ThreadLocal<T>`.

#### 4. Synchronization Mechanics & Trade-offs
`synchronized` enforces mutual exclusion using the object's intrinsic monitor. Entering claims ownership; exiting releases the monitor and flushes CPU caches to main RAM.
- **The Architectural Cost:** Intrinsic synchronization is a coarse-grained, pessimistic lock that forces threads into a single-file queue. High-throughput systems prefer lock-free primitives (`AtomicInteger`, `ConcurrentHashMap`) over heavy synchronized blocks.

---

### Livelock vs Starvation (Active Loops vs Unfair Denial)

| Concurrency Hazard | Operational State | Underlying Failure Mechanism | Remediation Strategy |
| :--- | :--- | :--- | :--- |
| **Starvation** | `BLOCKED` or `WAITING` | A thread is perpetually denied CPU time or lock acquisition because greedy or higher-priority threads continually bypass it in unfair queues. | Enforce strict FIFO fairness policies using `new ReentrantLock(true)`. |
| **Livelock** | `RUNNABLE` (Active CPU) | Threads actively execute instructions and respond to each other's state changes, but do so in an infinite loop that yields zero forward progress (the "hallway dance"). | Introduce randomized backoff delays (`Thread.sleep(random)`) before retrying contended resource acquisition. |

---

### Thread-per-Request vs Event-Driven & Virtual Threads Scalability

#### 1. Thread-per-Request Bottleneck (Spring MVC / Tomcat)
- Standard servlet containers assign one physical OS thread per HTTP request.
- **The Blocking I/O Problem:** If a thread queries a database or downstream microservice, it blocks and enters a dormant wait state. It does zero CPU work, but its 1MB stack remains committed in memory.
- **The C10k Wall:** 2,000 idle waiting threads consume ~2GB of RAM. Once thread limits are reached, the server rejects connections (`Connection Refused`) even if CPU utilization is under 10%.

#### 2. Event-Driven Non-Blocking Architecture (Spring WebFlux / Netty)
- Replaces thousands of blocking threads with an **Event Loop** executing over a small thread pool (typically matching CPU core counts).
- **Non-Blocking I/O (epoll):** Threads dispatch network requests asynchronously, register callbacks, and immediately process subsequent requests. When I/O data arrives, the OS notifies the event loop to complete the response.
- **Scalability:** Easily handles 100,000+ concurrent connections on minimal RAM, bounded purely by CPU processing capacity.
- **Trade-off:** Complex reactive syntax and difficult debugging (stack traces lose request continuity).

#### 3. Architectural Decision Guidelines
- **Thread-per-Request (Spring MVC):** Ideal for standard transactional CRUD systems with CPU-bound processing where concurrent connection volume remains bounded.
- **Event-Driven (Spring WebFlux):** Essential for high-concurrency API gateways, streaming pipelines, and chat applications.
- **Virtual Threads (Java 21):** Combines the simple, imperative coding model of Thread-per-Request with the massive connection density of event-driven architectures.

---

### Categories of Concurrency-Safe Data Structures
Modern Java architectures avoid concurrency issues using five distinct design strategies:

1. **Immutable Structures:** Mathematically eliminate race conditions. State is frozen post-construction; mutations return new instances (`java.lang.String`, `java.time.Instant`, `List.of()`).
2. **Thread-Confined Structures:** Allocate private, independent instances per thread, avoiding data sharing entirely (`ThreadLocal<T>`).
3. **Lock-Free / Non-Blocking Structures:** Eliminate OS thread parking using hardware-level Compare-And-Swap (CAS) spin loops (`AtomicInteger`, `ConcurrentLinkedQueue`).
4. **Partitioned / Striped Structures:** Divide datasets into isolated segments with independent locks, enabling concurrent multi-thread writes across distinct buckets (`ConcurrentHashMap`).
5. **Copy-On-Write Structures:** Treat internal arrays as immutable, duplicating the underlying structure on modification while providing 100% lock-free snapshot reads (`CopyOnWriteArrayList`).

---

### OS Synchronization Primitives Mapping to Java API
Java's high-level concurrency utilities map directly onto underlying operating system and hardware kernel primitives:

#### 1. Mutex Locks $\rightarrow$ `synchronized` / `ReentrantLock`
- **OS Kernel Level:** Operating systems implement Mutexes using CPU atomic instructions (e.g., `cmpxchg`). When Thread B requests a locked mutex, the OS executes a context switch, deschedules Thread B from the CPU, and places it into a suspended Wait Queue.
- **Java Abstraction:** The JVM abstracts this mechanism via `synchronized` monitor bytecode (`monitorenter`) and explicit `ReentrantLock.lock()` implementations.

#### 2. Condition Variables / Signals $\rightarrow$ `wait()`/`notify()` & `Condition.await()`/`signal()`
- **OS Kernel Level:** When a thread holding a mutex realizes a business precondition is unmet (e.g., an input buffer is empty), holding the mutex would deadlock the system, while spinning would overheat the CPU. The thread issues a Wait command: the OS atomically yields the mutex and transitions the thread to a Condition Queue. A subsequent thread signals the condition, prompting the OS to transfer the waiting thread back to the Ready Queue.
- **Java Abstraction:** Java exposes this pattern through `Object.wait()` / `notifyAll()` and `Condition.await()` / `signal()`. Modern enterprise design encapsulates these primitives within `java.util.concurrent.BlockingQueue`.

---

### Virtual Threads (Project Loom / Java 21)
Introduced in Java 21 (JEP 444), **Virtual Threads** are lightweight user-mode threads managed directly by the JVM runtime rather than the underlying operating system kernel.

```java
import java.util.concurrent.Executors;

public class VirtualThreadDemo {
    public static void main(String[] args) {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 1000; i++) {
                int taskId = i;
                executor.submit(() -> {
                    System.out.println("Task " + taskId + " running on " + Thread.currentThread());
                    try {
                        Thread.sleep(100); // Non-blocking: unmounts from carrier thread!
                    } catch (InterruptedException e) {}
                });
            }
        }
    }
}
```

#### Platform Threads vs Virtual Threads
- **Platform Thread:** Maps 1:1 to an operating system kernel thread. Heavyweight stack footprint (~1MB), expensive context switching, bounded to thousands.
- **Virtual Thread:** Multiplexes thousands or millions of virtual threads onto a tiny pool of OS "carrier threads" (managed by an internal `ForkJoinPool`).
- **Unmounting on Blocking I/O:** When a virtual thread executes a blocking operation (e.g., `Thread.sleep()`, JDBC query, HTTP socket read), the JVM unmounts the virtual thread, saves its continuation stack in the heap, and reassigns the underlying carrier OS thread to execute another task. Once I/O completes, the virtual thread is mounted back onto an available carrier thread.
- **Enterprise Use Case:** Highly scalable, I/O-intensive backend microservices. Allows writing clean, imperative blocking code (JDBC, RestTemplate) while scaling to millions of concurrent requests.
- **Pinning Warning:** Avoid long-held synchronized blocks and foreign native JNI calls inside virtual threads, as these can **pin** the virtual thread to its carrier, preventing unmounting.

---

## 2. JVM Architecture & Memory Subsystems

### Java Memory Model (JMM) Hardware Interaction
The Java Memory Model (JMM) defines the formal specification governing how threads interact through shared memory, reconciling the gap between high-level Java code and physical multicore hardware architectures.

#### The Physical Hardware Reality
- **Main Memory (RAM):** The central shared repository where object instances reside. Memory access across the system bus incurs high latency.
- **Working Memory (L1/L2/L3 Caches):** Every physical CPU core maintains ultra-fast local cache storage. When a thread accesses a variable, the hardware copies it into its local CPU cache line.

#### The Memory Visibility Problem
Because cores compute against local caches, modifications made by Thread A on Core 1 are not immediately flushed to Main RAM. Thread B running on Core 2 continues reading stale data from its local cache.

#### The Happens-Before Relationship
The JMM defines strict **Happens-Before** rules to guarantee that memory writes by one action are visible to subsequent reads:
1. **Monitor Lock Rule:** An unlock operation on a monitor strictly happens-before every subsequent lock acquisition on that same monitor.
2. **Volatile Variable Rule:** A write to a `volatile` field strictly happens-before every subsequent read of that same field.
3. **Thread Start Rule:** A call to `thread.start()` strictly happens-before any action in the started thread.
4. **Thread Join Rule:** All actions in a thread strictly happen-before any other thread successfully returns from `thread.join()` on it.

---

### Overall JVM Architecture & Primary Subsystems
The Java Virtual Machine consists of three primary, interconnected execution subsystems:

```
┌─────────────────────────────────────────────────────────────┐
│                 1. ClassLoader Subsystem                    │
│          Loading  ───>  Linking  ───>  Initialization       │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                 2. Runtime Data Areas                       │
│ ┌─────────────────────────┐     ┌─────────────────────────┐ │
│ │  Method Area/Metaspace  │     │       Heap Space        │ │
│ ├─────────────────────────┤     ├─────────────────────────┤ │
│ │   JVM Thread Stacks     │     │   PC Registers / Native │ │
│ └─────────────────────────┘     └─────────────────────────┘ │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                 3. Execution Engine                         │
│    Interpreter  ───>  JIT Compiler  ───> Garbage Collector  │
└─────────────────────────────────────────────────────────────┘
```

1. **ClassLoader Subsystem:** Discovers and ingests binary `.class` files, validates bytecode semantics, and allocates runtime metadata.
2. **Runtime Data Areas (JVM Memory):** Organizes application storage across Heap, Metaspace, Stacks, Program Counter (PC) Registers, and Native Method Stacks.
3. **Execution Engine:** Processes bytecode instructions via interpretation, compiles high-frequency "hotspots" to native machine code via the JIT Compiler, and reclaims heap memory via the Garbage Collector.

---

### ClassLoader Hierarchy & Delegation Mechanics

#### The Three-Tier Delegation Hierarchy
Java loads classes dynamically on demand across a parent-child hierarchy:
1. **Bootstrap ClassLoader:** Implemented in native C/C++. Loads foundational runtime classes (`java.base`, `java.lang.String`, `java.util.*`) from runtime modules.
2. **Platform / Extension ClassLoader:** Loads platform extensions and auxiliary JDK modules.
3. **Application / System ClassLoader:** Loads application-specific classes located on the application classpath (`.jar` files and build dependencies).

#### The Three Architectural Principles
- **Delegation Principle (Parent-First):** When a ClassLoader receives a loading request, it delegates upward to its parent before searching locally. Only if the parent hierarchy fails to find the class does the child attempt to load it.
- **Visibility Principle:** Child ClassLoaders can see classes loaded by parent ClassLoaders, but parent ClassLoaders cannot access classes loaded by their children.
- **Uniqueness Principle:** Guarantees that classes are loaded exactly once. Parent-first delegation prevents untrusted application code from overriding core classes like `java.lang.Object`.

#### The Three Phases of Class Loading
1. **Loading:** Reads binary byte data from disk/network and constructs the `java.lang.Class` blueprint in Metaspace.
2. **Linking:**
   - *Verification:* Ensures bytecode adheres to JVM specifications and contains no security/stack exploits.
   - *Preparation:* Allocates memory for static variables and initializes them to default zero-values (`null`, `0`, `false`).
   - *Resolution:* Resolves symbolic references in the constant pool into direct memory pointers.
3. **Initialization:** Executes `static { }` initialization blocks and assigns defined values to static variables.

---

### Stack Memory vs Heap Space Interaction

| Feature | Stack Memory | Heap Space |
| :--- | :--- | :--- |
| **Scope & Ownership** | **Thread-Isolated:** Every thread owns a private stack; invisible to other threads | **Global Shared:** A single common heap instance shared across all threads |
| **Stored Contents** | Local primitive variables, method execution frames, object reference pointers | Concrete object instances and their non-static instance fields |
| **Management Model** | Strictly LIFO (Last-In-First-Out) governed automatically by method entry/return | Managed dynamically by the Garbage Collector |
| **Failure Exception** | `java.lang.StackOverflowError` (runaway infinite recursion) | `java.lang.OutOfMemoryError: Java heap space` (memory leaks) |

#### Memory Interaction & Object Orphaning
When a method declares `User user = new User("Alice");`:
- The concrete `User` object (containing `"Alice"`) is allocated in **Heap Space**.
- The variable `user` is allocated in the current **Stack Frame**, holding the memory address pointer to the heap object.
- When the method finishes execution, its stack frame is popped. The `user` reference pointer is destroyed. Because the heap object no longer has active references tracing back to GC Roots, it becomes an **orphan** eligible for Garbage Collection.

---

## 3. Garbage Collection & Memory Optimization

### Generational Garbage Collection & Object Lifecycle
Java memory management relies on the **Weak Generational Hypothesis**: the empirical observation that the vast majority of allocated objects die very shortly after creation, while objects surviving initial cycles tend to persist long-term.

#### Segmented Heap Architecture
- **Young Generation:**
  - **Eden Space:** Initial allocation zone for all newly created objects.
  - **Survivor Spaces (S0 and S1):** Buffer zones for objects surviving Minor GC cycles.
- **Old (Tenured) Generation:** Houses long-lived objects (e.g., Spring singletons, connection pools, cache entries).

#### The Three GC Execution Phases
1. **Marking:** Traverses the object graph starting from **GC Roots** (active thread stacks, static variables, JNI references). Objects reachable from GC roots are marked live; unreachable objects are flagged as garbage.
2. **Sweeping:** Deallocates heap memory occupied by unmarked (dead) objects.
3. **Compacting:** Relocates surviving live objects contiguously in memory to eliminate free-space fragmentation.

#### Object Promotion Lifecycle
1. New objects are allocated in Eden.
2. When Eden fills, a **Minor GC** triggers. Surviving objects are evacuated to Survivor Space S0 with tenure age incremented to 1.
3. On subsequent cycles, surviving objects toggle between S0 and S1 (one survivor space is always empty).
4. Once an object's age surpasses the **Tenuring Threshold** (typically 15 cycles, configured via `-XX:MaxTenuringThreshold`), it is promoted to the **Old Generation**.
5. When the Old Generation reaches capacity, a **Major (Full) GC** triggers, scanning the entire heap and causing significant **Stop-The-World (STW)** application pauses.

---

### PermGen vs Metaspace Evolution
In Java 8, **Metaspace** completely replaced the legacy **Permanent Generation (PermGen)**.

| Architectural Feature | PermGen (Java 7 and earlier) | Metaspace (Java 8+) |
| :--- | :--- | :--- |
| **Memory Location** | Contiguous segment inside the JVM Heap | Native OS virtual memory (Off-Heap) |
| **Sizing Model** | Fixed limit defined at startup (`-XX:MaxPermSize`) | Dynamically auto-expanding bounded only by physical OS memory |
| **Out-Of-Memory Risk** | High: frequent `java.lang.OutOfMemoryError: PermGen space` | Rare: throws OOM only if host physical RAM is exhausted |
| **Garbage Collection** | Inefficient class metadata unloading | Optimized concurrent class unloading |

#### The Metaspace Native Memory Leak Trap
Modern frameworks (Spring, Hibernate, ByteBuddy) generate classes dynamically at runtime. While Metaspace prevents premature heap crashes, an unconstrained classloader leak will consume all host RAM, triggering the Linux **OOM Killer** to terminate the JVM abruptly.
- **Enterprise Best Practice:** Always cap Metaspace in production using `-XX:MaxMetaspaceSize=512m` to fail fast and safeguard host operating systems.
- *Note:* In Java 7, interned Strings and static variables were relocated to the regular Heap; Metaspace strictly stores class definitions and metadata.

---

### G1GC vs CMS Collector Mechanics

#### 1. Concurrent Mark Sweep (CMS) — The Deprecated Legacy
- Relied on a rigid, monolithic heap layout (fixed Young and Old generations).
- Executed marking and sweeping concurrently with application threads, but **did not compact memory**.
- **The Swiss Cheese Flaw:** Over time, Old Gen memory fragmented severely. When large objects could not find contiguous space, CMS collapsed into a catastrophic, single-threaded Stop-The-World Full GC.
- Officially removed in Java 14.

#### 2. Garbage-First Collector (G1GC) — The Modern Standard
- Default garbage collector since Java 9.
- **Region-Based Partitioning:** Divides the heap into thousands of equal-sized independent **Regions** (1MB to 32MB), dynamically designated as Eden, Survivor, or Old.
- **Incremental Compaction (Evacuation):** Reclaims memory by copying live objects out of fragmented regions into fresh empty regions, inherently compacting memory without dedicated full-heap compaction pauses.
- **Garbage-First Heuristic:** Tracks dead-object density across all regions and prioritizes collecting regions containing the highest concentration of garbage first.
- **Pause-Time Driven:** Tunable via `-XX:MaxGCPauseMillis` (e.g., target 200ms pauses).

#### Ultra-Low Latency Evolution: ZGC & Shenandoah
For massive heaps (gigabytes to terabytes) requiring sub-millisecond pauses, modern JVMs utilize **ZGC** or **Shenandoah**. These collectors execute all marking, evacuation, and compaction phases concurrently with application threads, guaranteeing $< 1\text{ms}$ pause times regardless of heap size.

---

### Memory Leak Detection & Remediation Strategies
A memory leak in Java occurs when unused objects remain transitively reachable from active GC Roots, preventing the Garbage Collector from reclaiming their memory and eventually triggering `OutOfMemoryError: Java heap space`.

#### Detection & Diagnostic Workflow
1. **Telemetry Observation (The Sawtooth Graph):** In monitoring tools (Datadog, Prometheus), a healthy heap exhibits a sawtooth pattern that resets to a stable baseline after GC cycles. A memory leak causes the post-GC baseline to rise steadily over time.
2. **Heap Dump Acquisition:** Capture a heap snapshot using `-XX:+HeapDumpOnOutOfMemoryError` or runtime CLI utilities (`jmap -dump:live,format=b,file=heap.hprof <pid>`).
3. **Memory Analysis Tools (MAT / VisualVM):** Inspect the dump using Eclipse Memory Analyzer Tool (MAT). Generate a **Dominator Tree** to identify top memory consumers and trace their **Shortest Path to GC Roots** to pinpoint the class holding references hostage.

#### Common Architectural Causes & Fixes
- **Homemade Unbounded Caches:** Storing entries in static `HashMap` structures without eviction. Fix: Migrate to bounded caching libraries (Caffeine, Guava) with strict size limits and Time-To-Live (TTL) expiration.
- **Bad Map Keys:** Custom keys with incorrect `equals()` and `hashCode()` implementations or keys mutated post-insertion become permanently unretrievable. Fix: Enforce immutable keys (Java records).
- **Unclosed System Resources:** Open database connections or stream buffers. Fix: Mandate `try-with-resources`.
- **Kubernetes OOMKills:** Linux kernel terminates containers when JVM exceeds container memory limits. Fix: Use `-XX:MaxRAMPercentage=75.0` to dynamically align JVM heap sizing with container boundaries.
