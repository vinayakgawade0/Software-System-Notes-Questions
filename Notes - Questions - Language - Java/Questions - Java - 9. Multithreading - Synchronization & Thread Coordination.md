# Questions: Multithreading - Synchronization & Thread Coordination

### How does the synchronized keyword work internally? Explain the Object Monitor and bytecode implementation.

> **Direct Answer:**
> - Every Java object contains an internal Monitor (intrinsic lock) whose ownership state is tracked in the Mark Word of the object header.
> - At bytecode compilation, the compiler strips the `synchronized` keyword and injects `monitorenter` at block entry and `monitorexit` at block exit (including an automatic `finally` block handler for exception safety).
> - Modern JVMs utilize Lock Escalation to optimize performance: Biased Locking (single thread, zero overhead) $\rightarrow$ Lightweight Locking (CAS spinning) $\rightarrow$ Heavyweight Locking (OS kernel mutex suspension).
>
> **Key Nuance / Trade-off:**
> - When a thread encounters contention on an escalated heavyweight lock, it is placed into the monitor's Entry Set in the `BLOCKED` state until the owner exits.
>
> **Reference Concept:** `The Object Monitor, Bytecode (monitorenter/monitorexit) & Lock Escalation`

---

### What are the different locking mechanisms in Java, and what are the critical architectural differences between the intrinsic synchronized keyword and the explicit java.util.concurrent.locks.Lock interface?

> **Direct Answer:**
> - Intrinsic locks (`synchronized`) are managed implicitly by the JVM bytecode and automatically release on exit or exception, but are rigid and unconditionally blocking.
> - Explicit locks (`Lock` / `ReentrantLock`) provide programmatic lock control and must be released manually inside a mandatory `finally` block.
> - `Lock` provides advanced thread-routing capabilities that `synchronized` lacks: non-blocking acquisition (`tryLock()`), timed acquisition (`tryLock(timeout)`), interruptibility (`lockInterruptibly()`), and configurable FIFO fairness.
>
> **Key Nuance / Trade-off:**
> - Forgetting to call `lock.unlock()` inside a `finally` block permanently leaks the lock, causing eventual application deadlocks.
>
> **Reference Concept:** `Intrinsic Locks vs java.util.concurrent.locks.Lock`

---

### What is reentrant locking, and why choose the explicit ReentrantLock over the standard synchronized keyword?

> **Direct Answer:**
> - Reentrant locking allows a thread to safely re-acquire a lock it already holds without blocking itself, incrementing an internal hold count and preventing self-deadlocks.
> - While both `synchronized` and `ReentrantLock` are reentrant, `ReentrantLock` is chosen when systems need non-blocking backoff (`tryLock`), interruptible lock waits, FIFO fairness policies, or multiple condition wait-queues.
>
> **Key Nuance / Trade-off:**
> - `synchronized` is simpler and eliminates leak risks, but `ReentrantLock` is essential for sophisticated concurrent architectures requiring timeout routing.
>
> **Reference Concept:** `Reentrant Locking Mechanics (ReentrantLock)`

---

### What are Condition objects in Java, and how do they work with ReentrantLock?

> **Direct Answer:**
> - Created via `lock.newCondition()`, a `Condition` object replaces the legacy `Object.wait()` and `notify()` methods by providing dedicated, targeted waiting queues tied to an explicit lock.
> - In a bounded buffer, separate conditions (`notFull` for producers, `notEmpty` for consumers) can be created from a single lock.
> - A producer signals `notEmpty.signal()` to wake strictly one consumer, eliminating the "Thundering Herd" performance issue where `notifyAll()` wakes all threads indiscriminately.
>
> **Key Nuance / Trade-off:**
> - The thread must hold the lock before calling `await()`; upon being signaled, the thread automatically re-acquires the lock before returning from `await()`.
>
> **Reference Concept:** `Condition Objects & Targeted Signaling (ReentrantLock.newCondition)`

---

### What is the exact difference between lock() and tryLock(), and what are the specific enterprise use cases for each?

> **Direct Answer:**
> - `lock()` blocks unconditionally and waits indefinitely until the lock is acquired, risking permanent thread freezing if the lock holder fails.
> - `tryLock()` evaluates lock availability conditionally, returning `true` immediately if acquired, or `false` instantly without waiting (or waiting up to an optional timeout).
> - Use `lock()` for mandatory transactions (e.g., account balance deductions); use `tryLock()` for deadlock avoidance backoff strategies, serving fallback cached data, or optional background jobs.
>
> **Key Nuance / Trade-off:**
> - Never place `tryLock()` directly above a `try-finally` block without checking its boolean return in an `if` condition; attempting to unlock an un-acquired lock throws `IllegalMonitorStateException`.
>
> **Reference Concept:** `lock() vs tryLock() Behavioral Contracts`

---

### What is a Read/Write Lock, how does it differ from a standard lock, and when should you use it?

> **Direct Answer:**
> - Standard locks enforce strict mutual exclusion for all operations; a `ReentrantReadWriteLock` splits locking into a shared Read Lock and an exclusive Write Lock.
> - Multiple threads can acquire the Read Lock simultaneously (Read + Read = Allowed); any thread acquiring the Write Lock blocks all readers and writers until completion.
> - It should be used exclusively in read-heavy workloads (e.g., 99% reads, 1% writes) with long-running operations.
>
> **Key Nuance / Trade-off:**
> - In write-heavy applications or very fast read operations, the internal CPU bookkeeping overhead of managing dual locks makes `ReadWriteLock` slower than a standard `ReentrantLock`; use `StampedLock` for optimistic lock-free reading.
>
> **Reference Concept:** `Read/Write Locks (ReentrantReadWriteLock) & StampedLock`

---

### What is the difference between coarse-grained and fine-grained locking, and what are the primary problems with each?

> **Direct Answer:**
> - **Coarse-Grained Locking:** Protects an entire system or multi-resource object with a single global lock; simple and deadlock-safe, but creates a single-threaded bottleneck destroying multicore concurrency.
> - **Fine-Grained Locking:** Uses multiple smaller locks protecting individual components (e.g., separate locks per account or per cache bucket); provides high throughput, but introduces severe deadlock risks if lock acquisition order is inconsistent.
> - Lock Striping (used in `ConcurrentHashMap`) is an optimal fine-grained design that isolates locks across independent bucket stripes.
>
> **Key Nuance / Trade-off:**
> - System designers must balance concurrency throughput against the architectural complexity and deadlock hazards of multi-lock ordering.
>
> **Reference Concept:** `Coarse-Grained vs Fine-Grained Locking & Lock Striping`

---

### Write a code snippet that may cause a deadlock. Thread synchronization and deadlock scenarios.

> **Direct Answer:**
> - A deadlock occurs when Thread 1 holds Resource A and waits for Resource B, while Thread 2 holds Resource B and waits for Resource A.
> - This cyclic lock dependency permanently freezes both threads:
>
> ```java
> Thread t1 = new Thread(() -> {
>     synchronized(r1) {
>         synchronized(r2) { System.out.println("T1 acquired both"); }
>     }
> });
> Thread t2 = new Thread(() -> {
>     synchronized(r2) {
>         synchronized(r1) { System.out.println("T2 acquired both"); }
>     }
> });
> ```
>
> **Key Nuance / Trade-off:**
> - Deadlocks can be prevented by enforcing a strict global lock acquisition hierarchy or using `tryLock()` with timeout backoff strategies.
>
> **Reference Concept:** `Deadlock Mechanisms & Code Scenarios`

---

### What guarantees does the volatile keyword provide, and what does it NOT guarantee?Can you explain the exact mechanism of the volatile keyword in Java, and what specific concurrency problem it solves?

> **Direct Answer:**
> - `volatile` guarantees **Visibility** and **Ordering**: it disables CPU L1/L2 caching, forcing all reads and writes to communicate directly with main RAM, and establishes happens-before memory fences to prevent instruction reordering.
> - It solves the visibility problem where one core's cached updates remain invisible to other cores.
> - It does **NOT** guarantee **Atomicity**: compound operations like `count++` (read-modify-write) are not atomic and require locks or CAS-based `AtomicInteger`.
>
> **Key Nuance / Trade-off:**
> - Marking a variable `volatile` is a lightweight memory barrier; it ensures updates are immediately observed across threads, but cannot prevent race conditions on compound state modifications.
>
> **Reference Concept:** `The volatile Keyword: Visibility vs Atomicity`

---

### What is the exact difference between a class-level lock and an object-level lock?

> **Direct Answer:**
> - An object-level lock (`synchronized(this)` or non-static synchronized method) locks the specific instance heap object; different instances have independent locks and execute concurrently.
> - A class-level lock (`synchronized(MyClass.class)` or static synchronized method) locks the single `Class` blueprint object in Metaspace, serializing access across all instances.
> - They operate on completely independent memory monitors and do not block each other.
>
> **Key Nuance / Trade-off:**
> - A thread executing a static synchronized method does not block another thread from concurrently running an instance synchronized method on an instance of that class.
>
> **Reference Concept:** `Class-Level Locks vs Object-Level Locks`

---

### How do you implement a thread-safe Singleton class, and what are the architectural drawbacks of this pattern?

> **Direct Answer:**
> - Implement using Double-Checked Locking with a `private static volatile` instance variable, a private constructor guarding against reflection, and a two-tier `getInstance()` check inside `synchronized(Class.class)`.
> - Alternatively, implement using an Enum Singleton for native reflection and serialization immunity.
> - Architectural drawbacks: Singletons hide global dependencies within code, make unit testing and mocking difficult, and introduce tight coupling.
>
> **Key Nuance / Trade-off:**
> - Enterprise applications replace raw singletons with Dependency Injection (Spring IoC), which manages bean lifecycles as singletons while keeping classes loosely coupled and testable.
>
> **Reference Concept:** `Double-Checked Locking Singleton & Memory Fence Resolution`

---

### Explain the Double-Checked Locking problem and how to fix it.

> **Direct Answer:**
> - Without `volatile`, the instruction sequence for `instance = new Singleton()` (allocate memory, run constructor, assign address) can be reordered by the CPU/compiler to allocate, assign address, and then run constructor ($1 \rightarrow 3 \rightarrow 2$).
> - A concurrent thread evaluating the first lock-free check observes `instance != null` before the constructor has completed, receiving a partially initialized object and throwing errors.
> - Fix: Declare `private static volatile Singleton instance;`—the `volatile` memory fence legally forbids the CPU from reordering memory assignment before constructor completion.
>
> **Key Nuance / Trade-off:**
> - `volatile` is strictly required for correctness; the double-check pattern without `volatile` is fundamentally broken under the Java Memory Model.
>
> **Reference Concept:** `Double-Checked Locking Singleton & Memory Fence Resolution`

---

### How can you ensure two threads run sequentially in Java?

> **Direct Answer:**
> - **`Thread.join()`:** Calling `t1.join()` halts the main thread until `t1` terminates, after which it starts `t2`.
> - **Single-Threaded Executor:** Submit tasks to `Executors.newSingleThreadExecutor()` to guarantee sequential FIFO execution.
> - **`CountDownLatch(1)`:** Thread 2 calls `latch.await()`, and Thread 1 executes `latch.countDown()` upon completion.
> - **`CompletableFuture`:** Chain async stages using `CompletableFuture.runAsync(t1).thenRun(t2)`.
> - **Shared Lock / State Flag:** Use `wait()` and `notify()` conditioned on an atomic sequence state.
>
> **Key Nuance / Trade-off:**
> - `join()` and `SingleThreadExecutor` are simplest, while `CompletableFuture` provides non-blocking composability for modern reactive pipelines.
>
> **Reference Concept:** `Sequential Thread Execution Strategies`

---

### What is a CountDownLatch in Java, how does it work, and what is a real-world use case for it?

> **Direct Answer:**
> - `CountDownLatch` is a synchronization gate initialized with an integer counter; calling `latch.await()` blocks the calling thread until worker threads decrement the count to 0 via `latch.countDown()`.
> - Once the count hits zero, the latch opens and waiting threads proceed.
> - Real-world use case: Service startup coordination, where a main server thread waits for concurrent background initializers (Database, Redis, Kafka) to complete before opening HTTP ports.
>
> **Key Nuance / Trade-off:**
> - `CountDownLatch` cannot be reused; once the counter reaches zero, the latch is permanently spent (use `CyclicBarrier` if reusability is needed).
>
> **Reference Concept:** `CountDownLatch vs CyclicBarrier`

---

### What is a CyclicBarrier in Java, how does it differ from a CountDownLatch, and what is a practical use case for it?

> **Direct Answer:**
> - `CyclicBarrier` is a synchronization primitive where a fixed number of participating worker threads all wait for each other at a common barrier point by calling `barrier.await()`.
> - When the final thread arrives, an optional barrier action `Runnable` executes, the barrier trips, all threads are released, and the barrier automatically resets for the next cycle.
> - Practical use case: Multi-phase parallel data aggregation (e.g., 4 threads processing chunks of a 100GB file per month, synchronizing at the barrier to merge monthly subtotals before starting the next month).
>
> **Key Nuance / Trade-off:**
> - In `CountDownLatch`, one master thread waits for $N$ background tasks; in `CyclicBarrier`, $N$ worker threads wait for each other repeatedly across multi-phase workflows.
>
> **Reference Concept:** `CountDownLatch vs CyclicBarrier`

---

### Explain the Producer-Consumer Problem and how to solve it in modern Java.

> **Direct Answer:**
> - The problem involves Producer threads writing to a fixed-size buffer and Consumer threads reading from it, where producers must wait when the buffer is full and consumers must wait when it is empty.
> - Legacy solutions used manual `synchronized`, `wait()`, and `notifyAll()` calls, which are prone to race conditions and deadlocks.
> - Modern enterprise solutions use `java.util.concurrent.BlockingQueue` (e.g., `ArrayBlockingQueue`), which encapsulates thread-safe waiting, buffer boundaries, and signaling automatically.
>
> **Key Nuance / Trade-off:**
> - `BlockingQueue` eliminates boilerplate condition loops and manual lock management, providing higher reliability and throughput.
>
> **Reference Concept:** `Producer-Consumer Problem: wait/notify vs BlockingQueue`

---

### How do you implement the producer-consumer problem? Explain both wait-notify and BlockingQueue approaches.

> **Direct Answer:**
> - **`wait()` / `notify()` Approach:** A shared queue object manages a data variable and an `available` flag; `produce()` waits while `available == true`, sets the value, and notifies; `consume()` waits while `available == false`, consumes the value, and notifies.
> - **`BlockingQueue` Approach:** Producers simply call `queue.put(item)` (which blocks automatically if full) and consumers call `queue.take()` (which blocks automatically if empty).
>
> **Key Nuance / Trade-off:**
> - The wait/notify approach requires manual `while` condition checks and synchronized blocks; `BlockingQueue` abstracts locking completely and optimizes performance using separate read/write locks.
>
> **Reference Concept:** `Producer-Consumer Problem: wait/notify vs BlockingQueue`

---

### How does BlockingQueue work internally, and what specific architectural problem does it solve?

> **Direct Answer:**
> - `BlockingQueue` provides thread-safe buffering with blocking `put()` (waits if full) and `take()` (waits if empty) operations, eliminating manual lock and condition management.
> - Provides a 4-method operational matrix: Throws Exception (`add`/`remove`), Special Value (`offer`/`poll`), Blocks (`put`/`take`), and Timeout (`offer`/`poll` with timeout).
> - `ArrayBlockingQueue` uses a single `ReentrantLock` for both insertions and removals; `LinkedBlockingQueue` uses two independent locks (`putLock` and `takeLock`), allowing concurrent simultaneous puts and takes.
>
> **Key Nuance / Trade-off:**
> - `LinkedBlockingQueue` provides higher concurrent throughput due to decoupled locks, but incurs GC allocation overhead for linked node wrappers compared to array storage.
>
> **Reference Concept:** `BlockingQueue Architecture & Implementation Mechanics`

---

### Q. What is the difference between `Future` and `CompletableFuture` in Java?
- `Future` only supports blocking retrieval (`get()`), lacks callback chaining, cannot combine multiple async tasks, and cannot be manually completed.
- `CompletableFuture` implements `CompletionStage`, enabling non-blocking pipelines (`thenApply`, `thenCompose`), parallel combination (`thenCombine`, `allOf`), declarative fallback recovery (`exceptionally`, `handle`), and timeout controls (`orTimeout`, `completeOnTimeout`).

---

### Q. What is the difference between `thenApply()` and `thenCompose()` in `CompletableFuture`?
- **`thenApply(fn)`** works like `Stream.map()`: it takes the resolved value `T` and applies a synchronous transformation returning `U`, yielding `CompletableFuture<U>`.
- **`thenCompose(fn)`** works like `Stream.flatMap()`: it takes `T` and invokes another asynchronous method returning `CompletableFuture<U>`, flattening the result into `CompletableFuture<U>` instead of `CompletableFuture<CompletableFuture<U>>`.

---

### Q. Why should you pass a custom `Executor` to `CompletableFuture.supplyAsync()` instead of relying on the default pool?
- Without an explicit `Executor`, `CompletableFuture` uses `ForkJoinPool.commonPool()`, which is shared across the entire JVM (including `parallelStream()`).
- Running blocking network or database I/O on `commonPool()` can exhaust its small thread count (`CPU cores - 1`) and stall unrelated tasks across the application. Passing a dedicated `ExecutorService` isolates I/O workloads.

---

### Q. How do `exceptionally()`, `handle()`, and `whenComplete()` differ?
- **`exceptionally(fn)`:** Runs **only on failure** and returns a fallback value to recover the pipeline.
- **`handle(biFn)`:** Runs **always** (receiving `(result, exception)`) and returns a transformed value or recovery result.
- **`whenComplete(biConsumer)`:** Runs **always** as a side-effect observer (e.g., logging/metrics) without modifying or swallowing the result or exception.

---