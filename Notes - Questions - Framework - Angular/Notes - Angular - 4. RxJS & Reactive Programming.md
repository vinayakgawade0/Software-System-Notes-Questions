# Angular — RxJS & Reactive Programming

## 1. Observable Fundamentals
- **What is RxJS?** Reactive Extensions for JavaScript—a library for composing asynchronous and event-based programs using observable sequences and functional operators.
- **Observable vs. Promise:**

| Feature | Observable | Promise |
| :--- | :--- | :--- |
| **Execution** | **Lazy** (executes only when `.subscribe()` is called) | **Eager** (executes immediately upon creation) |
| **Values Emitted** | **0 to many** values over time | **Single** value (or rejection) |
| **Cancellation** | **Cancellable** via `unsubscribe()` or operators (`takeUntil`) | **Not cancellable** natively |
| **Operators** | Rich pipeline (`map`, `switchMap`, `retry`, `debounceTime`) | Limited (`.then()`, `.catch()`, `.finally()`) |

- **Cold vs. Hot Observables:**
  - **Cold Observable:** Data producer is created *inside* the Observable per subscriber (e.g., `HttpClient.get()`, `of()`, `from()`). Every subscriber triggers a separate execution.
  - **Hot Observable:** Data producer exists *outside* the Observable and multicasts to active subscribers (e.g., DOM `fromEvent()`, `Subject`, WebSocket streams).

---

## 2. Subjects (Multicast Observables)
A `Subject` acts as both an **Observable** (can be subscribed to) and an **Observer** (can emit values via `.next()`, `.error()`, `.complete()`).

| Subject Type | Initial Value | Replay to Late Subscribers | Common Use Case |
| :--- | :--- | :--- | :--- |
| **`Subject<T>`** | No | None (misses past emissions) | Event buses, component action triggers (`destroy$`) |
| **`BehaviorSubject<T>`** | **Required** | Replays **1 latest** value immediately | Auth state, current user profile, active filter state |
| **`ReplaySubject<T>(n)`** | No | Replays last **`n`** emitted values | Recent activity logs, cached config history |
| **`AsyncSubject<T>`** | No | Emits **only the final** value upon `.complete()` | Heavy computation results where only final output matters |

> **Best Practice:** Keep `BehaviorSubject` `private` inside a service and expose it via `.asObservable()` so external components can read the stream without calling `.next()` directly.

---

## 3. Higher-Order Mapping (Flattening) Operators
Used when an outer Observable emits values that each map to an inner Observable (such as an HTTP request).

- **`switchMap`:** Cancels any active previous inner Observable when a new outer value arrives.
  - *Use Case:* Typeahead search input, route parameter changes (`route.paramMap`), reloading filtered tables.
- **`mergeMap` (alias `flatMap`):** Subscribes to all inner Observables concurrently in parallel; emits results in completion order (order not guaranteed).
  - *Use Case:* Parallel independent API calls (e.g., deleting or uploading multiple selected items at once).
- **`concatMap`:** Queues inner Observables and executes them strictly one after another in arrival order.
  - *Use Case:* Sequential step-by-step form saves or ordered audit updates where sequence matters.
- **`exhaustMap`:** Ignores all new outer emissions while an inner Observable is currently active.
  - *Use Case:* Login, checkout, or submit buttons to prevent duplicate API calls from rapid double-clicks.

---

## 4. Filtering & Rate-Limiting Operators
- **`debounceTime(ms)`:** Emits only after `ms` milliseconds of silence since the last emission (standard for search inputs).
- **`throttleTime(ms)`:** Emits the first value, then ignores subsequent values for `ms` milliseconds (scroll/resize handlers).
- **`distinctUntilChanged()`:** Suppresses consecutive duplicate emissions (prevents duplicate API calls if the user types and deletes a character quickly).
- **`filter(predicate)`:** Emits only values matching the condition (e.g., `filter(term => term.length >= 3)`).
- **`take(n)` / `first()`:** Takes `n` emissions and automatically completes the subscription.

---

## 5. Combination Operators
- **`forkJoin([obs1$, obs2$])`:** Waits for **all** Observables to complete, then emits a single array/object of their final values (equivalent to `Promise.all()`). Ideal for parallel initial page-load HTTP requests.
- **`combineLatest([obs1$, obs2$])`:** Once every source has emitted at least once, emits the latest value from all sources whenever **any** source updates. Ideal for multi-filter search dashboards.
- **`withLatestFrom(other$)`:** Emits only when the **primary** source emits, sampling the latest value from `other$` without triggering on `other$` updates.
- **`zip(obs1$, obs2$)`:** Pairs emissions strictly by index (`1st with 1st`, `2nd with 2nd`).

---

## 6. Multicasting & Caching (`shareReplay`)
- **`shareReplay(1)`:** Converts a cold HTTP Observable into a hot multicast stream and caches the last emitted value for future subscribers.
- **Memory Leak Trap (`refCount`):**
  - `shareReplay(1)` defaults to `refCount: false`—meaning the internal subscription stays alive forever even after all subscribers unsubscribe.
  - Use **`shareReplay({ bufferSize: 1, refCount: true })`** when you want the cache cleared once active subscriber count drops to zero.

---

## 7. Error Handling & Retries
- **`catchError(err => ...)` Placement Rule:**
  - **Outer `pipe()`:** Catches the error and **terminates** the main stream—subsequent user events (like typing in a search box) will no longer trigger API calls.
  - **Inner `pipe()` (inside `switchMap`):** Catches the HTTP error, returns a safe fallback (`of([])`), and **keeps the outer stream alive**.
- **`retry(count)` / `retry({ count: 2, delay: 1000 })`:** Re-subscribes to a failed Observable on transient network errors (e.g., `503 Service Unavailable`).
- **`finalize(() => ...)`:** Executes cleanup logic (e.g., `this.isLoading = false`) on both completion and error (equivalent to `try/finally`).

---

## 8. Subscription Cleanup & Signals Interop
- **Declarative Template Binding (`| async`):** Automatically subscribes and unsubscribes when the component is destroyed.
- **`takeUntilDestroyed()` (Angular 16+):**

```typescript
private destroyRef = inject(DestroyRef);

ngOnInit() {
  this.dataService.updates$
    .pipe(takeUntilDestroyed(this.destroyRef))
    .subscribe(val => this.data = val);
}
```

- **Classic `takeUntil(this.destroy$)`:** Emit and complete a `Subject<void>` inside `ngOnDestroy()`.
- **Signals Interop (`@angular/core/rxjs-interop`):**
  - **`toSignal(obs$, { initialValue: [] })`:** Converts an Observable into a read-only Signal and handles subscription lifecycle automatically.
  - **`toObservable(mySignal)`:** Converts a Signal into an Observable stream so you can apply RxJS operators like `debounceTime` and `switchMap`.