# Notes - JS - 2. Functions, Closures, Objects & Arrays

## 1. Functions, Arrow Functions & Parameters

### Function Declarations vs Function Expressions

| Feature | Function Declaration | Function Expression |
| :--- | :--- | :--- |
| **Syntax** | `function name() {}` | `const fn = function() {}` |
| **Hoisting** | **Fully hoisted** (name and body) | Variable hoisted, function not initialized |
| **Invocation Before Definition** | Allowed | Not allowed (`TypeError` or `ReferenceError`) |
| **Naming** | Always named | Can be anonymous or named |

```javascript
// Function Declaration (fully hoisted)
sayHello();
function sayHello() {
    console.log("Function declaration executed");
}

// Function Expression (cannot call before initialization)
// sayHi(); // ReferenceError / TypeError
const sayHi = function () {
    console.log("Function expression executed");
};
sayHi();
```

---

### Arrow Functions vs Regular Functions

Introduced in ES6, arrow functions (`=>`) provide concise syntax and lexical `this` binding:

```javascript
const add = (a, b) => a + b;
```

| Feature | Regular Function | Arrow Function |
| :--- | :--- | :--- |
| **Syntax** | `function fn() {}` | `() => {}` |
| **`this` Binding** | **Dynamic** (determined by how it is called) | **Lexical** (inherited from surrounding scope) |
| **`arguments` Object** | Available | Not available |
| **Constructor Usage (`new`)** | Can be called with `new` | Cannot be used with `new` (`TypeError`) |
| **Object Methods** | Recommended | Usually avoided |

#### When Arrow Functions Should NOT Be Used:
- **Object methods** needing `this` to refer to the calling object (`obj.greet()`).
- **Constructors** invoked with `new` (`const Person = (name) => { this.name = name; }` throws `TypeError`).
- **Event handlers** where dynamic `this` (the DOM element) is expected.
- **Functions needing the `arguments` object.**

---

### The arguments Object vs Rest Parameters (...args)

- **`arguments` Object:** A local, array-like object available inside all non-arrow functions capturing every argument passed at runtime, even if not declared in the parameter list.
  - Has `.length` and index access (`arguments[0]`), but **lacks array methods** (`.map()`, `.filter()`, `.sort()`). Must be converted via `Array.from(arguments)`.
- **Rest Parameters (`...args`):** Introduced in ES6; collects remaining arguments into a **real JavaScript Array**, works in arrow functions, and supports all Array methods directly.

| Feature | `arguments` Object | Rest Parameters (`...args`) |
| :--- | :--- | :--- |
| **Data Type** | Array-like object | Real Array |
| **Arrow Functions** | Not available (inherits outer non-arrow `arguments`!) | Available |
| **Array Methods** | Requires `Array.from()` conversion | Works out of the box |
| **Partial Capture** | Always captures all passed arguments | Can capture remaining arguments (`(a, ...rest)`) |

```javascript
function sumAll() {
    return Array.from(arguments).reduce((acc, n) => acc + n, 0);
}

const sumModern = (...nums) => nums.reduce((acc, n) => acc + n, 0);
```

---

## 2. Callbacks & Closures

### Callbacks & Callback Hell

- **Callback Function:** A function passed as an argument to another function to be invoked later after an operation or event completes.
  - **Use Cases:** Asynchronous operations (API calls), DOM event handlers, array iteration (`map`, `filter`, `forEach`), and timers (`setTimeout`, `setInterval`).
- **Callback Hell:** Deeply nested callbacks ("pyramid of doom") formed when chaining sequential asynchronous tasks, making code difficult to read, debug, and handle errors in. Modern JS resolves this with **Promises** and **Async/Await**.

```javascript
callback1(() => {
    callback2(() => {
        callback3(() => {
            callback4();
        });
    });
});
```

---

### Closures & Lexical Scope Encapsulation

- **Definition:** A **closure** is formed when an inner function remembers and retains access to variables from its outer lexical scope even after the outer function has finished executing.

#### Four Core Production Use Cases:

| Use Case | How Closures Help |
| :--- | :--- |
| **Data Encapsulation (Private Variables)** | Hides state inside a function scope so external code cannot mutate it directly |
| **Function Factories** | Generates specialized functions sharing logic with pre-configured parameters |
| **State Persistence** | Maintains state across repeated invocations without polluting global variables |
| **Event Handlers / Async Context** | Preserves context (like item IDs or messages) when async callbacks execute later |

```javascript
// Data Encapsulation & Private Variable Example
function createCounter() {
    let count = 0; // Private via closure
    return {
        increment: () => ++count,
        decrement: () => --count
    };
}
const counter = createCounter();
console.log(counter.increment()); // 1
console.log(counter.increment()); // 2
console.log(counter.count);       // undefined (inaccessible externally)

// Function Factory Example
function makeAdder(x) {
    return (y) => x + y;
}
const add5 = makeAdder(5);
console.log(add5(2)); // 7
```

---

## 3. Objects, `this` Keyword & Prototypes

### Object Creation Patterns & Property Operations

An **object** is a reference-type collection of key–value pairs (properties).

#### Five Ways to Create Objects:

| Method | Syntax Complexity | Primary Use Case |
| :--- | :--- | :--- |
| **Object Literal `{}`** | Very low | Quick, single configuration or data objects |
| **Constructor Function** | Medium | Legacy pattern using `new Person(...)` |
| **ES6 Class** | Medium | Modern syntactic sugar over prototypes (`class Animal {}`) |
| **`Object.create(proto)`** | High | Direct manual prototype linking without running a constructor |
| **`Object.assign(target, src)`** | Low | Shallow copying or merging properties between objects |

#### Accessing, Updating & Deleting Properties:
- **Dot Notation:** `obj.property` (static property names).
- **Bracket Notation:** `obj["property"]` (required when property names are dynamic variables or contain spaces).
- **Mutations:** Add/update via `obj.prop = value`; remove via `delete obj.prop`.

---

### Behavior of this Across Contexts

The `this` keyword refers to the execution context (object) running a function. Its value depends on **how the function is invoked**:

| Context | Value of `this` |
| :--- | :--- |
| **Object Method (`obj.fn()`)** | The calling object (`obj`) |
| **Global Scope** | `window` (browser) / `global` (Node.js) / `undefined` in strict mode |
| **Arrow Function** | Lexically inherited from surrounding enclosing scope |
| **Constructor (`new Fn()`)** | The newly created object instance `{}` |

#### Constructor `new` Workflow & `super()` Rules:
1. Creates a brand-new empty object `{}`.
2. Binds `this` to that empty object.
3. Executes constructor body to attach properties.
4. Automatically returns `this`.
- **Subclass Rule:** In ES6 `class Child extends Parent`, you **must call `super()` before accessing `this`**, otherwise JavaScript throws a `ReferenceError`.
- **Callback Pitfall inside Constructors:** Regular functions passed to `setTimeout` reset `this` to global/`undefined`. Use arrow functions inside constructors/methods so `this` remains bound to the instance.

```javascript
function User(name) {
    this.name = name;
    setTimeout(() => {
        console.log("Hello, " + this.name); // "Hello, Zack"
    }, 1000);
}
```

---

### Prototypes & The Prototype Chain

JavaScript uses **prototype-based inheritance**. Every object contains a hidden internal link `[[Prototype]]` (inspected via `Object.getPrototypeOf()` or legacy `__proto__`) pointing to its parent prototype object.

- **Prototype Chain Lookup:** When accessing a property or method on an object:
  1. JS searches the object itself.
  2. If not found, searches `[[Prototype]]`.
  3. Walks up the prototype chain until found or `null` is reached.
- **Memory Efficiency:** Attaching shared methods to `Constructor.prototype.methodName` ensures all instances share a single function reference in memory rather than duplicating methods per instance.

```javascript
function User(name) {
    this.name = name;
}
User.prototype.greet = function () {
    console.log("Hi, I'm " + this.name);
};

const userA = new User("Alice");
const userB = new User("Bob");
console.log(userA.greet === userB.greet); // true (shared prototype method)
```

---

## 4. Arrays, Iteration Methods & Destructuring

### Array Creation & Mutation Operations

An **array** is a zero-indexed, ordered object storing elements of any type.

- **Creation:** Array literal `[1, 2, 3]`, `new Array(1, 2, 3)`, `Array.of(1, 2, 3)`, `Array.from("abc")`.
- **Add/Remove:**

| Operation | Method | Description |
| :--- | :--- | :--- |
| Add at end | `push()` | Appends element(s) to the end |
| Remove from end | `pop()` | Removes and returns last element |
| Add at start | `unshift()` | Prepends element(s) to the beginning |
| Remove from start | `shift()` | Removes and returns first element |

---

### Higher-Order Array Methods (map, filter, reduce, forEach)

| Method | Purpose | Return Value | Mutates Original? |
| :--- | :--- | :--- | :--- |
| `map()` | Transforms every element | **New array** of equal length | No |
| `filter()` | Keeps elements passing predicate | **New array** of matching items | No |
| `reduce()` | Accumulates array into a single value | Single accumulated value | No |
| `forEach()` | Iterates for side effects (logging, saving) | `undefined` | Depends on callback |
| `find()` | Finds first matching element | Element or `undefined` | No |
| `some()` / `every()` | Checks if any / all pass predicate | Boolean (`true` / `false`) | No |

```javascript
const nums = [1, 2, 3, 4];
const doubled = nums.map(n => n * 2);                   // [2, 4, 6, 8]
const evens = nums.filter(n => n % 2 === 0);            // [2, 4]
const sum = nums.reduce((acc, curr) => acc + curr, 0);  // 10
```

---

### Destructuring (Objects, Arrays & Rest Pattern)

Introduced in ES6, **destructuring** unpacks values from arrays or properties from objects into distinct variables cleanly:

| Feature | Object Destructuring | Array Destructuring |
| :--- | :--- | :--- |
| **Syntax** | `{ prop }` | `[ item ]` |
| **Order Matters?** | No (matched by property key name) | Yes (matched by positional index) |
| **Renaming** | `{ name: userName }` | Not applicable |
| **Common Use Cases** | API JSON payloads, function parameter objects | Swapping variables, React hooks (`useState`) |

```javascript
// Object destructuring with renaming, defaults, and rest
const { name: userName, country = "India", ...others } = { name: "Vinayak", age: 25, city: "Mumbai" };

// Array destructuring & skipping elements
const [first, , third] = ["Red", "Green", "Blue"];

// Variable swapping without temp variable
let a = 1, b = 2;
[a, b] = [b, a];

// Function parameter destructuring
function displayUser({ name, age }) {
    console.log(`${name} is ${age}`);
}
```

---
