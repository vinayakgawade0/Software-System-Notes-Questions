# Notes - JS - 3. Asynchronous JS & Event Loop

## 1. Promises & Concurrency Methods

### Promises: States, Lifecycle & Chaining

A **Promise** is an object representing the eventual completion (or failure) of an asynchronous operation and its resulting value.

#### Three States of a Promise:
| State | Description |
| :--- | :--- |
| **Pending** | Initial state; operation has not yet completed |
| **Fulfilled (Resolved)** | Operation completed successfully with a resolved value |
| **Rejected** | Operation failed with an error/rejection reason |

```javascript
const myPromise = new Promise((resolve, reject) => {
    const success = true;
    if (success) {
        resolve("Data retrieved!");
    } else {
        reject("Error occurred.");
    }
});

myPromise
    .then(value => console.log(value))     // Runs if fulfilled
    .catch(error => console.error(error))  // Runs if rejected
    .finally(() => console.log("Done."));  // Always runs
```

#### Advantages of Promises Over Callbacks:

| Problem with Callbacks | How Promises Solve It |
| :--- | :--- |
| Deeply nested structure ("Callback Hell") | Flat linear chaining via `.then()` |
| Scattered error handling | Centralized error handling via a single `.catch()` |
| Poor readability | Clean, composable asynchronous pipelines |

---

### Promise Concurrency Methods (all, allSettled, race, any)

All four static concurrency combinators accept an iterable (array) of Promises:

| Method | Main Goal | Rejection Condition |
| :--- | :--- | :--- |
| `Promise.all([])` | **All or Nothing:** Wait for all promises to succeed | Rejects immediately if **any single** promise fails |
| `Promise.allSettled([])` | **The Completionist:** Wait for all to finish regardless of outcome | **Never rejects** (returns array of `{ status, value/reason }`) |
| `Promise.race([])` | **The Speedster:** Return the first settled result (success or error) | Rejects if the **fastest** promise rejects |
| `Promise.any([])` | **First Success:** Return the first fulfilled promise, ignoring errors | Rejects **only if every** promise fails (`AggregateError`) |

```javascript
// Promise.allSettled output format:
// [
//   { status: "fulfilled", value: "Photo 1 uploaded" },
//   { status: "rejected", reason: "Photo 2 failed" }
// ]
```

---

## 2. Async / Await

### Async/Await Syntax, Non-Blocking Execution & Error Handling

Introduced in ES2017, **`async`/`await`** is syntactic sugar built on top of Promises that allows asynchronous operations to be written in clean, readable synchronous-looking flow.

- **`async` keyword:** Placed before a function; guarantees the function **always returns a Promise** (non-Promise return values are automatically wrapped in a resolved Promise) and enables `await` inside it.
- **`await` keyword:** Can only be used inside an `async` function; pauses execution of *that specific function* (without blocking the Event Loop or browser) until the Promise settles, then unwraps its resolved value.

```javascript
async function getUserData() {
    try {
        const response = await fetch("https://api.example.com/user");
        const data = await response.json();
        console.log(data);
    } catch (error) {
        console.error("Fetch failed:", error);
    }
}
```

#### Promises (`.then`) vs `async`/`await`:

| Feature | Promises (`.then`) | `async` / `await` |
| :--- | :--- | :--- |
| **Syntax** | Chained callbacks (`.then().catch()`) | Procedural synchronous style |
| **Readability** | Moderate; complex conditional flows get messy | High; supports standard `for`/`if` blocks |
| **Error Handling** | `.catch()` | Standard `try...catch` blocks |
| **Debugging** | Harder to step across chained closures | Easy step-through debugging |

- **Parallelism Rule:** Avoid sequential `await` calls when tasks are independent. Combine `await Promise.all([task1(), task2()])` to execute independent requests concurrently.

---

## 3. JavaScript Runtime & Event Loop

### Event Loop Architecture & Queue Priorities (Microtask vs Macrotask)

JavaScript is **single-threaded** (one Call Stack executing one synchronous instruction at a time). To handle asynchronous non-blocking operations, the JavaScript runtime coordinates five components:

| Runtime Component | Role |
| :--- | :--- |
| **Call Stack** | Executes synchronous function frames (LIFO) |
| **Web APIs / Host APIs** | Executes background timers (`setTimeout`), HTTP (`fetch`), DOM events outside the JS thread |
| **Microtask Queue** | High-priority queue storing `Promise.then()`, `.catch()`, `.finally()`, `queueMicrotask()` |
| **Callback / Macrotask Queue** | Lower-priority queue storing `setTimeout`, `setInterval`, I/O callbacks |
| **Event Loop** | Continuously checks if the Call Stack is empty and schedules queued callbacks |

#### Strict Execution Priority:
1. **1st (Immediate):** Execute all synchronous code on the **Call Stack** until empty.
2. **2nd (High Priority):** Drain the **entire Microtask Queue** (`Promise` callbacks) until empty.
3. **3rd (Low Priority):** Pop **one** task from the **Callback (Macrotask) Queue** (`setTimeout`), execute it, and re-drain the Microtask Queue again.

```javascript
console.log("Start");

setTimeout(() => {
    console.log("Timeout callback");
}, 0);

Promise.resolve().then(() => {
    console.log("Promise resolved");
});

console.log("End");

// Output order:
// 1. Start
// 2. End
// 3. Promise resolved   (Microtask Queue runs before Macrotask Queue)
// 4. Timeout callback   (Callback / Macrotask Queue)
```

---
