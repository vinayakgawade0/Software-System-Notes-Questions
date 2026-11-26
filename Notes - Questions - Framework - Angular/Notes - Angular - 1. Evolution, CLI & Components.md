# Notes - Angular - 1. Evolution, CLI & Components

## 1. Framework Evolution & Tooling

### Angular Evolution & Versioning (AngularJS to Angular 19)

Angular has evolved through three major architectural eras since AngularJS:

| Version | Release | Major Architectural & Feature Milestones |
| :--- | :--- | :--- |
| **AngularJS (1.x)** | 2010 | First JavaScript MVVM/MVC framework for Single Page Applications (SPAs) using `$scope` and dirty checking. |
| **Angular 2** | 2016 | Complete TypeScript rewrite, component-based architecture, hierarchical DI, and Angular CLI. |
| **Angular 4** | 2017 | Smaller view engine compiler, faster rendering, `HttpClient` introduced. |
| **Angular 5** | 2017 | Progressive Web App (PWA) support, Angular Universal (SSR), improved HTTP client. |
| **Angular 6** | 2018 | Angular Elements, CLI `ng update`/`ng add`, RxJS 6, tree-shaking optimizations. |
| **Angular 7** | 2018 | Virtual scrolling, drag-and-drop CDK, build optimizations. |
| **Angular 8** | 2019 | Differential loading, dynamic `import()` lazy routing, Ivy opt-in preview. |
| **Angular 9** | 2020 | **Ivy renderer enabled by default**, smaller bundles, faster compilation, improved testing/debugging. |
| **Angular 10–12** | 2020–2021 | Strict template type checking, inline Sass, enhanced DevTools, TypeScript upgrades. |
| **Angular 13–14** | 2021–2022 | Standalone components preview, RxJS 7 support, typed reactive forms. |
| **Angular 15** | 2022 | **Stable Standalone Components** (optional `NgModule`), enhanced SSR hydration. |
| **Angular 16** | 2023 | **Signals API** preview for fine-grained reactivity, functional route guards. |
| **Angular 17–18** | 2023–2024 | Built-in control flow (`@if`, `@for`), deferrable views (`@defer`), experimental **Zoneless** change detection. |
| **Angular 19** | 2024 | Native ES Modules, Signals stabilized across inputs/outputs/queries, automatic bundle & SSR optimizations. |

- **Three Eras Summary:**
  - *The "Old" Era (v2–v14):* Module-centric (`NgModule`), heavier boilerplate.
  - *The "Revolution" Era (v15–v16):* Standalone Components (eliminating mandatory `app.module.ts`) + Signals preview.
  - *The "Modern" Era (v17–v19):* Built-in control flow (`@if`, `@for`), Zoneless support, Signals as the reactivity standard.
- **LTS & Support Policy:** Each major release receives **6 months of active support** followed by **12 months of Long-Term Support (LTS)** before reaching End-of-Life (EOL). Production teams target the latest LTS release.

## Safe Major Version Migration Strategy (e.g., v14 → v19/v21)
- Upgrade **one major version at a time** using `ng update @angular/core@X @angular/cli@X` following the official Angular Update Guide (`update.angular.dev`).
- Apply built-in migration schematics incrementally (`standalone`, `control-flow`, `inject-function`) rather than rewriting manually.
- Validate third-party library compatibility, unit tests, and SonarQube quality gates at each major step before advancing.

---

### Angular CLI Workflow & Core Commands

The **Angular CLI** (`@angular/cli`) is the official command-line tool for scaffolding, developing, testing, linting, and building Angular applications.

| Category | Command | Purpose |
| :--- | :--- | :--- |
| **Project Setup** | `ng new my-app` | Scaffolds a new Angular workspace |
| **Dev Server** | `ng serve -o` | Starts local live-reload dev server and opens browser |
| **Production Build** | `ng build` / `ng build --prod` | Compiles and bundles optimized production artifacts |
| **Code Generation** | `ng g c my-comp` / `ng g s my-svc` | Generates components (`c`), services (`s`), directives, pipes, modules |
| **Testing & QA** | `ng test` / `ng e2e` / `ng lint` | Runs unit tests (Karma/Jasmine), E2E tests, and linting |
| **Packages & Config** | `ng add <pkg>` / `ng update` / `ng version` / `ng config` | Installs configured libraries (e.g., Angular Material), updates Angular, inspects config |

---

## 2. Core Architecture, Components & Modules

### Angular SPA Architecture & AngularJS vs Angular

Angular uses a **component-based architecture** to build Single Page Applications (SPAs), updating portions of a single HTML page dynamically rather than reloading full pages from the server.

#### Core Building Blocks:
| Building Block | Purpose |
| :--- | :--- |
| **Modules** | Organize related components, directives, pipes, and services into logical units |
| **Components** | Control a specific portion of the UI (Template + TypeScript Class + CSS) |
| **Templates** | HTML view defining the component UI |
| **Services** | Encapsulate reusable business logic and backend HTTP calls |
| **Dependency Injection** | Supplies service instances into components automatically |
| **Routing** | Client-side navigation between views |
| **RxJS** | Reactive asynchronous data streams |

#### High-Level Bootstrap & Request Flow:
`Browser` $\to$ `main.ts` $\to$ `AppModule` (root) $\to$ `Components` $\to$ `Templates (HTML)` $\to$ `Services` $\to$ `Backend APIs`

#### AngularJS (1.x) vs Modern Angular (2+):

| Feature | AngularJS (2010) | Angular (2016+) |
| :--- | :--- | :--- |
| **Language** | JavaScript (`$scope`) | **TypeScript** (classes) |
| **Architecture** | MVC / MVVM | **Component-based** |
| **Performance** | Slower (digest dirty checking) | Faster (Ivy / Zone-based / Signals) |
| **Dependency Injection** | Basic | Hierarchical DI container |
| **Tooling** | Limited | Angular CLI |

---

### Angular Components & File Structure

A **Component** is the fundamental UI building block controlling a slice of the screen. It consists of three parts:
1. **Class (`user.component.ts`):** TypeScript logic and state.
2. **Template (`user.component.html`):** HTML view.
3. **Styles (`user.component.css`):** Scoped CSS styles.

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-user',
  template: `<h2>{{ name }}</h2>`
})
export class UserComponent {
  name = "Vinayak";
}
```

---

### Angular Modules (NgModule, Feature, Shared & Core Modules)

An **Angular Module (`@NgModule`)** groups related components, directives, pipes, and services. Every traditional Angular app has a root **`AppModule`**.

| `@NgModule` Property | Purpose |
| :--- | :--- |
| `declarations` | Components, directives, and pipes belonging to this module |
| `imports` | External modules whose exported items are needed here |
| `providers` | Services registered in this module's injector |
| `exports` | Components/pipes/directives made public to other importing modules |
| `bootstrap` | Root component instantiated at startup (`AppModule` only) |

#### Enterprise Module Categorization:

| Module Type | Purpose | Typical Contents |
| :--- | :--- | :--- |
| **Root Module (`AppModule`)** | Bootstraps the application | `BrowserModule`, `AppRoutingModule`, `AppComponent` |
| **Core Module (`CoreModule`)** | Application-wide singleton services | `AuthService`, HTTP interceptors, logger |
| **Shared Module (`SharedModule`)** | Reusable UI components, pipes, directives | `NavbarComponent`, `LoadingSpinnerComponent`, `CommonModule` |
| **Feature Modules (`UsersModule`)** | Domain-specific business features (lazy-loadable) | `UserListComponent`, `UserDetailComponent`, feature routing |

```text
app/
├── core/       (Singleton services)
├── shared/     (Reusable UI utilities)
├── users/      (Feature module)
├── auth/       (Feature module)
└── dashboard/  (Feature module)
```

---

### Smart vs Dumb Components & Input/Output Communication

Separating **Smart (Container)** components from **Dumb (Presentational)** components improves reusability, maintainability, and unit testability:

| Feature | Smart (Container) Component | Dumb (Presentational) Component |
| :--- | :--- | :--- |
| **Business Logic & State** | Yes | No |
| **API / Service Calls** | Yes (injects services) | No |
| **Data Flow** | Passes data down via `@Input()` | Receives data via `@Input()`, emits events up via `@Output()` |
| **Reusability** | Lower (coupled to feature) | **Highly reusable** across features |

```typescript
// Smart Component
export class UsersComponent {
  users: any[] = [];
  constructor(private userService: UserService) {}
  ngOnInit() {
    this.userService.getUsers().subscribe(data => this.users = data);
  }
  onUserSelected(id: number) { /* handle selection */ }
}

// Dumb Component
export class UserListComponent {
  @Input() users: any[] = [];
  @Output() userSelected = new EventEmitter<number>();
}
```

```html
<!-- Smart Template -->
<app-user-list [users]="users" (userSelected)="onUserSelected($event)"></app-user-list>

<!-- Dumb Template -->
<ul>
  <li *ngFor="let user of users" (click)="userSelected.emit(user.id)">
    {{ user.name }}
  </li>
</ul>
```

---
