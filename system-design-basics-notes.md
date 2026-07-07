# System Design (Basic) — Complete Notes

---

## 1. What is System Design?

**Simple definition:** System design is the process of planning **how the different pieces of a large software system fit together** — servers, databases, caches, queues, APIs — so the system is scalable, reliable, and performant under real-world load.

**Why "basic" system design matters for interviews:** You're rarely asked to design something from scratch with unlimited scale in mind. Instead, interviewers want to see that you can reason about **trade-offs** — for example, "should this be synchronous or asynchronous?", "how do we handle failure?", "how do we scale this component if traffic grows 100x?" The topics below are the most commonly asked basic system design questions, and each teaches a transferable pattern.

---

## 2. Authentication Flow

**Simple definition:** The end-to-end process of verifying a user's identity and keeping them "logged in" across requests, using tokens/sessions and secure storage.

### High-level flow (JWT-based, most common in modern APIs)

```
┌────────┐     1. login (email+password)     ┌────────┐
│ Client │ ────────────────────────────────► │ Server │
└────────┘                                    └────────┘
                                                   │ 2. verify credentials (bcrypt compare)
                                                   │    against DB
                                                   ▼
┌────────┐   3. access token + refresh token  ┌────────┐
│ Client │ ◄──────────────────────────────────│ Server │
└────────┘        (in httpOnly cookies)        └────────┘
    │
    │ 4. request with access token (Authorization: Bearer <token>)
    ▼
┌────────┐   5. verify token signature + exp   ┌────────┐
│ Server │ ────────────────────────────────►   │  API   │
└────────┘                                      └────────┘
    │
    │ 6. access token expired?
    ▼
┌────────┐   7. call /refresh with refresh     ┌────────┐
│ Client │ ─────────────token────────────────► │ Server │
└────────┘                                      └────────┘
    │ 8. new access token issued (no re-login needed)
```

### Step-by-step
1. **User submits credentials** → server looks up the user and compares the submitted password against the stored **hashed** password (bcrypt/Argon2 — never plain text).
2. **Server issues tokens:**
   - **Access token** (JWT, short-lived — 15–60 min): sent with every API request.
   - **Refresh token** (long-lived — days/weeks): used only to get a new access token, stored server-side so it can be revoked.
3. **Tokens are stored in httpOnly, Secure, SameSite cookies** — protecting against XSS (JavaScript can't read them) and CSRF (not sent on cross-site requests).
4. **Every protected API request** includes the access token; middleware verifies its signature and expiry before allowing the request through.
5. **When the access token expires**, the client silently calls a `/refresh` endpoint using the refresh token to get a new access token — the user never notices.
6. **Logout:** delete/revoke the refresh token server-side and clear cookies.

### Key design decisions to discuss in an interview
- **Stateless (JWT) vs stateful (sessions):** JWT scales better across multiple servers (no shared session store needed); sessions are easier to revoke instantly.
- **Where to store tokens:** httpOnly cookies (safer against XSS) vs `localStorage` (simpler, but vulnerable to token theft via XSS).
- **Multi-device login:** store multiple refresh tokens per user (one per device/session) so logging out of one device doesn't affect others.
- **Third-party login (OAuth):** delegate identity verification to Google/GitHub instead of managing passwords yourself.

---

## 3. File Upload

**Simple definition:** Designing a system to let users upload files (images, videos, documents) reliably and efficiently, without overloading your application server.

### Naive approach (small files only)
```
Client → App Server (receives file, saves to disk/DB) → Response
```
- **Problem:** Doesn't scale. Large files consume server memory/bandwidth, app servers become a bottleneck, and storing files on local disk doesn't work with multiple server instances (or ephemeral containers).

### Production-grade approach: Direct-to-cloud upload with pre-signed URLs

```
┌────────┐  1. "I want to upload photo.jpg" ┌─────────────┐
│ Client │ ───────────────────────────────► │ App Server  │
└────────┘                                   └─────────────┘
                                                   │ 2. generate a pre-signed URL
                                                   │    (short-lived, scoped to this file)
                                                   ▼
┌────────┐  3. pre-signed URL returned      ┌─────────────┐
│ Client │ ◄─────────────────────────────── │ App Server  │
└────────┘                                   └─────────────┘
    │
    │ 4. upload file DIRECTLY to storage using the pre-signed URL
    ▼
┌──────────────────┐
│ Cloud Storage     │  (S3 / GCS / Azure Blob)
└──────────────────┘
    │
    │ 5. (optional) storage triggers a webhook/event
    ▼
┌─────────────┐
│ App Server  │  6. save file metadata (URL, size, owner) to DB
└─────────────┘
```

### Why this design is better
- **App server never touches the file bytes** — it only issues a temporary, secure URL. This keeps the app server lightweight and avoids memory/bandwidth bottlenecks.
- **Pre-signed URLs** are time-limited and scoped (e.g., valid for 5 minutes, only for a specific file key) — secure even though the client uploads "directly."
- **Large file handling:** For very large files (videos), use **chunked/multipart uploads** — the file is split into pieces, uploaded in parallel, and reassembled by the storage service — allowing pause/resume and better reliability on flaky connections.

### Additional considerations
- **Validation:** Check file type/size **before** issuing the pre-signed URL (and again after upload, since client-side checks can be bypassed).
- **Virus scanning:** Run uploaded files through a scanning service (e.g., ClamAV, or a cloud provider's malware detection) before making them publicly accessible.
- **CDN:** Serve uploaded files (especially images) through a CDN for fast global delivery, rather than direct storage access.

---

## 4. Chat Application

**Simple definition:** Designing a system that lets users exchange messages in **near real-time**, requiring a persistent, bidirectional connection rather than the traditional request-response HTTP model.

### Why regular HTTP doesn't work well here
HTTP is request-response: the client asks, the server answers, connection closes. For chat, the server needs to **push** new messages to clients the moment they arrive — this requires a different approach.

### Core technology: WebSockets
**Simple definition:** WebSockets establish a **persistent, full-duplex connection** between client and server — once opened, either side can send data to the other at any time, without the overhead of repeated HTTP requests.

```
┌────────┐   1. HTTP handshake (upgrade to WebSocket)   ┌────────┐
│ Client │ ─────────────────────────────────────────►  │ Server │
└────────┘                                               └────────┘
    │◄──────────── persistent open connection ────────────►│
    │                                                        │
    │ 2. send message                                        │
    │──────────────────────────────────────────────────────►│
    │                                     3. broadcast to     │
    │                                        other connected  │
    │◄────────────────────────────────────  clients ─────────│
```

### High-level architecture

```
┌────────┐         ┌──────────────┐        ┌──────────────┐
│ Client │ ◄─────► │ WebSocket    │ ◄────► │  Message      │
│ (User A)│         │ Server(s)    │        │  Queue/Pub-Sub│  (e.g., Redis Pub/Sub, Kafka)
└────────┘         └──────────────┘        └──────────────┘
                          │                        │
                          ▼                        ▼
                   ┌──────────────┐        ┌──────────────┐
                   │   Database    │        │ WebSocket     │
                   │ (chat history)│        │ Server(s) —   │
                   └──────────────┘        │ other instances│
                                            └──────────────┘
                                                    │
                                                    ▼
                                             ┌────────┐
                                             │ Client │
                                             │(User B)│
                                             └────────┘
```

### Key design considerations
1. **Persisting messages:** Every message is saved to a database (often a NoSQL store like Cassandra/MongoDB, optimized for high write throughput) so chat history is available even if a user is offline.
2. **Scaling across multiple servers:** A single WebSocket server can't hold all connections at scale. If User A connects to Server 1 and User B connects to Server 2, Server 1 needs a way to notify Server 2 about a new message — this is solved with a **Pub/Sub system** (like Redis Pub/Sub or Kafka), where all servers subscribe to relevant channels and relay messages to their connected clients.
3. **Online/offline presence:** Typically tracked in a fast key-value store (Redis) — e.g., `user:123:status = online`, with a TTL/heartbeat mechanism to detect disconnects.
4. **Message delivery guarantees:** Use message IDs and acknowledgments so the client can detect if a message failed to send and retry, and so the server knows what's already been delivered (avoiding duplicates).
5. **Fallback:** For environments where WebSockets aren't available, **long polling** (client repeatedly asks "any new messages?" and the server holds the request open until there's data) is a common fallback.

---

## 5. URL Shortener

**Simple definition:** A service that takes a long URL and generates a short, unique code that redirects to the original — classic example: `bit.ly/abc123` → `https://very-long-url.com/...`.

### Core flow
```
1. POST /shorten { longUrl: "https://example.com/very/long/path" }
   → server generates a short code, stores mapping, returns short URL

2. GET /abc123
   → server looks up "abc123" in DB, finds original URL, issues a redirect (HTTP 301/302)
```

### How to generate the short code

**Option 1: Base62 encoding of an auto-incrementing ID**
- Use a database auto-increment counter (e.g., ID `125`) and encode it in base62 (`a-z`, `A-Z`, `0-9` = 62 characters) to produce a compact string.
```
ID 125 → base62 encode → "cb"
```
- **Pros:** Simple, no collisions possible (IDs are unique by definition).
- **Cons:** Sequential IDs are predictable/guessable, and a single counter can become a bottleneck at very high scale (mitigated with distributed ID generation, e.g., Twitter's Snowflake algorithm).

**Option 2: Hashing the long URL**
- Take a hash (e.g., MD5) of the long URL, then use the first 6-8 characters as the short code.
- **Cons:** Possible collisions (two different URLs producing the same short hash) — need a strategy to detect and resolve collisions (e.g., append a counter and re-hash).

### Database design
| Field | Description |
|---|---|
| `short_code` (Primary Key) | The generated short string, e.g., "abc123" |
| `long_url` | The original URL |
| `created_at` | Timestamp |
| `expires_at` | Optional expiration |
| `click_count` | Optional analytics counter |

### Key design considerations
1. **Read-heavy workload:** URL shorteners are read-heavy (many redirects per short link created) — a **cache** (Redis) in front of the database dramatically speeds up lookups, since the most popular links get hit repeatedly.
2. **Redirect type:**
   - `301 Moved Permanently` → browsers cache this redirect, meaning future clicks might not even hit your server again (good for performance, bad if you want accurate click analytics).
   - `302 Found` → not cached by browsers, so every click hits your server (better for analytics, slightly slower).
3. **Custom aliases:** Allow users to request a custom short code (e.g., `bit.ly/my-brand`) — requires checking for uniqueness before saving.
4. **Scaling:** Since reads vastly outnumber writes, this system scales well horizontally with **read replicas** and **caching**.
5. **Analytics:** Track click counts, geographic data, referrers — often handled asynchronously (via a message queue) so it doesn't slow down the actual redirect.

---

## 6. Notification System

**Simple definition:** A system that sends alerts to users through multiple channels (push notifications, email, SMS, in-app) triggered by events happening elsewhere in the application (e.g., "someone liked your post," "your order shipped").

### High-level architecture

```
┌──────────────┐   1. event occurs      ┌───────────────┐
│ App Service   │ ─────────────────────► │ Message Queue │  (e.g., Kafka, RabbitMQ, SQS)
│ (e.g., Orders)│  "order.shipped" event │               │
└──────────────┘                         └───────────────┘
                                                 │
                                                 ▼
                                        ┌─────────────────┐
                                        │ Notification      │
                                        │ Service (consumer) │
                                        └─────────────────┘
                                          │      │      │
                        ┌─────────────────┘      │      └─────────────────┐
                        ▼                         ▼                        ▼
                ┌──────────────┐        ┌──────────────┐         ┌──────────────┐
                │ Push Provider │        │ Email Provider│         │ SMS Provider  │
                │ (FCM/APNs)    │        │ (SendGrid/SES)│         │ (Twilio)      │
                └──────────────┘        └──────────────┘         └──────────────┘
```

### Why use a message queue instead of sending notifications directly?
- **Decoupling:** The "Orders" service just publishes an event ("order shipped") and moves on — it doesn't need to know or care how notifications are delivered. This keeps services independent and easier to maintain.
- **Reliability:** If the notification service is temporarily down, messages sit safely in the queue and get processed once it's back up — instead of being lost.
- **Scalability:** Multiple consumer instances can process notification events in parallel, and traffic spikes (e.g., a flash sale triggering thousands of notifications) don't overwhelm the core app.

### Key design considerations
1. **User preferences:** Store which channels each user wants for each notification type (e.g., "email me for orders, but not marketing") — checked before sending.
2. **Templates:** Notification content is usually templated (e.g., "Hi {{name}}, your order #{{orderId}} has shipped!") and rendered per-user.
3. **Retry logic:** If a provider (e.g., SendGrid) fails temporarily, retry with exponential backoff before giving up.
4. **Deduplication:** Avoid sending the same notification twice (e.g., using an idempotency key per event).
5. **Rate limiting/batching:** Avoid overwhelming users with too many notifications in a short time — batch similar notifications (e.g., "5 people liked your post" instead of 5 separate notifications).
6. **Delivery tracking:** Store notification status (sent, delivered, failed, read) for analytics and debugging.

---

## 7. Caching

**Simple definition:** Caching means **storing a copy of frequently accessed data in a fast-access location** (memory) so future requests for that data can be served instantly, without repeating an expensive operation (like a database query or external API call).

### Why caching matters
Databases are relatively slow compared to in-memory lookups. If the same data (e.g., a popular product's details) is requested thousands of times per minute, querying the database every single time wastes resources and adds latency. A cache stores that data in memory so most requests never even touch the database.

### Common caching strategies

**1. Cache-aside (lazy loading)** — most common pattern
```
1. App checks cache for data
2. If found (cache HIT) → return cached data
3. If not found (cache MISS) → query database, store result in cache, return data
```
```js
async function getProduct(id) {
  const cached = await redis.get(`product:${id}`);
  if (cached) return JSON.parse(cached); // cache HIT

  const product = await db.products.findById(id); // cache MISS
  await redis.set(`product:${id}`, JSON.stringify(product), "EX", 3600); // cache for 1 hour
  return product;
}
```

**2. Write-through** — data is written to the cache **and** the database at the same time, keeping them always in sync (slightly slower writes, but cache is never stale).

**3. Write-back (write-behind)** — data is written to the cache first, and asynchronously flushed to the database later (faster writes, but risk of data loss if the cache crashes before syncing).

### Cache invalidation
**Simple definition:** The process of removing or updating stale cached data when the underlying data changes — famously one of the "two hard things in computer science" (along with naming things).

- **TTL (Time-To-Live):** Simplest approach — cached data automatically expires after a set time (e.g., 1 hour), accepting some staleness in exchange for simplicity.
- **Explicit invalidation:** When data is updated (e.g., a product's price changes), explicitly delete/update the corresponding cache entry at that moment.
```js
async function updateProduct(id, newData) {
  await db.products.update(id, newData);
  await redis.del(`product:${id}`); // invalidate stale cache entry
}
```

### Where caching can be applied
| Layer | Example |
|---|---|
| Browser cache | Static assets (JS/CSS/images) cached via HTTP headers |
| CDN | Cached copies of pages/assets served from edge locations near the user |
| Application cache | Redis/Memcached storing DB query results, session data |
| Database cache | Query result caching built into the database engine itself |

---

## 8. Redis Basics

**Simple definition:** Redis (**RE**mote **DI**ctionary **S**erver) is an **in-memory data store** used as a cache, message broker, and lightweight database. It's extremely fast because it keeps data in RAM rather than on disk, and it supports rich data structures beyond simple key-value pairs.

### Why Redis is commonly used
- **Speed:** In-memory operations are orders of magnitude faster than disk-based databases (sub-millisecond response times).
- **Versatility:** Beyond basic caching, it supports data structures useful for many system design patterns (queues, leaderboards, rate limiting, pub/sub messaging).
- **Simplicity:** Simple command-based API, widely supported across every major language/framework.

### Core data structures

| Type | Description | Example use case |
|---|---|---|
| **String** | Simple key-value pair | Caching a serialized object, counters |
| **Hash** | A key mapping to multiple field-value pairs (like a mini object) | Storing a user's profile fields under one key |
| **List** | An ordered collection of values | Task queues, recent activity feed |
| **Set** | Unordered collection of unique values | Tracking unique visitors, tags |
| **Sorted Set (ZSet)** | Set with scores, automatically ordered | Leaderboards, rate limiting, priority queues |
| **Pub/Sub** | Publish/subscribe messaging channels | Real-time chat, live notifications between servers |

### Common commands
```bash
SET user:1:name "Alice"        # store a string
GET user:1:name                # retrieve it
EXPIRE user:1:name 3600        # set a TTL of 1 hour

HSET user:1 name "Alice" age 30   # store multiple fields (hash)
HGET user:1 name

LPUSH queue:tasks "task1"      # push to a list (used as a queue)
RPOP queue:tasks               # pop from the other end

SADD tags:post:1 "react" "javascript"  # add to a set
SISMEMBER tags:post:1 "react"          # check membership

ZADD leaderboard 1500 "Alice"  # sorted set with a score
ZRANGE leaderboard 0 -1 WITHSCORES  # get ranked results

PUBLISH chat-room-1 "Hello!"   # publish a message
SUBSCRIBE chat-room-1          # subscribe to receive messages
```

### Real-world Redis use cases (tying back to earlier topics)
- **Caching** (see above) — storing expensive DB query results.
- **Session storage** — storing session data for fast lookup across multiple app servers.
- **Rate limiting** — using counters with TTLs to track how many requests a client has made in a time window.
```js
const key = `rate_limit:${userId}`;
const requests = await redis.incr(key);
if (requests === 1) await redis.expire(key, 60); // reset counter every 60s
if (requests > 100) throw new Error("Rate limit exceeded");
```
- **Pub/Sub for chat/notifications** — relaying real-time events between multiple server instances (as discussed in the Chat Application section).
- **Leaderboards** — Sorted Sets naturally keep entries ranked by score, perfect for gaming leaderboards or "trending" content.
- **Distributed locks** — coordinating access to a shared resource across multiple servers (`SET key value NX EX 10` — set a key only if it doesn't already exist, with expiry, used to implement a simple lock).

### Redis persistence (it's in-memory, but not always volatile)
- **RDB (snapshotting):** Periodically saves a point-in-time snapshot of the dataset to disk.
- **AOF (Append-Only File):** Logs every write operation, allowing full reconstruction of data on restart.
- Redis can be configured with either or both, trading off performance vs durability.

---

## Quick Summary Cheat Sheet

| Concept | One-liner |
|---|---|
| Authentication Flow | Verify identity, issue access + refresh tokens, store securely in httpOnly cookies |
| File Upload | Use pre-signed URLs so clients upload directly to cloud storage, bypassing the app server |
| Chat Application | Use WebSockets for real-time bidirectional messaging, Pub/Sub to scale across servers |
| URL Shortener | Generate a unique short code (base62 ID or hash), cache reads heavily since it's read-dominant |
| Notification System | Decouple event producers from delivery via a message queue, fan out to push/email/SMS |
| Caching | Store frequently accessed data in memory to avoid repeated expensive DB queries |
| Redis Basics | In-memory data store with rich structures (String, Hash, List, Set, ZSet, Pub/Sub) used for caching, queues, rate limiting, and real-time messaging |

## Interview Tips for System Design Basics

1. **Always clarify requirements first** — ask about scale (how many users?), read vs write ratio, and latency requirements before designing.
2. **Start simple, then scale** — describe the naive/simple solution first, then explain what breaks at scale and how you'd fix it (this shows you understand trade-offs, not just buzzwords).
3. **Talk about trade-offs explicitly** — e.g., "301 vs 302 redirect," "SQL vs NoSQL," "WebSockets vs polling" — interviewers want to hear *why* you'd choose one over the other for this specific use case.
4. **Mention failure handling** — what happens if a service goes down, a message is lost, or a network call times out? Robust designs plan for failure, not just the happy path.
5. **Draw it out** — even simple boxes-and-arrows diagrams make your reasoning much easier to follow than a purely verbal explanation.
