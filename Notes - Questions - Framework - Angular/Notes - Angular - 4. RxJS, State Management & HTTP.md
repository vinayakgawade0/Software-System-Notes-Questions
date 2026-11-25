# Notes - Angular - 4. RxJS, State Management & HTTP

## 1. RxJS Observables & Operators

### Observables vs Promises & Subscription Lifecycle

An **Observable** (from RxJS) is a lazy, push-based stream capable of emitting zero, one, or multiple asynchronous values over time (`next`, `error`, `complete`). Angular's `HttpClient`, Router events, and Reactive Form `valueChanges` are built on Observables.

| Feature | Observable | Promise |
| :--- | :--- | :--- |
| **Values Emitted** | Multiple values over time (streams) | Single value only |
| **Execution** | **Lazy** (runs only upon `.subscribe()`) | **Eager** (executes immediately upon creation) |
| **Cancellation** | **Supported** (`.unsubscribe()`, `switchMap`) | Not cancelable natively |
| **Composition** | Rich declarative operators (`.pipe()`) | `.then()` / `.catch()` |

#### Subscription Cleanup & The `async` Pipe:
- **`HttpClient` Observables** complete automatically after emitting a single HTTP response (no manual unsubscribe needed).
- **Long-lived streams** (`valueChanges`, `interval`, custom `Subject`s) **must be unsubscribed** in `ngOnDestroy()` (or via `takeUntil` / `DestroyRef`) to prevent memory leaks.
- **`async` Pipe (`users$ | async`):** Automatically subscribes in the HTML template, updates the view on emissions, and **automatically unsubscribes** when the component is destroyed.

---

### Core RxJS Operators & Reactive Patterns

Operators inside `.pipe(...)` transform, filter, combine, or control timing on Observable streams declaratively:

| Operator | Category | Purpose & Key Use Case |
| :--- | :--- | :--- |
| `map` | Transformation | Transforms each emitted value |
| `filter` | Filtering | Emits only values matching a predicate |
| `tap` | Side Effect | Executes logging/debugging without modifying stream values |
| `debounceTime` / `distinctUntilChanged` | Timing | Waits for typing pauses and ignores duplicate consecutive values |
| `switchMap` | Flattening (Cancel) | **Cancels previous** inner observable on new emission (ideal for **typeahead search**) |
| `mergeMap` | Flattening (Parallel) | Runs multiple inner observables **concurrently** |
| `concatMap` | Flattening (Sequential) | Queues inner observables **strictly in order** |
| `exhaustMap` | Flattening (Ignore) | **Ignores new triggers** while one request is active (ideal for **login/submit buttons**) |
| `catchError` / `retry` | Error Handling | Recovers gracefully or retries transient failures |
| `take` / `takeUntil` | Completion | Auto-completes stream (clean unsubscribe pattern) |

```typescript
// Reactive Typeahead Search Pattern
this.query.valueChanges.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  switchMap(q => this.api.search(q)),
  catchError(() => of([]))
).subscribe(results => this.results = results);
```

---

### Subject vs BehaviorSubject & Service-Based State Management

Both `Subject` and `BehaviorSubject` act simultaneously as an Observable and an Observer for multicasting events to multiple subscribers:

| Feature | `Subject` | `BehaviorSubject` |
| :--- | :--- | :--- |
| **Initial Value** | Not required | **Required** (`new BehaviorSubject(0)`) |
| **Stores Latest Value** | No | **Yes** (accessible via `.getValue()`) |
| **Late Subscribers** | Receive only future emissions | **Immediately receive the latest stored value** |

#### Service-Based State Management Pattern:
For small-to-medium Angular apps, shared state is cleanly encapsulated in a root singleton service exposing a private `BehaviorSubject` as a read-only `Observable` via `.asObservable()`:

```typescript
@Injectable({ providedIn: 'root' })
export class AuthStateService {
  private readonly userSubject = new BehaviorSubject<User | null>(null);
  readonly currentUser$ = this.userSubject.asObservable();

  setUser(user: User | null) {
    this.userSubject.next(user);
  }
}
```

---

### Store-Based State Management (NgRx, NGXS, Akita)

For large enterprise Angular applications with complex cross-component state, caching, and optimistic UI updates, dedicated state libraries (most prominently **NgRx**, built on the **Redux pattern** + RxJS) provide deterministic, unidirectional data flow:

- **Store:** Single immutable centralized state container (Single Source of Truth).
- **Actions:** Plain event objects describing *what happened* (e.g., `[Users Page] Load Users`).
- **Reducers:** Pure functions taking `(currentState, action)` and returning a **new immutable state**.
- **Selectors:** Memoized pure functions deriving slices of state efficiently for components (pairing naturally with `OnPush` + `async` pipe).
- **Effects:** Isolate side effects (HTTP requests, localStorage) by listening to Actions and dispatching success/failure Actions back to the Store.
- *(Alternatives: **NGXS** reduces boilerplate using TypeScript classes and decorators; **Akita** provides an entity-store OOP model).*

---

## 2. HTTP Client & Interceptors

### HttpClient REST Operations

Angular's `HttpClient` (`@angular/common/http`, configured via `HttpClientModule` or `provideHttpClient()`) returns cold RxJS Observables for REST communication:

```typescript
@Injectable({ providedIn: 'root' })
export class DataService {
  constructor(private http: HttpClient) {}

  getData(): Observable<Item[]> {
    return this.http.get<Item[]>('https://api.example.com/data');
  }

  postData(payload: Item): Observable<Item> {
    return this.http.post<Item>('https://api.example.com/data', payload);
  }

  updateData(id: number, payload: Item): Observable<Item> {
    return this.http.put<Item>(`https://api.example.com/data/${id}`, payload);
  }

  deleteData(id: number): Observable<void> {
    return this.http.delete<void>(`https://api.example.com/data/${id}`);
  }
}
```

---

### HTTP Interceptors (Auth, Error Handling & Logging Middleware)

**HTTP Interceptors** implement a middleware pipeline that intercepts every outgoing HTTP request and incoming HTTP response globally.

- **Two Forms:**
  1. **Functional Interceptors (`HttpInterceptorFn`):** Lightweight functions receiving `(req, next)` (standard in modern standalone Angular).
  2. **DI Class Interceptors (`HttpInterceptor`):** Injectable classes registered globally under `HTTP_INTERCEPTORS` with `multi: true` so multiple interceptors chain in registration order.
- **Common Production Uses:**
  - Attaching JWT `Authorization: Bearer <token>` headers automatically.
  - Centralized global error handling (`catchError` redirecting 401/403 or showing toast alerts).
  - Global loading spinner state management (`finalize()`).
  - Request timing, logging, and automatic `retry()` on transient network errors.

---
