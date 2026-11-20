# Questions: Stream API

### Explain the Java Stream API and how stream pipelines work.

> **Direct Answer:**
> - Introduced in Java 8, the Stream API provides a declarative, functional approach to data processing, shifting away from imperative `for`-loop mutations.
> - A Stream does not store data and never modifies the underlying data source; it is a single-use pipeline that traverses elements once and closes permanently upon completion.
> - A stream pipeline operates in three phases:
>   1. **Source:** Origin of elements (`Collection`, array, I/O channel).
>   2. **Intermediate Operations:** Lazy transformations (`filter()`, `map()`) that record instructions without processing data.
>   3. **Terminal Operation:** Eager execution trigger (`collect()`, `forEach()`) that compiles intermediate steps into an optimized single pass and consumes the stream.
>
> **Key Nuance / Trade-off:**
> - Chaining dozens of intermediate operations consumes zero CPU cycles until a terminal operation is invoked; reusing a consumed stream throws `IllegalStateException`.
>
> **Reference Concept:** `Stream Concepts & Three-Phase Pipeline Architecture`

---

### What are intermediate and terminal operations in Java Streams?

> **Direct Answer:**
> - **Intermediate Operations (Lazy):** Transform an existing stream into another stream (`filter`, `map`, `flatMap`, `sorted`, `distinct`, `limit`); they defer execution until a terminal operation is invoked.
> - **Terminal Operations (Eager):** Terminate and consume the stream pipeline, producing a concrete result or side-effect (`collect`, `forEach`, `reduce`, `count`, `anyMatch`, `findFirst`).
> - Once a terminal operation executes, the pipeline is closed and cannot be re-operated upon.
>
> **Key Nuance / Trade-off:**
> - Short-circuiting terminal operations (`anyMatch`, `findFirst`) and intermediate operations (`limit`, `takeWhile`) terminate early without processing the full dataset, enabling safe processing of infinite streams.
>
> **Reference Concept:** `Intermediate vs Terminal Operations Reference`

---

### What are Short-circuiting Operations and Stateful vs. Stateless in Stram API

> **Direct Answer:**
> - **Short-Circuiting Operations:** Terminate early without inspecting the entire stream (e.g., intermediate `limit(n)`, `takeWhile()`; terminal `findFirst()`, `anyMatch()`), which is essential for processing large or infinite streams.
> - **Stateless Operations:** Process each element independently with $O(1)$ memory, requiring no awareness of other elements (`filter`, `map`, `flatMap`, `peek`); ideal for parallel streams.
> - **Stateful Operations:** Must buffer multiple or all elements before passing results downstream (`sorted`, `distinct`, `limit`, `skip`), requiring extra heap memory.
>
> **Key Nuance / Trade-off:**
> - Always place stateless filtering operations (`filter()`) before stateful operations (`sorted()`, `distinct()`) to minimize memory buffering and sorting overhead.
>
> **Reference Concept:** `Short-Circuiting & Stateful vs Stateless Operations`

---

### Explain operations such as filter(), map(), flatMap(), reduce() and collect() in Streams.

> **Direct Answer:**
> - **`filter(Predicate)`:** Lazy gatekeeper that retains elements evaluating to `true` and discards the rest.
> - **`map(Function)`:** Lazy $1:1$ transformation converting each element from type `T` to type `R`.
> - **`flatMap(Function)`:** Lazy $1:N$ transformation that extracts and flattens nested collections or streams into a single top-level continuous stream.
> - **`reduce(BinaryOperator)`:** Terminal aggregation that combines stream elements into a single immutable value (e.g., sum, product, min, max).
> - **`collect(Collector)`:** Terminal mutable reduction that packs streaming elements into a container structure (`List`, `Set`, `Map`) via `Collectors`.
>
> **Key Nuance / Trade-off:**
> - Use `reduce()` for immutable value aggregation (like numbers); avoid using `reduce()` to accumulate elements into collections, as creating new collection instances on each step causes $O(n^2)$ memory churn (use `collect()` instead).
>
> **Reference Concept:** `Core Transformations: filter, map, flatMap, reduce, and collect`

---

### What are parallel streams in Java, how do they work internally using the ForkJoinPool, and when should they be used or avoided in applications?

> **Direct Answer:**
> - Parallel streams split data recursively using `Spliterator` and process elements concurrently across multiple CPU cores using the shared `ForkJoinPool.commonPool()`.
> - Created using `collection.parallelStream()` or `stream().parallel()`, they leverage work-stealing algorithms to maximize CPU utilization.
> - They are best suited for CPU-intensive, stateless operations on large datasets ($N \times Q > 10,000$) where element order does not matter.
>
> **Key Nuance / Trade-off:**
> - Never execute blocking I/O (database calls, REST requests) in parallel streams, as doing so exhausts the JVM-wide `ForkJoinPool.commonPool()`; avoid parallel streams on small collections where thread coordination overhead outweighs parallel speedup.
>
> **Reference Concept:** `Parallel Streams & Common ForkJoinPool Architecture`

---

### Difference between Comparable and Comparator interfaces in Java.

> **Direct Answer:**
> - `Comparable<T>` (in `java.lang`) defines the single natural ordering of a class by implementing `int compareTo(T o)` directly inside the domain entity.
> - `Comparator<T>` (in `java.util`) defines multiple custom ordering strategies by implementing `int compare(T o1, T o2)` in separate classes, lambdas, or factory methods without modifying the target entity.
> - `Comparable` compares `this` against another instance; `Comparator` compares two independent object arguments.
>
> **Key Nuance / Trade-off:**
> - Java 8 enhanced `Comparator` with functional composition utilities (`Comparator.comparing()`, `thenComparing()`, `reversed()`), allowing declarative multi-attribute sorting in stream pipelines.
>
> **Reference Concept:** `Comparable vs Comparator Interfaces`

---
