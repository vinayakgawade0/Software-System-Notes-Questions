# Module 3: Objects & States

## 1. Object Contracts & Identity

### equals() vs hashCode() Contract
In Java, the `Object` class defines two fundamental methods that establish identity and logical equality: `equals(Object obj)` and `hashCode()`. When developing domain entities or data structures, adhering to their shared contract is mandatory.

#### Core Definitions & Differences
| Aspect | `equals(Object o)` | `hashCode()` |
| :--- | :--- | :--- |
| **Definition** | Determines logical equality between two objects | Returns an integer hash code representation of the object state |
| **Return Type** | `boolean` | `int` (32-bit signed integer) |
| **Default (`Object`)** | Reference equality (`this == o`, memory address check) | Native identity hash code derived from internal object memory address |
| **Override Purpose** | Define business/logical equality based on state attributes | Maintain uniform distribution and bucket consistency in hash-based collections |
| **Relationship** | If `a.equals(b) == true` $\rightarrow$ `a.hashCode() == b.hashCode()` | If `a.hashCode() == b.hashCode()` $\rightarrow$ `a.equals(b)` may be `true` or `false` |
| **Formal Contract** | Reflexive, Symmetric, Transitive, Consistent, Non-null | Consistent across invocations; identical for logically equal instances |

#### The Formal Contract Rules
1. **Reflexive:** For any non-null reference `x`, `x.equals(x)` must return `true`.
2. **Symmetric:** For any non-null references `x` and `y`, `x.equals(y)` must return `true` if and only if `y.equals(x)` returns `true`.
3. **Transitive:** For any non-null references `x`, `y`, and `z`, if `x.equals(y)` returns `true` and `y.equals(z)` returns `true`, then `x.equals(z)` must return `true`.
4. **Consistent:** For any non-null references `x` and `y`, multiple invocations of `x.equals(y)` must consistently return `true` or `false`, provided no information used in `equals` comparisons is modified.
5. **Non-nullity:** For any non-null reference `x`, `x.equals(null)` must return `false`.
6. **Hash Code Consistency:** Whenever `equals()` evaluates to `true` between two objects, their `hashCode()` values must be identical. If `equals()` evaluates to `false`, the hash codes do not need to be distinct (though producing distinct values improves hash table performance).

#### Implementation Example
```java
public class Employee {
    private final int id;
    private final String name;

    public Employee(int id, String name) {
        this.id = id;
        this.name = name;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee)) return false;
        Employee employee = (Employee) o;
        return this.id == employee.id; // Logical equality based on business key
    }

    @Override
    public int hashCode() {
        return Integer.hashCode(id); // Hash strictly aligned with equals comparison key
    }
}
```

```java
// Usage in HashSet:
Employee e1 = new Employee(1, "John");
Employee e2 = new Employee(1, "John");

Set<Employee> set = new HashSet<>();
set.add(e1);
set.add(e2);

System.out.println(set.size()); // 1 (contract upheld, duplicate eliminated)
```

---

### == Operator vs equals() Method
Java provides two ways to compare elements, operating at fundamentally different layers of the Java Virtual Machine.

| Aspect | `==` Operator | `equals()` Method |
| :--- | :--- | :--- |
| **Type of Comparison** | Reference equality for objects; literal bitwise value equality for primitives | Logical/state equality for objects |
| **Applicability** | Primitive types (`int`, `boolean`, `double`) and object references | Reference types (objects) only |
| **Default Behavior** | Checks if two references point to the exact same heap memory address | Inherited from `Object` as `this == obj` unless overridden |
| **Polymorphism** | Cannot be overridden (language operator) | Can be overridden by any class to define custom semantic equality |
| **Primitive Example** | `int a = 5, b = 5; (a == b) -> true` | Not applicable to primitive types |
| **Object Example** | `new String("Hi") == new String("Hi") -> false` | `new String("Hi").equals(new String("Hi")) -> true` |
| **Primary Use Case** | Checking reference identity or primitive equivalence | Checking semantic equality of domain entities or value objects |

---

### Contract Violations: Overriding equals() Without hashCode()
When a class overrides `equals()` to define logical equivalence (e.g., matching IDs) but fails to override `hashCode()`, it inherits the default `Object.hashCode()` implementation. This produces catastrophic bugs in hash-based collections (`HashMap`, `HashSet`, `ConcurrentHashMap`, `Hashtable`).

#### The Architectural Failure Mechanism
A `HashMap` stores key-value pairs across an internal array of buckets. The retrieval process requires two sequential stages:
1. **Routing to the Bucket:** The map computes `key.hashCode()`, applies bitwise spreading, and evaluates the array index `(n - 1) & hash`.
2. **Locating the Entry:** Once inside the bucket, the map traverses the bucket entries (linked list or red-black tree) and executes `equals()` to find the matching key.

```
Key Insertion:
Key A (ID: 1) -> Object.hashCode() -> Hash 0x1A4F -> Bucket Index 5 [Stored]

Key Retrieval:
Key B (ID: 1) -> Object.hashCode() -> Hash 0x7B9C -> Bucket Index 12 [Empty] -> Returns null!
```

#### Real-World Production Bugs
1. **Lost Data (Silent Null Returns):** Even though `Key A.equals(Key B)` evaluates to `true`, `map.get(Key B)` routes to Bucket 12 instead of Bucket 5. The lookup yields `null`, rendering the data unretrievable.
2. **Duplicate Key Accumulation & Memory Leaks:** If `map.put(Key B, "NewData")` is called, the map checks Bucket 12, finds it empty, and inserts a duplicate logical key. Both `Key A` and `Key B` now inhabit the map simultaneously. Over time, unreachable duplicate entries accumulate, continuously consuming heap space and ultimately triggering an `OutOfMemoryError` (OOM).

---

### Mathematical Asymmetry & Hash Collisions (The Pigeonhole Principle)

#### Why Equal Objects MUST Have the Same Hash Code
A hash code functions as an address routing mechanism. If a `HashMap` is conceptualized as a physical filing cabinet, `hashCode()` is the drawer label. If an object is filed into Drawer #5 using its hash code, any logically identical query object must calculate that exact same drawer label (#5) to locate it. If equal objects yielded different hash codes, the lookup algorithm would search the wrong drawer and immediately conclude the object is absent.

#### Why Same Hash Code Does NOT Guarantee Equality (The Pigeonhole Principle)
The relationship is inherently asymmetric due to basic mathematics:
- In Java, `hashCode()` returns a 32-bit signed integer (`int`), capping the universe of distinct hash codes at $2^{32} \approx 4,294,967,296$ (~4.29 billion).
- The universe of possible object states (e.g., all distinct character sequences of `java.lang.String`) is mathematically infinite.
- By the **Pigeonhole Principle**, mapping an infinite number of items into a finite set of $2^{32}$ pigeonholes guarantees that completely distinct, non-equal objects will inevitably produce the exact same integer hash code.

#### Java Collision Resolution Strategy
When two unequal objects produce identical hash codes (a **Hash Collision**), `HashMap` routes both entries to the same bucket.
- **Java 7:** Entries were stored as a singly linked list in the bucket. Retrieval took $O(n)$ time within that bucket.
- **Java 8+:** Entries are organized as a singly linked list until the bucket reaches `TREEIFY_THRESHOLD = 8` entries and total table capacity $\ge 64$. At this point, the bucket transforms into a balanced Red-Black Tree, improving collision lookup from $O(n)$ to $O(\log n)$.
- During retrieval from a collided bucket, Java iterates through the tree or linked list nodes and calls `key.equals(searchKey)` to disambiguate the entries and extract the exact target value.

---

### Mutable Objects as Map Keys
Using a mutable object as a key in a `HashMap` or `HashSet` is a critical anti-pattern that breaks collection invariants.

#### Case 1: equals() and hashCode() Are NOT Overridden (Inherited from Object)
- **Mechanism:** Default `hashCode()` is derived from the object's memory address (identity hash), and `equals()` checks reference identity (`==`).
- **Mutation Effect:** Modifying field values (e.g., `person.setName("John")`) does not alter the underlying heap memory address.
- **Lookup Behavior:**
  - `map.get(person)` using the original reference succeeds because its identity hash and reference remain unchanged.
  - Logical lookup fails: `map.get(new Person("John"))` returns `null` because the new instance resides at a different memory address with a different default hash code.

#### Case 2: equals() and hashCode() ARE Overridden (Based on Mutable State)
- **Mechanism:** When `map.put(person, data)` is called with `Person("Alice")`, the map computes the hash code from `"Alice"` and stores the node in Bucket $K$.
- **Mutation Effect:** Calling `person.setName("John")` modifies the fields participating in the hash computation while the object remains physically seated inside Bucket $K$.
- **Lookup Behavior & Memory Leak:**
  - Calling `map.get(person)` triggers a recalculation of `hashCode()` using `"John"`.
  - The map routes to Bucket $M$ ($M \neq K$). Bucket $M$ is empty (or contains unrelated data), so `map.get()` returns `null`.
  - **The Severe Memory Leak:** The original entry remains orphaned in Bucket $K$. It can never be retrieved, updated, or removed via standard map operations. Because the map maintains a strong reference to the object in Bucket $K$, the Garbage Collector cannot reclaim it, causing a permanent memory leak.

---

## 2. Object Cloning & Copying Strategies

### Shallow Copy vs Deep Copy

| Aspect | Shallow Copy | Deep Copy |
| :--- | :--- | :--- |
| **Definition** | Duplicates the top-level object only; copies primitive values and field references directly | Recursively duplicates the top-level object and all nested referenced objects |
| **Reference Sharing** | The original and cloned object share references to the same mutable internal objects | The original and cloned object point to completely independent heap memory graphs |
| **Mutation Side-Effects** | Modifying a nested mutable object via the copy mutates the original object | Modifying nested objects in the copy has zero impact on the original object |
| **Performance** | High performance: negligible memory overhead, simple pointer copying | Slower performance: allocates multiple new heap instances, traverses deep object graphs |
| **Default Mechanism** | `Object.clone()` performs a shallow field-by-field copy | Requires manual copy constructors, custom clone overrides, or serialization |

---

### Deep Copy Implementation Strategies & Trade-offs

#### 1. The Copy Constructor (Recommended Manual Approach)
A copy constructor accepts an instance of its own class and manually constructs new instances of all nested mutable components.

```java
public class Order {
    private final long id;
    private final Customer customer;

    public Order(Order source) {
        this.id = source.id;
        // Deep copy of nested mutable reference:
        this.customer = new Customer(source.customer);
    }
}
```
- **Architectural Verdict:** The safest, cleanest, and most transparent manual mechanism. Unlike `clone()`, it does not bypass constructor initialization and works seamlessly with `final` fields.
- **Trade-off:** High maintenance overhead. Adding a new field requires updating the copy constructor; failing to do so causes silent copy discrepancies.

#### 2. Overriding clone() with Cloneable (The Legacy Approach)
The class implements the `java.lang.Cloneable` marker interface and overrides `Object.clone()`. Because `super.clone()` only performs a shallow bitwise copy, every mutable field must be manually cloned.

```java
public class Order implements Cloneable {
    private Customer customer;

    @Override
    public Order clone() {
        try {
            Order copy = (Order) super.clone();
            copy.customer = (Customer) this.customer.clone(); // Manual deep clone
            return copy;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```
- **Architectural Verdict:** Actively discouraged in modern Java. Joshua Bloch noted that `Cloneable` is fundamentally broken: it creates objects without invoking constructors, cannot initialize or reassign `final` fields inside the clone method, and forces boilerplate handling of checked exceptions.

#### 3. Native Java Serialization (Byte Stream Round-Trip)
Serializes the object graph into a byte array and deserializes it back into a new object. All classes in the graph must implement `java.io.Serializable`.

```java
public static <T extends Serializable> T deepCopy(T object) throws Exception {
    ByteArrayOutputStream byteOut = new ByteArrayOutputStream();
    try (ObjectOutputStream out = new ObjectOutputStream(byteOut)) {
        out.writeObject(object);
    }
    try (ObjectInputStream in = new ObjectInputStream(new ByteArrayInputStream(byteOut.toByteArray()))) {
        return (T) in.readObject();
    }
}
```
- **Architectural Verdict:** Guarantees an automated, complete deep copy of arbitrary graphs without manual field mapping. However, it incurs massive CPU overhead, relies heavily on Java reflection, and is strictly avoided in low-latency or high-throughput enterprise systems.

#### 4. JSON Serialization Libraries (Modern Enterprise Standard)
Uses serialization frameworks (Jackson `ObjectMapper` or Google `Gson`) to serialize the object into a JSON string/byte buffer and parse it back into a fresh instance.

```java
ObjectMapper mapper = new ObjectMapper();
Order copy = mapper.readValue(mapper.writeValueAsString(originalOrder), Order.class);
```
- **Architectural Verdict:** The de-facto enterprise standard for microservices and DTO copying. It avoids `Serializable`/`Cloneable` constraints, runs much faster than native Java serialization, leverages standard constructors and setters, and safely ignores transient state.

---

## 3. Component Standards & JavaBeans

### Java Bean Specifications & Conventions
A Java Bean is a standardized, reusable software component that complies with strict convention rules to enable automated introspection and manipulation by frameworks (Spring, Hibernate, GUI builders).

#### Foundational Conventions
1. **Public Zero-Argument Constructor:** Must provide a `public` no-arg constructor to allow frameworks to instantiate the class via reflection (`Class.getDeclaredConstructor().newInstance()`).
2. **Encapsulated Properties:** All instance variables must be `private`.
3. **Standard Accessor & Mutator Methods:**
   - Getter naming: `getPropertyName()` (or `isPropertyName()` for primitive `boolean`).
   - Setter naming: `setPropertyName(PropertyType value)`.
4. **Serialization:** Should implement `java.io.Serializable` to allow state persistence, network transmission, and session replication across cluster nodes.

```java
public class Employee implements java.io.Serializable {
    private static final long serialVersionUID = 1L;

    private int id;
    private String name;

    public Employee() {
        // Required no-arg constructor
    }

    public int getId() { return id; }
    public void setId(int id) { this.id = id; }

    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
```

---

## 4. State Management: The Singleton Pattern

### Double-Checked Locking (Thread-Safe Lazy Initialization)
Used when a Singleton instance is resource-intensive and must be lazily instantiated on demand without introducing synchronization overhead on every read.

```java
public class Singleton {
    // volatile is mandatory to enforce memory barrier and visibility
    private static volatile Singleton instance;

    private Singleton() {
        // Enforce private constructor to block direct instantiation
    }

    public static Singleton getInstance() {
        if (instance == null) {                         // Check 1: Fast unsynchronized check
            synchronized (Singleton.class) {           // Synchronize class lock
                if (instance == null) {                 // Check 2: Thread-safe instantiation check
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

#### Why `volatile` is Strictly Mandatory
Without `volatile`, the CPU and JIT compiler can reorder instructions during object creation.
The assignment `instance = new Singleton()` involves three JVM steps:
1. Allocate heap memory for `Singleton`.
2. Execute the constructor to initialize fields.
3. Assign the memory address to the `instance` reference.

Without `volatile`, the compiler may reorder step 3 before step 2 (`1 -> 3 -> 2`). If Thread A executes steps 1 and 3 and is preempted, Thread B arrives at Check 1, observes `instance != null`, and immediately receives a reference to a **partially initialized object**, causing runtime data corruption. Marking `instance` as `volatile` prevents this reordering by establishing a happens-before relationship and introducing hardware memory barriers.

#### Synchronized Modifier vs Synchronized Statement
- **As a Method Modifier (`public static synchronized Singleton getInstance()`):** Obtains the class monitor for every single invocation. It destroys throughput because millions of concurrent read calls are serialized behind a single thread lock.
- **As a Statement Block (`synchronized (Singleton.class)`):** Granular locking. The lock is only acquired once when `instance == null` during initial creation. Subsequent reads execute purely via the fast unsynchronized check.

---

### Bill Pugh Singleton (Initialization-on-Demand Holder Idiom)
The most elegant lazy-loading Singleton pattern in Java. It achieves thread safety and lazy initialization without explicit `synchronized` or `volatile` keywords.

```java
public class Singleton {
    private Singleton() {}

    // Static nested class is not loaded when Singleton class loads
    private static class SingletonHelper {
        private static final Singleton INSTANCE = new Singleton();
    }

    public static Singleton getInstance() {
        return SingletonHelper.INSTANCE;
    }
}
```

#### Internal Mechanics
The JVM does not load static inner classes into memory when the outer class is loaded. `SingletonHelper` is only loaded and initialized when `getInstance()` is called for the first time. The JVM specification guarantees that class initialization phase (`<clinit>`) is thread-safe and executed under internal JVM locks. This delivers lock-free, zero-overhead lazy loading.

---

### Enum Singleton (The Joshua Bloch Bulletproof Approach)
Recommended by Joshua Bloch in *Effective Java* as the most robust singleton pattern in Java.

```java
public enum Singleton {
    INSTANCE;

    public void doSomething() {
        System.out.println("Executing thread-safe operation");
    }
}
```

#### Why It is Impervious to Attack
1. **Reflection Immunity:** Calling `Constructor.newInstance()` on an Enum class explicitly throws `IllegalArgumentException: Cannot reflectively create enum objects`.
2. **Serialization Safety:** The JVM manages enum serialization strictly through the enum constant name; it never creates a new instance during deserialization, preventing duplicate singleton instances.

---

## 5. Immutability & Modern State Encapsulation

### Rules for Creating Immutable Classes
An immutable class is one whose instances cannot be modified post-creation. Once initialized, its state remains strictly invariant across its entire lifecycle.

#### The Four Core Rules
1. **Declare the Class as `final`:** Prevents subclassing and method overriding that could inject mutable behavior.
2. **Make All Fields `private` and `final`:** Encapsulates fields from direct access and leverages the Java Memory Model's initialization guarantees.
3. **Provide No Mutator Methods (Setters):** Prevent any post-construction modification of internal fields.
4. **Initialize Exclusively via Constructor with Defensive Copying:** Enforce full initialization at construction time and prevent mutable internal references from leaking.

---

### Thread Safety & Concurrency Guarantees of Immutability

#### The Root Cause of Concurrency Bugs
Race conditions and data races strictly require two concurrent prerequisites:
1. **Shared State:** Multiple threads reference the identical memory object.
2. **Mutable State:** At least one thread executes write/mutation operations on that object.

Immutability physically eliminates the second condition (**Mutable State**). If state cannot be changed, race conditions become mathematically impossible.

#### Memory Mechanics
- **Forced Duplication Instead of In-Place Mutation:** If Thread A needs to alter a state (e.g., updating a name), it cannot modify the existing object. It must instantiate a brand new object.
- **Elimination of Synchronization:** Reading threads can access immutable instances across hundreds of concurrent threads without ever requiring locks, `synchronized` blocks, or atomic wrappers.
- **No Partially Constructed State:** All final fields are initialized atomically during constructor execution, ensuring other threads never observe an inconsistent intermediate state.

#### Java Memory Model (JMM) Initialization Safety
The JMM provides a special guarantee for `final` fields called **Initialization Safety**. When a constructor finishes executing, the JVM places an internal memory barrier (StoreStore barrier). This guarantees that any thread that acquires a reference to the immutable object will immediately see the correctly initialized values of all `final` fields without requiring explicit memory synchronization.

---

### Defensive Copying for Mutable Object References
When an immutable class contains a reference to a mutable object (e.g., `java.util.Date`, `java.util.List`), marking the reference field as `final` does not prevent caller code from mutating the referent object itself.

#### The Flawed Approach (Leaking Mutability)
```java
public final class Student {
    private final String name;
    private final List<String> courses; // Mutable object reference

    public Student(String name, List<String> courses) {
        this.name = name;
        this.courses = courses; // DANGER: Caller retains external reference to courses
    }

    public List<String> getCourses() {
        return courses; // DANGER: Caller can invoke getCourses().add("Hacking 101")
    }
}
```

#### The Senior Solution: Defensive Copying In and Out
To preserve immutability, a defensive copy must be made at both boundary entry (constructor) and boundary exit (getter):
1. **Defensive Copy In (Constructor):** Clone the incoming mutable collection or object so that mutations to the caller's reference have zero impact on internal state.
2. **Defensive Copy Out (Getter):** Return an unmodifiable view or an independent copy so callers cannot mutate internal state.

```java
public final class Student {
    private final String name;
    private final List<String> courses;

    public Student(String name, List<String> courses) {
        this.name = name;
        // 1. Defensive Copy In: Sever reference to incoming list
        this.courses = new ArrayList<>(courses);
    }

    public List<String> getCourses() {
        // 2. Defensive Copy Out: Return unmodifiable view (or List.copyOf in Java 10+)
        return Collections.unmodifiableList(this.courses);
    }
}
```

---

### Java Records & Compact Constructors (Java 16+)
Introduced as a standard feature in Java 16 (JEP 395), a `record` is a specialized class designed to act as a transparent carrier for immutable data.

#### Reducing Boilerplate
In traditional Java, an immutable DTO requires 50+ lines of code: `private final` fields, full constructor, getters, `equals()`, `hashCode()`, and `toString()`. A record condenses this into a single declaration:
```java
public record UserDto(Long id, String username, String email) {}
```

#### Senior Detail: Accessor Naming Convention
Records deliberately abandon the JavaBean `get` prefix. Accessor methods match the exact component name:
- **Incorrect:** `user.getUsername()`
- **Correct:** `user.username()`

#### The Record Mutable Reference Trap
Because record fields are backed by `final` references, the reference cannot be reassigned. However, if a record component is a mutable object (e.g., `List<String> roles`), the data inside that list can still be mutated by external callers:
```java
List<String> roles = new ArrayList<>();
roles.add("USER");

UserDto user = new UserDto(1L, "Alice", roles);
roles.add("ADMIN"); // Internal record state silently modified to [USER, ADMIN]!
```

#### Protection via Compact Constructor
To guarantee true immutability in records, developers override the **Compact Constructor** to execute defensive copying without parameter redeclaration:

```java
public record UserDto(Long id, String username, List<String> roles) {

    // Compact constructor: parameter list is omitted
    public UserDto {
        // Intercept and defensively copy the mutable list
        // List.copyOf() creates an unmodifiable copy and validates non-nullity
        roles = List.copyOf(roles);
    }
}
```
`List.copyOf()` severs external reference links and guarantees that `user.roles()` returns an unmodifiable list, preventing mutability leaks.
