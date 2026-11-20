# Questions: Multithreading - Thread & Asynchronous

### What is multithreading in Java?

> **Direct Answer:**
> - Multithreading allows concurrent execution of multiple lightweight execution paths (threads) within the same process.
> - While a process has its own isolated memory space, threads within the same process share common heap memory, enabling fast communication and high CPU utilization.
> - It is implemented via the `Thread` class and `Runnable` or `Callable` interfaces.
>
> **Key Nuance / Trade-off:**
> - Shared heap memory requires strict synchronization or atomic wrappers to prevent race conditions and memory visibility bugs across concurrent threads.
>
> **Reference Concept:** `Multithreading Concepts & Process vs Thread`

---

### What is the exact difference between calling a thread's start() method and its run() method?

> **Direct Answer:**
> - Calling `start()` requests the operating system to allocate a new physical thread; the JVM then asynchronously invokes `run()` on that new thread.
> - Calling `run()` directly performs an ordinary, synchronous method call on the caller's thread without allocating a new thread, freezing the calling thread until execution completes.
> - `start()` can only be invoked once on a thread instance; calling it a second time throws `IllegalThreadStateException`.
>
> **Key Nuance / Trade-off:**
> - Accidentally calling `run()` compiles cleanly and executes without crashing, but defeats multithreading by serializing execution onto the current thread.
>
> **Reference Concept:** `Thread Execution Mechanics: start() vs run()`

---

### How are threads created in Java?

> **Direct Answer:**
> - **Extending `Thread`:** Subclasses `java.lang.Thread` and overrides `run()`; inflexible due to single inheritance.
> - **Implementing `Runnable`:** Implements `run()`, leaving the class free to extend domain models and enabling Java 8 lambda usage; used for void fire-and-forget tasks.
> - **Implementing `Callable<V>`:** Implements `V call()`, allows returning values and throwing checked exceptions; submitted to an `ExecutorService` and accessed via `Future<V>`.
>
> **Key Nuance / Trade-off:**
> - Enterprise applications avoid extending `Thread` directly; tasks are packaged as `Runnable` or `Callable` and managed via an `ExecutorService` thread pool.
>
> **Reference Concept:** `Thread Creation Mechanisms (Thread vs Runnable vs Callable)`

---

### Explain Thread Lifecycle in Java

> **Direct Answer:**
> - Defined by `Thread.State`, a thread transitions through six distinct states:
>   - **`NEW`:** Instantiated, but `.start()` not yet called.
>   - **`RUNNABLE`:** Running on CPU or ready in the OS scheduling queue.
>   - **`BLOCKED`:** Waiting to acquire an intrinsic monitor lock to enter `synchronized`.
>   - **`WAITING`:** Suspended indefinitely until signaled (`wait()`, `join()`, `LockSupport.park()`).
>   - **`TIMED_WAITING`:** Suspended with a timeout (`sleep()`, `wait(ms)`).
>   - **`TERMINATED`:** Execution finished or halted by an unhandled exception.
>
> **Key Nuance / Trade-off:**
> - Thread states are non-reversible; once a thread transitions to `TERMINATED`, it cannot re-enter `RUNNABLE`.
>
> **Reference Concept:** `Thread Lifecycle & States (Thread.State)`

---

### What is a Daemon thread in Java, and what is its most critical operational difference from a standard User thread?

> **Direct Answer:**
> - A Daemon thread is a low-priority background support thread (e.g., the Garbage Collector or a heartbeat monitor) created by calling `thread.setDaemon(true)`.
> - The critical operational difference is JVM shutdown semantics: the JVM will not exit while at least one User thread is active, but completely ignores Daemon threads during shutdown.
> - When all User threads exit, the JVM terminates immediately and abruptly halts all running Daemon threads without executing their `finally` blocks.
>
> **Key Nuance / Trade-off:**
> - Never execute I/O operations (file writes, database updates) in Daemon threads, as sudden JVM termination corrupts external resources.
>
> **Reference Concept:** `User Threads vs Daemon Threads`

---

### How do wait(), notify(), and notifyAll() work? Why must they be called from inside a synchronized block?

> **Direct Answer:**
> - Defined in `Object`, `wait()` releases the intrinsic lock and places the thread into the object's Wait Set; `notify()` wakes one arbitrary thread from the Wait Set; `notifyAll()` wakes all waiting threads.
> - They must be called inside `synchronized` to eliminate the **Lost Wake-Up Problem**: without mutual exclusion, a condition check and subsequent `wait()` would not be atomic, allowing a notification to fire and be lost between the check and wait call.
> - Attempting to call them outside a synchronized block throws `IllegalMonitorStateException`.
>
> **Key Nuance / Trade-off:**
> - Always guard `wait()` within a `while` loop rather than an `if` statement to protect against spurious wakeups and re-verify condition validity after `notifyAll()` contention.
>
> **Reference Concept:** `Inter-Thread Communication: wait(), notify(), and notifyAll()`

---

### Why is notifyAll() usually safer than notify(), and what exact problem does it prevent?

> **Direct Answer:**
> - `notify()` wakes exactly one arbitrary thread from the monitor's Wait Set; because the JVM monitor does not track why a thread is waiting, it can wake the wrong thread type (e.g., waking a Consumer instead of a Producer in a bounded queue).
> - This scenario—**Signal Stealing**—causes the awakened thread to see an invalid condition, return to sleep, and leave all other threads permanently asleep, deadlocking the system.
> - `notifyAll()` prevents this by awakening every thread in the Wait Set, guaranteeing that any thread eligible to proceed can compete for the lock.
>
> **Key Nuance / Trade-off:**
> - `notifyAll()` prevents signal stealing at the cost of the "Thundering Herd" problem, where multiple threads wake up only for one to acquire the lock while others return to sleep.
>
> **Reference Concept:** `Signal Stealing & The Safety of notifyAll()`

---

### What is a Condition object in Java, and how does it solve the fundamental limitation of Object.wait()?

> **Direct Answer:**
> - A `Condition` object (created via `ReentrantLock.newCondition()`) splits the single, blind Object Monitor Wait Set into multiple dedicated waiting queues.
> - In a bounded queue, you can create separate conditions: `notFull` for producers and `notEmpty` for consumers.
> - Calling `notEmpty.signal()` targets and wakes strictly one consumer, eliminating signal stealing and avoiding the Thundering Herd overhead of waking all threads.
>
> **Key Nuance / Trade-off:**
> - When using explicit locks, method syntax shifts to `await()`, `signal()`, and `signalAll()`; accidentally calling `wait()` invokes the generic `Object` method and breaks synchronization.
>
> **Reference Concept:** `Explicit Thread Coordination: ReentrantLock & Condition Variables`

---

### Why can't we restart a thread after it has finished execution?

> **Direct Answer:**
> - The Java thread lifecycle is strictly one-way: once `run()` finishes, the thread enters `TERMINATED`, and the OS deallocates the underlying native thread engine and stack.
> - The Java `Thread` object remains in heap memory as a dead "zombie" shell with internal `threadStatus != 0`; calling `start()` again immediately throws `IllegalThreadStateException`.
> - Reusable execution is achieved through thread pools (`ExecutorService`), which intercept worker threads, maintain them in the `RUNNABLE` state, and feed them tasks sequentially so native threads never die.
>
> **Key Nuance / Trade-off:**
> - Java intentionally prevents reviving terminated threads to avoid corrupted execution against deallocated native stacks.
>
> **Reference Concept:** `Thread Termination & The Non-Restartable Rule`

---

### What is ThreadLocal in Java, how does it work, and when should you use InheritableThreadLocal?

> **Direct Answer:**
> - `ThreadLocal` provides thread-confined variables where each thread holds an isolated, independent copy of data, achieving thread safety without locks.
> - Each `Thread` object stores its variables in an internal `ThreadLocalMap` referenced via `Thread.currentThread().threadLocals`.
> - `InheritableThreadLocal` automatically propagates parent thread values to newly created child threads at creation time (useful for trace IDs and security contexts).
>
> **Key Nuance / Trade-off:**
> - `InheritableThreadLocal` fails in thread pools because pooled worker threads are reused rather than created per task, causing tasks to retain stale context from earlier threads; modern Java 21+ replaces this with `ScopedValue`.
>
> **Reference Concept:** `ThreadLocal Architecture & InheritableThreadLocal`

---

### Explain ThreadLocalMap concept

> **Direct Answer:**
> - `ThreadLocalMap` is a specialized hash map nested inside `ThreadLocal` that resides as an instance variable on each `Thread` object.
> - It uses open addressing with linear probing ($i + 1$) for collision resolution and stores entries where the key is a `WeakReference<ThreadLocal<?>>`, but the value is a strong reference.
> - In web servers with thread pools, failing to call `threadLocal.remove()` in a `finally` block leaves strong value references anchored to pooled threads, causing severe memory leaks and `OutOfMemoryError`.
>
> **Key Nuance / Trade-off:**
> - While weak keys allow the `ThreadLocal` variable to be collected, the value remains uncollected until explicitly evicted via `remove()`.
>
> **Reference Concept:** `ThreadLocalMap Internals & Memory Leak Prevention`

---

### What is the difference between Concurrency and Parallelism?

> **Direct Answer:**
> - **Concurrency:** Managing multiple tasks making progress across overlapping timeframes via interleaved execution and task-switching (can execute on a single CPU core).
> - **Parallelism:** Executing multiple computations simultaneously at the exact same physical instant (requires multi-core or multi-CPU hardware).
> - Concurrency is about application structure and responsiveness; parallelism is about physical simultaneous execution throughput.
>
> **Key Nuance / Trade-off:**
> - Multithreading on a single-core CPU provides concurrency via time-slicing, but zero parallelism.
>
> **Reference Concept:** `Concurrency vs Parallelism`

---

### What is the difference between Synchronous and Asynchronous programming, and how do you implement Asynchronous programming in a Java backend?

> **Direct Answer:**
> - **Synchronous (Blocking):** Operations execute sequentially; the calling thread halts and waits until the invoked task completes.
> - **Asynchronous (Non-Blocking):** The calling thread delegates execution to a background worker or non-blocking I/O channel and immediately proceeds; completion is handled via callbacks or events.
> - Backend Java implementations include:
>   - Spring Boot `@Async` method execution.
>   - Java 8 `CompletableFuture` pipelines.
>   - Reactive frameworks (Spring WebFlux / Project Reactor).
>   - Java 21 Virtual Threads (Project Loom) for high-scale thread-per-request workloads.
>
> **Key Nuance / Trade-off:**
> - Asynchronous programming increases throughput by decoupling waiting from execution, but requires careful error handling and context propagation across thread boundaries.
>
> **Reference Concept:** `Synchronous vs Asynchronous Programming Models`

---

### What is instruction reordering, why is it dangerous in multithreading, and how does Java prevent it?

> **Direct Answer:**
> - The compiler, JIT, and CPU hardware rearrange instructions to optimize execution speed, provided single-threaded results remain unchanged.
> - In multithreading, reordering can cause a publication flag (`ready = true`) to execute before data initialization (`data = 10`), causing concurrent readers to process uninitialized data.
> - Java prevents reordering using the **Happens-Before** rules enforced by **Memory Barriers (Fences)**:
>   - Marking fields as `volatile` inserts hardware fences that prevent reordering across the boundary.
>   - Exiting a `synchronized` block enforces a memory fence guaranteeing visibility.
>
> **Key Nuance / Trade-off:**
> - Instruction reordering famously broke legacy Double-Checked Locking; declaring the singleton instance `volatile` guarantees memory barrier protection against partially constructed objects.
>
> **Reference Concept:** `Instruction Reordering, Memory Barriers & Happens-Before Guarantee`

---

### Explain Future and CompletableFuture. Why did Java introduce CompletableFuture if we already had Future?

> **Direct Answer:**
> - Java 5 `Future<V>` acts as a placeholder for async computations, but has fatal limitations: retrieving results via `future.get()` is blocking, and it lacks capabilities for chaining, combining, or async exception handling.
> - Java 8 introduced `CompletableFuture<T>` (implementing `CompletionStage`), enabling an event-driven callback model where pipelines execute non-blockingly.
> - It enables fluent chaining (`thenApply`), combining multiple independent async calls (`thenCombine`), and graceful error fallbacks (`exceptionally`).
>
> **Key Nuance / Trade-off:**
> - `CompletableFuture` eliminates thread-blocking polling, allowing complex asynchronous dependency graphs to be composed declaratively.
>
> **Reference Concept:** `Future vs CompletableFuture Pipeline Orchestration`

---
