# Angular — RxJS & Reactive Programming Interview Questions

### Q1. What is the difference between an Observable and a Promise?
- **Execution:** Observables are **lazy** (do nothing until `.subscribe()` is called); Promises are **eager** (execute immediately on creation).
- **Emissions:** Observables emit **0 to many** values over time; Promises resolve or reject **once**.
- **Cancellation:** Observables can be cancelled via `.unsubscribe()` or `switchMap`/`takeUntil`; Promises cannot be cancelled natively.
- **Composition:** Observables offer declarative operators (`debounceTime`, `retry`, `switchMap`) via `.pipe()`.

---

### Q2. Compare `switchMap`, `mergeMap`, `concatMap`, and `exhaustMap` with real-world scenarios.
- **`switchMap`:** Cancels the in-flight inner request when a new emission arrives. *Scenario:* Search typeahead input or route ID changes.
- **`exhaustMap`:** Ignores new emissions until the active inner request finishes. *Scenario:* Preventing duplicate form/payment submissions on double-click.
- **`concatMap`:** Runs inner requests sequentially in strict FIFO order. *Scenario:* Ordered multi-step updates or sequential file chunks.
- **`mergeMap`:** Runs all inner requests concurrently without waiting or cancelling. *Scenario:* Parallel independent delete/upload actions.

---

### Q3. Why does placing `catchError` on the outer `pipe()` break a search bar after one API error, and how do you fix it?
- **Why:** In RxJS, an unhandled error terminates an Observable stream permanently. If `catchError` sits on the outer `valueChanges.pipe(...)`, one `500` error catches at the outer level and completes the search stream—future keystrokes stop firing API calls.
- **Fix:** Place `catchError` **inside** `switchMap` on the inner HTTP Observable so the failed request resolves to a fallback (e.g., `of([])`) while keeping the outer `valueChanges` stream alive:

```typescript
this.searchControl.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(term =>
    this.api.search(term).pipe(
      catchError(() => of([])) // Keeps outer stream alive
    )
  )
);
```

---

### Q4. When a page loads, the same API is called multiple times. How do you diagnose and fix it using RxJS?
- **Root Causes:**
  1. Multiple `| async` pipes in the template bound to the same cold `HttpClient` Observable (each `async` pipe creates a separate subscription and HTTP call).
  2. Calling an API-returning method inside template expressions `{{ getUsers() }}`.
  3. Subscribing to the same service method independently across parent and child components.
- **Fixes:**
  - Cache and multicast the stream in the service using **`shareReplay({ bufferSize: 1, refCount: true })`**.
  - Bind once in the template using `@let users = users$ | async;` or convert once with `toSignal()`.

---

### Q5. Difference between `Subject`, `BehaviorSubject`, `ReplaySubject`, and `AsyncSubject`?
- **`Subject`:** No initial value; subscribers only receive values emitted *after* they subscribe.
- **`BehaviorSubject`:** Holds a current value (requires an initial value) and immediately emits that latest value to any new subscriber—ideal for auth/user state.
- **`ReplaySubject(n)`:** Buffers and replays the last `n` values to new subscribers without requiring an initial value.
- **`AsyncSubject`:** Emits only the very last value, and only when the source Observable completes.

---

### Q6. When do you use `forkJoin` vs. `combineLatest` vs. `withLatestFrom`?
- **`forkJoin`:** Waits for all input Observables to **complete**, then emits one combined result. Best for parallel initial page-load HTTP calls where you need all responses before rendering.
- **`combineLatest`:** Emits whenever **any** input Observable emits (once all have emitted at least once). Best for reactive dashboards combining multiple filter dropdowns/streams.
- **`withLatestFrom`:** Triggers **only** when the primary source emits, attaching the latest snapshot of secondary streams. Best for combining a button click with current form/filter state.

---

### Q7. How do you handle JWT expiration and queue concurrent `401` API requests in an HTTP Interceptor?
- Track `isRefreshing = false` and `refreshToken$ = new BehaviorSubject<string | null>(null)`.
- On the first `401` (ignoring `/login` and `/refresh`), set `isRefreshing = true`, call `/refresh`, emit the new access token into `refreshToken$`, and retry the original request.
- Concurrent requests that fail with `401` while `isRefreshing === true` wait on:

```typescript
refreshToken$.pipe(
  filter(token => token !== null),
  take(1),
  switchMap(token => next(addAuthHeader(req, token!)))
)
```

- This guarantees `/refresh` is invoked **only once** while all paused requests resume cleanly with the new token.

---

### Q8. How do you prevent memory leaks from RxJS subscriptions in Angular?
- **Prefer `| async` pipe or `toSignal()`:** Both bind to the component lifecycle and unsubscribe automatically on destroy.
- **Use `takeUntilDestroyed()` (Angular 16+):** Automatically completes the stream when `DestroyRef` triggers.
- **Classic `takeUntil(this.destroy$)`:** Pipe `takeUntil(this.destroy$)` before `.subscribe()` and call `this.destroy$.next(); this.destroy$.complete();` in `ngOnDestroy()`.
- **Watch `shareReplay(1)`:** Use `{ bufferSize: 1, refCount: true }` so cached streams release resources when no active subscribers remain.

---

### Q9. How do RxJS and Angular Signals work together (`toSignal` vs. `toObservable`)?
- **Use Signals** for synchronous component/UI state (`computed`, `input`, local toggles) to avoid manual subscriptions and Zone.js overhead.
- **Use RxJS** for complex asynchronous event coordination (`debounceTime`, `switchMap`, `retry`, WebSockets, HTTP interceptors).
- **Bridge them** using `@angular/core/rxjs-interop`: `toSignal()` exposes an HTTP/RxJS stream cleanly to a Signal-based template, while `toObservable()` turns a Signal (like a search query) into a stream for RxJS rate-limiting operators.