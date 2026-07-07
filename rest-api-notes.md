# REST API Complete Notes

---

## 1. What is a REST API?

**Simple definition:** REST (**Representational State Transfer**) is an architectural style for designing networked applications. A REST API is a web service that follows REST's rules, letting clients (browsers, mobile apps, other servers) interact with server **resources** (users, products, orders, etc.) using standard HTTP methods.

**Key idea:** Everything is treated as a **resource**, identified by a **URL**, and you interact with it using standard HTTP verbs (GET, POST, PUT, DELETE...). The server doesn't remember anything about the client between requests — each request contains everything needed to understand it (this is called **statelessness**, explained below).

```
GET    /users        → get all users
GET    /users/5      → get user with id 5
POST   /users        → create a new user
PUT    /users/5      → replace user 5 entirely
DELETE /users/5      → delete user 5
```

---

## 2. CRUD APIs

**Simple definition:** CRUD stands for **Create, Read, Update, Delete** — the four basic operations you can perform on any data/resource. Almost every REST API is built around mapping these operations to HTTP methods and endpoints.

| Operation | HTTP Method | Example Endpoint | Description |
|---|---|---|---|
| Create | `POST` | `POST /users` | Add a new resource |
| Read | `GET` | `GET /users` or `GET /users/5` | Retrieve resource(s) |
| Update | `PUT` / `PATCH` | `PUT /users/5` | Modify an existing resource |
| Delete | `DELETE` | `DELETE /users/5` | Remove a resource |

### Example CRUD API (Express.js style)
```js
app.get("/users", getAllUsers);       // Read (all)
app.get("/users/:id", getUserById);   // Read (one)
app.post("/users", createUser);       // Create
app.put("/users/:id", updateUser);    // Update
app.delete("/users/:id", deleteUser); // Delete
```

---

## 3. HTTP Methods

**Simple definition:** HTTP methods (verbs) describe **what action** you want to perform on a resource. Choosing the right method matters for clarity, caching behavior, and REST correctness.

| Method | Purpose | Body? | Safe? | Idempotent? |
|---|---|---|---|---|
| `GET` | Retrieve a resource | No | ✅ Yes | ✅ Yes |
| `POST` | Create a new resource | Yes | ❌ No | ❌ No |
| `PUT` | Replace a resource entirely | Yes | ❌ No | ✅ Yes |
| `PATCH` | Partially update a resource | Yes | ❌ No | ⚠️ Usually, not guaranteed |
| `DELETE` | Remove a resource | No (usually) | ❌ No | ✅ Yes |
| `HEAD` | Like GET, but returns headers only (no body) | No | ✅ Yes | ✅ Yes |
| `OPTIONS` | Returns allowed methods/CORS info for a resource | No | ✅ Yes | ✅ Yes |

- **Safe** = doesn't change server state (read-only).
- **Idempotent** = calling it multiple times has the same effect as calling it once (see [Idempotency](#9-idempotency) section for full explanation).

```js
// Example usage
app.get("/products/:id", getProduct);     // safe, idempotent
app.post("/products", createProduct);     // not idempotent — calling twice creates 2 products
app.put("/products/:id", replaceProduct); // idempotent — same result every time
app.patch("/products/:id", updateField);  // partial update
app.delete("/products/:id", deleteProduct); // idempotent — deleting twice = still deleted
```

---

## 4. Status Codes

**Simple definition:** HTTP status codes are 3-digit numbers returned in every response, telling the client **what happened** with their request — success, client error, or server error.

### Categories
| Range | Category | Meaning |
|---|---|---|
| `1xx` | Informational | Request received, still processing |
| `2xx` | Success | Request succeeded |
| `3xx` | Redirection | Further action needed to complete request |
| `4xx` | Client Error | Something wrong with the request itself |
| `5xx` | Server Error | Something went wrong on the server |

### Most commonly used codes

| Code | Name | When to use |
|---|---|---|
| `200` | OK | Successful GET/PUT/PATCH request |
| `201` | Created | Successful POST that created a new resource |
| `204` | No Content | Successful request with no body to return (e.g., DELETE) |
| `301` | Moved Permanently | Resource URL has permanently changed |
| `304` | Not Modified | Cached version is still valid (conditional GET) |
| `400` | Bad Request | Malformed request, invalid data/syntax |
| `401` | Unauthorized | Missing or invalid authentication credentials |
| `403` | Forbidden | Authenticated, but not allowed to perform this action |
| `404` | Not Found | Resource doesn't exist |
| `405` | Method Not Allowed | HTTP method not supported for this endpoint |
| `409` | Conflict | Request conflicts with current server state (e.g., duplicate email) |
| `422` | Unprocessable Entity | Request is well-formed but fails validation rules |
| `429` | Too Many Requests | Rate limit exceeded |
| `500` | Internal Server Error | Generic server-side failure |
| `503` | Service Unavailable | Server is temporarily down/overloaded |

```js
app.post("/users", (req, res) => {
  if (!req.body.email) {
    return res.status(400).json({ message: "Email is required" });
  }
  // ... create user
  res.status(201).json({ message: "User created" });
});
```

---

## 5. Pagination

**Simple definition:** Pagination breaks up a large collection of results into smaller "pages," instead of returning thousands of records in a single response — improving performance and usability.

### Common approaches

**1. Offset-based pagination** (most common, simplest)
```
GET /products?page=2&limit=20
```
```js
app.get("/products", async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 10;
  const skip = (page - 1) * limit;

  const products = await Product.find().skip(skip).limit(limit);
  const total = await Product.countDocuments();

  res.json({
    data: products,
    pagination: { page, limit, total, totalPages: Math.ceil(total / limit) },
  });
});
```
- **Downside:** Can be slow on very large datasets (database still has to "skip" over rows), and can produce duplicate/missing items if data changes between requests.

**2. Cursor-based pagination** (better for large, real-time datasets)
```
GET /products?cursor=abc123&limit=20
```
- Instead of page numbers, you pass a pointer (usually the last item's ID or timestamp) — the server returns the next batch after that cursor.
- More stable when data is being added/removed frequently (e.g., social media feeds).

```js
app.get("/products", async (req, res) => {
  const { cursor, limit = 10 } = req.query;
  const query = cursor ? { _id: { $gt: cursor } } : {};

  const products = await Product.find(query).limit(Number(limit));
  const nextCursor = products.length ? products[products.length - 1]._id : null;

  res.json({ data: products, nextCursor });
});
```

---

## 6. Filtering

**Simple definition:** Filtering lets clients narrow down results to only the records matching certain criteria, usually via query parameters.

```
GET /products?category=electronics&minPrice=100&maxPrice=500&inStock=true
```

```js
app.get("/products", async (req, res) => {
  const { category, minPrice, maxPrice, inStock } = req.query;
  const filter = {};

  if (category) filter.category = category;
  if (inStock) filter.inStock = inStock === "true";
  if (minPrice || maxPrice) {
    filter.price = {};
    if (minPrice) filter.price.$gte = Number(minPrice);
    if (maxPrice) filter.price.$lte = Number(maxPrice);
  }

  const products = await Product.find(filter);
  res.json(products);
});
```
- **Best practice:** Validate and sanitize filter inputs to prevent invalid queries or injection attacks (never pass raw query params directly into a database query without validation).

---

## 7. Sorting

**Simple definition:** Sorting lets clients control the **order** of returned results, usually via a `sort` query parameter.

```
GET /products?sort=price          → ascending by price
GET /products?sort=-price         → descending by price (- prefix = descending)
GET /products?sort=price,-rating  → multiple sort fields
```

```js
app.get("/products", async (req, res) => {
  const sortParam = req.query.sort || "createdAt";
  // Convert "price,-rating" into Mongoose sort object: { price: 1, rating: -1 }
  const sortObj = {};
  sortParam.split(",").forEach((field) => {
    if (field.startsWith("-")) {
      sortObj[field.substring(1)] = -1;
    } else {
      sortObj[field] = 1;
    }
  });

  const products = await Product.find().sort(sortObj);
  res.json(products);
});
```

---

## 8. Authentication & Authorization (in REST context)

### Authentication
**Simple definition:** Confirms **who** is making the request. Common methods in REST APIs:
- **JWT (JSON Web Tokens):** Stateless, sent via `Authorization: Bearer <token>` header.
- **API Keys:** A static key sent in headers, common for server-to-server APIs.
- **OAuth 2.0:** Delegated auth (e.g., "Login with Google").
- **Session cookies:** Traditional stateful sessions stored server-side.

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

```js
function authenticate(req, res, next) {
  const token = req.headers.authorization?.split(" ")[1];
  if (!token) return res.status(401).json({ message: "Unauthorized" });

  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch {
    res.status(401).json({ message: "Invalid or expired token" });
  }
}
```

### Authorization
**Simple definition:** Confirms **what** an already-authenticated user is allowed to do — typically implemented via roles or permissions.

```js
function authorize(...roles) {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ message: "Forbidden" });
    }
    next();
  };
}

app.delete("/users/:id", authenticate, authorize("admin"), deleteUser);
```

---

## 9. Idempotency

**Simple definition:** An operation is **idempotent** if making the same request multiple times produces the **same result/state** as making it once — no extra side effects from repetition.

**Why it matters:** Networks are unreliable — a client might not receive a response and retry the same request. Idempotent operations are safe to retry; non-idempotent ones are not (e.g., retrying a payment `POST` could charge a customer twice).

| Method | Idempotent? | Explanation |
|---|---|---|
| `GET` | ✅ Yes | Reading data never changes state |
| `PUT` | ✅ Yes | Replacing a resource with the same data repeatedly leaves it in the same final state |
| `DELETE` | ✅ Yes | Deleting an already-deleted resource still results in "it's gone" |
| `PATCH` | ⚠️ Depends | If it says "set status to 'shipped'", repeating is fine (idempotent); if it says "increment quantity by 1", repeating changes the result each time (not idempotent) |
| `POST` | ❌ No | Typically creates a new resource each time — calling it 3 times = 3 new resources |

**Real-world technique:** For non-idempotent operations like payments, APIs often accept an **`Idempotency-Key`** header — the client generates a unique key per logical operation, and the server ensures repeated requests with the same key only get processed once.

```js
app.post("/payments", async (req, res) => {
  const idempotencyKey = req.headers["idempotency-key"];
  const existing = await Payment.findOne({ idempotencyKey });

  if (existing) {
    return res.status(200).json(existing); // return the original result, don't reprocess
  }

  const payment = await processPayment(req.body);
  payment.idempotencyKey = idempotencyKey;
  await payment.save();

  res.status(201).json(payment);
});
```

---

## 10. Versioning

**Simple definition:** API versioning lets you change/improve your API over time **without breaking existing clients** that depend on the older behavior.

### Common strategies

**1. URI versioning** (most common, easiest to understand)
```
GET /api/v1/users
GET /api/v2/users
```

**2. Header versioning**
```
GET /api/users
Headers: Accept-Version: v2
```

**3. Query parameter versioning**
```
GET /api/users?version=2
```

**4. Content negotiation (Accept header)**
```
Accept: application/vnd.myapp.v2+json
```

```js
// URI versioning example
app.use("/api/v1/users", require("./routes/v1/userRoutes"));
app.use("/api/v2/users", require("./routes/v2/userRoutes"));
```
- **Best practice:** URI versioning is the most widely adopted because it's explicit, cacheable, and easy for developers to test directly in a browser or Postman.

---

## 11. Rate Limiting

**Simple definition:** Restricting how many requests a client can make within a certain time window, protecting the API from abuse, brute-force attacks, and overload.

```js
const rateLimit = require("express-rate-limit");

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,                  // limit each IP to 100 requests per window
  standardHeaders: true,     // return rate limit info in RateLimit-* headers
  message: { message: "Too many requests, please try again later." },
});

app.use("/api/", limiter);
```

- When a client exceeds the limit, the server responds with **`429 Too Many Requests`**.
- Rate limit info is often returned in response headers:
  ```
  RateLimit-Limit: 100
  RateLimit-Remaining: 42
  RateLimit-Reset: 1690000000
  ```
- **Stricter limits** are typically applied to sensitive endpoints like `/login` to prevent brute-force password guessing.

---

## Interview Questions & Answers

### Q1. PUT vs PATCH?

| | `PUT` | `PATCH` |
|---|---|---|
| Purpose | Replace the **entire** resource | Partially update **specific fields** |
| Body | Must contain the full resource | Contains only the fields to change |
| Idempotent | ✅ Always | ⚠️ Usually, but not guaranteed |
| Example | `PUT /users/5` with full user object → replaces user 5 completely (missing fields may be wiped/reset) | `PATCH /users/5` with `{ "email": "new@mail.com" }` → updates only the email field |

```js
// PUT — must send the entire object
PUT /users/5
{ "name": "Alice", "email": "alice@mail.com", "age": 30 }

// PATCH — send only what changed
PATCH /users/5
{ "email": "alice-new@mail.com" }
```
**Simple rule to remember:** PUT = "replace the whole thing", PATCH = "change a piece of it."

### Q2. 401 vs 403?

| | `401 Unauthorized` | `403 Forbidden` |
|---|---|---|
| Meaning | "I don't know who you are" | "I know who you are, but you can't do this" |
| Cause | Missing, invalid, or expired credentials | Valid credentials, but insufficient permissions |
| Fix | Log in / provide valid token | Requires different permissions/role — logging in again won't help |
| Example | Accessing `/profile` with no token | A regular user trying to access an admin-only `/admin/dashboard` |

**Simple rule to remember:** 401 = "authenticate first," 403 = "you're authenticated, but not allowed."

### Q3. What are the core REST principles?

1. **Client-Server separation** — the client (UI) and server (data/logic) are independent; either can evolve without affecting the other, as long as the API contract stays consistent.

2. **Statelessness** — every request must contain all the information needed to process it (e.g., auth token). The server does **not** store client session state between requests.

3. **Cacheability** — responses must explicitly indicate whether they can be cached (via headers like `Cache-Control`), improving performance and scalability.

4. **Uniform Interface** — a consistent, standardized way of interacting with resources:
   - Resources identified via URLs (e.g., `/users/5`)
   - Manipulated through representations (typically JSON)
   - Self-descriptive messages (status codes, headers)
   - HATEOAS (Hypermedia as the Engine of Application State) — responses can include links to related actions/resources.

5. **Layered System** — the client can't necessarily tell whether it's talking directly to the server or through intermediaries (load balancers, caches, gateways) — this allows scalability and flexibility in backend architecture.

6. **Code on Demand (optional)** — servers can optionally send executable code (like JavaScript) to extend client functionality. Rarely used in typical REST APIs.

```js
// Example of HATEOAS-style response (uniform interface)
{
  "id": 5,
  "name": "Alice",
  "links": [
    { "rel": "self", "href": "/users/5" },
    { "rel": "orders", "href": "/users/5/orders" },
    { "rel": "delete", "href": "/users/5", "method": "DELETE" }
  ]
}
```

---

## Quick Summary Cheat Sheet

| Concept | One-liner |
|---|---|
| CRUD | Create, Read, Update, Delete — the 4 basic data operations |
| HTTP Methods | GET (read), POST (create), PUT (replace), PATCH (partial update), DELETE (remove) |
| Status Codes | 2xx success, 4xx client error, 5xx server error |
| Pagination | Splitting large result sets into pages (offset or cursor-based) |
| Filtering | Narrowing results via query parameters |
| Sorting | Controlling result order via a `sort` parameter |
| Authentication | Confirms identity (who) |
| Authorization | Confirms permissions (what you can do) |
| Idempotency | Same request repeated = same end result, safe to retry |
| Versioning | Evolving APIs without breaking existing clients (usually via `/v1/`, `/v2/`) |
| Rate Limiting | Capping requests per client to prevent abuse |
| PUT vs PATCH | Full replace vs partial update |
| 401 vs 403 | Not authenticated vs authenticated-but-not-allowed |
| REST Principles | Client-Server, Stateless, Cacheable, Uniform Interface, Layered System, Code on Demand |
