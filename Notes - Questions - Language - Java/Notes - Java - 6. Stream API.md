# Module 6: Stream API

## 1. Stream Architecture & Pipeline Lifecycle

### Stream Concepts & Three-Phase Pipeline Architecture
Introduced in Java 8, the Stream API represents a paradigm shift from imperative collection iteration (using explicit `for` loops, index counters, and mutable state flags) to a declarative, functional pipeline model.

#### Core Stream Characteristics
- **Not a Data Structure:** A Stream does not store data elements in memory; it acts as an abstraction of computational steps.
- **Non-Mutating Pipeline:** A Stream never modifies the underlying data source; it takes input from a source, passes elements through computational stages, and returns a new transformed representation or aggregated result.
- **Single-Use Consumption:** A Stream can only be traversed once. Once a terminal operation completes, the Stream is permanently consumed and closed. Invoking another operation on a consumed stream throws `java.lang.IllegalStateException: stream has already been operated upon or closed`.

#### The Three Phases of a Stream Pipeline
A complete stream pipeline consists of three sequential lifecycle stages:

```
[1. Source] ────────> [2. Intermediate Operations] ────────> [3. Terminal Operation]
Collection / Array          filter() -> map() (Lazy)              collect() / forEach() (Execution)
```

1. **The Source:** The origin of data elements, such as a `Collection` (`collection.stream()`), an array (`Arrays.stream(array)`), an I/O channel (`Files.lines(path)`), or a generator supplier (`Stream.generate()`).
2. **Intermediate Operations (The Lazy Phase):** Operations that transform a stream into another stream (e.g., `filter()`, `map()`, `sorted()`).
   - **Strict Laziness:** Intermediate operations execute zero CPU work when declared. They simply append an operation node to a linked chain of instructions within the internal pipeline definition.
3. **The Terminal Operation (The Execution Trigger):** Operations that terminate the pipeline and produce a result or side-effect (e.g., `collect()`, `reduce()`, `count()`, `forEach()`).
   - **Fused Pipeline Execution:** The invocation of a terminal operation triggers the JVM to compile the chained intermediate steps into an optimized, single-pass iteration loop, pulling elements through the pipeline one at a time.

#### Code Comparison: Imperative vs Functional Pipeline
```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David");

// Imperative: Hard to read, requires manual state tracking.
// Functional Stream: Clean, thread-safe, and declarative.
List<String> result = names.stream()               // 1. Source
    .filter(name -> name.length() > 4)             // 2. Intermediate (Records instruction)
    .map(String::toUpperCase)                      // 2. Intermediate (Records instruction)
    .collect(Collectors.toList());                 // 3. Terminal (Fires execution!)
```

---

## 2. Stream Operations Classification

### Intermediate vs Terminal Operations Reference

#### 1. Intermediate Operations (Lazy Evaluation)
Intermediate operations return a new `Stream<T>` and defer computation until a terminal operation is called.

| Operation | Category | Method Signature | Description |
| :--- | :--- | :--- | :--- |
| **`filter`** | Stateless | `Stream<T> filter(Predicate<? super T> predicate)` | Retains elements that evaluate to `true` against the predicate. |
| **`map`** | Stateless | `<R> Stream<R> map(Function<? super T, ? extends R> mapper)` | Transforms each element into another type or value ($1:1$ mapping). |
| **`flatMap`** | Stateless | `<R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper)` | Flattens nested streams/collections into a single continuous stream ($1:N$). |
| **`distinct`** | Stateful | `Stream<T> distinct()` | Filters duplicate elements based on `equals()` and `hashCode()`. |
| **`sorted`** | Stateful | `Stream<T> sorted()`, `Stream<T> sorted(Comparator<? super T> comparator)` | Sorts elements according to natural order or a custom `Comparator`. |
| **`peek`** | Stateless | `Stream<T> peek(Consumer<? super T> action)` | Executes an action on each element without altering the stream (for debugging). |
| **`limit`** | Stateful / Short-circuit | `Stream<T> limit(long maxSize)` | Truncates the stream length to at most `maxSize` elements. |
| **`skip`** | Stateful | `Stream<T> skip(long n)` | Discards the first `n` elements of the stream. |
| **`takeWhile`** | Stateful / Short-circuit | `Stream<T> takeWhile(Predicate<? super T> predicate)` | (Java 9+) Accepts elements while predicate is true; aborts on first `false`. |
| **`dropWhile`** | Stateful | `Stream<T> dropWhile(Predicate<? super T> predicate)` | (Java 9+) Drops elements while predicate is true; streams the remainder. |

#### 2. Terminal Operations (Eager Execution)
Terminal operations consume the stream, trigger the pipeline computation, and return a concrete result or side-effect.

| Operation | Short-Circuiting? | Return Type | Description |
| :--- | :--- | :--- | :--- |
| **`collect`** | No | `<R, A> R` | Executes a mutable reduction into a container (`List`, `Set`, `Map`) via `Collector`. |
| **`forEach`** | No | `void` | Traverses every element, executing a side-effecting action (`Consumer`). |
| **`reduce`** | No | `Optional<T>` / `T` | Combines elements into a single immutable value using an associative `BinaryOperator`. |
| **`toArray`** | No | `Object[]` / `A[]` | Collects stream elements into a concrete Java array. |
| **`count`** | No | `long` | Returns the total count of elements passing through the pipeline. |
| **`anyMatch`** | **Yes** | `boolean` | Evaluates if at least one element satisfies the predicate; aborts early on match. |
| **`allMatch`** | **Yes** | `boolean` | Evaluates if all elements satisfy the predicate; aborts early on failure. |
| **`noneMatch`** | **Yes** | `boolean` | Evaluates if zero elements satisfy the predicate; aborts early on match. |
| **`findFirst`**| **Yes** | `Optional<T>` | Returns the first element in encounter order; aborts as soon as found. |
| **`findAny`**  | **Yes** | `Optional<T>` | Returns any matching element (optimized for parallel stream completion). |
| **`min`**      | No | `Optional<T>` | Computes the minimum element according to a `Comparator`. |
| **`max`**      | No | `Optional<T>` | Computes the maximum element according to a `Comparator`. |

---

### Short-Circuiting & Stateful vs Stateless Operations

#### 1. Short-Circuiting Operations
A short-circuiting operation is one that can produce a definitive result without processing every element in the stream.
- **Infinite Streams:** Short-circuiting allows Java Streams to safely operate on unbounded/infinite streams (`Stream.iterate()`) without triggering an `OutOfMemoryError` or infinite loop.
- **Intermediate Short-circuiting:** `limit(n)`, `takeWhile(predicate)` stop passing elements downstream once the termination condition is reached.
- **Terminal Short-circuiting:** `anyMatch()`, `allMatch()`, `noneMatch()`, `findFirst()`, `findAny()` abort traversal as soon as the outcome is mathematically determined.

```java
List<String> names = List.of("John", "Jane", "Jack", "Doe");

boolean found = names.stream()
    .filter(s -> s.startsWith("J"))
    .anyMatch(s -> s.length() > 3);
// Once "John" matches (length > 3), the rest ("Jane", "Jack", "Doe") are never processed.
```

#### 2. Stateless vs Stateful Operations
- **Stateless Operations (`filter`, `map`, `flatMap`, `peek`):**
  - Process each element in complete isolation, requiring no knowledge of preceding or subsequent elements.
  - Require $O(1)$ internal memory per element and are natively parallelizable across CPU cores without thread synchronization.
- **Stateful Operations (`distinct`, `sorted`, `limit`, `skip`):**
  - Require observing or buffering multiple (or all) stream elements before yielding output downstream.
  - `sorted()` must accumulate every element into an internal array buffer before executing quicksort/tim-sort to determine the first sorted element.
  - **Senior Optimization Rule:** Always position stateless filtering operations (`filter()`) before stateful operations (`sorted()`, `distinct()`) to minimize the volume of elements buffered in heap memory.

---

### Core Transformations: filter, map, flatMap, reduce, and collect

#### 1. `filter()` (Intermediate / Lazy Gatekeeper)
Accepts a `Predicate<T>` and discards elements evaluating to `false`. Operates lazily and evaluates elements individually as they are pulled by the terminal operation:
```java
stream.filter(user -> user.getAge() >= 21)
```

#### 2. `map()` (Intermediate / 1:1 Transformation)
Accepts a `Function<T, R>` and transforms each element of type `T` into a corresponding element of type `R`. The total element count remains invariant:
```java
Stream<String> emails = users.stream().map(User::getEmail);
```

#### 3. `flatMap()` (Intermediate / 1:N Stream Flattening)
Used when elements contain nested collections or streams. Standard `map()` on a nested structure yields a stream of streams (`Stream<List<T>>`). `flatMap()` takes each inner collection, extracts its elements, and flattens them into a unified, top-level stream:
```java
// Given: School has List<Student>
// Map produces nested Stream<List<Student>>:
Stream<List<Student>> nestedStream = schools.stream().map(School::getStudents);

// FlatMap flattens into Stream<Student>:
Stream<Student> flatStream = schools.stream()
    .flatMap(school -> school.getStudents().stream());
```

#### 4. `reduce()` (Terminal / Immutable Aggregation)
Combines elements into a single definitive value using an associative accumulator `BinaryOperator<T>`. It is designed for immutable accumulation (numbers, strings):
```java
double total = orders.stream()
    .map(Order::getPrice)
    .reduce(0.0, (subtotal, price) -> subtotal + price);
```
*Note:* Do not use `reduce()` to aggregate elements into mutable collections (e.g., creating a new `ArrayList` on every addition causes $O(n^2)$ memory churn). Use `collect()` instead.

#### 5. `collect()` (Terminal / Mutable Reduction)
Gathers streaming elements into a mutable result container (`List`, `Set`, `Map`, `StringBuilder`) using the `java.util.stream.Collectors` factory:
```java
// 1. To List
List<String> list = stream.collect(Collectors.toList());

// 2. To Map
Map<Long, User> userMap = users.stream()
    .collect(Collectors.toMap(User::getId, Function.identity()));

// 3. Grouping By (Categorization)
Map<Department, List<User>> byDept = users.stream()
    .collect(Collectors.groupingBy(User::getDepartment));
```

---

## 3. Parallel Processing & Concurrency

### Parallel Streams & Common ForkJoinPool Architecture
Introduced in Java 8, Parallel Streams enable concurrent execution of stream operations across multiple processor cores without requiring manual thread management.

#### Internal Mechanics: The ForkJoinPool
- **Underlying Engine:** Parallel streams execute tasks on the shared JVM-wide `ForkJoinPool.commonPool()`.
- **Default Thread Pool Size:** The common pool defaults to:
  $$\text{Threads} = \text{Runtime.getRuntime().availableProcessors()} - 1$$
- **Work-Stealing Algorithm:** The `ForkJoinPool` divides the stream recursively into subtasks (using `Spliterator.trySplit()`), distributes slices across worker thread queues, and utilizes a **work-stealing algorithm** where idle threads steal tasks from the tails of busy thread queues.

#### Creating Parallel Streams
```java
// Approach 1: Convert sequential stream to parallel
stream.parallel().forEach(System.out::println);

// Approach 2: Create directly from a Collection
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
int sum = numbers.parallelStream()
    .mapToInt(Integer::intValue)
    .sum();
System.out.println("Sum: " + sum);
```

#### Senior Considerations & Anti-Patterns
1. **CPU-Intensive vs Blocking I/O:** Parallel streams are optimized for CPU-bound computations with zero blocking. Never execute blocking I/O (database queries, HTTP client requests) inside a parallel stream; doing so starves the shared `ForkJoinPool.commonPool()`, freezing parallel streams across the entire JVM application.
2. **Order Preservation:** Parallel processing executes concurrently, so execution order is non-deterministic. To enforce sequential ordering during terminal operations, use `forEachOrdered()` instead of `forEach()`.
3. **Thread Safety & Shared State:** Operations in parallel streams must be strictly stateless and side-effect-free. Mutating shared collections (e.g., adding to an external `ArrayList`) causes race conditions and data corruption.
4. **Data Size Threshold ($N \times Q$ Rule):** Parallelization incurs substantial thread dispatching and splitting overhead. For small datasets ($N < 10,000$), sequential streams consistently outperform parallel streams. Use parallel streams only when $N \times Q > 10,000$ (where $N$ is element count and $Q$ is computational cost per element).

---

## 4. Sorting & Ordering Contracts

### Comparable vs Comparator Interfaces
Java provides two distinct interfaces for ordering objects within collections and stream pipelines.

| Feature | `java.lang.Comparable<T>` | `java.util.Comparator<T>` |
| :--- | :--- | :--- |
| **Package** | `java.lang` (Core language package) | `java.util` (Collections utility package) |
| **Purpose** | Defines the single **Natural Ordering** of a domain class | Defines **Custom / Alternative Ordering** strategies |
| **Abstract Method** | `int compareTo(T o)` | `int compare(T o1, T o2)` |
| **Implementation** | Implemented directly by the target domain entity class | Implemented in separate helper classes, lambdas, or factory methods |
| **Signature Target** | Compares `this` against another object (`this.compareTo(other)`) | Compares two distinct object arguments (`compare(a, b)`) |
| **Class Modifications** | Requires modifying the original domain entity source code | Does not require modifying the target domain entity class |
| **Flexibility** | Exactly one ordering rule per class | Unlimited distinct ordering strategies can be defined |
| **Java 8+ Enhancements**| Natural comparator: `Comparator.naturalOrder()` | Functional chaining: `Comparator.comparing(User::getAge).thenComparing(User::getName)` |

```java
// 1. Natural ordering via Comparable
public class Employee implements Comparable<Employee> {
    private int id;
    private String name;

    @Override
    public int compareTo(Employee other) {
        return Integer.compare(this.id, other.id);
    }
}

// 2. Custom ordering via Comparator (Java 8 Lambda / Method Reference)
Comparator<Employee> byNameDesc = Comparator
    .comparing(Employee::getName)
    .reversed();

employees.stream().sorted(byNameDesc).collect(Collectors.toList());
```
