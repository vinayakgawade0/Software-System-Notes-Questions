# Angular — Forms, Signals & Modern Angular (v16–v21)

## 1. Reactive vs. Template-Driven Forms
- **Reactive Forms (`ReactiveFormsModule`):**
  - Model-driven in TypeScript using `FormControl`, `FormGroup`, `FormArray`, and `NonNullableFormBuilder`.
  - Synchronous, immutable data flow, scalable for dynamic forms, and easy to unit-test without DOM rendering.
  - Custom validators are pure functions returning `ValidationErrors | null`; async validators (e.g., checking unique username via API) return `Observable<ValidationErrors | null>`.
- **Template-Driven Forms (`FormsModule`):**
  - Directive-driven in HTML (`ngModel`, `ngForm`), asynchronous updates, suitable only for simple forms.
- **ControlValueAccessor (CVA):** Interface (`writeValue`, `registerOnChange`, `registerOnTouched`, `setDisabledState`) used to build reusable custom form controls compatible with Angular Forms.

## 2. Signals & Reactive Primitives (v16–v21)
- **`signal(val)`:** Writable reactive state container (`set()`, `update()`); tracks exact template/computed consumers without dirty-checking the whole component tree.
- **`computed(() => ...)`:** Read-only derived signal; lazy and memoized—recomputes only when read and a dependency changes.
- **`linkedSignal()` (v19+):** Writable derived signal that resets automatically whenever its source signal changes (e.g., resetting selected dropdown option when parent category changes).
- **`effect(() => ...)`:** Runs side effects (logging, `localStorage` sync, DOM APIs) when tracked signals change; avoid mutating state inside `effect`.
- **Signal Component APIs:** `input()`, `input.required()`, `output()`, and `model()` (two-way binding) replace `@Input()` / `@Output()`.
- **RxJS Interop (`@angular/core/rxjs-interop`):** `toSignal(obs$)` auto-subscribes and cleans up on destroy; `toObservable(sig)` converts a signal to an Observable stream.

## 3. Modern Control Flow, `@defer` & Zoneless
- **Built-in Control Flow (v17+):** `@if`, `@switch`, and `@for (item of items; track item.id)` replace structural directives (`*ngIf`, `*ngFor`) with better type narrowing and faster diffing. Mandatory `track` prevents full DOM re-creation.
- **`@defer` Blocks:** Declaratively lazy-loads heavy child components/bundles on triggers (`on viewport`, `on interaction`, `on idle`, `when condition`) with `@placeholder`, `@loading`, and `@error` blocks.
- **Zoneless Change Detection:** Eliminates `zone.js` overhead; Angular schedules change detection strictly from Signals, `async` pipe emissions, event listeners, or `ChangeDetectorRef.markForCheck()`.