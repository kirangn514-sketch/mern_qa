# TypeScript Complete Notes

---

## 1. What is TypeScript?

**Simple definition:** TypeScript is a **superset of JavaScript** that adds **static typing** — meaning you can declare the "shape" and type of your data (strings, numbers, objects, functions) and the compiler checks it **before your code ever runs**, catching bugs early instead of discovering them at runtime.

**Why it exists:** JavaScript is dynamically typed — a variable can hold anything, and type-related bugs (like calling `.toUpperCase()` on a number) only surface when that exact line of code actually executes, often in production. TypeScript catches these mistakes **at compile time**, directly in your editor, before you even run the code.

```ts
function greet(name: string) {
  return `Hello, ${name}`;
}

greet("Alice");   // ✅ OK
greet(42);        // ❌ Compile-time error: Argument of type 'number' is not assignable to type 'string'
```
TypeScript code is **compiled ("transpiled") down to plain JavaScript** before running in the browser or Node.js — types exist only during development, not at runtime.

---

## 2. Types (Basic Type Annotations)

**Simple definition:** A "type" describes **what kind of value** a variable, parameter, or function return can hold. TypeScript lets you explicitly declare these using a colon (`:`) syntax.

```ts
let age: number = 25;
let username: string = "alice";
let isActive: boolean = true;
let scores: number[] = [90, 85, 77];       // array of numbers
let tags: Array<string> = ["a", "b"];      // alternative array syntax
let anything: any = "could be anything";    // opts out of type checking (avoid when possible)
let notSure: unknown = fetchData();         // safer alternative to `any` (see below)
let nothing: void = undefined;              // typically used for functions with no return value
let neverHappens: never;                    // for values that should never occur (e.g., a function that always throws)

// Function types
function add(a: number, b: number): number {
  return a + b;
}

// Object type (inline)
let user: { name: string; age: number } = { name: "Alice", age: 25 };
```

### Type inference
TypeScript is often smart enough to **infer** types automatically, without you writing them explicitly:
```ts
let city = "New York"; // inferred as `string` — no need to write `: string`
```
- **Best practice:** Let TypeScript infer types when it's obvious from context; add explicit annotations for function parameters, return types, and cases where inference isn't clear.

---

## 3. Interfaces

**Simple definition:** An `interface` defines the **shape of an object** — what properties it must have and their types. It's a contract: "any object claiming to be this type must have these fields."

```ts
interface User {
  id: number;
  name: string;
  email: string;
  isAdmin?: boolean; // optional property (the `?` makes it not required)
}

function printUser(user: User) {
  console.log(user.name);
}

printUser({ id: 1, name: "Alice", email: "alice@mail.com" }); // ✅ valid — isAdmin is optional
printUser({ id: 1, name: "Alice" }); // ❌ Error: missing required property 'email'
```

### Interfaces can be extended
```ts
interface Animal {
  name: string;
}

interface Dog extends Animal {
  breed: string;
}

const myDog: Dog = { name: "Rex", breed: "Labrador" };
```

### Interfaces with functions and classes
```ts
interface Shape {
  area(): number; // method signature
}

class Circle implements Shape {
  constructor(private radius: number) {}
  area(): number {
    return Math.PI * this.radius ** 2;
  }
}
```

### Declaration merging (unique to interfaces)
Interfaces with the same name automatically **merge** their properties — useful for extending third-party library types.
```ts
interface Window {
  myCustomProperty: string;
}
interface Window {
  anotherProperty: number;
}
// Window now has BOTH myCustomProperty and anotherProperty
```

---

## 4. Type Alias

**Simple definition:** A `type` alias gives a **name to any type** — not just object shapes, but also primitives, unions, tuples, and functions. Think of it as a reusable label for a type definition.

```ts
type ID = string | number; // alias for a union type

type Point = {
  x: number;
  y: number;
};

type Callback = (data: string) => void; // alias for a function type

type Coordinates = [number, number]; // alias for a tuple

function logId(id: ID) {
  console.log(id);
}
```

### Type alias with objects (looks similar to interface)
```ts
type User = {
  id: number;
  name: string;
};

const user: User = { id: 1, name: "Alice" };
```

### Extending a type alias (via intersection, not `extends`)
```ts
type Animal = { name: string };
type Dog = Animal & { breed: string }; // combining types using `&` (intersection)
```

---

## 5. Interface vs Type Alias — Key Differences

| Feature | `interface` | `type` |
|---|---|---|
| Object shapes | ✅ Yes | ✅ Yes |
| Primitives, unions, tuples | ❌ No | ✅ Yes |
| Can be extended | ✅ `extends` keyword | ✅ via `&` (intersection) |
| Declaration merging (auto-combine same name) | ✅ Yes | ❌ No (causes a duplicate identifier error) |
| Typically used for | Object/class shapes, especially in OOP-style code | Unions, tuples, function types, complex compositions |

**Practical rule of thumb:** Use `interface` for defining object/class shapes (especially public APIs, since they support declaration merging for extensibility). Use `type` when you need unions, intersections, tuples, or more complex type compositions that `interface` can't express. In modern codebases, many teams pick one as a default convention and only reach for the other when its unique capability is needed.

---

## 6. Generics

**Simple definition:** Generics let you write **reusable, type-safe code** that works with multiple types, without losing type information — instead of hardcoding a specific type (or giving up and using `any`), you use a **placeholder type** (like `T`) that gets filled in when the function/class is actually used.

### Without generics (the problem)
```ts
function getFirstItem(arr: any[]): any {
  return arr[0];
}

const num = getFirstItem([1, 2, 3]);      // TypeScript thinks this is `any` — no type safety!
const str = getFirstItem(["a", "b"]);     // also `any` — lost the fact that it's a string
```

### With generics (the fix)
```ts
function getFirstItem<T>(arr: T[]): T {
  return arr[0];
}

const num = getFirstItem<number>([1, 2, 3]); // TypeScript knows this is `number`
const str = getFirstItem(["a", "b"]);        // T is inferred automatically as `string`
```
- `T` is a **type parameter** — a placeholder that becomes whatever type is actually passed in. You can name it anything, but `T`, `K`, `V`, `U` are common conventions.

### Generic interfaces and types
```ts
interface ApiResponse<T> {
  success: boolean;
  data: T;
}

const userResponse: ApiResponse<User> = {
  success: true,
  data: { id: 1, name: "Alice", email: "alice@mail.com" },
};
```

### Generic constraints
Sometimes you want to limit what types `T` can be — e.g., "T must have a `.length` property."
```ts
function logLength<T extends { length: number }>(item: T): void {
  console.log(item.length);
}

logLength("hello");        // ✅ strings have .length
logLength([1, 2, 3]);      // ✅ arrays have .length
logLength(42);             // ❌ Error: number doesn't have .length
```

### Multiple type parameters
```ts
function merge<T, U>(a: T, b: U): T & U {
  return { ...a, ...b };
}

const merged = merge({ name: "Alice" }, { age: 25 }); // { name: string } & { age: number }
```

---

## 7. Utility Types

**Simple definition:** Utility types are **built-in generic types** that transform an existing type into a new one — saving you from manually rewriting variations of the same interface.

### `Partial<T>`
Makes **all properties optional**. Useful for update functions where you only want to change some fields.
```ts
interface User {
  id: number;
  name: string;
  email: string;
}

function updateUser(id: number, updates: Partial<User>) {
  // updates can be { name: "New Name" } without needing id/email
}

updateUser(1, { name: "Bob" }); // ✅ valid — other fields are optional
```

### `Pick<T, Keys>`
Creates a new type by **selecting only specific properties** from an existing type.
```ts
type UserPreview = Pick<User, "id" | "name">;
// equivalent to: { id: number; name: string }

const preview: UserPreview = { id: 1, name: "Alice" };
```

### `Omit<T, Keys>`
The opposite of `Pick` — creates a new type by **excluding specific properties**.
```ts
type UserWithoutEmail = Omit<User, "email">;
// equivalent to: { id: number; name: string }
```

### `Record<Keys, ValueType>`
Creates an object type with a specific set of keys, all mapped to the same value type. Great for dictionaries/maps.
```ts
type Role = "admin" | "editor" | "viewer";

const rolePermissions: Record<Role, string[]> = {
  admin: ["create", "read", "update", "delete"],
  editor: ["create", "read", "update"],
  viewer: ["read"],
};
```

### `Required<T>`
The opposite of `Partial` — makes **all properties required**, even ones that were originally marked optional.
```ts
interface Config {
  host?: string;
  port?: number;
}

const fullConfig: Required<Config> = { host: "localhost", port: 3000 }; // both now mandatory
```

### Quick reference table
| Utility Type | What it does |
|---|---|
| `Partial<T>` | Makes all properties optional |
| `Pick<T, K>` | Keeps only the specified properties |
| `Omit<T, K>` | Removes the specified properties |
| `Record<K, V>` | Builds an object type with keys `K`, all mapped to type `V` |
| `Required<T>` | Makes all properties required |

---

## 8. Enums

**Simple definition:** An `enum` (enumeration) defines a set of **named constants**, making code more readable than using raw strings or numbers scattered throughout your codebase.

```ts
enum Role {
  Admin,
  Editor,
  Viewer,
}

const userRole: Role = Role.Admin;
console.log(userRole); // 0 (numeric enums default to auto-incrementing numbers starting at 0)
```

### String enums (more readable/debuggable than numeric enums)
```ts
enum Status {
  Pending = "PENDING",
  Approved = "APPROVED",
  Rejected = "REJECTED",
}

function handleStatus(status: Status) {
  if (status === Status.Approved) {
    console.log("Approved!");
  }
}

handleStatus(Status.Approved);
```

### Enums vs union of string literals
Many modern TypeScript codebases prefer a **union of string literals** over enums, since it compiles to nothing extra (enums generate actual JS code) and works more predictably with plain objects:
```ts
type Status = "PENDING" | "APPROVED" | "REJECTED";

function handleStatus(status: Status) {
  if (status === "APPROVED") { /* ... */ }
}
```

---

## 9. Union Types

**Simple definition:** A union type means a value can be **one of several specified types**, using the `|` symbol. Think "OR" — this value is a string OR a number.

```ts
function printId(id: string | number) {
  console.log(`ID: ${id}`);
}

printId(101);      // ✅ valid
printId("abc123"); // ✅ valid
printId(true);     // ❌ Error: boolean is not assignable to string | number
```

### Union with literal types (very common for "modes" or "states")
```ts
type Theme = "light" | "dark" | "system";

function setTheme(theme: Theme) {
  // theme can only be one of these 3 exact strings
}

setTheme("light");  // ✅
setTheme("blue");   // ❌ Error: not assignable to type 'Theme'
```

### Narrowing a union type
Since a union value could be multiple types, TypeScript requires you to **check** which one it actually is before using type-specific operations:
```ts
function formatValue(value: string | number) {
  if (typeof value === "string") {
    return value.toUpperCase(); // TypeScript knows it's a string here
  }
  return value.toFixed(2); // TypeScript knows it's a number here
}
```

---

## 10. Intersection Types

**Simple definition:** An intersection type combines **multiple types into one**, using the `&` symbol. Think "AND" — the resulting type must satisfy all combined types at once.

```ts
type Person = { name: string };
type Employee = { employeeId: number };

type StaffMember = Person & Employee; // must have BOTH name AND employeeId

const staff: StaffMember = { name: "Alice", employeeId: 42 };
```

### Union vs Intersection — quick distinction
| | Union (`|`) | Intersection (`&`) |
|---|---|---|
| Meaning | "This OR that" | "This AND that" |
| Value must satisfy | At least one of the types | All of the types combined |
| Common use | Representing multiple possible types/states | Combining/merging multiple type shapes together |

---

## 11. Type Guards

**Simple definition:** A type guard is a **runtime check** that lets TypeScript narrow down a broader type (like a union) into a more specific one within a certain code block — enabling safe access to type-specific properties/methods.

### `typeof` type guard (for primitives)
```ts
function process(value: string | number) {
  if (typeof value === "string") {
    console.log(value.toUpperCase()); // safe — TS knows it's a string here
  } else {
    console.log(value.toFixed(2)); // safe — TS knows it's a number here
  }
}
```

### `instanceof` type guard (for classes)
```ts
class Cat { meow() { console.log("Meow!"); } }
class Dog { bark() { console.log("Woof!"); } }

function makeSound(animal: Cat | Dog) {
  if (animal instanceof Cat) {
    animal.meow();
  } else {
    animal.bark();
  }
}
```

### `in` operator type guard (checking property existence)
```ts
interface Bird { fly(): void; }
interface Fish { swim(): void; }

function move(animal: Bird | Fish) {
  if ("fly" in animal) {
    animal.fly();
  } else {
    animal.swim();
  }
}
```

### Custom type guard functions (using `is`)
```ts
interface Cat { meow(): void; }
interface Dog { bark(): void; }

function isCat(animal: Cat | Dog): animal is Cat {
  return (animal as Cat).meow !== undefined;
}

function makeSound(animal: Cat | Dog) {
  if (isCat(animal)) {
    animal.meow(); // TypeScript now knows it's specifically a Cat
  } else {
    animal.bark();
  }
}
```

---

## 12. Type Assertions

**Simple definition:** A type assertion tells the compiler **"trust me, I know this value's type better than you do"** — it doesn't perform any actual conversion or runtime check, it just overrides TypeScript's own inference. Use with caution.

```ts
// Two equivalent syntaxes:
const input = document.getElementById("username") as HTMLInputElement;
const input2 = <HTMLInputElement>document.getElementById("username"); // not usable in .tsx files

console.log(input.value); // TypeScript now allows .value, since we asserted it's an HTMLInputElement
```

### `as unknown as X` — double assertion (escape hatch, use sparingly)
Used when TypeScript won't allow a direct assertion because the types are too different:
```ts
const value = someValue as unknown as SpecificType;
```

### ⚠️ Danger of misusing type assertions
Type assertions **don't perform actual runtime checks** — if you assert the wrong type, TypeScript won't catch it, and you'll get a runtime error instead:
```ts
const value = "hello" as unknown as number;
console.log(value.toFixed(2)); // compiles fine, but CRASHES at runtime — "hello" isn't actually a number
```
**Best practice:** Prefer type guards (which actually check at runtime) over type assertions (which don't) whenever possible. Only use assertions when you have external knowledge TypeScript can't infer (e.g., you know a DOM element with a specific ID definitely exists and is an `<input>`).

---

## 13. `any` vs `unknown`

**Simple definition:**
- **`any`** completely **disables type checking** for that value — you can do anything with it, and TypeScript won't complain, even if it's wrong. It's like telling the compiler "don't check this at all."
- **`unknown`** is a **type-safe alternative to `any`** — it can hold any value too, but you **must narrow/check its type** before you're allowed to use it in any specific way.

```ts
let valueAny: any = fetchData();
valueAny.toUpperCase(); // ✅ compiles fine, even though this might crash at runtime if it's not a string

let valueUnknown: unknown = fetchData();
valueUnknown.toUpperCase(); // ❌ Compile-time error: Object is of type 'unknown'

// You must narrow it first:
if (typeof valueUnknown === "string") {
  valueUnknown.toUpperCase(); // ✅ now safe — TypeScript knows it's a string
}
```

| | `any` | `unknown` |
|---|---|---|
| Type checking | Disabled entirely | Enforced — must narrow before use |
| Safety | Unsafe — easy to introduce runtime bugs | Safe — forces you to verify the type first |
| When to use | Rarely — legacy code migration, or truly dynamic values you can't type | Preferred when you don't know a value's type upfront (e.g., API responses, `catch` blocks) |

**Best practice:** Avoid `any` wherever possible — it defeats the entire purpose of using TypeScript. Prefer `unknown` for genuinely unknown values, and narrow it with type guards before use.

---

## Quick Summary Cheat Sheet

| Concept | One-liner |
|---|---|
| Types | Basic annotations describing what kind of value something is |
| Interface | Defines the shape of an object/class; supports declaration merging |
| Type Alias | Names any type — objects, unions, tuples, functions |
| Generics | Reusable, type-safe code using placeholder types (`T`) |
| `Partial<T>` | Makes all properties optional |
| `Pick<T, K>` | Selects specific properties from a type |
| `Omit<T, K>` | Excludes specific properties from a type |
| `Record<K, V>` | Builds a dictionary-style object type |
| `Required<T>` | Makes all properties required |
| Enums | Named sets of constants (numeric or string-based) |
| Union Types (`|`) | Value can be one of several types ("OR") |
| Intersection Types (`&`) | Combines multiple types into one ("AND") |
| Type Guards | Runtime checks that narrow a broader type to a specific one |
| Type Assertions | Manually overriding TypeScript's inferred type (no runtime check) |

## Interview Questions & Answers

### Q1. Interface vs Type — when would you use each?
Both can describe object shapes, but they differ in capability: `interface` supports **declaration merging** (multiple declarations with the same name automatically combine — useful for extending third-party types) and is conventionally used for defining object/class contracts. `type` can represent **unions, intersections, tuples, and primitives**, which `interface` cannot. A common convention: use `interface` for public object/class shapes, and `type` for everything else (unions, function types, mapped/complex types).

### Q2. `any` vs `unknown`?
`any` disables type checking entirely — you can call any method or access any property without the compiler complaining, which can hide real bugs until runtime. `unknown` is the type-safe alternative: it can also hold any value, but TypeScript **forces you to narrow/verify the type** (via `typeof`, `instanceof`, or a type guard) before you're allowed to use it. `unknown` gives you the same flexibility as `any` for accepting arbitrary values, while still catching type mistakes at compile time.

### Q3. What are generic functions, and why use them?
A generic function uses a **type parameter** (commonly `T`) as a placeholder for a type that's determined when the function is actually called — allowing one function definition to work correctly and safely across many different types, without resorting to `any` (which would lose all type safety).
```ts
function identity<T>(value: T): T {
  return value;
}

const num = identity(42);      // T inferred as number
const str = identity("hello"); // T inferred as string
```
The key benefit: **type safety is preserved** — the caller gets back the exact type they passed in, and TypeScript can catch mistakes (e.g., trying to call `.toUpperCase()` on a number returned from a generic function).

### Q4. Why use TypeScript instead of plain JavaScript?
1. **Catches bugs at compile time** — type mismatches, typos in property names, and incorrect function arguments are caught before the code ever runs, rather than surfacing as runtime crashes in production.
2. **Better developer experience** — editors can provide accurate autocomplete, inline documentation, and safe refactoring (e.g., renaming a property updates every usage) because the tooling understands your data shapes.
3. **Self-documenting code** — types act as always-up-to-date documentation for what a function expects and returns, reducing the need to dig through implementation details or guess from usage.
4. **Safer refactoring at scale** — in large codebases with many contributors, changing a shared interface immediately flags every place that needs to be updated, rather than relying on manual searching or waiting for a runtime error to reveal a missed spot.
5. **Gradual adoption** — TypeScript is a superset of JavaScript, so you can adopt it incrementally in an existing JS codebase rather than doing a full rewrite.

**Trade-off to mention:** TypeScript adds a compilation step and a learning curve, and can feel like extra ceremony for small scripts/prototypes — its benefits scale with codebase size and team size, so it's less critical for a quick one-off script.
