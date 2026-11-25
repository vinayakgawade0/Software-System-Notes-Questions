# Questions - JS - 3. Asynchronous JS & Event Loop

### What is a Promise in JavaScript?

> **Direct Answer:**
> - A Promise is an object representing the eventual completion or failure of an asynchronous operation and its resulting value.
> - Every Promise exists in one of three states: **Pending** (initial/in-progress), **Fulfilled** (completed with a value), or **Rejected** (failed with an error).
>
> **Key Nuance / Trade-off:**
> - Once a Promise settles (becomes Fulfilled or Rejected), its state and value are permanently immutable—it cannot transition again.
>
> **Reference Concept:** `Promises: States, Lifecycle & Chaining`

---

### How do `then`, `catch`, and `finally` work with promises?

> **Direct Answer:**
> - `.then()` registers callbacks for successful resolution (and returns a new Promise for chaining), `.catch()` handles rejected errors anywhere upstream in the chain, and `.finally()` executes cleanup logic regardless of whether the Promise resolved or rejected.
>
> **Key Nuance / Trade-off:**
> - Placing a single `.catch()` at the end of a `.then()` chain handles rejections from any preceding step in the pipeline cleanly.
>
> **Reference Concept:** `Promises: States, Lifecycle & Chaining`

---

### What are the advantages of Promises over callbacks?

> **Direct Answer:**
> - Promises replace deeply nested callback pyramids ("Callback Hell") with flat, linear `.then()` chaining, provide centralized error propagation via `.catch()`, and offer predictable, composable state management.
>
> **Key Nuance / Trade-off:**
> - Promises also unlock built-in concurrency combinators (`Promise.all`, `Promise.race`, etc.) that are error-prone to implement manually with raw callbacks.
>
> **Reference Concept:** `Promises: States, Lifecycle & Chaining`

---

### Explain the important Methods in Promise: `Promise.all`, `Promise.allSettled`, `Promise.race`, `Promise.any`?

> **Direct Answer:**
> - **`Promise.all([])`:** Resolves when all input promises succeed, or rejects immediately if **any single** promise fails ("all or nothing").
> - **`Promise.allSettled([])`:** Waits for every promise to finish regardless of success or failure and never rejects, returning an array of `{ status, value/reason }` objects.
> - **`Promise.race([])`:** Settles immediately with the result or error of whichever promise finishes **first**.
> - **`Promise.any([])`:** Resolves with the **first successful** promise, ignoring rejections unless every promise fails (throwing `AggregateError`).
>
> **Key Nuance / Trade-off:**
> - Use `Promise.race` with a timeout Promise to enforce request deadlines, and `Promise.allSettled` for batch operations where partial failures must still be reported individually.
>
> **Reference Concept:** `Promise Concurrency Methods (all, allSettled, race, any)`

---

### What are `async` and `await` in JavaScript?

> **Direct Answer:**
> - `async` and `await` (ES2017) are syntactic sugar built on top of Promises allowing asynchronous code to be written in a clean, synchronous-looking style.
> - Marking a function `async` ensures it always returns a Promise, while `await` pauses execution inside that `async` function (without blocking the Event Loop) until the awaited Promise settles.
>
> **Key Nuance / Trade-off:**
> - Avoid awaiting independent asynchronous operations sequentially in separate lines; group independent calls with `await Promise.all([...])` so they execute concurrently.
>
> **Reference Concept:** `Async/Await Syntax, Non-Blocking Execution & Error Handling`

---

### How does error handling work with `async`/`await`?

> **Direct Answer:**
> - Errors from rejected Promises inside an `async` function are caught using standard synchronous `try...catch` blocks (and optional `finally` blocks) wrapped around `await` statements.
>
> **Key Nuance / Trade-off:**
> - Uncaught rejections inside an `async` function cause the returned Promise itself to reject; failing to handle it triggers an unhandled promise rejection.
>
> **Reference Concept:** `Async/Await Syntax, Non-Blocking Execution & Error Handling`

---

### What is the advantage of `async`/`await` over promises?

> **Direct Answer:**
> - `async`/`await` eliminates verbose `.then()` chains, handles errors cleanly with standard `try...catch` blocks, integrates naturally with `if`/`for` control flow, and makes step-through debugging significantly easier.
>
> **Key Nuance / Trade-off:**
> - Under the hood, `async`/`await` still compiles down to Promises and Microtasks—it improves readability and debugging without changing runtime semantics.
>
> **Reference Concept:** `Async/Await Syntax, Non-Blocking Execution & Error Handling`

---

### What is the JavaScript Event Loop?

> **Direct Answer:**
> - Because JavaScript is single-threaded, the **Event Loop** coordinates asynchronous execution by monitoring the **Call Stack**, **Microtask Queue** (`Promise.then`, `catch`, `finally`), and **Callback / Macrotask Queue** (`setTimeout`, `setInterval`, I/O).
> - Whenever the Call Stack is empty, the Event Loop drains the **entire Microtask Queue** before executing the next task from the Callback Queue.
>
> **Key Nuance / Trade-off:**
> - Because Microtasks always run before Macrotasks, a `Promise.resolve().then(...)` callback executes *before* a `setTimeout(..., 0)` callback even when `setTimeout` was scheduled first.
>
> **Reference Concept:** `Event Loop Architecture & Queue Priorities (Microtask vs Macrotask)`

---
