# Module 5: Functional Programming

## 1. Functional Interfaces & Lambda Foundations

### Lambda Expressions & The SAM Principle
Introduced in Java 8 (2014), lambda expressions provide a concise, expression-based syntax for representing anonymous functions (functions without a declared name). They introduce functional programming idioms into Java, eliminating verbose boilerplate and serving as the primary syntactic engine for the Streams API.

#### Syntax & Structure
```java
// Parameterless lambda
() -> System.out.println("Executing")

// Single parameter (type inference allows omitting parentheses)
s -> s.length()

// Multiple parameters with explicit block body
(int a, int b) -> {
    int sum = a + b;
    return sum;
}
```

#### Comparison: Anonymous Inner Classes vs Lambdas
```java
// Prior to Java 8 (Anonymous Inner Class):
Runnable r1 = new Runnable() {
    @Override
    public void run() {
        System.out.println("Running task...");
    }
};

// With Java 8 Lambda Expression:
Runnable r2 = () -> System.out.println("Running task...");

// Comparator Example:
List<String> names = Arrays.asList("John", "Alice", "Bob");

// Prior to Java 8:
Collections.sort(names, new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.compareTo(b);
    }
});

// With Lambda Expression:
Collections.sort(names, (a, b) -> a.compareTo(b));
```

#### The Single Abstract Method (SAM) Requirement
A lambda expression cannot stand alone as an arbitrary function pointer; it requires a **target type**. In Java, that target type is a **Functional Interface**—an interface that defines **exactly one abstract method** (SAM).
- Functional interfaces can define multiple `default` or `static` methods, but only one abstract method.
- The `@FunctionalInterface` annotation informs the compiler to enforce the SAM rule at compile time.

```java
@FunctionalInterface
public interface Calculator {
    int operate(int a, int b); // Exactly one abstract method
}

// Target typing:
Calculator add = (a, b) -> a + b;
System.out.println(add.operate(5, 3)); // 8
```

#### Lambda Target Typing in Streams API
When executing `stream().filter(s -> s.startsWith("A"))`, the parameter accepted by `filter` is declared as:
```java
Stream<T> filter(Predicate<? super T> predicate);
```
Here, `Predicate<T>` is the functional interface target type:
```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}
```
The Java compiler binds `(s -> s.startsWith("A"))` as the implementation of `Predicate<String>.test()`.

---

### Built-in Functional Interfaces (java.util.function)
The `java.util.function` package provides standard general-purpose functional interfaces to represent common operational signatures across functional pipelines.

| Interface | Method Signature | Description | Enterprise Example Usage |
| :--- | :--- | :--- | :--- |
| **`Predicate<T>`** | `boolean test(T t)` | Evaluates a boolean condition on an input argument | Filtering collections (`stream().filter(e -> e.isActive())`) |
| **`Function<T, R>`** | `R apply(T t)` | Transforms an input argument of type `T` to a result of type `R` | Transforming entities (`stream().map(User::getId)`) |
| **`Consumer<T>`** | `void accept(T t)` | Performs an operation on an input argument with no return value | Terminal consumption (`stream().forEach(System.out::println)`) |
| **`Supplier<T>`** | `T get()` | Generates or supplies an instance of type `T` without taking input | Factory/lazy instantiation (`Optional.orElseGet(() -> fetchDefault())`) |
| **`BiFunction<T, U, R>`**| `R apply(T t, U u)` | Transforms two input arguments of types `T` and `U` into a result `R` | Combining two datasets or mapping pairs into DTOs |
| **`UnaryOperator<T>`** | `T apply(T t)` | Extends `Function<T, T>`; transforms an argument to the identical type | Unary calculations (e.g., `String::toUpperCase`, negating numbers) |
| **`BinaryOperator<T>`**| `T apply(T t1, T t2)`| Extends `BiFunction<T, T, T>`; combines two arguments of identical type | Stream reduction (`stream().reduce(0, Integer::sum)`) |

---

### Abstract Method Count Rules & The java.lang.Object Exception

#### The SAM Constraint
By definition, a functional interface must have exactly one abstract method. Declaring two abstract methods breaks the SAM contract:
```java
@FunctionalInterface
public interface MyInvalidFunc {
    void process();
    void validate(); // COMPILATION ERROR: Invalid @FunctionalInterface specification
}
```

#### The `java.lang.Object` Exception
An essential language specification detail: **Abstract methods that override public methods of `java.lang.Object` do not count toward the abstract method limit.**
All interface implementations inherently inherit public methods from `java.lang.Object` (such as `equals`, `hashCode`, `toString`). Therefore, explicitly redeclaring them as abstract methods within an interface does not alter the SAM count:

```java
@FunctionalInterface
public interface MyValidFunc {
    void execute(); // The single abstract method

    // These DO NOT increment the SAM count:
    boolean equals(Object obj);
    String toString();
    int hashCode();
}
```
This interface compiles cleanly as a valid `@FunctionalInterface` because `execute()` remains the sole non-`Object` abstract method.

---

## 2. Compilation Mechanics & Scoping

### Lambda Expressions vs Anonymous Inner Classes

| Aspect | Lambda Expression | Anonymous Inner Class |
| :--- | :--- | :--- |
| **Syntax & Conciseness** | Concise, inline expression-based syntax | Verbose class declaration boilerplate |
| **Target Applicability** | Restricted exclusively to Functional Interfaces (SAM) | Can implement interfaces or extend abstract/concrete classes with any method count |
| **`this` Reference Scope** | **Lexical scope:** `this` points to the enclosing class instance | **Class scope:** `this` points to the anonymous inner class instance itself |
| **Variable Capture** | Can capture `final` or **effectively final** local variables | Can capture `final` or effectively final local variables |
| **Bytecode & Compilation** | Compiled via Java 7+ `invokedynamic` opcode (`LambdaMetafactory`) | Compiled into a separate `.class` file (e.g., `Enclosing$1.class`) |
| **Runtime Performance** | High: Avoids class loading and per-instance heap allocation overhead | Lower: Incurs class-loading overhead and allocates new heap objects on instantiation |
| **State & Member Fields** | Cannot declare independent fields or instance state | Can define custom fields, methods, and initialization blocks |

---

### Lexical Scoping & The this Keyword Behavior
A fundamental design choice in Java lambdas is **Lexical Scoping**: lambdas do not introduce a new variable scope or separate `this` identity.

#### Scoping Mechanics
- **Anonymous Inner Class:** Introduces its own scope. Inside the class, `this` evaluates to the anonymous inner class instance. Accessing the enclosing outer class requires using `OuterClass.this`.
- **Lambda Expression:** Inherits the lexical scope of the enclosing method/class. Inside the lambda, `this` refers to the **enclosing outer class instance**.

```java
public class ScopeTest {
    private String name = "OuterClassInstance";

    public void demonstrateScope() {
        // 1. Anonymous Inner Class:
        Runnable r1 = new Runnable() {
            @Override
            public void run() {
                // Evaluates to: ScopeTest$1
                System.out.println("Anonymous this: " + this.getClass().getName());
            }
        };

        // 2. Lambda Expression:
        Runnable r2 = () -> {
            // Evaluates to: ScopeTest (Lexical scoping retains outer instance)
            System.out.println("Lambda this: " + this.getClass().getName());
        };

        r1.run();
        r2.run();
    }

    public static void main(String[] args) {
        new ScopeTest().demonstrateScope();
    }
}
```

#### Effectively Final Local Variables
Local variables declared outside a lambda and accessed within it must be `final` or **effectively final** (a variable whose value is never modified after initialization). This restriction prevents data races when lambdas execute concurrently across background threads.

---

## 3. Method References & Syntax Categories

### Method Reference Classifications (::)
Introduced in Java 8, method references provide shorthand syntax for lambda expressions that merely forward arguments directly to an existing named method.

```
Lambda Syntax:           (args) -> TargetClass.existingMethod(args)
Method Reference Syntax: TargetClass::existingMethod
```

#### The Four Categories of Method References
| Category | Syntax Pattern | Direct Lambda Equivalent | Real-World Example |
| :--- | :--- | :--- | :--- |
| **1. Static Method Reference** | `ClassName::staticMethod` | `(args) -> ClassName.staticMethod(args)` | `Integer::parseInt`<br>`(s) -> Integer.parseInt(s)` |
| **2. Bound Instance Method** | `instance::instanceMethod` | `(args) -> instance.instanceMethod(args)` | `System.out::println`<br>`(x) -> System.out.println(x)` |
| **3. Unbound Instance Method** | `ClassName::instanceMethod` | `(arg0, rest) -> arg0.instanceMethod(rest)` | `String::toLowerCase`<br>`(s) -> s.toLowerCase()` |
| **4. Constructor Reference** | `ClassName::new` | `(args) -> new ClassName(args)` | `ArrayList::new`<br>`() -> new ArrayList<>()` |

#### Code Implementation
```java
public class MethodReferenceExample {
    public static void printUpper(String s) {
        System.out.println(s.toUpperCase());
    }

    public static void main(String[] args) {
        List<String> names = Arrays.asList("alice", "bob", "charlie");

        // 1. Standard Lambda:
        names.forEach(s -> MethodReferenceExample.printUpper(s));

        // 2. Equivalent Static Method Reference:
        names.forEach(MethodReferenceExample::printUpper);

        // 3. Unbound Instance Method Reference:
        List<String> upperNames = names.stream()
            .map(String::toUpperCase) // (s) -> s.toUpperCase()
            .collect(Collectors.toList());

        // 4. Constructor Reference:
        Supplier<List<String>> listFactory = ArrayList::new; // () -> new ArrayList<>()
    }
}
```

---

## 4. Modern Interface Evolution

### Default Methods vs Static Methods in Interfaces
Prior to Java 8, interface methods were strictly `public abstract`. Java 8 introduced `default` and `static` methods to allow API evolution (such as adding `stream()` and `forEach()` to `java.util.Collection`) without breaking backward compatibility for third-party implementing classes.

| Feature | `default` Method | `static` Method |
| :--- | :--- | :--- |
| **Keyword** | Declared with `default` keyword | Declared with `static` keyword |
| **Inheritance** | Inherited by all implementing classes | Not inherited by implementing classes |
| **Overriding** | Can be overridden by implementing classes | Cannot be overridden by implementing classes |
| **Invocation** | Invoked on an object instance reference (`instance.method()`) | Invoked on the interface name directly (`InterfaceName.method()`) |
| **Design Purpose** | Provide optional or default behavior; enables backward-compatible API evolution | Provide utility and helper functions tightly coupled to the interface domain |
| **Access Modifiers** | Always `public` (private instance methods introduced in Java 9 for helper encapsulation) | Always `public` (private static methods introduced in Java 9) |

```java
public interface Vehicle {
    void drive(); // Abstract method (SAM candidate if alone)

    // Default method: Inherited and overridable by implementations
    default void honk() {
        System.out.println("Default Beep!");
    }

    // Static method: Utility function scoped strictly to the interface
    static boolean isValidSpeed(int speed) {
        return speed >= 0 && speed <= 300;
    }
}
```
Implementing classes inherit `honk()` automatically, but invoking `isValidSpeed` requires `Vehicle.isValidSpeed(100)`—implementations cannot access it via instance references.
