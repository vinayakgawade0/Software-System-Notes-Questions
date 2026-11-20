# Questions: Functional Programming

### What are lambda expressions in Java and why were they introduced?

> **Direct Answer:**
> - Introduced in Java 8, lambda expressions provide a concise, expression-based syntax for representing anonymous functions (functions without declared names).
> - They enable functional programming paradigms in Java, drastically eliminate anonymous inner class boilerplate, and serve as the core syntax for the Streams API.
> - A lambda expression acts as an inline implementation of a Single Abstract Method (SAM) defined on a functional interface.
>
> **Key Nuance / Trade-off:**
> - Unlike anonymous inner classes which compile into separate `.class` bytecode files, lambdas are dynamically translated at runtime using the `invokedynamic` opcode (`LambdaMetafactory`), reducing memory overhead.
>
> **Reference Concept:** `Lambda Expressions & The SAM Principle`

---

### What is a functional interface and how is it used with lambda expressions?

> **Direct Answer:**
> - A functional interface is an interface that declares **exactly one abstract method** (SAM - Single Abstract Method).
> - It serves as the explicit target type for lambda expressions and method references, allowing behaviors to be passed as method arguments.
> - It may declare any number of `default` or `static` methods without violating the single abstract method contract, and is typically marked with `@FunctionalInterface`.
>
> **Key Nuance / Trade-off:**
> - In stream pipelines, methods like `.filter(Predicate)` and `.map(Function)` accept functional interfaces; the compiler transparently binds the inline lambda as the implementation of the interface's abstract method.
>
> **Reference Concept:** `Lambda Expressions & The SAM Principle`

---

### What are the built-in functional interfaces in Java and where are they used?

> **Direct Answer:**
> - Located in `java.util.function`, common built-in interfaces include:
>   - `Predicate<T>` (`boolean test(T t)`): Used for conditional testing and filtering.
>   - `Function<T, R>` (`R apply(T t)`): Used for type transformations and value mapping.
>   - `Consumer<T>` (`void accept(T t)`): Used for executing actions on elements without returning a value.
>   - `Supplier<T>` (`T get()`): Used for lazy generation or provisioning of instances.
>   - `BiFunction<T, U, R>`, `UnaryOperator<T>`, and `BinaryOperator<T>`: Used for multi-argument mapping and stream reductions.
>
> **Key Nuance / Trade-off:**
> - Built-in functional interfaces standardize signatures across the Java ecosystem, avoiding the need to define custom single-method interfaces for standard functional operations.
>
> **Reference Concept:** `Built-in Functional Interfaces (java.util.function)`

---

### Difference between lambda expressions and anonymous inner classes in Java.

> **Direct Answer:**
> - Lambdas provide concise, expression-based syntax exclusively for functional interfaces (SAM), whereas anonymous inner classes use verbose class declarations and can extend classes or implement multi-method interfaces.
> - Inside a lambda, the `this` keyword lexically refers to the enclosing outer class instance; inside an anonymous inner class, `this` refers to the anonymous class instance itself.
> - Lambdas compile into `invokedynamic` bytecode calls avoiding separate disk `.class` files, whereas anonymous inner classes generate distinct compiled class files (e.g., `Outer$1.class`).
>
> **Key Nuance / Trade-off:**
> - Lambdas cannot declare their own instance variables or maintain state, whereas anonymous inner classes can declare custom fields, constructors, and helper methods.
>
> **Reference Concept:** `Lambda Expressions vs Anonymous Inner Classes`

---

### Explain method references (::) in Java and the different types of method references.

> **Direct Answer:**
> - A method reference is a shorthand notation for a lambda expression that simply delegates arguments directly to an existing named method.
> - There are four distinct types:
>   1. **Static Method Reference:** `ClassName::staticMethod` (e.g., `Integer::parseInt`).
>   2. **Bound Instance Method Reference:** `instance::instanceMethod` (e.g., `System.out::println`).
>   3. **Unbound Instance Method Reference:** `ClassName::instanceMethod` (e.g., `String::toLowerCase`).
>   4. **Constructor Reference:** `ClassName::new` (e.g., `ArrayList::new`).
>
> **Key Nuance / Trade-off:**
> - Method references improve readability and eliminate redundant parameter declarations when a lambda does nothing other than pass its inputs into an existing method.
>
> **Reference Concept:** `Method Reference Classifications (::)`

---

### What is the difference between static methods and default methods in interfaces?

> **Direct Answer:**
> - `default` methods are declared with the `default` keyword, are inherited by implementing classes, can be overridden, and are invoked on an instance reference.
> - `static` methods are declared with the `static` keyword, are not inherited by implementing classes, cannot be overridden, and must be invoked directly on the interface name.
> - `default` methods allow backward-compatible interface evolution by supplying fallback behavior, while `static` methods provide cohesive utility functions for the interface.
>
> **Key Nuance / Trade-off:**
> - Prior to Java 8, adding methods to an interface broke all external implementations; `default` methods solved this by allowing library designers to add methods (like `Collection.stream()`) without breaking existing classes.
>
> **Reference Concept:** `Default Methods vs Static Methods in Interfaces`

---

### Explain how the this keyword behaves inside and outside lambda expressions.

> **Direct Answer:**
> - Outside a lambda (in standard methods), `this` refers to the current instance of the enclosing class.
> - Inside a lambda, `this` **still refers to the enclosing outer class instance** due to lexical scoping; lambdas do not introduce a new scope or independent identity.
> - In contrast, inside an anonymous inner class, `this` refers exclusively to the anonymous inner class instance itself.
>
> **Key Nuance / Trade-off:**
> - Because lambdas lack their own `this` pointer, they cannot refer to themselves recursively or inspect their own generated runtime class without external references.
>
> **Reference Concept:** `Lexical Scoping & The this Keyword Behavior`

---

### What happens if a functional interface contains more than one abstract method?

> **Direct Answer:**
> - An interface with more than one abstract method violates the Single Abstract Method (SAM) contract and will fail compilation if annotated with `@FunctionalInterface`.
> - It cannot be used as a target type for lambda expressions or method references.
>
> **Key Nuance / Trade-off:**
> - **The `java.lang.Object` Exception:** Abstract methods that match public methods of `java.lang.Object` (such as `equals(Object)`, `hashCode()`, or `toString()`) do **not** increment the abstract method count, because all implementations inherit them from `Object`.
>
> **Reference Concept:** `Abstract Method Count Rules & The java.lang.Object Exception`

---
