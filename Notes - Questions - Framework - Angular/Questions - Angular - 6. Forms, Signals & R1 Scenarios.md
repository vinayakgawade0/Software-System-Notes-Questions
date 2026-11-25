# Angular — Forms, Signals & R1 Scenario Questions

### Q1. When a page opens, the same API is called multiple times. How do you debug and fix it?
- **Check:**
  1. Multiple `| async` pipes bound to the same cold HTTP Observable in the template (each creates a separate subscription).
  2. Calling a method inside template interpolation `{{ fetchUsers() }}` (executes on every change detection cycle).
  3. Duplicate initialization in parent + child components or uncleaned subscriptions on route re-entry.
- **Fix:**
  - Cache/multicast the HTTP stream using **`shareReplay({ bufferSize: 1, refCount: true })`**.
  - Bind once via `@let users = users$ | async;` or convert to a Signal with **`toSignal()`**.
  - Move API calls out of template expressions into `ngOnInit` or a Route Resolver.

---

### Q2. How do you handle JWT expiration and concurrent `401` requests in an HTTP Interceptor?
- Maintain `isRefreshing = false` and `refreshSubject = new BehaviorSubject<string | null>(null)`.
- On the first `401` (excluding `/login` and `/refresh` URLs), set `isRefreshing = true`, call `/refresh`, push the new token into `refreshSubject`, and retry the failed request.
- Concurrent requests hitting `401` while `isRefreshing === true` wait via `refreshSubject.pipe(filter(token => token !== null), take(1), switchMap(token => next(addToken(req, token))))` so only **one** refresh network call is made.

---

### Q3. Reactive Forms vs. Template-Driven Forms — when do you choose which, and what is `FormArray`?
- **Reactive Forms:** Chosen for enterprise/complex forms requiring dynamic fields, cross-field validation, RxJS `valueChanges` pipelines (`debounceTime`), and unit testing.
- **Template-Driven:** Chosen only for basic forms (e.g., simple login/search) with minimal validation.
- **`FormArray`:** Manages a dynamic list of `FormControl` or `FormGroup` instances (e.g., adding/removing multiple phone numbers or line items dynamically).

---

### Q4. `signal()` vs. `computed()` vs. `linkedSignal()` vs. `effect()`?
- **`signal()`:** Base writable state.
- **`computed()`:** Read-only, cached derived state.
- **`linkedSignal()`:** Writable derived state that resets when its source signal changes.
- **`effect()`:** Side-effect execution when tracked signals change (never used to derive state).

---

### Q5. Why is `track` mandatory in `@for`, and how does it improve performance?
- `track` gives Angular a unique identity key (like `item.id`) per item so when an array updates, Angular updates/moves only the changed DOM elements instead of destroying and recreating every DOM node in the list.

---

### Q6. How do you prevent blank screens on unhandled UI or API errors in Angular?
- **API Errors:** Catch `HttpErrorResponse` globally in a functional `HttpInterceptorFn` using `catchError`, map status codes (`400`, `403`, `500`) to user-friendly toast alerts, and return fallback data or `EMPTY` where appropriate.
- **Runtime UI Errors:** Implement Angular's global `ErrorHandler` class (`handleError(error: unknown)`) to log the error cleanly and display a fallback banner/reload option instead of crashing the component tree.