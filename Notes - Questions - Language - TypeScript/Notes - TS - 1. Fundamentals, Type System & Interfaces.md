# Notes - TS - 1. Fundamentals, Type System & Interfaces

## 1. TypeScript Foundations & Build Output

### TypeScript Overview, Benefits & Large-Scale Adoption

**TypeScript** is a strongly typed **superset of JavaScript** that adds static typing, compile-time type checking, interfaces, generics, and modern language features on top of JavaScript.

- TypeScript is **transpiled** by the TypeScript Compiler (`tsc`) into plain JavaScript that executes in any JS runtime (Browsers, Node.js, Deno).
- Because TS is a strict superset, **all valid JavaScript code is also valid TypeScript code**.

#### JavaScript vs TypeScript:

| Feature | JavaScript | TypeScript |
| :--- | :--- | :--- |
| **Typing** | Dynamic | **Static** |
| **Execution** | Interpreted directly | Compiled (transpiled) to JS |
| **Error Detection** | Runtime | **Compile time** |
| **IDE Tooling** | Limited | Strong IntelliSense, autocomplete, safe refactoring |
| **Interfaces & Generics** | Not supported | Supported |

#### Why TypeScript is Essential in Large Applications:
- **Compile-time Bug Detection:** Catches silent JS coercion bugs (e.g., `add(5, "10")` returning `"510"`) and API contract mismatches (`Property 'name' does not exist`) before code runs.
- **Self-Documenting Contracts:** Interfaces and types document module boundaries clearly across large multi-developer teams.
- **Safe Refactoring:** Renaming fields or changing function signatures flags every broken call site project-wide.

---

### Compiled Output Artifacts (dist/ Folder)

When `tsc` compiles `src/fundamentals.ts`, it emits specialized artifacts into `dist/`:

```text
├── package.json
├── tsconfig.json
├── dist/
│   ├── fundamentals.js
│   ├── fundamentals.js.map
│   ├── fundamentals.d.ts
│   └── fundamentals.d.ts.map
└── src/
    └── fundamentals.ts
```

| File Extension | Content Type | Primary Use Case |
| :--- | :--- | :--- |
| `.js` | **Production Executable Code** | Stripped of all TS type annotations so browsers/Node.js can execute it |
| `.d.ts` | **Declaration File (Type Metadata)** | Blueprint describing function/class/variable shapes for library consumers (IntelliSense & type checking) |
| `.js.map` | **Source Map (Debugging Data)** | Maps runtime errors in `.js` back to the exact line/column in original `.ts` |
| `.d.ts.map` | **Declaration Source Map** | Enables "Go to Definition" in IDEs to jump directly to `.ts` implementation rather than `.d.ts` |

---

## 2. Type Annotations, Inference & Core Types

### Type Annotations vs Type Inference

- **Type Annotation:** Explicitly specifying the type of a variable, parameter, or return value using `: Type`.
  - Preferred when the type cannot be inferred, when declaring uninitialized variables (`let score: number;`), or when enforcing explicit function contracts.
- **Type Inference:** TypeScript automatically deduces the type from the assigned value (`let age = 25;` $\to$ `number`, `let numbers = [1, 2, 3];` $\to$ `number[]`).

```typescript
// Explicit Type Annotations
function add(a: number, b: number): number {
    return a + b;
}

// Type Inference
let username = "Vinayak"; // inferred as string
function multiply(a: number, b: number) {
    return a * b; // return type inferred as number
}
```

---

### Primitive Types, any vs unknown, void vs never

#### Primitive Types (Immutable):
`string`, `number`, `boolean`, `null`, `undefined`, `symbol`, `bigint`.

#### `any` vs `unknown`:

| Feature | `any` | `unknown` |
| :--- | :--- | :--- |
| **Type Safety** | **No** (opts out of type checking) | **Yes** (type-safe top type) |
| **Compiler Checks** | Disabled | Required |
| **Allowed Operations** | Any operation allowed (unsafe) | **Must narrow type first** before using |
| **Recommendation** | Avoid | **Preferred** for dynamic inputs |

```typescript
let safeVal: unknown = "Hello";
// safeVal.toUpperCase(); // Compile Error!
if (typeof safeVal === "string") {
    console.log(safeVal.toUpperCase()); // Safe after narrowing
}
```

#### `void` vs `never`:

| Feature | `void` | `never` |
| :--- | :--- | :--- |
| **Meaning** | Function returns no useful value | Function **never completes / never returns** |
| **Execution** | Finishes execution normally | Throws an exception or runs an infinite loop |
| **Example** | `function log(msg: string): void` | `function fail(msg: string): never { throw new Error(msg); }` |

---

### Union Types (|) & Intersection Types (&)

- **Union Types (`|`):** Allows a value to hold one of several possible types (`let val: string | number`). Requires type narrowing (`typeof`, `instanceof`, etc.) before type-specific method calls.
- **Intersection Types (`&`):** Combines multiple types into a single composite type requiring all properties from every constituent type.

```typescript
type Person = { name: string };
type Employee = { employeeId: number };
type Staff = Person & Employee;

const staff: Staff = { name: "Vinayak", employeeId: 101 };
```

---

## 3. Type Aliases, Interfaces, Utility Types & Enums

### Type Aliases vs Interfaces (Optional & Readonly Properties)

- **Type Alias (`type`):** Assigns a custom name to object shapes, primitives, unions, intersections, or function signatures. Disappears completely at runtime after compilation.
- **Interface (`interface`):** Defines structural object contracts, API responses, and class `implements` contracts.
- **Modifiers:**
  - **Optional (`?`):** Property may be omitted (`department?: string`).
  - **Readonly (`readonly`):** Cannot be reassigned after object initialization (`readonly accountId: number`).

| Feature | `interface` | `type` |
| :--- | :--- | :--- |
| **Object definitions** | Yes | Yes |
| **Union types (`\|`)** | No | **Yes** |
| **Intersection / Extending** | `extends` | `&` |
| **Declaration merging** | **Yes** (multiple declarations with same name merge) | No |

```typescript
type Smartphone = {
    readonly brand: string;
    model: string;
    storageGB: number;
    is5G?: boolean;
};

interface Person { name: string; }
interface Developer extends Person { language: string; }
```

---

### Built-in Utility Types (Partial, Readonly, Pick, Omit, Record, Required)

Built-in Utility Types transform existing types following the DRY principle:

| Utility Type | Transformation | Best Used For |
| :--- | :--- | :--- |
| `Partial<T>` | Makes all properties optional (`?`) | PATCH / update payload forms |
| `Required<T>` | Makes all properties mandatory | Enforcing complete config objects |
| `Readonly<T>` | Marks all properties `readonly` | Immutable configurations and constants |
| `Pick<T, K>` | Includes **only** specified keys `K` | Narrowing large DTOs for UI previews |
| `Omit<T, K>` | Excludes specified keys `K` | Stripping sensitive fields (e.g., `password`) |
| `Record<K, T>` | Maps key set `K` to value type `T` | Dictionaries and lookup maps |

```typescript
type User = { id: number; name: string; email: string };
const update: Partial<User> = { email: "new@email.com" };
type UserPreview = Pick<User, "id" | "name">;
```

---

### Enums (Numeric, String, const enum) vs Union Types

- **Numeric Enums:** Auto-increment from `0` by default (supports forward and reverse mapping at runtime).
- **String Enums:** Explicit string values (`enum UserRole { Admin = "ADMIN" }`). No reverse mapping; preferred for readable logs and JSON APIs.
- **`const enum`:** Completely **inlined and erased** during compilation—generates zero runtime JavaScript object overhead (`let move = Direction.Up` compiles to `let move = 0`).

#### Enum vs Union Types:

| Feature | `enum` | Union Type (`"success" \| "error"`) |
| :--- | :--- | :--- |
| **Runtime Object** | Yes (unless `const enum`) | **No** (erased at compile time) |
| **Compile Output** | Emits JS code | Zero JS output |
| **Flexibility & Narrowing** | Less flexible | **More flexible**, modern TS preference |

---
