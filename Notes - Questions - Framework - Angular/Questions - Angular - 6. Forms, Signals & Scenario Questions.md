# Angular — Forms, Signals & R1 Scenario Questions

### Q1. Reactive Forms vs. Template-Driven Forms — when do you choose which, and what is `FormArray`?
- **Reactive Forms:** Chosen for enterprise/complex forms requiring dynamic fields, cross-field validation, RxJS `valueChanges` pipelines (`debounceTime`), and unit testing without DOM rendering.
- **Template-Driven:** Chosen only for basic forms (e.g., simple login/search) with minimal validation.
- **`FormArray`:** Manages a dynamic list of `FormControl` or `FormGroup` instances (e.g., adding/removing multiple phone numbers or line items dynamically at runtime).

---

### Q2. How do you implement cross-field validation and custom reusable form controls in Angular?
- **Cross-Field Validation:** Attach a validator function to the parent `FormGroup` that reads sibling controls (`group.get('password')?.value === group.get('confirm')?.value`) and returns `{ mismatch: true }` or `null`.
- **Custom Reusable Form Controls:** Implement `ControlValueAccessor` (`writeValue`, `registerOnChange`, `registerOnTouched`, `setDisabledState`) and register `NG_VALUE_ACCESSOR` so custom UI components integrate seamlessly with Reactive Forms.

---

### Q3. How do `signal()`, `computed()`, `linkedSignal()`, and `effect()` differ?
- **`signal()`:** Holds base writable reactive state.
- **`computed()`:** Read-only, lazy, cached derived state that updates only when dependencies change.
- **`linkedSignal()`:** Writable derived state that auto-resets whenever its source signal updates.
- **`effect()`:** Side-effect execution (DOM, analytics, `localStorage`) when tracked signals change—never used to derive state.

---

### Q4. Why is `track` mandatory in `@for`, and how does it improve performance?
- `track` gives Angular a unique identity key (like `item.id`) per item so when an array updates, Angular updates or moves only the changed DOM elements instead of destroying and recreating every DOM node in the list.

---

### Q5. How does `OnPush` improve performance, and what common mistake breaks it?
- **How:** Skips checking the component subtree unless an `@Input()` reference changes, a local DOM event fires, an `async` pipe emits, or a Signal read in the template updates.
- **What breaks it:** Mutating an object or array in place (`this.items.push(x)`) instead of passing a new immutable reference (`this.items = [...this.items, x]`).

---

### Q6. When a page opens, the same API is called multiple times. How do you debug and fix it?
- **Check:**
  1. Multiple `| async` pipes bound to the same cold HTTP Observable in the template (each creates a separate subscription).
  2. Calling a method inside template interpolation `{{ fetchUsers() }}` (executes on every change detection cycle).
  3. Duplicate initialization in parent + child components or uncleaned subscriptions on route re-entry.
- **Fix:**
  - Cache/multicast the HTTP stream using **`shareReplay({ bufferSize: 1, refCount: true })`**.
  - Bind once via `@let users = users$ | async;` or convert to a Signal with **`toSignal()`**.
  - Move API calls out of template expressions into `ngOnInit` or a Route Resolver.

---

### Q7. How do you handle JWT expiration and concurrent `401` requests in an HTTP Interceptor?
- Maintain `isRefreshing = false` and `refreshSubject = new BehaviorSubject<string | null>(null)`.
- On the first `401` (excluding `/login` and `/refresh` URLs), set `isRefreshing = true`, call `/refresh`, push the new token into `refreshSubject`, and retry the failed request.
- Concurrent requests hitting `401` while `isRefreshing === true` wait via `refreshSubject.pipe(filter(token => token !== null), take(1), switchMap(token => next(addToken(req, token))))` so only **one** refresh network call is made.

---

### Q8. How do you prevent blank screens on unhandled UI or API errors in Angular?
- **API Errors:** Catch `HttpErrorResponse` globally in a functional `HttpInterceptorFn` using `catchError`, map status codes (`400`, `403`, `500`) to user-friendly toast alerts, and return fallback data or `EMPTY` where appropriate.
- **Runtime UI Errors:** Implement Angular's global `ErrorHandler` class (`handleError(error: unknown)`) to log the error cleanly and display a fallback banner/reload option instead of crashing the component tree.