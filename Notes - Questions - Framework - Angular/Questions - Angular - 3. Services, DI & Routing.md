# Questions - Angular - 3. Services, DI & Routing

### What are Services and Dependency Injection in Angular, and what roles do providers, injectors, and the DI hierarchy play?

> **Direct Answer:**
> - **Services** encapsulate reusable business logic, shared state, and HTTP calls outside UI components.
> - Angular's **Dependency Injection (DI)** container resolves constructor dependencies automatically through three components: **Providers** (recipes telling Angular how to construct instances), **Injection Tokens** (identifiers such as class types), and **Hierarchical Injectors** (walking upward from Component Injector $\to$ Module Injector $\to$ Root Injector until a provider matches).
>
> **Key Nuance / Trade-off:**
> - Always prefer `@Injectable({ providedIn: 'root' })` over `@NgModule({ providers: [...] })` because `providedIn: 'root'` creates an app-wide **singleton** and enables **tree-shaking** (stripping unused services from production bundles).
>
> **Reference Concept:** `Dependency Injection Mechanics (Providers, Injectors, Tokens & Hierarchy)`

---

### What is the difference between `providedIn` and `providers`?

> **Direct Answer:**
> - `providedIn: 'root'` inside `@Injectable()` registers a single **global singleton** instance in the root injector and supports **tree-shaking**.
> - `providers: [MyService]` inside `@Component` or `@NgModule` creates a **scoped service instance** dedicated to that component (and its children) or module, and is not tree-shakable.
>
> **Key Nuance / Trade-off:**
> - Use component-level `providers` intentionally only when each component instance requires its own isolated state copy (such as a multi-tab form wizard).
>
> **Reference Concept:** `Dependency Injection Mechanics (Providers, Injectors, Tokens & Hierarchy)`

---

### What is `RouterModule`, Route Configuration, Child Routes, and Programmatic Navigation in Angular?

> **Direct Answer:**
> - `RouterModule` enables client-side SPA navigation using `RouterModule.forRoot(routes)` at the application root and `RouterModule.forChild(routes)` in feature modules, rendering matched components inside `<router-outlet>`.
> - **Child Routes** (`children: [...]`) nest sub-views (like `/dashboard/profile`) inside a parent layout's nested `<router-outlet>`, while programmatic navigation uses the injected `Router` service (`this.router.navigate(['/users', 10])`).
>
> **Key Nuance / Trade-off:**
> - Always call `forRoot()` strictly once in the root routing module and `forChild()` in all feature modules to avoid registering duplicate router service singletons.
>
> **Reference Concept:** `RouterModule, Route Configuration & Child Routes`

---

### What is Lazy Loading and what are Route Guards in Angular?

> **Direct Answer:**
> - **Lazy Loading** (`loadChildren: () => import(...)`) defers loading feature modules until the user navigates to their route, significantly shrinking the initial bundle size and speeding up startup.
> - **Route Guards** intercept navigation to enforce rules: `CanActivate` (restricts route entry), `CanDeactivate` (warns before leaving unsaved forms), `Resolve` (pre-fetches data before activation), and `CanLoad` (blocks downloading lazy chunks if unauthorized).
>
> **Key Nuance / Trade-off:**
> - Note that in modern Angular (v15+), class-based guards are being replaced by lighter **functional route guards** (`CanActivateFn`), and standalone components can be lazy-loaded directly via `loadComponent`.
>
> **Reference Concept:** `Lazy Loading & Route Guards`

---
