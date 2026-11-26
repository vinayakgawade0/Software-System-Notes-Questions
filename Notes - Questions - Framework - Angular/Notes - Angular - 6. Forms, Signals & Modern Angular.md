# Angular — Forms, Signals & Modern Angular (v16–v21)

## 1. Reactive vs. Template-Driven Forms

| Feature | Reactive Forms (`ReactiveFormsModule`) | Template-Driven Forms (`FormsModule`) |
| :--- | :--- | :--- |
| **Setup & Model** | Explicit in TypeScript (`FormControl`, `FormGroup`, `FormArray`) | Implicit in HTML via directives (`ngModel`, `ngForm`) |
| **Data Flow** | Synchronous & immutable | Asynchronous |
| **Scalability** | Best for complex, dynamic, or multi-step enterprise forms | Best only for simple forms (login, basic search) |
| **Testing** | Easy unit testing in TypeScript without DOM rendering | Requires DOM/fixture rendering |

### Core Building Blocks & Validation
- **`NonNullableFormBuilder`:** Ensures resetting a form restores initial typed defaults instead of `null`.
- **`FormArray`:** Manages dynamic lists of controls or groups (e.g., adding/removing multiple addresses or line items).
- **Custom & Async Validators:**
  - **Sync Validator:** Pure function `(control: AbstractControl) => ValidationErrors | null`.
  - **Cross-Field Validator:** Attached at the `FormGroup` level (e.g., matching `password` and `confirmPassword`).
  - **Async Validator:** Returns `Observable<ValidationErrors | null>` (e.g., checking username availability via API, paired with `debounceTime`).
- **`ControlValueAccessor` (CVA):** Interface (`writeValue`, `registerOnChange`, `registerOnTouched`, `setDisabledState`) used to wrap custom UI inputs so they integrate natively with `formControlName` and `ngModel`.

---

## 2. Signals & Reactive Primitives (v16–v21)
- **`signal(initialValue)`:** Writable reactive state container (`set()`, `update()`); tracks exact template/computed consumers without dirty-checking the whole component tree.
- **`computed(() => ...)`:** Read-only derived signal; lazy and memoized—recomputes only when read and a dependent signal changes.
- **`linkedSignal()` (v19+):** Writable derived signal that resets automatically whenever its source signal changes (e.g., resetting a child dropdown selection when a parent category changes).
- **`effect(() => ...)`:** Runs side effects (logging, `localStorage` sync, DOM APIs) when tracked signals change; avoid mutating state inside `effect` by default.
- **Signal Component APIs:** `input()`, `input.required()`, `output()`, and `model()` (two-way binding) replace `@Input()` / `@Output()` decorators with better type safety and reactivity.
- **`resource()` / `httpResource()`:** Declarative async data loading tied to signals, automatically tracking `value()`, `isLoading()`, and `error()`.
- **RxJS Interop (`@angular/core/rxjs-interop`):** `toSignal(obs$)` auto-subscribes and cleans up on destroy; `toObservable(sig)` converts a signal to an Observable stream.

---

## 3. Modern Control Flow, `@defer` & Zoneless
- **Built-in Control Flow (v17+):** `@if`, `@switch`, and `@for (item of items; track item.id)` replace structural directives (`*ngIf`, `*ngFor`) with better type narrowing and faster diffing. Mandatory `track` prevents full DOM re-creation.
- **`@defer` Blocks:** Declaratively lazy-loads heavy child components/bundles on triggers (`on viewport`, `on interaction`, `on idle`, `when condition`) with `@placeholder`, `@loading`, and `@error` blocks.
- **Zoneless Change Detection:** Eliminates `zone.js` monkey-patching; Angular schedules change detection strictly from Signals, `async` pipe emissions, event listeners, or `ChangeDetectorRef.markForCheck()`.