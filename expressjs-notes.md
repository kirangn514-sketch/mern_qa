# Express.js Complete Notes

---

## 1. What is Express.js?

**Simple definition:** Express.js is a **minimal, unopinionated web framework for Node.js**. It sits on top of Node's built-in `http` module and makes it much easier to build web servers and APIs — handling routing, requests, responses, and middleware without you writing repetitive low-level code.

**Why it exists:** Plain Node.js `http` server requires you to manually parse URLs, handle different HTTP methods, parse request bodies, and manage everything yourself. Express provides a clean, structured API (`app.get()`, `app.post()`, `app.use()`, etc.) to do all this with far less code.

```js
const express = require("express");
const app = express();

app.get("/", (req, res) => {
  res.send("Hello World!");
});

app.listen(3000, () => console.log("Server running on port 3000"));
```

---

## 2. Middleware

**Simple definition:** Middleware is a function that sits **in the middle** of the request-response cycle. It receives the request (`req`), the response (`res`), and a `next` function — and it can inspect/modify the request, end the response, or pass control to the next middleware/route handler.

**Think of it like:** an assembly line — each middleware function does one job (logging, authentication, parsing body, etc.) and then either passes the request along or stops it.

### Anatomy of middleware
```js
function myMiddleware(req, res, next) {
  console.log(`${req.method} ${req.url}`); // do something
  next(); // pass control to the next middleware/route
}

app.use(myMiddleware);
```

- If `next()` is **not called**, the request hangs forever (unless you send a response).
- Middleware can be applied:
  - **Globally**: `app.use(middleware)` — runs on every request.
  - **Per route**: `app.get("/admin", middleware, handler)` — runs only for that route.
  - **Per router**: applied to a specific `express.Router()` group.

### Types of middleware
| Type | Purpose | Example |
|---|---|---|
| Application-level | Runs for all/some routes | Logging, auth |
| Router-level | Attached to a specific `express.Router()` | Feature-specific middleware |
| Built-in | Comes with Express | `express.json()`, `express.static()` |
| Third-party | Installed via npm | `cors`, `helmet`, `morgan` |
| Error-handling | Catches errors (see below) | Custom error handler |

```js
// Built-in middleware example
app.use(express.json()); // parses incoming JSON request bodies into req.body
app.use(express.static("public")); // serves static files (images, CSS, etc.)
```

---

## 3. Routing

**Simple definition:** Routing is how your app decides **which function should handle a request**, based on the **URL path** and **HTTP method** (GET, POST, PUT, DELETE, etc.).

### Basic routing
```js
app.get("/users", (req, res) => {
  res.json({ message: "Get all users" });
});

app.post("/users", (req, res) => {
  res.json({ message: "Create a user" });
});

app.put("/users/:id", (req, res) => {
  res.json({ message: `Update user ${req.params.id}` });
});

app.delete("/users/:id", (req, res) => {
  res.json({ message: `Delete user ${req.params.id}` });
});
```

### Route parameters and query strings
```js
// Route param: /users/42
app.get("/users/:id", (req, res) => {
  res.send(`User ID: ${req.params.id}`);
});

// Query string: /search?name=alice
app.get("/search", (req, res) => {
  res.send(`Searching for: ${req.query.name}`);
});
```

### `express.Router()` — modular routing
- **Definition:** Lets you group related routes into separate files/modules, keeping the codebase organized (this is the backbone of MVC structure — see below).

```js
// routes/userRoutes.js
const express = require("express");
const router = express.Router();

router.get("/", (req, res) => res.send("All users"));
router.get("/:id", (req, res) => res.send("Single user"));

module.exports = router;
```

```js
// app.js
const userRoutes = require("./routes/userRoutes");
app.use("/api/users", userRoutes); // all routes prefixed with /api/users
```

---

## 4. Error Middleware

**Simple definition:** A special type of middleware **dedicated to handling errors** in one central place, instead of writing try/catch everywhere. Express identifies it by its **four parameters**: `(err, req, res, next)`.

**Key rule:** Error-handling middleware must be defined **last**, after all other `app.use()` and routes.

```js
// Normal route that passes an error along
app.get("/user/:id", (req, res, next) => {
  const user = findUser(req.params.id);
  if (!user) {
    const error = new Error("User not found");
    error.status = 404;
    return next(error); // passing an argument to next() triggers error middleware
  }
  res.json(user);
});

// Error-handling middleware (must have 4 params)
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(err.status || 500).json({
    success: false,
    message: err.message || "Internal Server Error",
  });
});
```

### Async errors
- Express doesn't automatically catch errors thrown inside `async` functions — you must catch and forward them manually, or use a wrapper.

```js
const asyncHandler = (fn) => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);

app.get("/data", asyncHandler(async (req, res) => {
  const data = await someAsyncOperation(); // if this throws, asyncHandler forwards it to error middleware
  res.json(data);
}));
```

---

## 5. Authentication

**Simple definition:** Authentication answers the question **"Who are you?"** — verifying a user's identity, typically via username/password, tokens, or OAuth.

### Common flow (with sessions or JWT)
1. User submits login credentials (email + password).
2. Server verifies credentials against the database (comparing **hashed** passwords, never plain text).
3. Server issues a **session cookie** or a **JWT token** proving the user is logged in.
4. On future requests, the client sends that session/token to prove identity.

```js
const bcrypt = require("bcrypt");

app.post("/login", async (req, res) => {
  const { email, password } = req.body;
  const user = await User.findOne({ email });

  if (!user || !(await bcrypt.compare(password, user.passwordHash))) {
    return res.status(401).json({ message: "Invalid credentials" });
  }

  // issue a token/session here (see JWT section)
  res.json({ message: "Login successful" });
});
```

- **Password hashing:** Always store passwords hashed with `bcrypt` or `argon2` — never store or compare plain-text passwords.

---

## 6. Authorization

**Simple definition:** Authorization answers **"What are you allowed to do?"** — it comes *after* authentication and checks whether the authenticated user has permission to perform a specific action (e.g., only admins can delete users).

```js
function authorize(...allowedRoles) {
  return (req, res, next) => {
    if (!allowedRoles.includes(req.user.role)) {
      return res.status(403).json({ message: "Forbidden: insufficient permissions" });
    }
    next();
  };
}

// Usage: only "admin" role can access this route
app.delete("/users/:id", authenticate, authorize("admin"), (req, res) => {
  res.json({ message: "User deleted" });
});
```

| | Authentication | Authorization |
|---|---|---|
| Question answered | Who are you? | What can you do? |
| Happens | First | After authentication |
| Example | Logging in | Checking if user is "admin" before allowing delete |

---

## 7. JWT (JSON Web Token)

**Simple definition:** JWT is a compact, self-contained token format used to securely transmit identity information between client and server. It's **stateless** — the server doesn't need to store session data; it just verifies the token's signature.

### Structure
A JWT looks like: `header.payload.signature`
- **Header:** algorithm used (e.g., HS256).
- **Payload:** claims/data (e.g., `userId`, `role`, `exp` for expiry) — this part is **encoded, not encrypted**, so never put secrets in it.
- **Signature:** ensures the token hasn't been tampered with.

### Issuing a token (on login)
```js
const jwt = require("jsonwebtoken");

const token = jwt.sign(
  { id: user._id, role: user.role },
  process.env.JWT_SECRET,
  { expiresIn: "1h" }
);

res.json({ token });
```

### Verifying a token (protecting routes)
```js
function authenticate(req, res, next) {
  const authHeader = req.headers.authorization; // "Bearer <token>"
  const token = authHeader && authHeader.split(" ")[1];

  if (!token) return res.status(401).json({ message: "No token provided" });

  jwt.verify(token, process.env.JWT_SECRET, (err, decoded) => {
    if (err) return res.status(403).json({ message: "Invalid or expired token" });
    req.user = decoded; // attach decoded payload (id, role) to request
    next();
  });
}

app.get("/profile", authenticate, (req, res) => {
  res.json({ message: `Welcome user ${req.user.id}` });
});
```

### Best practices
- Use **short expiry times** (15–60 min) for access tokens, paired with a longer-lived **refresh token**.
- Store tokens in **httpOnly cookies** (not `localStorage`) to protect against XSS attacks.
- Always sign with a strong secret (or use RS256 with public/private key pairs) and keep it in environment variables, never hardcoded.

---

## 8. CORS (Cross-Origin Resource Sharing)

**Simple definition:** A browser security feature that blocks web pages from making requests to a **different domain/origin** than the one that served the page, unless the server explicitly allows it. CORS middleware tells the browser "it's okay, this frontend origin is allowed to call my API."

**Why it matters:** If your frontend runs on `http://localhost:3000` and your API runs on `http://localhost:5000`, the browser will block requests between them by default — until the server sends the right CORS headers.

```js
const cors = require("cors");

// Allow all origins (fine for public APIs, not recommended for sensitive apps)
app.use(cors());

// Restrict to specific origin(s) — recommended for production
app.use(cors({
  origin: "https://myfrontend.com",
  methods: ["GET", "POST", "PUT", "DELETE"],
  credentials: true, // allow cookies to be sent
}));
```
- ⚠️ Never use a **CORS wildcard (`origin: "*"`) together with `credentials: true`** — browsers will reject this combination, and it's also a security risk.

---

## 9. Helmet

**Simple definition:** A middleware package that automatically sets various **HTTP security headers** to protect your app from common web vulnerabilities (XSS, clickjacking, MIME-sniffing, etc.) — essentially "free" security hardening with one line.

```js
const helmet = require("helmet");
app.use(helmet());
```

### What it does (examples of headers it sets)
| Header | Protects against |
|---|---|
| `X-Content-Type-Options: nosniff` | MIME-type sniffing attacks |
| `X-Frame-Options: DENY` | Clickjacking (embedding your site in an iframe) |
| `Strict-Transport-Security` | Forces HTTPS |
| `Content-Security-Policy` | Restricts sources for scripts/styles (XSS mitigation) |

- Always place `helmet()` near the top of your middleware stack, before route definitions.

---

## 10. Rate Limiter

**Simple definition:** Middleware that limits **how many requests** a client (usually identified by IP) can make in a given time window — protecting your API from brute-force attacks, abuse, and denial-of-service attempts.

```js
const rateLimit = require("express-rate-limit");

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: "Too many requests, please try again later.",
});

app.use(limiter); // apply globally

// Or apply stricter limits on sensitive routes, e.g. login
const loginLimiter = rateLimit({ windowMs: 15 * 60 * 1000, max: 5 });
app.post("/login", loginLimiter, loginHandler);
```
- **Common use case:** Much stricter limits on `/login` and `/signup` routes to prevent brute-force password guessing.

---

## 11. Validation

**Simple definition:** Checking that incoming request data (body, params, query) is in the **correct format and meets expected rules** before your app processes it — preventing bad data, crashes, and security issues like injection attacks.

### Using `express-validator`
```js
const { body, validationResult } = require("express-validator");

app.post(
  "/signup",
  [
    body("email").isEmail().withMessage("Must be a valid email"),
    body("password").isLength({ min: 8 }).withMessage("Password must be at least 8 characters"),
  ],
  (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    // proceed with valid data
    res.json({ message: "Signup successful" });
  }
);
```

### Using `Joi` (schema-based validation)
```js
const Joi = require("joi");

const schema = Joi.object({
  email: Joi.string().email().required(),
  password: Joi.string().min(8).required(),
});

function validateBody(req, res, next) {
  const { error } = schema.validate(req.body);
  if (error) return res.status(400).json({ message: error.details[0].message });
  next();
}

app.post("/signup", validateBody, (req, res) => {
  res.json({ message: "Signup successful" });
});
```
- **Why it matters:** Never trust client input — always validate on the **server side**, even if you also validate on the frontend.

---

## 12. MVC Structure

**Simple definition:** MVC (**Model–View–Controller**) is an architectural pattern that separates your app into three distinct responsibilities, keeping code organized, testable, and maintainable as it grows.

- **Model:** Handles data and business logic — interacts with the database (e.g., Mongoose/Sequelize schemas).
- **View:** The presentation layer — in a REST API, this is usually just the JSON response (no templating engine needed); in traditional web apps, this could be EJS/Pug templates.
- **Controller:** Contains the logic that connects Model and View — receives requests, calls the model, and sends a response.

### Typical folder structure
```
project/
├── models/
│   └── User.js
├── controllers/
│   └── userController.js
├── routes/
│   └── userRoutes.js
├── middlewares/
│   ├── auth.js
│   └── errorHandler.js
├── app.js
└── server.js
```

### Example implementation

```js
// models/User.js
const mongoose = require("mongoose");

const userSchema = new mongoose.Schema({
  name: String,
  email: { type: String, unique: true },
  passwordHash: String,
});

module.exports = mongoose.model("User", userSchema);
```

```js
// controllers/userController.js
const User = require("../models/User");

exports.getAllUsers = async (req, res, next) => {
  try {
    const users = await User.find();
    res.json({ success: true, data: users });
  } catch (err) {
    next(err);
  }
};

exports.createUser = async (req, res, next) => {
  try {
    const user = await User.create(req.body);
    res.status(201).json({ success: true, data: user });
  } catch (err) {
    next(err);
  }
};
```

```js
// routes/userRoutes.js
const express = require("express");
const router = express.Router();
const { getAllUsers, createUser } = require("../controllers/userController");

router.get("/", getAllUsers);
router.post("/", createUser);

module.exports = router;
```

```js
// app.js
const express = require("express");
const userRoutes = require("./routes/userRoutes");
const errorHandler = require("./middlewares/errorHandler");

const app = express();
app.use(express.json());
app.use("/api/users", userRoutes);
app.use(errorHandler); // error middleware always last

module.exports = app;
```

**Why use MVC:** Without it, all logic (DB queries, validation, response formatting) tends to pile up inside route files, making the app hard to test and maintain as it grows. MVC keeps each concern isolated and reusable.

---

## Quick Summary Cheat Sheet

| Concept | One-liner |
|---|---|
| Middleware | Function with `(req, res, next)` that runs during the request lifecycle |
| Routing | Mapping URL + HTTP method to a handler function |
| Error Middleware | Centralized error handling via `(err, req, res, next)`, defined last |
| Authentication | Verifying *who* the user is |
| Authorization | Verifying *what* the user can do |
| JWT | Stateless token used to prove identity across requests |
| CORS | Controls which origins are allowed to call your API from a browser |
| Helmet | Sets secure HTTP headers automatically |
| Rate Limiter | Restricts number of requests per IP in a time window |
| Validation | Ensures incoming data is correct before processing it |
| MVC Structure | Separates app into Model (data), View (response), Controller (logic) |

## Interview-Style Quick Answers

**Q: What is middleware and why is order important?**
Middleware are functions executed in sequence for each request. Order matters because each middleware can modify `req`/`res` or short-circuit the chain — e.g., body-parsing middleware must run before route handlers that read `req.body`, and error middleware must be defined last to catch errors from everything above it.

**Q: How does Express know a middleware is for error handling?**
By its **arity** — it must accept exactly 4 parameters: `(err, req, res, next)`. Express treats any 4-argument middleware function specially.

**Q: Difference between authentication and authorization?**
Authentication confirms identity (login); authorization checks permissions for an already-authenticated user (role/access checks).

**Q: Is a JWT encrypted?**
No — the payload is only **base64-encoded**, not encrypted, so it's readable by anyone who has the token. Never store sensitive data (passwords, secrets) in the payload. The **signature** only guarantees the token hasn't been tampered with, not confidentiality.

**Q: Why use both Helmet and CORS?**
They solve different problems: Helmet hardens your server against common attacks via response headers; CORS controls which frontend origins are permitted to make cross-origin requests to your API. Both are commonly used together.
