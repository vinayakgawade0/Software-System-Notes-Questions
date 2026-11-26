# Notes - Angular - 5. Change Detection & Performance

## 1. Change Detection & Zone.js

### Angular Change Detection Mechanism & Zone.js Event Loop Integration

**Change Detection** is Angular's internal synchronization mechanism that checks component data models and updates the DOM whenever application state changes.

#### How `Zone.js` Connects the JS Event Loop to Angular:
1. Angular runs inside **Zone.js**, which monkey-patches asynchronous browser APIs (`addEventListener` clicks/inputs, `setTimeout`, Promises, `XMLHttpRequest`/`fetch`).
2. When a user clicks a button or an HTTP response resolves on the JavaScript Event Loop, Zone.js intercepts the async task completion and notifies Angular.
3. Angular triggers a **change detection cycle**, walking from the root component down the component tree, comparing template expressions against previous values, and patching only changed DOM nodes.

```text
User Event / setTimeout / HTTP Response
       ↓
Zone.js intercepts async completion
       ↓
Angular runs Change Detection across component tree
       ↓
DOM updates where values changed
```

#### Escaping Zone.js for Heavy Background Work:
- For high-frequency tasks that don't require immediate UI updates (canvas animations, mousemove tracking, WebGL), inject `NgZone` and run them via `this.ngZone.runOutsideAngular(() => ...)` to avoid triggering excessive change detection cycles.

---

### Default vs OnPush Change Detection Strategy

| Strategy | Behavior | Performance Impact |
| :--- | :--- | :--- |
| **`ChangeDetectionStrategy.Default`** | Checks **every component** in the component tree on every change detection cycle | Simple, but degrades performance in large trees |
| **`ChangeDetectionStrategy.OnPush`** | Skips the component subtree **unless** a specific trigger occurs | **High performance** for large enterprise apps |

#### When `OnPush` Components Re-Check:
1. An **`@Input()` reference changes** (requires **immutable data updates**, e.g., `user = { ...user, name: 'Alice' }`—mutating `user.name = 'Alice'` will *not* trigger `OnPush`!).
2. A **DOM event** originates inside the component or one of its children (`(click)`).
3. An **Observable** bound with the `async` pipe emits a new value.
4. Manual invocation via `ChangeDetectorRef.markForCheck()` or `detectChanges()`.

```typescript
import { ChangeDetectionStrategy, Component, Input } from '@angular/core';

@Component({
  selector: 'app-user',
  template: `{{ user.name }}`,
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class UserComponent {
  @Input() user!: { name: string };
}
```

---

### Performance Optimization Techniques & trackBy

| Optimization Technique | Performance Benefit |
| :--- | :--- |
| **`ChangeDetectionStrategy.OnPush`** | Prunes unchanged component subtrees from check cycles |
| **Immutable Data Updates** | Enables fast `===` reference checks for `OnPush` |
| **`async` Pipe** | Triggers `markForCheck()` automatically on emissions and auto-unsubscribes |
| **`trackBy` in `*ngFor`** | Prevents Angular from destroying/recreating all DOM `<li>` nodes when arrays update |
| **Lazy Loading (`loadChildren`)** | Reduces initial JS bundle size and component tree footprint |
| **`ngZone.runOutsideAngular()` & Debouncing** | Prevents high-frequency scroll/animation/input events from flooding change detection |

#### `trackBy` Example:
Without `trackBy`, replacing an array from an API causes Angular to tear down and rebuild every DOM list node. With `trackBy`, Angular compares unique IDs and updates only modified rows:

```html
<li *ngFor="let user of users; trackBy: trackById">
  {{ user.name }}
</li>
```
```typescript
trackById(index: number, user: { id: number }): number {
  return user.id;
}
```

---

## Modern Signals & Zoneless Change Detection (v16–v21)
- **`signal(initialValue)`:** Writable reactive primitive (`set()`, `update()`) providing fine-grained reactivity without checking the entire component tree.
- **`computed(() => ...)`:** Read-only derived signal; lazy and memoized—recomputes only when read and a dependent signal changes.
- **`linkedSignal()` (v19+):** Writable derived signal that resets automatically when its source signal updates (e.g., resetting a child dropdown selection when a parent category changes).
- **`effect(() => ...)`:** Side-effect runner (logging, `localStorage` sync, DOM APIs) triggered when tracked signals change; avoid mutating state inside `effect`.
- **Signal Component APIs:** `input()`, `input.required()`, `output()`, and `model()` (two-way binding) replace `@Input()` / `@Output()` decorators.
- **`resource()` / `httpResource()`:** Declarative async data loading tied to signals, exposing `value()`, `isLoading()`, and `error()`.
- **Zoneless Change Detection:** Removes `zone.js` monkey-patching; UI updates are scheduled directly by Signals, `async` pipe, DOM events, or `ChangeDetectorRef.markForCheck()`.

## Modern Control Flow & `@defer` (v17+)
- **`@for` & mandatory `track`:** Replaces `*ngFor` + `trackBy`. Requiring `track item.id` lets Angular reuse and move existing DOM nodes instead of destroying and recreating the list.
- **`@defer` Blocks:** Declaratively lazy-loads heavy child components/bundles on triggers (`on viewport`, `on interaction`, `on idle`, `when condition`) with `@placeholder`, `@loading`, and `@error` states.

---