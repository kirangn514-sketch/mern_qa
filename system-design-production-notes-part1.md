# System Design & Production Engineering — Interview Notes

*This is a large reference covering system design scenarios, security, performance, and production operations. Each question includes a simple definition and a detailed, interview-ready explanation.*

---

# Section 1: Design Architecture for a New Project

**How to approach ANY "design X application" question (use this framework for all 25 below):**
1. **Clarify scope** — ask about scale (users, requests/sec), core features (MVP vs full), and constraints.
2. **Identify core entities/domains** → map to services or modules (e.g., Users, Orders, Payments).
3. **Choose architecture** — monolith for smaller scope/teams, microservices if scale/team size justifies it.
4. **Pick data stores** — relational (SQL) for structured, relational data needing transactions; NoSQL (MongoDB) for flexible/nested data; Redis for caching/sessions.
5. **Define communication** — REST for direct client-facing APIs; async messaging (Kafka/RabbitMQ) for background/cross-service events.
6. **Address cross-cutting concerns** — auth, caching, rate limiting, logging, monitoring.
7. **Discuss trade-offs and failure handling** explicitly — this is what separates a strong answer from a shallow one.

Below are the specific considerations that make each of these applications distinct:

**E-commerce application:** Core entities: User, Product, Cart, Order, Payment, Inventory. Key challenges: inventory consistency during checkout (use a Saga pattern or DB-level locking/reservation), search (Elasticsearch for product search), payment idempotency (avoid double charges).

**Hospital Management System:** Core entities: Patient, Doctor, Appointment, Prescription, Billing. Key challenges: strict data privacy/compliance (HIPAA-like regulations), complex role-based access (doctors, nurses, admin, patients each see different data), audit logging of every record access/change.

**Food Delivery application:** Core entities: Restaurant, MenuItem, Order, DeliveryPartner, Payment. Key challenges: real-time order tracking (WebSockets), matching orders to nearby delivery partners (geospatial queries), handling high read traffic on restaurant/menu browsing (heavy caching).

**Employee Management System:** Core entities: Employee, Department, Attendance, Payroll, Leave. Key challenges: role-based permissions (HR vs manager vs employee views), payroll calculation accuracy/auditability, integration with attendance/biometric systems.

**CRM application:** Core entities: Lead, Contact, Deal/Opportunity, Activity/Task, Company. Key challenges: complex reporting/analytics on sales pipelines, multi-tenancy if sold as SaaS to multiple companies, integration with email/calendar systems.

**School Management System:** Core entities: Student, Teacher, Class, Grade, Attendance, Fee. Key challenges: role-based access (parent, teacher, admin), report card generation, fee payment tracking.

**Chat Application:** Covered in depth in the System Design notes — WebSockets for real-time messaging, Pub/Sub (Redis/Kafka) to scale across server instances, message persistence in a write-optimized database.

**URL Shortener:** Covered in depth in the System Design notes — base62 encoding or hashing for short codes, heavy read caching since it's read-dominant.

**Notification Service:** Covered in depth in the System Design notes — message queue decoupling event producers from multi-channel delivery (push/email/SMS), retry with backoff, user preference management.

**Authentication & Authorization system:** Covered in depth in the Authentication notes — JWT access + refresh tokens, RBAC for authorization, secure cookie storage (httpOnly, SameSite).

**File Upload Service:** Covered in depth in the System Design notes — pre-signed URLs for direct-to-cloud-storage uploads, chunked uploads for large files, virus scanning before making files accessible.

**Inventory Management System:** Core entities: Product, Warehouse, StockLevel, StockMovement. Key challenges: preventing overselling (atomic stock decrement operations, or optimistic locking with version numbers), handling concurrent updates from multiple channels (online store + physical POS), audit trail of every stock change.

**Payment Gateway Integration:** Core concerns: PCI-DSS compliance (never store raw card numbers — use a tokenization service like Stripe/Razorpay), idempotency keys to prevent duplicate charges on retry, webhook handling for asynchronous payment status updates, reconciliation processes to catch mismatches.

**Multi-Tenant SaaS application:** **Simple definition of multi-tenancy:** A single application instance serves multiple customers ("tenants"), with each tenant's data logically isolated from others. Approaches: (1) **Separate databases per tenant** — strongest isolation, more operational overhead; (2) **Shared database, separate schemas** — middle ground; (3) **Shared database, shared schema with a `tenant_id` column** — most common, cheapest to operate, but requires strict enforcement (every single query must filter by `tenant_id`) to avoid data leaks between tenants.

**Online Booking System:** Core entities: Resource (room/slot/seat), Booking, User, Payment. Key challenge: preventing double-booking of the same resource/slot — typically solved with database-level unique constraints or optimistic locking on the specific time-slot record during the booking transaction.

**Order Management System:** Covered in the Microservices notes example (Order, Payment, Inventory, Notification services) — key challenge is coordinating the multi-step order lifecycle (created → paid → fulfilled → shipped) with proper state transitions and Saga-based rollback if any step fails.

**Blog/CMS platform:** Core entities: Post, Author, Category, Comment, Media. Key challenges: content versioning/drafts, SEO (server-rendered pages, metadata), rich media handling (image optimization/CDN), caching for high-read public content.

**Online Examination System:** Core entities: Exam, Question, Student, Submission, Result. Key challenges: preventing cheating (time limits enforced server-side, tab-switch detection, randomized question order), handling simultaneous exam start for thousands of students (load spike planning), auto-grading logic.

**Video Streaming Platform:** Core concerns: video transcoding into multiple resolutions/formats (usually via a background processing pipeline), adaptive bitrate streaming (HLS/DASH), CDN for global content delivery, storing large binary files in object storage (S3) rather than a database.

**Social Media application:** Core entities: User, Post, Comment, Like, Follow. Key challenges: feed generation at scale (fan-out on write vs fan-out on read — precomputing feeds for users with few followers, computing on-demand for celebrity accounts with millions of followers), handling viral content spikes, real-time notifications.

**Workflow Management System:** Core entities: Workflow, Step/Stage, Task, Approver. Key challenge: modeling a flexible state machine that can represent different approval chains/business processes without hardcoding logic per workflow type — often implemented via a configurable state machine or workflow engine pattern.

**Ticket Booking System:** Similar to Online Booking, with the added challenge of extremely high concurrent demand at "on-sale" moments (e.g., popular concert tickets) — requires strategies like a virtual waiting room/queue, short-lived seat reservations (hold a seat for a few minutes during checkout, then release if payment isn't completed), and heavy read caching for seat-map availability.

**Real-time Dashboard:** Core concerns: streaming live data updates to the frontend (WebSockets or Server-Sent Events), aggregating/pre-computing metrics rather than running expensive queries on every page load, time-series data storage for historical trends.

**Audit Logging System:** Core requirement: an **immutable**, append-only log capturing who did what, when, and the before/after state of changed data (covered in the organization's audit logging standards: eventType, entityType, entityId, performedBy, performedAt, oldValue, newValue) — should never be editable or deletable by application logic, often stored in a separate, write-optimized store.

**Email & SMS Notification Service:** A specific instance of the general Notification Service pattern — key considerations: provider abstraction (ability to switch between SendGrid/SES for email or Twilio for SMS without changing business logic), delivery status tracking via provider webhooks, template management, and compliance with opt-out/unsubscribe requirements.

---

# Section 2: Architecture Scenario-Based Questions

**Q: Why did you choose Microservices instead of Monolith?**
Typically justified by specific pain points: independent scaling needs (some services need far more resources than others), independent deployment (avoiding one team's release blocking another's), team autonomy (multiple teams working without constant coordination), or technology diversity needs. The honest answer should reference a concrete problem microservices solved — not "because it's modern."

**Q: How would you divide services in a large application?**
Split along **business domain boundaries** (Domain-Driven Design's "bounded contexts") rather than technical layers — e.g., a full "Order" service handling everything order-related, not a separate "database layer service" and "business logic service." Each service should have high cohesion (focused on one domain) and loose coupling (minimal dependency on other services' internals).

**Q: How would services communicate?**
Synchronous (REST/gRPC) when the caller needs an immediate response to proceed; asynchronous (Kafka/RabbitMQ) when the operation can happen independently or needs to notify multiple interested services without tight coupling. See the Microservices notes for full detail.

**Q: Which database would you choose and why?**
Depends on data shape and access patterns: relational (PostgreSQL/MySQL) for structured data with relationships requiring strong consistency and transactions (financial data, orders); MongoDB for flexible, nested, rapidly-evolving schemas; Redis for caching, sessions, and leaderboards; Elasticsearch for full-text search; a time-series database (InfluxDB) for metrics/analytics workloads.

**Q: How would you implement authentication?**
JWT-based stateless authentication with short-lived access tokens and longer-lived, revocable refresh tokens, stored in httpOnly/Secure/SameSite cookies — validated at the API Gateway for microservices, or via middleware in a monolith. See the Authentication notes for complete detail.

**Q: How would you implement authorization?**
Role-Based Access Control (RBAC) — assign users roles (admin, editor, viewer), and check the authenticated user's role against required permissions in middleware before allowing an action. For more granular needs, Attribute-Based Access Control (ABAC) considers additional context (e.g., "can edit only their own posts").

**Q: How would you scale the application?**
**Vertically** (bigger servers — quick but has a ceiling) and **horizontally** (more server instances behind a load balancer — the standard approach for real scale). Combine with caching (reduce database load), database read replicas (scale reads independently from writes), and asynchronous processing (offload slow work to background jobs/queues).

**Q: How would you handle millions of users?**
Horizontal scaling of stateless application servers behind a load balancer, database read replicas and/or sharding, aggressive caching (Redis/CDN) for frequently accessed data, asynchronous processing for non-critical-path work, and a CDN for static assets — combined with proper monitoring to identify the actual bottleneck rather than scaling blindly.

**Q: How would you handle concurrent requests?**
At the application level: ensure code is stateless so any instance can handle any request. At the database level: use appropriate locking (optimistic locking with version numbers for low-contention scenarios, pessimistic locking for high-contention critical sections) or atomic operations (`UPDATE ... SET stock = stock - 1 WHERE stock > 0`) to prevent race conditions like overselling inventory.

**Q: How would you handle duplicate API requests?**
Use **idempotency keys** — the client generates a unique key per logical operation (e.g., a UUID for a specific checkout attempt); the server checks if that key has already been processed and returns the cached original result instead of reprocessing, preventing issues like double-charging a customer on a network retry.

**Q: How would you implement caching?**
Cache-aside pattern with Redis: check cache first, fall back to the database on a miss, then populate the cache; use TTLs for automatic staleness handling, and explicit invalidation when the underlying data changes. See the Caching section in the System Design notes for full detail and code.

**Q: How would you implement file uploads?**
Use pre-signed URLs so clients upload directly to cloud object storage (S3/GCS), bypassing the application server entirely for the actual file bytes — the app server only issues a temporary, scoped upload URL and later records metadata. See the System Design notes for the full flow diagram.

**Q: How would you implement notifications?**
Publish an event when something notification-worthy happens (e.g., "OrderShipped"), consumed by a dedicated Notification Service that fans out to the appropriate channels (push/email/SMS) based on user preferences — decoupled via a message queue so the core business flow isn't blocked by notification delivery.

**Q: How would you implement audit logs?**
Capture an immutable record for every create/update/delete on sensitive data: event type, entity type/ID, who performed it, when (UTC), and the old/new values — write to an append-only store, never allow edits/deletes of audit records, and never log sensitive fields like passwords or tokens.

**Q: How would you implement API versioning?**
URI versioning (`/api/v1/`, `/api/v2/`) is the most common and explicit approach — allows old and new versions to run simultaneously, giving clients time to migrate before deprecating the old version. See the REST API notes for alternative approaches (header/content-negotiation versioning).

**Q: How would you design a search feature?**
For simple exact/prefix matching, a database index may suffice. For full-text, fuzzy, or relevance-ranked search across large datasets, use a dedicated search engine like **Elasticsearch** or **Algolia** — the application publishes/syncs data changes into the search index (often asynchronously via events) so search queries never hit the primary database directly.

**Q: How would you handle third-party API failures?**
Apply a **circuit breaker** to stop repeatedly calling a failing third party, combined with **retries with exponential backoff** for transient failures, a sensible **timeout** so calls don't hang indefinitely, and a **fallback** response/behavior (cached data, degraded functionality, or a queued retry) so your own system doesn't fail just because a dependency did.

**Q: How would you implement retry mechanisms?**
Retry only for transient, likely-recoverable errors (timeouts, 503s) — not for client errors (400s) that will fail identically every time. Use exponential backoff (increasing delay between attempts) plus a maximum retry count, and ensure the retried operation is **idempotent** so retries don't cause duplicate side effects (e.g., double payments).

**Q: How would you handle background jobs?**
Offload slow or non-critical-path work (sending emails, generating reports, processing images) to a **job queue** (BullMQ/Redis, RabbitMQ, or SQS) processed by separate worker processes — this keeps the main API responsive and allows independent scaling of worker capacity based on queue depth.

**Q: How would you schedule cron jobs?**
For a single-server setup, a simple library (`node-cron`) suffices. For distributed/multi-instance systems, use a dedicated scheduler (e.g., a cloud provider's scheduled functions, or a distributed cron tool like `node-cron` combined with a distributed lock in Redis) to ensure the job runs exactly once, not once per running instance.

**Q: How would you design a reporting module?**
Avoid running expensive aggregate queries directly against the live production/transactional database. Instead, use a **read replica** or a separate **data warehouse**, and often pre-compute/aggregate reports on a schedule (batch jobs) rather than calculating them on-demand for every request — especially for historical or large-scale analytics.

**Q: How would you secure APIs?**
HTTPS everywhere, authentication (JWT/OAuth) on every protected route, authorization checks (RBAC), input validation/sanitization, rate limiting, security headers (via Helmet), CORS configured to specific trusted origins, and secrets stored outside of code (environment variables/secret managers) — see the full Security section below for depth.

**Q: How would you monitor services?**
Track the "RED" metrics (Rate, Errors, Duration) per service using Prometheus + Grafana dashboards, implement health check endpoints for load balancers/orchestrators, and set up alerting for anomalies (error rate spikes, latency increases) so the team is notified before users report issues.

**Q: How would you implement centralized logging?**
Aggregate logs from all services into one system (ELK stack or Grafana Loki), and propagate a **correlation/trace ID** through every service call in a request so logs from all services involved in one request can be searched and reconstructed together. See the Microservices notes for detail.

**Q: How would you implement health checks?**
Expose a `/health` endpoint that checks the service's own status plus its critical dependencies (database connection, cache availability) — return a simple healthy/unhealthy status that load balancers and orchestrators (Kubernetes) use to decide whether to route traffic to that instance or restart it.

**Q: How would you implement service discovery?**
In Kubernetes, this is largely handled automatically via internal DNS and Services. Outside Kubernetes, use a dedicated registry (Consul, Eureka) where service instances register themselves on startup and other services query the registry for current addresses instead of hardcoding them. See the Microservices notes for full detail.

**Q: How would you implement distributed transactions?**
Use the **Saga pattern** — break the operation into a sequence of local transactions per service, with compensating actions to undo previously completed steps if a later step fails, since a traditional cross-database ACID transaction isn't possible across independent services. See the Microservices notes for the full explanation and choreography vs orchestration comparison.

**Q: How would you ensure data consistency?**
Within a single service/database, rely on ACID transactions. Across services, accept **eventual consistency** via the Saga pattern and event-driven updates, and design the UI/business logic to tolerate a brief window of inconsistency (e.g., showing "processing" states) rather than assuming instant consistency everywhere.

**Q: How would you implement rollback strategies?**
For a single-database operation, wrap it in a transaction and `ROLLBACK` on failure. For distributed operations, implement explicit **compensating transactions** (e.g., a refund to reverse a completed charge) as part of the Saga flow. For deployments, maintain the ability to quickly redeploy the previous known-good version (see Blue-Green/Rolling deployment strategies below).

**Q: What design patterns would you use?**
Common patterns in backend systems: **Repository pattern** (abstracting data access logic away from business logic), **Factory pattern** (creating objects without specifying the exact class), **Singleton** (single shared instance, e.g., a database connection pool), **Strategy pattern** (swapping algorithms/behaviors at runtime, e.g., different payment providers), **Circuit Breaker** and **Saga** (resilience/distributed transaction patterns already covered), and **Observer/Pub-Sub** (event-driven communication).

---

# Section 3: Security Implementation Questions

**Q: How do you secure a Node.js application?**
Layered approach: validate all input, use parameterized queries/ORM to prevent injection, hash passwords with bcrypt, use HTTPS, add Helmet for secure headers, configure CORS restrictively, implement rate limiting, keep dependencies updated (audit for known vulnerabilities), store secrets in environment variables/secret managers (never in code), and use JWT/session-based auth with proper expiry.

**Q: How do you implement JWT authentication?**
Issue a signed JWT on successful login containing the user's ID/role as claims, with a short expiry; verify the token's signature and expiry on every protected request via middleware, extracting the user info from its payload. See the Authentication notes for full code examples.

**Q: Access Token vs Refresh Token?**
Access tokens are short-lived (minutes) and sent with every API request; refresh tokens are long-lived (days/weeks), stored securely, and used only to obtain a new access token without re-login — this balances security (short-lived tokens limit exposure if stolen) with usability (users don't need to log in constantly). Full detail in the Authentication notes.

**Q: JWT vs Session?**
Sessions are stateful — the server stores session data and the client holds only a reference (session ID), requiring a shared session store across multiple servers. JWTs are stateless — all necessary data is embedded in the signed token, and any server can verify it independently without a shared store, making JWTs better suited to distributed/microservices architectures, while sessions are simpler to revoke instantly.

**Q: How do you prevent SQL Injection?**
Always use **parameterized queries** or an ORM (never string-concatenate user input directly into SQL) — the database treats parameters strictly as data, never as executable SQL syntax, regardless of what the user submits.
```js
// Vulnerable
db.query(`SELECT * FROM users WHERE email = '${email}'`);
// Safe — parameterized
db.query("SELECT * FROM users WHERE email = ?", [email]);
```

**Q: How do you prevent NoSQL Injection?**
Similarly, never pass raw, unvalidated user input directly into a MongoDB query object — a malicious payload like `{ "$gt": "" }` submitted as a password field could bypass authentication logic. Validate/sanitize input types explicitly (ensure expected fields are actually strings, not objects) and use libraries like `express-mongo-sanitize` to strip out MongoDB operator characters from user input.

**Q: How do you prevent XSS (Cross-Site Scripting)?**
Never render untrusted user input directly as raw HTML — escape/encode output by default (most modern frontend frameworks like React do this automatically unless you explicitly use `dangerouslySetInnerHTML`), sanitize any user-generated HTML content with a library like DOMPurify if rich text must be rendered, and set a `Content-Security-Policy` header restricting which scripts can execute.

**Q: How do you prevent CSRF (Cross-Site Request Forgery)?**
Use the `SameSite` cookie attribute (`Strict` or `Lax`) to prevent cookies from being sent on cross-site requests, and/or implement CSRF tokens (a unique, unpredictable token embedded in forms/requests that the server verifies matches what it issued) for state-changing requests. See the Authentication notes for a detailed example of the attack and defense.

**Q: How do you secure REST APIs?**
HTTPS only, authentication + authorization on every protected endpoint, strict input validation, rate limiting, CORS restricted to known origins, proper HTTP status codes and generic error messages (avoid leaking stack traces/internal details), and versioning to allow safe evolution.

**Q: How do you implement RBAC (Role-Based Access Control)?**
Assign each user one or more roles (admin, editor, viewer); define which actions/resources each role can access; check the authenticated user's role against the required permission in middleware before allowing the request to proceed.
```js
function authorize(...allowedRoles) {
  return (req, res, next) => {
    if (!allowedRoles.includes(req.user.role)) {
      return res.status(403).json({ message: "Forbidden" });
    }
    next();
  };
}
```

**Q: How do you store passwords securely?**
Never store plain-text passwords. Hash them with a slow, salted hashing algorithm designed specifically for passwords — **bcrypt** or **Argon2** — never a fast general-purpose hash like MD5/SHA1, which are crackable at scale via brute force/rainbow tables.

**Q: Why use bcrypt?**
Bcrypt is intentionally **slow** and includes a built-in random **salt** for each hash — this makes brute-force and rainbow-table attacks impractical, since an attacker can't precompute hashes for common passwords (salting) and each guess takes meaningful computational time (deliberate slowness), unlike fast hashes like MD5 which can be brute-forced at billions of attempts per second on modern hardware.

**Q: How do you encrypt sensitive data?**
For data that must be recoverable (not just verified, like passwords), use strong symmetric encryption (AES-256) with keys managed by a dedicated key management service (AWS KMS, Azure Key Vault) — never store encryption keys alongside the encrypted data itself. Enable encryption at rest (database-level, e.g., MSSQL's TDE/Always Encrypted) and in transit (TLS).

**Q: How do you manage secrets?**
Never hardcode secrets (API keys, DB passwords, JWT secrets) in code or commit them to version control. Use environment variables for local development and a dedicated secret management service (Azure Key Vault, AWS Secrets Manager, HashiCorp Vault) in production, with access tightly scoped and secrets rotated periodically.

**Q: How do you secure environment variables?**
Keep `.env` files out of version control (`.gitignore`), never log environment variables, use different values per environment (dev/staging/prod), and in production prefer a managed secret store over plain `.env` files where possible, since secret stores offer access auditing, rotation, and finer-grained access control.

**Q: What is Helmet middleware?**
An Express middleware that automatically sets several security-related HTTP response headers (preventing MIME-sniffing, clickjacking, forcing HTTPS, restricting content sources via CSP) — essentially free, one-line hardening against a class of common web vulnerabilities. See the Express.js notes for full detail.

**Q: How do you configure CORS?**
Explicitly whitelist only the specific trusted origins that should be allowed to call your API from a browser, rather than using a wildcard (`*`), especially when `credentials: true` is needed (cookies) — a wildcard combined with credentials is both insecure and rejected by browsers. See the Express.js notes for the `cors` middleware example.

**Q: How do you implement rate limiting?**
Use middleware (e.g., `express-rate-limit`) to cap the number of requests a client (by IP or user ID) can make within a time window, returning `429 Too Many Requests` when exceeded — apply stricter limits on sensitive endpoints like login/signup to reduce brute-force risk.

**Q: How do you prevent brute-force attacks?**
Combine rate limiting on authentication endpoints, account lockout or exponential delay after repeated failed login attempts, CAPTCHA after a threshold of failures, and monitoring/alerting on unusual login failure patterns from a single IP or against a single account.

**Q: How do you validate user input?**
Use a schema-based validation library (Joi, express-validator, Zod) to strictly define expected shape, types, and constraints for every incoming request, rejecting anything that doesn't conform **before** it reaches business logic — never trust client-side validation alone, since it can be bypassed.

**Q: How do you sanitize inputs?**
Beyond validating shape/type, sanitize input by stripping/escaping potentially dangerous characters (HTML tags for XSS prevention, MongoDB operators for NoSQL injection prevention) using dedicated sanitization libraries rather than writing custom regex-based cleaning, which is easy to get wrong.

**Q: How do you implement API throttling?**
Similar to rate limiting but often tiered — e.g., different limits per API key/subscription plan, with response headers (`RateLimit-Remaining`, `RateLimit-Reset`) informing clients of their current usage, allowing well-behaved clients to self-regulate before hitting hard limits.

**Q: How do you secure file uploads?**
Validate file type and size before accepting (both client-side for UX and server-side, since client checks can be bypassed), scan uploaded files for malware, store files outside the web-servable directory or in dedicated object storage (never execute uploaded files), and generate randomized filenames to prevent path traversal or overwrite attacks.

**Q: How do you prevent unauthorized access?**
Enforce authentication (`[Authorize]` by default per the org's API standards) and authorization checks on every endpoint (never rely solely on "security by obscurity," like an unguessable URL), validate that a user can only access/modify their **own** resources (object-level authorization — a common vulnerability class is when authenticated users can access other users' data simply by changing an ID in the URL).

**Q: How do you implement OAuth?**
Use the Authorization Code flow: redirect the user to the provider (Google/GitHub) with your client ID and requested scopes; the provider redirects back with an authorization code; your backend exchanges that code (plus your client secret) for an access token directly with the provider's server; use that token to fetch the user's profile and establish your own session/JWT for them. Full detail and code in the Authentication notes.

**Q: How do you secure cookies?**
Set `httpOnly` (prevents JavaScript access, mitigating XSS-based token theft), `secure` (HTTPS-only transmission), and an appropriate `sameSite` value (mitigating CSRF), along with a reasonable expiry.

**Q: HttpOnly vs Secure cookies?**
`HttpOnly` prevents the cookie from being read by client-side JavaScript (defends against XSS stealing the cookie's value). `Secure` ensures the cookie is only ever sent over HTTPS connections, never plain HTTP (defends against network eavesdropping). They address different threats and are typically used together.

**Q: SameSite cookie options?**
`Strict` — never sent on cross-site requests (most secure, can break some legitimate cross-site navigation flows); `Lax` (default in modern browsers) — sent on top-level navigations but not on cross-site subrequests/AJAX (good balance); `None` — sent on all cross-site requests, must be paired with `Secure`, used for legitimate cross-site integration needs. Full detail with a CSRF example in the Authentication notes.

**Q: How do you log security events?**
Log authentication attempts (success/failure), authorization failures (403s), password changes, and administrative actions to a centralized, immutable log — include enough context to investigate (timestamp, user ID, IP address, action) but **never** log sensitive data like passwords, tokens, or full card numbers.

**Q: What security headers do you use?**
Commonly set via Helmet: `Strict-Transport-Security` (forces HTTPS), `X-Content-Type-Options: nosniff` (prevents MIME sniffing), `X-Frame-Options: DENY` (prevents clickjacking), `Content-Security-Policy` (restricts allowed script/style/resource sources, a strong defense against XSS).

---

# Section 4: Performance Improvement Questions

**Q: API response time increased to 5 seconds. What would you do?**
First, check monitoring/APM dashboards to see if it's a database issue (slow query), a downstream/third-party dependency slowdown, resource exhaustion (CPU/memory) on the server, or a recent deployment correlating with the regression. Isolate with `EXPLAIN` on suspect queries, check external API latency, and review recent changes before guessing — measure before optimizing.

**Q: Frontend loads slowly. How do you optimize it?**
Reduce JavaScript bundle size (code splitting, lazy loading, tree shaking), optimize images (compression, modern formats, lazy loading), leverage a CDN for static assets, minimize render-blocking resources, and use performance profiling tools (Lighthouse, Chrome DevTools) to identify the actual bottleneck (Largest Contentful Paint, Time to Interactive) rather than guessing.

**Q: Node.js server CPU reaches 100%.**
Likely causes: an inefficient/blocking synchronous operation on the event loop (e.g., heavy computation, `JSON.parse` on huge payloads, poorly optimized regex causing catastrophic backtracking), an infinite loop, or simply insufficient capacity for current load. Use profiling tools (Node's built-in `--prof`, or `clinic.js`) to identify the hot function, and consider offloading CPU-heavy work to worker threads or a separate service.

**Q: Memory usage keeps increasing.**
This points to a **memory leak** — commonly caused by: event listeners that are added but never removed, growing caches/arrays with no eviction policy, closures unintentionally retaining references to large objects, or unclosed database/network connections. Use heap snapshots (Chrome DevTools/Node's `--inspect`) taken at intervals to compare and identify what's growing unboundedly.

**Q: Database queries are slow.**
Check the query's execution plan (`EXPLAIN`) for full table scans, missing indexes on `WHERE`/`JOIN`/`ORDER BY` columns, or inefficient query patterns (`SELECT *`, unnecessary subqueries). Add appropriate indexes, rewrite the query, or consider caching the result if it's read frequently and doesn't change often.

**Q: Application slows with 10,000 users.**
Identify the actual bottleneck first (database connections exhausted? CPU-bound on the app server? A specific slow endpoint being hit disproportionately?) via monitoring — then address it specifically: add caching, scale horizontally, add database read replicas, or optimize the specific slow code path. Don't scale blindly without knowing the bottleneck.

**Q: How would you implement caching?**
Cache-aside pattern with Redis for frequently-read, infrequently-changing data — check cache, fall back to DB on miss, populate cache, use TTL and/or explicit invalidation on writes. Full detail and code in the earlier Caching notes.

**Q: Where would you use Redis?**
Caching expensive query results, session storage, rate limiting counters, real-time leaderboards (sorted sets), Pub/Sub for real-time features (chat, notifications across service instances), and as a fast job queue backend (BullMQ). Full detail in the Redis Basics notes.

**Q: How do you reduce API latency?**
Add caching for expensive/frequent operations, optimize database queries and add indexes, use connection pooling, minimize the number of sequential external calls (parallelize independent calls with `Promise.all`), and consider a CDN or edge caching for geographically distributed users.

**Q: How do you optimize SQL queries?**
Add indexes on columns used in `WHERE`/`JOIN`/`ORDER BY`, avoid `SELECT *` (fetch only needed columns), avoid wrapping indexed columns in functions in the `WHERE` clause (breaks index usage), rewrite correlated subqueries as joins where possible, and review the execution plan to confirm the optimizer is actually using your indexes.

**Q: How do you optimize MongoDB queries?**
Add indexes matching your query patterns (including compound indexes respecting the fields you filter/sort by), use `$project` early in aggregation pipelines to reduce the data volume flowing through later stages, avoid unnecessary `$lookup`s by considering embedding for frequently-joined data, and use `.explain()` to inspect whether indexes are actually being used.

**Q: How do you optimize React rendering?**
Use `React.memo`, `useMemo`, and `useCallback` deliberately to avoid unnecessary re-renders and recalculations, virtualize long lists (`react-window`), split code by route (lazy loading), and avoid creating new object/array/function references inline in JSX where they'd defeat memoization. Full detail in the Performance and React.js notes.

**Q: How do you reduce bundle size?**
Tree shaking (ES modules), code splitting (route-based lazy loading), analyzing the bundle to find and replace/remove oversized dependencies, minification, and avoiding heavy libraries when a lighter alternative exists (e.g., `date-fns` over `moment.js`). Full detail in the Performance notes.

**Q: How do you optimize images?**
Use modern compressed formats (WebP/AVIF), serve responsively sized images (`srcset`), lazy-load off-screen images, and set explicit dimensions to prevent layout shift. Full detail in the Performance notes.

**Q: What is lazy loading?**
Deferring the loading of a resource (code, image, component) until it's actually needed, rather than upfront — reducing initial load time. Full definition and examples in the Performance and React.js notes.

**Q: What is code splitting?**
Breaking a large JavaScript bundle into smaller chunks loaded on demand (commonly per route), so users don't download code for parts of the app they haven't visited yet. Full definition and examples in the Performance notes.

**Q: What is a CDN?**
**Simple definition:** A Content Delivery Network is a geographically distributed network of servers that cache and serve static content (images, JS, CSS, videos) from a location physically close to the end user — dramatically reducing latency compared to serving everything from one origin server, and offloading traffic away from your main servers.

**Q: How do you optimize file downloads?**
Serve large files via a CDN or direct object storage links (not through the application server), use streaming responses instead of loading entire files into memory, support range requests for resumable downloads, and compress content where appropriate (gzip/Brotli).

**Q: How do you optimize pagination?**
For large or frequently-changing datasets, prefer **cursor-based pagination** over offset-based — offset pagination requires the database to skip over rows (slow at high offsets) and can produce inconsistent results if data changes between page loads, while cursor-based pagination uses a stable pointer (e.g., last seen ID) for consistent, efficient forward paging.

**Q: How do you reduce memory leaks?**
Always remove event listeners/subscriptions when no longer needed (component unmount, connection close), avoid unbounded caches (use TTLs or size limits), close database/file connections properly, and periodically profile memory usage in long-running processes to catch leaks before they cause outages.

**Q: How do you improve server throughput?**
Use asynchronous, non-blocking I/O properly (avoid blocking the event loop in Node.js), scale horizontally with a load balancer, use connection pooling for databases, offload heavy work to background workers/queues, and cache frequently-requested data to reduce redundant processing.

**Q: How do you optimize WebSocket connections?**
Use a Pub/Sub layer (Redis) to broadcast messages efficiently across multiple WebSocket server instances rather than each instance independently querying data, implement heartbeats to detect and clean up dead connections promptly, and batch/throttle high-frequency updates rather than sending every single change individually.

**Q: How do you optimize SSR (Server-Side Rendering) performance?**
Cache rendered HTML output where content doesn't change per-request (or use ISR — Incremental Static Regeneration, in Next.js), avoid redundant data fetching on the server for data that could be fetched client-side after initial render, and monitor server rendering time separately from client-side metrics since SSR adds server CPU load per request.

**Q: How do you optimize API Gateway performance?**
Cache responses for cacheable, frequently-requested endpoints at the gateway level, keep authentication/authorization checks lightweight (e.g., JWT signature verification rather than a database call per request), and ensure the gateway itself is horizontally scalable and doesn't become a single bottleneck for all traffic.

**Q: What metrics would you monitor?**
The "RED" method: **R**ate (requests per second), **E**rrors (error rate/count), **D**uration (latency, ideally p50/p95/p99 percentiles, not just average) — plus infrastructure metrics (CPU, memory, disk, network) and business-specific metrics (orders per minute, active users).

---

*Continued in Part 2: Sections 5-8 (Production Issue Investigation, Resolving Production Issues, Deployment, Database Performance)*
