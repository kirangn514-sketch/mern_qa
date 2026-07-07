# CSS Complete Notes

---

## 1. What is CSS?

**Simple definition:** CSS (Cascading Style Sheets) is the language used to control the **visual presentation** of HTML — colors, spacing, layout, fonts, and responsiveness. HTML defines *structure/content*, CSS defines *how it looks*.

This guide focuses on **layout systems** (Flexbox, Grid), **responsive design**, and the major **styling approaches** used in modern component-based frontend development (Tailwind, Styled Components, CSS Modules).

---

## 2. Flexbox

**Simple definition:** Flexbox (Flexible Box Layout) is a **one-dimensional layout system** — designed to arrange items in a single row **or** a single column, distributing space and aligning items easily even when their sizes are unknown or dynamic.

### Basic setup
```css
.container {
  display: flex;
}
```
Once a container is `display: flex`, all its direct children become **flex items** and automatically line up in a row by default.

### Key container properties

**`flex-direction`** — sets the main axis direction
```css
.container {
  flex-direction: row;        /* default: left to right */
  flex-direction: column;     /* top to bottom */
  flex-direction: row-reverse;
}
```

**`justify-content`** — aligns items along the **main axis**
```css
.container {
  justify-content: flex-start;   /* default — items packed at the start */
  justify-content: center;       /* items centered */
  justify-content: space-between; /* even spacing, first/last items touch edges */
  justify-content: space-around;  /* even spacing around each item */
}
```

**`align-items`** — aligns items along the **cross axis** (perpendicular to main axis)
```css
.container {
  align-items: stretch;    /* default — items stretch to fill cross-axis */
  align-items: center;     /* items centered on cross axis */
  align-items: flex-start; /* items aligned to the start of cross axis */
}
```

**`flex-wrap`** — controls whether items wrap onto multiple lines if they don't fit
```css
.container {
  flex-wrap: nowrap; /* default — items shrink to fit on one line */
  flex-wrap: wrap;   /* items wrap onto new lines as needed */
}
```

### Key item properties

**`flex-grow`** — how much an item should grow to fill available extra space, relative to siblings
```css
.item { flex-grow: 1; } /* this item grows to take up remaining space */
```

**`flex-shrink`** — how much an item should shrink when there isn't enough space
```css
.item { flex-shrink: 0; } /* this item won't shrink, even if space is tight */
```

**`flex-basis`** — the item's initial/default size before growing or shrinking is applied
```css
.item { flex-basis: 200px; } /* start at 200px, then grow/shrink from there */
```

**Shorthand:** `flex: <grow> <shrink> <basis>` — e.g., `flex: 1 1 200px;`

### Classic example — a centered navbar
```css
.navbar {
  display: flex;
  justify-content: space-between; /* logo on left, nav links on right */
  align-items: center;             /* vertically centered */
}
```

**Best for:** Navbars, toolbars, card layouts, centering content, evenly distributing items in a row or column — anything primarily one-directional.

---

## 3. Grid

**Simple definition:** CSS Grid is a **two-dimensional layout system** — it lets you control rows AND columns simultaneously, making it ideal for full page layouts and complex arrangements that Flexbox isn't designed to handle as cleanly.

### Basic setup
```css
.container {
  display: grid;
  grid-template-columns: 200px 1fr 1fr; /* 3 columns: fixed 200px, then two equal flexible columns */
  grid-template-rows: 80px auto 60px;   /* header, flexible content, footer */
  gap: 16px;                             /* spacing between grid cells */
}
```

### The `fr` unit
**Simple definition:** `fr` (fraction unit) represents a **share of the available space** — extremely useful for flexible, responsive grid layouts without manual pixel math.
```css
.container {
  grid-template-columns: 1fr 2fr 1fr; /* middle column gets twice the space of the outer two */
}
```

### Placing items explicitly
```css
.header  { grid-column: 1 / 4; grid-row: 1; }       /* spans all 3 columns */
.sidebar { grid-column: 1; grid-row: 2; }
.content { grid-column: 2 / 4; grid-row: 2; }
.footer  { grid-column: 1 / 4; grid-row: 3; }
```

### Named template areas (very readable approach)
```css
.container {
  display: grid;
  grid-template-columns: 200px 1fr;
  grid-template-areas:
    "sidebar header"
    "sidebar content"
    "sidebar footer";
}

.sidebar { grid-area: sidebar; }
.header  { grid-area: header; }
.content { grid-area: content; }
.footer  { grid-area: footer; }
```
This visually maps out the entire page layout directly in the CSS, making it very easy to understand at a glance.

### Auto-fit / auto-fill for responsive grids without media queries
```css
.gallery {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 16px;
}
```
This automatically fits as many 200px-minimum columns as will fit in the available width, wrapping to new rows as needed — a common pattern for responsive image galleries/card grids without writing a single media query.

### Flexbox vs Grid — when to use which
| | Flexbox | Grid |
|---|---|---|
| Dimensions | One-dimensional (row OR column) | Two-dimensional (rows AND columns together) |
| Best for | Navbars, button groups, aligning items in a line, distributing space along one axis | Full page layouts, dashboards, image galleries, complex structured layouts |
| Content-driven vs layout-driven | Content often dictates size (content-first) | Layout is typically defined first, content fits into it (layout-first) |

**Practical rule of thumb:** Use Grid for the overall page/section layout structure, and Flexbox for aligning items *within* those sections (e.g., a Grid for the page skeleton, Flexbox inside a card component to align its icon and text).

---

## 4. Responsive Design

**Simple definition:** Designing a UI that **adapts and looks good across different screen sizes** — phones, tablets, laptops, large monitors — instead of building a fixed layout that only works for one specific screen width.

### Core techniques

**1. Fluid units instead of fixed pixels**
```css
.container {
  width: 90%;        /* relative to parent, not a fixed pixel value */
  max-width: 1200px;  /* but don't let it grow unreasonably large on huge screens */
  font-size: 1rem;    /* relative to root font size, scales better than px */
}
```

**2. Media queries** — apply different styles based on screen width (or other device characteristics)
```css
/* Base styles apply to all screens (mobile-first approach) */
.card {
  width: 100%;
}

/* Applies only when viewport is at least 768px wide */
@media (min-width: 768px) {
  .card {
    width: 50%;
  }
}

/* Applies only when viewport is at least 1024px wide */
@media (min-width: 1024px) {
  .card {
    width: 33.33%;
  }
}
```

**3. Mobile-first vs Desktop-first approach**
- **Mobile-first (recommended):** Write base styles for small screens, then use `min-width` media queries to add complexity for larger screens.
- **Desktop-first:** Write base styles for large screens, then use `max-width` media queries to simplify for smaller screens.
- **Why mobile-first is generally preferred:** It forces you to prioritize essential content/functionality first, and generally results in less CSS overriding itself as screens get larger (progressive enhancement) rather than fighting to strip things down for smaller screens.

**4. Flexible layouts (Flexbox/Grid with `auto-fit`/`minmax`)** — as shown above, letting the layout naturally adapt without needing a media query for every possible screen size.

**5. Responsive images**
```html
<img src="photo.jpg" style="max-width: 100%; height: auto;" alt="Photo" />
```
- `max-width: 100%` ensures images never overflow their container, scaling down on smaller screens.

### Common breakpoints (general convention, not a strict rule)
| Device | Typical width |
|---|---|
| Mobile | < 640px |
| Tablet | 640px – 1024px |
| Desktop | > 1024px |

---

## 5. Tailwind CSS

**Simple definition:** Tailwind is a **utility-first CSS framework** — instead of writing custom CSS classes and rules yourself, you compose pre-defined, single-purpose utility classes directly in your HTML/JSX to build designs quickly.

### Example
```html
<!-- Traditional CSS approach -->
<div class="card">Hello</div>
<style>
  .card {
    padding: 16px;
    background-color: white;
    border-radius: 8px;
    box-shadow: 0 1px 3px rgba(0,0,0,0.1);
  }
</style>

<!-- Tailwind approach — no separate CSS file needed -->
<div class="p-4 bg-white rounded-lg shadow-sm">Hello</div>
```
Each class does exactly one thing: `p-4` = padding, `bg-white` = background color, `rounded-lg` = border radius, `shadow-sm` = box shadow.

### Responsive design with Tailwind
Tailwind uses **prefix-based responsive utilities**, mobile-first by default:
```html
<div class="w-full md:w-1/2 lg:w-1/3">
  <!-- full width on mobile, 50% on tablets (md), 33% on desktop (lg) -->
</div>
```

### Why it's popular
- **Speed:** No context-switching between HTML and separate CSS files — styling happens right where the markup is.
- **Consistency:** A predefined design system (spacing scale, color palette, font sizes) prevents arbitrary, inconsistent values across a codebase.
- **No unused CSS bloat in production:** Tailwind scans your code and only includes the utility classes you actually used in the final build (via its JIT compiler), keeping bundle size small despite the framework having thousands of possible utility classes.
- **No naming struggles:** You never have to invent class names like `.card-wrapper-inner-v2` — you just compose existing utilities.

### Common criticism (worth mentioning for balance)
- HTML can look cluttered with many utility classes ("class soup"), especially for complex components.
- Requires learning Tailwind's specific naming conventions and scale system.
- Some teams extract repeated utility combinations into reusable components (React/Vue components) rather than repeating long class strings everywhere.

---

## 6. Styled Components

**Simple definition:** A **CSS-in-JS** library that lets you write actual CSS **inside your JavaScript/React files**, scoped directly to a specific component — styles and markup live together, and class names are automatically generated to avoid collisions.

```jsx
import styled from "styled-components";

const Button = styled.button`
  padding: 10px 20px;
  background-color: #3b82f6;
  color: white;
  border-radius: 6px;
  border: none;
  cursor: pointer;

  &:hover {
    background-color: #2563eb;
  }
`;

function App() {
  return <Button>Click Me</Button>;
}
```
- `Button` is now a real React component with the defined styles baked in — you use it just like any other component.

### Dynamic styling based on props
```jsx
const Button = styled.button`
  padding: 10px 20px;
  background-color: ${(props) => (props.primary ? "#3b82f6" : "#e5e7eb")};
  color: ${(props) => (props.primary ? "white" : "#111827")};
`;

<Button primary>Primary Button</Button>
<Button>Secondary Button</Button>
```

### Why it's used
- **Automatic scoping:** Styles are automatically scoped to the component via generated unique class names — no risk of CSS class name collisions across the app (a common problem in large plain-CSS codebases).
- **Dynamic styling with JS:** Since styles are written in JavaScript, you can use props, conditionals, and functions directly within your CSS — something plain CSS can't do natively.
- **Colocation:** Styles live right next to the component that uses them, making components more self-contained and portable.

### Trade-offs
- Adds a runtime cost (styles are generated/injected via JavaScript at runtime, though some tools support build-time extraction) — can impact performance slightly compared to plain static CSS.
- Ties your styling to a specific library/syntax rather than standard CSS, which some teams see as an added dependency and learning curve.

---

## 7. CSS Modules

**Simple definition:** A CSS authoring approach where you write **regular CSS files**, but each class name is **automatically scoped locally** to the component that imports it — preventing global class name collisions, while still writing plain, standard CSS syntax.

```css
/* Button.module.css */
.button {
  padding: 10px 20px;
  background-color: #3b82f6;
  color: white;
  border-radius: 6px;
}

.primary {
  background-color: #1d4ed8;
}
```

```jsx
// Button.jsx
import styles from "./Button.module.css";

function Button({ primary }) {
  return (
    <button className={`${styles.button} ${primary ? styles.primary : ""}`}>
      Click Me
    </button>
  );
}
```

### What actually happens under the hood
When the build tool processes `Button.module.css`, it transforms `.button` into something unique like `.Button_button__a1b2c`, and `styles.button` in your JS resolves to that exact generated class name — guaranteeing no collision with a `.button` class defined anywhere else in the app, even if another file also has a class literally named `.button`.

### Why it's used
- **Local scoping without a new syntax/library:** You write plain, familiar CSS — no special templating syntax to learn (unlike Styled Components' tagged template literals).
- **No runtime cost:** Unlike CSS-in-JS libraries, CSS Modules are processed entirely at **build time** — the output is just plain CSS files, so there's no JavaScript runtime overhead for styling.
- **Prevents the classic "global CSS collision" problem** that plagues large codebases using plain, un-scoped CSS files.

### CSS Modules vs Styled Components vs Tailwind — comparison
| | Tailwind CSS | Styled Components | CSS Modules |
|---|---|---|---|
| Approach | Utility classes in markup | CSS written inside JS, scoped per component | Plain CSS files, auto-scoped per component |
| Runtime cost | None (pure CSS output) | Some (styles generated/injected via JS) | None (pure CSS output) |
| Dynamic styling based on props | Via conditional class strings | Native — directly in the CSS via JS | Via conditional class strings |
| Learning curve | Need to learn utility class names | Need to learn tagged template literal syntax | Minimal — just plain CSS |
| Best for | Rapid UI development, consistent design systems | Component libraries needing heavy dynamic/prop-based styling | Teams wanting scoped CSS without a CSS-in-JS runtime cost |

---

## 8. Box Model

**Simple definition:** Every HTML element is rendered as a rectangular box made of four layers, from inside out: **content → padding → border → margin**. Understanding this is fundamental to controlling spacing and sizing.

```
┌─────────────────────────────┐
│           margin             │  ← space outside the border (transparent)
│   ┌─────────────────────┐   │
│   │        border         │   │
│   │  ┌─────────────────┐ │   │
│   │  │     padding       │ │   │
│   │  │  ┌─────────────┐ │ │   │
│   │  │  │   content    │ │ │   │
│   │  │  └─────────────┘ │ │   │
│   │  └─────────────────┘ │   │
│   └─────────────────────┘   │
└─────────────────────────────┘
```

```css
.box {
  width: 200px;
  padding: 20px;   /* space between content and border */
  border: 2px solid black;
  margin: 10px;    /* space outside the border, between this and other elements */
}
```

### `box-sizing` — the most important gotcha
```css
.box {
  box-sizing: content-box; /* DEFAULT — width/height apply ONLY to content;
                                padding & border are added ON TOP, making the box bigger than specified */
  box-sizing: border-box;  /* width/height INCLUDE padding & border;
                                the box stays exactly the size you specify */
}
```
**Best practice:** Most modern CSS resets set `box-sizing: border-box` globally, since it makes sizing far more predictable — `width: 200px` actually means the box is 200px wide, no matter how much padding/border you add.
```css
* {
  box-sizing: border-box;
}
```

---

## 9. Position

**Simple definition:** The `position` property controls **how an element is placed** in the document — whether it flows normally, or is deliberately taken out of the normal flow and placed relative to something else.

| Value | Behavior |
|---|---|
| `static` | Default — follows normal document flow; `top`/`left`/etc. have no effect |
| `relative` | Stays in normal flow, but can be nudged from its original position using `top`/`left`/`right`/`bottom` — and becomes a positioning reference for `absolute` children |
| `absolute` | Removed from normal flow entirely; positioned relative to its nearest **positioned** ancestor (`relative`/`absolute`/`fixed`), or the document if none exists |
| `fixed` | Removed from normal flow; positioned relative to the **viewport** — stays in place even when scrolling |
| `sticky` | Behaves like `relative` until a scroll threshold is met, then "sticks" like `fixed` within its container |

```css
.parent {
  position: relative; /* establishes a positioning context for children */
}

.tooltip {
  position: absolute;
  top: 100%;    /* positioned relative to .parent, not the whole page */
  left: 0;
}

.navbar {
  position: sticky;
  top: 0; /* sticks to the top of the viewport once scrolled to */
}

.modal-overlay {
  position: fixed;
  top: 0; left: 0; right: 0; bottom: 0; /* covers the entire viewport, stays put on scroll */
}
```
**Classic pattern — `relative` parent + `absolute` child:** This is the most common positioning combo, used for things like tooltips, dropdown menus, and badges anchored to a specific parent element rather than the whole page.

---

## 10. z-index & Stacking Context

**Simple definition:** `z-index` controls which elements appear **in front of** or **behind** others when they overlap — higher values sit on top. It only works on elements with a `position` value other than `static`.

```css
.modal {
  position: fixed;
  z-index: 1000; /* sits above most other content */
}

.dropdown {
  position: absolute;
  z-index: 10;
}
```

**Common gotcha — stacking contexts:** `z-index` values are only compared **within the same stacking context**. Certain CSS properties (e.g., `opacity < 1`, `transform`, `filter`, `position: fixed`) create a **new stacking context** on an element, meaning its children's `z-index` values are trapped inside it and can't compete directly with `z-index` values outside that context — even a child with `z-index: 9999` can't escape above a sibling outside its parent's stacking context if that parent has a lower z-index.

**Practical tip:** If a high `z-index` "isn't working," the real problem is often that the element is stuck inside a stacking context created by an ancestor — the fix is usually to adjust the ancestor's `z-index`/positioning, not just keep raising the child's value.

---

## 11. Specificity & the Cascade

**Simple definition:** When multiple CSS rules target the same element with conflicting styles, **specificity** determines which rule "wins." The "C" in CSS (Cascading) refers to this exact resolution process.

### Specificity hierarchy (highest to lowest)
| Selector type | Example | Specificity weight |
|---|---|---|
| Inline styles | `style="color: red;"` | Highest (1000) |
| ID selectors | `#header` | 100 |
| Class, attribute, pseudo-class selectors | `.button`, `[type="text"]`, `:hover` | 10 |
| Element/type selectors | `div`, `p` | 1 |
| Universal selector | `*` | 0 |

```css
p { color: blue; }              /* specificity: 1 */
.text { color: green; }         /* specificity: 10 — wins over the above */
#main .text { color: red; }     /* specificity: 110 — wins over both above */
```
**`!important`** overrides normal specificity entirely (highest priority possible) — but should be used very sparingly, since it makes styles harder to override later and is often a sign of a specificity problem elsewhere that should be fixed properly instead.

### The cascade order (when specificity is equal)
When specificity ties, CSS falls back to **source order** — the rule that appears **later** in the stylesheet (or is loaded later) wins.
```css
.button { color: blue; }
.button { color: red; } /* same specificity, but comes later — red wins */
```

---

## 12. Common Selectors, Pseudo-classes & Pseudo-elements

**Simple definition:** Selectors target *which* elements a rule applies to. Pseudo-classes target elements in a particular **state**; pseudo-elements target a specific **part** of an element that isn't a real, separate DOM node.

### Combinators
```css
.parent > .child   { }  /* direct children only */
.parent .descendant { } /* any descendant, at any depth */
.item + .item       { }  /* the element immediately after a sibling */
.item ~ .sibling     { }  /* any sibling that comes after */
```

### Pseudo-classes (state-based)
```css
a:hover { color: red; }              /* when the mouse is over it */
input:focus { border-color: blue; }  /* when the element has focus */
li:first-child { font-weight: bold; } /* the first child among siblings */
li:nth-child(2n) { background: #eee; } /* every even-numbered child (zebra striping) */
button:disabled { opacity: 0.5; }     /* when disabled */
```

### Pseudo-elements (targeting a "part" of an element)
```css
p::first-line { font-weight: bold; }   /* styles just the first line of text */
.quote::before { content: "“"; }        /* inserts generated content before the element */
.quote::after  { content: "”"; }        /* inserts generated content after the element */
```
**Note:** Pseudo-elements use double colons (`::`) in modern CSS syntax to distinguish them from pseudo-classes (`:`), though single-colon syntax for the older pseudo-elements still works for backward compatibility.

---

## 13. CSS Custom Properties (CSS Variables)

**Simple definition:** CSS variables let you define **reusable values** (colors, spacing, fonts) once, referenced everywhere via `var()` — making global design changes (like re-theming a site) a one-line edit instead of a find-and-replace across every file.

```css
:root {
  --primary-color: #3b82f6;
  --spacing-unit: 8px;
  --font-heading: "Inter", sans-serif;
}

.button {
  background-color: var(--primary-color);
  padding: calc(var(--spacing-unit) * 2);
}

.card {
  border: 1px solid var(--primary-color);
}
```
- Defined on `:root` (the `<html>` element) to make them globally available, but they can also be scoped to a specific component/container by defining them on a more specific selector.
- Unlike Sass/Less variables (which are compiled away at build time), CSS custom properties are **live in the browser** — they can be read and changed dynamically with JavaScript, and even updated based on media queries or user interaction (e.g., toggling dark mode).

```js
// Changing a CSS variable dynamically with JavaScript — enables live theming
document.documentElement.style.setProperty("--primary-color", "#10b981");
```

```css
/* Dark mode toggle using CSS variables */
:root {
  --bg-color: white;
  --text-color: black;
}
[data-theme="dark"] {
  --bg-color: #111827;
  --text-color: white;
}
body {
  background-color: var(--bg-color);
  color: var(--text-color);
}
```

---

## 14. Transitions & Animations

**Simple definition:** These let elements change styles **smoothly over time** instead of snapping instantly — improving perceived polish and giving users visual feedback for interactions.

### Transitions — for simple state changes (e.g., hover)
```css
.button {
  background-color: #3b82f6;
  transition: background-color 0.3s ease, transform 0.2s ease;
}

.button:hover {
  background-color: #2563eb;
  transform: scale(1.05);
}
```
- `transition: <property> <duration> <timing-function>` — smoothly animates a property change whenever it happens (e.g., on `:hover`, class toggle, etc.), rather than needing explicit keyframes.

### Animations — for more complex, multi-step, or looping sequences
```css
@keyframes spin {
  from { transform: rotate(0deg); }
  to   { transform: rotate(360deg); }
}

.loader {
  animation: spin 1s linear infinite;
}
```
```css
@keyframes fadeInUp {
  0%   { opacity: 0; transform: translateY(20px); }
  100% { opacity: 1; transform: translateY(0); }
}

.card {
  animation: fadeInUp 0.5s ease-out;
}
```

### Transitions vs Animations
| | Transitions | Animations |
|---|---|---|
| Trigger | Requires a state change (hover, class toggle, etc.) | Can run automatically on load, loop indefinitely |
| Steps | Only interpolates between a start and end state | Supports multiple keyframe steps (0%, 25%, 50%...) |
| Complexity | Simple, quick effects | Complex, multi-stage sequences |

**Performance tip:** Animating `transform` and `opacity` is much cheaper for the browser than animating properties like `width`, `height`, `top`, or `left` — the former can often be handled by the GPU without triggering expensive layout recalculations, while the latter forces the browser to recompute the layout of surrounding elements on every frame.

---

## 15. CSS Units

**Simple definition:** Units determine how a value (width, font size, spacing) is measured — choosing the right unit is key to building layouts that scale and remain accessible.

| Unit | Type | Relative to | Common use |
|---|---|---|---|
| `px` | Absolute | Fixed pixel value | Borders, precise small values |
| `%` | Relative | Parent element's size | Fluid widths |
| `em` | Relative | The **current element's** font size (or parent's, for non-font properties) | Spacing that scales with local text size |
| `rem` | Relative | The **root** (`<html>`) font size | Font sizes, spacing — predictable, doesn't compound like `em` |
| `vw` / `vh` | Relative | 1% of the viewport's width / height | Full-screen sections, responsive typography |
| `fr` | Relative | A fraction of available space (Grid only) | Flexible grid columns/rows |

```css
html { font-size: 16px; } /* the root reference for rem */

.title {
  font-size: 2rem;   /* always 32px (2 × 16px root), regardless of nesting */
}

.nested-box {
  font-size: 1.5em;  /* 1.5 × PARENT's font size — compounds if nested repeatedly, can get unpredictable */
}

.hero {
  height: 100vh; /* always exactly the full viewport height */
}
```
**Best practice:** Prefer `rem` for font sizes and spacing (predictable, scales consistently if the user changes their browser's base font size for accessibility), `%`/`fr` for flexible widths, and `px` sparingly for things that genuinely shouldn't scale (like a 1px border).

---

## 16. BEM Naming Convention

**Simple definition:** BEM (**Block, Element, Modifier**) is a naming methodology for writing predictable, collision-resistant CSS class names in plain CSS projects (most relevant when NOT using Tailwind/CSS Modules/Styled Components, which each solve the naming/scoping problem differently).

```
.block__element--modifier
```
- **Block:** A standalone, reusable component (`card`, `nav`, `button`).
- **Element:** A part of that block, connected with `__` (`card__title`, `nav__item`).
- **Modifier:** A variation of the block or element, connected with `--` (`button--primary`, `card--highlighted`).

```html
<div class="card card--highlighted">
  <h2 class="card__title">Title</h2>
  <p class="card__description">Description text</p>
  <button class="card__button card__button--primary">Buy Now</button>
</div>
```
```css
.card { padding: 16px; border-radius: 8px; }
.card--highlighted { border: 2px solid gold; }
.card__title { font-size: 1.25rem; }
.card__button--primary { background-color: blue; color: white; }
```
**Why it helps:** Without a convention, plain CSS class names easily collide across a large codebase (two different developers both using `.title`, with unpredictable results depending on load order/specificity). BEM's structure makes relationships between elements clear from the class name alone, and naturally avoids collisions since each class is namespaced under its block.

---

## Quick Summary Cheat Sheet

| Concept | One-liner |
|---|---|
| Flexbox | One-dimensional layout for aligning/distributing items in a row or column |
| Grid | Two-dimensional layout for controlling rows and columns together |
| Responsive Design | Adapting layout/styles across different screen sizes via fluid units and media queries |
| Tailwind CSS | Utility-first framework — compose pre-built classes directly in markup |
| Styled Components | CSS-in-JS — write scoped CSS directly inside your JavaScript/React components |
| CSS Modules | Plain CSS files with automatic local scoping per component, no runtime cost |
| Box Model | Content → padding → border → margin; `box-sizing: border-box` makes sizing predictable |
| Position | Controls how an element is placed: static, relative, absolute, fixed, sticky |
| z-index & Stacking Context | Controls overlap order; only compares within the same stacking context |
| Specificity & Cascade | Determines which conflicting rule wins: inline > ID > class > element |
| Selectors/Pseudo-classes/elements | Target elements by relationship (`>`, `~`), state (`:hover`), or sub-part (`::before`) |
| CSS Variables | Reusable, live-updatable values via `--name` and `var()` |
| Transitions & Animations | Smoothly animate style changes over time; prefer `transform`/`opacity` for performance |
| CSS Units | `rem` for predictable scalable sizing, `%`/`fr` for flexible widths, `vh`/`vw` for viewport-based sizing |
| BEM | `.block__element--modifier` naming convention to avoid class name collisions in plain CSS |

## Quick Decision Guide

- **Building a page layout?** → CSS Grid.
- **Aligning items within a section (navbar, card, button group)?** → Flexbox.
- **Need it to look good on all devices?** → Combine fluid units + media queries + Grid/Flexbox's built-in flexibility (mobile-first approach).
- **Want to move fast with a consistent design system, comfortable with utility classes in markup?** → Tailwind CSS.
- **Need heavy dynamic, prop-based styling logic tightly coupled to components?** → Styled Components.
- **Want scoped CSS with zero runtime cost, using familiar plain CSS syntax?** → CSS Modules.
