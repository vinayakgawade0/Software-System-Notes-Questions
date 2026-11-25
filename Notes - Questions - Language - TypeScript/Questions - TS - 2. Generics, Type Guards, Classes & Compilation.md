# Questions - TS - 2. Generics, Type Guards, Classes & Compilation

### What are Generics in TypeScript?

> **Direct Answer:**
> - Generics enable writing reusable functions, interfaces, and classes parameterized over type placeholders (`<T>`) so they work across multiple data types while retaining strict compile-time type safety.
>
> **Key Nuance / Trade-off:**
> - Unlike `any` (which erases type relationships between inputs and outputs), Generics preserve the exact input type through to the return type (`identity<string>("TS")` returns `string`).
>
> **Reference Concept:** `Generics: Functions, Interfaces, Constraints & Classes`

---

### What are Generic Functions?

> **Direct Answer:**
> - A generic function declares a type parameter `<T>` on its signature (e.g., `function getFirstElement<T>(arr: T[]): T`) so the return type stays strictly tied to the argument type passed at each call site.
>
> **Key Nuance / Trade-off:**
> - TypeScript automatically infers `<T>` from the passed arguments (`getFirstElement([1, 2, 3])`), making explicit `<number>` annotations optional in most calls.
>
> **Reference Concept:** `Generics: Functions, Interfaces, Constraints & Classes`

---

### What are Generic Interfaces?

> **Direct Answer:**
> - A generic interface parameterizes property types (e.g., `interface ApiResponse<T> { data: T; status: number; }`), enabling reusable wrapper contracts for varying payload types.
>
> **Key Nuance / Trade-off:**
> - Generic interfaces are heavily used in Angular/REST clients (`HttpClient.get<ApiResponse<User>>`) to enforce end-to-end payload safety.
>
> **Reference Concept:** `Generics: Functions, Interfaces, Constraints & Classes`

---

### What are Generic Constraints?

> **Direct Answer:**
> - Generic constraints use the `extends` keyword (e.g., `<T extends { length: number }>`) to restrict which types can be substituted for `<T>`, guaranteeing the existence of required properties or methods inside the generic body.
>
> **Key Nuance / Trade-off:**
> - Constraints combine seamlessly with `keyof` (`<T, K extends keyof T>`) to build type-safe dynamic property getters.
>
> **Reference Concept:** `Generics: Functions, Interfaces, Constraints & Classes`

---

### What are Generic Classes?

> **Direct Answer:**
> - A generic class defines one or more type parameters at the class level (`class Box<T>`) shared across its properties, constructor, and methods—ideal for building reusable repositories, stacks, queues, and state containers.
>
> **Key Nuance / Trade-off:**
> - Static class members cannot reference the class's instance type parameter `<T>`, because static members belong to the constructor rather than individual typed instances.
>
> **Reference Concept:** `Generics: Functions, Interfaces, Constraints & Classes`

---

### What are Modules in TypeScript?

> **Direct Answer:**
> - A module is any TypeScript file containing at least one top-level `import` or `export` statement, isolating its declarations in file-level scope and preventing global namespace pollution.
>
> **Key Nuance / Trade-off:**
> - TypeScript aligns with the standard ECMAScript Module (ESM) system rather than legacy global scripts.
>
> **Reference Concept:** `ES Modules (import/export) vs Legacy Namespaces`

---

### What are Import and Export in TypeScript?

> **Direct Answer:**
> - `export` exposes functions, variables, interfaces, or classes from a module, while `import` consumes those members in another file—supporting **named exports** (multiple per file, matched by exact name) and **default exports** (one primary export per file).
>
> **Key Nuance / Trade-off:**
> - Named exports (`export function add`) are generally preferred in large TypeScript teams because IDE auto-imports, refactoring, and tree-shaking work more consistently across consistent symbol names.
>
> **Reference Concept:** `ES Modules (import/export) vs Legacy Namespaces`

---

### What are Default Exports (and Namespace vs Modules)?

> **Direct Answer:**
> - A **default export** (`export default ...`) exports one primary value from a module and can be imported using any local identifier name.
> - **Namespaces vs. Modules:** Legacy `namespace` blocks wrapped code in a single global object; modern TypeScript replaces namespaces with file-based ES **Modules** (`import`/`export`).
>
> **Key Nuance / Trade-off:**
> - Only one `export default` is allowed per module file.
>
> **Reference Concept:** `ES Modules (import/export) vs Legacy Namespaces`

---

### What is Type Narrowing in TypeScript?

> **Direct Answer:**
> - Type narrowing is the compiler's control-flow analysis process of refining a broader union type (like `string | number`) down to a more specific type inside conditional blocks (`if`/`else`/`switch`) so type-specific methods can be safely called.
>
> **Key Nuance / Trade-off:**
> - Narrowing happens automatically via `typeof`, `instanceof`, `in`, equality checks, and custom type predicates.
>
> **Reference Concept:** `Type Narrowing & Type Guards (typeof, instanceof, in, Custom Predicates)`

---

### What are Type Guards?

> **Direct Answer:**
> - Type guards are runtime expressions or custom functions that instruct TypeScript to narrow a variable's type within a conditional scope—specifically `typeof`, `instanceof`, the `in` operator, and user-defined type predicates (`param is Type`).
>
> **Key Nuance / Trade-off:**
> - Type guards bridge compile-time static types with runtime JavaScript checks, preventing runtime errors like `undefined is not a function`.
>
> **Reference Concept:** `Type Narrowing & Type Guards (typeof, instanceof, in, Custom Predicates)`

---

### What is a `typeof` Type Guard?

> **Direct Answer:**
> - The `typeof` type guard checks JavaScript primitive types (`"string"`, `"number"`, `"boolean"`, `"symbol"`, `"bigint"`, `"undefined"`, `"function"`, `"object"`) inside conditional branches to narrow union types.
>
> **Key Nuance / Trade-off:**
> - Remember JavaScript's `typeof null === "object"` quirk—always check `val !== null` before assuming an `"object"` is non-null.
>
> **Reference Concept:** `Type Narrowing & Type Guards (typeof, instanceof, in, Custom Predicates)`

---

### What is an `instanceof` Type Guard?

> **Direct Answer:**
> - The `instanceof` type guard checks whether an object's prototype chain contains a specific class constructor (`animal instanceof Dog`), narrowing class unions safely for polymorphic method calls.
>
> **Key Nuance / Trade-off:**
> - `instanceof` only works on runtime classes/constructors—it cannot be used on TypeScript `interface` or `type` aliases because those are erased at runtime.
>
> **Reference Concept:** `Type Narrowing & Type Guards (typeof, instanceof, in, Custom Predicates)`

---

### What is the `in` Operator Type Guard?

> **Direct Answer:**
> - The `in` operator checks whether a named property exists on an object (`"swim" in animal`), allowing TypeScript to distinguish between different interface/object shapes inside a union without needing runtime classes.
>
> **Key Nuance / Trade-off:**
> - Unlike `instanceof`, `"prop" in obj` works seamlessly on plain parsed JSON objects typed with interfaces or type aliases.
>
> **Reference Concept:** `Type Narrowing & Type Guards (typeof, instanceof, in, Custom Predicates)`

---

### What are Custom Type Guards?

> **Direct Answer:**
> - Custom type guards are user-defined boolean functions returning a **type predicate** (`parameterName is Type`, e.g., `person is Admin`) that tell TypeScript when `true` is returned, the argument is guaranteed to be of that specific type.
>
> **Key Nuance / Trade-off:**
> - TypeScript trusts the developer's custom guard logic—if the runtime check inside the predicate is flawed, TypeScript's narrowed type will be unsound at runtime.
>
> **Reference Concept:** `Type Narrowing & Type Guards (typeof, instanceof, in, Custom Predicates)`

---

### What are Decorators in TS?

> **Direct Answer:**
> - Decorators (`@expression`) are special declarations attached to classes, methods, properties, or accessors that wrap or attach metadata to separate cross-cutting concerns (such as logging, validation, or `@Component` dependency injection in Angular/NestJS).
>
> **Key Nuance / Trade-off:**
> - Legacy TS decorators require `"experimentalDecorators": true` in `tsconfig.json`, whereas TypeScript 5.0+ also supports the standardized Stage 3 ECMAScript decorators.
>
> **Reference Concept:** `Decorators (@expression) & Metadata`

---

### How do Classes Differ in TypeScript Compared to JavaScript?

> **Direct Answer:**
> - While ES6 JavaScript classes provide syntactic sugar over prototypes with dynamic properties, TypeScript classes add compile-time static typing, access modifiers (`public`, `private`, `protected`), `interface` implementation (`implements`), `abstract` classes, `readonly` fields, and constructor parameter properties.
>
> **Key Nuance / Trade-off:**
> - TypeScript access modifiers (`private`, `protected`) are enforced at compile time only; for true runtime encapsulation in emitted JS, use ES2022 private fields (`#balance`).
>
> **Reference Concept:** `TypeScript Classes & Access Modifiers (public, private, protected)`

---

### What are Access Modifiers in TypeScript?

> **Direct Answer:**
> - Access modifiers control member visibility: **`public`** (default, accessible everywhere), **`private`** (accessible only inside the declaring class), and **`protected`** (accessible inside the declaring class and subclasses).
>
> **Key Nuance / Trade-off:**
> - Combining modifiers directly in constructor parameters (`constructor(private readonly service: MyService)`) automatically declares and assigns the class field in one line, which is idiomatic in Angular dependency injection.
>
> **Reference Concept:** `TypeScript Classes & Access Modifiers (public, private, protected)`

---

### How does TypeScript compilation work?

> **Direct Answer:**
> - `tsc` transpiles `.ts` files into JavaScript in three phases: **Parsing** (building an Abstract Syntax Tree), **Type Checking** (validating all types, interfaces, and guards against the AST), and **Emitting** (performing **Type Erasure** to strip types and emit `.js`, `.d.ts`, and `.js.map` files).
>
> **Key Nuance / Trade-off:**
> - By default, `tsc` may still emit `.js` files even when type errors exist unless `"noEmitOnError": true` is configured in `tsconfig.json`.
>
> **Reference Concept:** `TypeScript Compilation Pipeline (tsc) & Config Files (tsconfig.json vs package.json)`

---

### What are `tsconfig.json` and `package.json` files?

> **Direct Answer:**
> - `tsconfig.json` configures the **TypeScript compiler (`tsc`)** (`target`, `module`, `strict`, `outDir`, included/excluded files), whereas `package.json` configures the **Node.js / npm project** (metadata, dependencies, entry point, and build/test `scripts`).
>
> **Key Nuance / Trade-off:**
> - Running `npm run build` looks up the `"build": "tsc"` script in `package.json`, which invokes `tsc`, which then reads `tsconfig.json` to emit compiled code into `./dist`.
>
> **Reference Concept:** `TypeScript Compilation Pipeline (tsc) & Config Files (tsconfig.json vs package.json)`

---

### Can TypeScript Work with Existing JavaScript Code?

> **Direct Answer:**
> - Yes. Because TypeScript is a superset of JavaScript, teams can migrate existing JS codebases **incrementally** by compiling JS alongside TS, enabling JS type checking, using `.d.ts` declaration files, and importing JS modules directly.
>
> **Key Nuance / Trade-off:**
> - Incremental migration avoids risky big-bang rewrites in large enterprise frontend codebases.
>
> **Reference Concept:** `TypeScript-JavaScript Interoperability (allowJs, checkJs, .d.ts)`

---

### How Does TypeScript Allow Type Checking in JavaScript Files?

> **Direct Answer:**
> - Enabling `"allowJs": true` and `"checkJs": true` in `tsconfig.json` allows `tsc` to include `.js` files and type-check them using type inference and JSDoc annotations (such as `/** @param {number} a */`).
>
> **Key Nuance / Trade-off:**
> - This lets teams catch type bugs inside legacy `.js` files immediately before even renaming them to `.ts`.
>
> **Reference Concept:** `TypeScript-JavaScript Interoperability (allowJs, checkJs, .d.ts)`

---

### What are Declaration Files (`.d.ts`) in TypeScript?

> **Direct Answer:**
> - Declaration files (`.d.ts`) provide static type signatures for JavaScript code without containing runtime implementation (e.g., `declare function greet(name: string): string`), giving TypeScript full autocomplete and type checking for external JS libraries.
>
> **Key Nuance / Trade-off:**
> - Third-party JS libraries that don't bundle their own `.d.ts` files usually have community-maintained type definitions available via the DefinitelyTyped repository (`npm install -D @types/package-name`).
>
> **Reference Concept:** `TypeScript-JavaScript Interoperability (allowJs, checkJs, .d.ts)`

---

### How Can TypeScript Import JavaScript Modules?

> **Direct Answer:**
> - TypeScript imports `.js` modules using standard `import` syntax (`import { add } from "./math-utils.js"`), inferring types directly from the JS source or reading companion `.d.ts` declaration files when available.
>
> **Key Nuance / Trade-off:**
> - With `"allowJs": true`, ES module JS exports integrate seamlessly into TypeScript modules with inferred parameter/return checks.
>
> **Reference Concept:** `TypeScript-JavaScript Interoperability (allowJs, checkJs, .d.ts)`

---
