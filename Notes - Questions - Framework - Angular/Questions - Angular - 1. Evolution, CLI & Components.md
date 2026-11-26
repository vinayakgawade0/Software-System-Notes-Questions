# Questions - Angular - 1. Evolution, CLI & Components

### What are the major Angular versions from AngularJS to the latest releases, and what key features or architectural improvements were introduced in each version?

> **Direct Answer:**
> - **AngularJS (1.x, 2010)** introduced JavaScript MVC/MVVM with `$scope` and dirty checking, followed by **Angular 2 (2016)** which was a complete TypeScript component-based rewrite with hierarchical DI and the Angular CLI.
> - Key milestones include **v4–v5** (`HttpClient`, PWA, SSR), **v8–v9** (differential loading and the **Ivy renderer** by default for smaller bundles), **v13–v15** (stable **Standalone Components** eliminating mandatory `NgModule`), **v16** (**Signals API**), and **v17–v19** (built-in `@if`/`@for` control flow, Zoneless change detection support, and stabilized Signals).
>
> **Key Nuance / Trade-off:**
> - Angular follows a predictable release cadence providing **6 months of active support** and **12 months of LTS** per major version; enterprise production systems target the latest LTS version for security and stability.
>
> **Reference Concept:** `Angular Evolution & Versioning (AngularJS to Angular 19)`

---

### What is the Angular CLI and what are some commonly used commands for creating, building, testing, and managing Angular projects?

> **Direct Answer:**
> - The Angular CLI (`@angular/cli`) is the official tooling interface for scaffolding, serving, generating code, testing, and building Angular apps.
> - Essential commands include `ng new <app>`, `ng serve -o`, `ng build --prod`, `ng generate component/service` (`ng g c` / `ng g s`), `ng test`, `ng e2e`, `ng lint`, `ng add <pkg>`, `ng update`, and `ng version`.
>
> **Key Nuance / Trade-off:**
> - Using `ng generate` enforces consistent file naming, spec file scaffolding, and automatic module/standalone registration across teams.
>
> **Reference Concept:** `Angular CLI Workflow & Core Commands`

---

### What is Angular Architecture?

> **Direct Answer:**
> - Angular follows a modular, **component-based Single Page Application (SPA)** architecture where views are updated dynamically inside a single HTML page rather than reloading full documents.
> - Core building blocks include **Modules**, **Components**, **Templates**, **Services**, **Dependency Injection**, **Routing**, and **RxJS**, bootstrapping from `main.ts` $\to$ `AppModule` $\to$ Components $\to$ Services $\to$ Backend APIs.
>
> **Key Nuance / Trade-off:**
> - Separating UI rendering (Components/Templates) from business logic and HTTP communication (Services) keeps components thin, focused, and easy to unit test.
>
> **Reference Concept:** `Angular SPA Architecture & AngularJS vs Angular`

---

### Difference Between AngularJS and Angular?

> **Direct Answer:**
> - AngularJS (2010) is a legacy JavaScript MVC framework built on `$scope` and digest-cycle dirty checking.
> - Modern Angular (2+, 2016+) is a complete TypeScript rewrite featuring a component-based architecture, hierarchical Dependency Injection, Angular CLI tooling, and faster change detection (Ivy/Zone.js/Signals).
>
> **Key Nuance / Trade-off:**
> - Because Angular 2+ is a ground-up architectural rewrite rather than an incremental upgrade, migrating directly from AngularJS requires either a rewrite or hybrid bootstrapping via `ngUpgrade`.
>
> **Reference Concept:** `Angular SPA Architecture & AngularJS vs Angular`

---

### What is a Component?

> **Direct Answer:**
> - A Component is the fundamental UI building block in Angular controlling a discrete view region, composed of a TypeScript **Class** (logic/state), an HTML **Template** (view), and scoped **CSS Styles**.
>
> **Key Nuance / Trade-off:**
> - Modern Angular (v15+) supports `standalone: true` components that declare their own template dependencies directly via `imports` without needing an `@NgModule`.
>
> **Reference Concept:** `Angular Components & File Structure`

---

### What is a Module (and Feature vs Shared Modules)?

> **Direct Answer:**
> - An Angular `@NgModule` groups related components, directives, pipes, and services using `declarations`, `imports`, `providers`, `exports`, and `bootstrap`.
> - Large applications organize modules into a **Root Module (`AppModule`)**, **Core Module** (singleton services), **Shared Module** (reusable UI components/pipes exported for reuse), and lazy-loadable **Feature Modules** (domain workflows like `UsersModule`).
>
> **Key Nuance / Trade-off:**
> - Never import a `SharedModule` with `providers` into lazy-loaded Feature Modules, as that can inadvertently create duplicate service instances instead of a single root singleton.
>
> **Reference Concept:** `Angular Modules (NgModule, Feature, Shared & Core Modules)`

---

### What are Smart and Dumb Components in Angular?

> **Direct Answer:**
> - **Smart (Container) Components** handle business logic, manage state, and inject services to fetch data, passing state down to child components.
> - **Dumb (Presentational) Components** focus purely on rendering UI from `@Input()` data and emitting user interactions upward via `@Output()` `EventEmitter`s without calling services directly.
>
> **Key Nuance / Trade-off:**
> - Dumb components pair naturally with `ChangeDetectionStrategy.OnPush` because their inputs only change via immutable parent references, maximizing rendering performance and component reuse.
>
> **Reference Concept:** `Smart vs Dumb Components & Input/Output Communication`

---

### Q. How do you migrate an older Angular application (e.g., v14 → v19/v21) safely?
- Upgrade **one major version at a time** via `ng update @angular/core@X @angular/cli@X` using `update.angular.dev`.
- Run official Angular CLI schematics (`standalone`, `control-flow`, `inject-function`) incrementally after each hop.
- Verify third-party dependencies, unit tests, and build pipelines at every major version before proceeding to the next.

---
