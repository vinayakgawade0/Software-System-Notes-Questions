# Questions - TS - 1. Fundamentals, Type System & Interfaces

### What is TypeScript?

> **Direct Answer:**
> - TypeScript is a strongly typed superset of JavaScript that adds static typing, interfaces, generics, and compile-time error checking on top of standard JavaScript.
> - TypeScript code is transpiled by the TypeScript Compiler (`tsc`) into plain JavaScript so it can run in browsers, Node.js, or Deno; all valid JavaScript is also valid TypeScript.
>
> **Key Nuance / Trade-off:**
> - TypeScript performs **type erasure** during compilation—all static type annotations are stripped from the emitted `.js` output and do not exist at runtime.
>
> **Reference Concept:** `TypeScript Overview, Benefits & Large-Scale Adoption`

---

### Why use TypeScript?

> **Direct Answer:**
> - TypeScript catches type mismatches and silent JavaScript coercion errors at **compile time** rather than in production at runtime.
> - It significantly improves developer productivity via rich IDE IntelliSense, autocomplete, safe project-wide refactoring, and self-documenting code contracts.
>
> **Key Nuance / Trade-off:**
> - It adds a compilation step (`tsc`) and initial learning curve for advanced type constructs, which pays off substantially as project size and team headcount grow.
>
> **Reference Concept:** `TypeScript Overview, Benefits & Large-Scale Adoption`

---

### How does TypeScript differ from JavaScript?

> **Direct Answer:**
> - JavaScript is dynamically typed and interpreted directly at runtime, lacking native static interfaces or generics.
> - TypeScript is statically typed, enforces type safety via compile-time static analysis, supports interfaces, generics, and access modifiers, and must be compiled down to JavaScript before execution.
>
> **Key Nuance / Trade-off:**
> - Because TypeScript enforces structural typing at compile time, changing an API response interface immediately flags every affected component across the codebase before deployment.
>
> **Reference Concept:** `TypeScript Overview, Benefits & Large-Scale Adoption`

---

### Why is TypeScript used in large applications?

> **Direct Answer:**
> - Large applications involve thousands of files and multiple teams where dynamically typed JavaScript suffers from untraceable type mutations, risky refactoring, and weak IDE tooling.
> - TypeScript resolves this by enforcing strict **Interfaces**, reusable **Generics**, compile-time contract validation, and safe automated refactoring across module boundaries.
>
> **Key Nuance / Trade-off:**
> - Frameworks like Angular adopt TypeScript natively so dependency injection, component contracts, and HTTP responses remain strictly verified across large enterprise teams.
>
> **Reference Concept:** `TypeScript Overview, Benefits & Large-Scale Adoption`

---

### When I have compiled the TypeScript code, files like `.js`, `.d.ts`, `.js.map`, and `.d.ts.map` are generated in `dist` folder—what are those?

> **Direct Answer:**
> - **`.js`:** The transpiled, executable JavaScript code stripped of TypeScript type annotations for browsers/Node.js.
> - **`.d.ts` (Declaration File):** Type metadata blueprints describing exported shapes (functions, classes, interfaces) so external consumers get IntelliSense and type checking.
> - **`.js.map` (Source Map):** Maps runtime JavaScript execution and stack traces back to the exact lines in original `.ts` files during debugging.
> - **`.d.ts.map` (Declaration Source Map):** Links `.d.ts` declarations back to original `.ts` source implementations so IDE "Go to Definition" jumps directly to `.ts` code.
>
> **Key Nuance / Trade-off:**
> - Source maps (`.js.map`) are essential for debugging, but in production web deployments teams often upload them privately to error trackers (e.g., Sentry) rather than serving original `.ts` source publicly.
>
> **Reference Concept:** `Compiled Output Artifacts (dist/ Folder)`

---

### What is Type Annotation in TypeScript?

> **Direct Answer:**
> - Type annotation is explicitly specifying the data type of a variable, function parameter, or return value using the `: Type` syntax (e.g., `let age: number = 25;`).
>
> **Key Nuance / Trade-off:**
> - Explicitly annotate function parameters, return signatures, and uninitialized variables (`let score: number;`), while letting TypeScript infer obvious local variable initializations to avoid noisy code.
>
> **Reference Concept:** `Type Annotations vs Type Inference`

---

### What is Type Inference in TypeScript?

> **Direct Answer:**
> - Type inference is TypeScript's automatic deduction of a variable's, array's, object's, or function return's type directly from its assigned value without explicit annotations (e.g., `let age = 25;` infers `number`).
>
> **Key Nuance / Trade-off:**
> - Always prefer inference for straightforward local initializations so code stays concise while preserving full static type safety.
>
> **Reference Concept:** `Type Annotations vs Type Inference`

---

### What are the Primitive Types in TypeScript?

> **Direct Answer:**
> - TypeScript's primitive types correspond to JavaScript's 7 immutable primitives with static typing support: `string`, `number`, `boolean`, `null`, `undefined`, `symbol`, and `bigint`.
>
> **Key Nuance / Trade-off:**
> - Always use lowercase primitive types (`string`, `number`, `boolean`) rather than capitalized wrapper object types (`String`, `Number`, `Boolean`).
>
> **Reference Concept:** `Primitive Types, any vs unknown, void vs never`

---

### What is the Difference Between `any` and `unknown`?

> **Direct Answer:**
> - Both `any` and `unknown` accept values of any type, but **`any` disables all compiler type checking** (allowing unsafe method calls), whereas **`unknown` is type-safe** and forces the developer to narrow or validate the type (e.g., `typeof val === "string"`) before performing operations on it.
>
> **Key Nuance / Trade-off:**
> - Always prefer `unknown` over `any` for external/untrusted data (such as parsed JSON payloads or `catch (err: unknown)`) so type safety is preserved.
>
> **Reference Concept:** `Primitive Types, any vs unknown, void vs never`

---

### What are Union Types in TypeScript?

> **Direct Answer:**
> - A union type (`|`) allows a variable or parameter to hold values of multiple specified types (e.g., `string | number`).
>
> **Key Nuance / Trade-off:**
> - When operating on a union type, TypeScript only permits methods shared across *all* members of the union unless you first narrow the type using `typeof`, `instanceof`, or custom guards.
>
> **Reference Concept:** `Union Types (|) & Intersection Types (&)`

---

### What are Intersection Types?

> **Direct Answer:**
> - An intersection type (`&`) combines multiple types into a single composite type that must satisfy all properties of every constituent type simultaneously (e.g., `type Staff = Person & Employee`).
>
> **Key Nuance / Trade-off:**
> - Intersecting incompatible primitive types (e.g., `string & number`) resolves to `never` because no value can satisfy both simultaneously.
>
> **Reference Concept:** `Union Types (|) & Intersection Types (&)`

---

### What is the `never` Type in TypeScript (and Difference Between `void` and `never`)?

> **Direct Answer:**
> - **`never`** represents values that never occur—used as the return type for functions that **never complete** (always throw an exception or execute an infinite loop).
> - **`void`** indicates a function completes execution normally without returning a useful value.
>
> **Key Nuance / Trade-off:**
> - `never` is also used in exhaustive `switch` checks on discriminated unions to guarantee at compile time that every possible case has been handled.
>
> **Reference Concept:** `Primitive Types, any vs unknown, void vs never`

---

### What are Object Type Aliases in TS?

> **Direct Answer:**
> - An Object Type Alias defines a reusable named blueprint describing the required, optional (`?`), and `readonly` properties of an object structure (e.g., `type Smartphone = { readonly brand: string; model: string; is5G?: boolean; }`).
>
> **Key Nuance / Trade-off:**
> - Object type aliases provide DRY reusability, prevent property-name typos at compile time, and are completely erased during compilation to JavaScript.
>
> **Reference Concept:** `Type Aliases vs Interfaces (Optional & Readonly Properties)`

---

### What is a Type Alias in TypeScript?

> **Direct Answer:**
> - A Type Alias (`type Name = ...`) assigns a custom name to any TypeScript type—including object shapes, union types, intersection types, primitive aliases, and function signatures.
>
> **Key Nuance / Trade-off:**
> - Unlike `interface`, a `type` alias can represent unions (`"loading" | "success"`), tuples, and mapped types directly.
>
> **Reference Concept:** `Type Aliases vs Interfaces (Optional & Readonly Properties)`

---

### What is an Interface in TypeScript?

> **Direct Answer:**
> - An `interface` defines a structural contract (shape) specifying the properties, methods, and types an object or implementing `class` must provide.
>
> **Key Nuance / Trade-off:**
> - Interfaces are widely used for API response models, component props, and OOP `class ... implements ...` contracts, and support **declaration merging** across modules.
>
> **Reference Concept:** `Type Aliases vs Interfaces (Optional & Readonly Properties)`

---

### What are Optional Properties in TypeScript?

> **Direct Answer:**
> - Optional properties (marked with `?`, e.g., `department?: string`) indicate that a property may be omitted when constructing an object of that type.
>
> **Key Nuance / Trade-off:**
> - Under `strictNullChecks`, reading an optional property yields `T | undefined`, requiring safe handling (`?.` or null checks) before invoking methods on it.
>
> **Reference Concept:** `Type Aliases vs Interfaces (Optional & Readonly Properties)`

---

### What are Readonly Properties in TypeScript?

> **Direct Answer:**
> - A `readonly` property (e.g., `readonly accountId: number`) can be assigned only during initialization and raises a compile-time error if reassigned later.
>
> **Key Nuance / Trade-off:**
> - `readonly` is a compile-time check only (removed in emitted JS) and is shallow—if a `readonly` property holds an object/array, its inner contents can still be mutated unless wrapped in `Readonly<T>` recursively or frozen at runtime.
>
> **Reference Concept:** `Type Aliases vs Interfaces (Optional & Readonly Properties)`

---

### What is the Difference Between `interface` and `type`?

> **Direct Answer:**
> - Both define object shapes, but **`interface`** supports **declaration merging** and extends via `extends`, whereas **`type`** supports **union types (`|`)**, tuples, primitive aliases, and combines via intersections (`&`).
>
> **Key Nuance / Trade-off:**
> - Prefer `interface` for public library APIs/OOP contracts (where declaration merging is helpful) and `type` for union types, component props, and utility type compositions.
>
> **Reference Concept:** `Type Aliases vs Interfaces (Optional & Readonly Properties)`

---

### What are Utility Types in TS?

> **Direct Answer:**
> - Utility Types are built-in compile-time type transformations that derive new types from existing ones without duplicating definitions—most commonly `Partial<T>`, `Readonly<T>`, `Pick<T, K>`, `Omit<T, K>`, `Record<K, T>`, and `Required<T>`.
>
> **Key Nuance / Trade-off:**
> - `Partial<T>` is ideal for PATCH update payloads, `Omit<T, "password">` strips sensitive fields from DTOs, and `Pick<T, K>` narrows large domain models down for focused UI components.
>
> **Reference Concept:** `Built-in Utility Types (Partial, Readonly, Pick, Omit, Record, Required)`

---

### What are Enums in TypeScript?

> **Direct Answer:**
> - Enums define a named set of constant values (numeric by default starting at `0`, or custom numbers like `HttpStatus.OK = 200`), replacing magic numbers/strings with readable identifiers.
>
> **Key Nuance / Trade-off:**
> - Standard enums emit a JavaScript runtime object (and numeric enums generate bidirectional reverse-mapping lookups), unlike TypeScript union types which compile away completely.
>
> **Reference Concept:** `Enums (Numeric, String, const enum) vs Union Types`

---

### What are String Enums?

> **Direct Answer:**
> - String enums explicitly assign readable string literals to each member (e.g., `enum UserRole { Admin = "ADMIN" }`).
>
> **Key Nuance / Trade-off:**
> - Unlike numeric enums, string enums do **not** generate reverse mappings at runtime, making serialized API payloads and server logs clear and easy to debug.
>
> **Reference Concept:** `Enums (Numeric, String, const enum) vs Union Types`

---

### What are Const Enums (and Enum vs Union Types)?

> **Direct Answer:**
> - A `const enum` is completely erased during compilation—`tsc` inlines the literal values directly at call sites (`Direction.Up` $\to$ `0`) without generating a runtime JS object.
> - **Enum vs. Union Types:** Modern TypeScript codebases often prefer string **Union Types** (`type Status = "success" | "error"`) over standard enums because unions emit zero runtime JavaScript code and integrate cleanly with type narrowing.
>
> **Key Nuance / Trade-off:**
> - Avoid `const enum` when publishing libraries with `isolatedModules: true` (such as Babel/esbuild/Vite setups), as single-file transpilers cannot inline `const enum` values across file boundaries.
>
> **Reference Concept:** `Enums (Numeric, String, const enum) vs Union Types`

---
