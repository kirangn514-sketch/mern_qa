# System Design & Production Engineering — Interview Notes (Part 2)

*Continued from Part 1 (Design Architecture, Architecture Scenarios, Security, Performance). This part covers Production Issue Investigation, Resolving Production Issues, Deployment, and Database Performance.*

---

# Section 5: Production Issue Investigation Questions

**General principle for ALL of these:** The first step is always to **stay calm, confirm the scope of impact, and gather data before making changes** — check monitoring dashboards, recent deployments, and logs before jumping to conclusions or making risky changes to a live production system.

**Q: Production application is down. What is your first step?**
Confirm the outage's scope (all users, or a subset? one region? one service?), check monitoring/alerting dashboards and recent deployment history, and communicate status to stakeholders early — then start investigating logs and metrics for the affected component, prioritizing restoring service (e.g., rollback) over root-causing immediately if a recent deploy is the likely culprit.

**Q: APIs return 500 errors only in production.**
Check production logs for the actual stack trace/exception first. Common causes: environment-specific configuration differences (missing/incorrect env variables that exist correctly in dev), a dependency or database that's unreachable/misconfigured in production, or a code path that only triggers with production-scale/real data.

**Q: Login works locally but fails in production.**
Very commonly caused by environment-specific configuration: incorrect `JWT_SECRET`/OAuth credentials for the production environment, CORS misconfiguration blocking the production frontend's origin, cookie settings (`secure: true` requiring HTTPS, which may not be set up correctly), or a database connection issue specific to the production environment.

**Q: High CPU usage on production server.**
Identify which process/request pattern is consuming CPU (via APM tools or `top`/process monitoring), check for inefficient code paths (unoptimized loops, regex catastrophic backtracking, synchronous blocking operations), a traffic spike beyond normal capacity, or a recent deploy introducing a performance regression.

**Q: High memory usage on production server.**
Likely a memory leak (see Performance section) or simply undersized infrastructure for current load — take heap snapshots to identify what's accumulating, check for recently deployed code that introduced unbounded caching/retained references, and check whether memory usage correlates with request volume (organic scaling need) or grows independently over time (leak).

**Q: Database connections are exhausted.**
Usually caused by: connection leaks (connections opened but never released back to the pool, often due to missing `finally`/cleanup logic), an undersized connection pool for current traffic, or long-running queries holding connections open too long. Check the connection pool configuration and audit code paths for unclosed connections.

**Q: Redis is unavailable.**
Application should be designed to **degrade gracefully** when the cache is down — fall back to querying the database directly (slower, but functional) rather than crashing entirely. Investigate Redis's own health (memory limits reached causing evictions/crashes, network connectivity, or the Redis instance itself being down) and check for a proper reconnection/retry strategy in the client.

**Q: Queue processing has stopped.**
Check whether worker processes are actually running and healthy, whether the queue itself (RabbitMQ/Kafka/Redis) is reachable and healthy, and whether a specific "poison message" is repeatedly failing and blocking the queue (in which case a dead letter queue should isolate it) rather than stalling all subsequent processing.

**Q: Emails are not being sent.**
Check the email provider's (SendGrid/SES) status/dashboard for delivery failures or rate limit issues, verify API keys/credentials haven't expired, check the application's own queue/worker responsible for sending emails is running, and check spam/bounce reports in case emails are being sent but rejected/filtered.

**Q: Payment service is failing.**
High-priority investigation: check the payment provider's (Stripe/Razorpay) status page for an outage on their end, verify API credentials/webhooks are correctly configured, check for recent code changes to the payment flow, and ensure failures are being handled gracefully (not silently losing failed payment attempts) with proper logging for reconciliation.

**Q: Users report duplicate orders.**
Almost always an **idempotency** failure — check whether the checkout/order-creation endpoint properly deduplicates retried requests (e.g., a user double-clicking "submit," or a network retry after a timeout) using an idempotency key, and check for race conditions if multiple requests can be processed concurrently for the same cart/session.

**Q: Notifications are delayed.**
Check queue depth/backlog for the notification service (are workers keeping up with the volume?), check for a bottleneck in a specific delivery channel (e.g., the SMS provider being slow), and verify worker scaling is adequate for current load — may require scaling up consumer instances.

**Q: APIs randomly timeout.**
Check for intermittent network issues, a downstream dependency (database, third-party API) occasionally being slow, resource contention (CPU/memory spikes causing the event loop to be blocked intermittently), or connection pool exhaustion happening under specific load patterns. Distributed tracing helps pinpoint exactly where the time is being spent in these intermittent cases.

**Q: SSL certificate expired.**
Immediate fix: renew and reinstall the certificate as quickly as possible (via your certificate provider or Let's Encrypt/Certbot). Long-term fix: set up automated certificate renewal (e.g., Certbot's auto-renewal) and monitoring/alerting well before expiry (e.g., 30 days out) so this doesn't happen again.

**Q: Disk space is full.**
Identify what's consuming space (often log files that were never rotated, or accumulated temporary files) and clean up immediately to restore service, then implement log rotation policies and disk usage monitoring/alerting to catch this proactively before it recurs.

**Q: One API is significantly slower than others.**
Investigate that specific endpoint's query patterns (missing index, N+1 query problem), whether it does unnecessary heavy computation or serial (rather than parallel) calls to multiple dependencies, or whether it's simply handling a disproportionately larger payload/dataset than other endpoints.

**Q: Server restarts automatically.**
Check for an unhandled exception/crash in the application causing the process manager (PM2, Kubernetes) to restart it, out-of-memory kills (OOMKilled in Kubernetes — check if the container's memory limit is being hit), or health check failures triggering an orchestrator-initiated restart.

**Q: Application crashes every few hours.**
Points to a slow-building issue — most commonly a memory leak that eventually exhausts available memory, or an accumulating resource (open file handles, database connections) hitting a limit. Monitor the specific resource's trend over time to identify what's growing before the crash occurs.

**Q: Some users face issues while others don't.**
Check for patterns: are affected users on a specific server instance (if load-balanced, this can point to one bad instance), a specific geographic region (CDN/regional infrastructure issue), a specific feature flag/A-B test group, or a specific data condition (e.g., users with a particular account state triggering an edge-case bug).

**Q: Third-party API latency increased.**
Confirm via the provider's own status page/dashboard, check if this correlates with your own increased request volume (rate limiting kicking in on their end), and ensure your system has appropriate timeouts and a circuit breaker so their slowdown doesn't cascade into your own system becoming unresponsive.

**Q: Cron jobs are not executing.**
Check the scheduler's own health/logs (is the cron service itself running?), verify the server's system time/timezone configuration is correct, check for silent failures in the job itself (exceptions being swallowed without logging), and in distributed setups, check whether a distributed lock mechanism is incorrectly preventing execution.

**Q: Background workers stopped processing.**
Check worker process health (crashed and not restarted?), check the queue/broker's own health and connectivity, and look for a "poison message" that's repeatedly crashing the worker on the same message without progressing (should be moved to a dead letter queue instead).

**Q: Logs are not being generated.**
Check the logging library's configuration (log level set too restrictively, e.g., only "error" when you need "info"), verify the log destination (file path permissions, or a centralized logging service's connectivity/API key), and confirm the application isn't crashing before reaching the logging code.

**Q: Environment variables are incorrect.**
Verify the actual runtime environment's variables (not just the local `.env` file) — a common cause is a deployment pipeline not properly injecting production secrets, or a typo/stale value in the secret management system. Add startup validation that fails fast with a clear error if required environment variables are missing, rather than failing mysteriously later.

**Q: Recent deployment broke production.**
Roll back to the previous known-good version immediately to restore service, then investigate the diff between versions to identify the specific breaking change in a non-production environment — restoring service takes priority over root-causing while users are impacted.

**Q: Load balancer sends traffic to unhealthy servers.**
Check the health check endpoint's logic and configuration (is it actually reflecting true service health, including dependency checks? Is the health check interval/threshold too lenient?), and verify the load balancer's configuration is correctly removing unhealthy instances from rotation based on health check results.

**Q: Database replication lag increased.**
Check the replica's resource usage (CPU/disk I/O may be a bottleneck), check for a large batch write/migration on the primary that's overwhelming replication, and check network connectivity/bandwidth between primary and replica — if the application reads from replicas, be aware this lag can cause users to see stale data shortly after a write.

**Q: High network latency between services.**
Check whether services are deployed in the same region/availability zone (cross-region calls add significant latency), review whether excessive serial (rather than parallel/batched) calls between services are compounding latency, and check for network-level issues (DNS resolution delays, misconfigured service mesh/proxy overhead).

**Q: CDN serves outdated content.**
Check the CDN's cache TTL/invalidation configuration — likely the cached content's TTL hasn't expired yet, or a cache invalidation/purge wasn't triggered after deploying updated content. Trigger a manual cache purge for the affected content and review the caching headers/strategy going forward.

**Q: Health check endpoint fails.**
Investigate what the health check actually verifies — if it checks a dependency (database, cache) and that dependency is genuinely unhealthy, the failure may be accurate and pointing to a real underlying issue rather than a bug in the health check itself; verify which specific check within the health endpoint is failing.

---

# Section 6: Resolving Production Issues

**Q: Walk me through your production debugging process.**
1. Confirm scope/impact and communicate status. 2. Check monitoring dashboards and recent deployments/changes for correlation. 3. Check centralized logs (using correlation IDs if distributed) for the actual error. 4. Form a hypothesis and verify it with data (don't guess-and-check on production). 5. Apply the fix (or rollback if faster/safer). 6. Verify the fix resolved the issue via monitoring. 7. Conduct a postmortem/RCA afterward to prevent recurrence.

**Q: How do you identify the root cause?**
Use the "5 Whys" technique — repeatedly ask why the symptom occurred until you reach the actual underlying cause rather than stopping at a superficial explanation. Combine this with data: logs, metrics, and traces from the actual incident, not assumptions — and verify your hypothesis actually explains all observed symptoms before considering it confirmed.

**Q: How do you perform a rollback?**
Redeploy the last known-good version/build (most CI/CD pipelines and container registries retain previous versions/images for exactly this purpose) — this should be a fast, well-tested, and ideally automated process, since restoring service quickly during an incident matters more than a "clean" fix in the moment.

**Q: How do you deploy a hotfix?**
Branch from the current production version (not from an in-progress development branch that may contain unrelated, unreleased changes), apply the minimal fix needed, test it as thoroughly as time allows, and deploy through an expedited (but still reviewed, where possible) path — avoiding skipping critical safety checks like automated tests even under time pressure.

**Q: How do you avoid downtime during fixes?**
Use zero-downtime deployment strategies (Rolling, Blue-Green, or Canary — detailed in Section 7), ensure database migrations are backward-compatible with both old and new code versions during the transition, and use feature flags to decouple code deployment from feature activation.

**Q: How do you communicate incidents?**
Notify stakeholders early with what's known (even if incomplete) rather than waiting for full certainty, provide regular updates at a predictable cadence, clearly state current impact and expected next update time, and follow up with a summary once resolved — transparency during an incident builds more trust than silence.

**Q: How do you verify the fix?**
Monitor the specific metrics/error rates that indicated the original problem, confirm they've returned to normal baseline levels, and if possible, reproduce the original failing scenario to explicitly confirm it now behaves correctly — don't declare "fixed" based on the absence of new reports alone.

**Q: How do you monitor after deployment?**
Watch error rates, latency, and resource usage metrics closely for a period after any deployment (sometimes called a "bake time"), ideally with automated alerting configured to catch regressions quickly, and be prepared to roll back immediately if metrics degrade.

**Q: How do you perform Root Cause Analysis (RCA)?**
After the immediate incident is resolved, conduct a structured review: what happened, what was the actual root cause (not just the immediate trigger), what was the impact/timeline, what went well and what didn't in the response, and what specific action items will prevent recurrence — documented and shared, ideally in a blameless format focused on systemic fixes rather than individual blame.

**Q: How do you prevent recurrence?**
Turn RCA findings into concrete action items: add monitoring/alerting for the specific failure mode if it wasn't caught proactively, add automated tests covering the scenario, improve documentation/runbooks, and address any process gaps (e.g., missing staging environment testing) that let the issue reach production.

**Q: How do you restore corrupted data?**
Restore from the most recent clean backup, and if possible, reconcile the gap between the backup and the corruption point using transaction logs or event history (if using event sourcing) to replay legitimate changes that occurred after the backup but before the corruption.

**Q: How do you recover from backup?**
Follow a tested, documented recovery procedure (recovery procedures that have never been tested often fail exactly when needed) — restore the backup to a separate environment first to verify integrity, then cut over, minimizing additional downtime and data loss.

**Q: How do you fix duplicate transactions?**
Identify and deduplicate the specific duplicate records (often distinguishable via an idempotency key or timestamp proximity), reverse any resulting incorrect side effects (e.g., refund a duplicate charge), and fix the underlying cause (missing idempotency handling) to prevent recurrence.

**Q: How do you fix data inconsistencies?**
Identify the scope of affected records (query for the specific inconsistent pattern), write a careful, tested correction script (ideally run against a copy first, with a dry-run/preview mode), and address the root cause — often a missing transaction boundary, a race condition, or a failed step in a distributed Saga that wasn't properly compensated.

**Q: How do you handle partial failures?**
Design operations to be resumable/idempotent where possible, use the Saga pattern's compensating transactions for distributed operations, and ensure partial failures are logged with enough detail to manually or automatically complete/reverse the incomplete operation rather than leaving it in an ambiguous state.

**Q: How do you resolve deadlocks?**
Identify the conflicting queries/transactions via the database's deadlock logs, ensure transactions acquire locks in a **consistent order** across the application (a common deadlock cause is two transactions locking the same two resources in opposite order), keep transactions short, and consider adding appropriate indexes to reduce lock scope/duration.

**Q: How do you fix memory leaks?**
Take heap snapshots over time to identify what's accumulating, trace back to the code responsible (unremoved event listeners, unbounded caches, retained closures), fix the specific leak, and add ongoing memory monitoring to catch future regressions early.

**Q: How do you fix connection pool exhaustion?**
Audit code for connections that aren't properly released (missing `finally`/`.close()` calls), increase pool size if genuinely undersized for load, add connection timeout/queueing configuration, and add monitoring on pool utilization to catch this trending toward exhaustion before it causes an outage.

**Q: How do you fix file upload failures?**
Check for size limit misconfigurations (both client and server/proxy level, e.g., Nginx's `client_max_body_size`), verify storage service connectivity/permissions, check for network timeout issues on large files (consider chunked/resumable uploads), and ensure proper error handling surfaces the actual failure reason rather than a generic error.

**Q: How do you fix cache inconsistencies?**
Identify whether the issue is stale data (missing invalidation on write) or a race condition (a stale value written to cache after a fresher one, due to concurrent requests) — fix by ensuring every write path properly invalidates/updates the relevant cache keys, and consider shorter TTLs as a safety net for data where staleness is costly.

**Q: How do you resolve DNS issues?**
Verify DNS records are correctly configured and propagated (DNS changes can take time to propagate globally), check TTL settings, and use tools like `dig`/`nslookup` to confirm what's actually being resolved from different locations, comparing against the expected configuration.

**Q: How do you fix SSL issues?**
Verify the certificate is valid, not expired, correctly matches the domain, and is properly chained (intermediate certificates included) — use tools like SSL Labs' checker to diagnose configuration issues, and ensure automated renewal is in place going forward.

**Q: How do you resolve container crashes?**
Check the container's logs for the actual crash reason (`kubectl logs`, `docker logs`), verify resource limits aren't too restrictive (OOMKilled scenarios), check the container's health check/readiness probe configuration, and verify the application handles graceful shutdown signals (SIGTERM) properly.

**Q: How do you handle emergency deployments?**
Follow an expedited but still safe path — minimal, targeted changes only, ideally still peer-reviewed even under time pressure, deployed via the standard automated pipeline (not manual/ad-hoc steps that skip safety checks) wherever possible, with close post-deployment monitoring.

**Q: What postmortem process do you follow?**
A blameless, written summary covering: timeline of events, impact (users/revenue affected, duration), root cause, what went well/poorly in detection and response, and concrete, assigned action items with owners and deadlines — shared with the broader team to spread learnings, not just filed away.

---

# Section 7: Deployment Related Questions

**Q: Explain your deployment pipeline.**
Typically: code pushed → automated tests run (CI) → build artifact/Docker image created → deployed to a staging environment for further validation → (after approval/automated checks pass) deployed to production using a safe strategy (rolling/blue-green/canary) → post-deployment monitoring confirms health.

**Q: How do you deploy a Node.js application?**
Package the app (often as a Docker container), run it behind a process manager (PM2) or container orchestrator (Kubernetes) for automatic restarts, put it behind a reverse proxy (Nginx) handling SSL termination and load balancing, and manage configuration via environment variables injected per environment.

**Q: How do you deploy a Next.js application?**
Can be deployed to Vercel (zero-config, built for Next.js, supports SSR/ISR natively) or self-hosted via `next build && next start` behind a Node.js process, or containerized with Docker and deployed to any container platform — the choice depends on whether you need Vercel's specific edge/ISR features or prefer full infrastructure control.

**Q: How do you deploy using Docker?**
Build a Docker image from a Dockerfile, push it to a container registry (Docker Hub, ECR, GCR), and run it on the target infrastructure (a single server, Docker Compose for multi-container local/simple setups, or Kubernetes for orchestrated production deployments).

**Q: What is Docker Compose?**
**Simple definition:** A tool for defining and running **multi-container** Docker applications using a single YAML configuration file — instead of manually starting each container (app, database, Redis) with separate `docker run` commands, `docker-compose up` starts them all together with their networking and dependencies configured declaratively. Primarily used for local development and simple deployments, not typically for large-scale production orchestration (that's Kubernetes's job).

**Q: What is Kubernetes?**
A container orchestration platform automating deployment, scaling, healing, and networking of containerized applications across a cluster of machines. Full detail in the Microservices notes.

**Q: Explain Blue-Green Deployment.**
**Simple definition:** Maintain two identical production environments — "Blue" (currently live) and "Green" (the new version) — deploy and fully test the new version on Green while Blue continues serving all live traffic, then switch the router/load balancer to send traffic to Green instantly once verified. If something's wrong, you can instantly switch back to Blue.
```
Before: Router → Blue (v1, live)      Green (v2, idle, being tested)
After:  Router → Green (v2, live)     Blue (v1, idle, kept as instant rollback)
```
**Pros:** Instant rollback (just switch the router back), zero downtime, full testing of the new version under production-like conditions before it receives real traffic. **Cons:** Requires double the infrastructure (running two full environments simultaneously, even briefly).

**Q: Explain Rolling Deployment.**
**Simple definition:** Gradually replace old instances with new ones, one (or a small batch) at a time, rather than switching everything at once — at any given moment during the rollout, some instances run the old version and some run the new version, both serving live traffic simultaneously.
```
Instances: [v1] [v1] [v1] [v1]
Step 1:    [v2] [v1] [v1] [v1]
Step 2:    [v2] [v2] [v1] [v1]
Step 3:    [v2] [v2] [v2] [v1]
Step 4:    [v2] [v2] [v2] [v2]
```
**Pros:** No extra infrastructure needed (unlike Blue-Green). **Cons:** Rollback is slower (must roll back the same way, instance by instance), and both versions run simultaneously for a period, which requires the two versions to be compatible with each other and with the shared database schema.

**Q: Explain Canary Deployment.**
**Simple definition:** Roll out the new version to a **small subset** of real users/traffic first (e.g., 5%), monitor closely for errors/performance issues, and only gradually increase the rollout percentage if the canary group shows no problems — named after the historical practice of using canaries to detect danger in coal mines before it affected miners.
```
95% of traffic → v1 (stable)
 5% of traffic → v2 (canary — being carefully monitored)

If healthy → gradually shift more traffic to v2 → eventually 100% v2
If unhealthy → shift traffic back to v1, investigate
```
**Pros:** Limits the "blast radius" of a bad deployment to a small fraction of users, allows real-world validation with real traffic before full exposure. **Cons:** More complex to set up (requires traffic-splitting infrastructure) and takes longer to fully roll out than an all-at-once deployment.

**Q: How do you achieve Zero Downtime Deployment?**
Use Rolling, Blue-Green, or Canary strategies (rather than stopping the old version before starting the new one), ensure the load balancer only routes to healthy, ready instances (via readiness probes), and ensure database migrations are backward-compatible with the previous code version during the transition window (since old and new code may run simultaneously briefly).

**Q: What is PM2?**
A production process manager for Node.js applications — keeps the app running (auto-restarting on crashes), supports zero-downtime reloads, cluster mode (running multiple instances to use all CPU cores on a single server), and basic monitoring/log management, without needing a full container orchestrator for simpler deployments.

**Q: How do you configure Nginx?**
Commonly used as a **reverse proxy** in front of a Node.js app — terminating SSL, load balancing across multiple app instances, serving static files directly (bypassing the app server for efficiency), and setting security headers/rate limiting at the proxy layer.
```nginx
server {
  listen 443 ssl;
  server_name example.com;

  location / {
    proxy_pass http://localhost:3000;
    proxy_set_header Host $host;
  }
}
```

**Q: What causes a 502 Bad Gateway error?**
Nginx (or another reverse proxy) received an invalid/no response from the upstream application server — commonly because the app server crashed, isn't running, is still starting up, or timed out. Check the application server's own logs/health directly, not just the proxy's logs, to find the real cause.

**Q: How do you configure SSL?**
Obtain a certificate (via a Certificate Authority or free services like Let's Encrypt), configure your web server/reverse proxy (Nginx) to use it for HTTPS termination, redirect all HTTP traffic to HTTPS, and set up automated renewal (Let's Encrypt certificates expire every 90 days and are typically auto-renewed via Certbot).

**Q: How do you manage environment variables?**
Store different values per environment (dev/staging/prod), never commit secrets to version control, inject them via the deployment platform's secret management (Kubernetes Secrets, cloud provider's parameter store) rather than plain `.env` files in production, and validate required variables are present at application startup.

**Q: How do you automate deployments?**
Set up a CI/CD pipeline (GitHub Actions, Jenkins) that automatically runs tests, builds artifacts, and deploys to each environment on specific triggers (e.g., merge to `main` triggers a staging deploy; a tagged release triggers a production deploy) — removing manual, error-prone deployment steps.

**Q: How do you rollback a deployment?**
Redeploy the previous known-good image/artifact (most CI/CD systems and container registries retain a history for exactly this), or with Kubernetes, use `kubectl rollout undo` to revert a Deployment to its previous revision automatically.

**Q: How do you debug deployment failures?**
Check the CI/CD pipeline logs for the specific failing stage (build, test, deploy), check the target environment's own logs if the deployment succeeded but the app fails to start, and verify environment-specific configuration (env vars, secrets, network access) matches what the application expects.

**Q: What if Docker containers keep restarting?**
Check `docker logs <container>` for the actual crash reason, verify the container isn't hitting a memory/resource limit (OOMKilled), check the health check configuration isn't incorrectly marking a healthy container as failed, and confirm required environment variables/dependencies (database connectivity) are available when the container starts.

**Q: What if Kubernetes Pods are in CrashLoopBackOff?**
This state means the pod's container keeps crashing shortly after starting, and Kubernetes is backing off between restart attempts. Check `kubectl logs <pod> --previous` (to see the crashed container's logs, since the current one may have just restarted), `kubectl describe pod <pod>` for events, and check for missing ConfigMaps/Secrets, failed startup dependencies, or resource limit issues.

**Q: How do you monitor deployments?**
Watch error rates, latency, and resource metrics immediately after a deployment (often with a defined "bake time"), compare against pre-deployment baselines, and configure automated rollback triggers if key metrics degrade beyond a threshold, where the tooling supports it.

**Q: How do you handle failed migrations?**
Ensure migrations are written with both `UP` and `DOWN` scripts so they can be reversed if something goes wrong, test migrations against a copy of production-like data before running them live, and for large tables, consider running migrations in a way that doesn't lock the table for extended periods (e.g., online schema change tools).

**Q: How do you deploy database changes safely?**
Make schema changes **backward-compatible** with the currently-running application version (e.g., add new columns as nullable first, deploy code that can handle both old and new schema, then backfill/enforce constraints in a later step) — this allows safe rolling/blue-green deployments where old and new code might briefly run against the same database.

**Q: How do you manage application versions?**
Use semantic versioning (`MAJOR.MINOR.PATCH`) for releases, tag Docker images and Git commits/releases consistently, and maintain a changelog — for APIs specifically, follow the versioning strategy discussed in the REST API notes (typically URI-based, e.g., `/v1/`, `/v2/`).

**Q: What CI/CD tools have you used?**
Common answers: GitHub Actions (tightly integrated with GitHub repos, YAML-based pipelines), Jenkins (highly customizable, self-hosted, widely used in enterprise/legacy setups), GitLab CI, CircleCI, or Azure DevOps Pipelines — be ready to describe the actual pipeline stages you configured (build, test, deploy) in a specific tool you've used.

---

# Section 8: Database Performance Improvement Questions

**Q: How do you optimize slow SQL queries?**
Run `EXPLAIN` to see the execution plan, add indexes on `WHERE`/`JOIN`/`ORDER BY` columns if a full table scan is occurring, avoid `SELECT *`, rewrite inefficient subqueries as joins, and ensure `WHERE` clauses don't wrap indexed columns in functions (which prevents index usage). Full detail in the Database notes.

**Q: How do you optimize MongoDB queries?**
Add indexes matching actual query patterns (including compound indexes in the correct field order), use `.explain()` to verify index usage, project only needed fields, and place `$match`/`$project` stages as early as possible in aggregation pipelines to reduce the data volume processed by later stages.

**Q: What is indexing?**
A separate, sorted data structure (typically a B-Tree) that lets the database find matching rows without scanning the entire table — turning an O(n) search into something close to O(log n), at the cost of extra storage and slightly slower writes (since indexes must also be updated). Full detail in the Database notes.

**Q: When should you create composite indexes?**
When queries frequently filter/sort by **multiple columns together** — a composite index on `(user_id, status)` efficiently supports queries filtering by `user_id` alone or by `user_id AND status` together (leftmost prefix rule), but a query filtering only by `status` won't benefit from that same index.

**Q: How do you identify missing indexes?**
Use the database's slow query log to find frequently-run, slow queries, then run `EXPLAIN` on them to check for full table scans; many databases and cloud providers also offer built-in tools/advisors that specifically suggest missing indexes based on observed query patterns.

**Q: What is an execution plan?**
The database engine's chosen strategy for executing a specific query — showing whether it's doing an index scan/seek versus a full table scan, the order joins are performed in, and the estimated number of rows processed at each step. Reviewing it (via `EXPLAIN`) is the primary tool for diagnosing why a query is slow.

**Q: How do you optimize JOIN queries?**
Ensure the columns used in the `JOIN ... ON` condition are indexed on both tables, filter data as early as possible (`WHERE` clauses applied before the join where the query planner allows), and avoid unnecessary joins to tables/columns you don't actually need in the result.

**Q: How do you optimize GROUP BY queries?**
Ensure an index exists on the columns being grouped by (this can allow the database to use the index's existing sort order instead of performing a separate, expensive sorting operation), and filter with `WHERE` before grouping wherever possible, since it's cheaper to exclude rows early than to group them and discard groups afterward with `HAVING`.

**Q: How do you optimize pagination?**
Prefer cursor-based pagination (using a stable pointer like the last seen ID) over offset-based pagination for large or frequently-changing datasets — see the next question for the detailed comparison.

**Q: OFFSET vs Cursor Pagination?**
`OFFSET`-based pagination (`LIMIT 20 OFFSET 1000`) requires the database to scan and discard all preceding rows before returning results — this gets progressively slower at higher offsets, and can return inconsistent results if rows are inserted/deleted between page requests. Cursor-based pagination instead uses a `WHERE` condition based on the last-seen record's identifier/timestamp (`WHERE id > last_seen_id LIMIT 20`), which is consistently fast (uses an index seek, not a scan) regardless of how deep into the results you are, and is stable even as underlying data changes.

**Q: How do you optimize aggregate queries?**
Index the columns used in filtering/grouping, consider pre-computing and periodically caching expensive aggregates rather than calculating them live on every request (especially for dashboards/reports), and run heavy aggregate queries against a read replica or dedicated reporting database rather than the primary transactional database.

**Q: How do you reduce database locks?**
Keep transactions as short as possible (don't hold a transaction open while doing slow, unrelated work like calling an external API), access resources in a consistent order across the application to avoid deadlocks, use optimistic locking (version numbers) instead of pessimistic locking where contention is low, and choose an appropriate isolation level (not always the strictest one) for the specific use case.

**Q: How do you prevent deadlocks?**
Ensure all parts of the application acquire locks on shared resources in the **same, consistent order** — the most common deadlock cause is Transaction A locking Resource 1 then waiting for Resource 2, while Transaction B simultaneously locks Resource 2 then waits for Resource 1. Keeping transactions short and using appropriate indexes (which reduces the range of rows locked) also reduces deadlock likelihood.

**Q: How do you optimize inserts?**
Batch multiple inserts into a single statement/transaction rather than one insert per round-trip, temporarily disable non-essential indexes/triggers during very large bulk loads (re-enabling/rebuilding them afterward), and ensure the table isn't over-indexed (every index adds write overhead).

**Q: How do you optimize updates?**
Update only the specific rows/columns that actually need to change (avoid broad `UPDATE` statements without a precise `WHERE` clause), batch large update operations into smaller chunks to avoid long-held locks on huge row ranges, and ensure the `WHERE` clause itself is indexed so the matching rows can be found efficiently.

**Q: How do you optimize deletes?**
For large deletions, delete in smaller batches (e.g., 1000 rows at a time in a loop) rather than one massive `DELETE` statement, which can hold locks for a long time and bloat transaction logs; consider soft deletes (`is_deleted` flag, per this organization's DB standards) if data needs to be retained for audit/recovery purposes.

**Q: How do you optimize transactions?**
Keep them as short as possible (minimize the time locks are held), avoid doing slow external operations (API calls, email sending) inside a database transaction, and only include the operations that genuinely need atomicity together within the transaction boundary.

**Q: What is connection pooling?**
**Simple definition:** Instead of opening and closing a new database connection for every single query (which is relatively slow/expensive), a connection pool maintains a set of **reusable, already-open connections** that application code borrows and returns — dramatically reducing the overhead of connection setup/teardown under load, and preventing the database from being overwhelmed by too many simultaneous raw connections.

**Q: How do you cache database results?**
Use the cache-aside pattern with Redis for frequently-read, expensive-to-compute query results, with a TTL for automatic staleness handling and explicit invalidation when the underlying data is written/updated. Full detail in the Caching section of the System Design notes.

**Q: When would you denormalize data?**
When read performance is critical and the cost of extra `JOIN`s outweighs the benefit of eliminating data duplication — common in read-heavy analytics/reporting systems, or when a specific frequently-accessed value (e.g., a running order count) is worth duplicating/precomputing rather than recalculating via a join or aggregate on every read.

**Q: When would you partition tables?**
When a single table grows too large for efficient querying/maintenance (very large log or time-series tables are a classic case) — partitioning splits the table's data across multiple physical segments (often by date range or a specific key), so queries that target a specific partition (e.g., "this month's data") only need to scan that partition rather than the entire table.

**Q: What is database sharding?**
**Simple definition:** Splitting a single logical database's data **horizontally across multiple separate database instances/servers** (each called a "shard"), typically based on a shard key (e.g., `user_id` range or hash) — done when a single database server can no longer handle the write throughput or data volume alone, even with read replicas and indexing. Unlike partitioning (usually within one database server), sharding distributes data across genuinely separate database servers, adding significant application complexity (queries spanning multiple shards, rebalancing when adding shards) in exchange for near-limitless horizontal write scalability.

**Q: How do you archive old data?**
Move infrequently-accessed historical data out of the primary, actively-queried tables into a separate archive table, database, or cold storage (e.g., S3) — this keeps the primary tables smaller and faster for everyday operations, while archived data remains available (with slightly higher retrieval latency) for the rare occasions it's actually needed (compliance, historical reporting).

**Q: How do you monitor database performance?**
Track query latency (especially p95/p99, not just averages), slow query logs, connection pool utilization, replication lag (if using replicas), CPU/memory/disk I/O on the database server, and lock wait times — using tools like the database's native monitoring, or third-party APM tools that integrate with the database layer.

**Q: How do you troubleshoot high database CPU usage?**
Check the slow query log and currently running queries (`SHOW PROCESSLIST` in MySQL, `pg_stat_activity` in PostgreSQL) to identify expensive queries actively consuming CPU, check for missing indexes causing full table scans on frequently-run queries, check for a recent spike in traffic/query volume, and review recent schema/query changes that might correlate with the CPU increase.

---

## Overall Quick Reference — Cross-Cutting Themes

Many of the 200+ questions above repeatedly draw on the same underlying concepts. If you master these core ideas, you can reason through almost any variation asked in an interview:

| Core Concept | Appears In |
|---|---|
| **Idempotency** | Duplicate requests, duplicate orders, retry mechanisms, payment failures |
| **Indexing & Execution Plans** | Slow queries, MongoDB optimization, JOIN/GROUP BY optimization, missing index detection |
| **Caching (cache-aside, TTL, invalidation)** | Performance improvement, Redis usage, reducing latency, reporting modules |
| **Circuit Breaker + Retry + Timeout** | Third-party API failures, resilience, cascading failure prevention |
| **Saga Pattern / Compensating Transactions** | Distributed transactions, data consistency, rollback strategies |
| **Correlation/Trace ID + Centralized Logging** | Debugging across services, production investigation |
| **Health Checks + Load Balancing** | Deployment safety, unhealthy server routing, Kubernetes scaling |
| **Rollback readiness (Blue-Green/Rolling/Canary)** | Zero-downtime deployment, recovering from bad deployments |
| **Root Cause Analysis discipline** | Every production issue and resolution question |
