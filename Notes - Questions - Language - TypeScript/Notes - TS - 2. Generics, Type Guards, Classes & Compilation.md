# Notes - TS - 2. Generics, Type Guards, Classes & Compilation

## 1. Generics

### Generics: Functions, Interfaces, Constraints & Classes

**Generics** allow writing reusable, flexible components that operate over multiple types while preserving full compile-time type safety using type parameter placeholders (`<T>`).

#### 1. Generic Functions:
```typescript
function getFirstElement<T>(arr: T[]): T {
    return arr[0];
}
getFirstElement<number>([1, 2, 3]);
getFirstElement<string>(["A", "B", "C"]);
```

#### 2. Generic Interfaces:
```typescript
interface ApiResponse<T> {
    data: T;
    status: number;
}
const userRes: ApiResponse<string> = { data: "User Data", status: 200 };
```

#### 3. Generic Constraints (`extends`):
Restricts the allowed types for `<T>` so the implementation can safely access required properties:

```typescript
function printLength<T extends { length: number }>(value: T): void {
    console.log(value.length);
}
printLength("TypeScript"); // Valid
printLength([1, 2, 3]);    // Valid
// printLength(10);        // Compile Error: number lacks 'length'
```

#### 4. Generic Classes:
```typescript
class Box<T> {
    constructor(public value: T) {}
    getValue(): T {
        return this.value;
    }
}
const numBox = new Box<number>(100);
```

---

## 2. Modules vs Namespaces

### ES Modules (import/export) vs Legacy Namespaces

- **Modules:** Any file containing at least one top-level `import` or `export` is an isolated module (based on the ES Module standard), preventing global scope pollution.
  - **Named Exports:** Multiple exports per file (`export function add(...) {}`), imported using matching names (`import { add } from "./math-utils"`).
  - **Default Exports:** One primary export per file (`export default function multiply(...) {}`), imported with any chosen identifier (`import multiply from "./math-utils"`).

| Feature | Namespace (Legacy) | Modules (Modern Standard) |
| :--- | :--- | :--- |
| **Scope** | Wraps code into a single global object | File-based lexical isolation |
| **Syntax** | `namespace MathUtils {}` | `import` / `export` |
| **Recommendation** | Legacy / Avoid in modern TS | **Recommended standard** |

---

## 3. Type Narrowing & Type Guards

### Type Narrowing & Type Guards (typeof, instanceof, in, Custom Predicates)

**Type Narrowing** is the process by which TypeScript reduces a broader union type (e.g., `string | number`) down to a specific type inside conditional blocks using **Type Guards**:

| Type Guard | Best Used For | Example |
| :--- | :--- | :--- |
| `typeof` | Primitive types (`string`, `number`, `boolean`, etc.) | `typeof x === "string"` |
| `instanceof` | Class instances & OOP hierarchies | `animal instanceof Dog` |
| `in` operator | Checking property existence on object unions | `"swim" in animal` |
| **Custom Type Predicate** | Complex validation / API payload checks | `function isAdmin(p: Admin \| User): p is Admin` |

```typescript
// Custom User-Defined Type Guard (Type Predicate)
type Admin = { name: string; role: "admin" };
type User = { name: string; role: "user" };

function isAdmin(person: Admin | User): person is Admin {
    return person.role === "admin";
}
```

---

## 4. Decorators & Object-Oriented Classes

### Decorators (@expression) & Metadata

A **Decorator** is a special declaration (`@expression`) attached to a class, method, accessor, property, or parameter that wraps or attaches metadata to separate cross-cutting concerns (logging, dependency injection, validation, routing in Angular `@Component` or NestJS `@Get`).

- **Configuration:** Legacy TypeScript decorators require `"experimentalDecorators": true` in `tsconfig.json`. TypeScript 5.0+ also supports the official Stage 3 ECMAScript standard decorators.

| Decorator Type | Target / Arguments Received | Typical Use Case |
| :--- | :--- | :--- |
| **Class Decorator** | Class constructor function | Registering components (`@Component`), sealing classes |
| **Method Decorator** | Prototype, method name, `PropertyDescriptor` | Logging, execution timing, `@ReadOnly`, authorization checks |
| **Property Decorator** | Prototype, property key | Validation rules, dependency injection |
| **Accessor Decorator** | Getter / Setter descriptor | Controlling access or memoization |

---

### TypeScript Classes & Access Modifiers (public, private, protected)

TypeScript extends ES6 classes with compile-time static typing, interface `implements`, abstract classes, parameter properties, and access modifiers:

| Modifier | Visibility Scope |
| :--- | :--- |
| `public` | Accessible everywhere (**default** if omitted) |
| `private` | Accessible **only inside** the declaring class |
| `protected` | Accessible inside the declaring class **and subclasses** |

```typescript
class BankAccount {
    private balance: number;
    constructor(balance: number) {
        this.balance = balance;
    }
    public getBalance(): number {
        return this.balance;
    }
}
```

---

## 5. Compilation & JavaScript Interoperability

### TypeScript Compilation Pipeline (tsc) & Config Files (tsconfig.json vs package.json)

The TypeScript Compiler (`tsc`) transpiles `.ts` files into JavaScript in **three phases**:
1. **Parsing:** Builds an Abstract Syntax Tree (AST) from `.ts` source code.
2. **Type Checking:** Validates interfaces, type aliases, generics, and type guards across the AST.
3. **Emitting:** Performs **Type Erasure** (strips all TS types) and emits `.js` (plus optional `.d.ts` and `.js.map` files).

| Aspect | `tsconfig.json` (Compiler Brain) | `package.json` (Project Backbone) |
| :--- | :--- | :--- |
| **Primary Role** | Configures `tsc` compiler behavior | Configures Node.js / npm dependencies and scripts |
| **Key Settings** | `target`, `module`, `strict`, `outDir`, `include`/`exclude` | `name`, `version`, `scripts`, `dependencies`, `main` |

---

### TypeScript-JavaScript Interoperability (allowJs, checkJs, .d.ts)

Because TS is a superset of JS, legacy JavaScript projects can migrate **incrementally**:
- **`allowJs: true`:** Allows `.js` files to live alongside `.ts` files and be compiled by `tsc`.
- **`checkJs: true`:** Enables type checking inside `.js` files (using JSDoc comments like `/** @param {number} a */`).
- **Declaration Files (`.d.ts`):** Describe the static types of external JavaScript libraries without implementation (either bundled with packages or installed via `@types/*` from DefinitelyTyped).
- **Direct JS Module Imports:** TypeScript can import `.js` modules directly, inferring types or reading companion `.d.ts` files.

```json
{
  "compilerOptions": {
    "allowJs": true,
    "checkJs": true
  }
}
```

---
