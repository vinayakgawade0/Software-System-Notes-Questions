# Object-Oriented Programming (OOP) in Java - Interview Q&A

### What are the key OOP principles in Java? Explain encapsulation, abstraction, inheritance and polymorphism with examples.

> **Direct Answer:**
> - **Encapsulation:** Bundles state and behavior into a single class while protecting internal variables via access modifiers (`private`) and exposing controlled access through methods.
> - **Abstraction:** Hides complex implementation details and exposes only essential contracts through abstract classes and interfaces.
> - **Inheritance:** Enables code reuse and creates hierarchical relationships where a child class acquires fields and behaviors from a parent class using `extends`.
> - **Polymorphism:** Allows treating objects of different concrete subclasses through a unified parent or interface reference, resolved at compile-time (overloading) or runtime (overriding).
>
> **Key Nuance / Trade-off:**
> - While inheritance provides immediate code reuse, it introduces tight coupling (the fragile base class problem); modern object-oriented system design favors composition (`HAS-A`) over inheritance (`IS-A`).
>
> **Reference Concept:** `The Four Pillars of OOP (Encapsulation, Abstraction, Inheritance, Polymorphism)`

---

### Can Polymorphism exist without Inheritance?

> **Direct Answer:**
> - **Yes, Absolutely:** Polymorphism does not require class inheritance (`extends`).
> - **Compile-Time Polymorphism:** Method overloading achieves polymorphism within a single class without any inheritance relationship.
> - **Interface-Based Polymorphism:** Implementing interfaces (`implements`) provides polymorphic behavioral contracts without inheriting implementation state.
> - **Generics & Functional Lambdas:** Parametric polymorphism via Generics and behavior parameterization via functional interfaces and lambdas execute polymorphic logic without class hierarchies.
>
> **Key Nuance / Trade-off:**
> - In dynamic languages this is known as "Duck Typing"; in Java, interface-based and functional polymorphism deliver the flexibility of dynamic polymorphism while retaining strict compile-time type safety.
>
> **Reference Concept:** `Polymorphism Without Inheritance (Overloading, Interfaces, Duck Typing)`

---

### Explain the abstract keyword in Java - from when it exists, what it is, and why it's there?

> **Direct Answer:**
> - **Core Language History:** The `abstract` keyword has been an integral keyword in Java since Java 1.0 (1996).
> - **Language Definition:** Applied to classes to prevent direct instantiation via `new`, and to methods to declare signatures without implementation bodies.
> - **Architectural Purpose:** Represents incomplete domain abstractions that should only exist as specialized subclasses (e.g., you instantiate `SavingsAccount`, not generic `Account`), enforcing mandatory contracts while permitting shared state.
>
> **Key Nuance / Trade-off:**
> - An abstract class cannot be declared `final`, and abstract methods cannot be declared `private` or `static` because they require subclass extension and dynamic overriding.
>
> **Reference Concept:** `The abstract Keyword: Evolution, Purpose & Abstract Classes`

---

### What is the difference between abstract class and interface (especially after Java 8 & 9)?

> **Direct Answer:**
> - **State vs. Stateless:** Abstract classes can declare mutable instance variables with any access modifier; interfaces can only declare `public static final` constants.
> - **Constructors & Multi-Inheritance:** Abstract classes can have constructors and participate in single inheritance only (`extends`); interfaces cannot have constructors and support multiple inheritance (`implements`).
> - **Modern Interface Methods:** Since Java 8, interfaces support `default` and `static` methods; Java 9 introduced `private` helper methods for internal default method logic.
> - **Selection Guide:** Use abstract classes for core domain identity and shared mutable state across closely related classes; use interfaces to establish behavioral contracts across unrelated classes.
>
> **Key Nuance / Trade-off:**
> - Even with default and private methods, interfaces remain fundamentally stateless contracts because they cannot hold instance fields or initialize object state through constructors.
>
> **Reference Concept:** `Abstract Class vs. Interface Matrix (Java 8 & Java 9 Modernization)`

---

### Why does Java not support multiple inheritance with classes and how do interfaces solve this problem?

> **Direct Answer:**
> - **The Diamond Problem:** If Class `D` extends both Class `B` and Class `C`, and both inherit and override `foo()` from Class `A`, Class `D` cannot determine which parent implementation to execute.
> - **State Duplication:** Inheriting multiple concrete classes causes duplicate parent instance fields in heap memory, creating complex memory layouts.
> - **The Interface Solution:** Historically, interfaces held no method bodies or instance state, eliminating conflict. With Java 8 default methods, conflicting default signatures force the implementing class to explicitly override the method or specify `InterfaceName.super.method()`.
>
> **Key Nuance / Trade-off:**
> - Java supports multiple inheritance of *type* and *behavior* (via interfaces), but strictly forbids multiple inheritance of *state* (via classes).
>
> **Reference Concept:** `Multiple Inheritance & The Diamond Problem Resolution via Interfaces`

---

### What are default methods in Java interfaces (introduced in Java 8), why were they introduced, and how are method conflicts resolved when multiple interfaces define the same default method?

> **Direct Answer:**
> - **Why Introduced:** Allowed the non-breaking evolution of existing JDK core interfaces (e.g., adding `stream()` and `forEach()` to `java.util.Collection`) without breaking thousands of legacy implementation classes.
> - **Rule 1 (Classes Win):** Concrete methods in a superclass or abstract class always override interface default methods.
> - **Rule 2 (Sub-Interfaces Win):** The most specific default method in an interface hierarchy wins (child interface default overrides parent interface default).
> - **Rule 3 (Explicit Override):** If two unrelated interfaces define identical default methods, the implementing class must explicitly override the method and resolve conflict using `InterfaceName.super.method()`.
>
> **Key Nuance / Trade-off:**
> - Default methods cannot override `java.lang.Object` methods (like `equals()`, `hashCode()`, or `toString()`); the class-level implementation always takes precedence.
>
> **Reference Concept:** `Default Methods in Interfaces & Conflict Resolution Rules`

---

### Explain the IS-A and HAS-A relationships strictly from an Object-Oriented Programming (OOP) perspective, with examples, and how do you decide between them?

> **Direct Answer:**
> - **`IS-A` (Inheritance):** Expressed using `extends` or `implements` (`Dog IS-A Animal`). Represents specialization and subtyping where the child is an authentic instance of the parent.
> - **`HAS-A` (Composition):** Expressed by holding references to other objects as private instance fields (`Car HAS-A Engine`).
> - **Decision Guideline:** Favor `HAS-A` (composition) over `IS-A` (inheritance). Use `IS-A` only when the relationship satisfies the Liskov Substitution Principle and the subclass will never need to decouple from the parent class implementation.
>
> **Key Nuance / Trade-off:**
> - Inheritance creates a compile-time compile lock where parent modifications risk breaking child invariants; composition provides loose coupling and permits changing components dynamically at runtime.
>
> **Reference Concept:** `IS-A vs. HAS-A Relationships (Inheritance vs. Composition)`

---

### Can we overload or override a static method in Java?

> **Direct Answer:**
> - **Overloading (Allowed):** Yes, static methods can be overloaded by defining methods with the same name but different parameter lists within the same class or child class.
> - **Overriding (Impossible - Method Hiding):** No, static methods cannot be overridden dynamically. If a child class defines a static method with the same signature, it causes **Method Hiding**.
> - **Compile-Time Binding:** Static method invocation is resolved at compile time based strictly on the declared reference type, not the underlying runtime instance.
>
> **Key Nuance / Trade-off:**
> - Calling a static method on an instance variable (e.g., `parentRef.staticMethod()`) executes the method bound to the *reference type* (`Parent`), completely ignoring the runtime object (`Child`).
>
> **Reference Concept:** `Static Method Overloading vs. Method Hiding`

---

### Why does the Java Compiler forbid method overloading by changing only the return type, and where exactly does the ambiguity occur?

> **Direct Answer:**
> - **The "Fire-and-Forget" Problem:** In Java, method calls can be executed as standalone statements where the return value is discarded (e.g., `calculate(10);`).
> - **Compiler Ambiguity:** If two methods existed differing only by return type (`int calculate(int x)` vs. `double calculate(int x)`), the compiler cannot determine which method to invoke when the return value is ignored.
> - **Signature Rule:** A method signature in Java consists exclusively of **Method Name + Parameter Types**; return type is strictly excluded from signature resolution.
>
> **Key Nuance / Trade-off:**
> - Even when assigning the result to a variable (`int x = calculate(10);`), the compiler determines method invocation from arguments *before* evaluating assignment compatibility.
>
> **Reference Concept:** `Return Type Ambiguity in Method Overloading ("Fire and Forget" Invocation)`

---

### If the return type is not part of the method signature, can we change the return type when overriding a method?

> **Direct Answer:**
> - **Covariant Return Types (Java 5+):** Yes, an overriding method in a subclass can change the return type provided the new return type is a **subtype** (narrower type) of the parent method's return type.
> - **Type Safety:** Maintains the Liskov Substitution Principle because any caller expecting the parent return type can safely accept the child subtype.
> - **Eliminates Casting:** Allows callers to invoke the method on the concrete child class and receive the specific type directly without manual downcasting.
>
> **Key Nuance / Trade-off:**
> - The return type can only be narrowed (covariant); broadening the return type (e.g., returning `Object` when parent returns `String`) or returning primitive alternatives causes a compile-time error.
>
> **Reference Concept:** `Covariant Return Types in Method Overriding`

---

### What is a marker interface in Java, why does it exist, and when would you still use it given that we now have annotations?

> **Direct Answer:**
> - **Type-Based Tagging:** A marker interface is an empty interface with zero methods or constants (`Serializable`, `Cloneable`) used to tag classes with a specific capability.
> - **Historical Purpose:** Used before Java 5 annotations to signal JVM internal subsystems or libraries via `instanceof` checks.
> - **Modern Use Case (Compile-Time Type Safety):** Use a marker interface when you need to enforce a compile-time constraint on method parameters (e.g., `void persist(AuditedEntity e)`), which annotations cannot enforce at compile time.
>
> **Key Nuance / Trade-off:**
> - Marker interfaces define genuine Java types that participate in polymorphism and can be extended by other interfaces, whereas annotations are purely metadata constructs.
>
> **Reference Concept:** `Marker Interfaces vs. Annotations Matrix & Custom Annotation Creation`

---

### Compare marker interfaces and annotations in Java. When would you choose one over the other, and how do you create a custom annotation?

> **Direct Answer:**
> - **Type vs. Metadata:** Marker interfaces define a type participating in polymorphism and compile-time API parameter restrictions; annotations attach metadata to classes, methods, or fields without altering type hierarchy.
> - **When to Choose:** Choose marker interfaces when you need compile-time parameter type safety; choose annotations when adding operational metadata consumed via reflection or framework aspects (`@Transactional`, `@Entity`).
> - **Custom Annotation Syntax:** Declare using `@interface` paired with `@Retention(RetentionPolicy.RUNTIME)` and `@Target(...)`.
>
> **Key Nuance / Trade-off:**
> - Marker interfaces cannot carry configurable parameters or target individual methods and fields, whereas annotations support parameterized attributes with default values.
>
> **Reference Concept:** `Marker Interfaces vs. Annotations Matrix & Custom Annotation Creation`

---

### What does the transient keyword do in Java, how does it interact with serialization, and what are practical use cases and caveats?

> **Direct Answer:**
> - **Serialization Exclusion:** Instructs the Java native serialization engine (`ObjectOutputStream`) to ignore the annotated field when converting an object to a byte stream.
> - **Deserialization State:** Upon deserialization, transient fields are reset to their default JVM values (`null` for objects, `0` for numbers, `false` for booleans).
> - **Practical Use Cases:** Protecting sensitive data (passwords, PINs), excluding environment-bound resources (socket connections, thread handles), and omitting calculated caches.
>
> **Key Nuance / Trade-off:**
> - The `transient` keyword strictly affects standard Java binary serialization (`java.io.Serializable`); modern JSON mappers (Jackson, Gson) ignore `transient` unless configured or supplemented with `@JsonIgnore`.
>
> **Reference Concept:** `The transient Keyword in Java Serialization & Serialization Caveats`

---

### What are sealed classes in Java, and why are they useful?

> **Direct Answer:**
> - **Restricted Class Hierarchies (Java 17):** Allows a class or interface to strictly declare and control which subclasses are permitted to extend or implement it using `sealed` and `permits`.
> - **Domain Modeling:** Enables modeling closed, algebraic data types (e.g., `PaymentMethod permits CreditCard, PayPal, Crypto`).
> - **Exhaustive Pattern Matching:** Allows the compiler to verify that all permitted subtypes are covered in modern `switch` expressions without requiring a defensive `default` branch.
>
> **Key Nuance / Trade-off:**
> - All permitted subclasses must be declared in the same module (or same package if in an unnamed module) as the sealed parent class.
>
> **Reference Concept:** `Sealed Classes & Interfaces (sealed, permits)`

---

### In a sealed class hierarchy, what happens if a permitted subclass is declared as non-sealed, and that subclass is extended further?

> **Direct Answer:**
> - **The "Open Gate":** Marking a permitted subclass as `non-sealed` explicitly breaks out of the sealed restrictions and re-opens that branch of the inheritance hierarchy to arbitrary extension.
> - **Unrestricted Subtyping:** Any class in any package can now freely extend the `non-sealed` subclass.
> - **Compiler Exhaustiveness Impact:** Because the hierarchy is no longer completely closed, compiler exhaustiveness checks in `switch` statements can no longer guarantee every possible subclass is known; a `default` case or a pattern covering the `non-sealed` type becomes mandatory.
>
> **Key Nuance / Trade-off:**
> - Every permitted subclass must declare strictly one modifier: `final` (closes branch), `sealed` (continues restricted hierarchy), or `non-sealed` (opens branch to unrestricted extension).
>
> **Reference Concept:** `Sealed Class Hierarchy Modifiers (final, sealed, non-sealed Open Gates)`

---

### Can a private method in parent be overridden in child? Explain with code

> **Direct Answer:**
> - **No, Impossible:** A private method in a parent class cannot be overridden by a child class.
> - **Visibility & Binding:** Private methods are completely invisible outside the declaring class and use static binding (`invokespecial`), bypassing the virtual method table entirely.
> - **Accidental Redeclaration:** If a child class defines a method with the identical signature, it is a completely new, unrelated method in the child class; adding `@Override` will result in a compile error.
>
> **Key Nuance / Trade-off:**
> - Calling the method on a parent reference pointing to a child object will always invoke the parent's private implementation, demonstrating that no dynamic polymorphism exists for private methods.
>
> **Reference Concept:** `Private Methods and Dynamic Binding Restrictions`

---

### Can a child class overload a method inherited from its parent class? Explain with a coding example.

> **Direct Answer:**
> - **Yes, Allowed:** A child class can overload an inherited method by defining a method with the exact same name but a different parameter list.
> - **Overload vs. Override:** The child class now has access to both methods: the inherited version from the parent and the overloaded version declared in the child.
> - **Compile-Time Reference Resolution:** Overload resolution is determined at compile time based strictly on the declared reference type.
>
> **Key Nuance / Trade-off:**
> - If an overloaded child object is upcast to the parent reference type (`Parent p = new Child()`), the compiler will only allow calls to the parent's parameter signature; attempting to pass the child's overloaded parameter type causes a compile error.
>
> **Reference Concept:** `Overloading Inherited Methods in Subclasses`
