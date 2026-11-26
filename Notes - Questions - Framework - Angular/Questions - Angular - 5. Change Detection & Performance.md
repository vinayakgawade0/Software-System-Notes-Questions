# Questions - Angular - 5. Change Detection & Performance

### What is change detection in Angular and how does the JavaScript event loop interact with Angular through Zone.js?

> **Direct Answer:**
> - **Change Detection** is Angular's mechanism for keeping the DOM synchronized with component state by walking the component tree after state-changing events and updating only modified DOM nodes.
> - **Zone.js** monkey-patches asynchronous browser APIs (DOM events, `setTimeout`, Promises, HTTP requests) so whenever an async callback completes on the JS Event Loop, Zone.js automatically notifies Angular to execute a change detection pass.
>
> **Key Nuance / Trade-off:**
> - High-frequency tasks (like `mousemove` or animations) can flood Zone.js with unnecessary check cycles; use `ngZone.runOutsideAngular()` or RxJS `debounceTime` to keep the UI responsive, or adopt Angular 17/18+ Zoneless Signals.
>
> **Reference Concept:** `Angular Change Detection Mechanism & Zone.js Event Loop Integration`

---

### What are the different change detection strategies in Angular (`Default` vs `OnPush`), and how do they impact application performance?

> **Direct Answer:**
> - **`Default` Strategy** checks every component in the tree on every change detection cycle regardless of whether its inputs changed, which becomes expensive in large component hierarchies.
> - **`OnPush` Strategy** (`ChangeDetectionStrategy.OnPush`) skips checking the component subtree unless an `@Input()` changes **by reference**, an event fires inside the component/children, an `async`-piped Observable emits, or `ChangeDetectorRef.markForCheck()` is called manually.
>
> **Key Nuance / Trade-off:**
> - `OnPush` relies strictly on **immutable data patterns** (`user = { ...user, name: 'Alice' }`); mutating an object property in place (`user.name = 'Alice'`) keeps the same reference and fails to trigger UI updates under `OnPush`.
>
> **Reference Concept:** `Default vs OnPush Change Detection Strategy`

---

### How can Angular Change Detection be optimized for performance (including `trackBy` in `*ngFor`)?

> **Direct Answer:**
> - Key optimizations include enabling **`ChangeDetectionStrategy.OnPush`** with **immutable data**, binding streams with the **`async` pipe**, using **Lazy Loading** to shrink active component trees, debouncing noisy events (`runOutsideAngular()`), and specifying **`trackBy`** in `*ngFor`.
> - **`trackBy`** returns a unique identifier (like `user.id`) so Angular updates only changed items instead of tearing down and recreating the entire list's DOM nodes when an array reference updates.
>
> **Key Nuance / Trade-off:**
> - Avoid calling heavy methods directly inside Angular HTML templates (`{{ calculateTotal() }}`) because template functions re-execute on every change detection cycle; use pure pipes, memoized selectors, or Signals instead.
>
> **Reference Concept:** `Performance Optimization Techniques & trackBy`

---

### Q. How do `signal()`, `computed()`, `linkedSignal()`, and `effect()` differ?
- **`signal()`:** Holds base writable reactive state.
- **`computed()`:** Read-only derived state; lazy and memoized until dependencies change.
- **`linkedSignal()`:** Writable derived state that auto-resets whenever its source signal updates.
- **`effect()`:** Executes side effects (DOM, analytics, storage) when tracked signals change—not for deriving state.

### Q. Why is `track` mandatory in `@for`, and how does it improve performance?
- `track` provides a unique identity key (e.g., `item.id`) per item so when an array mutates, Angular updates or moves only the changed DOM nodes instead of tearing down and recreating every element in the list.

### Q. How does `OnPush` improve performance, and what common mistake breaks it?
- **How:** Skips checking the component subtree unless an `@Input()` reference changes, a local DOM event fires, an `async` pipe emits, or a Signal read in the template updates.
- **What breaks it:** Mutating an object or array in place (`this.items.push(x)`) instead of passing a new immutable reference (`this.items = [...this.items, x]`).

---