# Notes - JS - 1. Fundamentals, Types & Coercion

## 1. Variables & Scoping

### Variable Declarations: var, let, and const

JavaScript provides three variable declaration keywords differing in scope, redeclaration rules, reassignment, and hoisting behavior:

| Feature | `var` | `let` | `const` |
| :--- | :--- | :--- | :--- |
| **Scope** | Function scoped | Block scoped (`{}`) | Block scoped (`{}`) |
| **Redeclaration** | Allowed | Not allowed | Not allowed |
| **Reassignment** | Allowed | Allowed | Not allowed |
| **Hoisting** | Hoisted and initialized with `undefined` | Hoisted but uninitialized (TDZ) | Hoisted but uninitialized (TDZ) |
| **Recommended Usage** | Avoid in modern JS | Use for mutable bindings | Use by default for constants |

```javascript
var a = 10;
var a = 20; // Redeclaration allowed

let b = 30;
b = 40;     // Reassignment allowed

const c = 50;
// c = 60;  // TypeError: Assignment to constant variable
```

#### Why `var` is Avoided in Modern JavaScript

| Problem | Explanation |
| :--- | :--- |
| **No block scope** | Variables leak outside `if` or `for` `{}` blocks into the enclosing function/global scope |
| **Redeclaration allowed** | Can silently overwrite existing variables with the same name |
| **Hoisting confusion** | Accessing before declaration returns `undefined` instead of throwing an error |
| **Loop closure issues** | Shared function-scoped loop variable causes classic closure bugs in asynchronous callbacks |

```javascript
if (true) {
    var x = 100;
}
console.log(x); // 100 (leaks outside block)

if (true) {
    let y = 200;
}
// console.log(y); // ReferenceError: y is not defined
```

#### Modifying `const` Objects
- `const` prevents **reassignment of the variable binding**, not mutation of the underlying object's properties in memory.

| Operation on `const` Object | Allowed? |
| :--- | :--- |
| Modify property (`user.age = 26`) | Yes |
| Add property (`user.city = "Mumbai"`) | Yes |
| Delete property (`delete user.age`) | Yes |
| Reassign variable (`user = {}`) | No (`TypeError`) |

```javascript
const user = { name: "Vinayak", age: 25 };
user.age = 26; // Allowed
// user = {};  // TypeError
```

---

### Hoisting & Temporal Dead Zone (TDZ)

- **Hoisting:** A JavaScript mechanism where variable and function declarations are registered at the top of their enclosing scope during the compilation/creation phase prior to code execution.
  - Only **declarations** are hoisted, **not initializations**.
  - `var` declarations are hoisted and automatically initialized with `undefined`.
  - `let` and `const` declarations are hoisted but remain uninitialized inside the **Temporal Dead Zone (TDZ)**.

```javascript
console.log(a); // undefined
var a = 10;

// Internally interpreted as:
var a;
console.log(a);
a = 10;
```

- **Temporal Dead Zone (TDZ):** The execution span between entering a block scope (where `let`/`const` is hoisted) and the actual declaration line being evaluated. Accessing a `let` or `const` variable inside the TDZ throws a `ReferenceError`.

```javascript
console.log(x); // ReferenceError: Cannot access 'x' before initialization
let x = 10;
```

---

## 2. Data Types & Typing Model

### JavaScript Data Types (Primitive vs Reference)

JavaScript has **8 data types**, divided into **7 primitive types** and **1 reference (object) type**:

| Category | Types |
| :--- | :--- |
| **Primitive Types (7)** | `string`, `number`, `boolean`, `null`, `undefined`, `symbol`, `bigint` |
| **Reference Type (1)** | `object` (includes plain objects, arrays, functions, dates, etc.) |

#### Primitive Data Types Detail:

| Primitive Type | Description | Example |
| :--- | :--- | :--- |
| `string` | Immutable textual data | `"Hello"` |
| `number` | 64-bit IEEE-754 floating-point numbers | `10`, `3.14` |
| `boolean` | Logical values | `true`, `false` |
| `null` | Intentional absence of any object value | `null` |
| `undefined` | Variable declared but not assigned a value | `undefined` |
| `symbol` | Unique, immutable identifier (ES6) | `Symbol("id")` |
| `bigint` | Arbitrary-precision integers beyond `Number.MAX_SAFE_INTEGER` | `9007199254740991n` |

#### Primitive vs Reference Types:

| Feature | Primitive Types | Reference Types |
| :--- | :--- | :--- |
| **Stored As** | Actual value directly in stack/variable | Memory reference (pointer) to heap object |
| **Mutability** | Immutable | Mutable |
| **Comparison** | Compared by value | Compared by reference |
| **Examples** | `number`, `string`, `boolean` | `object`, `array`, `function` |

```javascript
// Primitive copy (by value)
let a = 10;
let b = a;
b = 20;
console.log(a); // 10
console.log(b); // 20

// Reference copy (by memory address)
let obj1 = { name: "Vinayak" };
let obj2 = obj1;
obj2.name = "Rahul";
console.log(obj1.name); // "Rahul"
console.log(obj2.name); // "Rahul"
```

---

### The typeof null Historical Bug & Dynamic Typing

- **Why `typeof null === "object"`:**
  - In JavaScript's original 1995 implementation, values were stored with a 32-bit type tag in the lower bits. Objects carried a type tag of `0` (`000`).
  - `null` was represented as the NULL pointer (`0x00`—all zero bits). Consequently, `typeof` inspected the `0` tag and erroneously reported `"object"`.
  - Despite being a primitive, this behavior was preserved permanently for backward compatibility across the web.

```javascript
console.log(typeof null); // "object"
```

- **Dynamic Typing:** JavaScript is **dynamically typed**—variable types are resolved at runtime, type annotations are not required, and the same variable can hold different types during execution.

| Language | Typing Discipline |
| :--- | :--- |
| Java, C++ | Statically typed |
| JavaScript, Python | Dynamically typed |

```javascript
let value = 10;
console.log(typeof value); // "number"
value = "JavaScript";
console.log(typeof value); // "string"
```

*(Tools like TypeScript add static type checking on top of JavaScript at compile time).*

---

## 3. Type Coercion & Equality

### Type Coercion Rules (Implicit vs Explicit)

**Type coercion** is the conversion of a value from one data type to another during operations or comparisons.

| Type | Description | Performed By | Example |
| :--- | :--- | :--- | :--- |
| **Implicit Coercion** | Automatic conversion during mixed-type operations | JavaScript engine | `"5" + 2` $\to$ `"52"`, `"10" * 2` $\to$ `20` |
| **Explicit Coercion** | Intentional manual conversion using built-in functions | Developer | `Number("100")`, `Boolean(val)`, `String(123)` |

#### 1. Boolean Coercion (Truthy vs. Falsy)
- **Falsy values (6 core values, plus `-0` and `0n`):** `false`, `0` (`-0`, `0n`), `""` (empty string), `null`, `undefined`, `NaN`.
- **Truthy values:** Everything else—including `"0"`, `"false"`, `" "` (whitespace), `[]` (empty array), `{}` (empty object), functions, and BigInts like `10n`.

```javascript
if ([]) {
    console.log("Empty arrays are truthy!"); // Executes
}
console.log(Boolean("0")); // true
console.log(Boolean({}));  // true
```

#### 2. Numeric Coercion (`ToNumber` Rule)
Mathematical operators (`-`, `*`, `/`, `>`)—except `+`—convert operands to numbers:

| Value | Converted Number |
| :--- | :--- |
| `null` | `0` |
| `undefined` | `NaN` |
| `true` / `false` | `1` / `0` |
| `"42"` / `"abc"` | `42` / `NaN` |

```javascript
console.log("10" - 2); // 8
console.log(true + 5); // 6
console.log(null * 3); // 0
```

#### 3. The Addition (`+`) Exception
If either operand of `+` is a `string`, JavaScript converts the other operand to a `string` and concatenates them. Otherwise, it converts both to numbers and adds them.

```javascript
console.log(5 + "5");     // "55"
console.log(5 + 5 + "5"); // "105" (left-to-right: 10 + "5")
console.log("5" + 5 + 5); // "555" (left-to-right: "55" + 5)
```

#### 4. Object-to-Primitive Coercion (`ToPrimitive`)
When an object or array is used where a primitive is expected (e.g., `obj + 1`):
1. Calls `valueOf()`. If it returns a primitive, uses it.
2. Otherwise calls `toString()`. Arrays convert to comma-joined strings (`[]` $\to$ `""`).
3. If neither yields a primitive, throws `TypeError`.

```javascript
console.log([1, 2] + [3, 4]); // "1,23,4" ("1,2" + "3,4")
```

---

### Loose Equality (==) vs Strict Equality (===)

| Feature | `==` (Loose Equality) | `===` (Strict Equality) |
| :--- | :--- | :--- |
| **Type Conversion** | Yes (performs implicit coercion) | No (disables coercion) |
| **Comparison** | Compares value after coercing to common type | Checks **both value AND type** |
| **Performance** | Slightly slower due to coercion steps | Slightly faster |
| **Predictability** | Low (prone to counterintuitive bugs) | High (consistent and safe) |

```javascript
console.log(5 == "5");         // true
console.log(1 == true);        // true
console.log(null == undefined);// true (special spec rule)

console.log(5 === "5");        // false
console.log(0 === false);      // false
console.log(null === undefined);// false
```

#### Famous Strict Equality Exceptions (`NaN` and `-0`):
- `NaN === NaN` evaluates to `false` (use `Number.isNaN(x)` to check for `NaN`).
- `+0 === -0` evaluates to `true`.
- Use `Object.is(a, b)` when exact SameValue equality (distinguishing `-0` and treating `NaN` as equal to `NaN`) is required.

---
