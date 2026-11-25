# Questions - JS - 2. Functions, Closures, Objects & Arrays

### What is a function declaration in JavaScript?

> **Direct Answer:**
> - A function declaration defines a named function using the `function` keyword (e.g., `function greet(name) { ... }`).
> - It is completely hoisted in the current scope, allowing it to be invoked even before its line of definition in the source file.
>
> **Key Nuance / Trade-off:**
> - Unlike function expressions assigned to `const`/`let`, function declarations pollute the enclosing scope if not modularized via ES Modules.
>
> **Reference Concept:** `Function Declarations vs Function Expressions`

---

### Are function declarations hoisted in JavaScript?

> **Direct Answer:**
> - Yes. Function declarations are **fully hoisted**—both the function identifier and its executable body are registered during compilation, making it valid to call the function prior to its definition line.
>
> **Key Nuance / Trade-off:**
> - This applies strictly to function declarations (`function foo() {}`), not function expressions (`const foo = function() {}` or arrow functions).
>
> **Reference Concept:** `Function Declarations vs Function Expressions`

---

### What is the difference between a function declaration and a function expression?

> **Direct Answer:**
> - A **function declaration** (`function name() {}`) is always named and is fully hoisted, allowing invocation before its definition.
> - A **function expression** (`const fn = function() {}`) can be anonymous, assigns a function as a value to a variable, and cannot be invoked prior to initialization (throws `ReferenceError` for `let`/`const` or `TypeError` for `var`).
>
> **Key Nuance / Trade-off:**
> - Function expressions are preferred when passing callbacks, creating conditional functions, or guaranteeing block-scoped bindings with `const`.
>
> **Reference Concept:** `Function Declarations vs Function Expressions`

---

### What are arrow functions in JavaScript?

> **Direct Answer:**
> - Arrow functions (`() => {}`), introduced in ES6, provide a concise syntax for writing function expressions and callbacks.
> - They do not bind their own `this`, lack an `arguments` object, and cannot be invoked as constructors with `new`.
>
> **Key Nuance / Trade-off:**
> - Single-expression arrow functions support implicit return (`(a, b) => a + b`) without curly braces or the `return` keyword.
>
> **Reference Concept:** `Arrow Functions vs Regular Functions`

---

### What is the difference between a regular function and an arrow function?

> **Direct Answer:**
> - **Regular functions** bind `this` dynamically based on the caller, have a local `arguments` object, and can be used as constructors with `new`.
> - **Arrow functions** inherit `this` lexically from their enclosing scope, lack an `arguments` object, and throw a `TypeError` if called with `new`.
>
> **Key Nuance / Trade-off:**
> - Arrow functions excel in callbacks (`map`, `setTimeout`, Promise chains) where preserving outer `this` is desired, whereas regular functions are suited for object methods and constructors.
>
> **Reference Concept:** `Arrow Functions vs Regular Functions`

---

### What is `arguments` object in JS functions?

> **Direct Answer:**
> - The `arguments` object is a local, array-like object automatically available inside all non-arrow functions containing every argument passed by the caller, regardless of declared parameters.
> - It has `.length` and numeric indices (`arguments[0]`), but lacks Array methods like `.map()` or `.sort()`.
>
> **Key Nuance / Trade-off:**
> - Modern ES6+ code strongly prefers **Rest Parameters (`...args`)** over `arguments` because `...args` produces a true Array, works inside arrow functions, and supports capturing remaining parameters cleanly.
>
> **Reference Concept:** `The arguments Object vs Rest Parameters (...args)`

---

### How does `this` behave in arrow functions?

> **Direct Answer:**
> - Arrow functions do not create their own `this` binding; instead, they capture `this` **lexically** from the surrounding enclosing scope at the time the function is defined.
>
> **Key Nuance / Trade-off:**
> - Even `.call()`, `.apply()`, or `.bind()` cannot override the lexical `this` of an arrow function.
>
> **Reference Concept:** `Arrow Functions vs Regular Functions`

---

### When should arrow functions not be used?

> **Direct Answer:**
> - Arrow functions should not be used for **object methods** (where `this` must refer to the object), **constructors** (cannot use `new`), **DOM event handlers** needing `this` to reference the clicked element, or functions relying on the `arguments` object.
>
> **Key Nuance / Trade-off:**
> - Using an arrow function as an object literal method binds `this` to the global/module scope instead of the object itself, causing `this.property` to evaluate to `undefined`.
>
> **Reference Concept:** `Arrow Functions vs Regular Functions`

---

### What is a callback function in JavaScript?

> **Direct Answer:**
> - A callback function is a function passed as an argument into another function so that it can be invoked later once a task finishes or an event triggers.
>
> **Key Nuance / Trade-off:**
> - Callbacks can be synchronous (such as the mapper passed to `Array.prototype.map`) or asynchronous (such as `setTimeout` or DOM event listeners).
>
> **Reference Concept:** `Callbacks & Callback Hell`

---

### Why are callback functions used in JavaScript?

> **Direct Answer:**
> - Callbacks enable non-blocking execution and inversion of control, allowing code to run after an operation completes across asynchronous API calls, DOM event listeners, timers (`setTimeout`/`setInterval`), and array iteration methods (`map`, `filter`, `forEach`).
>
> **Key Nuance / Trade-off:**
> - Passing callbacks directly for sequential asynchronous tasks creates deeply nested code structures, which led to the adoption of Promises and `async/await`.
>
> **Reference Concept:** `Callbacks & Callback Hell`

---

### What is Callback Hell?

> **Direct Answer:**
> - Callback Hell (also called the "pyramid of doom") occurs when multiple dependent asynchronous operations are nested inside one another's callbacks, producing deeply indented, unreadable, and error-prone code.
>
> **Key Nuance / Trade-off:**
> - Modern JavaScript eliminates callback hell using flat **Promise chaining** (`.then().catch()`) or sequential **`async`/`await`** syntax with `try/catch`.
>
> **Reference Concept:** `Callbacks & Callback Hell`

---

### What is a closure in JavaScript?

> **Direct Answer:**
> - A closure is formed when an inner function remembers and retains live access to variables from its outer lexical scope even after the outer function has completed execution and popped off the call stack.
>
> **Key Nuance / Trade-off:**
> - Closures hold references to outer variables (not copies), meaning retained closures keep referenced objects alive in memory until the inner function is no longer reachable.
>
> **Reference Concept:** `Closures & Lexical Scope Encapsulation`

---

### Why are closures useful?

> **Direct Answer:**
> - Closures enable **data encapsulation** (private state hidden from external code), **function factories** (generating specialized functions with pre-bound parameters), **state persistence** across calls without global variables, and **context preservation** inside asynchronous callbacks and event handlers.
>
> **Key Nuance / Trade-off:**
> - Over-retaining large objects inside long-lived closures (such as unremoved DOM event listeners) can cause memory leaks in single-page applications.
>
> **Reference Concept:** `Closures & Lexical Scope Encapsulation`

---

### How do closures help implement private variables?

> **Direct Answer:**
> - By declaring variables inside an outer function (`let count = 0`) and returning inner methods or functions that read/update them, those variables remain completely inaccessible from outside code while remaining alive via the closure.
>
> **Key Nuance / Trade-off:**
> - Before ES2022 private class fields (`#count`), closures were the standard JavaScript module and factory pattern for enforcing strict data privacy.
>
> **Reference Concept:** `Closures & Lexical Scope Encapsulation`

---

### What is an object in JavaScript?

> **Direct Answer:**
> - An object is a mutable, reference-type collection of key–value pairs (properties), where values can be primitives, other objects, or functions (methods).
>
> **Key Nuance / Trade-off:**
> - Object keys are automatically converted to strings (or Symbols), and object variables hold memory references rather than direct copies.
>
> **Reference Concept:** `Object Creation Patterns & Property Operations`

---

### What are the different ways to create objects in JavaScript?

> **Direct Answer:**
> - Objects can be created using **Object Literals** (`{}`), **Constructor Functions** (`new Person()`), **ES6 Classes** (`class Animal {}`), **`Object.create(proto)`** (explicit prototype inheritance), and **`Object.assign(target, source)`** (merging/cloning properties).
>
> **Key Nuance / Trade-off:**
> - ES6 Classes are syntactic sugar over JavaScript's underlying constructor-prototype model, while `Object.create(null)` creates a pure dictionary object with no `Object.prototype` inheritance.
>
> **Reference Concept:** `Object Creation Patterns & Property Operations`

---

### How can you access object properties?

> **Direct Answer:**
> - Object properties are accessed via **Dot notation** (`user.name`) for static valid identifiers, or **Bracket notation** (`user["age"]`) when property keys are stored dynamically in variables or contain spaces/special characters.
>
> **Key Nuance / Trade-off:**
> - Accessing a non-existent property returns `undefined` rather than throwing an error; use optional chaining (`user?.address?.city`) to avoid `TypeError` on nested `null`/`undefined` objects.
>
> **Reference Concept:** `Object Creation Patterns & Property Operations`

---

### How can you add, update, and delete object properties?

> **Direct Answer:**
> - Add or update properties using assignment (`obj.age = 25;` or `obj["age"] = 25;`), and remove a property completely from the object using the `delete` operator (`delete obj.age;`).
>
> **Key Nuance / Trade-off:**
> - Setting `obj.age = undefined` leaves the key present on the object (`"age" in obj` remains `true`), whereas `delete obj.age` removes the key entirely.
>
> **Reference Concept:** `Object Creation Patterns & Property Operations`

---

### What is the `this` keyword in JavaScript?

> **Direct Answer:**
> - `this` is a dynamic execution-context keyword pointing to the object currently executing a function, determined by how the function is invoked (object method $\to$ caller object, global $\to$ global object/`undefined`, constructor $\to$ new instance, arrow function $\to$ lexical parent scope).
>
> **Key Nuance / Trade-off:**
> - Detaching a method from its object (`const fn = user.greet; fn();`) loses the `this` binding unless explicitly preserved with `.bind(user)` or wrapped in an arrow function.
>
> **Reference Concept:** `Behavior of this Across Contexts`

---

### How does `this` behave inside an object method?

> **Direct Answer:**
> - When a regular function is invoked as a method on an object (`user.greet()`), `this` refers to the object preceding the dot (`user`).
>
> **Key Nuance / Trade-off:**
> - Defining the method as an arrow function breaks this behavior because arrow functions ignore the caller object and inherit `this` from the outer lexical scope.
>
> **Reference Concept:** `Behavior of this Across Contexts`

---

### How does `this` behave inside a JS constructor?

> **Direct Answer:**
> - When invoked with `new`, JavaScript creates an empty object `{}`, binds `this` to that new instance, executes the constructor body to attach properties, and returns `this` automatically.
> - In ES6 subclasses (`extends`), `super()` must be invoked before accessing `this`, or a `ReferenceError` is thrown.
>
> **Key Nuance / Trade-off:**
> - Regular callback functions (`setTimeout(function() {})`) inside a constructor lose the instance `this` binding; use arrow functions inside constructors so callbacks inherit the instance `this`.
>
> **Reference Concept:** `Behavior of this Across Contexts`

---

### Why should arrow functions not be used as object methods?

> **Direct Answer:**
> - Because arrow functions lack their own `this` binding, using one as an object method causes `this` to inherit from the enclosing outer scope (such as `window` or `undefined`) rather than referring to the object itself.
>
> **Key Nuance / Trade-off:**
> - Use ES6 shorthand method syntax (`greet() { console.log(this.name); }`) on objects so `this` binds properly to the object instance.
>
> **Reference Concept:** `Behavior of this Across Contexts`

---

### What is a prototype in JavaScript?

> **Direct Answer:**
> - A prototype is an object from which other JavaScript objects inherit properties and methods via an internal `[[Prototype]]` link, forming the foundation of JavaScript's prototype-based inheritance.
>
> **Key Nuance / Trade-off:**
> - Placing shared methods on a constructor's `.prototype` saves memory by sharing one function definition across every created instance.
>
> **Reference Concept:** `Prototypes & The Prototype Chain`

---

### What is the prototype chain?

> **Direct Answer:**
> - The prototype chain is the hierarchical lookup mechanism JavaScript follows when accessing a property: it checks the object itself, then walks up its `[[Prototype]]` links recursively until the property is found or `null` (the end of the chain) is reached.
>
> **Key Nuance / Trade-off:**
> - Built-in methods like `[1, 2].push(3)` and `obj.toString()` work because every array inherits from `Array.prototype` and every object ultimately inherits from `Object.prototype`.
>
> **Reference Concept:** `Prototypes & The Prototype Chain`

---

### How can methods be added to a prototype?

> **Direct Answer:**
> - Shared methods are added by assigning them onto the constructor's `prototype` object (`Person.prototype.greet = function () { ... }`), or automatically via ES6 `class` method syntax.
>
> **Key Nuance / Trade-off:**
> - Never use an arrow function on `Constructor.prototype` if the method relies on `this` to read instance properties.
>
> **Reference Concept:** `Prototypes & The Prototype Chain`

---

### What is an array in JavaScript?

> **Direct Answer:**
> - An array is a zero-indexed, ordered collection of elements (which can hold mixed data types) stored under a single variable; under the hood, arrays are specialized objects in JavaScript.
>
> **Key Nuance / Trade-off:**
> - Because arrays are objects, `typeof [1, 2, 3]` returns `"object"`; always use `Array.isArray(val)` to check if a value is an array.
>
> **Reference Concept:** `Array Creation & Mutation Operations`

---

### How can you create an array in JavaScript?

> **Direct Answer:**
> - Arrays can be created using **array literals** (`[1, 2, 3]`), the **`new Array()` constructor**, **`Array.of(1, 2, 3)`**, or **`Array.from("abc")`** (converting iterables or array-like objects).
>
> **Key Nuance / Trade-off:**
> - Avoid `new Array(5)` for single numbers because it creates an empty sparse array of length 5 instead of `[5]`; `Array.of(5)` or `[5]` avoids this pitfall.
>
> **Reference Concept:** `Array Creation & Mutation Operations`

---

### How do you add and remove elements from an array?

> **Direct Answer:**
> - Use `push()` to add and `pop()` to remove elements at the **end** of an array ($O(1)$), and `unshift()` to add and `shift()` to remove elements at the **beginning** ($O(n)$ due to index shifting).
>
> **Key Nuance / Trade-off:**
> - All four methods mutate the original array in place; for immutable updates (e.g., in Angular/React state), use spread syntax `[...arr, newItem]` or `.slice()`.
>
> **Reference Concept:** `Array Creation & Mutation Operations`

---

### What are array methods in JavaScript?

> **Direct Answer:**
> - Array methods are built-in higher-order functions on `Array.prototype` for iteration, transformation, filtering, searching, and aggregation—most notably `map()`, `filter()`, `reduce()`, `forEach()`, `find()`, `some()`, and `every()`.
>
> **Key Nuance / Trade-off:**
> - Methods like `map`, `filter`, and `reduce` are non-mutating and return new derived values, enabling clean declarative chaining, whereas `sort()` and `splice()` mutate the original array in place.
>
> **Reference Concept:** `Higher-Order Array Methods (map, filter, reduce, forEach)`

---

### What is the difference between `map()` and `forEach()`?

> **Direct Answer:**
> - `map()` transforms every element via a callback and returns a **brand-new array** of the same length, making it chainable and non-mutating.
> - `forEach()` iterates over elements strictly to execute side effects (logging, saving) and always returns `undefined`.
>
> **Key Nuance / Trade-off:**
> - Neither `map()` nor `forEach()` supports `break` or `continue` early exit; use a `for...of` loop, `.find()`, or `.some()` when short-circuiting is required.
>
> **Reference Concept:** `Higher-Order Array Methods (map, filter, reduce, forEach)`

---

### What does the `filter()` method do?

> **Direct Answer:**
> - `filter()` evaluates a boolean predicate callback against each element and returns a **new array** containing only the elements for which the callback returns `true` (truthy).
>
> **Key Nuance / Trade-off:**
> - `filter()` returns an empty array `[]` if no elements match, and preserves the original array untouched.
>
> **Reference Concept:** `Higher-Order Array Methods (map, filter, reduce, forEach)`

---

### What is `reduce()` used for?

> **Direct Answer:**
> - `reduce()` processes every element in an array sequentially using an accumulator callback to collapse the entire array into a **single resulting value**—such as a numeric sum, average, frequency count, or grouped object.
>
> **Key Nuance / Trade-off:**
> - Always pass an explicit initial accumulator value (e.g., `0` or `{}`) as the second argument to `reduce()` so empty arrays do not throw a `TypeError`.
>
> **Reference Concept:** `Higher-Order Array Methods (map, filter, reduce, forEach)`

---

### What is Destructuring in JavaScript?

> **Direct Answer:**
> - Destructuring (introduced in ES6) is a concise syntax for unpacking properties from objects (`const { name, age } = user`) or positional elements from arrays (`const [first, , third] = colors`) directly into distinct variables.
>
> **Key Nuance / Trade-off:**
> - Object destructuring matches by key name and supports renaming (`{ name: userName }`) and default fallback values (`{ country = "USA" }`), whereas array destructuring matches strictly by positional index and enables clean variable swapping (`[a, b] = [b, a]`).
>
> **Reference Concept:** `Destructuring (Objects, Arrays & Rest Pattern)`

---
