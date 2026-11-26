# Module 8: Multithreading - Thread & Asynchronous

## 1. Thread Fundamentals & Creation

### Multithreading Concepts & Process vs Thread
Multithreading in Java enables concurrent execution of multiple lightweight execution paths within a single process, maximizing CPU utilization and system responsiveness.

#### Core Structural Concepts
- **Process:** An executing operating system program with isolated, dedicated memory space (separate heap and runtime environment).
- **Thread:** The smallest dispatchable unit of execution within a process. Multiple threads within the same process share common heap memory, enabling high-speed inter-thread communication, but introducing synchronization and race condition hazards.

```java
class Task implements Runnable {
    @Override
    public void run() {
        System.out.println("Running in: " + Thread.currentThread().getName());
    }
}

public class Main {
    public static void main(String[] args) {
        Thread t1 = new Thread(new Task());
        Thread t2 = new Thread(new Task());

        t1.start(); // Asynchronously launches thread 1
        t2.start(); // Asynchronously launches thread 2
    }
}
```

---

### Thread Execution Mechanics: start() vs run()

| Aspect | `thread.start()` | `thread.run()` |
| :--- | :--- | :--- |
| **Execution Mode** | **Asynchronous:** Spawns a new operating system thread | **Synchronous:** Executes directly on the current calling thread |
| **JVM Action** | Requests OS thread allocation; JVM calls `run()` on the new thread | Acts as an ordinary method invocation; no new thread is created |
| **Calling Thread Impact** | Calling thread immediately continues execution without waiting | Calling thread blocks and freezes until `run()` completes |
| **Invocation Limit** | Can be called **exactly once** per thread instance | Can be called manually unlimited times |
| **Repeated Call Exception** | Throws `java.lang.IllegalThreadStateException` | Executes without exception |

---

### Thread Creation Mechanisms (Thread vs Runnable vs Callable)

#### 1. Extending `java.lang.Thread` (Legacy Approach)
Subclasses `Thread` and overrides `run()`.
- **Architectural Flaw:** Java enforces single class inheritance. Extending `Thread` prevents the class from extending domain entities, tightly coupling business logic to low-level threading infrastructure. Discouraged in enterprise design.

```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Executing in Thread extension");
    }
}
new MyThread().start();
```

#### 2. Implementing `java.lang.Runnable` (Standard Approach)
Implements the `@FunctionalInterface` `Runnable` and passes it to a `Thread` or thread pool.
- **Advantages:** Leaves inheritance open for domain models. Can be written as a concise Java 8 lambda expression.
- **Limitation:** `run()` returns `void` and cannot throw checked exceptions; strictly suited for fire-and-forget tasks.

```java
// Lambda execution:
new Thread(() -> System.out.println("Executing via Runnable Lambda")).start();
```

#### 3. Implementing `java.util.concurrent.Callable<V>` (Enterprise Standard)
Introduced in Java 5, `Callable` is designed for background computations that must return a value or throw checked exceptions.
- **Execution Mechanism:** Cannot be passed directly to `new Thread()`; must be submitted to an `ExecutorService`.
- **The `Future<V>` Handle:** The pool immediately returns a `Future<V>` placeholder. The main thread can perform other work and later retrieve the result via the blocking method `future.get()`.

```java
Callable<String> expensiveTask = () -> {
    Thread.sleep(2000); // Simulate network or database work
    return "Query Result";
};

ExecutorService executor = Executors.newFixedThreadPool(2);
Future<String> futureResult = executor.submit(expensiveTask);

try {
    // Blocks calling thread until background calculation completes:
    String result = futureResult.get();
    System.out.println("Retrieved: " + result);
} catch (Exception e) {
    e.printStackTrace();
} finally {
    executor.shutdown();
}
```

---

## 2. Thread Lifecycle & Coordination

### Thread Lifecycle & States (Thread.State)
In Java, a thread transitions through six distinct lifecycle states managed by the JVM and OS scheduler:

```
[NEW] ──start()──> [RUNNABLE] <──Lock Acquired── [BLOCKED]
                       │                           ▲
                 wait()/join()                  wait lock
                       ▼                           │
                   [WAITING] ──────────────────────┘
                       │
              wait(time)/sleep(time)
                       ▼
                [TIMED_WAITING]
                       │
             run() finishes / uncaught exception
                       ▼
                  [TERMINATED]
```

1. **`NEW`:** The thread object is instantiated (`new Thread()`), but `.start()` has not yet been invoked.
2. **`RUNNABLE`:** `.start()` was called. The thread is either actively executing on a CPU core or queued in the OS scheduler ready pool waiting for a CPU time-slice.
3. **`BLOCKED`:** The thread is suspended, waiting to acquire an intrinsic monitor lock to enter a `synchronized` block/method.
4. **`WAITING`:** The thread is waiting indefinitely for another thread to perform a specific action (entered via `Object.wait()`, `Thread.join()`, or `LockSupport.park()`).
5. **`TIMED_WAITING`:** The thread is waiting for a specified duration (entered via `Thread.sleep(ms)`, `Object.wait(ms)`, `Thread.join(ms)`, or `LockSupport.parkNanos()`).
6. **`TERMINATED`:** Execution of `run()` has completed normally, or terminated abruptly due to an uncaught exception. A terminated thread cannot be revived.

---

### User Threads vs Daemon Threads

| Feature | User Thread (Default) | Daemon Thread |
| :--- | :--- | :--- |
| **Role & Purpose** | Executes core application business logic | Background support service (e.g., GC, metrics, heartbeat) |
| **JVM Shutdown Impact** | The JVM **will not exit** as long as at least one User thread is alive | The JVM **completely ignores** Daemon threads during shutdown |
| **Termination Semantics** | Gracefully runs until `run()` completes or unhandled exception occurs | Instantly and abruptly terminated when all User threads exit |
| **`finally` Execution** | Guaranteed to execute `finally` blocks upon exit | JVM exit terminates thread immediately; `finally` blocks **do not execute** |
| **Configuration** | Default configuration for all created threads | Must call `thread.setDaemon(true)` prior to `thread.start()` |
| **Critical Hazard** | None | **Never perform I/O in Daemon threads** (sudden JVM shutdown causes data corruption) |

---

### Thread Termination & The Non-Restartable Rule
Once a thread completes execution and transitions to `TERMINATED`, it cannot be restarted. Attempting to invoke `.start()` on a terminated thread immediately throws `java.lang.IllegalThreadStateException`.

#### Why Threads Cannot Be Restarted
1. **OS Thread Deallocation:** The Java `Thread` instance is merely a heap wrapper object. Calling `start()` links it to a native OS thread engine and stack. When `run()` exits, the OS engine destroys the thread stack and reclaims system resources. The Java object remains as an empty "zombie" shell.
2. **Internal State Check:** The JVM checks `threadStatus != 0`. When a thread has already started or died, `threadStatus` is non-zero, triggering the exception.
3. **Thread Pool Solution:** High-throughput servers never recreate or restart threads. Instead, an `ExecutorService` maintains worker threads in the `RUNNABLE` state, feeding them tasks sequentially from a blocking work queue so native OS threads never terminate.

---

### Inter-Thread Communication: wait(), notify(), and notifyAll()
The `wait()`, `notify()`, and `notifyAll()` methods are declared in `java.lang.Object` (not `Thread`) because they manipulate the monitor lock associated with a specific memory object.

#### Method Roles
- **`wait()`:** The thread holding the monitor releases the lock, ceases execution, and enters the object's **Wait Set**.
- **`notify()`:** Wakes up **one random thread** from the object's Wait Set, moving it to the **Entry Set** to compete for lock acquisition. The calling thread retains the lock until its synchronized block exits.
- **`notifyAll()`:** Wakes up **all threads** in the Wait Set, transferring them to the Entry Set.

#### The Lost Wake-Up Problem (Why `synchronized` Is Mandatory)
Invoking `wait()` or `notify()` outside a synchronized block throws `java.lang.IllegalMonitorStateException`.
- **The Race Condition:** If locks were not required, a Consumer thread could evaluate `if (queue.isEmpty()) == true`. Before it calls `wait()`, the OS preempts the thread. A Producer inserts an item and calls `notify()`. Because the Consumer has not yet entered the wait state, the notification signal is lost. When the Consumer resumes and calls `wait()`, it sleeps indefinitely despite data being available.
- Wrapping the condition check and `wait()` in `synchronized` ensures the check and state transition occur as an atomic transaction.

#### The Mandatory `while` Loop Guard
Always guard `wait()` with a `while` loop, never an `if` statement:
```java
synchronized (queue) {
    while (queue.isEmpty()) { // Mandatory while loop
        queue.wait();
    }
    return queue.remove();
}
```
**Why an `if` Statement Fails:**
1. **Spurious Wakeups:** Under certain OS conditions, threads can wake up from `wait()` without any explicit `notify()` invocation.
2. **notifyAll() Contention:** If `notifyAll()` awakens five consumer threads for a single available item, the first thread claims the item; the remaining four threads must loop back, observe `queue.isEmpty() == true`, and re-enter `wait()`.

---

### Signal Stealing & The Safety of notifyAll()
The Object Monitor maintains a single blind Wait Set bucket for all waiting threads, regardless of what condition they are awaiting.

#### The Threat of Signal Stealing
In a bounded buffer with both Producer and Consumer threads waiting in the same Wait Set:
1. Producer A and Producer B wait because the buffer is full.
2. Consumer A takes an item, frees a slot, and calls `notify()`.
3. The JVM randomly selects Consumer B (instead of a Producer).
4. Consumer B sees the buffer is empty, re-enters `wait()`, and goes back to sleep.
5. The wake-up signal is permanently stolen; Producers continue sleeping, and the entire application enters **Deadlock**.

#### The `notifyAll()` Solution
Calling `notifyAll()` awakens every thread in the Wait Set. Consumer B realizes the queue is empty and sleeps; Producer A acquires the lock, fills the slot, and keeps the pipeline alive.

---

### Explicit Thread Coordination: ReentrantLock & Condition Variables
`java.util.concurrent.locks.Condition` (created via `ReentrantLock.newCondition()`) overcomes the single Wait Set limitation of `Object.wait()`.

#### Precision Signaling Architecture
Instead of dumping all threads into one blind wait set, `Condition` allows creating distinct, dedicated waiting queues:
```java
Lock lock = new ReentrantLock();
Condition notFull = lock.newCondition();  // Waiting queue for Producers
Condition notEmpty = lock.newCondition(); // Waiting queue for Consumers
```

- When a Producer fills a slot, it calls `notEmpty.signal()`, waking **strictly one Consumer**.
- When a Consumer frees a slot, it calls `notFull.signal()`, waking **strictly one Producer**.
- This eliminates the Thundering Herd context-switching overhead of `notifyAll()` and eliminates signal stealing.

#### Method Equivalencies
| Intrinsic Monitor (`synchronized`) | Explicit Lock (`ReentrantLock` + `Condition`) |
| :--- | :--- |
| `Object.wait()` | `Condition.await()` |
| `Object.notify()` | `Condition.signal()` |
| `Object.notifyAll()` | `Condition.signalAll()` |

---

## 3. Thread-Local Storage & Scoped Values

### ThreadLocal Architecture & InheritableThreadLocal
`ThreadLocal` provides thread-confined variables. Each thread accessing a `ThreadLocal` holds an isolated, independent copy of the variable, achieving thread safety without locks.

#### Architectural Mechanics
`ThreadLocal` does not store values inside a central synchronized map. Instead:
- Every `Thread` object contains a private package-private field: `ThreadLocal.ThreadLocalMap threadLocals`.
- Calling `threadLocal.get()` extracts `Thread.currentThread().threadLocals` and retrieves the value using the `ThreadLocal` instance as the lookup key.

#### `InheritableThreadLocal`
A subclass of `ThreadLocal` that automatically copies parent thread context to newly spawned child threads upon creation.
- **Use Cases:** Propagating trace IDs, correlation IDs (MDC), or security tokens down synchronous thread hierarchies.
- **The Thread Pool Disaster:** Thread pools reuse threads rather than creating new ones. Context is only inherited when a worker thread is initially instantiated, causing subsequent tasks executed by that pooled thread to retain stale parent contexts.

---

### ThreadLocalMap Internals & Memory Leak Prevention

#### Internal Structure
- **Linear Probing:** Uses an open-addressed array with linear probing ($i + 1$) instead of linked chaining to optimize CPU cache locality.
- **Weak Keys vs Strong Values:** Entries extend `WeakReference<ThreadLocal<?>>`. The key is a weak reference, but the stored value is a **strong reference**.

```
Thread Object ──> ThreadLocalMap ──> Entry[i]
                                      ├── Key   (WeakReference to ThreadLocal)
                                      └── Value (Strong Reference to Object)
```

#### The Thread Pool Memory Leak Trap
When a request thread finishes in an application server (Tomcat, Undertow), the thread returns to the pool without dying. While the `ThreadLocal` key may be garbage collected (turning the key `null`), the **strong reference to the value remains anchored to the alive thread's map**. As requests pass through, abandoned values accumulate, triggering an `OutOfMemoryError`.

#### Safe Usage Pattern (Mandatory `finally` Block)
```java
public class SecurityContextHolder {
    private static final ThreadLocal<String> currentUser = new ThreadLocal<>();

    public static void set(String user) { currentUser.set(user); }
    public static String get() { return currentUser.get(); }

    public static void clear() {
        currentUser.remove(); // CRITICAL: Evicts entry from ThreadLocalMap
    }
}

// Enterprise usage in Filters / Interceptors:
public void handleRequest(Request req) {
    try {
        SecurityContextHolder.set(req.getUser());
        processRequest(req);
    } finally {
        // Guaranteed cleanup before thread returns to the pool
        SecurityContextHolder.clear();
    }
}
```

---

## 4. Concurrency Models & Asynchronous Execution

### Concurrency vs Parallelism

| Concept | Concurrency | Parallelism |
| :--- | :--- | :--- |
| **Definition** | Multiple tasks making progress across overlapping timeframes (interleaved) | Multiple tasks executing at the exact same physical instant |
| **Execution Focus** | Task switching, context management, structuring responsive systems | True simultaneous hardware execution |
| **Hardware Requirement**| Can execute on a single CPU core via rapid time-slicing | Requires multi-core processors or multi-socket CPUs |
| **Real-World Analogy** | A single chef switching back and forth between two dishes | Two chefs cooking two dishes simultaneously in parallel |

---

### Synchronous vs Asynchronous Programming Models
- **Synchronous (Blocking):** Execution proceeds strictly sequentially. When a thread initiates an I/O operation, it is suspended, sitting idle in memory until the operation returns.
- **Asynchronous (Non-Blocking):** The initiating thread delegates work to a background worker or non-blocking OS I/O channel and immediately proceeds to execute subsequent instructions. Completion is handled via callbacks, events, or futures.

#### Implementing Async in Enterprise Java
1. **Spring `@Async`:** Annotate methods with `@Async` to have Spring route execution to a managed thread pool.
2. **`CompletableFuture` (Java 8+):** Functional async composition using event-driven pipeline stages.
3. **Reactive Pipelines (Project Reactor / Spring WebFlux):** Event-loop execution avoiding thread-per-request models.
4. **Virtual Threads (Java 21 Project Loom):** Lightweight JVM-managed threads allowing synchronous code to execute with asynchronous scalability.

---

### Instruction Reordering, Memory Barriers & Happens-Before Guarantee

#### The Threat of Instruction Reordering
To maximize execution throughput, the Java Compiler, JIT Compiler, and CPU hardware reorder machine instructions, provided the reordering does not alter the outcome of a single-threaded execution.

```java
int data = 0;
boolean ready = false;

// Thread 1
public void write() {
    data = 10;     // Instruction A
    ready = true;  // Instruction B
}

// Thread 2
public void read() {
    if (ready) {
        System.out.println(data); // Can print 0 if B is reordered before A!
    }
}
```

#### The Happens-Before Relationship & Memory Fences
The Java Memory Model (JMM) defines **Happens-Before** rules to guarantee visibility and ordering:
- **`volatile` Memory Fence:** Marking a variable as `volatile` inserts hardware memory barriers (LoadStore, StoreStore, StoreLoad) preventing instructions from reordering across the barrier and flushing writes to main memory.
- **`synchronized` Monitor Fence:** Exiting a synchronized block enforces a store fence, ensuring all writes happen-before any subsequent thread acquires the same lock.

---

### Future vs CompletableFuture Pipeline Orchestration

#### Limitations of Java 5 `Future<V>`
- **Blocking Retrieval:** `future.get()` blocks the calling thread indefinitely until completion.
- **No Chaining:** Cannot trigger downstream actions upon completion without polling.
- **No Composition:** Cannot wait for multiple futures to complete and merge results without manual thread coordination.
- **No Async Error Recovery:** Lacks functional exception handling pipelines.

#### Java 8 `CompletableFuture<T>` Capabilities
Implements `CompletionStage<T>`, providing a non-blocking, event-driven callback architecture:
```java
CompletableFuture.supplyAsync(() -> fetchUser(userId))
    .thenApply(user -> user.getEmail())                         // Non-blocking transformation
    .thenCombine(fetchPromoCode(), (email, promo) -> send(email, promo)) // Combine 2 futures
    .exceptionally(ex -> {                                      // Graceful fallback
        logger.error("Failed to process", ex);
        return "Default Notification";
    });
```

## `CompletableFuture` & Asynchronous Programming (Java 8+)

### 1. Why `CompletableFuture` Over `Future`?
Traditional `java.util.concurrent.Future` has major limitations:
- **Blocking retrieval:** Calling `future.get()` blocks the calling thread until completion.
- **No chaining/callbacks:** Cannot attach non-blocking transformations (`thenApply`) when a result arrives.
- **No composition:** Difficult to combine multiple dependent or independent async tasks cleanly.
- **No declarative error handling:** Exceptions are wrapped inside `ExecutionException` only when `.get()` is called.
- **Cannot be completed manually:** `CompletableFuture<T>` implements both `Future<T>` and `CompletionStage<T>`, allowing manual completion via `complete(value)` or `completeExceptionally(ex)`.

---

### 2. Thread Pool Execution (`ForkJoinPool` vs. Custom `Executor`)
- By default, `*Async` methods without an `Executor` argument run on `ForkJoinPool.commonPool()` (sized to `Runtime.getRuntime().availableProcessors() - 1`).
- **Production Best Practice:** Always pass a dedicated `ExecutorService` for blocking I/O tasks (REST calls, DB queries) so slow I/O does not starve `commonPool()` used by parallel streams and other async tasks.

---

### 3. Core API Categories

#### A. Creating Asynchronous Tasks
- `CompletableFuture.runAsync(Runnable, executor)`: Runs a background task that returns no result (`CompletableFuture<Void>`).
- `CompletableFuture.supplyAsync(Supplier<T>, executor)`: Runs a background task that returns a value (`CompletableFuture<T>`).
- `CompletableFuture.completedFuture(value)`: Returns an already-completed future (useful for caching or mock fallbacks).

#### B. Transforming & Consuming Results (Callbacks)
- `thenApply(Function<T, U>)`: Transforms the result synchronously and returns `CompletableFuture<U>` (analogous to `Stream.map()`).
- `thenAccept(Consumer<T>)`: Consumes the result without returning a value (`CompletableFuture<Void>`).
- `thenRun(Runnable)`: Executes an action after completion without accessing the previous result.
- *Note:* `*Async` variants (`thenApplyAsync`, `thenAcceptAsync`) execute the callback on another worker thread (or custom `Executor`) rather than the thread that completed the previous stage.

#### C. Composing Dependent & Independent Futures
- **`thenCompose(Function<T, CompletionStage<U>>)`:** Chains two **dependent** async calls sequentially without nesting `CompletableFuture<CompletableFuture<U>>` (analogous to `Stream.flatMap()`).
- **`thenCombine(otherFuture, BiFunction<T, U, R>)`:** Runs two **independent** futures in parallel and merges their results once both complete.
- **`allOf(cf1, cf2, ...)`:** Waits for **all** provided futures to complete (`CompletableFuture<Void>`); results are extracted individually via `.join()`.
- **`anyOf(cf1, cf2, ...)`:** Completes as soon as **any one** of the provided futures completes (`CompletableFuture<Object>`).

#### D. Exception Handling & Timeouts (Java 9+)
- **`exceptionally(Function<Throwable, T>)`:** Recovers from an exception by returning a fallback value (similar to `catch`).
- **`handle(BiFunction<T, Throwable, U>)`:** Always executes on both success and failure (`result` or `ex` will be non-null) and transforms the outcome.
- **`whenComplete(BiConsumer<T, Throwable>)`:** Inspects result or exception for logging/metrics without altering the outcome.
- **`orTimeout(long timeout, TimeUnit)` (Java 9+):** Fails the future with `TimeoutException` if not completed within the deadline.
- **`completeOnTimeout(T defaultValue, long timeout, TimeUnit)` (Java 9+):** Resolves with a fallback default value if the deadline expires.

---

### 4. Real-World Enterprise Use Cases

#### Use Case 1: Parallel Microservice / API Aggregation (Scatter-Gather)
Fetching user profile, account balances, and recent orders concurrently for a dashboard endpoint instead of sequential blocking calls:
```java
ExecutorService ioExecutor = Executors.newFixedThreadPool(10);

CompletableFuture<UserProfile> profileFuture =
    CompletableFuture.supplyAsync(() -> userClient.getProfile(userId), ioExecutor);

CompletableFuture<List<Order>> ordersFuture =
    CompletableFuture.supplyAsync(() -> orderClient.getRecentOrders(userId), ioExecutor)
                     .completeOnTimeout(List.of(), 2, TimeUnit.SECONDS);

CompletableFuture<DashboardResponse> dashboardFuture =
    profileFuture.thenCombine(ordersFuture, DashboardResponse::new);

DashboardResponse response = dashboardFuture.join();
```
*Impact:* Total latency drops from `sum(latencies)` to `max(latencies)`, with graceful degradation if the non-critical orders service times out.

#### Use Case 2: Dependent Multi-Step Workflow (`thenCompose`)
Authenticating to obtain a token, then fetching customer portfolio data using that token:
```java
CompletableFuture<Portfolio> portfolioFuture =
    CompletableFuture.supplyAsync(() -> authClient.fetchToken(clientId), ioExecutor)
        .thenCompose(token -> CompletableFuture.supplyAsync(
            () -> portfolioClient.getPortfolio(token, accountId), ioExecutor))
        .exceptionally(ex -> Portfolio.emptyFallback());
```

#### Use Case 3: Parallel Batch Processing with `allOf`
Processing a batch of financial documents or notifications concurrently and collecting all results:
```java
List<CompletableFuture<ProcessResult>> futures = documents.stream()
    .map(doc -> CompletableFuture.supplyAsync(() -> processor.process(doc), ioExecutor))
    .toList();

CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();

List<ProcessResult> results = futures.stream()
    .map(CompletableFuture::join)
    .toList();
```

#### Use Case 4: Racing Redundant Replicas / Cache vs. Database (`anyOf`)
Querying two read replicas or cache nodes simultaneously and returning whichever responds first.
