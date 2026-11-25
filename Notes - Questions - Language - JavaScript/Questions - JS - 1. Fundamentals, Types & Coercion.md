# Questions - JS - 1. Fundamentals, Types & Coercion

### What is the difference between `var`, `let`, and `const` in JavaScript?

> **Direct Answer:**
> - `var` is function-scoped, allows redeclaration and reassignment, and is hoisted with an initial value of `undefined`.
> - `let` and `const` are block-scoped (`{}`), disallow redeclaration within the same scope, and are hoisted into the Temporal Dead Zone (TDZ) until their declaration executes. `let` allows reassignment, whereas `const` prohibits reassigning the variable binding.
>
> **Key Nuance / Trade-off:**
> - Default to `const` for all bindings that won't be reassigned, use `let` only when reassignment is needed, and avoid `var` entirely in modern JavaScript.
>
> **Reference Concept:** `Variable Declarations: var, let, and const`

---

### What is Hoisting in JavaScript?

> **Direct Answer:**
> - Hoisting is JavaScript's compilation-phase behavior where variable and function declarations are registered at the top of their enclosing scope before execution begins.
> - Only declarations are hoisted—never initializations. `var` is initialized to `undefined`, whereas `let` and `const` remain uninitialized in the Temporal Dead Zone.
>
> **Key Nuance / Trade-off:**
> - Function declarations are hoisted completely (both signature and body), allowing invocation before their line of definition, unlike `var` or function expressions.
>
> **Reference Concept:** `Hoisting & Temporal Dead Zone (TDZ)`

---

### What is the Temporal Dead Zone (TDZ)?

> **Direct Answer:**
> - The Temporal Dead Zone (TDZ) is the period between entering a block scope—where a `let` or `const` variable is hoisted—and the execution of its declaration statement.
> - Attempting to read or write the variable inside the TDZ throws a `ReferenceError`.
>
> **Key Nuance / Trade-off:**
> - The TDZ exists intentionally to catch use-before-declaration bugs early rather than silently returning `undefined` like `var`.
>
> **Reference Concept:** `Hoisting & Temporal Dead Zone (TDZ)`

---

### Why is `var` avoided in modern JavaScript?

> **Direct Answer:**
> - `var` lacks block scope (leaking out of `if`/`for` blocks), permits accidental redeclaration, returns `undefined` when accessed before declaration due to hoisting initialization, and shares a single function-scoped binding across asynchronous loop iterations causing closure bugs.
>
> **Key Nuance / Trade-off:**
> - Replacing `var` with `let` in `for` loops creates a fresh block-scoped binding per iteration, solving asynchronous callback closure bugs cleanly.
>
> **Reference Concept:** `Variable Declarations: var, let, and const`

---

### Can `const` objects be modified in JavaScript?

> **Direct Answer:**
> - Yes. `const` only makes the variable reference (binding) immutable—it cannot be reassigned to a new object—but the properties inside the referenced object or array can still be added, modified, or deleted.
>
> **Key Nuance / Trade-off:**
> - To make an object's properties immutable at runtime, use `Object.freeze(obj)` (which performs a shallow freeze).
>
> **Reference Concept:** `Variable Declarations: var, let, and const`

---

### How many data types exist in JavaScript?

> **Direct Answer:**
> - JavaScript has **8 data types**: 7 primitive types (`string`, `number`, `boolean`, `null`, `undefined`, `symbol`, `bigint`) and 1 reference type (`object`, which encompasses plain objects, arrays, and functions).
>
> **Key Nuance / Trade-off:**
> - Primitive values are immutable and copied by value, whereas objects are mutable and copied by memory reference.
>
> **Reference Concept:** `JavaScript Data Types (Primitive vs Reference)`

---

### List the primitive data types in JavaScript.

> **Direct Answer:**
> - The 7 primitive data types in JavaScript are: `string`, `number`, `boolean`, `null`, `undefined`, `symbol` (ES6 unique identifier), and `bigint` (large integers beyond `Number.MAX_SAFE_INTEGER`).
>
> **Key Nuance / Trade-off:**
> - Primitives have no methods of their own; when you call `"hello".toUpperCase()`, JavaScript temporarily wraps the primitive in a corresponding object wrapper (`String`) and discards it immediately after.
>
> **Reference Concept:** `JavaScript Data Types (Primitive vs Reference)`

---

### What is the difference between primitive and reference types?

> **Direct Answer:**
> - Primitive types store their actual immutable value directly and are copied and compared by value.
> - Reference types (`object`, `array`, `function`) store a memory reference pointing to a mutable heap structure; copying a reference variable points both variables to the exact same object in memory.
>
> **Key Nuance / Trade-off:**
> - Mutating a copied object reference (`obj2.name = "Rahul"`) mutates the original (`obj1.name`), which is why shallow/deep cloning (`{...obj}`, `structuredClone()`) is necessary for immutable state updates.
>
> **Reference Concept:** `JavaScript Data Types (Primitive vs Reference)`

---

### Why does `typeof null` return `"object"`?

> **Direct Answer:**
> - `typeof null` returning `"object"` is a legacy bug from JavaScript's first 1995 implementation: object values had a binary type tag of `0`, and `null` was represented as the all-zero NULL pointer (`0x00`), causing `typeof` to misidentify it as `"object"`.
>
> **Key Nuance / Trade-off:**
> - Despite `typeof null === "object"`, `null` is a primitive type. Always check explicitly using `val === null` rather than relying on `typeof`.
>
> **Reference Concept:** `The typeof null Historical Bug & Dynamic Typing`

---

### Is JavaScript statically typed?

> **Direct Answer:**
> - No. JavaScript is a **dynamically typed** language—variable types are determined at runtime, type annotations are not required, and a variable can be reassigned values of different types during execution.
>
> **Key Nuance / Trade-off:**
> - Dynamic typing speeds up prototyping but allows silent runtime type mismatches; large enterprise projects adopt TypeScript to add static compile-time type checking on top of JavaScript.
>
> **Reference Concept:** `The typeof null Historical Bug & Dynamic Typing`

---

### What is type coercion in JavaScript?

> **Direct Answer:**
> - Type coercion is the automatic (implicit) or manual (explicit) conversion of a value from one data type to another during operations or comparisons (e.g., `"5" + 2` $\to$ `"52"`, `"5" - 2` $\to$ `3`, `true + 1` $\to$ `2`).
>
> **Key Nuance / Trade-off:**
> - The `+` operator favors string concatenation whenever either operand is a string, whereas `-`, `*`, `/`, and `>` always trigger numeric (`ToNumber`) coercion.
>
> **Reference Concept:** `Type Coercion Rules (Implicit vs Explicit)`

---

### What is the difference between implicit and explicit type coercion?

> **Direct Answer:**
> - **Implicit coercion** is automatic type conversion performed silently by the JavaScript engine during mixed-type expressions (e.g., `"10" * 2` evaluates to `20`).
> - **Explicit coercion** is deliberate manual conversion written by the developer using built-in conversion functions like `Number("100")`, `String(123)`, `Boolean(val)`, or `parseInt()`.
>
> **Key Nuance / Trade-off:**
> - Relying on implicit coercion hides intent and causes subtle bugs; prefer explicit conversion functions so type transformations are self-documenting.
>
> **Reference Concept:** `Type Coercion Rules (Implicit vs Explicit)`

---

### What is the difference between `==` and `===` in JavaScript?

> **Direct Answer:**
> - `==` (Loose Equality) converts operands to a common type via implicit coercion before comparing values (e.g., `5 == "5"` is `true`, `null == undefined` is `true`).
> - `===` (Strict Equality) disables type coercion and checks **both value and type**, immediately returning `false` if types differ.
>
> **Key Nuance / Trade-off:**
> - Even under `===`, `NaN === NaN` is `false` (use `Number.isNaN()`) and `+0 === -0` is `true`; use `Object.is(a, b)` when exact value identity for `NaN` and signed zero is needed.
>
> **Reference Concept:** `Loose Equality (==) vs Strict Equality (===)`

---

### Why is `===` preferred over `==`?

> **Direct Answer:**
> - `===` is preferred because it avoids hidden implicit type coercion, prevents unexpected truthy/numeric conversion bugs (such as `0 == false` or `"" == 0` evaluating to `true`), and guarantees predictable, consistent behavior.
>
> **Key Nuance / Trade-off:**
> - The only common exception where developers occasionally leverage `==` intentionally is `x == null`, which concisely checks for both `null` and `undefined` simultaneously.
>
> **Reference Concept:** `Loose Equality (==) vs Strict Equality (===)`

---

### What are falsy values in JavaScript?

> **Direct Answer:**
> - Falsy values are values that evaluate to `false` when coerced to a boolean in conditional contexts (`if`, `while`, logical operators).
> - JavaScript has six core falsy values: `false`, `0` (plus `-0` and `0n`), `""` (empty string), `null`, `undefined`, and `NaN`.
>
> **Key Nuance / Trade-off:**
> - Be careful when checking numeric variables like `count = 0` with `if (count)`—since `0` is falsy, valid zero values will fail truthiness checks; use `count !== undefined && count !== null` or nullish coalescing (`??`) instead.
>
> **Reference Concept:** `Type Coercion Rules (Implicit vs Explicit)`

---

### What are truthy values in JavaScript?

> **Direct Answer:**
> - Truthy values are all values in JavaScript that evaluate to `true` when converted to a boolean—which includes **every value not on the falsy list** (non-empty strings, non-zero numbers, objects `{}`, arrays `[]`, functions, `Symbol()`, and non-zero BigInts).
>
> **Key Nuance / Trade-off:**
> - Common interview edge cases that look falsy but are actually **truthy**: `"0"`, `"false"`, `" "` (a space string), empty array `[]`, and empty object `{}`.
>
> **Reference Concept:** `Type Coercion Rules (Implicit vs Explicit)`

---
