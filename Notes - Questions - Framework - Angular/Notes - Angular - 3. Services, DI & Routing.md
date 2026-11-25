# Notes - Angular - 3. Services, DI & Routing

## 1. Services & Dependency Injection (DI)

### Services & Layered Angular Architecture

A **Service** is an `@Injectable` TypeScript class encapsulating reusable business logic, HTTP data access, shared state, and logging—keeping Components focused strictly on UI rendering.

```text
Components (UI) ---> Services (Business Logic) ---> HttpClient ---> Backend REST APIs
```

```typescript
@Injectable({
  providedIn: 'root'
})
export class UserService {
  constructor(private http: HttpClient) {}

  getUsers() {
    return this.http.get('/api/users');
  }
}
```

---

### Dependency Injection Mechanics (Providers, Injectors, Tokens & Hierarchy)

**Dependency Injection (DI)** is a design pattern and core Angular container mechanism that creates and supplies dependencies externally rather than having classes instantiate them (`new UserService()`), promoting loose coupling, modularity, and easy unit test mocking.

#### Core DI Building Blocks:
1. **Providers:** Recipes instructing Angular how to instantiate a dependency (registered via `providedIn` or `providers: []`).
2. **Injection Tokens:** Unique lookup keys identifying dependencies in the DI container (typically class types like `UserService`, or `InjectionToken` for interfaces/strings).
3. **Hierarchical Injectors:** A tree of injectors mirroring the component tree:
   `Root Injector` $\to$ `Module Injector` $\to$ `Component Injector` $\to$ `Child Component Injector`
   - When a component requests a dependency in its `constructor`, Angular checks the component's injector first and walks **upward** to module and root injectors until found.

#### `providedIn: 'root'` vs `providers: [...]`:

| Feature | `providedIn: 'root'` (Recommended) | `providers: [Service]` (Module / Component) |
| :--- | :--- | :--- |
| **Location** | Inside `@Injectable()` decorator | Inside `@NgModule` or `@Component` metadata |
| **Instance Scope** | **Global Singleton** shared app-wide | Scoped instance (new instance per component subtree) |
| **Tree-Shaking** | **Supported** (unused services dropped from bundle) | Not tree-shakable |

---

## 2. Angular Routing & Navigation

### RouterModule, Route Configuration & Child Routes

**Angular Routing** enables Single Page Application (SPA) navigation without full browser page reloads.

- **`RouterModule.forRoot(routes)`:** Configures routes at the root application level (`AppRoutingModule`).
- **`RouterModule.forChild(routes)`:** Configures routes inside lazy-loaded or feature modules.
- **`<router-outlet>`:** Placeholder directive where matched routed components render.
- **Declarative vs Programmatic Navigation:** Use `<a routerLink="/users">` in templates, or inject `Router` and call `this.router.navigate(['/users', 10])` in TypeScript.

```typescript
const routes: Routes = [
  { path: '', component: HomeComponent },
  { path: 'users', component: UsersComponent },
  {
    path: 'dashboard',
    component: DashboardComponent,
    children: [
      { path: 'profile', component: ProfileComponent },     // /dashboard/profile
      { path: 'settings', component: SettingsComponent }    // /dashboard/settings
    ]
  }
];
```

---

### Lazy Loading & Route Guards

#### Lazy Loading (`loadChildren`)
By default, eager loading bundles every module into `main.js`, slowing initial startup. **Lazy Loading** splits feature modules into separate chunks loaded on demand only when the user navigates to that route:

```typescript
const routes: Routes = [
  {
    path: 'users',
    loadChildren: () => import('./users/users.module').then(m => m.UsersModule)
  }
];
```
- **Benefits:** Smaller initial bundle size, faster first paint, and modular scalability.

#### Route Guards:
Control navigation access and preload route data:

| Guard | Purpose |
| :--- | :--- |
| `CanActivate` | Controls whether a user is allowed to enter a route (auth/roles) |
| `CanDeactivate` | Prevents accidentally leaving a route (e.g., unsaved form changes) |
| `Resolve` | Pre-fetches required route data before activating the component |
| `CanLoad` | Prevents downloading a lazy-loaded module chunk if unauthorized |

---
