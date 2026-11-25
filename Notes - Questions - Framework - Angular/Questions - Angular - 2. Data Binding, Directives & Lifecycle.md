# Questions - Angular - 2. Data Binding, Directives & Lifecycle

### What is Data Binding in Angular (Interpolation, Property Binding, Event Binding, Two-Way Binding, and `ngModel`)?

> **Direct Answer:**
> - Data binding synchronizes state between the TypeScript component and HTML template across four modes: **Interpolation** (`{{ expr }}`, Component $\to$ View text), **Property Binding** (`[prop]="expr"`, Component $\to$ DOM property), **Event Binding** (`(event)="fn()"`, View $\to$ Component), and **Two-Way Binding** (`[(ngModel)]="prop"`, synchronizing both directions).
> - `ngModel` requires importing `FormsModule` and combines property binding `[ngModel]` with event binding `(ngModelChange)` under the hood.
>
> **Key Nuance / Trade-off:**
> - Use `[property]` binding rather than `{{ }}` interpolation when binding non-string values (like booleans `[disabled]="isDisabled"` or objects), because interpolation converts values to strings.
>
> **Reference Concept:** `Data Binding Overview & Four Binding Modes`

---

### What are Directives in Angular, and how do Structural Directives (`*ngIf`, `*ngFor`, `*ngSwitch`) differ from Attribute Directives (`ngClass`, `ngStyle`)?

> **Direct Answer:**
> - **Structural Directives** (prefixed with `*`, e.g., `*ngIf`, `*ngFor`, `*ngSwitch`) alter the **DOM structure itself** by adding, removing, or repeating elements. Notably, `*ngIf="false"` removes the element completely from the DOM rather than hiding it with CSS `display: none`.
> - **Attribute Directives** (`ngClass`, `ngStyle`) modify the **appearance or behavior** of an existing DOM element without altering DOM structure.
>
> **Key Nuance / Trade-off:**
> - Components themselves are technically directives that include an HTML template; in Angular 17+, built-in `@if` and `@for` control flow syntax provides a cleaner, faster alternative to `*ngIf` and `*ngFor`.
>
> **Reference Concept:** `Structural Directives (*ngIf, *ngFor, *ngSwitch)`

---

### Describe the Angular component lifecycle and explain the purpose of key lifecycle hooks such as `ngOnInit`, `ngOnChanges`, `ngAfterViewInit`, and `ngOnDestroy` (plus `ngDoCheck`)?

> **Direct Answer:**
> - Angular invokes component lifecycle hooks in sequence: `constructor` $\to$ `ngOnChanges` $\to$ `ngOnInit` $\to$ `ngDoCheck` $\to$ `ngAfterContentInit`/`Checked` $\to$ `ngAfterViewInit`/`Checked` $\to$ `ngOnDestroy`.
> - **`ngOnChanges(changes: SimpleChanges)`** runs before `ngOnInit` and whenever an `@Input()` property updates; **`ngOnInit()`** runs once for API calls and initialization after inputs are ready; **`ngDoCheck()`** runs on every change detection cycle for custom diffing; **`ngAfterViewInit()`** runs once the DOM and `@ViewChild` elements are rendered; and **`ngOnDestroy()`** cleans up RxJS subscriptions, timers, and listeners to prevent memory leaks.
>
> **Key Nuance / Trade-off:**
> - Keep the `constructor()` strictly for Dependency Injection and avoid reading `@Input()` properties there because inputs are not bound yet until `ngOnChanges`/`ngOnInit`. Also avoid heavy logic inside `ngDoCheck()` since it executes on every change detection pass.
>
> **Reference Concept:** `Angular Component Lifecycle Phases & Hook Order`

---
