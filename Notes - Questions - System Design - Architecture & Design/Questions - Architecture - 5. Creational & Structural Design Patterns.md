# Chapter 5: Creational & Structural Design Patterns

## Creational Design Patterns

### Singleton Pattern and Implementations

#### Explain Singleton design pattern. How many ways can you break the Singleton pattern?

> **Direct Answer:** The Singleton design pattern guarantees that a class has exactly one instance in memory within a classloader and provides a global access point to it. A standard Singleton class can be broken in four primary ways: via **Java Reflection** (`Constructor.setAccessible(true)` can invoke private constructors); via **Serialization / Deserialization** (reading a serialized singleton stream creates a new heap object); via **Cloning** (calling `clone()` on a `Cloneable` subclass duplicates the object); and through **Multiple ClassLoaders** independently loading and instantiating copies in the same JVM.

> **Key Nuance / Trade-off:** The most elegant and robust defense against reflection, serialization, and multithreading vulnerabilities in Java is the **Enum Singleton** (recommended by Joshua Bloch in *Effective Java*), as the JVM internally guarantees enum uniqueness, serializability, and reflection immunity.

> **Reference Concept:** [Singleton Pattern and Implementations](./Notes%20-%20Architecture%20-%205.%20Creational%20%26%20Structural%20Design%20Patterns.md#singleton-pattern-and-implementations)

---

#### How would you implement a Singleton class in Java and what are the different approaches?
> **Company Attribution:** [JP Morgan]  
> **Concept Tag:** [Concept: Singleton Pattern, Design Patterns]

> **Direct Answer:** Java supports six primary Singleton implementations: **Eager Initialization** (simple, thread-safe, but instantiates at class-load time); **Lazy Initialization** (memory-efficient but strictly non-thread-safe); **Synchronized Method** (thread-safe, but incurs a 10x–100x performance penalty on every access); **Double-Checked Locking** (thread-safe and efficient, requiring a `volatile` instance variable to establish a memory barrier preventing instruction reordering); the **Bill Pugh Holder Idiom** (initialization-on-demand static inner class, providing thread-safe lazy loading with zero synchronization overhead); and the **Enum Singleton** (simplest, safest, reflection-proof).

> **Key Nuance / Trade-off:** In Double-Checked Locking, omitting the `volatile` keyword creates a critical concurrency bug: instruction reordering during bytecode execution can assign the heap memory address before constructor initialization finishes, causing other threads to consume a partially initialized, corrupted object.

> **Reference Concept:** [Singleton Pattern and Implementations](./Notes%20-%20Architecture%20-%205.%20Creational%20%26%20Structural%20Design%20Patterns.md#singleton-pattern-and-implementations)

---

### Factory Pattern and Decoupled Instantiation

#### Explain Factory Design Pattern with example.

> **Direct Answer:** The Factory Method pattern defines an interface for object creation, but delegates the actual instantiation logic to factory classes or specialized static methods based on input parameters. For example, a `NotificationFactory` exposes `createNotification(channel)` that returns a common `Notification` interface implemented by `SMSNotification`, `EmailNotification`, or `PushNotification`. This adheres strictly to the Open/Closed and Dependency Inversion principles by ensuring client calling code depends solely on the high-level interface abstraction, completely decoupled from concrete constructor configurations and vendor SDKs.

> **Key Nuance / Trade-off:** The Factory pattern is most valuable when object initialization involves complex configuration (credentials, network timeouts, cryptographic keys) or varies dynamically based on runtime environment profiles. For simple POJOs with trivial constructors, introducing a factory creates unnecessary indirection.

> **Reference Concept:** [Factory Pattern and Decoupled Instantiation](./Notes%20-%20Architecture%20-%205.%20Creational%20%26%20Structural%20Design%20Patterns.md#factory-pattern-and-decoupled-instantiation)

---

## Structural and Behavioral Design Patterns

### Adapter Pattern and Anti-Corruption Layer

#### Adapter Pattern - How do you integrate incompatible third-party libraries or legacy systems into your modern application without polluting your core business logic, and how does the Adapter Pattern solve this?

> **Direct Answer:** When integrating third-party vendor APIs or legacy systems (e.g., SOAP XML endpoints), their incompatible data types, error structures, and proprietary exceptions risk contaminating clean application domain logic. The Adapter pattern solves this by acting as a translation bridge: the application interacts exclusively with a pristine Target Interface (e.g., `PaymentProcessor`), while an Adapter class implements that interface, wraps the incompatible Adaptee (e.g., vendor `StripeService`), maps domain data types into vendor parameters, and translates vendor exceptions into domain errors.

> **Key Nuance / Trade-off:** In Domain-Driven Design (DDD), the Adapter pattern is elevated to an **Anti-Corruption Layer (ACL)**. The ACL serves as a defensive quarantine boundary, ensuring third-party SDK updates or legacy schema modifications never bleed into or force changes upon core business entities.

> **Reference Concept:** [Adapter Pattern and Anti-Corruption Layer](./Notes%20-%20Architecture%20-%205.%20Creational%20%26%20Structural%20Design%20Patterns.md#adapter-pattern-and-anti-corruption-layer)

---

### Observer Pattern and Event-Driven Decoupling

#### Observer Pattern - How do you decouple core business logic from downstream side effects (like sending emails or logging) when a system state changes, and how does the Observer Pattern facilitate this?

> **Direct Answer:** The Observer pattern decouples an event publisher (the Subject) from interested subscribers (Observers) by shifting from imperative commands to event-driven push notifications. In modern Spring Boot systems, when `UserService` registers a user, it saves the entity and simply calls `eventPublisher.publishEvent(new UserRegisteredEvent(userId))`. Independent downstream listeners (`EmailService`, `AnalyticsService`) annotate their handler methods with `@EventListener` and `@Async` to react to the event in separate background threads, ensuring core user registration succeeds even if downstream email servers fail.

> **Key Nuance / Trade-off:** In distributed microservices, the Observer pattern is scaled to the infrastructure level using message brokers (Apache Kafka or RabbitMQ) where Kafka Topics act as distributed Subjects and consumer groups act as distributed Observers, and in reactive programming where Project Reactor's `Flux` and `Mono` streams embody the Observer reactive model.

> **Reference Concept:** [Observer Pattern and Event-Driven Decoupling](./Notes%20-%20Architecture%20-%205.%20Creational%20%26%20Structural%20Design%20Patterns.md#observer-pattern-and-event-driven-decoupling)
