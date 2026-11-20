# Questions - Java - 12. Testing

### What is the difference between `doReturn().when()` and `when().thenReturn()` in Mockito, and when must you use one over the other?

> **Direct Answer:**
> - `when(mock.method()).thenReturn(value)` is the standard, compile-time type-safe Mockito stubbing syntax providing fluent, readable test definitions.
> - `doReturn(value).when(mock).method()` is an alternative syntax that bypasses real method execution entirely and handles `void` or polymorphic edge cases.
> - `when().thenReturn()` must be used by default for standard `@Mock` instances because it guarantees compile-time type checking between the method signature and return value.
>
> **Key Nuance / Trade-off:**
> - When stubbing a `@Spy` (partial mock), `when(spy.method()).thenReturn(...)` **actually executes the real method** once during test setup. If that real method accesses remote resources, uninitialized state, or throws exceptions, the test setup crashes. You **must** use `doReturn().when(spy)` to stub spies safely without triggering real method side effects. Additionally, `doThrow()` and `doNothing()` must be used for methods returning `void`.
>
> **Reference Concept:** `Mockito Stubbing: when-then vs doReturn-when`

---

### How do you unit test a method that has a `void` return type?

> **Direct Answer:**
> - Testing `void` methods is approached through two primary methodologies depending on whether the method mutates internal state or coordinates external dependencies:
>   1. **State Verification:** If the `void` method alters the object's internal state (e.g., adding an item to a collection, modifying an entity property), invoke the method and assert the modified state using public getter or inspector methods with standard assertions (`assertEquals`, `assertThat`).
>   2. **Behavior Verification:** If the `void` method delegates tasks to external collaborators (e.g., saving to a repository, dispatching a message, sending an email), mock the collaborators and use Mockito's `verify(collaborator, times(n)).method(...)` to confirm that the expected interaction occurred with the right parameters.
>
> **Key Nuance / Trade-off:**
> - For complex interaction arguments passed to collaborators, use `ArgumentCaptor<T>` to capture the exact runtime object and run deep assertions on its internal fields. When testing failure scenarios for `void` methods, assert exception propagation using JUnit 5's `assertThrows()`. Prefer state verification over behavior verification where possible to prevent over-coupling tests to internal implementation details.
>
> **Reference Concept:** `Testing Void Methods: State vs Behavior Verification`

---
