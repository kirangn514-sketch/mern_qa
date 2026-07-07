# JavaScript (ES6+) — Complete Interview Notes

---

## 1. `var`, `let`, `const`

**Simple definition:** These are the three ways to declare a variable in JavaScript. They differ in **scope**, **hoisting behavior**, and whether they can be **reassigned**.

| Feature | `var` | `let` | `const` |
|---|---|---|---|
| Scope | Function-scoped | Block-scoped | Block-scoped |
| Hoisting | Hoisted & initialized as `undefined` | Hoisted but NOT initialized (Temporal Dead Zone) | Hoisted but NOT initialized (Temporal Dead Zone) |
| Re-declaration | Allowed | Not allowed in same scope | Not allowed in same scope |
| Re-assignment | Allowed | Allowed | Not allowed |
| Attached to `window`/global object | Yes | No | No |

```javascript
var a = 10;   // function-scoped, can be redeclared
let b = 20;   // block-scoped, can be reassigned
const c = 30; // block-scoped, cannot be reassigned

if (true) {
  var a = 100; // same 'a' — leaks outside the block
  let b = 200; // new 'b' — only exists inside this block
}
console.log(a); // 100
console.log(b); // 20
```

**Key point:** `const` doesn't make objects immutable — it only prevents reassigning the variable itself.

```javascript
const obj = { name: "Alex" };
obj.name = "John"; // ✅ allowed — mutating the object
obj = {};           // ❌ error — reassigning the variable
```

---

## 2. Scope

**Simple definition:** Scope determines **where** a variable is accessible in your code.

Types of scope:
1. **Global Scope** — declared outside any function/block; accessible everywhere.
2. **Function Scope** — variables declared inside a function are accessible only within it (`var` follows this).
3. **Block Scope** — variables declared inside `{ }` (like `if`, `for`) are accessible only within that block (`let`/`const` follow this).
4. **Lexical Scope** — a nested function has access to variables defined in its outer (parent) function, based on where it's physically written in the code.

```javascript
function outer() {
  let x = 10;
  function inner() {
    console.log(x); // lexical scope: inner can access outer's x
  }
  inner();
}
```

---

## 3. Hoisting

**Simple definition:** Hoisting is JavaScript's behavior of moving **declarations** (not initializations) to the top of their scope before code execution.

```javascript
console.log(x); // undefined (not an error)
var x = 5;

console.log(y); // ❌ ReferenceError: Cannot access 'y' before initialization
let y = 10;
```

- `var` declarations are hoisted and initialized with `undefined`.
- `let` and `const` are hoisted but stay in the **Temporal Dead Zone (TDZ)** — the period between entering scope and the actual declaration line, where accessing them throws an error.
- Function declarations are fully hoisted (you can call them before they appear in code).
- Function expressions/arrow functions are NOT hoisted the same way (only the variable is hoisted, not the function body).

```javascript
sayHi(); // ✅ Works — function declarations are fully hoisted
function sayHi() { console.log("Hi"); }

sayBye(); // ❌ TypeError: sayBye is not a function
var sayBye = function() { console.log("Bye"); };
```

---

## 4. Closures

**Simple definition:** A closure is when a function **"remembers"** the variables from its outer (enclosing) scope even after that outer function has finished executing.

```javascript
function counter() {
  let count = 0;
  return function () {
    count++;
    return count;
  };
}

const increment = counter();
console.log(increment()); // 1
console.log(increment()); // 2
console.log(increment()); // 3
```

Here, the returned inner function keeps access to `count` even though `counter()` has already returned. This happens because the inner function forms a **closure** over its lexical environment.

**Real-world uses:** data privacy/encapsulation, function factories, memoization, event handlers, `debounce`/`throttle` implementations.

---

## 5. Event Loop

**Simple definition:** The Event Loop is the mechanism that allows JavaScript (which is single-threaded) to handle asynchronous operations like timers, network calls, and events without blocking the main thread.

**How it works (step by step):**
1. Synchronous code runs first, on the **Call Stack**.
2. Async operations (like `setTimeout`, `fetch`, promises) are handed off to the browser/Node APIs.
3. When those operations finish, their callbacks go into a **queue** (Microtask Queue or Macrotask Queue).
4. The **Event Loop** constantly checks: "Is the Call Stack empty?" If yes, it pushes the next queued task onto the stack to execute.
5. **Microtasks (like Promises) always run before Macrotasks (like setTimeout).**

```javascript
console.log("1");

setTimeout(() => console.log("2"), 0);

Promise.resolve().then(() => console.log("3"));

console.log("4");

// Output: 1, 4, 3, 2
```

**Why this order?** `1` and `4` run synchronously first. Then the microtask queue (`3`) is drained before the macrotask queue (`2`) gets a turn.

---

## 6. Call Stack

**Simple definition:** The Call Stack is a data structure (LIFO — Last In, First Out) that keeps track of function calls currently being executed.

```javascript
function a() { b(); }
function b() { c(); }
function c() { console.log("Hi"); }
a();

// Call Stack grows: a() → b() → c()
// c() finishes and pops off → then b() → then a()
```

- When a function is called, it's **pushed** onto the stack.
- When it returns, it's **popped** off.
- If functions call each other infinitely (like unchecked recursion), you get a **"Stack Overflow"** error.
- JavaScript can only do ONE thing at a time on the call stack — this is why JS is called **single-threaded**.

---

## 7. Microtasks vs Macrotasks

**Simple definition:** Both are queues for asynchronous callbacks, but they have different priority levels in the Event Loop.

| | Microtasks | Macrotasks |
|---|---|---|
| Examples | Promises (`.then`, `.catch`, `.finally`), `async/await`, `queueMicrotask()`, `MutationObserver` | `setTimeout`, `setInterval`, `setImmediate`, I/O, UI rendering |
| Priority | Higher — runs first | Lower — runs after |
| When executed | Immediately after current synchronous code finishes, before rendering/next macrotask | One at a time, after the microtask queue is fully empty |

**Golden Rule:** After every single macrotask, the Event Loop drains the ENTIRE microtask queue before moving to the next macrotask.

```javascript
setTimeout(() => console.log("macrotask"), 0);
Promise.resolve().then(() => console.log("microtask"));
// Output: microtask → macrotask
```

---

## 8. Promises

**Simple definition:** A Promise is an object representing the **eventual completion (or failure)** of an asynchronous operation. It's a cleaner alternative to callback-based async code (avoids "callback hell").

A Promise has 3 states:
- **Pending** — initial state, operation not finished yet.
- **Fulfilled** — operation completed successfully.
- **Rejected** — operation failed.

```javascript
const myPromise = new Promise((resolve, reject) => {
  let success = true;
  setTimeout(() => {
    if (success) resolve("Data fetched!");
    else reject("Error fetching data");
  }, 1000);
});

myPromise
  .then((result) => console.log(result))
  .catch((error) => console.log(error))
  .finally(() => console.log("Done"));
```

**Useful static methods:**
- `Promise.all([p1, p2])` — waits for all to resolve; rejects immediately if any one fails.
- `Promise.allSettled([p1, p2])` — waits for all, regardless of success/failure.
- `Promise.race([p1, p2])` — resolves/rejects as soon as the FIRST promise settles.
- `Promise.any([p1, p2])` — resolves as soon as the first one succeeds; ignores rejections unless all fail.

---

## 9. async/await

**Simple definition:** `async/await` is **syntactic sugar** built on top of Promises that lets you write asynchronous code that *looks* synchronous, making it easier to read.

```javascript
function getData() {
  return new Promise((resolve) => setTimeout(() => resolve("Data"), 1000));
}

async function fetchData() {
  console.log("Start");
  const result = await getData(); // pauses here until Promise resolves
  console.log(result);
  console.log("End");
}

fetchData();
```

- An `async` function always returns a Promise.
- `await` pauses execution of that function (without blocking the rest of the program) until the awaited Promise settles.
- Error handling is done using `try/catch` instead of `.catch()`.

```javascript
async function fetchData() {
  try {
    const result = await getData();
    console.log(result);
  } catch (error) {
    console.log("Error:", error);
  }
}
```

---

## 10. Prototype & Inheritance

**Simple definition:** Every JavaScript object has an internal hidden link to another object called its **prototype**. If a property/method isn't found on the object itself, JS looks up the **prototype chain** to find it. This is how inheritance works in JS (it's **prototype-based**, not classical class-based like Java/C++).

```javascript
function Animal(name) {
  this.name = name;
}
Animal.prototype.speak = function () {
  console.log(`${this.name} makes a sound.`);
};

const dog = new Animal("Dog");
dog.speak(); // "Dog makes a sound." — found via prototype chain
```

**ES6 `class` syntax** (syntactic sugar over prototypes):

```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }
  speak() {
    console.log(`${this.name} makes a sound.`);
  }
}

class Dog extends Animal {
  speak() {
    console.log(`${this.name} barks.`);
  }
}

const d = new Dog("Rex");
d.speak(); // "Rex barks."
```

`extends` sets up the prototype chain automatically; `super()` calls the parent constructor/methods.

---

## 11. `this`

**Simple definition:** `this` refers to the object that is currently **executing/calling** the function. Its value is determined by **how a function is called**, not where it's defined (except for arrow functions).

| Context | Value of `this` |
|---|---|
| Global scope (non-strict) | `window` (browser) / `global` (Node) |
| Inside a regular function (non-strict) | `window`/`global` (or `undefined` in strict mode) |
| Inside an object method | The object the method is called on |
| Inside a constructor function/class | The newly created instance |
| Arrow function | Inherits `this` from its surrounding (lexical) scope |
| Explicit binding (`call`/`apply`/`bind`) | Whatever object you specify |

```javascript
const obj = {
  name: "Alice",
  regularFn: function () { console.log(this.name); },
  arrowFn: () => { console.log(this.name); }
};

obj.regularFn(); // "Alice" — this = obj
obj.arrowFn();    // undefined — this = outer/lexical scope (not obj)
```

---

## 12. `call()`, `apply()`, `bind()`

**Simple definition:** These three methods let you explicitly control what `this` refers to inside a function.

```javascript
const person = { name: "John" };

function greet(greeting, punctuation) {
  console.log(`${greeting}, ${this.name}${punctuation}`);
}

// call(): invokes immediately, arguments passed individually
greet.call(person, "Hello", "!"); // "Hello, John!"

// apply(): invokes immediately, arguments passed as an array
greet.apply(person, ["Hi", "?"]); // "Hi, John?"

// bind(): returns a NEW function with 'this' permanently bound (doesn't invoke immediately)
const boundGreet = greet.bind(person, "Hey");
boundGreet("."); // "Hey, John."
```

| Method | Invokes immediately? | Arguments format |
|---|---|---|
| `call()` | Yes | Comma-separated list |
| `apply()` | Yes | Array |
| `bind()` | No — returns new function | Comma-separated list |

---

## 13. Arrow Functions

**Simple definition:** A shorter syntax for writing functions, introduced in ES6. Their key defining feature is that they **do NOT have their own `this`** — they inherit `this` from the enclosing lexical scope.

```javascript
// Regular function
function add(a, b) { return a + b; }

// Arrow function
const add = (a, b) => a + b;
```

**Differences from regular functions:**
- No own `this`, `arguments`, or `super` binding — all inherited from surrounding scope.
- Cannot be used as constructors (`new arrowFn()` throws an error).
- Cannot use `yield` (not usable as generator functions).
- Implicit return when using single-expression syntax without `{ }`.

```javascript
const obj = {
  name: "Sam",
  greet: function () {
    setTimeout(() => {
      console.log(this.name); // "Sam" — arrow inherits 'this' from greet()
    }, 1000);
  }
};
obj.greet();
```

---

## 14. Destructuring

**Simple definition:** A syntax to "unpack" values from arrays or properties from objects into distinct variables in a single, clean statement.

```javascript
// Array destructuring
const [a, b, c] = [1, 2, 3];
console.log(a, b, c); // 1 2 3

// Object destructuring
const { name, age } = { name: "Emma", age: 25 };
console.log(name, age); // Emma 25

// With renaming and default values
const { name: fullName, city = "Unknown" } = { name: "Emma" };
console.log(fullName, city); // Emma Unknown

// Nested destructuring
const { address: { pin } } = { address: { pin: 400001 } };

// Function parameter destructuring
function display({ name, age }) {
  console.log(`${name} is ${age}`);
}
```

---

## 15. Spread vs Rest

**Simple definition:** Both use the `...` syntax, but they do **opposite** things depending on context.

- **Spread** — expands/unpacks an array or object into individual elements.
- **Rest** — collects multiple elements/arguments back **into** a single array or object.

```javascript
// SPREAD — expanding
const arr1 = [1, 2, 3];
const arr2 = [...arr1, 4, 5]; // [1, 2, 3, 4, 5]

const obj1 = { a: 1, b: 2 };
const obj2 = { ...obj1, c: 3 }; // { a: 1, b: 2, c: 3 }

function sum(x, y, z) { return x + y + z; }
sum(...[1, 2, 3]); // spreads array as individual arguments

// REST — collecting
function sumAll(...numbers) { // numbers = [1, 2, 3, 4]
  return numbers.reduce((total, n) => total + n, 0);
}
sumAll(1, 2, 3, 4);

const [first, ...rest] = [1, 2, 3, 4]; // first = 1, rest = [2, 3, 4]
```

**Rule of thumb:** Spread is used on the **right side** (unpacking a value), Rest is used on the **left side** / in function parameters (packing values together).

---

## 16. Map, Filter, Reduce

**Simple definition:** These are array methods for transforming and processing data in a **functional, non-mutating** way (they don't change the original array).

```javascript
const numbers = [1, 2, 3, 4, 5];

// map() — transforms each element, returns a NEW array of same length
const doubled = numbers.map(n => n * 2); // [2, 4, 6, 8, 10]

// filter() — keeps elements that pass a condition, returns a NEW (possibly shorter) array
const evens = numbers.filter(n => n % 2 === 0); // [2, 4]

// reduce() — reduces the array to a SINGLE value (sum, object, etc.)
const total = numbers.reduce((acc, curr) => acc + curr, 0); // 15
```

| Method | Purpose | Returns |
|---|---|---|
| `map()` | Transform every element | New array, same length |
| `filter()` | Select elements matching a condition | New array, same or shorter length |
| `reduce()` | Combine all elements into one value | Single value (number, object, array, etc.) |

---

## 17. Debounce

**Simple definition:** Debounce ensures a function runs only **once**, after a specified delay has passed **since the last time it was called**. Useful for search inputs, resize events — anything that fires rapidly but you only want to react once things settle.

```javascript
function debounce(fn, delay) {
  let timer;
  return function (...args) {
    clearTimeout(timer); // cancel the previous scheduled call
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

const handleSearch = debounce((query) => {
  console.log("Searching for:", query);
}, 500);

// Even if called rapidly, the API only fires 500ms after the LAST keystroke
input.addEventListener("input", (e) => handleSearch(e.target.value));
```

---

## 18. Throttle

**Simple definition:** Throttle ensures a function runs **at most once every X milliseconds**, no matter how many times it's triggered. Useful for scroll events, mouse-move tracking, button-click rate limiting.

```javascript
function throttle(fn, limit) {
  let inThrottle;
  return function (...args) {
    if (!inThrottle) {
      fn.apply(this, args);
      inThrottle = true;
      setTimeout(() => (inThrottle = false), limit);
    }
  };
}

const handleScroll = throttle(() => {
  console.log("Scroll event handled");
}, 1000);

window.addEventListener("scroll", handleScroll);
```

**Debounce vs Throttle:**
| | Debounce | Throttle |
|---|---|---|
| Behavior | Waits for a pause in activity, then runs once | Runs at a regular, fixed rate regardless of activity |
| Use case | Search-box autocomplete, form validation | Scroll listeners, window resize, button spam prevention |

---

## 19. Deep Copy vs Shallow Copy

**Simple definition:** This is about how you **duplicate** objects/arrays — do the nested/inner objects get truly copied, or just referenced?

- **Shallow Copy** — copies only the top-level properties. Nested objects are still shared by reference (changing a nested object in the copy affects the original).
- **Deep Copy** — recursively copies ALL levels, so nested objects are fully independent.

```javascript
const original = { name: "Tom", address: { city: "NYC" } };

// SHALLOW COPY
const shallow = { ...original }; // or Object.assign({}, original)
shallow.address.city = "LA";
console.log(original.address.city); // "LA" — original got affected too!

// DEEP COPY
const deep = JSON.parse(JSON.stringify(original)); // simple method (has limitations: no functions, undefined, dates get stringified)
deep.address.city = "Chicago";
console.log(original.address.city); // "LA" — original unaffected

// Modern deep copy method:
const deep2 = structuredClone(original); // handles Dates, Maps, Sets, circular refs — recommended
```

---

## 20. Memory Leaks

**Simple definition:** A memory leak happens when your program keeps memory allocated for objects/data that are no longer needed, because JavaScript's Garbage Collector can't reclaim them (something is still holding a reference to them).

**Common causes:**
1. **Global variables** — accidentally creating variables in global scope that never get cleared.
2. **Forgotten timers/intervals** — `setInterval` that's never cleared keeps its closure (and everything it references) alive forever.
3. **Detached DOM elements** — removing an element from the DOM but still holding a JS reference to it somewhere.
4. **Uncleared event listeners** — attaching listeners without removing them when no longer needed.
5. **Closures holding references** — a closure unintentionally keeping large objects alive longer than necessary.

```javascript
// Example leak: interval never cleared
function startTimer() {
  const largeData = new Array(1000000).fill("data");
  setInterval(() => {
    console.log(largeData.length); // keeps largeData alive forever
  }, 1000);
}
```

**Fix:** Always clean up — `clearInterval()`, `removeEventListener()`, nullify unused references.

---

## 21. Optional Chaining (`?.`)

**Simple definition:** A safe way to access deeply nested object properties **without** throwing an error if an intermediate property is `null` or `undefined`.

```javascript
const user = { profile: { name: "Alex" } };

console.log(user.profile?.name);      // "Alex"
console.log(user.address?.city);      // undefined (no error, even though 'address' doesn't exist)
console.log(user.address.city);       // ❌ TypeError: Cannot read properties of undefined

// Also works with function calls and array access
user.greet?.();      // does nothing if 'greet' doesn't exist, instead of throwing an error
arr?.[0];             // safely access array index
```

---

## 22. Nullish Coalescing (`??`)

**Simple definition:** Returns the right-hand value **only if** the left-hand value is `null` or `undefined` — unlike `||`, which also treats falsy values like `0`, `""`, and `false` as "empty."

```javascript
const count = 0;

console.log(count || 10); // 10 — WRONG! (0 is falsy, so || incorrectly replaces it)
console.log(count ?? 10); // 0  — CORRECT (0 is not null/undefined, so it's kept)

let userName;
console.log(userName ?? "Guest"); // "Guest" — userName is undefined
```

**Rule:** Use `??` when you specifically want to check for `null`/`undefined` only, and preserve legitimate falsy values like `0` or `""`.

---

## 23. Modules (`import`/`export`)

**Simple definition:** ES6 Modules let you split code into separate reusable files, and share variables/functions between them explicitly using `export` and `import`.

**Named exports** (can export multiple things per file):
```javascript
// math.js
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;

// app.js
import { add, subtract } from "./math.js";
```

**Default export** (one per file, can be named anything on import):
```javascript
// user.js
export default function greet() { console.log("Hello"); }

// app.js
import greet from "./user.js";
```

**Combining both:**
```javascript
import greet, { add, subtract } from "./module.js";
```

**Key benefits:** better code organization, reusability, avoids polluting the global namespace, supports **tree-shaking** (removing unused code during bundling).

---

# 🎯 Interview Questions — Model Answers

### Q1. Explain the Event Loop.
The Event Loop is what allows JavaScript, despite being single-threaded, to handle asynchronous tasks. Synchronous code runs on the **Call Stack** first. Async operations (timers, HTTP requests) are offloaded to Web APIs/Node APIs, and their callbacks are placed into either the **Microtask Queue** (Promises) or **Macrotask Queue** (`setTimeout`, etc.) once ready. The Event Loop continuously checks if the Call Stack is empty — if it is, it pushes the next task from the queue onto the stack. It always **fully empties the Microtask Queue before processing the next Macrotask**.

### Q2. Difference between Promise and async/await.
A **Promise** is the underlying object representing a future value with `.then()`/`.catch()` chaining syntax. **async/await** is syntactic sugar built on top of Promises that makes asynchronous code look and read like synchronous code, improving readability and making error handling simpler via `try/catch` instead of `.catch()`. Under the hood, `await` still pauses execution until the Promise settles — it doesn't change how Promises work, just how we write around them.

### Q3. What is Closure?
A closure is a function that retains access to variables from its **outer (enclosing) scope**, even after that outer function has finished executing. This happens because the inner function keeps a reference to its lexical environment. Closures are the foundation behind data privacy, memoization, and function factories in JavaScript.

### Q4. Explain Hoisting.
Hoisting is JavaScript's behavior of moving variable and function **declarations** to the top of their scope during the compilation phase, before the code actually runs. `var` variables are hoisted and initialized as `undefined`. `let` and `const` are hoisted too, but remain uninitialized in the **Temporal Dead Zone** until their declaration line is reached — accessing them earlier throws a `ReferenceError`. Function declarations are hoisted completely (including their body), so they can be called before they appear in the code.

### Q5. Difference between `==` and `===`.
`==` (loose equality) compares two values **after** converting them to a common type (type coercion) — e.g., `"5" == 5` is `true`. `===` (strict equality) compares both **value and type** without any conversion — e.g., `"5" === 5` is `false`. Best practice is to always use `===` to avoid unexpected bugs from implicit type coercion.

```javascript
console.log(0 == "");     // true  (coerced)
console.log(0 === "");    // false (different types)
console.log(null == undefined);  // true
console.log(null === undefined); // false
```

### Q6. How does `this` work?
`this` refers to the object executing the current function, and its value is determined by **how the function is called** (not where it's written) — except for arrow functions, which don't have their own `this` and instead inherit it lexically from their surrounding scope. When a function is called as a method (`obj.method()`), `this` is the object before the dot. When called standalone, `this` is `undefined` (strict mode) or the global object. `call()`, `apply()`, and `bind()` let you manually set what `this` refers to, and `new` sets `this` to the newly created instance.

---

*End of notes.*
