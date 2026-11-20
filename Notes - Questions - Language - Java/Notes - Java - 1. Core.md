# Java Core Language, Concurrency, Records & Generics

## Java Language Evolution, LTS Releases & Modular Architecture

### Java Keyword Categorization (Access, Flow Control, OOP, Modifiers, Exceptions)
- **1. Access Modifiers (Visibility & Encapsulation):**
  - `public`: Accessible from any class in any package.
  - `private`: Accessible strictly within the declaring class.
  - `protected`: Accessible within the same package and by subclasses in other packages.
  - *Default (Package-Private)*: Accessible only within the declaring package.
- **2. Flow Control:**
  - *Selection:* `if`, `else`, `switch`, `case`, `default`.
  - *Iteration:* `for`, `while`, `do`.
  - *Transfer:* `break` (exit loop/switch), `continue` (skip iteration), `return` (exit method with optional value).
- **3. Object-Oriented Programming (OOP):**
  - `class`, `interface`, `extends`, `implements`, `abstract`.
  - `this` (current instance), `super` (parent instance), `new` (heap allocation), `instanceof` (type check).
- **4. Non-Access Modifiers (JVM Behavior):**
  - `static`: Belongs to the class rather than instances; shared across all instances.
  - `final`: Declares variables immutable, methods non-overridable, and classes non-extensible.
  - `synchronized`: Mutual exclusion lock ensuring only one thread executes a block/method concurrently.
  - `volatile`: Guarantees main memory visibility and prevents instruction reordering.
  - `transient`: Prevents fields from being serialized by standard Java serialization.
  - `native`: Escapes the JVM to invoke platform-specific C/C++ compiled binaries.
- **5. Exception Handling:**
  - `try`, `catch`, `finally`, `throw`, `throws`.
- **6. Primitive Types & Literals:**
  - Primitives: `byte`, `short`, `int`, `long`, `float`, `double`, `boolean`, `char`.
  - Return: `void`.
  - Literals: `true`, `false`, `null`.

### Major Java LTS Milestones (Java 8, 11, 17, 21, 25)
- **Release Cadence:** Oracle and the OpenJDK community maintain a 2-year Long-Term Support (LTS) cadence for enterprise stability, security patches, and broad vendor support (Adoptium/Temurin, Amazon Corretto).
- **Java 5 (September 2004 - Milestone):** Generics, Enums, Autoboxing/Unboxing, Annotations (`@Override`), enhanced for-loop, Varargs, and the `java.util.concurrent` framework.
- **Java 8 (LTS, March 2014):** Lambda expressions, Stream API, functional interfaces (`java.util.function`), modern Date/Time API (`java.time`), default/static interface methods, and `Optional<T>`.
- **Java 11 (LTS, September 2018):** Standardized HTTP Client (`java.net.http`), local-variable syntax (`var`) in lambda parameters, String utilities (`isBlank`, `lines`, `strip`, `repeat`), and removal of legacy Java EE/CORBA modules.
- **Java 17 (LTS, September 2021):** Sealed classes (`permits`), Pattern Matching for `instanceof`, finalized Switch Expressions, Records (immutable data carriers), strong encapsulation of JDK internals, and macOS Metal rendering.
- **Java 21 (LTS, September 2023):** Virtual Threads (Project Loom), Record Patterns, finalized Pattern Matching for `switch`, Sequenced Collections (`SequencedCollection`, `SequencedMap`), and Generational ZGC.
- **Java 25 (LTS, September 2025):** Flexible Constructor Bodies (statements before `super(...)`), Scoped Values (`ScopedValue`), Stream Gatherers, Class-File API, Compact Object Headers, and finalized Foreign Function & Memory (FFM) API.

### Java Evolution Timeline & Release Feature Matrix
| Version | Major Features Added |
| :--- | :--- |
| **Java 8** | Lambdas, Streams, Method References, `Optional`, Default Methods, `java.time` |
| **Java 9** | JPMS (Modules), Private Interface Methods, JShell, Collection Factory Methods (`List.of`) |
| **Java 10** | Local-Variable Type Inference (`var`) |
| **Java 11** | Standard HTTP Client, `var` in Lambdas, String enhancements (`isBlank`, `repeat`) |
| **Java 14** | Switch Expressions (finalized) |
| **Java 15** | Text Blocks (finalized) |
| **Java 16** | Records (finalized), Pattern Matching for `instanceof` (finalized) |
| **Java 17** | Sealed Classes (finalized), Strong Encapsulation of JDK Internals |
| **Java 21** | Virtual Threads (Project Loom), Sequenced Collections, Record Patterns |

### Java Platform Module System (JPMS, Project Jigsaw & jlink)
- **The Problem with Legacy Classpath:**
  - *JAR Hell (Unreliable Configuration):* The flat classpath caused missing runtime dependencies to fail late in production via `NoClassDefFoundError`.
  - *The Public Loophole (Weak Encapsulation):* Any class marked `public` was exposed to all classes across all JARs, preventing libraries from hiding internal implementation packages.
- **The JPMS Solution (`module-info.java`):**
  - Declares an explicit dependency graph at the module root:
    ```java
    // payment.provider/src/module-info.java
    module payment.provider {
        exports com.payment.api; // Only export public API
        // com.payment.internal is completely hidden from consumers
    }

    // store.app/src/module-info.java
    module store.app {
        requires payment.provider; // Checked at startup
        requires java.sql;
    }
    ```
- **Architectural Advantages:**
  1. **Strong Encapsulation (`exports`):** Classes in unexported packages cannot be accessed outside the module, even via reflection (unless explicitly opened).
  2. **Reliable Configuration (`requires`):** The JVM validates the entire module dependency graph at startup; if a required module is missing, the JVM fails fast immediately.
  3. **Custom Cloud Runtimes (`jlink`):** Analyzes `module-info.java` and generates an ultra-small, custom JRE containing strictly the required JVM modules, slashing Docker image sizes and cloud infrastructure costs.

---

## Java 8 Functional Fundamentals & Null Safety

### Java 8 Core Features (java.time, Stream API, Optional)
- **1. Modern Date & Time API (`java.time`):**
  - Immutable, thread-safe, and ISO-8601 compliant (replaces mutable `java.util.Date` and `Calendar`).
  - Key types: `LocalDate` (date only), `LocalTime` (time only), `LocalDateTime` (date + time), `ZonedDateTime` (time-zone aware), `Instant` (UTC timestamp), `Period` (date-based distance), `Duration` (time-based distance).
  ```java
  LocalDate today = LocalDate.now();
  LocalDate birthday = LocalDate.of(1995, 9, 10);
  Period age = Period.between(birthday, today);
  ```
- **2. The Stream API (`java.util.stream`):**
  - Functional pipelines over collections/arrays; does not store data.
  - **Pipeline Structure:**
    - *Source:* `Collection.stream()`, `Arrays.stream()`, `Stream.of()`.
    - *Intermediate Operations (Lazy):* `filter()`, `map()`, `flatMap()`, `distinct()`, `sorted()`, `limit()`, `skip()`.
    - *Terminal Operations (Eager Execution):* `collect()`, `reduce()`, `forEach()`, `count()`, `anyMatch()`, `findFirst()`.
  ```java
  List<String> names = List.of("John", "Alice", "Bob", "Alex");
  List<String> result = names.stream()
      .filter(n -> n.startsWith("A"))
      .map(String::toUpperCase)
      .collect(Collectors.toList()); // [ALICE, ALEX]
  ```
  - **Parallel Streams:** `collection.parallelStream()` leverages the Common `ForkJoinPool`; best reserved for CPU-intensive calculations over large in-memory datasets.
- **3. Optional Container (`Optional<T>`):**
  - A container object representing the presence or absence of a non-null value to eliminate `NullPointerException`.
  - Methods: `Optional.of(value)` (non-null), `Optional.ofNullable(value)` (null-safe), `orElse(defaultVal)`, `orElseGet(Supplier)`, `orElseThrow()`, `ifPresent(Consumer)`.

### Null Safety Architecture & Avoiding NullPointerExceptions
- **Anti-Pattern:** Returning raw `null` from methods forces callers to write brittle nested `if (obj != null)` checks or risk runtime `NullPointerException`.
- **Modern Null-Safe Idioms:**
  1. Return empty collections (`Collections.emptyList()`) instead of `null` for list/set queries.
  2. Return `Optional<T>` for methods that may legitimately lack a return value:
     ```java
     public Optional<User> findUserById(String id) {
         return Optional.ofNullable(userDatabase.get(id));
     }
     ```
  3. Transform safely with `map()` and `flatMap()`:
     ```java
     String city = findUserById(id)
         .map(User::getAddress)
         .map(Address::getCity)
         .orElse("Unknown City");
     ```
  4. Use `Objects.requireNonNull(arg, "arg must not be null")` in constructors for fail-fast validation.

---

## Core Language Modifiers, Low-Level Memory & JNI

### static, final and Class Constants
- **`static` (Class-Level Scope):**
  - Memory allocated once in the Metaspace when the class is loaded.
  - Accessible via `ClassName.member` without creating object instances.
- **`final` (Immutability & Extensibility Restriction):**
  - *Variables:* Reference or primitive value cannot be reassigned after initialization.
  - *Methods:* Prevents subclasses from overriding the method.
  - *Classes:* Prevents inheritance (e.g., `public final class String`).
- **Class Constants (`public static final`):**
  - Combines class-level single allocation (`static`) with immutability (`final`).
  - Constant names follow uppercase snake-case convention (`public static final int MAX_RETRY_COUNT = 3;`).
  - Replaced inline by the compiler at compile time for primitive/string constants.

### Local Variable Type Inference (var in Variables & Lambda Parameters)
- **Local Variable Inference (`var`, Java 10):**
  - Compiler infers the concrete type from the right-hand initialization expression.
  - Strictly limited to local variables with initializers; forbidden for class fields, method parameters, and return types.
- **`var` in Lambda Parameters (Java 11):**
  - Enables applying type annotations to lambda parameters without writing verbose explicit types:
    ```java
    // Allowed in Java 11+:
    (@NonNull var s1, @Nullable var s2) -> s1.concat(s2);
    ```

### transient, native and volatile Keywords
- **`transient` (Serialization Exclusion):**
  - Informs the JVM serialization engine (`Serializable`) to skip writing the field during object serialization. Restores to default values (`null`, `0`) upon deserialization. Used for passwords, file handles, and derived caches.
- **`native` (Platform Interop):**
  - Declares a method implemented in platform-specific compiled languages (C/C++) via the Java Native Interface (JNI). Has no Java method body.
- **`volatile` (Visibility & Memory Ordering):**
  - Informs the JVM and CPU that reads/writes must go directly to L1/L3 cache-coherent Main Memory, bypassing per-core CPU registers and preventing instruction reordering.

### Java Native Interface (JNI), the native Keyword & JVM Sandbox Escape
- **The Core Problem:** The JVM executes inside a managed, hardware-abstracted sandbox. Standard Java cannot directly invoke CPU instructions, talk to proprietary hardware drivers, or reuse legacy C/C++ libraries.
- **JNI Architecture:** `native` methods declare signatures in Java, while implementation libraries (`.dll`, `.so`) are compiled natively and loaded via `System.loadLibrary()`.
- **Architectural Trade-offs:**
  - *Drawbacks:* Breaks "Write Once, Run Anywhere" portability, exposes JVM to memory corruption and segmentation faults (bypassing GC), and incurs JNI transition invocation overhead.
  - *Modern Alternative:* Java 22+ Foreign Function & Memory (FFM) API replaces JNI with safe, pure-Java native access.

### High-Performance Memory Copying (System.arraycopy vs. For Loops)
- **The For-Loop Bottleneck:** Iterating over an array element-by-element in Java bytecode performs boundary checks on every iteration, loads elements into registers one by one, and generates multiple CPU branch instructions.
- **`System.arraycopy()` Mechanics:**
  - A `native` JVM intrinsic implemented in C++.
  - Uses direct hardware memory block transfers (equivalent to C's `memmove()` or CPU block instructions like `REP MOVSB`).
  - Bypasses per-element bounds checking after a single upfront boundary check, copying contiguous blocks of memory in constant CPU bursts.
- **The Shallow Copy Caveat:** For object arrays, `System.arraycopy()` copies object reference pointers, not the underlying objects themselves.

---

## Thread Safety, Atomic Primitives & Concurrency

### volatile vs. synchronized vs. Atomic Variables
- **`volatile`:**
  - Guarantees **Visibility**: all threads see the most up-to-date value written to main memory.
  - Does **NOT** guarantee Atomicity: compound operations like `count++` (read -> update -> write) remain vulnerable to race conditions.
- **`synchronized`:**
  - Guarantees **Visibility AND Atomicity** via mutual exclusion locks (Monitors).
  - Pessimistic locking: blocks competing threads, causing thread state suspension and OS kernel context-switch overhead.
- **Atomic Classes (`AtomicInteger`, `AtomicLong`):**
  - Guarantees **Visibility AND Atomicity** without thread blocking.
  - Employs non-blocking, optimistic lock-free algorithms driven by hardware CPU instructions.

### AtomicInteger vs. synchronized int (Optimistic vs. Pessimistic Locking)
- **Pessimistic Locking (`synchronized`):** Assumes collision will happen. Threads acquire monitor locks; ungranted threads are suspended by the OS scheduler, incurring heavy context-switching penalties.
- **Optimistic Concurrency (`AtomicInteger`):** Assumes collision is rare. The thread reads the value, computes the update, and attempts to write using an atomic CPU instruction. If another thread intervened, it simply re-reads and loops (spin-lock).

### Hardware-Level Compare-And-Set (CAS) & Lock-Free Concurrency
- **The CAS Instruction:** A hardware-level atomic CPU instruction (e.g., `CMPXCHG` on x86/x64).
- **Three Operands:**
  - $V$: Memory location to read/update.
  - $A$: Expected existing value.
  - $B$: New value to write.
- **Mechanics:** The CPU atomically checks: "Does location $V$ equal expected value $A$?" If yes, it writes $B$ and returns `true`. If no, it aborts without writing and returns `false`.
- **AtomicInteger Internal Spin-Loop:**
  ```java
  public final int getAndIncrement() {
      return U.getAndAddInt(this, VALUE, 1);
  }
  ```
  Under high contention, threads retry in a tight CPU loop without sleeping or context-switching.

### The Concurrency ABA Problem & AtomicStampedReference Resolution
- **The Core Flaw:** A thread reads value `A`. Before it can perform CAS, Thread 2 changes `A` to `B`, and then changes it back to `A`. When Thread 1 executes `CAS(A, C)`, the check passes because memory reads `A`, even though internal state mutated in between.
- **The Solution (`AtomicStampedReference`):**
  - Pairs the object reference with an integer timestamp or version stamp.
  - CAS validates both the expected reference AND the expected stamp:
    $$\text{compareAndSet}(\text{expectedRef}, \text{newRef}, \text{expectedStamp}, \text{newStamp})$$
  - When $A \to B \to A$ occurs, the version stamp advances from $1 \to 2 \to 3$. Thread 1's CAS fails because stamp $1 \ne 3$.

---

## Modern Language Constructs (Pattern Matching & Records)

### Pattern Matching for instanceof
- **Legacy Boilerplate:**
  ```java
  if (obj instanceof String) {
      String s = (String) obj; // Redundant explicit cast
      System.out.println(s.length());
  }
  ```
- **Pattern Matching (Java 16+):**
  ```java
  if (obj instanceof String s) {
      System.out.println(s.length()); // Directly usable
  }
  ```
- **Flow Scoping:** The variable `s` is only in scope where the pattern matches (e.g., inside the `if` block, or in the right-hand condition of `&&`).

### Java Records: Immutability, Compact Constructors & State Validation
- **Record Mechanics:** Introduced in Java 16 as transparent, shallowly immutable data carriers (`public record User(String id, String email) {}`).
- **Compiler Synthesis:** Automatically generates `final` fields, canonical constructor, accessors (`user.id()`), `equals()`, `hashCode()`, and `toString()`.
- **Architectural Restrictions:**
  - Cannot extend other classes (already implicitly extends `java.lang.Record`).
  - Cannot declare additional instance fields.
  - All fields are implicitly `final`.
- **Compact Constructor for Validation:**
  ```java
  public record User(String id, String email) {
      // Compact constructor: no parameter list or manual field assignment
      public User {
          if (id == null || id.isBlank()) {
              throw new IllegalArgumentException("User ID cannot be blank");
          }
          if (email == null || !email.contains("@")) {
              throw new IllegalArgumentException("Invalid email format");
          }
      }
  }
  ```

### Static Fields & Factory Methods in Java Records
- **Why Allowed:** While records strictly forbid additional *instance* variables to preserve immutable state transparency, `static` members belong to the class definition in Metaspace, not the record instance.
- **Use Cases:**
  1. **Domain Constants:** `public static final int MAX_NAME_LENGTH = 100;`
  2. **Class Utilities:** `private static final Logger log = LoggerFactory.getLogger(User.class);`
  3. **Static Factory Methods:** `public static User guest() { return new User("0", "guest@example.com"); }`

---

## Method Mechanics & Constructor State Integrity

### Unhandled Method Return Values
- In Java, invoking a method that returns a value without assigning or evaluating it is completely legal:
  ```java
  list.add("Apple"); // Returns boolean (true), but return value is ignored
  ```
- **Mechanics:** The JVM evaluates the method, pushes the return value onto the operand stack, and immediately issues a `POP` or `POP2` bytecode instruction to discard it.
- **Gotcha:** Ignoring return values on immutable objects (e.g., `string.toUpperCase()`, `bigDecimal.add()`) causes silent logic bugs because the original object is never mutated.

### Constructor Execution Beyond Field Initialization (State Validation & Fail-Fast)
- **Execution Scope:** Constructors are executable bytecode methods (`<init>`) invoked during object creation. They are permitted to run loops, invoke helper methods, and execute validation checks.
- **Fail-Fast Validation:** Constructors should enforce class invariants and state validation. Throwing exceptions (`IllegalArgumentException`, `NullPointerException`) inside constructors guarantees invalid or corrupted objects cannot exist in memory.
- **Constructors Trap:** Avoid calling overridable (non-final, non-private) methods inside constructors; subclasses have not initialized their fields yet when the superclass constructor executes, causing `NullPointerException`.

---

## Java Generics & Runtime Bytecode Mechanics

### Generics Scope & Syntax Placement (Class-Level vs. Method-Level)
- **Class-Level Generics (`public class Box<T>`):**
  - Scoped to the lifecycle of the object instance.
  - Usable across instance fields, method arguments, and return types within the class.
  - *Restriction:* Static methods **cannot** use class-level type parameters because static contexts exist independent of any instance.
- **Method-Level Generics (`public static <T> void print(T[] array)`):**
  - Type parameter `<T>` is declared **before** the return type.
  - Scoped strictly to that single method execution.
  - Mandatory for generic static utility methods.

### Type Erasure Mechanics, Backward Compatibility & Runtime Traps
- **The Concept:** Generics exist strictly at compile time for type safety. During compilation, the Java compiler replaces all generic parameters with `Object` (or their upper bound, e.g., `Number`) and inserts synthetic type casts.
- **Why Type Erasure?** Designed in Java 5 to maintain 100% backward binary compatibility with millions of lines of pre-generic Java 1.4 bytecode.
- **The Three Runtime Traps:**
  1. **No `instanceof` with Parameterized Types:** `if (list instanceof List<String>)` is illegal because the JVM only knows it is a raw `List`. Use `if (list instanceof List<?>)`.
  2. **Cannot Instantiate Type Parameters:** `new T()` is illegal because the erased type at runtime is `Object`.
  3. **No Overloading Solely on Generic Parameters:** `void print(List<String> l)` and `void print(List<Integer> l)` generate identical erased signatures `void print(List l)`, causing compiler collision.
