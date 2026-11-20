# Module 9: Multithreading - Synchronization & Thread Coordination

## 1. Monitor Mechanics & Intrinsic Locking

### The Object Monitor, Bytecode (monitorenter/monitorexit) & Lock Escalation
Every object in the Java Virtual Machine is intrinsically associated with an internal synchronization construct called a **Monitor** (or intrinsic mutex). The Monitor enforces mutual exclusion: only one thread may hold ownership of an object's monitor at any point in time.

#### Bytecode Implementation (`monitorenter` and `monitorexit`)
When a `synchronized` block is compiled into bytecode, the Java compiler wraps the critical section with two explicit opcodes:
- **`monitorenter`:** When a thread reaches this instruction, it attempts to acquire ownership of the target object's monitor.
  - If the monitor's entry count is 0, the thread acquires the monitor and increments the count to 1.
  - If the thread already owns the monitor, it re-enters and increments the counter (reentrancy).
  - If another thread holds the monitor, the calling thread is suspended into the `BLOCKED` state and enqueued in the monitor's **Entry Set**.
- **`monitorexit`:** Decrements the entry counter. When the count returns to 0, ownership is relinquished, and threads in the Entry Set compete for acquisition.
- **Exception Safety:** The compiler automatically inserts a secondary `monitorexit` instruction inside an implicit `finally` exception handler table. This guarantees that the monitor is released even if runtime exceptions (`NullPointerException`, `ArithmeticException`) occur within the block.

#### Physical Storage: The Object Header Mark Word
The ownership state of an intrinsic lock is stored directly in the heap within the **Object Header**, specifically inside the 64-bit **Mark Word**. When a thread claims a lock, the JVM writes the thread's ID and locking state flags into the Mark Word.

#### Modern JVM Lock Escalation (Biased $\rightarrow$ Lightweight $\rightarrow$ Heavyweight)
To eliminate the severe operating system context-switching overhead of legacy heavyweight locks, modern JVMs utilize adaptive **Lock Escalation**:
1. **Biased Locking:** The JVM assumes that a lock will only ever be accessed by a single thread. It writes the thread ID into the Mark Word without CAS operations. Subsequent acquisitions by that same thread incur zero synchronization overhead.
2. **Lightweight Locking (Spinning via CAS):** If a second thread attempts to acquire the lock, the lock escalates. The JVM avoids asking the OS kernel to suspend the contending thread. Instead, it forces the thread to execute a short busy-wait loop (**spinning**) while attempting atomic Compare-And-Swap (CAS) operations on the Mark Word, anticipating that the lock holder will finish rapidly.
3. **Heavyweight Locking (OS Mutex):** If contention persists beyond the spin threshold, the lock escalates to a Heavyweight Lock. The JVM allocates an OS-level mutex and physically parks the thread into a suspended kernel state, eliminating CPU churn while waiting.

---

### Class-Level Locks vs Object-Level Locks

| Feature | Object-Level Lock (Instance Lock) | Class-Level Lock (Static Lock) |
| :--- | :--- | :--- |
| **Monitored Target** | The specific instance on the heap (`this`) | The `java.lang.Class` blueprint in Metaspace |
| **Syntax Patterns** | `synchronized(this)`<br>`public synchronized void doWork()` | `synchronized(MyClass.class)`<br>`public static synchronized void doWork()` |
| **Lock Independence** | Independent per object instance; Thread 1 locking `accountA` does not block Thread 2 on `accountB` | Global per ClassLoader; Thread 1 locking `MyClass.class` blocks Thread 2 across all instances |
| **Mutual Interference** | Does **NOT** block threads holding a Class-level lock | Does **NOT** block threads holding an Object-level lock |

#### Independence of Class and Object Monitors
A Class-level lock and an Object-level lock operate on completely separate memory monitors. Thread 1 can safely execute a `static synchronized` method (locking `BankAccount.class` to update a global interest rate) while Thread 2 simultaneously executes an instance `synchronized` method (locking `accountA` to deposit funds) without contention.

---

## 2. Explicit Locks & Advanced Thread Routing

### Intrinsic Locks vs java.util.concurrent.locks.Lock

| Feature | Intrinsic Lock (`synchronized`) | Explicit Lock (`Lock` / `ReentrantLock`) |
| :--- | :--- | :--- |
| **Management** | Implicit; managed by JVM bytecode | Explicit; managed programmatically by developer |
| **Acquisition Mode** | Unconditional blocking only | Unconditional, timed (`tryLock(timeout)`), or non-blocking (`tryLock()`) |
| **Release Mechanism** | Automatic upon block exit or exception | Manual via `lock.unlock()` inside a mandatory `finally` block |
| **Interruptibility** | Cannot be interrupted while waiting in line | Can be interrupted using `lockInterruptibly()` |
| **Fairness Guarantee**| Unfair; arbitrary thread selection (risk of starvation) | Can configure strict FIFO fairness (`new ReentrantLock(true)`) |
| **Condition Queues** | Exactly one blind Wait Set per object | Multiple distinct `Condition` instances per lock (`lock.newCondition()`) |

#### The Mandatory `try-finally` Idiom
Because the JVM does not automatically release explicit locks, failing to release in a `finally` block causes permanent lock leakage:
```java
Lock lock = new ReentrantLock();
lock.lock(); // 1. Acquire explicitly
try {
    processPayment(); // 2. Critical business logic
} finally {
    lock.unlock(); // 3. Mandatory release in a finally block
}
```

---

### Reentrant Locking Mechanics (ReentrantLock)
A lock is **reentrant** if a thread that already holds the lock can acquire it again without blocking itself.

#### The Problem It Solves (Self-Deadlock Prevention)
If method `A()` is locked and calls locked method `B()` on the same instance:
```java
public synchronized void methodA() {
    methodB(); // Re-enters lock
}
public synchronized void methodB() {
    // Critical work
}
```
If locks were non-reentrant, the thread would deadlock waiting for itself to release the lock held in `methodA()`.

#### Tracking Mechanism
`ReentrantLock` records the `ownerThread` ID and an internal integer `holdCount`. Each reentrant acquisition increments `holdCount`; each `unlock()` decrements `holdCount`. The lock is only fully relinquished when `holdCount == 0`.

---

### lock() vs tryLock() Behavioral Contracts
- **`lock()` (Unconditional Blocking):** The calling thread waits indefinitely until the lock is acquired. If the current owner hangs, the calling thread is permanently blocked.
- **`tryLock()` (Non-Blocking / Conditional):** Immediately evaluates lock availability. Returns `true` if acquired, or `false` instantly without waiting.

#### Enterprise Architectural Patterns for `tryLock()`
1. **Deadlock Prevention with Backoff:** If a thread fails to acquire secondary locks, it retreats, releases its primary lock, waits a randomized backoff period, and retries.
2. **Graceful Service Degradation:**
   ```java
   if (lock.tryLock(2, TimeUnit.SECONDS)) {
       try {
           fetchLiveDatabaseRecords();
       } finally {
           lock.unlock();
       }
   } else {
       // Primary resource busy; serve stale cache rather than freezing the caller
       fetchFallbackCachedRecords();
   }
   ```
3. **Optional Scheduled Cleanups:** Periodic maintenance tasks abort silently if a resource is actively occupied, deferring until the next cron cycle.

#### The Whiteboard Syntax Trap
Calling `lock.unlock()` when `tryLock()` failed throws `IllegalMonitorStateException`. Always evaluate `tryLock()` in an `if` statement before the `try-finally` block:
```java
// CORRECT PATTERN:
if (lock.tryLock()) {
    try {
        doWork();
    } finally {
        lock.unlock();
    }
} else {
    handleBusyState();
}
```

---

### Read/Write Locks (ReentrantReadWriteLock) & StampedLock
Standard mutual exclusion forces readers to wait for other readers, degrading throughput in read-heavy applications.

#### Rules of Engagement for `ReentrantReadWriteLock`
```
Read  + Read  = ALLOWED (Concurrent non-blocking reads)
Read  + Write = BLOCKED (Writers wait until all active readers exit)
Write + Read  = BLOCKED (Readers wait until active writer completes)
Write + Write = BLOCKED (Exclusive mutual exclusion)
```

#### When to Use and Avoid
- **Use When:** Read operations vastly outnumber writes (e.g., 99% reads, 1% writes) and critical sections take significant computational time.
- **Avoid When:** Write frequency is high or critical sections are microscopic; lock management overhead will exceed the cost of standard `ReentrantLock`.

#### Java 8 `StampedLock` & Optimistic Reading
Under heavy read concurrency, `ReentrantReadWriteLock` can cause **Writer Starvation**. `StampedLock` introduces optimistic reading without acquiring an actual read lock:
```java
long stamp = stampedLock.tryOptimisticRead();
int currentX = this.x;
if (!stampedLock.validate(stamp)) { // Check if a writer intervened
    stamp = stampedLock.readLock(); // Fall back to pessimistic read lock
    try {
        currentX = this.x;
    } finally {
        stampedLock.unlockRead(stamp);
    }
}
```

---

### Coarse-Grained vs Fine-Grained Locking & Lock Striping
- **Coarse-Grained Locking:** Uses a single global lock over an entire subsystem (e.g., locking an entire `DatabaseManager`). Extremely simple and deadlock-free, but creates a single-threaded bottleneck on multicore architectures.
- **Fine-Grained Locking:** Uses separate locks for individual resources (e.g., distinct locks for `userLock` and `invoiceLock`). Delivers maximum concurrency, but dramatically increases the risk of circular deadlocks if lock acquisition order is not strictly deterministic.
- **Lock Striping:** A specialized fine-grained pattern (exemplified by `ConcurrentHashMap`) that partitions a data array into multiple bucket stripes, locking only the specific bucket being accessed.

---

## 3. Advanced Synchronization & Coordination Primitives

### Condition Objects & Targeted Signaling (ReentrantLock.newCondition)
`ReentrantLock.newCondition()` replaces the single blind Wait Set of intrinsic monitors with multiple targeted routing queues.

#### Bounded Buffer Implementation
```java
public class BoundedBuffer {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull  = lock.newCondition(); // Queue for waiting Producers
    private final Condition notEmpty = lock.newCondition(); // Queue for waiting Consumers
    private int count = 0;

    public void put() throws InterruptedException {
        lock.lock();
        try {
            while (count == 100) {
                notFull.await(); // Producers sleep here
            }
            count++;
            notEmpty.signal();   // Specifically wakes one sleeping Consumer
        } finally {
            lock.unlock();
        }
    }

    public void take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await(); // Consumers sleep here
            }
            count--;
            notFull.signal();    // Specifically wakes one sleeping Producer
        } finally {
            lock.unlock();
        }
    }
}
```

---

### Deadlock Mechanisms & Code Scenarios
A deadlock occurs when two or more threads are permanently blocked, each holding a lock that the other thread requires to proceed.

```java
class Resource {}
Resource r1 = new Resource();
Resource r2 = new Resource();

Thread t1 = new Thread(() -> {
    synchronized(r1) {
        System.out.println("T1 locked r1");
        try { Thread.sleep(100); } catch(Exception e){}
        synchronized(r2) { System.out.println("T1 locked r2"); }
    }
});

Thread t2 = new Thread(() -> {
    synchronized(r2) {
        System.out.println("T2 locked r2");
        try { Thread.sleep(100); } catch(Exception e){}
        synchronized(r1) { System.out.println("T2 locked r1"); }
    }
});

t1.start();
t2.start();
```
**The Flaw:** Thread 1 acquires `r1` and waits for `r2`; Thread 2 acquires `r2` and waits for `r1`. Both threads wait forever in mutual deadlock.

---

### The volatile Keyword: Visibility vs Atomicity
- **The Visibility Problem:** In multicore processors, each CPU core caches shared variables in local L1/L2 caches. Updates made by Core 1 remain invisible to Core 2 until flushed.
- **The `volatile` Fix:** Disables CPU caching for the variable. Every read fetches directly from main RAM, and every write flushes immediately to main RAM.
- **What It Does NOT Guarantee (Atomicity):** `volatile` does not make compound operations atomic. Operations like `count++` involve three steps (read, increment, write). Concurrent threads can read the same value and overwrite each other. Use `AtomicInteger` (CAS) for atomicity.

---

### Double-Checked Locking Singleton & Memory Fence Resolution

```java
public class DatabasePool {
    // volatile is mandatory to enforce memory barrier
    private static volatile DatabasePool instance;

    private DatabasePool() {
        if (instance != null) {
            throw new RuntimeException("Reflection attack blocked!");
        }
    }

    public static DatabasePool getInstance() {
        if (instance == null) {                         // Check 1: Fast lock-free check
            synchronized (DatabasePool.class) {        // Synchronize class lock
                if (instance == null) {                 // Check 2: Thread-safe instantiation
                    instance = new DatabasePool();
                }
            }
        }
        return instance;
    }
}
```
**Why Broken Without `volatile`:** The creation `new DatabasePool()` involves: 1. Allocate memory, 2. Run constructor, 3. Assign memory address. Without `volatile`, the CPU can reorder to $1 \rightarrow 3 \rightarrow 2$. Another thread observes `instance != null` at Check 1 before the constructor completes, reading an uninitialized object.

---

### Sequential Thread Execution Strategies
To guarantee that Thread 1 finishes completely before Thread 2 begins:

1. **Using `Thread.join()`:** Calling `t1.join()` causes the main thread to block until `t1` terminates, after which it starts `t2`.
2. **Using Single Thread Executor:** `Executors.newSingleThreadExecutor()` guarantees sequential execution order.
3. **Using `CountDownLatch(1)`:** Thread 2 calls `latch.await()`, and Thread 1 executes `latch.countDown()` at the end of its work.
4. **Using `CompletableFuture`:** `CompletableFuture.runAsync(task1).thenRun(task2)`.
5. **Using `synchronized` with State Flags:** Thread 2 loops on a condition flag that Thread 1 flips inside a synchronized block.

---

### CountDownLatch vs CyclicBarrier

| Feature | `CountDownLatch` | `CyclicBarrier` |
| :--- | :--- | :--- |
| **Coordination Model** | One (or more) threads wait for $N$ tasks to complete | $N$ participating threads all wait for each other at a barrier |
| **Reusability** | **Non-reusable:** Once count reaches 0, the latch is spent | **Reusable (Cyclic):** Automatically resets after tripping |
| **Action Hook** | None | Supports an optional barrier action `Runnable` executed when tripped |
| **Enterprise Use Case**| Backend service startup (wait for DB, Redis, Kafka connections) | Multi-phase parallel data aggregation (split-merge-iterate) |

---

## 4. The Producer-Consumer Architecture

### Producer-Consumer Problem: wait/notify vs BlockingQueue

#### Approach 1: Low-Level `wait()` / `notify()` Coordination
```java
class Queue {
    private int data;
    private boolean available = false;

    synchronized void produce(int value) throws InterruptedException {
        while (available) wait();
        data = value;
        available = true;
        System.out.println("Produced: " + value);
        notify();
    }

    synchronized void consume() throws InterruptedException {
        while (!available) wait();
        System.out.println("Consumed: " + data);
        available = false;
        notify();
    }
}
```

#### Approach 2: Modern Enterprise Solution (`BlockingQueue`)
```java
BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(5);

// Producer Thread
new Thread(() -> {
    IntStream.range(1, 6).forEach(i -> {
        try {
            queue.put(i); // Blocks automatically if queue is full
            System.out.println("Produced: " + i);
        } catch (Exception e) {}
    });
}).start();

// Consumer Thread
new Thread(() -> {
    while (true) {
        try {
            System.out.println("Consumed: " + queue.take()); // Blocks automatically if empty
        } catch (Exception e) {}
    }
}).start();
```

---

### BlockingQueue Architecture & Implementation Mechanics

#### The 4 Method Types Decision Matrix
| Operation Type | Throws Exception | Returns Special Value | Blocks Indefinitely | Blocks with Timeout |
| :--- | :--- | :--- | :--- | :--- |
| **Insert** | `add(e)` | `offer(e)` (`false` if full) | `put(e)` | `offer(e, time, unit)` |
| **Remove** | `remove()` | `poll()` (`null` if empty) | `take()` | `poll(time, unit)` |
| **Examine** | `element()` | `peek()` | *N/A* | *N/A* |

#### ArrayBlockingQueue vs LinkedBlockingQueue
- **`ArrayBlockingQueue` (Single Lock):** Backed by a fixed circular array. Uses a **single `ReentrantLock`** for both reads and writes. Producers and consumers cannot operate simultaneously, resulting in lower throughput but minimal memory allocation.
- **`LinkedBlockingQueue` (Two Separate Locks):** Backed by linked nodes. Uses two distinct locks: `putLock` and `takeLock`. Producers can insert items at the exact instant consumers take items, maximizing concurrency throughput at the cost of Node object GC overhead.
