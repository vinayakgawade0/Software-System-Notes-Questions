# Object-Oriented Programming (OOP) in Java

## Core OOP Pillars & Polymorphic Architectures

### The Four Pillars of OOP (Encapsulation, Abstraction, Inheritance, Polymorphism)
- **1. Encapsulation (Information Hiding):**
  - Bundles state (data) and behavior (methods) into a single unit (class) while restricting direct access to internal components using access modifiers (`private`, `protected`, `public`).
  - Access is controlled via getters, setters, or business methods enforcing class invariants.
  ```java
  public class BankAccount {
      private double balance; // Hidden state

      public void deposit(double amount) {
          if (amount > 0) this.balance += amount; // Invariant enforced
      }
      public double getBalance() { return balance; }
  }
  ```
- **2. Abstraction (Hiding Complexity):**
  - Hides internal implementation mechanics and exposes only essential operations through abstract classes and interfaces.
  - Consumers depend on high-level contracts rather than low-level implementation details.
- **3. Inheritance (Code Reuse & Hierarchy):**
  - Allows child classes (`subclasses`) to inherit fields and behaviors from parent classes (`superclasses`) using `extends`.
  - Establishes taxonomic relationships (`Dog IS-A Animal`), but introduces tight coupling between classes.
- **4. Polymorphism (Many Forms):**
  - Enables treating objects of different concrete subclasses through a shared parent class or interface reference.
  - *Compile-Time (Static):* Method overloading (resolved at compile time).
  - *Runtime (Dynamic):* Method overriding (resolved at runtime via virtual method tables).

### Polymorphism Without Inheritance (Overloading, Interfaces, Duck Typing)
- **Common Misconception:** Polymorphism does **NOT** require class inheritance (`extends`).
- **Mechanisms for Polymorphism Without Inheritance:**
  1. **Compile-Time Polymorphism (Method Overloading):** The same class defines multiple methods sharing one name with distinct parameter lists. The compiler determines which method to call based strictly on arguments without any inheritance involved.
  2. **Interface-Based Polymorphism:** A class implements an interface (`implements`). Interfaces carry no inherited state or class implementation; they define purely behavioral contracts.
  3. **Generics & Parametric Polymorphism:** Generic classes/methods operate uniformly across different types without requiring those types to share a common class inheritance tree.
  4. **Functional Interfaces & Lambdas:** Passing lambda expressions (e.g., `Predicate<T>`, `Function<T, R>`) into higher-order methods produces dynamic polymorphic behavior without class hierarchies.

### The abstract Keyword: Evolution, Purpose & Abstract Classes
- **From When It Exists:** The `abstract` keyword has been a core keyword in Java since Java 1.0 (1996).
- **Abstract Class:** A class marked with `abstract` that cannot be instantiated directly via `new`. It serves as a partial template meant to be extended by concrete subclasses.
- **Abstract Method:** A method declared without a body (`abstract void execute();`) inside an abstract class or interface that concrete subclasses must implement.
- **Why It Exists:**
  - Prevents instantiating incomplete, generic domain concepts (e.g., you should instantiate a `SavingsAccount`, but instantiating a raw, generic `Account` makes no business sense).
  - Enforces a mandatory contract on child classes while allowing shared instance state and reusable base methods to reside in the parent class.

### Abstract Class vs. Interface Matrix (Java 8 & Java 9 Modernization)
| Feature | Abstract Class | Interface (Java 8 & 9 Onwards) |
| :--- | :--- | :--- |
| **Instantiation** | Cannot be instantiated directly | Cannot be instantiated directly |
| **Methods Supported** | Abstract, concrete, static, final | Abstract, default (Java 8), static (Java 8), private (Java 9) |
| **State / Variables** | Instance variables with any access modifier | Only `public static final` constants |
| **Constructors** | Can have constructors (to initialize shared state) | Cannot have constructors |
| **Inheritance Model** | Single inheritance only (`extends OneClass`) | Multiple inheritance (`implements InterfaceA, InterfaceB`) |
| **Access Modifiers** | Methods can be `public`, `protected`, `package-private` | Abstract/default methods implicitly `public`; `private` methods allowed |
| **Architectural Role** | Core identity & shared mutable state across family | Behavioral contract across completely unrelated classes |

### Multiple Inheritance & The Diamond Problem Resolution via Interfaces
- **Why Java Disallows Multiple Class Inheritance:**
  - *The Diamond Problem:* If Class `D` extends both Class `B` and Class `C`, and both `B` and `C` override method `foo()` from Class `A`, which `foo()` does `D` inherit? The compiler faces ambiguity, and managing duplicate inherited object state creates memory complexity.
- **How Interfaces Solve the Problem:**
  - Interfaces originally held no state and no method bodies, eliminating ambiguity.
  - With Java 8 default methods, multiple interface default method conflicts are strictly resolved by compiler enforcement: the implementing class **must explicitly override** the conflicting method or specify `InterfaceName.super.method()`.

### Default Methods in Interfaces & Conflict Resolution Rules
- **Why Introduced in Java 8:**
  - To allow non-breaking evolution of existing JDK core interfaces. For example, adding `stream()` or `forEach()` directly to `java.util.Collection` would have broken every third-party collection library in existence without default implementations.
- **Three Conflict Resolution Rules:**
  1. **Rule 1 (Classes Win Over Interfaces):** A concrete method declared in a superclass or abstract class always takes precedence over any interface default method.
  2. **Rule 2 (Sub-Interfaces Win):** If Interface `B` extends Interface `A`, and both provide a default method, Interface `B`'s more specific method wins.
  3. **Rule 3 (Explicit Disambiguation):** If a class implements two completely independent interfaces (`InterfaceA` and `InterfaceB`) defining identical default signatures, the code will not compile until the implementing class explicitly overrides the method:
     ```java
     public class MyClass implements InterfaceA, InterfaceB {
         @Override
         public void doWork() {
             InterfaceA.super.doWork(); // Explicitly choose InterfaceA
         }
     }
     ```

### IS-A vs. HAS-A Relationships (Inheritance vs. Composition)
- **`IS-A` (Class Inheritance):**
  - Implemented using `extends` or `implements`.
  - Represents specialized subtyping (`Dog IS-A Animal`, `SavingsAccount IS-A Account`).
  - *The Architectural Risk:* Creates tight coupling. Fragile base class problems occur when changes in parent classes inadvertently break child subclass invariants.
- **`HAS-A` (Composition / Aggregation):**
  - Implemented by holding an instance reference as a private field (`Car HAS-A Engine`).
  - Promotes loose coupling, allows dynamic runtime swapping of components, and facilitates mocking during unit testing.
- **The Design Rule:** *"Favor object composition over class inheritance."* Use `IS-A` only when the subclass is truly a complete specialized substitute for the parent class (Liskov Substitution Principle).

---

## Method Mechanics: Overloading, Overriding & Static Methods

### Static Method Overloading vs. Method Hiding
- **Overloading Static Methods (Allowed):**
  - Static methods can be overloaded within the same class or child class by changing the parameter list:
    ```java
    public static void log(String msg) { ... }
    public static void log(String msg, int level) { ... }
    ```
- **Overriding Static Methods (Impossible - Method Hiding):**
  - Static methods cannot be overridden dynamically.
  - If a child class defines a static method with the exact same signature as a static method in its parent class, this is **Method Hiding**, not overriding.
  - *Compile-Time Binding:* The method executed is determined strictly by the declared reference type at compile time, not the runtime object:
    ```java
    Parent p = new Child();
    p.staticMethod(); // Invokes Parent.staticMethod()!
    ```

### Return Type Ambiguity in Method Overloading ("Fire and Forget" Invocation)
- **Why Return Type Overloading is Forbidden:**
  - In Java, method calls can be executed as standalone statements without assigning or evaluating the return value ("fire and forget"):
    ```java
    calculate(10); // Return value discarded
    ```
  - If two methods exist: `int calculate(int x)` and `double calculate(int x)`, the compiler cannot deduce which method the developer intended to execute when the return value is unassigned.
  - Therefore, method signatures in Java consist strictly of **Method Name + Parameter Types**. Return type is not part of the signature.

### Covariant Return Types in Method Overriding
- **The Rule:** In Java 5+, when overriding a method in a subclass, the overridden method's return type can be narrowed down to a subclass of the return type declared in the parent method.
- **Why It Matters:** Eliminates explicit casting when callers invoke methods on concrete child classes:
  ```java
  class Animal {
      public Animal reproduce() { return new Animal(); }
  }

  class Dog extends Animal {
      @Override
      public Dog reproduce() { return new Dog(); } // Covariant: Dog is a subtype of Animal
  }

  // Caller usage without casting:
  Dog dog = new Dog();
  Dog puppy = dog.reproduce(); // No (Dog) casting required
  ```

### Private Methods and Dynamic Binding Restrictions
- **Can a private method in a parent class be overridden?**
  - **No.** Private methods are strictly invisible to child subclasses.
  - If a child class creates a method with the identical name and signature as a private method in its parent class, it is a completely new, unrelated method.
  - Private methods use `invokespecial` bytecode instructions (static binding resolved at compile time), bypassing the virtual method table (`invokevirtual`).

### Overloading Inherited Methods in Subclasses
- A child class can overload a method inherited from its parent class by defining a method with the same name but different parameters.
- **Reference Binding:** Overloading is resolved at compile time based on the declared reference type:
  ```java
  class Parent {
      public void show(String s) { System.out.println("Parent String: " + s); }
  }

  class Child extends Parent {
      public void show(int i) { System.out.println("Child int: " + i); } // Overload
  }

  Parent p = new Child();
  // p.show(10); // COMPILE ERROR! Parent reference has no show(int)
  ((Child) p).show(10); // Works: Child int: 10
  ```

---

## Metadata, Type Tagging & Serialization

### Marker Interfaces vs. Annotations Matrix & Custom Annotation Creation
- **Marker Interface:** An empty interface without methods or fields (`Cloneable`, `Serializable`, `Remote`) used to categorize or tag classes.
- **Annotation (`@interface`):** A metadata construct introduced in Java 5 that attaches declarative metadata to classes, methods, fields, and parameters.
- **Comparison Matrix:**
  | Aspect | Marker Interface | Annotation |
  | :--- | :--- | :--- |
  | **Core Nature** | Defines a type; participates in Java type hierarchy | Adds declarative metadata; not a Java type |
  | **Compile-Time Checking** | Can restrict method parameter types (`void save(AuditedEntity e)`) | Cannot restrict parameter types at compile time alone |
  | **Target Flexibility** | Restricted strictly to classes and interfaces | Flexible targeting via `@Target` (methods, fields, parameters) |
  | **Evolution** | Adding methods breaks all implementing classes | Can add new elements with default values without breaking code |
  | **Primary Modern Use** | Type-safe compile-time API constraints | Framework metadata (`@Transactional`, `@Entity`, `@RestController`) |
- **Custom Annotation Creation:**
  ```java
  @Retention(RetentionPolicy.RUNTIME) // Available at runtime for reflection
  @Target(ElementType.METHOD)          // Applicable strictly to methods
  public @interface Audited {
      String action() default "DEFAULT_AUDIT";
      int priority() default 1;
  }
  ```

### The transient Keyword in Java Serialization & Serialization Caveats
- **Purpose:** Instructs the Java serialization engine (`ObjectOutputStream.writeObject()`) to skip persisting the designated field.
- **Post-Deserialization State:** During deserialization (`readObject()`), transient fields are initialized to their JVM default primitive or object values (`null`, `0`, `false`).
- **Practical Use Cases:**
  1. **Security:** Excluding sensitive credentials, passwords, or encryption keys from serialized state.
  2. **Non-Serializable Handles:** Excluding platform-bound resources like open file descriptors, network sockets, or database connections.
  3. **Derived / Cache Fields:** Excluding computational fields that can be easily recomputed on demand from base data.
- **Caveat:** `transient` only affects standard Java binary serialization (`Serializable`). External JSON serialization libraries (like Jackson or Gson) ignore `transient` unless explicitly configured.

---

## Modern Java Sealed Class Hierarchies

### Sealed Classes & Interfaces (sealed, permits)
- **Core Concept (Java 17):** Allows a class or interface to strictly control and restrict which subclasses are permitted to extend or implement it using `sealed` and `permits`.
- **Syntax:**
  ```java
  public sealed interface PaymentMethod 
      permits CreditCard, PayPal, CryptoPayment {}

  public final class CreditCard implements PaymentMethod {}
  public final class PayPal implements PaymentMethod {}
  public non-sealed class CryptoPayment implements PaymentMethod {}
  ```
- **Architectural Value:**
  - Establishes closed, domain-exhaustive algebraic data hierarchies.
  - Enables the Java compiler to verify exhaustive pattern matching in modern `switch` statements without requiring a default fallback branch.

### Sealed Class Hierarchy Modifiers (final, sealed, non-sealed Open Gates)
- Every permitted subclass of a sealed class must explicitly declare one of three modifier keywords:
  1. **`final`:** The subclass cannot be extended any further; the hierarchy ends here.
  2. **`sealed`:** The subclass continues the restricted hierarchy, explicitly declaring its own `permits` clause.
  3. **`non-sealed` (The "Open Gate"):** Breaks out of the sealed restrictions and re-opens the inheritance tree, allowing arbitrary subclasses to extend it freely.
- **Exhaustive Switch Implication:** If a permitted subclass is marked `non-sealed`, the compiler knows unknown subclasses could exist; an exhaustive `switch` on the sealed interface will require a `default` case or a catch-all pattern for the `non-sealed` branch.
