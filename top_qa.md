# Top Interview Questions — System Design, Security, Performance & Production

*A consolidated, self-contained reference answering the most frequently asked full-stack/backend interview questions — each with a simple definition and a detailed, interview-ready explanation.*

---

## 1. Design an E-commerce System

**Core entities:** User, Product, Cart, Order, Payment, Inventory, Review.

**Simple approach:** Start with the main user journey — browse products → add to cart → checkout → pay → order confirmation — and map each step to a service/module.

```
┌────────┐  ┌───────────┐  ┌────────┐  ┌──────────┐  ┌───────────┐
│  User    │  │  Product    │  │  Cart   │  │  Order    │  │  Payment   │
│  Service  │  │  Service    │  │  Service │  │  Service  │  │  Service   │
└────────┘  └───────────┘  └────────┘  └──────────┘  └───────────┘
                                                            │
                                                    ┌──────────────┐
                                                    │  Inventory     │
                                                    │  Service       │
                                                    └──────────────┘
```

**Key design decisions:**
- **Database:** Relational (PostgreSQL) for Orders/Payments (need strong consistency, transactions); can use MongoDB for Product catalog (flexible attributes per category).
- **Search:** Elasticsearch for fast, relevance-ranked product search rather than querying the primary database directly.
- **Inventory consistency:** The hardest part — prevent overselling using atomic decrements (`UPDATE inventory SET stock = stock - 1 WHERE stock > 0`) or a short-lived stock **reservation** during checkout, released if payment isn't completed within a few minutes.
- **Checkout flow (multi-service consistency):** Use the **Saga pattern** — create order → charge payment → reserve inventory → confirm. If any step fails, compensating actions run (refund payment, cancel order) rather than a traditional cross-service rollback.
- **Payment idempotency:** Use an idempotency key per checkout attempt to prevent duplicate charges on network retries.
- **Caching:** Cache product listings/details in Redis (read-heavy, changes infrequently); invalidate on price/stock updates.

---

## 2. Design a Chat Application

**Simple definition of the core challenge:** Unlike typical HTTP request-response, chat needs the server to **push** messages to clients instantly — this requires **WebSockets**, a persistent, full-duplex connection between client and server.

```
Client A ◄──────► WebSocket Server ◄──────► Client B
                        │
                        ▼
                 Redis Pub/Sub (broadcasts across
                 multiple WebSocket server instances)
                        │
                        ▼
                 Database (persists message history)
```

**Key design points:**
- **Scaling across servers:** A single WebSocket server can't hold all connections at scale. If User A is connected to Server 1 and User B to Server 2, use **Redis Pub/Sub or Kafka** so any server can broadcast a message to users connected on any other server.
- **Message persistence:** Every message is saved to a database optimized for high write throughput (Cassandra/MongoDB), so chat history survives even if a user is offline when a message is sent.
- **Presence (online/offline):** Tracked in Redis with a TTL/heartbeat mechanism — if a client stops sending heartbeats, they're marked offline after the TTL expires.
- **Delivery guarantees:** Use message IDs + acknowledgments so the client can detect failed sends and retry, and the server can avoid re-delivering already-acknowledged messages.
- **Fallback:** Long polling for environments where WebSockets aren't available.

---

## 3. Design a Notification Service

**Simple definition:** A service that sends alerts to users through multiple channels (push, email, SMS, in-app), triggered by events elsewhere in the application — decoupled from the core business logic via a message queue.

```
Order Service → publishes "OrderShipped" event → Message Queue
                                                        │
                                    ┌───────────────────┼───────────────────┐
                                    ▼                    ▼                    ▼
                             Push Provider        Email Provider        SMS Provider
                             (FCM/APNs)           (SendGrid/SES)        (Twilio)
```

**Why a queue instead of calling providers directly?** Decoupling — the Order Service just publishes an event and moves on; if a notification provider is slow/down, it doesn't block the core order flow. It also allows independent scaling of notification workers during traffic spikes.

**Key design points:**
- **User preferences:** Check per-user, per-channel opt-in/out settings before sending.
- **Templates:** Render notification content from templates populated with per-user data.
- **Retry with backoff:** If a provider call fails temporarily, retry with increasing delay before giving up.
- **Deduplication:** Use an idempotency key per event to avoid sending the same notification twice.
- **Batching:** Group similar notifications (e.g., "5 people liked your post") instead of flooding the user with individual alerts.
- **Delivery tracking:** Store status (sent/delivered/failed/read) for analytics and debugging.

---

## 4. Design an Authentication System

**Simple definition:** A system that verifies user identity and maintains that verified state across subsequent requests.

**Standard modern flow:**
```
1. User submits credentials → Server verifies against bcrypt-hashed password in DB
2. Server issues: Access Token (JWT, short-lived) + Refresh Token (long-lived, stored server-side)
3. Both stored in httpOnly, Secure, SameSite cookies
4. Every request includes the Access Token → middleware verifies signature + expiry
5. When Access Token expires → client calls /refresh with the Refresh Token → gets a new Access Token
6. Logout → Refresh Token is revoked/deleted server-side
```

**Key design decisions:**
- **Stateless (JWT) vs stateful (sessions):** JWT scales better across multiple servers without a shared session store; sessions are simpler to revoke instantly. Full comparison in the JWT section below.
- **Password storage:** Always bcrypt/Argon2 hashed, never plain text or fast hashes like MD5/SHA1.
- **Multi-device support:** Store multiple refresh tokens per user (one per device/session) so logging out of one device doesn't affect others.
- **Third-party login:** OAuth 2.0 delegates identity verification to Google/GitHub, avoiding the need to manage passwords for those users at all.
- **Brute-force protection:** Rate limit login attempts, add account lockout/CAPTCHA after repeated failures.

---

## 5. Monolith vs Microservices

**Simple definition:** A **monolith** is a single, unified codebase and deployment unit handling all application functionality, typically sharing one database. **Microservices** split the application into independent, separately deployable services, each owning its own data, communicating over a network.

| Aspect | Monolith | Microservices |
|---|---|---|
| Deployment | One unit, deployed together | Independent, per service |
| Database | Usually shared | One per service |
| Scaling | Scale the whole app | Scale only what needs it |
| Team ownership | Harder to split cleanly | Teams can own services end-to-end |
| Complexity | Simple to start | More operational overhead (service discovery, distributed debugging) |
| Failure isolation | One crash can affect everything | A failure can be contained (with circuit breakers) |

**When to choose Monolith:** Small teams, early-stage products, unclear domain boundaries, or when the operational overhead of microservices isn't yet justified by a concrete scaling/organizational pain point. **"Monolith-first"** is a common, sensible approach — split into microservices only once you have a specific, real reason to.

**When to choose Microservices:** Clear domain boundaries, need for independent scaling of specific components, multiple teams needing to deploy independently without coordinating releases, or differing technology needs across parts of the system.

---

## 6. JWT Authentication Flow

**Simple definition:** JWT (JSON Web Token) is a self-contained, cryptographically signed token used to prove identity without the server needing to store session state.

**Structure:** `header.payload.signature` — the header and payload are base64-encoded (readable, NOT encrypted), and the signature proves the token hasn't been tampered with.

**Flow:**
```
1. Login: POST /login with credentials
2. Server verifies credentials → signs a JWT containing { userId, role, exp }
3. Server returns the JWT to the client (stored in an httpOnly cookie)
4. Client sends the JWT on every subsequent request: Authorization: Bearer <token>
5. Middleware verifies the signature (using the server's secret/public key) and checks expiry
6. If valid → request proceeds with req.user populated from the token's payload
7. If invalid/expired → 401 Unauthorized returned
```

```js
// Issuing
const token = jwt.sign({ id: user.id, role: user.role }, process.env.JWT_SECRET, { expiresIn: "15m" });

// Verifying (middleware)
jwt.verify(token, process.env.JWT_SECRET, (err, decoded) => {
  if (err) return res.status(401).json({ message: "Invalid or expired token" });
  req.user = decoded;
  next();
});
```

**Key point often asked:** JWTs are **not encrypted**, only signed — never store sensitive data (passwords, secrets) in the payload, since anyone can decode and read it; the signature only guarantees it wasn't altered.

---

## 7. Refresh Token Implementation

**Simple definition:** A long-lived, server-tracked token used solely to obtain a new access token once the short-lived one expires — avoiding forcing the user to log in again every 15 minutes.

**Why both tokens exist:** Access tokens are kept short-lived for security (limiting damage if stolen), but that alone would mean frequent re-logins — the refresh token bridges that gap.

```js
app.post("/refresh", async (req, res) => {
  const refreshToken = req.cookies.refreshToken;
  if (!refreshToken) return res.status(401).json({ message: "No refresh token" });

  const stored = await RefreshToken.findOne({ token: refreshToken });
  if (!stored) return res.status(403).json({ message: "Invalid refresh token" });

  jwt.verify(refreshToken, process.env.REFRESH_SECRET, (err, decoded) => {
    if (err) return res.status(403).json({ message: "Expired refresh token" });

    const newAccessToken = jwt.sign(
      { id: decoded.id, role: decoded.role },
      process.env.JWT_SECRET,
      { expiresIn: "15m" }
    );
    res.json({ accessToken: newAccessToken });
  });
});
```

**Best practices:**
- Store refresh tokens **server-side** (DB) so they can be explicitly revoked (e.g., on logout, or if compromised) — unlike access tokens, which remain valid until they naturally expire.
- Store in **httpOnly, Secure, SameSite cookies**, never `localStorage` (protects against XSS token theft).
- Implement **refresh token rotation**: issue a new refresh token every time one is used, invalidating the old one — if a stolen, already-rotated token is reused, it signals theft and all sessions can be revoked.

---

## 8. How Would You Scale a Node.js Application?

**Vertical scaling:** Bigger server (more CPU/RAM) — quick but has a hard ceiling and a single point of failure.

**Horizontal scaling (the standard approach):**
1. **Multiple instances behind a load balancer** — since Node.js is single-threaded per process, run one instance per CPU core using **Cluster mode** or **PM2**, plus multiple machines behind a load balancer for further scale.
2. **Statelessness:** Ensure the app doesn't store session state in memory (use Redis for shared session/cache state) so any instance can handle any request.
3. **Database scaling:** Add read replicas for read-heavy workloads; consider sharding if a single database can no longer handle write throughput.
4. **Caching:** Reduce database load with Redis caching for frequently-read data.
5. **Offload heavy work:** Move CPU-intensive tasks (image processing, report generation) to background workers/queues so they don't block the event loop for other requests.
6. **CDN:** Serve static assets from a CDN, not the Node.js server.

---

## 9. How Would You Optimize a Slow API?

**Step-by-step approach:**
1. **Measure first** — use APM/monitoring tools to identify whether the slowness is in the database, a third-party call, or application code, rather than guessing.
2. **Database:** Run `EXPLAIN` on suspect queries, add missing indexes, avoid `SELECT *`, fix N+1 query patterns.
3. **Caching:** Cache expensive, frequently-repeated results in Redis.
4. **Parallelize independent operations:** Use `Promise.all()` instead of sequential `await` calls when operations don't depend on each other.
5. **Reduce payload size:** Return only needed fields (avoid over-fetching), use pagination for large lists.
6. **Connection pooling:** Ensure database connections are reused, not opened per request.
7. **Offload non-critical work:** Move anything not needed for the immediate response (sending a notification email) to a background job.

---

## 10. How Would You Optimize React Performance?

1. **Memoization:** `React.memo` for components re-rendering unnecessarily due to parent re-renders; `useMemo` for expensive computations; `useCallback` to stabilize function references passed to memoized children.
2. **Code splitting & lazy loading:** Load route/component code only when needed (`React.lazy` + `Suspense`), reducing initial bundle size.
3. **Virtualization:** Use `react-window` for long lists — render only visible items instead of thousands of DOM nodes.
4. **Avoid unnecessary re-renders:** Don't create new object/array/function references inline in JSX for memoized children (defeats the memoization).
5. **Bundle optimization:** Tree shaking, analyzing bundle size to remove heavy/unused dependencies.
6. **Image optimization:** Compressed, responsive images, lazy-loaded off-screen.

**Golden rule to mention:** Memoize deliberately, based on a measured performance problem — over-memoizing everything adds comparison overhead that can outweigh the benefit for cheap components.

---

## 11. How Would You Optimize a SQL Query?

1. **Run `EXPLAIN`** to see the execution plan — look for full table scans instead of index usage.
2. **Add indexes** on columns used in `WHERE`, `JOIN`, and `ORDER BY`.
3. **Avoid `SELECT *`** — fetch only needed columns.
4. **Avoid wrapping indexed columns in functions** in `WHERE` clauses (`WHERE YEAR(date) = 2024` prevents index use — rewrite as a range: `WHERE date >= '2024-01-01' AND date < '2025-01-01'`).
5. **Rewrite subqueries as JOINs** where the optimizer handles joins more efficiently.
6. **Composite indexes:** For multi-column filters, create a compound index respecting the leftmost-prefix rule (see below).
7. **Use cursor-based pagination** instead of large `OFFSET` values for big datasets.

---

## 12. Redis Caching Strategy

**Simple definition:** Store frequently-accessed data in memory (Redis) to avoid repeated, expensive database queries.

**Cache-aside (most common pattern):**
```js
async function getProduct(id) {
  const cached = await redis.get(`product:${id}`);
  if (cached) return JSON.parse(cached);           // cache HIT

  const product = await db.products.findById(id);   // cache MISS
  await redis.set(`product:${id}`, JSON.stringify(product), "EX", 3600);
  return product;
}
```
- **Invalidation on write:** Delete/update the cache key whenever the underlying data changes, to avoid serving stale data.
- **TTL:** Set an expiry as a safety net even if explicit invalidation is missed.
- **Write-through/write-behind:** Alternative strategies where writes go to cache and DB together (always consistent, slower writes) or to cache first with async DB sync (fast, but risk of loss on cache failure).
- **Other Redis use cases:** Session storage, rate limiting counters, leaderboards (Sorted Sets), Pub/Sub for real-time features.

---

## 13. Database Indexing

**Simple definition:** A separate, sorted data structure (typically a B-Tree) that lets the database find matching rows without scanning the entire table — turning an O(n) search into roughly O(log n), at the cost of extra storage and slower writes.

```sql
CREATE INDEX idx_users_email ON users(email);
SELECT * FROM users WHERE email = 'alice@mail.com'; -- now uses the index
```

**Composite indexes** and the **leftmost-prefix rule:**
```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
-- Efficiently supports: WHERE user_id = 5, and WHERE user_id = 5 AND status = 'shipped'
-- Does NOT efficiently support: WHERE status = 'shipped' alone
```

**Trade-off to always mention:** Indexes speed up reads but slow down writes (every insert/update must also update the index) and consume extra storage — index based on actual query patterns, not every column.

---

## 14. API Versioning

**Simple definition:** Evolving an API over time without breaking clients that depend on the current behavior.

**Most common approach — URI versioning:**
```
GET /api/v1/users
GET /api/v2/users
```
Explicit, cacheable, easy to test directly — old and new versions can run simultaneously, giving clients time to migrate before deprecating the old one.

**Alternatives:** Header-based (`Accept-Version: v2`) or content negotiation (`Accept: application/vnd.myapp.v2+json`) — less discoverable but keep URLs clean.

---

## 15. Rate Limiting

**Simple definition:** Restricting how many requests a client can make within a time window, to prevent abuse, brute-force attacks, and overload.

```js
const rateLimit = require("express-rate-limit");

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,
  message: { message: "Too many requests, please try again later." },
});

app.use("/api/", limiter);
```
- Returns `429 Too Many Requests` when exceeded.
- Apply **stricter limits on sensitive endpoints** (login/signup) to reduce brute-force risk.
- Often implemented in Redis for distributed rate limiting across multiple server instances (a single in-memory counter per instance wouldn't work correctly behind a load balancer).

---

## 16. Prevent SQL Injection

**Simple definition:** SQL injection occurs when untrusted user input is concatenated directly into a SQL query, letting an attacker inject their own SQL logic.

```js
// ❌ Vulnerable
db.query(`SELECT * FROM users WHERE email = '${email}'`);

// ✅ Safe — parameterized query
db.query("SELECT * FROM users WHERE email = ?", [email]);
```
**Prevention:** Always use **parameterized queries** or an ORM — the database treats parameters strictly as data, never as executable SQL syntax, regardless of what's submitted. Never string-concatenate user input into a query.

---

## 17. Prevent XSS & CSRF

**XSS (Cross-Site Scripting):** Injecting malicious scripts into a page viewed by other users. **Prevention:** Never render untrusted input as raw HTML (escape/encode by default — React does this automatically unless `dangerouslySetInnerHTML` is used), sanitize any user-generated HTML with a library like DOMPurify, and set a `Content-Security-Policy` header.

**CSRF (Cross-Site Request Forgery):** Tricking a logged-in user's browser into making an unwanted request to your site (the browser auto-attaches cookies to any request to your domain, even from a malicious site). **Prevention:** Set the `SameSite` cookie attribute (`Strict` or `Lax`) to stop cookies from being sent on cross-site requests, and/or use CSRF tokens (a unique, unpredictable value the server issues and verifies on state-changing requests).

---

## 18. Secure File Uploads

1. **Validate file type and size** server-side (never trust client-side checks alone, since they can be bypassed).
2. **Scan for malware** before making uploaded files accessible.
3. **Use pre-signed URLs** so clients upload directly to cloud storage (S3), bypassing the app server for the actual file bytes.
4. **Randomize filenames** to prevent path traversal or overwrite attacks.
5. **Store outside the web-servable directory** (or in dedicated object storage) — never execute uploaded files.
6. **Set appropriate size limits** at both the application and reverse proxy level (e.g., Nginx's `client_max_body_size`).

---

## 19. Production Application Is Down — What Next?

1. **Confirm scope** — is it all users, one region, one service?
2. **Check monitoring/alerting dashboards** and recent deployment history for correlation.
3. **Communicate status early** to stakeholders, even with incomplete information.
4. **Prioritize restoring service** — if a recent deployment is the likely cause, **rollback first**, root-cause afterward.
5. Once stable, investigate logs (using correlation IDs if distributed) to find the actual cause, then conduct a postmortem/RCA to prevent recurrence.

**Key principle:** Restoring service takes priority over understanding the root cause while users are actively impacted.

---

## 20. API Returns 500 Only in Production

Check the actual production logs/stack trace first — don't guess. Common causes:
- **Environment-specific configuration** differences (missing/incorrect environment variables that exist correctly in dev/staging).
- A **dependency unreachable or misconfigured** in production (database, third-party API).
- A code path that **only triggers at production scale/data** (e.g., an edge case with real, messier data that doesn't appear in test datasets).

---

## 21. High CPU Usage in Production

Likely causes: an inefficient/blocking synchronous operation on the Node.js event loop (heavy computation, regex catastrophic backtracking), an infinite loop, or genuinely insufficient capacity for current traffic.

**Investigation:** Use profiling tools (Node's `--prof`, or `clinic.js`) to identify the specific hot function consuming CPU, and check whether the spike correlates with a recent deployment or a traffic increase.

**Fix:** Optimize/offload the CPU-heavy code (move it to a worker thread or separate service), or scale horizontally if it's genuinely a capacity issue.

---

## 22. Memory Leak Debugging

**Simple definition:** Memory usage grows continuously over time without being released, eventually leading to a crash (out-of-memory).

**Common causes:** Event listeners added but never removed, unbounded caches/arrays with no eviction, closures unintentionally retaining references to large objects, unclosed database/network connections.

**Debugging process:**
1. Take **heap snapshots** at intervals (Chrome DevTools/Node's `--inspect`) and compare them to identify what's growing.
2. Trace the growing object back to the responsible code path.
3. Fix the leak (remove listeners on cleanup, add cache size limits/TTLs, ensure connections are properly closed).
4. Add ongoing memory monitoring to catch regressions early going forward.

---

## 23. Database Connection Pool Exhaustion

**Simple definition:** All available database connections in the pool are in use, and new requests are stuck waiting (or failing) because none are free.

**Common causes:** Connection leaks (connections opened but never released, often due to missing `finally`/cleanup logic), an undersized pool for current traffic, or long-running queries holding connections open too long.

**What is connection pooling?** Instead of opening/closing a new connection per query (slow), a pool maintains a set of reusable, already-open connections that application code borrows and returns — reducing overhead and preventing the database from being overwhelmed by too many raw connections.

**Fix:** Audit code for unclosed connections, increase pool size if genuinely needed, add connection timeout/queueing configuration, and monitor pool utilization proactively.

---

## 24. Recent Deployment Broke Production

**Immediate action:** Roll back to the previous known-good version to restore service as quickly as possible — this should be a fast, automated process (most CI/CD systems and container registries retain previous versions for exactly this reason).

**After restoring service:** Diff the broken version against the previous one in a non-production environment to identify the specific breaking change, and add a test/check to prevent it recurring. Restoring service always comes before root-causing while users are impacted.

---

## 25. Docker Container Restart Loop

Check `docker logs <container>` for the actual crash reason first. Common causes:
- The container is hitting its **memory limit** (OOMKilled).
- Required **environment variables/dependencies** (database connectivity) aren't available at startup.
- A misconfigured **health check** incorrectly marking a healthy container as failed, triggering restarts.
- An unhandled exception crashing the process immediately on startup.

---

## 26. Nginx 502 Bad Gateway

**Simple definition:** Nginx (or another reverse proxy) received an invalid or no response from the upstream application server.

**Common causes:** The app server crashed or isn't running, it's still starting up (not yet listening on the expected port), or it timed out responding.

**Fix:** Check the application server's own logs directly (not just Nginx's logs) to find the real underlying cause, and verify the proxy's `proxy_pass` target and timeout settings are correctly configured.

---

## 27. SSL Certificate Expired

**Immediate fix:** Renew and reinstall the certificate as quickly as possible (via your CA or Let's Encrypt/Certbot).

**Long-term fix:** Set up **automated renewal** (Let's Encrypt certificates expire every 90 days and are typically auto-renewed via Certbot) and monitoring/alerting well before expiry (e.g., 30 days out) so this doesn't recur.

---

## 28. Kubernetes Pod CrashLoopBackOff

**Simple definition:** The pod's container keeps crashing shortly after starting, and Kubernetes is backing off (waiting longer between each restart attempt).

**Debugging steps:**
```bash
kubectl logs <pod> --previous   # see the CRASHED container's logs (current one may have just restarted)
kubectl describe pod <pod>      # check events for scheduling/resource/probe issues
```
**Common causes:** Missing ConfigMaps/Secrets the app depends on at startup, a failed dependency connection (database not reachable), or resource limits set too low causing the container to be killed.

---

## 29. Zero Downtime Deployment

**Simple definition:** Deploying a new version of an application without any period where users experience errors or downtime.

**How it's achieved:**
- Use **Rolling, Blue-Green, or Canary** deployment strategies (rather than stopping the old version before starting the new one) — see the next question.
- Load balancer only routes to instances passing **readiness probes**.
- Database migrations must be **backward-compatible** with the previous code version during the transition window, since old and new code may briefly run simultaneously.

---

## 30. Blue-Green vs Rolling Deployment

**Blue-Green:** Maintain two identical environments — "Blue" (live) and "Green" (new version). Deploy and fully test Green while Blue serves all traffic, then switch the router instantly once verified.
```
Before: Router → Blue (v1, live)      Green (v2, idle, testing)
After:  Router → Green (v2, live)     Blue (v1, idle, instant rollback)
```
**Pros:** Instant rollback, zero downtime, full production-like testing before real traffic hits it. **Cons:** Requires double the infrastructure simultaneously.

**Rolling:** Gradually replace old instances with new ones, one (or a small batch) at a time — both versions serve live traffic simultaneously during the rollout.
```
[v1][v1][v1][v1] → [v2][v1][v1][v1] → [v2][v2][v1][v1] → [v2][v2][v2][v2]
```
**Pros:** No extra infrastructure needed. **Cons:** Slower rollback (must roll back instance by instance), and both versions must be compatible with each other and the shared database schema during the transition.

**Canary (bonus, closely related):** Roll out to a small percentage of real traffic first, monitor closely, then gradually increase — limits the "blast radius" of a bad deployment.

---

## 31. Rollback Strategy

**For code:** Redeploy the previous known-good artifact/image — this should be a fast, tested, ideally automated process (Kubernetes: `kubectl rollout undo`).

**For database changes:** Write migrations with both `UP` and `DOWN` scripts so schema changes can be reversed; ensure schema changes are backward-compatible with the previous app version during any transition.

**For distributed transactions:** Use **compensating transactions** as part of a Saga (e.g., a refund to reverse a completed charge) since a true cross-service rollback isn't possible.

---

## 32. CI/CD Pipeline Explanation

**Simple definition:** CI (Continuous Integration) automatically builds and tests every code change; CD (Continuous Delivery/Deployment) automatically ships passing changes toward (or into) production.

**Typical pipeline:**
```
Code pushed → Automated tests run (CI) → Build artifact/Docker image
  → Deploy to staging → (approval/automated checks) → Deploy to production
  (Rolling/Blue-Green/Canary) → Post-deployment monitoring
```
**Why it matters:** Removes manual, error-prone deployment steps, and with many independently deployable services, manual deployment simply doesn't scale — CI/CD is what makes "independent deployment" a practical reality. Common tools: GitHub Actions, Jenkins, GitLab CI.

---

## 33. Saga Pattern

**Simple definition:** Manages data consistency across multiple services in a distributed transaction by breaking it into a sequence of **local transactions**, each committed independently within its own service. If a step fails, previously completed steps are undone via **compensating transactions** (explicit reversing actions), since a true cross-database rollback isn't possible.

```
1. Create Order        ✅
2. Process Payment      ✅
3. Reserve Inventory    ❌ FAILS
   → Compensate: Refund Payment, Cancel Order
```

**Two implementation styles:**
- **Choreography:** No central coordinator — each service listens for events and reacts, publishing its own events in turn. Simple, decentralized, but harder to see the overall flow as it grows.
- **Orchestration:** A central orchestrator explicitly tells each service what to do next and handles failure/compensation logic. Easier to understand and monitor, but the orchestrator becomes a critical, potentially complex component.

---

## 34. Circuit Breaker Pattern

**Simple definition:** Prevents a service from repeatedly calling a downstream service that's known to be failing — instead of piling up doomed requests, the circuit "trips" and immediately returns a fallback, giving the failing service time to recover.

**Three states:**
| State | Behavior |
|---|---|
| Closed | Normal operation — requests pass through |
| Open | Failures exceeded threshold — requests immediately fail/fallback without attempting the call |
| Half-Open | After a cooldown, a few test requests are allowed through to check recovery |

**Why it matters:** Without it, a slow/down dependency can cause calling services' threads/connections to pile up waiting for timeouts, cascading the failure into services that were otherwise healthy. Popular libraries: Resilience4j, (historically) Hystrix.

---

## 35. Retry with Exponential Backoff

**Simple definition:** Automatically re-attempting a failed operation, since many distributed system failures are transient (brief network blips) — with the wait time between attempts **increasing exponentially** (1s, 2s, 4s, 8s...) rather than retrying immediately and repeatedly.

**Why exponential backoff specifically:** Retrying immediately and repeatedly can overwhelm an already-struggling service with a burst of retry traffic; increasing delays give it room to recover.

**Important pairing:** Only retry **idempotent** operations (or ones made idempotent via an idempotency key) — otherwise retries can cause duplicate side effects (e.g., double payments). Combine with a maximum retry count and a circuit breaker to stop retrying against a service that's genuinely down long-term.

---

## 36. Idempotency in APIs

**Simple definition:** An operation is idempotent if making the same request multiple times produces the same end result as making it once — safe to retry without unintended side effects.

| Method | Idempotent? |
|---|---|
| GET, PUT, DELETE | ✅ Yes |
| POST | ❌ No — typically creates a new resource each time |

**Real-world technique — Idempotency Keys:** The client generates a unique key per logical operation (e.g., a UUID for one checkout attempt); the server checks if that key was already processed and returns the original result instead of reprocessing — critical for payment/order endpoints where a network retry shouldn't cause a duplicate charge.

---

## 37. Handling Duplicate Orders

Almost always an **idempotency failure** — a user double-clicking "submit," or a network retry after a timeout, hitting the order-creation endpoint twice.

**Fix:** Implement an idempotency key on the checkout/order-creation endpoint — the client sends a unique key per checkout attempt, and the server deduplicates based on that key rather than creating a new order each time it's called. Also check for race conditions if multiple requests for the same cart/session could be processed concurrently, and consider a short-lived lock on the cart during checkout.

---

## 38. Third-Party API Failures

**Combine three resilience patterns:**
1. **Timeout** — don't let a call hang indefinitely, tying up resources.
2. **Retry with exponential backoff** — for transient failures.
3. **Circuit breaker** — stop repeatedly calling a provider that's clearly down for an extended period.
4. **Fallback** — cached data, a degraded experience, or a queued retry, so your own system doesn't fail just because a dependency did.

Also monitor the provider's own status page and check whether your own request volume is triggering their rate limits.

---

## 39. Logging and Monitoring Strategy

**Logging:** Centralize logs from all services into one system (ELK stack or Grafana Loki), and propagate a **correlation/trace ID** through every service call in a request, so logs from all services touched by one request can be searched together.

**Monitoring:** Track the **"RED" metrics** — Rate, Errors, Duration (ideally p95/p99 latency, not just averages) — using Prometheus + Grafana dashboards. Implement **health check endpoints** for load balancers/orchestrators, and configure **alerting** so the team is notified before users report issues.

---

## 40. Root Cause Analysis (RCA)

**Process:**
1. Use the **"5 Whys"** technique — repeatedly ask why the symptom occurred until reaching the actual underlying cause, not a superficial explanation.
2. Base conclusions on **actual data** (logs, metrics, traces from the real incident), not assumptions.
3. Document: what happened, the timeline, the actual root cause, impact, what went well/poorly in the response, and concrete action items with owners.
4. Conduct it in a **blameless** format, focused on systemic fixes rather than individual blame — this encourages honest reporting and better long-term fixes.

---

## 41. Optimizing MongoDB Queries

1. **Add indexes** matching actual query patterns, including compound indexes in the correct field order.
2. **Use `.explain()`** to verify indexes are actually being used, not a collection scan.
3. **Project only needed fields** (`$project`) rather than returning entire documents.
4. **Place `$match`/`$project` early** in aggregation pipelines to reduce data volume flowing into later, more expensive stages.
5. **Avoid unnecessary `$lookup`s** by considering embedding for frequently-joined, small/bounded related data.

---

## 42. Composite Indexes

**Simple definition:** An index built across **multiple columns/fields together**, used when queries frequently filter or sort by more than one field at once.

```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```
**Leftmost-prefix rule:** This index efficiently supports queries filtering by `user_id` alone, or `user_id AND status` together — but **not** a query filtering only by `status`, since the index is physically sorted by `user_id` first. Column order in the index definition should match the most common query patterns.

---

## 43. Cursor vs Offset Pagination

**Offset pagination** (`LIMIT 20 OFFSET 1000`): The database must scan and discard all preceding rows before returning results — progressively slower at higher offsets, and can return inconsistent results if rows are inserted/deleted between page requests.

**Cursor-based pagination** (`WHERE id > last_seen_id LIMIT 20`): Uses a stable pointer (e.g., the last-seen record's ID) — consistently fast (uses an index seek, not a scan) regardless of how deep into the results you are, and stable even as underlying data changes.

**When to use which:** Cursor-based is strongly preferred for large or frequently-changing datasets (social feeds, activity logs); offset pagination is acceptable for small, relatively static datasets where jumping to an arbitrary page number is a genuine UX requirement.

---

## 44. Background Job Processing

**Simple definition:** Offloading slow or non-critical-path work (sending emails, generating reports, processing images/video) to a **job queue**, processed by separate worker processes — keeping the main API responsive.

```
API Request → Enqueue Job → (immediate response to user) 
                    │
                    ▼
              Job Queue (Redis/BullMQ, RabbitMQ, SQS)
                    │
                    ▼
              Worker Process(es) → performs the actual slow work
```
**Benefits:** The user-facing request returns quickly, workers can be scaled independently based on queue depth, and failed jobs can be retried without affecting the original request. A **dead letter queue** isolates jobs that repeatedly fail, so they don't block the rest of the queue.

---

## 45. Health Checks

**Simple definition:** An endpoint (e.g., `GET /health`) a service exposes to report whether it's running correctly — including whether its critical dependencies (database, cache) are reachable — used by load balancers/orchestrators to decide whether to route traffic to that instance or restart it.

```js
app.get("/health", async (req, res) => {
  const dbHealthy = await checkDatabaseConnection();
  if (!dbHealthy) return res.status(503).json({ status: "unhealthy" });
  res.status(200).json({ status: "healthy" });
});
```
**Distinguishing liveness vs readiness** (common in Kubernetes): **Liveness** checks if the app is running at all (restart if it fails); **readiness** checks if it's ready to receive traffic right now (temporarily remove from load balancer rotation if it fails, without restarting).

---

## 46. Load Balancing

**Simple definition:** Distributing incoming requests across multiple instances of a service so no single instance is overwhelmed while others sit idle.

**Common algorithms:**
- **Round Robin:** Requests distributed sequentially across instances in turn.
- **Least Connection:** Routes to whichever instance currently has the fewest active connections — better when request processing times vary significantly.
- **Sticky Session:** Routes a specific client's requests consistently to the same instance (useful if session state is stored in-memory on that instance, though storing sessions externally in Redis removes the need for stickiness entirely).

---

## 47. CDN Usage

**Simple definition:** A Content Delivery Network is a geographically distributed network of servers that cache and serve static content (images, JS, CSS, videos) from a location physically close to the end user, dramatically reducing latency compared to serving everything from one origin server.

**Where to use it:** Static assets (JS/CSS bundles, images, fonts), video streaming, and even caching full HTML pages for content that doesn't change per-user (e.g., a blog). Offloads significant traffic away from your origin servers, and improves load times for geographically distant users.

---

## 48. File Upload Architecture

**Simple definition:** Rather than routing uploaded file bytes through the application server (which doesn't scale well for large files), use **pre-signed URLs** so clients upload directly to cloud object storage.

```
1. Client: "I want to upload photo.jpg" → App Server
2. App Server generates a short-lived, scoped pre-signed URL → returns it to Client
3. Client uploads the file DIRECTLY to Cloud Storage (S3/GCS) using that URL
4. App Server saves file metadata (URL, size, owner) to the database
```
**Why this design:** The app server never touches the file bytes, keeping it lightweight and avoiding memory/bandwidth bottlenecks. For very large files, use **chunked/multipart uploads** to support pause/resume and parallel upload of pieces. Always validate file type/size and scan for malware before making uploaded files publicly accessible.

---

## 49. Real-time Communication with WebSockets

**Simple definition:** WebSockets establish a persistent, full-duplex connection between client and server — once opened, either side can send data at any time, without the overhead of repeated HTTP request-response cycles.

```
Client ──── HTTP handshake (upgrade to WebSocket) ────► Server
Client ◄─────────── persistent open connection ───────► Server
```

**Scaling across multiple servers:** A single WebSocket server can't hold every connection at scale. Use **Redis Pub/Sub** or **Kafka** so any server instance can broadcast a message to clients connected to any other instance — this is essential for any real-time feature (chat, live notifications, collaborative editing) running behind a load balancer with multiple server instances.

**Fallback:** Long polling for environments/proxies that don't support WebSockets well.

---

## 50. Designing for High Availability

**Simple definition:** Designing a system so it remains operational (or degrades gracefully) even when individual components fail — measured by "uptime" (e.g., "five nines" = 99.999% availability).

**Key techniques:**
1. **Redundancy:** Run multiple instances of every critical component (app servers, databases) so a single failure doesn't cause an outage.
2. **Load balancing with health checks:** Automatically route traffic away from unhealthy instances.
3. **Database replication:** Read replicas for read scaling and failover; automated failover to a standby if the primary fails.
4. **Multi-region/multi-AZ deployment:** Protects against a single data center/region outage.
5. **Circuit breakers and graceful degradation:** If a non-critical dependency fails (e.g., a recommendation service), the core functionality (browsing/checkout) should still work, just without that feature — rather than the whole system going down.
6. **Automated failover and self-healing:** Orchestrators like Kubernetes automatically restart crashed containers and reschedule them on healthy nodes.
7. **Monitoring and alerting:** Detect degradation early, before it becomes a full outage.

**Trade-off to mention:** High availability adds cost and complexity (redundant infrastructure, cross-region data replication challenges) — the right level of investment depends on the actual business cost of downtime for that specific system.

---

## Quick Cross-Reference Cheat Sheet

| Theme | Core Idea |
|---|---|
| Idempotency | Same request repeated = same result — critical for retries, duplicate orders, payments |
| Circuit Breaker + Retry + Timeout | Stop cascading failures from a struggling dependency |
| Saga Pattern | Distributed transactions via local steps + compensating actions |
| Caching (cache-aside + TTL + invalidation) | Reduce load on slow, expensive data sources |
| Indexing (single/composite, leftmost-prefix) | Turn full table scans into fast lookups |
| Correlation ID + Centralized Logging | Trace one request across many services |
| Health Checks + Load Balancing | Keep traffic only on healthy instances |
| Blue-Green / Rolling / Canary | Deploy safely with fast rollback options |
| Cursor Pagination | Consistent, fast pagination for large/changing datasets |
| RCA (5 Whys, blameless) | Turn incidents into lasting systemic fixes |
