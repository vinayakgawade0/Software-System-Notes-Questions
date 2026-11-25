# Questions - Angular - 4. RxJS, State Management & HTTP

### What are Observables in Angular, how do they compare to Promises, and what is the `async` Pipe?

> **Direct Answer:**
> - An **Observable** is a lazy RxJS data stream capable of emitting multiple values asynchronously over time, supporting cancellation (`.unsubscribe()`) and declarative `.pipe()` operators—powering `HttpClient`, Router events, and Reactive Forms.
> - Unlike **Promises** (eager, single-value, non-cancelable), Observables only execute when subscribed to.
> - The **`async` pipe** (`users$ | async`) subscribes automatically inside templates, updates the UI on new emissions, and automatically unsubscribes on component destruction to prevent memory leaks.
>
> **Key Nuance / Trade-off:**
> - `HttpClient` Observables complete automatically after emitting once, whereas long-lived streams (`valueChanges`, `interval`) require manual cleanup (`ngOnDestroy` / `takeUntil` / `async` pipe).
>
> **Reference Concept:** `Observables vs Promises & Subscription Lifecycle`

---

### What are RxJS Operators (such as `map`, `filter`, `switchMap`, `mergeMap`, `concatMap`, `exhaustMap`, `tap`, `catchError`)?

> **Direct Answer:**
> - RxJS operators are pure composable functions inside `.pipe()` that transform (`map`), filter (`filter`), log (`tap`), handle errors (`catchError`, `retry`), debounce (`debounceTime`), and flatten higher-order Observables.
> - Flattening operators differ by concurrency behavior: **`switchMap`** cancels the previous in-flight request (typeahead search), **`mergeMap`** runs requests in parallel, **`concatMap`** queues requests strictly sequentially, and **`exhaustMap`** ignores new triggers while a request is active (preventing double form submits).
>
> **Key Nuance / Trade-off:**
> - Choosing the wrong flattening operator causes subtle race conditions—e.g., using `mergeMap` instead of `switchMap` in a search box can display stale search results out of order if an older request returns slower than a newer one.
>
> **Reference Concept:** `Core RxJS Operators & Reactive Patterns`

---

### What is the difference between `Subject` and `BehaviorSubject`?

> **Direct Answer:**
> - Both are multicast RxJS Observables/Observers, but **`Subject`** has no initial value and emits only future values to subscribers, whereas **`BehaviorSubject`** requires an initial value, stores the latest emitted value, and immediately emits that current value to any new late subscriber.
>
> **Key Nuance / Trade-off:**
> - `BehaviorSubject` is the foundational primitive for reactive state services (auth state, shopping carts) because newly mounted components immediately receive the latest state on subscription.
>
> **Reference Concept:** `Subject vs BehaviorSubject & Service-Based State Management`

---

### How can application state be managed in Angular using services and RxJS subjects such as `BehaviorSubject`?

> **Direct Answer:**
> - Encapsulate a private `BehaviorSubject` inside a root singleton `@Injectable({ providedIn: 'root' })` service and expose it publicly only as a read-only stream via `.asObservable()` alongside explicit state-mutation methods.
> - Components subscribe using the `async` pipe to reactively render state updates without exposing `.next()` mutability directly to views.
>
> **Key Nuance / Trade-off:**
> - Service-based `BehaviorSubject` state management requires minimal boilerplate and scales well for small-to-medium apps, though large teams may outgrow it when needing strict action audit logs, time-travel debugging, or normalized entity caching.
>
> **Reference Concept:** `Subject vs BehaviorSubject & Service-Based State Management`

---

### How do state management libraries such as NgRx, Akita, or NGXS help manage application state in large Angular applications?

> **Direct Answer:**
> - Libraries like **NgRx** (Redux + RxJS) enforce a predictable unidirectional architecture centered around a single immutable **Store**, declarative **Actions**, pure **Reducers**, memoized **Selectors**, and **Effects** for async side effects.
> - **NGXS** and **Akita** offer similar centralized stores with reduced TypeScript/decorator boilerplate.
>
> **Key Nuance / Trade-off:**
> - Store libraries prevent tangled cross-service mutations in large enterprise teams and pair seamlessly with `OnPush` change detection via memoized selectors, at the cost of extra upfront architectural boilerplate.
>
> **Reference Concept:** `Store-Based State Management (NgRx, NGXS, Akita)`

---

### How do you handle HTTP requests in Angular using the `HttpClient` module for operations such as GET, POST, PUT, and DELETE?

> **Direct Answer:**
> - Import `HttpClientModule` (or `provideHttpClient()`), inject `HttpClient` into a service, and call typed methods (`this.http.get<T>()`, `post<T>()`, `put<T>()`, `delete<T>()`) which return lazy RxJS Observables.
> - Components or services compose `.pipe(map(...), catchError(...))` and subscribe (or bind via `async` pipe) to trigger execution and handle responses or errors.
>
> **Key Nuance / Trade-off:**
> - Because `HttpClient` Observables are cold, defining `this.http.get(...)` does **not** send any network request until `.subscribe()` (or `async` pipe) is invoked.
>
> **Reference Concept:** `HttpClient REST Operations`

---

### What are HTTP interceptors in Angular and how can they be used for tasks such as authentication, request modification, global error handling, and logging?

> **Direct Answer:**
> - HTTP Interceptors act as global middleware chaining outgoing `HttpRequest`s and incoming `HttpResponse`s—implemented either as functional interceptors (`HttpInterceptorFn`) or `HttpInterceptor` classes registered with `HTTP_INTERCEPTORS` (`multi: true`).
> - They centralize cross-cutting concerns including attaching JWT `Authorization` headers, global API error handling (`catchError`), automatic retries, loading spinner state, and request duration logging.
>
> **Key Nuance / Trade-off:**
> - Because `HttpRequest` objects are immutable, interceptors must call `req.clone({ setHeaders: ... })` to attach authentication tokens before passing the cloned request to `next.handle(clonedReq)`.
>
> **Reference Concept:** `HTTP Interceptors (Auth, Error Handling & Logging Middleware)`

---
