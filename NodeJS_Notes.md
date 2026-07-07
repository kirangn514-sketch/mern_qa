# Node.js — Complete Interview Notes

---

# 🏛️ Architecture

## 1. Event Loop

**Simple definition:** The Event Loop is the core mechanism that lets Node.js (which runs JavaScript on a **single thread**) handle thousands of concurrent operations (file reads, network calls, timers) **without blocking**. It's what makes Node.js asynchronous and non-blocking.

**How it works:**
Node.js uses the **libuv** library, which provides an Event Loop with multiple **phases**. Each phase has its own queue of callbacks to process. The loop cycles through these phases repeatedly as long as the program is running.

**Event Loop Phases (in order):**

1. **Timers** — executes callbacks scheduled by `setTimeout()` and `setInterval()` whose delay has elapsed.
2. **Pending Callbacks** — executes I/O callbacks deferred from the previous cycle (e.g., certain system errors like TCP errors).
3. **Idle, Prepare** — internal use only, not relevant to application code.
4. **Poll** — retrieves new I/O events (like reading files, incoming network requests); executes their callbacks. Node will wait here if there's nothing else to do.
5. **Check** — executes `setImmediate()` callbacks, right after the Poll phase.
6. **Close Callbacks** — executes cleanup callbacks like `socket.on('close', ...)`.

Between **every phase**, and even between individual callbacks within a phase, Node drains the **microtask queue** (`process.nextTick()` callbacks first, then Promise `.then()` callbacks).

```javascript
console.log("Start");

setTimeout(() => console.log("setTimeout"), 0);
setImmediate(() => console.log("setImmediate"));

Promise.resolve().then(() => console.log("Promise"));
process.nextTick(() => console.log("nextTick"));

console.log("End");

// Output:
// Start
// End
// nextTick        (highest priority microtask)
// Promise
// setTimeout / setImmediate (order can vary depending on context)
```

**Key point:** `process.nextTick()` callbacks run **before** Promise microtasks, and both run before moving to the next Event Loop phase.

---

## 2. Non-blocking I/O

**Simple definition:** Non-blocking I/O means that when Node.js performs an input/output operation (reading a file, querying a database, making an HTTP request), it does **NOT wait (block)** for that operation to finish before executing the next line of code. Instead, it registers a callback and moves on, getting notified later when the operation completes.

```javascript
const fs = require("fs");

console.log("1. Start reading file");

// Non-blocking — doesn't wait here
fs.readFile("data.txt", "utf8", (err, data) => {
  console.log("3. File data:", data);
});

console.log("2. This runs before file reading finishes");

// Output order: 1 → 2 → 3
```

Compare this to the **blocking (synchronous)** version:
```javascript
const data = fs.readFileSync("data.txt", "utf8"); // blocks everything until done
console.log(data);
```

**Why this matters:** Since Node.js is single-threaded, if I/O were blocking, the entire server would freeze while waiting for a slow disk read or database query — unable to serve any other requests. Non-blocking I/O (delegated to libuv's thread pool or OS-level async APIs) is why Node handles high-concurrency workloads efficiently despite being single-threaded.

---

## 3. Streams

**Simple definition:** Streams are a way to handle reading/writing data **piece by piece (in chunks)** instead of loading the entire data into memory at once. Ideal for large files, video processing, or network data.

**Types of Streams:**
| Type | Purpose | Example |
|---|---|---|
| **Readable** | Read data from a source | `fs.createReadStream()` |
| **Writable** | Write data to a destination | `fs.createWriteStream()` |
| **Duplex** | Both readable and writable | TCP sockets |
| **Transform** | Duplex stream that modifies data as it passes through | `zlib.createGzip()` (compression) |

```javascript
const fs = require("fs");

// Without streams: loads ENTIRE file into memory (bad for large files)
// const data = fs.readFileSync("large-file.txt");

// With streams: reads in small chunks, memory-efficient
const readStream = fs.createReadStream("large-file.txt", { encoding: "utf8" });
const writeStream = fs.createWriteStream("output.txt");

readStream.on("data", (chunk) => {
  console.log("Received chunk of size:", chunk.length);
});

readStream.pipe(writeStream); // piping: connects readable output directly to writable input
```

**Why streams matter:** Instead of waiting for a 2GB file to fully load into RAM, you process it in small chunks (e.g., 64KB at a time) — massively improving memory efficiency and speed.

---

## 4. Buffers

**Simple definition:** A Buffer is a temporary storage area for **raw binary data** — used when Node.js works with things that aren't plain text (like image files, video, TCP packets). JavaScript originally had no way to handle raw binary data directly, so Node introduced the `Buffer` class.

```javascript
const buf = Buffer.from("Hello");
console.log(buf);          // <Buffer 48 65 6c 6c 6f> — binary data in hexadecimal
console.log(buf.toString()); // "Hello" — convert back to readable string

const buf2 = Buffer.alloc(10); // allocates 10 bytes, all zeroed
```

**Key points:**
- Buffers store data outside the regular JavaScript memory heap (in raw memory), making them fast for handling binary data.
- Streams internally use Buffers to hold chunks of data as they're being read/written.
- Fixed size once created — cannot be resized.

---

## 5. Cluster

**Simple definition:** Since Node.js runs on a **single thread** and can only use **one CPU core**, the `cluster` module allows you to create multiple copies (workers) of your Node application, each running on a separate CPU core, to handle more traffic and utilize multi-core systems fully.

```javascript
const cluster = require("cluster");
const http = require("http");
const os = require("os");

if (cluster.isMaster) {
  const numCPUs = os.cpus().length;
  console.log(`Master process. Forking ${numCPUs} workers`);

  for (let i = 0; i < numCPUs; i++) {
    cluster.fork(); // create a worker process per CPU core
  }

  cluster.on("exit", (worker) => {
    console.log(`Worker ${worker.process.pid} died. Restarting...`);
    cluster.fork(); // restart a crashed worker
  });
} else {
  // Each worker runs its own instance of the server
  http.createServer((req, res) => {
    res.end("Handled by worker " + process.pid);
  }).listen(3000);
}
```

**How it works:** The master process doesn't handle requests itself — it distributes incoming connections across worker processes (usually round-robin), and each worker has its own memory and Event Loop. If one worker crashes, others keep serving requests.

---

# 📦 Core Modules

## `fs` (File System)

**Simple definition:** Built-in module for interacting with the file system — reading, writing, updating, deleting files/directories.

```javascript
const fs = require("fs");

// Asynchronous (non-blocking, callback-based)
fs.readFile("file.txt", "utf8", (err, data) => {
  if (err) throw err;
  console.log(data);
});

// Synchronous (blocking)
const data = fs.readFileSync("file.txt", "utf8");

// Promise-based (modern approach)
const fsPromises = require("fs/promises");
async function read() {
  const data = await fsPromises.readFile("file.txt", "utf8");
  console.log(data);
}

fs.writeFile("output.txt", "Hello World", (err) => {});
fs.appendFile("output.txt", " More text", (err) => {});
fs.unlink("output.txt", (err) => {}); // delete file
fs.mkdir("newFolder", (err) => {});
```

## `path`

**Simple definition:** Provides utilities for working with file and directory paths, handling differences between operating systems (Windows uses `\`, Linux/Mac use `/`) automatically.

```javascript
const path = require("path");

path.join("/users", "john", "docs", "file.txt");
// "/users/john/docs/file.txt" — safely joins path segments

path.resolve("file.txt");
// returns absolute path based on current working directory

path.basename("/users/john/file.txt"); // "file.txt"
path.dirname("/users/john/file.txt");  // "/users/john"
path.extname("/users/john/file.txt");  // ".txt"
```

## `http`

**Simple definition:** The core module used to create web servers and handle HTTP requests/responses without needing any external framework (like Express).

```javascript
const http = require("http");

const server = http.createServer((req, res) => {
  res.writeHead(200, { "Content-Type": "text/plain" });
  res.end("Hello World");
});

server.listen(3000, () => {
  console.log("Server running on port 3000");
});
```

Frameworks like **Express.js** are built on top of this module to make routing, middleware, and request handling much easier.

## `os`

**Simple definition:** Provides information about the operating system the Node.js process is running on — CPU, memory, network interfaces, platform, etc.

```javascript
const os = require("os");

console.log(os.platform());   // 'win32', 'linux', 'darwin'
console.log(os.cpus().length); // number of CPU cores
console.log(os.totalmem());   // total system memory in bytes
console.log(os.freemem());    // free memory in bytes
console.log(os.homedir());    // user's home directory
```

## `crypto`

**Simple definition:** Provides cryptographic functionality — hashing, encryption, decryption, generating secure random tokens. Commonly used for password hashing, generating secure IDs, and signing tokens.

```javascript
const crypto = require("crypto");

// Hashing (one-way — cannot be reversed)
const hash = crypto.createHash("sha256").update("password123").digest("hex");

// Generating a random secure token (e.g., for password-reset links)
const token = crypto.randomBytes(32).toString("hex");

// HMAC (Hash-based Message Authentication Code) - used to verify data integrity/authenticity
const hmac = crypto.createHmac("sha256", "secretKey").update("data").digest("hex");
```

---

# ⚠️ Error Handling

**Simple definition:** Error handling in Node.js means properly catching and managing errors — both **synchronous** (immediate, e.g., invalid JSON parsing) and **asynchronous** (delayed, e.g., failed database queries) — so your app doesn't crash unexpectedly and gives meaningful feedback.

**Synchronous errors — use try/catch:**
```javascript
try {
  JSON.parse("invalid json");
} catch (err) {
  console.log("Caught error:", err.message);
}
```

**Asynchronous errors — callback pattern (error-first callbacks):**
```javascript
fs.readFile("nofile.txt", (err, data) => {
  if (err) {
    console.log("Error:", err.message);
    return;
  }
  console.log(data);
});
```

**Promise-based errors:**
```javascript
async function getData() {
  try {
    const data = await fetchFromDB();
  } catch (err) {
    console.log("DB error:", err.message);
  }
}
```

**Express error-handling middleware** (centralizes error handling across all routes):
```javascript
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ message: "Something went wrong!" });
});
```

**Handling uncaught errors globally** (last line of defense — should log and gracefully exit, not just ignore):
```javascript
process.on("uncaughtException", (err) => {
  console.error("Uncaught Exception:", err);
  process.exit(1);
});

process.on("unhandledRejection", (reason) => {
  console.error("Unhandled Rejection:", reason);
});
```

---

# 📝 Logging

**Simple definition:** Logging means recording what's happening inside your application (requests, errors, performance data) so you can monitor, debug, and audit it — especially critical in production where you can't just "watch the console."

**Basic approach:**
```javascript
console.log("Info: Server started");
console.error("Error: Something failed");
```

**Problems with `console.log` in production:** no log levels, no timestamps, no file output, no log rotation, hard to search/filter at scale.

**Production-grade logging — using a library like `winston` or `pino`:**
```javascript
const winston = require("winston");

const logger = winston.createLogger({
  level: "info",
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: "error.log", level: "error" }),
    new winston.transports.File({ filename: "combined.log" }),
  ],
});

logger.info("Server started on port 3000");
logger.error("Database connection failed");
```

**Log levels** (from most to least severe): `error` → `warn` → `info` → `http` → `verbose` → `debug` → `silly`. In production, you typically log `info` and above; in development, you might enable `debug` too.

---

# 🔐 JWT Authentication

**Simple definition:** JWT (JSON Web Token) is a compact, self-contained token used to securely transmit information between a client and server, commonly for **authentication**. Once a user logs in, the server issues a JWT; the client sends it back with every subsequent request to prove their identity — **without the server needing to store session data**.

**Structure of a JWT:** `header.payload.signature`
- **Header** — algorithm and token type.
- **Payload** — the actual data (user ID, role, expiry) — NOT encrypted, just base64-encoded (never put sensitive data here).
- **Signature** — created by signing header+payload with a secret key, used to verify the token hasn't been tampered with.

```javascript
const jwt = require("jsonwebtoken");

// Creating (signing) a token after login
const token = jwt.sign(
  { userId: user._id, role: user.role },
  process.env.JWT_SECRET,
  { expiresIn: "15m" }
);

// Verifying a token (in protected route middleware)
function authMiddleware(req, res, next) {
  const token = req.headers.authorization?.split(" ")[1]; // "Bearer <token>"
  if (!token) return res.status(401).json({ message: "No token provided" });

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded; // attach decoded user info to request
    next();
  } catch (err) {
    res.status(401).json({ message: "Invalid or expired token" });
  }
}
```

**Why JWT is popular:** it's **stateless** — the server doesn't need to store session info in memory/DB, which makes it easy to scale across multiple servers.

---

# 🔄 Refresh Token

**Simple definition:** Since JWT **access tokens** are short-lived (for security — e.g., 15 minutes), a **refresh token** is a longer-lived, securely stored token used to obtain a **new** access token without forcing the user to log in again.

**Typical flow:**
1. User logs in → server issues **short-lived access token** (e.g., 15 min) + **long-lived refresh token** (e.g., 7 days).
2. Access token is used for normal API requests.
3. When the access token expires, the client sends the refresh token to a special `/refresh` endpoint.
4. Server verifies the refresh token (usually checked against a database/whitelist so it can be revoked) and issues a **new access token**.
5. If the refresh token itself is invalid/expired/revoked, the user must log in again.

```javascript
// /refresh endpoint
app.post("/refresh-token", async (req, res) => {
  const { refreshToken } = req.cookies;
  if (!refreshToken) return res.sendStatus(401);

  const storedToken = await RefreshToken.findOne({ token: refreshToken });
  if (!storedToken) return res.sendStatus(403); // token revoked/invalid

  jwt.verify(refreshToken, process.env.REFRESH_SECRET, (err, decoded) => {
    if (err) return res.sendStatus(403);

    const newAccessToken = jwt.sign(
      { userId: decoded.userId },
      process.env.JWT_SECRET,
      { expiresIn: "15m" }
    );
    res.json({ accessToken: newAccessToken });
  });
});
```

**Best practice:** Store the refresh token in an **httpOnly cookie** (not accessible via JavaScript, protecting against XSS attacks), and store a record of issued refresh tokens server-side so they can be revoked (e.g., on logout or password change).

---

# 🔒 Password Hashing

**Simple definition:** Storing passwords in **plain text** is extremely dangerous — if the database is breached, all user passwords are exposed instantly. **Hashing** converts a password into an irreversible, fixed-length string, so even if the database leaks, the actual passwords can't be recovered.

**Key property:** Hashing is a **one-way function** — you can hash a password to compare it, but you can never "unhash" it back to the original text.

```javascript
const crypto = require("crypto");

const hash = crypto.createHash("sha256").update("myPassword123").digest("hex");
```

**Problem with plain SHA-256 for passwords:** it's too fast — attackers can brute-force millions of hash guesses per second using powerful hardware (GPU/ASIC). This is why we use specialized password-hashing algorithms like **bcrypt**, which are intentionally slow and include a technique called **salting**.

**Salting:** adding a random unique string to each password before hashing, so that even if two users have the same password, their stored hashes are completely different — this defeats precomputed "rainbow table" attacks.

---

# 🧂 Bcrypt

**Simple definition:** Bcrypt is a widely-used password-hashing library specifically designed to be **slow and resource-intensive** (unlike SHA-256), making brute-force attacks impractical. It automatically handles salting for you.

```javascript
const bcrypt = require("bcrypt");

// Hashing a password during signup
async function hashPassword(plainPassword) {
  const saltRounds = 10; // cost factor — higher = slower/more secure, but more CPU-intensive
  const hashedPassword = await bcrypt.hash(plainPassword, saltRounds);
  return hashedPassword;
}

// Comparing a password during login
async function checkPassword(plainPassword, hashedPassword) {
  const isMatch = await bcrypt.compare(plainPassword, hashedPassword);
  return isMatch; // true or false
}
```

**How `saltRounds` works:** it determines how many times the hashing algorithm runs internally (2^saltRounds iterations). A value like `10-12` is a common balance between security and performance — higher values make brute-forcing exponentially harder but also slow down your own server's hashing operations.

**Important:** Never store or compare plain-text passwords — always hash on signup and use `bcrypt.compare()` on login (never manually re-hash and compare strings directly).

---

# 📤 File Upload

**Simple definition:** File upload refers to allowing users to send files (images, PDFs, videos) from the client to the server, which then stores them — either on the local disk, or in cloud storage (like AWS S3, Cloudinary).

**Key concept — `multipart/form-data`:** Regular JSON request bodies can't carry binary file data efficiently. File uploads use a special content type called `multipart/form-data`, which splits the request into named "parts" (fields + files). Node itself doesn't parse this natively — you need a library like **Multer** to handle it.

---

# 📁 Multer

**Simple definition:** Multer is a middleware for Express.js specifically built to handle `multipart/form-data`, making file uploads easy to work with in Node.js.

```javascript
const express = require("express");
const multer = require("multer");
const app = express();

// Configure storage location and filename
const storage = multer.diskStorage({
  destination: (req, file, cb) => cb(null, "uploads/"),
  filename: (req, file, cb) => {
    cb(null, Date.now() + "-" + file.originalname);
  },
});

const upload = multer({
  storage,
  limits: { fileSize: 5 * 1024 * 1024 }, // 5MB limit
  fileFilter: (req, file, cb) => {
    if (file.mimetype.startsWith("image/")) cb(null, true);
    else cb(new Error("Only image files allowed"), false);
  },
});

// Single file upload
app.post("/upload", upload.single("avatar"), (req, res) => {
  console.log(req.file); // uploaded file info
  res.json({ message: "File uploaded", file: req.file });
});

// Multiple files upload
app.post("/upload-multiple", upload.array("photos", 5), (req, res) => {
  console.log(req.files); // array of uploaded files
  res.json({ message: "Files uploaded" });
});
```

**Note:** For production apps, it's common to upload directly to cloud storage (AWS S3, Cloudinary, etc.) instead of the local server disk, since local storage doesn't scale well across multiple servers and can fill up disk space.

---

# 📧 Email

**Simple definition:** Sending emails from a Node.js backend (for welcome emails, password resets, notifications) usually uses a library like **Nodemailer**, connected to an SMTP provider (Gmail, SendGrid, Mailgun, AWS SES).

```javascript
const nodemailer = require("nodemailer");

const transporter = nodemailer.createTransport({
  service: "gmail",
  auth: {
    user: process.env.EMAIL_USER,
    pass: process.env.EMAIL_PASS, // use an "app password", never your real password
  },
});

async function sendEmail(to, subject, text) {
  await transporter.sendMail({
    from: process.env.EMAIL_USER,
    to,
    subject,
    text,
  });
}

sendEmail("user@example.com", "Welcome!", "Thanks for signing up.");
```

**In production:** Dedicated email services (SendGrid, AWS SES, Mailgun, Resend) are preferred over Gmail SMTP, since Gmail has strict sending limits and can flag automated mail as spam.

---

# 🔢 OTP (One-Time Password)

**Simple definition:** An OTP is a temporary, randomly generated code (usually 4-6 digits) sent to a user via email or SMS, used to verify their identity — commonly for signup verification, login 2FA (two-factor authentication), or password resets. It's valid for a short time and can only be used once.

```javascript
const crypto = require("crypto");

// Generate a 6-digit OTP
function generateOTP() {
  return crypto.randomInt(100000, 999999).toString();
}

// Typical flow:
async function sendOTP(email) {
  const otp = generateOTP();
  const expiresAt = Date.now() + 5 * 60 * 1000; // valid for 5 minutes

  // Store OTP (hashed, ideally) + expiry in DB against the user's email
  await OtpModel.create({ email, otp, expiresAt });

  await sendEmail(email, "Your OTP Code", `Your OTP is ${otp}. It expires in 5 minutes.`);
}

async function verifyOTP(email, userOtp) {
  const record = await OtpModel.findOne({ email });
  if (!record) return false;
  if (Date.now() > record.expiresAt) return false; // expired
  return record.otp === userOtp;
}
```

**Best practices:** always set a short expiry, limit verification attempts (e.g., max 5 tries) to prevent brute-forcing, and delete/invalidate the OTP once used successfully.

---

# 🎯 Interview Questions — Model Answers

### Q1. Why is Node.js fast?
Node.js is fast primarily because of its **non-blocking, event-driven architecture**. Instead of creating a new thread for every request (which is expensive in memory and context-switching), Node uses a **single thread** with an **Event Loop** to handle many concurrent connections. I/O operations (file reads, DB queries, network calls) are delegated to the system kernel or a background thread pool (via libuv), and Node continues executing other code while waiting — only running the callback once the operation completes. It's also built on Google's **V8 engine**, which compiles JavaScript directly to optimized machine code, making execution itself very fast. This combination makes Node especially efficient for I/O-heavy applications like APIs, real-time apps, and streaming services (though it's less ideal for CPU-heavy tasks, since those would block the single thread).

### Q2. Explain the Event Loop phases.
The Event Loop in Node.js (powered by libuv) cycles through several distinct phases each iteration: **Timers** (runs expired `setTimeout`/`setInterval` callbacks), **Pending Callbacks** (handles deferred system-level callbacks), **Poll** (retrieves new I/O events and executes their callbacks — this is where most work happens, and Node waits here if idle), **Check** (runs `setImmediate()` callbacks), and **Close Callbacks** (handles cleanup like closed sockets). Between every phase — and between callbacks within a phase — Node fully drains the **microtask queue**: first `process.nextTick()` callbacks, then Promise callbacks, before moving on.

### Q3. Explain Streams.
Streams are Node's way of handling data incrementally, in **chunks**, rather than loading everything into memory at once. There are four types: **Readable** (source of data, e.g. reading a file), **Writable** (destination for data, e.g. writing to a file), **Duplex** (both readable and writable, e.g. a TCP socket), and **Transform** (a duplex stream that modifies data as it flows through, e.g. gzip compression). Streams are essential for handling large files or continuous data (like video) efficiently, since you avoid loading gigabytes of data into RAM. The `.pipe()` method lets you connect a readable stream directly to a writable stream, letting data flow automatically with backpressure handling built in.

### Q4. Explain Cluster.
Since Node runs JavaScript on a single thread, by default a Node application can only use **one CPU core**, even on a multi-core machine. The `cluster` module solves this by spawning multiple **worker processes** (each running its own instance of the Node app with its own Event Loop and memory), one per CPU core. A master process manages these workers and distributes incoming network connections among them, typically round-robin. This allows a Node application to fully utilize multi-core servers and improves fault tolerance — if one worker crashes, the master can spawn a replacement while other workers keep serving traffic.

### Q5. Explain Worker Threads.
While `cluster` creates entirely separate **processes** (each with isolated memory), **Worker Threads** (from the `worker_threads` module) let you run JavaScript in **parallel threads within the same process**, sharing memory via `SharedArrayBuffer` if needed. This is specifically designed for **CPU-intensive tasks** (image processing, complex calculations, data compression) that would otherwise block Node's single main thread and freeze the Event Loop. Unlike `cluster` (which is about scaling network I/O across cores), Worker Threads are about offloading heavy synchronous computation so the main thread stays responsive to handle other requests.

```javascript
const { Worker, isMainThread, parentPort } = require("worker_threads");

if (isMainThread) {
  const worker = new Worker(__filename);
  worker.on("message", (result) => console.log("Result:", result));
} else {
  // Heavy computation done in a separate thread
  let sum = 0;
  for (let i = 0; i < 1e9; i++) sum += i;
  parentPort.postMessage(sum);
}
```

**Cluster vs Worker Threads:**
| | Cluster | Worker Threads |
|---|---|---|
| Isolation | Separate processes (own memory) | Threads within same process (can share memory) |
| Best for | Scaling network/I/O-bound apps across CPU cores | Offloading CPU-intensive/blocking computation |
| Communication | IPC (Inter-Process Communication) | Message passing / SharedArrayBuffer |

---

*End of notes.*
