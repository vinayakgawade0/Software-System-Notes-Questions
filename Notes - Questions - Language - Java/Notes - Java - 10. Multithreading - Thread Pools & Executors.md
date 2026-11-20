# Module 10: Multithreading - Thread Pools & Executors

## 1. Thread Pool Architecture & ThreadPoolExecutor

### Pitfalls of Manual Thread Creation & The Executor Solution
Manually instantiating threads via `new Thread().start()` in production environments is a severe architectural anti-pattern that leads to system instability.

#### The Three Critical Hazards of Unbounded Threads
1. **Memory Exhaustion (`OutOfMemoryError`):** In Java, every thread allocates an independent call stack (typically 1 Megabyte by default via `-Xss`). Spawning 10,000 concurrent threads consumes 10 Gigabytes of heap/native RAM solely for thread stack allocations, rapidly causing an `OutOfMemoryError: unable to create new native thread`.
2. **CPU Context-Switching Thrashing:** On an 8-core CPU, only 8 threads can execute physically simultaneously. Creating 1,000 threads forces the operating system into frantic thread preemption and context switching. The CPU spends more computational cycles saving and restoring thread registers than executing business logic, causing application throughput to collapse.
3. **Lack of Lifecycle Control:** Manually created threads run unbounded. They cannot be easily throttled, pooled, cancelled, or monitored. Unhandled runtime exceptions cause silent thread termination without notifying the caller.

#### The Enterprise Solution: Bounded Thread Pools
The `ExecutorService` framework resolves these hazards:
- **Pre-allocation & Throttling:** Manages a bounded number of worker threads (e.g., 50).
- **Task Queuing:** Incoming requests exceeding thread capacity are enqueued in a thread-safe `BlockingQueue`.
- **Worker Recycling:** When a worker finishes a task, it does not terminate. It pulls the next task from the queue, maintaining steady CPU utilization without thread churn.

---

### ExecutorService Framework & Enterprise Roles
Introduced in Java 5 by Doug Lea (`java.util.concurrent`), the Executor framework decouples **Task Creation** (what computation to perform) from **Thread Management** (how and when to execute it).

```
java.util.concurrent.Executor
 └── java.util.concurrent.ExecutorService
      ├── AbstractExecutorService
      │    └── ThreadPoolExecutor (Standard extensible thread pool)
      │    └── ForkJoinPool (Work-stealing engine)
      └── ScheduledExecutorService
           └── ScheduledThreadPoolExecutor (Timer / Delayed pool)
```

#### Primary Enterprise Roles
- **Application Servers (Tomcat / Jetty):** Manages worker thread pools where each incoming HTTP request is dispatched to an available pooled thread.
- **Asynchronous Task Offloading:** Background email delivery, notification dispatch, and telemetry reporting execute asynchronously without blocking HTTP response threads.
- **Parallel Work Aggregation:** Submitting concurrent queries to separate downstream microservices and assembling aggregated API responses.

---

### ThreadPoolExecutor Parameters & Task Routing Flow
`ThreadPoolExecutor` is the foundational execution engine behind standard thread pools.

#### The 6 Core Configuration Parameters
1. **`corePoolSize`:** The baseline number of worker threads maintained in the pool, even if they remain completely idle.
2. **`maximumPoolSize`:** The hard ceiling on the number of concurrent threads allowed during severe workload spikes.
3. **`keepAliveTime`:** The duration that threads spawned above `corePoolSize` can remain idle before being terminated.
4. **`workQueue`:** The `BlockingQueue<Runnable>` holding tasks submitted when all core threads are busy.
5. **`threadFactory`:** Custom factory for thread naming (e.g., `Payment-Worker-%d`), priority assignment, and daemon configuration.
6. **`rejectedExecutionHandler`:** Fallback strategy invoked when the queue is full and threads have reached `maximumPoolSize`:
   - **`AbortPolicy` (Default):** Throws `RejectedExecutionException`.
   - **`CallerRunsPolicy` (Enterprise Best Practice):** Forces the calling thread to execute the task directly, throttling incoming traffic naturally.
   - **`DiscardPolicy` / `DiscardOldestPolicy`:** Drops the task silently or evicts the oldest queued task.

#### Task Routing Execution Flow (The 4 Steps)
When a task is submitted to `ThreadPoolExecutor`, it routes strictly in this sequence:

```
[Task Arrives]
      │
      ▼
active < corePoolSize? ──────Yes─────> [Spawn Core Thread & Execute]
      │ No
      ▼
workQueue.offer() succeeds? ─Yes─────> [Enqueue Task in workQueue]
      │ No (Queue Full)
      ▼
active < maximumPoolSize? ───Yes─────> [Spawn Non-Core Thread & Execute]
      │ No (Max Reached)
      ▼
[Invoke RejectedExecutionHandler]
```

---

### Thread Pool Types (Fixed, Cached, Scheduled)

```java
// 1. FixedThreadPool: Fixed thread count, unbounded queue
ExecutorService fixedEx = Executors.newFixedThreadPool(3);

// 2. CachedThreadPool: Dynamic 0 to MAX threads, SynchronousQueue
ExecutorService cachedEx = Executors.newCachedThreadPool();

// 3. ScheduledThreadPool: Periodic or delayed task execution
ScheduledExecutorService schedEx = Executors.newScheduledThreadPool(2);
schedEx.scheduleAtFixedRate(() -> System.out.println("Heartbeat"), 1, 3, TimeUnit.SECONDS);
```

#### Comparison of Standard Thread Pools
| Pool Type | Core / Max Threads | Underlying Work Queue | Key Architectural Vulnerability |
| :--- | :--- | :--- | :--- |
| **`FixedThreadPool`** | $N$ Core / $N$ Max | `LinkedBlockingQueue` (Unbounded) | **Memory Exhaustion (OOM):** If tasks arrive faster than worker throughput, the unbounded queue balloons indefinitely in RAM. |
| **`CachedThreadPool`** | 0 Core / $\infty$ Max | `SynchronousQueue` (Capacity 0) | **CPU Thrashing:** Spawns a new OS thread for every concurrent task; heavy traffic spikes create thousands of threads, freezing the CPU. |
| **`ScheduledThreadPool`** | $N$ Core / $\infty$ Max | `DelayedWorkQueue` (Min-Heap) | Tasks with unhandled runtime exceptions terminate worker threads, though the pool automatically replaces them. |

---

### Thread Allocation Dynamics in FixedThreadPool
When initializing `Executors.newFixedThreadPool(5)`:
- The pool does **not** pre-allocate 5 physical OS threads into memory at startup.
- Threads are created **lazily on demand** as tasks are submitted, up to the `corePoolSize` limit.
- **Scenario:** If only 3 tasks are submitted to a `newFixedThreadPool(5)`, strictly 3 threads are created in memory. The remaining 2 threads are instantiated only if subsequent concurrent tasks arrive while existing threads are busy.

---

### Multi-Threaded Batch Processing: Concurrent File Reading
Using an explicit bounded thread pool to process files concurrently while capping resource utilization:

```java
ExecutorService ex = Executors.newFixedThreadPool(10);
File folder = new File("/path/to/files");

for (File f : folder.listFiles()) {
    ex.submit(() -> {
        try (BufferedReader br = new BufferedReader(new FileReader(f))) {
            System.out.println(f.getName() + ": " + br.readLine());
        } catch (IOException e) {
            e.printStackTrace();
        }
    });
}
ex.shutdown(); // Graceful shutdown after tasks finish
```
This architecture caps concurrent disk reads to 10 threads, preventing file descriptor exhaustion and uncontrolled memory consumption.

---

## 2. Task Models & Execution APIs

### Runnable vs Callable & Exception Handling Parity

| Feature | `java.lang.Runnable` | `java.util.concurrent.Callable<V>` |
| :--- | :--- | :--- |
| **Introduction** | Java 1.0 | Java 5.0 (`java.util.concurrent`) |
| **Method Signature** | `public void run()` | `public V call() throws Exception` |
| **Return Value** | `void` (Fire-and-forget) | Generic type `V` (Calculate-and-return) |
| **Checked Exceptions** | Cannot throw checked exceptions (must handle internally) | Can declare `throws Exception` back to caller |
| **Direct Thread Support**| Can be passed directly to `new Thread(runnable)` | Must be submitted to an `ExecutorService` |
| **Exception Handling** | **Silent Death:** Unhandled exceptions kill the thread silently | **Captured Safety Net:** Captured and wrapped inside `ExecutionException` |

---

### execute() vs submit() API Comparison

| Feature | `execute(Runnable)` | `submit(Callable / Runnable)` |
| :--- | :--- | :--- |
| **Origin Interface** | `java.util.concurrent.Executor` | `java.util.concurrent.ExecutorService` |
| **Accepted Tasks** | Accepts strictly `Runnable` tasks | Accepts both `Callable<T>` and `Runnable` tasks |
| **Return Type** | `void` | Returns `Future<T>` placeholder |
| **Exception Handling** | **Crashes Worker:** Uncaught exceptions terminate the thread, print stack trace to stderr, and force pool to spawn replacement thread | **Swallowed into Future:** Uncaught exceptions are stored inside the `Future` and re-thrown only when `future.get()` is invoked |

---

### Future API & Resolving the Silent Failure Trap

#### The 5 Methods of `Future<V>`
1. **`get()`:** Blocks calling thread indefinitely until computation finishes.
2. **`get(timeout, unit)`:** Blocks for a specified duration, throwing `TimeoutException` if computation exceeds the threshold.
3. **`isDone()`:** Non-blocking check returning `true` if completed, cancelled, or terminated with exception.
4. **`cancel(mayInterruptIfRunning)`:** Attempts task cancellation, optionally sending an interrupt signal.
5. **`isCancelled()`:** Returns `true` if task was cancelled before natural completion.

#### The Silent Failure Trap of `submit()`
If a task submitted via `submit()` throws a `NullPointerException` and the caller never invokes `future.get()`, the error is permanently swallowed.

#### Strategies to Escape the Trap
- **Strategy A (Defensive Try-Catch):** Wrap task bodies in internal try-catch blocks to log exceptions immediately:
  ```java
  executor.submit(() -> {
      try {
          processPayment();
      } catch (Exception e) {
          logger.error("Async payment execution failed", e);
      }
  });
  ```
- **Strategy B (Batch Collection Verification):** Collect all `Future` receipts and invoke `get()` during post-processing to force exceptions into the open:
  ```java
  for (Future<?> future : futures) {
      try {
          future.get();
      } catch (ExecutionException e) {
          logger.error("Task failed with: ", e.getCause());
      }
  }
  ```
- **Strategy C (CompletableFuture Callback):** Utilize non-blocking `exceptionally()` handlers.

---

### Binding CompletableFuture to Custom Thread Pools
By default, `CompletableFuture.supplyAsync(supplier)` executes on the shared `ForkJoinPool.commonPool()`.
- **The Hidden Threat:** The common pool is shared across the entire JVM. Executing blocking I/O (database queries, external HTTP calls) inside the common pool exhausts worker threads, degrading performance across all parallel streams and async tasks in the JVM.
- **The Custom Pool Pattern:** Always pass a dedicated bounded `ExecutorService` as the explicit second parameter:

```java
ExecutorService customPool = Executors.newFixedThreadPool(10);

CompletableFuture<String> asyncPipeline = CompletableFuture.supplyAsync(() -> {
    return processPayment(); // Safely executes inside customPool
}, customPool)
.exceptionally(ex -> {
    logger.error("Caught payment exception without calling get()", ex);
    return "FALLBACK_STATUS";
});
```

---

## 3. Advanced Schedulers & Work-Stealing Engines

### ScheduledExecutorService Internals & DelayedWorkQueue
`ScheduledThreadPoolExecutor` manages delayed and recurring tasks using an internal priority queue: the `DelayedWorkQueue`.

#### Internal Architecture
1. **The Wrapper (`ScheduledFutureTask`):** Wraps submitted tasks with:
   - `time`: Absolute nanosecond timestamp when the task must execute.
   - `sequenceNumber`: Tie-breaker ensuring FIFO execution for identical timestamps.
   - `period`: Repetition interval for recurring schedules (0 for one-shot tasks).
2. **The Engine (`DelayedWorkQueue`):** Implemented as a **Binary Min-Heap**. Tasks are sorted strictly by their target execution time ($O(\log n)$ insertion/removal); the task due soonest is always at the heap root.
3. **Non-Thrashing Worker Parking:** A worker thread inspects the root task. If `time > now`, the thread calculates the difference and calls `LockSupport.parkNanos(difference)`. The OS parks the thread into a dormant `TIMED_WAITING` state, consuming zero CPU cycles until the timer fires.
4. **Task Recycling in Recurring Schedules:** When `scheduleAtFixedRate()` finishes, the executor updates the task's `time = time + period` and re-inserts the identical `ScheduledFutureTask` back into the heap, avoiding continuous garbage collection allocation.

---

### ForkJoinPool Architecture & The Work-Stealing Algorithm
Introduced in Java 7, `ForkJoinPool` is designed for compute-intensive, divide-and-conquer parallel algorithms (`RecursiveTask` and `RecursiveAction`).

```
Worker Thread 1                    Worker Thread 2 (Idle)
┌─────────────────┐                ┌─────────────────┐
│  Private Deque  │                │  Private Deque  │
│ ┌─────────────┐ │                │ (Empty Queue)   │
│ │ Task (LIFO) │ │ ◄── Owner      └─────────────────┘
│ ├─────────────┤ │     Pops/Pushes         ▲
│ │ Subtask     │ │                         │ Steals FIFO
│ ├─────────────┤ │                         │ (Large chunk)
│ │ Large Task  │ │ ────────────────────────┘
│ └─────────────┘ │ ◄── Stealer
└─────────────────┘
```

#### The Work-Stealing Mechanics
Unlike standard executors that share a single contentious blocking queue, `ForkJoinPool` assigns **every worker thread its own private Double-Ended Queue (Deque)**.
1. **Owner Thread (LIFO Stack Operation):** The owning thread pushes and pops sub-tasks from the **top (head)** of its private deque. Operating in Last-In-First-Out order optimizes CPU L1/L2 cache locality, processing the smallest, most recently split tasks immediately.
2. **Stealer Thread (FIFO Queue Operation):** When an idle thread empties its own deque, it randomly inspects another thread's deque and **steals work from the bottom (tail)** in First-In-First-Out order.
3. **Why Steal from the Bottom?** Tasks at the bottom are the oldest and largest un-split chunks. Stealing a large chunk guarantees the stealer has substantial work to divide, minimizing subsequent theft attempts.
4. **Lock-Free Concurrency:** Because the owner operates at the top and the stealer operates at the bottom, operations proceed almost entirely lock-free.

#### When to Avoid ForkJoinPool
`ForkJoinPool` is strictly tied to physical CPU core counts. **Never execute blocking I/O (database, network) in a ForkJoinPool.** Blocking worker threads exhausts the pool, paralyzing parallel streams and async processing across the entire JVM.
