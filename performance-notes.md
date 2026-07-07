# Frontend Performance Complete Notes (React)

---

## 1. Why Performance Matters

**Simple definition:** Performance optimization is about making your app **load faster** and **run smoother** — reducing the amount of code the browser has to download, and reducing unnecessary work the browser has to do while rendering and updating the UI.

Two broad categories of performance problems:
1. **Load-time performance** — how much JavaScript/assets does the browser need to download and parse before the app is usable? (Solved by: code splitting, lazy loading, image optimization, bundle optimization.)
2. **Runtime performance** — how efficiently does the app re-render and update as the user interacts with it? (Solved by: memoization, `React.memo`, `useMemo`, `useCallback`, virtualization.)

---

## 2. Code Splitting

**Simple definition:** Code splitting means **breaking your JavaScript bundle into smaller chunks** that are loaded **only when needed**, instead of shipping one giant bundle containing your entire app upfront.

**Why it matters:** Without code splitting, a user visiting your homepage downloads the JavaScript for *every* page/feature in your app — even ones they may never visit (like an admin panel or a settings page). This slows down the initial load significantly, especially on slower networks/devices.

### How it's done in React (via dynamic `import()`)
```jsx
// Without code splitting — everything bundled together
import Dashboard from "./Dashboard";
import Settings from "./Settings";
import AdminPanel from "./AdminPanel";
```

```jsx
// With code splitting — each becomes a separate chunk, loaded on demand
import { lazy } from "react";

const Dashboard = lazy(() => import("./Dashboard"));
const Settings = lazy(() => import("./Settings"));
const AdminPanel = lazy(() => import("./AdminPanel"));
```

- Bundlers like **Webpack** or **Vite** automatically create separate `.js` chunk files for each dynamically imported module.
- Most commonly applied at the **route level** — each page/route becomes its own chunk, loaded only when the user navigates there.

```jsx
// Route-based code splitting example (React Router)
import { lazy, Suspense } from "react";
import { Routes, Route } from "react-router-dom";

const Dashboard = lazy(() => import("./pages/Dashboard"));
const Profile = lazy(() => import("./pages/Profile"));

function App() {
  return (
    <Suspense fallback={<p>Loading...</p>}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/profile" element={<Profile />} />
      </Routes>
    </Suspense>
  );
}
```

---

## 3. Lazy Loading

**Simple definition:** Lazy loading is the broader principle of **deferring the loading of a resource until it's actually needed** — instead of loading everything upfront. Code splitting (above) is essentially lazy loading applied to JavaScript; the same idea also applies to images, components, and other assets.

### Lazy loading a component (React)
```jsx
import { lazy, Suspense, useState } from "react";

const HeavyChart = lazy(() => import("./HeavyChart")); // not downloaded until rendered

function Dashboard() {
  const [showChart, setShowChart] = useState(false);

  return (
    <div>
      <button onClick={() => setShowChart(true)}>Show Chart</button>
      {showChart && (
        <Suspense fallback={<p>Loading chart...</p>}>
          <HeavyChart />
        </Suspense>
      )}
    </div>
  );
}
```
- `HeavyChart`'s code is only fetched from the network the moment `showChart` becomes `true` — not as part of the initial page load.

### Lazy loading images (native browser support)
```html
<img src="banner.jpg" alt="Banner" loading="lazy" />
```
- The browser automatically defers loading this image until it's about to scroll into the viewport — great for long pages with many images (e.g., product listings, image galleries).

**Rule of thumb:** Code splitting = lazy loading *applied to your JS bundles*. Lazy loading is the general strategy that also applies to images, videos, and other heavy resources.

---

## 4. Image Optimization

**Simple definition:** Reducing image file sizes and delivering the right image for the right context (device size, format support) — since images are often the **heaviest assets** on a webpage and directly impact load time.

### Key techniques
1. **Modern formats:** Use **WebP** or **AVIF** instead of JPEG/PNG — same visual quality at a fraction of the file size.
2. **Responsive images:** Serve different image sizes based on the viewport, using `srcset`.
   ```html
   <img
     src="photo-800.jpg"
     srcset="photo-400.jpg 400w, photo-800.jpg 800w, photo-1200.jpg 1200w"
     sizes="(max-width: 600px) 400px, 800px"
     alt="Photo"
   />
   ```
3. **Lazy loading** (see above) — don't load off-screen images immediately.
4. **Compression** — tools like `sharp`, `imagemin`, or CDN-based image services (Cloudinary, Imgix) automatically compress and resize images on the fly.
5. **Correct dimensions** — always set explicit `width`/`height` (or `aspect-ratio`) so the browser can reserve space before the image loads, preventing **layout shift** (a key Core Web Vital: CLS).

### In Next.js
```jsx
import Image from "next/image";

<Image
  src="/product.jpg"
  alt="Product"
  width={400}
  height={300}
  priority // for above-the-fold images that shouldn't be lazy-loaded
/>
```
- Next.js's `<Image>` component automatically handles resizing, format conversion (WebP), and lazy loading for you.

---

## 5. Memoization (General Concept)

**Simple definition:** Memoization is a technique where you **cache the result of an expensive computation** and reuse that cached result instead of recalculating it — as long as the inputs haven't changed.

**Real-world analogy:** If someone keeps asking you "What's 47 × 89?" and you already calculated it once, you just recall the answer from memory instead of redoing the multiplication every time — that's memoization.

In React specifically, memoization is used to **avoid unnecessary re-renders and recalculations**, which is where `React.memo`, `useMemo`, and `useCallback` come in.

---

## 6. `React.memo`

**Simple definition:** A higher-order component that **wraps a component** and tells React: "Only re-render this component if its **props** actually changed." Without it, a child component re-renders every time its parent re-renders — even if the child's props are exactly the same as before.

```jsx
import { memo } from "react";

const ProductCard = memo(function ProductCard({ name, price }) {
  console.log("Rendering:", name);
  return (
    <div>
      <h3>{name}</h3>
      <p>${price}</p>
    </div>
  );
});
```

```jsx
function ProductList() {
  const [count, setCount] = useState(0); // unrelated state

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>Clicked {count} times</button>
      {/* Without React.memo, ProductCard re-renders every time count changes,
          even though its props (name, price) never change */}
      <ProductCard name="Laptop" price={999} />
    </div>
  );
}
```

### Important gotcha: object/function props
`React.memo` does a **shallow comparison** of props. If you pass a new object, array, or function on every render (even with the same values), `React.memo` will still think the props "changed" because the reference is different.

```jsx
// ❌ This breaks React.memo — a NEW function is created every render
<ProductCard onClick={() => handleClick(id)} />

// ✅ Fix: memoize the function itself with useCallback (see below)
const handleCardClick = useCallback(() => handleClick(id), [id]);
<ProductCard onClick={handleCardClick} />
```

**When to use it:** Components that render often due to parent re-renders, but whose own props rarely change — especially "expensive" components (complex UI, heavy computations, large lists).

**When NOT to use it:** Small, cheap-to-render components — the overhead of comparing props isn't worth it if rendering the component is already trivial. Overusing `React.memo` everywhere can actually add unnecessary comparison overhead.

---

## 7. `useMemo`

**Simple definition:** A hook that **memoizes the result of a computation** (a value), recalculating it only when its dependencies change — avoiding expensive recalculations on every render.

```jsx
import { useMemo, useState } from "react";

function ProductList({ products }) {
  const [searchTerm, setSearchTerm] = useState("");

  // Without useMemo, this expensive filter runs on EVERY render,
  // even if unrelated state changes elsewhere in the component
  const filteredProducts = useMemo(() => {
    console.log("Filtering products...");
    return products.filter((p) =>
      p.name.toLowerCase().includes(searchTerm.toLowerCase())
    );
  }, [products, searchTerm]); // only recompute when these change

  return (
    <div>
      <input value={searchTerm} onChange={(e) => setSearchTerm(e.target.value)} />
      <ul>{filteredProducts.map((p) => <li key={p.id}>{p.name}</li>)}</ul>
    </div>
  );
}
```

**When to use it:**
- Expensive calculations (heavy filtering/sorting/transforming of large arrays, complex math).
- Preventing unnecessary re-creation of objects/arrays passed as props to a `React.memo`-wrapped child (since a new object reference on every render would defeat `React.memo`'s comparison).

**When NOT to use it:** For cheap computations (like `a + b`), `useMemo` adds overhead (storing and comparing dependencies) that isn't worth it — it can actually make things slightly slower. Don't wrap everything in `useMemo` "just in case."

---

## 8. `useCallback`

**Simple definition:** A hook that **memoizes a function itself** (not its result), so the same function reference is reused across renders instead of creating a brand-new function every time — important for preventing unnecessary re-renders in child components wrapped with `React.memo`.

```jsx
import { useCallback, useState } from "react";

function ParentComponent() {
  const [count, setCount] = useState(0);

  // Without useCallback, a NEW function is created on every render of ParentComponent,
  // which would break React.memo on ChildButton below
  const handleClick = useCallback(() => {
    console.log("Button clicked");
  }, []); // empty deps = function reference never changes

  return (
    <div>
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
      <ChildButton onClick={handleClick} />
    </div>
  );
}

const ChildButton = memo(function ChildButton({ onClick }) {
  console.log("ChildButton rendered");
  return <button onClick={onClick}>Click me</button>;
});
```
- Without `useCallback`, clicking the counter button would cause `ParentComponent` to re-render, creating a *new* `handleClick` function each time — which would make `ChildButton`'s `React.memo` comparison fail (different function reference), causing an unnecessary re-render of `ChildButton` too.

### `useMemo` vs `useCallback`
| | `useMemo` | `useCallback` |
|---|---|---|
| Memoizes | A **value** (result of a computation) | A **function** (the function itself) |
| Returns | The computed value | The function reference |
| Equivalent to | `useMemo(() => fn, deps)` | — |

```js
// useCallback(fn, deps) is literally shorthand for:
useMemo(() => fn, deps);
```

**Golden rule for both:** Only use them when there's an actual, measurable performance problem (expensive computation, or breaking `React.memo` on a child) — not by default on every function/value. Premature memoization adds code complexity and can even hurt performance in trivial cases.

---

## 9. Virtualization

**Simple definition:** Virtualization (a.k.a. "windowing") means **rendering only the items currently visible in the viewport** — instead of rendering an entire long list (which could be thousands of items) all at once. As the user scrolls, items are dynamically rendered/removed from the DOM.

**Why it matters:** Rendering 10,000 DOM nodes for a long list is extremely slow and memory-intensive — even if the user can only see 10-15 items on screen at a time. Virtualization renders just those visible items (plus a small buffer), dramatically improving performance for large lists/tables.

### Example using `react-window`
```jsx
import { FixedSizeList } from "react-window";

function VirtualizedList({ items }) {
  const Row = ({ index, style }) => (
    <div style={style}>{items[index].name}</div>
  );

  return (
    <FixedSizeList
      height={400}       // visible container height
      itemCount={items.length}
      itemSize={35}       // height of each row
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}
```
- Even if `items` has 50,000 entries, only the ~12 rows visible in the 400px-tall container (plus a small overscan buffer) actually exist in the DOM at any moment.

**Common use cases:** Long chat message lists, infinite-scroll feeds, large data tables/grids, dropdown lists with thousands of options.

---

## 10. Bundle Optimization

**Simple definition:** Reducing the overall size of the JavaScript/CSS bundles shipped to the browser, so there's less to download, parse, and execute before the app becomes interactive.

### Key techniques

**1. Tree shaking**
- Automatically removes unused code from your final bundle (supported by modern bundlers like Webpack/Vite/Rollup), as long as you use ES modules (`import`/`export`) rather than CommonJS.
```js
// Only the used function gets included in the final bundle — the rest of lodash is dropped
import { debounce } from "lodash-es";
```

**2. Analyzing bundle size**
- Tools like `webpack-bundle-analyzer` or Vite's built-in visualizer show exactly which packages/modules are taking up the most space, helping you find bloated dependencies.

**3. Avoiding heavy dependencies**
- Prefer lightweight alternatives where possible (e.g., `date-fns` over the full `moment.js`, which is much heavier and not tree-shakeable).

**4. Code splitting** (already covered above) — splitting large bundles into smaller, on-demand chunks.

**5. Minification & compression**
- Minification (removing whitespace, shortening variable names) is handled automatically by build tools in production mode.
- Serve assets with **gzip or Brotli compression** at the server/CDN level for further size reduction over the network.

**6. Dynamic imports for rarely-used libraries**
```js
// Only load a heavy PDF library when the user actually clicks "Export PDF"
async function exportToPDF() {
  const { generatePDF } = await import("./pdfGenerator");
  generatePDF();
}
```

**7. Removing dead code and unused CSS**
- Tools like PurgeCSS remove unused CSS classes (especially relevant with utility frameworks like Tailwind, which generate PurgeCSS-style output by default in production builds).

---

## Quick Summary Cheat Sheet

| Concept | One-liner |
|---|---|
| Code Splitting | Breaking JS into smaller chunks loaded on demand (usually per-route) |
| Lazy Loading | General principle of deferring resource loading until needed (JS, images, etc.) |
| Image Optimization | Serving smaller, modern-format, correctly-sized images |
| Memoization | Caching a result to avoid recalculating it unnecessarily |
| `React.memo` | Skips re-rendering a component if its props haven't changed |
| `useMemo` | Caches a **computed value** between renders |
| `useCallback` | Caches a **function reference** between renders |
| Virtualization | Renders only visible list items instead of the entire list |
| Bundle Optimization | Reducing overall JS/CSS size via tree shaking, minification, avoiding heavy deps |

## Interview-Style Quick Answers

**Q: What's the difference between code splitting and lazy loading?**
Lazy loading is the general strategy of deferring resource loading until it's needed. Code splitting is the specific technique of breaking a JavaScript bundle into separate chunks so lazy loading can be applied to your app's code — code splitting is essentially "lazy loading for JS."

**Q: When would `React.memo` NOT help, even if used correctly?**
If new object/array/function props are created on every render of the parent (without `useMemo`/`useCallback`), `React.memo`'s shallow prop comparison will always see "different" props and re-render anyway — `React.memo` only helps when the actual prop values/references are stable across renders.

**Q: Why not wrap every component in `React.memo` and every value in `useMemo`?**
Memoization itself has a cost — storing previous values/dependencies and comparing them on every render. For cheap, fast-rendering components or trivial calculations, this overhead can outweigh the benefit, sometimes making performance slightly worse. Memoize deliberately, based on measured performance issues — not everywhere by default.

**Q: How does virtualization improve performance for long lists?**
Instead of creating DOM nodes for every item in a list (which is slow and memory-heavy for thousands of items), virtualization only renders the small subset of items currently visible in the viewport (plus a small buffer), recycling DOM nodes as the user scrolls — keeping the number of active DOM nodes constant regardless of total list size.
