# Notes - Angular - 2. Data Binding, Directives & Lifecycle

## 1. Data Binding

### Data Binding Overview & Four Binding Modes

**Data binding** synchronizes state between the TypeScript Component class and the HTML Template view automatically:

| Binding Type | Data Direction | Syntax | Purpose |
| :--- | :--- | :--- | :--- |
| **Interpolation** | Component $\to$ View | `{{ expression }}` | Renders evaluated text/expressions inside HTML |
| **Property Binding** | Component $\to$ DOM Property | `[property]="expr"` | Sets element/component DOM properties dynamically |
| **Event Binding** | View $\to$ Component | `(event)="handler($event)"` | Listens for DOM/custom events (`click`, `input`, `submit`) |
| **Two-Way Binding** | Component $\leftrightarrow$ View | `[(ngModel)]="prop"` | Combines property + event binding (requires `FormsModule`) |

```html
<!-- Interpolation -->
<h1>{{ title.toUpperCase() }}</h1>

<!-- Property Binding -->
<img [src]="imageUrl">
<button [disabled]="isDisabled">Submit</button>

<!-- Event Binding -->
<button (click)="showMessage()">Click Me</button>
<input (input)="onInput($event)">

<!-- Two-Way Binding (requires FormsModule imported) -->
<input [(ngModel)]="username">
<p>Hello {{ username }}</p>
```

---

## 2. Angular Directives

### Structural Directives (*ngIf, *ngFor, *ngSwitch)

Directives attach dynamic behavior to DOM elements. **Structural directives** (prefixed with `*`) alter the **DOM structure itself** by adding, removing, or recreating elements:

- **`*ngIf`:** Conditionally creates or **completely removes** an element from the DOM (unlike CSS `display: none`, which merely hides it while keeping it in the DOM).
- **`*ngFor`:** Iterates over a collection to stamp out DOM elements dynamically; exposes local variables like `index`, `first`, and `last`.
- **`*ngSwitch`:** Renders only the DOM branch matching `[ngSwitch]` (`*ngSwitchCase` / `*ngSwitchDefault`).

```html
<div *ngIf="isLoggedIn">Welcome User</div>

<ul>
  <li *ngFor="let user of users; index as i">
    {{ i }} - {{ user }}
  </li>
</ul>

<div [ngSwitch]="role">
  <p *ngSwitchCase="'admin'">Admin Dashboard</p>
  <p *ngSwitchCase="'user'">User Dashboard</p>
  <p *ngSwitchDefault>Guest Dashboard</p>
</div>
```

---

### Attribute Directives (ngClass, ngStyle)

**Attribute directives** modify the appearance or behavior of an *existing* DOM element without adding or removing nodes:

- **`ngClass`:** Dynamically toggles CSS classes based on boolean conditions or arrays.
- **`ngStyle`:** Dynamically applies inline CSS styles from key-value objects.

```html
<div [ngClass]="{ 'active': isActive }">Status</div>
<div [ngClass]="['box', 'highlight']">Multiple Classes</div>

<p [ngStyle]="{ 'color': color, 'font-size': '20px' }">Hello Angular</p>
```

---

## 3. Component Lifecycle Hooks

### Angular Component Lifecycle Phases & Hook Order

Angular manages each component from creation through destruction, invoking lifecycle hooks in a deterministic order:

`constructor()` $\to$ `ngOnChanges()` $\to$ `ngOnInit()` $\to$ `ngDoCheck()` $\to$ `ngAfterContentInit()` $\to$ `ngAfterContentChecked()` $\to$ `ngAfterViewInit()` $\to$ `ngAfterViewChecked()` $\to$ `ngOnDestroy()`

| Hook | When It Runs | Primary Use Case |
| :--- | :--- | :--- |
| `constructor()` | Class instantiation (before inputs) | Dependency injection and simple field setup only |
| `ngOnChanges(changes: SimpleChanges)` | Before `ngOnInit` and whenever `@Input()` changes | React to input property updates (`previousValue`, `currentValue`) |
| `ngOnInit()` | **Once** after first `ngOnChanges` | Component initialization, initial HTTP API calls, default state |
| `ngDoCheck()` | Every change detection cycle | Custom deep diffing not caught by default checks (avoid heavy logic) |
| `ngAfterContentInit()` | Once after `<ng-content>` projection | Setup logic for projected external content |
| `ngAfterContentChecked()` | After every projected content check | Respond to projected content updates |
| `ngAfterViewInit()` | Once after component & child views render | Access `@ViewChild` DOM elements, initialize third-party chart/UI libs |
| `ngAfterViewChecked()` | After every view check | Respond to child view updates |
| `ngOnDestroy()` | **Just before** component destruction | **Cleanup:** unsubscribe RxJS streams, clear `setInterval`, detach listeners |

#### Key Lifecycle Patterns:

```typescript
export class UserComponent implements OnInit, OnChanges, AfterViewInit, OnDestroy {
  @Input() name!: string;
  @ViewChild('inputBox') inputBox!: ElementRef;
  private subscription!: Subscription;

  constructor(private userService: UserService) {
    // Only DI here; @Input() properties are NOT yet populated
  }

  ngOnChanges(changes: SimpleChanges) {
    console.log(changes['name'].currentValue);
  }

  ngOnInit() {
    // Safe to read @Input() and trigger API calls
    this.subscription = this.userService.getData().subscribe();
  }

  ngAfterViewInit() {
    // DOM is fully rendered; @ViewChild is now safe to access
    this.inputBox.nativeElement.focus();
  }

  ngOnDestroy() {
    // Prevent memory leaks
    this.subscription.unsubscribe();
  }
}
```

---
