# React.js Complete Notes

---

## 1. What is React?

**Simple definition:** React is a **JavaScript library for building user interfaces**, based on the idea of breaking your UI into reusable **components** that manage their own data (state) and re-render automatically when that data changes.

**Core idea:** Instead of manually updating the DOM whenever data changes (as you would with plain JavaScript), you describe **what the UI should look like** for a given state, and React figures out **how to efficiently update the actual DOM** to match.

---

# Part 1: Hooks

**Simple definition:** Hooks are special functions (all starting with `use`) that let you "hook into" React features — state, lifecycle, context — from **functional components**, without needing class components.

---

## 2. `useState`

**Simple definition:** Lets a functional component **hold and update its own local state**. Calling the state-setter triggers React to re-render the component with the new value.

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0); // [currentValue, setterFunction]

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
    </div>
  );
}
```

### Key behaviors
- Calling `setCount` **schedules a re-render** — it doesn't update `count` immediately in the current execution (state updates are asynchronous/batched).
- When the new state depends on the **previous** state, use the functional updater form to avoid stale-value bugs:
```jsx
setCount((prevCount) => prevCount + 1); // safer than setCount(count + 1) in rapid updates
```
- State is preserved **between re-renders** but reset if the component unmounts.

---

## 3. `useEffect`

**Simple definition:** Lets you run **side effects** in a functional component — code that interacts with something outside of React's rendering (API calls, subscriptions, timers, manually modifying the DOM) — after the component renders.

```jsx
import { useEffect, useState } from "react";

function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then((res) => res.json())
      .then(setUser);
  }, [userId]); // dependency array — effect re-runs only when userId changes

  return <div>{user ? user.name : "Loading..."}</div>;
}
```

### The dependency array controls when it runs
| Dependency array | Behavior |
|---|---|
| Omitted entirely | Runs after **every** render |
| `[]` (empty array) | Runs **once**, after the initial render only |
| `[value1, value2]` | Runs after the initial render, and again whenever any listed value changes |

### Cleanup function
If your effect sets up something that needs to be undone (a subscription, timer, event listener), return a **cleanup function** — it runs before the effect re-runs, and when the component unmounts.
```jsx
useEffect(() => {
  const timer = setInterval(() => console.log("tick"), 1000);
  return () => clearInterval(timer); // cleanup — prevents memory leaks
}, []);
```

---

## 4. `useMemo`

**Simple definition:** Caches the **result of an expensive computation**, recalculating it only when its dependencies change — avoiding unnecessary recalculation on every render.

```jsx
import { useMemo } from "react";

function ProductList({ products, searchTerm }) {
  const filtered = useMemo(() => {
    return products.filter((p) => p.name.includes(searchTerm));
  }, [products, searchTerm]); // only recompute when these change

  return <ul>{filtered.map((p) => <li key={p.id}>{p.name}</li>)}</ul>;
}
```
**When to use it:** Expensive calculations (large array filtering/sorting), or preventing new object/array references from breaking a child's `React.memo` optimization.

---

## 5. `useCallback`

**Simple definition:** Caches a **function reference** itself, so the same function isn't recreated on every render — important for preventing unnecessary re-renders of child components wrapped in `React.memo`.

```jsx
import { useCallback, useState, memo } from "react";

const Child = memo(function Child({ onClick }) {
  console.log("Child rendered");
  return <button onClick={onClick}>Click</button>;
});

function Parent() {
  const [count, setCount] = useState(0);

  const handleClick = useCallback(() => {
    console.log("Clicked");
  }, []); // same function reference across renders

  return (
    <>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <Child onClick={handleClick} />
    </>
  );
}
```
Without `useCallback`, `handleClick` would be a brand-new function on every render of `Parent`, defeating `Child`'s `React.memo` optimization (different reference = "changed" prop, even though the logic is identical).

---

## 6. `useRef`

**Simple definition:** Creates a **mutable reference** that persists across renders **without causing a re-render** when it changes. Commonly used to (1) directly access a DOM element, or (2) store a value you need to keep around between renders but that shouldn't trigger UI updates.

### 1. Accessing a DOM element
```jsx
import { useRef } from "react";

function TextInput() {
  const inputRef = useRef(null);

  const focusInput = () => inputRef.current.focus();

  return (
    <>
      <input ref={inputRef} type="text" />
      <button onClick={focusInput}>Focus the input</button>
    </>
  );
}
```

### 2. Storing a mutable value (doesn't trigger re-render)
```jsx
function Timer() {
  const countRef = useRef(0);

  useEffect(() => {
    const interval = setInterval(() => {
      countRef.current += 1; // updates the value, but does NOT re-render the component
      console.log(countRef.current);
    }, 1000);
    return () => clearInterval(interval);
  }, []);

  return <p>Check console for count</p>;
}
```

### `useRef` vs `useState`
| | `useRef` | `useState` |
|---|---|---|
| Triggers re-render on change | ❌ No | ✅ Yes |
| Value persists across renders | ✅ Yes | ✅ Yes |
| Common use | DOM access, storing values that don't affect UI (timers, previous values) | Data that affects what's rendered |

---

## 7. `useReducer`

**Simple definition:** An alternative to `useState` for managing **more complex state logic** — especially when the next state depends on the previous state in complicated ways, or when multiple related pieces of state update together. Works like a mini-Redux inside a single component.

```jsx
import { useReducer } from "react";

const initialState = { count: 0 };

function reducer(state, action) {
  switch (action.type) {
    case "increment":
      return { count: state.count + 1 };
    case "decrement":
      return { count: state.count - 1 };
    case "reset":
      return initialState;
    default:
      throw new Error("Unknown action type");
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "decrement" })}>-</button>
      <button onClick={() => dispatch({ type: "reset" })}>Reset</button>
    </div>
  );
}
```

**When to prefer `useReducer` over `useState`:**
- State transitions are complex (multiple sub-values updating together, or logic depending heavily on the previous state).
- The next state logic is easier to read as a single reducer function rather than scattered `setState` calls.
- You want to pass `dispatch` down to child components instead of multiple individual setter functions.

---

## 8. Custom Hooks

**Simple definition:** A custom hook is simply a **JavaScript function whose name starts with `use`** that can call other hooks inside it — letting you **extract and reuse stateful logic** across multiple components, without duplicating code.

```jsx
// useFetch.js — a custom hook
import { useState, useEffect } from "react";

function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    setLoading(true);
    fetch(url)
      .then((res) => res.json())
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [url]);

  return { data, loading, error };
}

export default useFetch;
```

```jsx
// Using it in any component — logic is fully reused
function UserProfile({ userId }) {
  const { data: user, loading, error } = useFetch(`/api/users/${userId}`);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error loading user</p>;
  return <p>{user.name}</p>;
}

function ProductPage({ productId }) {
  const { data: product, loading } = useFetch(`/api/products/${productId}`);
  // same fetching logic reused, completely different data
}
```
**Why custom hooks matter:** Without them, this fetch-loading-error logic would be copy-pasted into every component that needs to fetch data. Custom hooks package up reusable stateful logic into a single, testable, composable unit.

---

# Part 2: React Concepts

## 9. Virtual DOM

**Simple definition:** The Virtual DOM is a **lightweight, in-memory representation of the actual DOM** (just a plain JavaScript object tree). React uses it to figure out the **minimal set of changes** needed before touching the real, much slower browser DOM.

**Why it matters:** Directly manipulating the real DOM is slow — every change can trigger layout recalculations and repaints. Instead of updating the DOM directly every time state changes, React:
1. Builds a new Virtual DOM tree reflecting the updated state.
2. Compares it against the previous Virtual DOM tree (this comparison process is called **reconciliation** — see below).
3. Calculates the minimal set of actual DOM changes needed.
4. Applies only those specific changes to the real DOM.

This batching and diffing process is generally much faster than naive direct DOM manipulation, especially for frequent, complex UI updates.

---

## 10. Reconciliation

**Simple definition:** Reconciliation is the **algorithm React uses to compare** the previous Virtual DOM tree with the new one, determining exactly what changed so it can update the real DOM efficiently, instead of re-rendering everything from scratch.

### How it works (simplified)
1. **Element type comparison:** If an element's type changes (e.g., a `<div>` becomes a `<span>`), React tears down the old tree and builds a completely new one from scratch at that point.
2. **Same type, different props:** React keeps the same underlying DOM node and just updates the changed attributes/props.
3. **Lists — the role of `key`:** When rendering lists, React uses the `key` prop to track which items are which across re-renders — without stable keys, React may re-render/reorder list items incorrectly or inefficiently.

```jsx
// ❌ Using array index as key — can cause bugs if list order changes
{items.map((item, index) => <li key={index}>{item.name}</li>)}

// ✅ Using a stable, unique identifier
{items.map((item) => <li key={item.id}>{item.name}</li>)}
```
- **Why index keys are risky:** If items are reordered, added, or removed from the middle of the list, index-based keys shift and no longer correctly correspond to the same logical item — React may reuse the wrong DOM node/state for the wrong item (e.g., an input's typed text ending up attached to the wrong row).

---

## 11. React Rendering (The Render Process)

**Simple definition:** "Rendering" is React's process of calling your component functions to figure out what the UI **should** look like, then updating the actual DOM to match.

### The render process, step by step
1. **Trigger:** A render is triggered either by initial mount, or a state/props update (`setState`, parent re-render, context change).
2. **Render phase:** React calls your component function(s), building a new Virtual DOM tree. This phase is "pure" — no side effects should happen here (that's what `useEffect` is for).
3. **Reconciliation:** React diffs the new Virtual DOM against the previous one (as described above).
4. **Commit phase:** React applies the calculated minimal changes to the actual DOM.
5. **Effects run:** After the DOM is updated, `useEffect` callbacks run (browser paints first, then effects run asynchronously by default).

### Important nuance: re-rendering a component doesn't always mean the DOM changes
A component function running again (a "re-render") just recalculates what its Virtual DOM *should* look like — if the output is identical to before, React won't touch the real DOM at all for that part.

---

## 12. Controlled Components

**Simple definition:** A form element (input, textarea, select) whose value is **fully controlled by React state** — the displayed value always comes from state, and every change updates that state via an event handler.

```jsx
function ControlledInput() {
  const [value, setValue] = useState("");

  return (
    <input
      value={value}                              // value comes FROM state
      onChange={(e) => setValue(e.target.value)} // every keystroke updates state
    />
  );
}
```
**Benefit:** React state is always the single source of truth — you can validate, transform, or react to every keystroke immediately (e.g., real-time character count, live validation).

---

## 13. Uncontrolled Components

**Simple definition:** A form element that manages **its own internal state** in the DOM (like plain HTML), and React only reads its value **when needed** (e.g., on submit) — typically via a `ref`, instead of updating state on every keystroke.

```jsx
function UncontrolledInput() {
  const inputRef = useRef(null);

  const handleSubmit = () => {
    console.log(inputRef.current.value); // read the value only when needed
  };

  return (
    <>
      <input ref={inputRef} defaultValue="" />
      <button onClick={handleSubmit}>Submit</button>
    </>
  );
}
```

### Controlled vs Uncontrolled
| | Controlled | Uncontrolled |
|---|---|---|
| Source of truth | React state | The DOM itself |
| Re-renders on every keystroke | ✅ Yes | ❌ No |
| Good for | Real-time validation, conditional UI based on input | Simple forms, integrating with non-React code, performance-sensitive large forms |

---

## 14. Lifting State Up

**Simple definition:** When two or more sibling components need to **share and stay in sync with the same state**, you move (lift) that state to their **closest common parent**, which then passes it down via props.

```jsx
function Parent() {
  const [temperature, setTemperature] = useState(0); // lifted state lives here

  return (
    <>
      <CelsiusInput temperature={temperature} onChange={setTemperature} />
      <FahrenheitDisplay temperature={temperature} />
    </>
  );
}

function CelsiusInput({ temperature, onChange }) {
  return <input value={temperature} onChange={(e) => onChange(e.target.value)} />;
}

function FahrenheitDisplay({ temperature }) {
  return <p>{(temperature * 9) / 5 + 32}°F</p>;
}
```
**Why:** Without lifting state up, each sibling would have its own separate, unsynchronized copy of the data — lifting it ensures there's a single source of truth that both children read from and update.

---

## 15. Props Drilling

**Simple definition:** The problem of passing props down through **many layers of components** just to get data from a top-level ancestor to a deeply nested descendant — even though the intermediate components don't actually use that data themselves.

```jsx
function App() {
  const [user, setUser] = useState({ name: "Alice" });
  return <Layout user={user} />;
}

function Layout({ user }) {
  return <Sidebar user={user} />; // Layout doesn't use `user`, just passes it along
}

function Sidebar({ user }) {
  return <UserBadge user={user} />; // Sidebar doesn't use it either
}

function UserBadge({ user }) {
  return <p>{user.name}</p>; // finally used here, 3 levels deep
}
```
**The fix:** Use **Context API** (or a state management library like Redux/Zustand) to let `UserBadge` access `user` directly, without every intermediate component needing to know or forward it.

---

## 16. Context API

**Simple definition:** A built-in React feature that lets you share data across the component tree **without manually passing props at every level** — solving the props drilling problem for relatively simple, infrequently-changing shared data.

```jsx
const UserContext = createContext();

function App() {
  const [user, setUser] = useState({ name: "Alice" });
  return (
    <UserContext.Provider value={user}>
      <Layout />
    </UserContext.Provider>
  );
}

function UserBadge() {
  const user = useContext(UserContext); // reads directly, no drilling needed
  return <p>{user.name}</p>;
}
```
**Limitation to know:** Every component consuming a context re-renders whenever *any* part of that context's value changes — there's no built-in selector mechanism, so it's best suited for data that changes rarely (theme, authenticated user, locale) rather than frequently-updating state.

---

## 17. Lazy Loading

**Simple definition:** Deferring the loading of a component's code until it's actually needed, instead of including it in the initial bundle — reducing the amount of JavaScript the browser must download upfront.

```jsx
import { lazy, Suspense } from "react";

const HeavyComponent = lazy(() => import("./HeavyComponent"));

function App() {
  return (
    <Suspense fallback={<p>Loading...</p>}>
      <HeavyComponent />
    </Suspense>
  );
}
```
- `HeavyComponent`'s code is fetched from the network only when it's actually rendered — commonly used for route-based code splitting (each page loads only when visited).

---

## 18. Suspense

**Simple definition:** A React component that lets you **declaratively show a fallback UI** (like a loading spinner) while its children aren't ready yet — most commonly used alongside `lazy()` for code-split components, and also usable for data fetching in newer React patterns.

```jsx
<Suspense fallback={<Spinner />}>
  <LazyLoadedComponent />
</Suspense>
```
- Think of `Suspense` as saying: "While anything inside here isn't ready, show this fallback instead."
- Multiple `Suspense` boundaries can be nested to show granular loading states for different parts of the UI independently.

---

## 19. Error Boundary

**Simple definition:** A component that **catches JavaScript errors** thrown anywhere in its child component tree during rendering, and displays a fallback UI instead of letting the entire app crash with a blank white screen.

```jsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true }; // update state so next render shows the fallback
  }

  componentDidCatch(error, info) {
    console.error("Caught an error:", error, info); // log to an error reporting service
  }

  render() {
    if (this.state.hasError) {
      return <h2>Something went wrong.</h2>;
    }
    return this.props.children;
  }
}
```
```jsx
<ErrorBoundary>
  <SomeComponentThatMightCrash />
</ErrorBoundary>
```
**Important limitation:** Error boundaries **must currently be class components** (there's no official hook equivalent), and they only catch errors during **rendering**, lifecycle methods, and constructors of child components — they do **not** catch errors in event handlers, asynchronous code, or server-side rendering; those need regular `try/catch`.

---

## 20. Memoization (in React, recap)

**Simple definition:** Caching a value or a component's render output so it isn't unnecessarily recomputed/re-rendered when nothing relevant has changed. In React, this is implemented via:
- **`React.memo`** — skips re-rendering a component if its props haven't changed.
- **`useMemo`** — caches a computed value between renders.
- **`useCallback`** — caches a function reference between renders.

```jsx
const ExpensiveComponent = memo(function ExpensiveComponent({ data }) {
  // only re-renders if `data` prop actually changes
  return <div>{data}</div>;
});
```
**Golden rule:** Use memoization deliberately, based on a real, measured performance issue — not by default everywhere, since the comparison/caching overhead can outweigh the benefit for cheap components/values.

---

## Interview Questions & Answers

### Q1. Difference between `useMemo` and `useCallback`?

| | `useMemo` | `useCallback` |
|---|---|---|
| Memoizes | A **value** (the result of running a function) | A **function** (the function itself, not its result) |
| Syntax | `useMemo(() => computeValue(), deps)` | `useCallback(() => doSomething(), deps)` |
| Equivalent | — | `useCallback(fn, deps)` is shorthand for `useMemo(() => fn, deps)` |
| Typical use | Avoiding expensive recalculations | Preventing new function references from breaking `React.memo` on children |

**One-line answer:** `useMemo` caches a *computed value*; `useCallback` caches a *function reference*. Both exist to prevent unnecessary work/re-renders by avoiding creating new references on every render.

### Q2. When does `useEffect` run?

`useEffect` runs **after the browser has painted the updated DOM** to the screen (asynchronously, after the render/commit phases) — not during rendering itself. Its exact timing depends on the dependency array:
- **No dependency array:** Runs after every single render.
- **Empty array `[]`:** Runs once, only after the initial mount.
- **Array with values `[a, b]`:** Runs after the initial mount, and again anytime `a` or `b` changes between renders.
- If a **cleanup function** is returned, it runs right before the effect re-runs again (when dependencies change) and once more when the component unmounts.

**Important nuance:** Because effects run *after* paint, they're the correct place for side effects (data fetching, subscriptions, manual DOM work) that shouldn't block the visual update — but this also means there's a brief moment where the DOM is already updated before the effect has run, which matters for effects that measure or adjust layout (for those rare cases, `useLayoutEffect` runs synchronously before paint instead).

### Q3. React lifecycle — how does it map to hooks?

Class components have lifecycle methods; functional components achieve the same behavior using hooks:

| Class lifecycle method | Functional equivalent |
|---|---|
| `constructor` / initial state | `useState(initialValue)` |
| `componentDidMount` | `useEffect(() => { ... }, [])` |
| `componentDidUpdate` | `useEffect(() => { ... }, [dependency])` |
| `componentWillUnmount` | `useEffect(() => { return () => { /* cleanup */ } }, [])` |
| `componentDidCatch` / `getDerivedStateFromError` | Error Boundaries (still require a class component) |

**Conceptually, the three broad lifecycle phases are:**
1. **Mounting** — component is created and inserted into the DOM for the first time.
2. **Updating** — component re-renders due to changed state/props.
3. **Unmounting** — component is removed from the DOM (cleanup time).

### Q4. Explain the React rendering process end-to-end.

1. **Trigger:** Something causes a render — initial mount, a `setState` call, a parent re-rendering, or a context value changing.
2. **Render phase:** React calls the component function(s) to compute what the UI should look like, producing a new Virtual DOM tree. This phase must be pure — no side effects.
3. **Reconciliation (diffing):** React compares the new Virtual DOM tree against the previous one to calculate the minimal set of actual changes needed.
4. **Commit phase:** React applies those specific changes to the real DOM (inserting, updating, or removing actual DOM nodes).
5. **Browser paints** the updated UI on screen.
6. **Effects run:** `useEffect` callbacks fire after the paint (asynchronously); `useLayoutEffect` callbacks fire synchronously right after the DOM is updated but before the browser paints, for cases needing to measure/adjust the DOM before the user sees it.

**Key insight to mention in an interview:** Re-rendering a component (step 2) doesn't automatically mean the real DOM changes — if reconciliation finds the output is identical to before, React skips the commit step for that part entirely, which is a big part of why React can be so efficient even with frequent re-renders.

---

## Quick Summary Cheat Sheet

| Concept | One-liner |
|---|---|
| `useState` | Local component state that triggers re-renders on change |
| `useEffect` | Runs side effects after render (data fetching, subscriptions) |
| `useMemo` | Caches a computed value between renders |
| `useCallback` | Caches a function reference between renders |
| `useRef` | Mutable value/DOM reference that persists without causing re-renders |
| `useReducer` | Manages complex state logic via a reducer function + dispatch |
| Custom Hooks | Reusable stateful logic extracted into a `use`-prefixed function |
| Virtual DOM | In-memory lightweight copy of the DOM used for efficient diffing |
| Reconciliation | Algorithm comparing old vs new Virtual DOM to find minimal changes |
| Controlled Components | Form input value driven by React state |
| Uncontrolled Components | Form input manages its own value in the DOM, read via ref |
| Lifting State Up | Moving shared state to the closest common parent component |
| Props Drilling | Passing props through many layers just to reach a deeply nested child |
| Context API | Built-in way to share data without prop drilling |
| Lazy Loading | Deferring a component's code until it's actually needed |
| Suspense | Declarative fallback UI while children aren't ready yet |
| Error Boundary | Class component that catches render-time errors in its children |
| Memoization | Caching values/renders to avoid unnecessary recomputation |
