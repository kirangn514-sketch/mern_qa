# Next.js Complete Notes

---

## 1. What is Next.js?

**Simple definition:** Next.js is a **React framework** — it takes plain React (which only handles the UI/view layer) and adds everything needed to build a full production app: routing, server-side rendering, API endpoints, image optimization, and more.

**Why it exists:** React by itself is just a UI library. It doesn't tell you how to route between pages, how to fetch data on the server, or how to optimize your app for search engines. Next.js fills these gaps with built-in conventions and tooling.

---

## 2. App Router

**Simple definition:** The App Router is Next.js's modern routing system (introduced in Next.js 13, stable since 13.4+) based on the `app/` folder. It replaces the older `pages/` folder system (Pages Router) and is built around **React Server Components**.

### `app` folder
- The root of the App Router. Every folder inside `app/` represents a **route segment**, and the URL path is determined by the folder structure.
- Example:
  ```
  app/
    page.tsx          → /
    about/page.tsx    → /about
    blog/[slug]/page.tsx → /blog/:slug
  ```
- Special files (`layout.tsx`, `page.tsx`, `loading.tsx`, etc.) inside these folders give each route special behavior.

### `layout.tsx`
- **Definition:** Defines shared UI that **wraps** a page or a group of pages (e.g., navbar, sidebar, footer) and **persists across navigations** — it doesn't re-render when you move between child pages.
- Every app needs a **root layout** (`app/layout.tsx`) containing `<html>` and `<body>` tags.
- Layouts can be nested — each folder can have its own `layout.tsx` that wraps only the pages inside it.

```tsx
// app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <nav>My Navbar</nav>
        {children}
      </body>
    </html>
  );
}
```

### `page.tsx`
- **Definition:** Defines the actual **unique UI** for a route — this is what makes a folder publicly accessible as a URL. Without `page.tsx`, a folder is not a routable page.

```tsx
// app/about/page.tsx
export default function AboutPage() {
  return <h1>About Us</h1>;
}
```

### `loading.tsx`
- **Definition:** An automatic **loading UI** shown instantly while the content of `page.tsx` (and its data) is being fetched. Built on top of React Suspense.
- No manual `if (loading)` state needed — Next.js shows this file automatically during navigation.

```tsx
// app/dashboard/loading.tsx
export default function Loading() {
  return <p>Loading dashboard...</p>;
}
```

### `error.tsx`
- **Definition:** An automatic **error boundary UI** for a route segment. If anything throws inside that segment (rendering error, data fetching error), this file is shown instead of crashing the whole app.
- Must be a **Client Component** (`"use client"`), since error boundaries rely on React state.

```tsx
// app/dashboard/error.tsx
"use client";
export default function Error({ error, reset }: { error: Error; reset: () => void }) {
  return (
    <div>
      <p>Something went wrong: {error.message}</p>
      <button onClick={() => reset()}>Try again</button>
    </div>
  );
}
```

### `not-found.tsx`
- **Definition:** Shown when a route doesn't exist, or when you manually call the `notFound()` function (e.g., a product ID that doesn't exist in the database).

```tsx
// app/not-found.tsx
export default function NotFound() {
  return <h2>404 - Page Not Found</h2>;
}
```

```tsx
// Triggering it manually inside a page
import { notFound } from "next/navigation";

if (!product) notFound();
```

### `route.ts`
- **Definition:** Used to create **API endpoints** (Route Handlers) inside the `app/` folder, replacing the old `pages/api` folder. You export functions named after HTTP methods.

```tsx
// app/api/users/route.ts
import { NextResponse } from "next/server";

export async function GET() {
  return NextResponse.json({ users: ["Alice", "Bob"] });
}

export async function POST(request: Request) {
  const body = await request.json();
  return NextResponse.json({ created: body }, { status: 201 });
}
```
- Accessible at `/api/users`.

---

## 3. Rendering Strategies

**Simple definition:** Rendering strategy = **when and where** your HTML is generated: on the server ahead of time, on the server per-request, or in the browser.

### SSR (Server-Side Rendering)
- HTML is generated **on the server for every request**, then sent to the browser.
- **Best for:** pages with frequently changing, per-user, or personalized data (e.g., a logged-in dashboard).
- **Example:** A user profile page that must always show live data.
```tsx
// app/profile/page.tsx (SSR by default when using dynamic data)
async function getProfile() {
  const res = await fetch("https://api.example.com/profile", { cache: "no-store" });
  return res.json();
}

export default async function ProfilePage() {
  const profile = await getProfile();
  return <div>{profile.name}</div>;
}
```
- `cache: "no-store"` forces a fresh fetch on every request → SSR behavior.

### CSR (Client-Side Rendering)
- The browser downloads a mostly empty HTML shell + JavaScript, then JavaScript fetches data and renders content **in the browser**.
- **Best for:** highly interactive, dashboard-like widgets where SEO doesn't matter (e.g., an admin analytics panel).
- **Example:**
```tsx
"use client";
import { useEffect, useState } from "react";

export default function Notifications() {
  const [data, setData] = useState(null);

  useEffect(() => {
    fetch("/api/notifications")
      .then((res) => res.json())
      .then(setData);
  }, []);

  return <div>{data ? data.length : "Loading..."}</div>;
}
```

### SSG (Static Site Generation)
- HTML is generated **once, at build time**, and reused for every request (served from CDN — extremely fast).
- **Best for:** content that rarely changes (e.g., blog posts, marketing pages, docs).
- **Example:**
```tsx
// app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await fetch("https://api.example.com/posts").then((r) => r.json());
  return posts.map((post: any) => ({ slug: post.slug }));
}

async function getPost(slug: string) {
  const res = await fetch(`https://api.example.com/posts/${slug}`); // default cache: "force-cache" → SSG
  return res.json();
}

export default async function BlogPost({ params }: { params: { slug: string } }) {
  const post = await getPost(params.slug);
  return <h1>{post.title}</h1>;
}
```

### ISR (Incremental Static Regeneration)
- Like SSG, but pages are **re-generated automatically in the background** after a set time interval, without rebuilding the whole app.
- **Best for:** content that changes occasionally (e.g., product prices, news articles updated every few minutes).
- **Example:**
```tsx
async function getProduct(id: string) {
  const res = await fetch(`https://api.example.com/products/${id}`, {
    next: { revalidate: 60 }, // regenerate at most once every 60 seconds
  });
  return res.json();
}
```

### Quick Comparison Table

| Feature          | SSR                        | CSR                     | SSG                        | ISR                              |
|------------------|-----------------------------|--------------------------|------------------------------|-----------------------------------|
| When HTML is built | Every request (server)    | In browser after load    | Once at build time           | At build time + periodic refresh |
| Speed to user    | Medium                      | Slow first load          | Fastest                      | Fast (like SSG)                  |
| SEO              | Good                        | Poor                     | Excellent                    | Excellent                        |
| Data freshness   | Always fresh                | Always fresh             | Stale until next build       | Fresh within revalidation window |
| Use case         | Dashboards, personalized data | Highly interactive UI  | Blogs, marketing pages       | Product pages, news feeds        |

---

## 4. Server Components vs Client Components

### Server Components
- **Simple definition:** Components that render **only on the server**. Their JavaScript is never sent to the browser — only the resulting HTML is.
- **Default in App Router** — every component is a Server Component unless marked otherwise.
- **Benefits:**
  - Smaller JavaScript bundle sent to the browser (faster page loads).
  - Can directly access backend resources (databases, file system, secret API keys) safely.
  - Better SEO and initial load performance.
- **Limitations:** Cannot use browser-only features like `useState`, `useEffect`, `onClick`, or `window`.

```tsx
// app/products/page.tsx — Server Component (no directive needed)
import { db } from "@/lib/db";

export default async function ProductsPage() {
  const products = await db.product.findMany(); // safe to query DB directly
  return (
    <ul>
      {products.map((p) => <li key={p.id}>{p.name}</li>)}
    </ul>
  );
}
```

### Client Components
- **Simple definition:** Components that render in the **browser** and can use interactivity (state, effects, event handlers, browser APIs).
- Marked explicitly with the `"use client"` directive at the top of the file.

```tsx
"use client";

import { useState } from "react";

export default function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}
```

### `"use client"`
- **Definition:** A directive placed at the **top of a file** that tells Next.js "this component and everything it imports should be bundled and run in the browser."
- Needed whenever you use: `useState`, `useEffect`, `useContext`, event handlers (`onClick`, `onChange`), or browser-only APIs (`window`, `localStorage`).
- **Rule of thumb:** Keep Server Components for data fetching / static content, and push `"use client"` down to only the small interactive parts (buttons, forms, toggles) — this is called "pushing client components to the leaves."

| Aspect              | Server Component            | Client Component            |
|---------------------|------------------------------|------------------------------|
| Runs on             | Server only                  | Browser (after hydration)    |
| Can use `useState`  | ❌                            | ✅                            |
| Can access DB/secrets directly | ✅                 | ❌                            |
| Sent as JS bundle   | No (only HTML)                | Yes                          |
| Default in App Router | ✅ Yes                      | ❌ Needs `"use client"`       |

---

## 5. Data Fetching

**Simple definition:** In the App Router, you fetch data directly inside **async Server Components** using a supercharged version of the native `fetch()` — no `getServerSideProps` or `getStaticProps` needed anymore.

### `fetch()`
- Works natively inside Server Components as `async/await`.
```tsx
export default async function Page() {
  const res = await fetch("https://api.example.com/data");
  const data = await res.json();
  return <div>{data.title}</div>;
}
```

### `cache`
- **Definition:** Next.js extends `fetch()` with a built-in caching layer. Depending on the `cache` option, the result is stored and reused:
  - `cache: "force-cache"` (default) → cached indefinitely, used for **SSG**.
  - `cache: "no-store"` → never cached, fetched fresh every time, used for **SSR**.

```tsx
// SSG behavior (default)
fetch("https://api.example.com/data");

// SSR behavior
fetch("https://api.example.com/data", { cache: "no-store" });
```

### `revalidate`
- **Definition:** Controls **how often** a cached fetch should be refreshed — this is what powers **ISR**.
- Can be set per-fetch or per-route.

```tsx
// Per fetch call
fetch("https://api.example.com/data", { next: { revalidate: 60 } }); // refresh every 60s

// Per route segment (in page.tsx)
export const revalidate = 60;
```
- You can also trigger on-demand revalidation using `revalidatePath()` or `revalidateTag()` (e.g., after a form submission updates data).

```tsx
import { revalidatePath } from "next/cache";

export async function updatePost() {
  "use server";
  // ...update logic
  revalidatePath("/blog");
}
```

---

## 6. Routing

### Dynamic Routes
- **Definition:** Routes with a variable segment, defined using square brackets `[param]`.
```
app/blog/[slug]/page.tsx  →  /blog/hello-world  (slug = "hello-world")
```
```tsx
export default function BlogPost({ params }: { params: { slug: string } }) {
  return <h1>Post: {params.slug}</h1>;
}
```

### Nested Routes
- **Definition:** Routes are automatically nested based on **folder structure** — a folder inside a folder creates a sub-route, and layouts can nest along with them.
```
app/dashboard/settings/page.tsx  →  /dashboard/settings
```

### Catch-all Routes
- **Definition:** Match **multiple path segments** using `[...param]` (catches one or more segments) or `[[...param]]` (catches zero or more — optional catch-all).
```
app/docs/[...slug]/page.tsx
```
| URL                     | `params.slug`            |
|--------------------------|---------------------------|
| `/docs/a`                | `["a"]`                   |
| `/docs/a/b`              | `["a", "b"]`               |
| `/docs/a/b/c`            | `["a", "b", "c"]`           |

```tsx
export default function Docs({ params }: { params: { slug: string[] } }) {
  return <p>Path: {params.slug.join("/")}</p>;
}
```

---

## 7. Middleware

**Simple definition:** Code that runs **before a request is completed** — intercepting the request at the edge, before it hits your page or API route. Useful for auth checks, redirects, logging, A/B testing, and localization.

- File must be named `middleware.ts` at the **root** of the project (same level as `app/`).

```tsx
// middleware.ts
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  const token = request.cookies.get("token");

  if (!token && request.nextUrl.pathname.startsWith("/dashboard")) {
    return NextResponse.redirect(new URL("/login", request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/dashboard/:path*"],
};
```

---

## 8. Authentication

**Simple definition:** Verifying who a user is (login) and controlling what routes/data they can access. Next.js doesn't include built-in auth — you typically combine it with libraries like **NextAuth.js (Auth.js)**, **Clerk**, or **Supabase Auth**.

**Common pattern:**
1. Middleware checks for a valid session/JWT token on protected routes.
2. Server Components fetch the session and conditionally render UI.
3. Client Components handle login/logout forms (interactive).

```tsx
// Example: checking session in a Server Component
import { getServerSession } from "next-auth";

export default async function Dashboard() {
  const session = await getServerSession();
  if (!session) return <p>Please log in.</p>;
  return <p>Welcome, {session.user.name}</p>;
}
```

---

## 9. Image Optimization

**Simple definition:** Next.js provides a built-in `<Image />` component that automatically optimizes images — resizing, compressing, converting to modern formats (WebP), lazy loading, and preventing layout shift.

```tsx
import Image from "next/image";

export default function Avatar() {
  return (
    <Image
      src="/profile.jpg"
      alt="Profile picture"
      width={200}
      height={200}
      priority // loads eagerly for above-the-fold images
    />
  );
}
```
- **Why it matters:** Regular `<img>` tags don't optimize file size or loading behavior — `<Image>` does this automatically, improving Core Web Vitals (LCP, CLS).

---

## 10. SEO

**Simple definition:** Search Engine Optimization — making sure your pages are easily crawlable and display rich, correct information (titles, descriptions, previews) in search engines and social media.

Next.js helps with SEO because:
- Server-rendered/static HTML is crawlable immediately (unlike CSR-only apps).
- Built-in **Metadata API** (see below) makes it easy to set titles, descriptions, Open Graph tags.
- Automatic `sitemap.xml` and `robots.txt` generation support.

```tsx
// app/sitemap.ts
import { MetadataRoute } from "next";

export default function sitemap(): MetadataRoute.Sitemap {
  return [
    { url: "https://example.com", lastModified: new Date() },
    { url: "https://example.com/about", lastModified: new Date() },
  ];
}
```

---

## 11. Metadata API

**Simple definition:** A built-in way to define `<title>`, `<meta>` tags, Open Graph data, and more — either statically or dynamically — without manually writing `<head>` tags.

### Static metadata
```tsx
// app/about/page.tsx
export const metadata = {
  title: "About Us | My Company",
  description: "Learn more about our mission and team.",
};

export default function About() {
  return <h1>About</h1>;
}
```

### Dynamic metadata
```tsx
// app/blog/[slug]/page.tsx
export async function generateMetadata({ params }: { params: { slug: string } }) {
  const post = await getPost(params.slug);
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: { images: [post.coverImage] },
  };
}
```

---

## 12. Environment Variables

**Simple definition:** Values (API keys, DB URLs, secrets) stored outside your code, loaded via `.env` files, so sensitive data isn't hardcoded or committed to version control.

- `.env.local` → local secrets (not committed to git).
- Server-only variables: accessible only in Server Components / API routes.
  ```
  DATABASE_URL=postgres://user:pass@localhost/db
  ```
- Client-exposed variables: **must** be prefixed with `NEXT_PUBLIC_` to be accessible in the browser.
  ```
  NEXT_PUBLIC_API_URL=https://api.example.com
  ```
- Usage:
```tsx
const dbUrl = process.env.DATABASE_URL;         // server only
const apiUrl = process.env.NEXT_PUBLIC_API_URL; // available in browser too
```
- ⚠️ Never prefix real secrets with `NEXT_PUBLIC_` — anything with that prefix is bundled into client-side JavaScript and visible to anyone.

---

## 13. Deployment

**Simple definition:** Publishing your Next.js app so it's live on the internet.

- **Vercel** (made by the creators of Next.js) — zero-config deployment, automatic SSR/ISR/edge support, git-based deploys.
- **Other options:** Docker containers on AWS/Azure/GCP, Netlify, self-hosted Node.js server (`next start`), or static export (`next export`) for fully static sites (no SSR/ISR features in that case).

**Basic production flow:**
```bash
npm run build   # creates optimized production build
npm run start   # starts the Node.js production server
```

---

## 14. Interview Questions & Answers

### Q1. SSR vs CSR?
- **SSR:** HTML generated on the server per request; great for SEO and fast first paint of dynamic content; slightly slower Time-to-First-Byte since the server does work each request.
- **CSR:** HTML shell is empty, JS renders content in the browser; great for highly interactive apps but poor SEO and slower initial content visibility (users see a blank/loading screen first).

### Q2. SSG vs ISR?
- **SSG:** Pages built once at build time; fastest possible delivery; but content becomes **stale** until the next full rebuild.
- **ISR:** Same as SSG but pages **automatically regenerate** after a specified `revalidate` time, giving near-static speed with periodically fresh data — no full rebuild needed.

### Q3. App Router vs Pages Router?
| Aspect | Pages Router (`pages/`) | App Router (`app/`) |
|---|---|---|
| Introduced | Original Next.js routing | Next.js 13+ (stable 13.4+) |
| Component model | Client Components only (by default) | Server Components by default |
| Data fetching | `getServerSideProps`, `getStaticProps` | `fetch()` directly in async components |
| Layouts | Manual, repeated per page | Native nested `layout.tsx` |
| Loading/Error UI | Manual implementation | Built-in `loading.tsx` / `error.tsx` |
| Recommended for new projects | No (legacy) | Yes |

### Q4. What are Server Components?
Components that render entirely on the server, send zero JS to the client for that component, and can directly access backend resources (DB, filesystem, secrets). Default in the App Router.

### Q5. What are Client Components?
Components marked with `"use client"` that run in the browser, enabling interactivity — state, event handlers, effects, and browser APIs.

### Q6. Why Next.js instead of plain React?
- React alone has no built-in routing, no SSR/SSG, no image optimization, no API route handling, and no SEO tooling.
- Next.js provides all of this out of the box: file-based routing, hybrid rendering (SSR/SSG/ISR/CSR), built-in performance optimizations (`<Image>`, code splitting, font optimization), API routes, and first-class SEO support (Metadata API, sitemaps).
- Result: faster development, better performance by default, and better SEO — without assembling a dozen separate tools yourself.

---

## Quick Summary Cheat Sheet

| Concept | One-liner |
|---|---|
| App Router | File-based router using the `app/` folder and Server Components |
| `layout.tsx` | Shared, persistent UI wrapper for a route segment |
| `page.tsx` | The unique content for a route |
| `loading.tsx` | Auto Suspense-based loading UI |
| `error.tsx` | Auto error boundary UI |
| `not-found.tsx` | 404 UI |
| `route.ts` | API endpoint handler |
| SSR | Render HTML per request on the server |
| CSR | Render HTML in the browser via JS |
| SSG | Render HTML once at build time |
| ISR | Static + periodic background regeneration |
| Server Component | Renders on server, no JS shipped, default |
| Client Component | Renders in browser, needs `"use client"`, for interactivity |
| `cache`/`revalidate` | Control freshness of `fetch()` data |
| Middleware | Code that runs before a request completes (auth, redirects) |
| Metadata API | Declarative way to set SEO tags |
