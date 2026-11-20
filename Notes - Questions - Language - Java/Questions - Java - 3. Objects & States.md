# Questions: Objects & States

### What is the difference between equals() and hashCode() in Java?

> **Direct Answer:**
> - `equals(Object o)` determines logical equality between two objects, checking if their state represents the same logical business entity.
> - `hashCode()` returns a 32-bit integer that represents the object in memory and is used to compute bucket indices in hash-based collections (`HashMap`, `HashSet`).
> - By contract, if two objects evaluate to equal via `equals()`, their `hashCode()` outputs must be strictly identical.
>
> **Key Nuance / Trade-off:**
> - Equal hash codes do not imply equal objects; hash collisions are mathematically inevitable and handled via linked lists or red-black trees in bucket storage.
>
> **Reference Concept:** `equals() vs hashCode() Contract`

---

### Difference between == and equals() in Java.

> **Direct Answer:**
> - The `==` operator checks reference identity for objects (verifying whether both references point to the exact same heap memory address) and checks primitive bit values directly.
> - The `equals()` method evaluates logical equality based on internal field data, provided the class overrides the default implementation inherited from `Object`.
>
> **Key Nuance / Trade-off:**
> - Because `==` is an operator, it cannot be overridden, whereas `equals()` is polymorphic and can be customized to define semantic equivalence.
>
> **Reference Concept:** `== Operator vs equals() Method`

---

### What happens if you override equals() but not hashCode()?

> **Direct Answer:**
> - The class violates the Java object contract, causing logically equal objects to inherit distinct default memory-address-based hash codes from `Object`.
> - Hash-based collections (`HashMap`, `HashSet`) route equal keys to completely different internal buckets, resulting in silent lookup failures where `get()` returns `null`.
>
> **Key Nuance / Trade-off:**
> - Over time, putting duplicate logical keys into the map leads to silent duplicate key accumulation, severe memory bloat, and eventually an `OutOfMemoryError`.
>
> **Reference Concept:** `Contract Violations: Overriding equals() Without hashCode()`

---

### Why must equal objects have the same hash code, but objects with same hash code may not be equal?

> **Direct Answer:**
> - Equal objects must have identical hash codes because hash-based collections use the hash code as an address routing mechanism to determine the exact bucket location; differing hash codes cause lookups to check the wrong bucket.
> - Objects with the same hash code are not necessarily equal due to the Pigeonhole Principle: mapping an infinite universe of objects into a finite 32-bit integer space ($2^{32}$) mathematically forces hash collisions.
>
> **Key Nuance / Trade-off:**
> - Hash collisions are resolved internally via chaining (linked lists or red-black trees), where `equals()` is invoked as a secondary filter to disambiguate collided entries.
>
> **Reference Concept:** `Mathematical Asymmetry & Hash Collisions (The Pigeonhole Principle)`

---

### What is the difference between shallow copy and deep copy of an object?

> **Direct Answer:**
> - A shallow copy duplicates the primary object structure but copies field references directly, meaning both original and copy share references to the same internal mutable objects.
> - A deep copy recursively duplicates the top-level object and all referenced child objects, creating a completely independent memory graph.
>
> **Key Nuance / Trade-off:**
> - In a shallow copy, mutating an internal child object reflects across both copies; deep copy eliminates shared reference side-effects at the cost of higher CPU and memory allocation.
>
> **Reference Concept:** `Shallow Copy vs Deep Copy`

---

### How can you implement deep copy in Java?

> **Direct Answer:**
> - **Copy Constructor:** Manually instantiate new copies of all nested mutable objects (the cleanest and recommended manual approach).
> - **JSON/Serialization Libraries:** Serialize the graph to JSON via Jackson or Gson and deserialize back to a fresh instance (modern enterprise standard for DTOs).
> - **Native Serialization:** Serialize through `ByteArrayOutputStream`/`ObjectOutputStream` using `Serializable` (costly due to reflection).
> - **Overriding `clone()`:** Implement `Cloneable` and manually clone nested fields (discouraged legacy approach).
>
> **Key Nuance / Trade-off:**
> - Joshua Bloch famously noted `Cloneable` is broken because it bypasses constructors and cannot assign `final` fields; copy constructors or JSON serialization are standard in enterprise design.
>
> **Reference Concept:** `Deep Copy Implementation Strategies & Trade-offs`

---

### What is a Java Bean and what conventions must a class follow to be considered a Java Bean?

> **Direct Answer:**
> - A Java Bean is a reusable, standardized software component designed for introspection, manipulation, and serialization by frameworks.
> - It must provide a `public` no-argument constructor to support reflective instantiation.
> - Its properties must be `private`, accessed exclusively via standard getters and setters (`getProp()`, `setProp()`).
>
> **Key Nuance / Trade-off:**
> - While implementing `java.io.Serializable` is technically optional in specification, it is strongly recommended to enable distributed session persistence and remote invocation.
>
> **Reference Concept:** `Java Bean Specifications & Conventions`

---

### If a Person class is mutable, how does it affect using it as a key in a HashMap, specifically comparing when equals() and hashCode() are overridden versus when they are not?

> **Direct Answer:**
> - **When NOT overridden:** The map uses default identity hashing based on memory address; mutating field values still allows lookup using the original reference, but logical lookup with a newly created equivalent instance fails with `null`.
> - **When OVERRIDDEN:** Mutating key fields after insertion causes the object's logical `hashCode()` to change; subsequent lookups calculate a new bucket index, find an empty bucket, and return `null`.
>
> **Key Nuance / Trade-off:**
> - Mutating overridden keys causes permanent memory leaks: the object remains stuck in its original insertion bucket forever, unretrievable by standard methods and shielded from garbage collection.
>
> **Reference Concept:** `Mutable Objects as Map Keys`

---

### How would you implement a singleton class in Java?

> **Direct Answer:**
> - **Double-Checked Locking:** Combines a private constructor, a `private static volatile` instance variable, and a two-tier `instance == null` check inside a synchronized block for thread-safe lazy loading.
> - **Bill Pugh Pattern:** Uses a private static inner holder class whose initialization is deferred until `getInstance()` is called, achieving zero-overhead lock-free lazy loading via the JVM classloader.
> - **Enum Singleton:** Declares a single-element enum, providing JVM-enforced single instantiation.
>
> **Key Nuance / Trade-off:**
> - Enum Singletons are the most robust against advanced reflection attacks and serialization duplicates, but do not support class inheritance.
>
> **Reference Concept:** `Double-Checked Locking (Thread-Safe Lazy Initialization)`

---

### How would you implement a thread-safe singleton in Java that also optimizes for performance?

> **Direct Answer:**
> - Use the **Double-Checked Locking** idiom with a `private static volatile` instance reference, avoiding synchronization once the instance is initialized.
> - Alternatively, implement the **Bill Pugh Initialization-on-demand holder idiom**, leveraging classloader synchronization to eliminate runtime locking overhead completely.
>
> **Key Nuance / Trade-off:**
> - Synchronizing the entire `getInstance()` method modifier creates a severe concurrency bottleneck; using block-level synchronization or holder idioms avoids thread contention on read paths.
>
> **Reference Concept:** `Double-Checked Locking (Thread-Safe Lazy Initialization)`

---

### What are Immutable classes? How do you make a class immutable?

> **Direct Answer:**
> - An immutable class is one whose internal state cannot be altered after instantiation.
> - Declare the class as `final` to prevent inheritance and overriding.
> - Make all fields `private` and `final` to ensure single assignment and memory barrier visibility.
> - Provide no mutator methods (setters) and initialize all state via the constructor using defensive copying.
>
> **Key Nuance / Trade-off:**
> - Immutable instances are inherently thread-safe and make ideal `HashMap` keys, but creating modified variations requires allocating new heap instances.
>
> **Reference Concept:** `Rules for Creating Immutable Classes`

---

### Why are immutable objects inherently thread-safe, and how does this eliminate the need for synchronization?

> **Direct Answer:**
> - Concurrency bugs (race conditions) mathematically require both shared state and mutable state; immutability completely eliminates mutable state.
> - Because internal state can never be modified post-construction, multiple concurrent threads can read the same instance simultaneously without locks or memory barriers.
>
> **Key Nuance / Trade-off:**
> - Under the Java Memory Model, marking fields as `final` guarantees Initialization Safety: a hardware-level memory barrier ensures all threads observe fully initialized state without stale data or partial construction.
>
> **Reference Concept:** `Thread Safety & Concurrency Guarantees of Immutability`

---

### What if an immutable class has references to mutable Objects? How do you handle this?

> **Direct Answer:**
> - Marking a field as `final` only protects the reference pointer itself, not the internal state of the referenced object.
> - Apply **Defensive Copying** at both entry and exit boundaries: clone incoming mutable objects inside the constructor and return unmodifiable wrappers (or fresh clones) from getters.
>
> **Key Nuance / Trade-off:**
> - Failing to defensively copy outgoing references allows callers to execute mutating operations (e.g., `student.getCourses().add("Hacking 101")`), silently compromising immutability.
>
> **Reference Concept:** `Defensive Copying for Mutable Object References`

---

### What are record classes, how do you construct one, and how do you protect it if it contains references to mutable objects?

> **Direct Answer:**
> - Introduced in Java 16, a `record` is a compact, immutable data carrier class that eliminates boilerplate by automatically generating `final` fields, canonical constructors, accessors, `equals()`, `hashCode()`, and `toString()`.
> - Accessors use component names directly without the JavaBean `get` prefix (e.g., `user.username()`).
> - To protect mutable components (such as lists), override the **Compact Constructor** and apply defensive copying (e.g., `roles = List.copyOf(roles);`).
>
> **Key Nuance / Trade-off:**
> - The compiler generates `final` fields for record components, but does not perform automated defensive copying; mutable collections passed to records remain vulnerable without compact constructor safeguards.
>
> **Reference Concept:** `Java Records & Compact Constructors (Java 16+)`

---
