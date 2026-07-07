# Microservices Interview Preparation — Complete Notes

---

## 1. What are Microservices?

### Definition
**Simple definition:** Microservices is an architectural style where an application is built as a collection of **small, independent services**, each responsible for a **single business capability**, communicating with each other over a network (usually via APIs or messages) — instead of being built as one large, tightly-coupled codebase.

### Why Microservices?
As applications grow, a single large codebase becomes harder to understand, test, deploy, and scale. Different parts of the system often have very different needs — e.g., a "search" feature might need heavy read scaling, while "checkout" needs strong consistency. Microservices let you develop, deploy, and scale each business capability **independently**, using the right technology for each job, with different teams able to own different services without stepping on each other.

### Monolithic vs Microservices

**Monolithic:** The entire application (UI, business logic, data access) is built and deployed as **one single unit**. All features share the same codebase, same process, and typically the same database.

```
┌─────────────────────────────────────┐
│           Monolithic App              │
│  ┌────────┐ ┌────────┐ ┌──────────┐ │
│  │  Users  │ │ Orders  │ │ Payments  │ │
│  └────────┘ └────────┘ └──────────┘ │
│              (one codebase,           │
│           one deployment, one DB)     │
└─────────────────────────────────────┘
```

**Microservices:** The application is split into **independent services**, each with its own codebase, deployment, and often its own database.

```
┌──────────┐   ┌──────────┐   ┌──────────┐
│  User     │   │  Order    │   │  Payment  │
│  Service   │   │  Service   │   │  Service   │
│  + own DB  │   │  + own DB  │   │  + own DB  │
└──────────┘   └──────────┘   └──────────┘
      (communicate over network — REST/gRPC/events)
```

| Aspect | Monolithic | Microservices |
|---|---|---|
| Codebase | Single, unified | Multiple, independent per service |
| Deployment | Entire app deployed together | Each service deployed independently |
| Database | Usually one shared database | Database per service |
| Scaling | Scale the entire app, even if only one part needs it | Scale only the specific service that needs it |
| Technology | Usually one tech stack for everything | Each service can use a different stack if needed |
| Team ownership | Harder to split ownership cleanly | Teams can own specific services end-to-end |
| Complexity | Simple to start, harder to maintain at scale | More operational complexity, but scales better organizationally |

### Advantages
- **Independent deployability** — ship a fix to the Payment Service without redeploying the entire application.
- **Independent scalability** — scale only the services under heavy load (e.g., scale Product Service during a sale, without scaling everything else).
- **Technology flexibility** — each service can use the best-suited language/database for its specific job.
- **Fault isolation** — a crash in one service doesn't necessarily bring down the entire system (if designed with resilience patterns like circuit breakers).
- **Team autonomy** — smaller, focused teams can own a service end-to-end, moving faster without coordinating every change across the whole organization.

### Disadvantages
- **Distributed system complexity** — network calls can fail, be slow, or arrive out of order — problems a monolith simply doesn't have (in-process function calls don't fail like network calls do).
- **Data consistency challenges** — no single database/transaction spanning all data, requiring patterns like Saga (see below) instead of simple ACID transactions.
- **Operational overhead** — requires infrastructure for service discovery, API gateways, centralized logging, monitoring, and CI/CD pipelines per service.
- **Harder debugging** — a single user request might touch 5-10 services, making it harder to trace what went wrong without proper tooling (correlation IDs, distributed tracing).
- **Network latency** — calls between services over a network are inherently slower than in-process function calls within a monolith.

### When NOT to use Microservices
- **Small teams / early-stage startups** — the operational overhead of microservices (multiple deployments, service discovery, distributed debugging) often isn't worth it before you have real scaling or organizational pain points.
- **Simple applications** with limited scope, low traffic, or a small number of features.
- **Unclear domain boundaries** — if you don't yet understand your business domains well, splitting into services prematurely often leads to poorly-drawn boundaries that require expensive refactoring later ("distributed monolith" — all the complexity of microservices, none of the benefits).
- **Limited DevOps maturity** — without solid CI/CD, monitoring, and container orchestration experience, managing many independent services becomes a significant burden.

**Common advice:** "Start with a monolith, split into microservices only when you have a concrete reason to" (scaling bottlenecks, team coordination pain, or clear domain boundaries) — this is often called the "monolith-first" approach.

### Interview Questions

**Q: What is a Microservice?**
A microservice is a small, independently deployable service that implements a single business capability, owns its own data, and communicates with other services over the network (typically via REST, gRPC, or asynchronous messaging).

**Q: Difference between Monolithic and Microservices?**
A monolith is a single, unified codebase and deployment unit handling all application functionality, typically sharing one database. Microservices split the application into independent, separately deployable services, each owning its own data store, communicating over a network — trading simplicity for independent scalability, deployability, and team autonomy.

**Q: When would you choose a Monolith?**
When the team is small, the domain isn't well understood yet, time-to-market matters more than long-term scalability, or the application's complexity doesn't yet justify the operational overhead of running and coordinating many independent services. Many successful systems start as a monolith and are split into microservices only once specific scaling or organizational pain points emerge.

---

## 2. Microservice Architecture

### Service Per Business Domain
**Simple definition:** Each microservice should be built around a specific **business capability** (e.g., "manage orders," "process payments") rather than a technical layer (e.g., "all database access code") — this is based on **Domain-Driven Design (DDD)**, specifically the concept of a "bounded context."

### Independent Deployment
Each service can be built, tested, and deployed **without needing to coordinate a release with other services** — a bug fix in the Notification Service shouldn't require redeploying the Payment Service.

### Independent Database
Each service owns and manages its own data store — no service should directly query another service's database (covered in detail in section 6).

### Loose Coupling
**Simple definition:** Services should depend on each other as little as possible — ideally only through well-defined APIs/events, not shared code, shared databases, or hidden dependencies on each other's internal implementation. Changes inside one service shouldn't force changes in another.

### High Cohesion
**Simple definition:** Everything within a single service should be closely related and focused on one business capability. A service that has high cohesion does "one thing well" — e.g., the Order Service handles everything related to order creation, updates, and status, and nothing unrelated (like sending emails, which belongs to the Notification Service).

**Why loose coupling + high cohesion together matter:** This combination is the actual goal behind "microservices" — not just "small size." A service can be relatively large but still be a good microservice if it's highly cohesive (focused on one domain) and loosely coupled (doesn't leak dependencies into other services).

### Example: E-commerce microservices breakdown

```
┌──────────────┐  ┌───────────────┐  ┌──────────────┐
│  User Service  │  │ Product Service │  │ Order Service  │
│  - signup/login │  │ - catalog        │  │ - cart/checkout │
│  - profile        │  │ - search          │  │ - order status   │
└──────────────┘  └───────────────┘  └──────────────┘

┌────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ Payment Service   │  │ Notification Service │  │ Inventory Service   │
│ - process payments  │  │ - email/SMS/push       │  │ - stock levels        │
│ - refunds             │  │ - order confirmations   │  │ - reserve/release stock │
└────────────────┘  └──────────────────┘  └──────────────────┘
```
Each of these owns its own logic and data, and a checkout flow involves the Order Service coordinating with Payment, Inventory, and Notification services — via the communication patterns discussed next.

---

## 3. Communication Between Services

### Synchronous Communication
**Simple definition:** The calling service sends a request and **waits (blocks)** for a response before continuing — like a phone call, where you wait for the other person to answer.

**REST API:** Standard HTTP-based communication (as covered in the REST API notes) — simple, widely understood, uses standard status codes and JSON payloads.

**gRPC:** A high-performance RPC (Remote Procedure Call) framework using **Protocol Buffers** (a binary serialization format) instead of JSON — faster and more compact than REST/JSON, well-suited for internal service-to-service communication where performance matters.

```protobuf
// Example gRPC service definition (.proto file)
service OrderService {
  rpc GetOrder (OrderRequest) returns (OrderResponse);
}

message OrderRequest {
  string orderId = 1;
}

message OrderResponse {
  string status = 1;
  double amount = 2;
}
```

**Pros of synchronous communication:**
- **Simple** — request/response is an intuitive mental model, similar to a normal function call.
- **Easy debugging** — you get an immediate response (or error) right away, making it straightforward to trace what happened.

**Cons of synchronous communication:**
- **Tight coupling** — the calling service must know about and directly depend on the availability of the service it's calling.
- **Waiting for a response** — if the downstream service is slow or down, the calling service is blocked waiting, which can cascade into wider outages (a slow Payment Service can make Order Service requests pile up and time out too).

### Asynchronous Communication
**Simple definition:** The calling service publishes a message/event and **moves on immediately**, without waiting for the receiver to process it — like sending a letter, where you don't wait by the mailbox for a reply.

**Kafka:** A distributed event streaming platform, built for high-throughput, durable event logs (details in section 9).

**RabbitMQ:** A traditional message broker/queue system, good for task distribution and routing (details in section 9).

**Redis Pub/Sub:** Lightweight publish/subscribe messaging built into Redis — fast, but messages aren't persisted (if no subscriber is listening at the moment of publish, the message is lost).

**Pros of asynchronous communication:**
- **Better scalability** — producers and consumers are decoupled in time; a slow consumer doesn't block the producer.
- **Event-driven** — naturally supports reacting to things happening elsewhere in the system (e.g., "when an order is placed, notify inventory, notify shipping, notify analytics" — all independently, all triggered by one event).

**Cons of asynchronous communication:**
- **Eventual consistency** — data across services isn't updated at the exact same instant; there's a window where different services have different, temporarily-inconsistent views of the data.
- **Debugging difficulty** — tracing what happened across a chain of asynchronous events (published by one service, consumed by three others at different times) is significantly harder than following a straightforward request-response chain.

### Interview Question: When should REST be used instead of Kafka?
Use REST (synchronous) when the caller **needs an immediate response** to proceed (e.g., "is this payment approved right now?" during checkout), when the interaction is a simple, direct request for data (e.g., "get user profile"), or when strong immediate consistency matters more than scalability. Use Kafka/async messaging when the operation doesn't require an immediate response, when multiple services need to react to the same event independently (fan-out), or when you want to decouple services so a slow/down consumer doesn't block the producer.

---

## 4. API Gateway

**Simple definition:** An API Gateway is a **single entry point** that sits in front of all your microservices — clients talk to the gateway, and the gateway routes requests to the appropriate backend service, while also handling common cross-cutting concerns centrally.

```
┌────────┐        ┌─────────────┐       ┌───────────────┐
│ Client  │ ─────► │ API Gateway  │ ────► │ User Service    │
└────────┘        │              │       ├───────────────┤
                   │ - Auth        │ ────► │ Order Service   │
                   │ - Rate limit  │       ├───────────────┤
                   │ - Routing     │ ────► │ Payment Service │
                   │ - Logging     │       └───────────────┘
                   └─────────────┘
```

### Responsibilities
- **Authentication** — verify who the client is (e.g., validate a JWT) before forwarding the request.
- **Authorization** — check the client has permission for the requested action.
- **Rate Limiting** — protect backend services from being overwhelmed by too many requests.
- **Routing** — direct incoming requests to the correct downstream microservice based on the URL path.
- **Logging** — centrally log all incoming requests for observability.
- **SSL Termination** — handle HTTPS encryption/decryption at the gateway, so internal services can communicate over plain HTTP within a trusted network.
- **Load Balancing** — distribute incoming requests across multiple instances of a service.

### Popular API Gateway tools
- **Kong** — open-source, plugin-based API gateway built on Nginx.
- **Nginx** — widely used as a reverse proxy/API gateway due to its performance and flexibility.
- **AWS API Gateway** — fully managed cloud service, integrates tightly with other AWS services (Lambda, Cognito, etc.).

### Interview Question: Why API Gateway?
Without an API Gateway, clients would need to know the address of every individual microservice, implement authentication/rate-limiting logic themselves, and handle the complexity of calling multiple services directly (increasing coupling between the client and internal architecture). The API Gateway centralizes these cross-cutting concerns in one place, simplifies the client's interaction to a single entry point, and lets internal service architecture change freely without breaking clients.

---

## 5. Service Discovery

### Concept
**Simple definition:** In a microservices system, service instances can be created, destroyed, or moved to different network addresses frequently (especially with auto-scaling/container orchestration) — service discovery is the mechanism that lets services **find each other's current network location dynamically**, instead of relying on hardcoded addresses that would quickly become outdated.

```
❌ Without service discovery — hardcoded, fragile:
Order Service → calls "http://192.168.1.10:5000" (Payment Service)
   (breaks the moment Payment Service moves to a new address)

✅ With service discovery — dynamic lookup:
Order Service → asks Service Registry "where is payment-service right now?"
             → gets back the current address
             → calls that address
```

### How it works (simplified)
1. Each service instance **registers itself** with a central **Service Registry** on startup (announcing its name and current network address).
2. When Order Service needs to call Payment Service, it **queries the registry** by name ("payment-service") instead of using a hardcoded IP/port.
3. The registry returns the current, healthy address(es) for that service.
4. If a service instance goes down, it's removed from the registry (often via periodic health checks), so traffic isn't routed to dead instances.

### Examples
- **Eureka** — a service registry originally from Netflix's microservices stack, commonly used in Spring Boot/Java ecosystems.
- **Consul** — a service discovery and configuration tool from HashiCorp, supports health checking and key-value configuration storage too.
- **Kubernetes DNS** — in Kubernetes, services get a stable DNS name automatically (e.g., `payment-service.default.svc.cluster.local`), and Kubernetes handles routing that name to healthy pod instances — meaning many teams running on Kubernetes get service discovery "for free" without needing a separate tool like Eureka/Consul.

---

## 6. Database Per Service

**Simple definition:** Each microservice owns and manages its **own dedicated database** — no other service is allowed to directly read or write to that database. Any data another service needs must be obtained through that owning service's API (or via asynchronous events), never through direct database access.

```
User Service  →  MongoDB
Order Service →  PostgreSQL
Inventory     →  MySQL
```

**Why this matters — it's what actually enables independent services:** If multiple services shared one database, changing a table's schema in one service could silently break another service that also reads that table directly — this creates hidden coupling that defeats the entire purpose of splitting into microservices in the first place. Database-per-service ensures each team can change their own data model freely, as long as their public API contract stays stable.

**The trade-off this creates:** Since data lives in separate databases, you lose the ability to do simple SQL `JOIN`s across services' data, and you lose single-database ACID transactions spanning multiple services — this is exactly why patterns like the **Saga pattern** (section 7) and event-driven architecture (section 8) exist.

### Interview Question: Can one service access another service's database?
**Answer: No.** Doing so creates tight coupling between services at the data layer, defeating the purpose of microservices — a schema change in one service could silently break another. Instead, services should expose their data through well-defined APIs or publish events when their data changes, letting other services keep their own local copy of whatever data they need (often maintained via events).

---

## 7. Distributed Transactions

### The Problem
In a monolith with one database, a multi-step operation (like placing an order) can be wrapped in a single ACID transaction — if any step fails, everything rolls back automatically. But in microservices, each step might touch a **different service with its own database**:

```
1. Create Order        (Order Service DB)
        ↓
2. Process Payment      (Payment Service DB)
        ↓
3. Reserve Inventory    (Inventory Service DB)  ← fails!

Now what? The order was created and payment was taken,
but there's no stock. How do we "roll back" steps 1 and 2,
when they live in completely separate databases with no shared transaction?
```

### The Solution: Saga Pattern
**Simple definition:** A Saga is a sequence of **local transactions**, where each step updates its own service's database and then triggers the next step. If any step fails, the Saga executes a series of **compensating transactions** to undo the effects of the previous successful steps — since you can't do a true database rollback across services, you manually "undo" via explicit reversing actions.

```
1. Create Order        ✅
2. Process Payment      ✅
3. Reserve Inventory    ❌ FAILS

Compensating actions triggered:
   → Refund Payment (undo step 2)
   → Cancel Order (undo step 1)
```

### Types of Saga

**1. Choreography** — no central coordinator; each service listens for events and reacts, publishing its own events in turn.
```
Order Service:   creates order → publishes "OrderCreated"
Payment Service: listens for "OrderCreated" → charges payment → publishes "PaymentCompleted"
Inventory:       listens for "PaymentCompleted" → reserves stock → publishes "InventoryReserved" (or "InventoryFailed")
Order Service:   listens for "InventoryFailed" → triggers compensating "CancelOrder"
```
- **Pros:** Simple, fully decentralized, no single point of failure/coordination.
- **Cons:** As the number of steps grows, it becomes hard to see/understand the overall flow, since the logic is scattered across many services' event handlers.

**2. Orchestration** — a central **orchestrator** service explicitly tells each service what to do next and handles the overall flow/compensation logic.
```
Order Orchestrator:
  1. Tell Order Service: create order
  2. Tell Payment Service: charge payment
  3. Tell Inventory Service: reserve stock
     → if this fails: tell Payment Service to refund, tell Order Service to cancel
```
- **Pros:** Centralized, explicit flow — much easier to understand, monitor, and modify the overall business process in one place.
- **Cons:** The orchestrator itself becomes a critical component (needs to be highly available), and can become a bottleneck or a "God object" if it grows to encapsulate too much logic.

### Interview Question: Explain the Saga Pattern.
The Saga pattern manages data consistency across multiple services in a distributed transaction by breaking it into a sequence of local transactions, each committed independently within its own service. If any step fails, previously completed steps are undone using compensating transactions (explicit reversing operations, like a refund to undo a charge), rather than relying on a traditional atomic rollback — which isn't possible across separate databases. It can be implemented via choreography (services react to each other's events independently) or orchestration (a central coordinator explicitly directs each step and handles failures).

---

## 8. Event Driven Architecture

**Simple definition:** An architectural style where services communicate primarily by **publishing and reacting to events** — "something happened" — rather than directly calling each other. Services that care about a particular event **subscribe** to it and react independently, without the publisher needing to know who's listening.

```
Order Created
     ↓ (published as an event)
     ├──► Inventory Service   (reserves stock)
     ├──► Notification Service (sends confirmation email)
     └──► Analytics Service    (records the sale for reporting)
```
**Key benefit:** The Order Service doesn't need to know about Inventory, Notification, or Analytics at all — it just publishes "OrderCreated," and any current or future service that cares can subscribe, without ever modifying the Order Service's code. This is a powerful form of loose coupling.

### Key concepts to prepare
- **Event Publisher:** The service that detects something happened and emits an event describing it.
- **Event Consumer:** A service that subscribes to and processes specific event types.
- **Retry:** If a consumer fails to process an event (e.g., a temporary database outage), the message broker should support retrying delivery rather than losing the event.
- **Dead Letter Queue (DLQ):** After repeated failed processing attempts, a message is moved to a separate "dead letter" queue instead of being retried forever — allowing engineers to investigate and manually handle problematic messages without blocking the main queue.

---

## 9. Message Broker

### Kafka
**Simple definition:** Kafka is a **distributed event streaming platform** designed for high-throughput, durable, ordered event logs — commonly used as the backbone for event-driven microservices architectures.

| Concept | Meaning |
|---|---|
| **Topic** | A named category/feed to which events are published (e.g., "order-events") |
| **Producer** | A service that publishes (writes) events to a topic |
| **Consumer** | A service that reads/processes events from a topic |
| **Partition** | A topic is split into partitions for parallelism — each partition is an ordered, append-only log |
| **Offset** | A sequential ID identifying a message's position within a partition — consumers track their offset to know what they've already processed |
| **Consumer Group** | A set of consumers that share the work of processing a topic's messages — each partition is consumed by only one consumer within the group at a time, enabling horizontal scaling of consumption |

```
Topic: "order-events"
  Partition 0: [msg0, msg1, msg2, ...]
  Partition 1: [msg0, msg1, msg2, ...]

Consumer Group "inventory-service":
  Consumer A → reads Partition 0
  Consumer B → reads Partition 1
  (work is split across consumers in the group)
```

### RabbitMQ
**Simple definition:** RabbitMQ is a traditional **message broker** implementing the AMQP protocol — messages are routed through **exchanges** into **queues**, and consumers pull messages off those queues.

| Concept | Meaning |
|---|---|
| **Queue** | Where messages actually sit, waiting to be consumed (FIFO by default) |
| **Exchange** | Receives messages from producers and routes them to the appropriate queue(s) based on rules |
| **Routing Key** | A label attached to a message, used by the exchange to decide which queue(s) it should go to |

### Kafka vs RabbitMQ
| | Kafka | RabbitMQ |
|---|---|---|
| Core model | Distributed, persistent event log | Traditional message queue |
| Message retention | Retains messages for a configured period, even after consumption (replayable) | Messages typically removed once consumed/acknowledged |
| Throughput | Very high — built for massive event streams | High, but generally lower than Kafka at extreme scale |
| Ordering | Guaranteed within a partition | Guaranteed within a single queue |
| Best for | Event sourcing, log aggregation, high-throughput streaming, event replay | Complex routing logic, task queues, request/reply patterns |
| Complexity | More operationally complex to run | Generally simpler to set up and reason about for smaller-scale needs |

---

## 10. Caching (Redis, in a microservices context)

### Cache Aside (Lazy Loading)
The application checks the cache first; on a miss, it queries the database and populates the cache for next time (see the earlier Caching notes for full detail and code examples).

### Write Through
Every write goes to the cache **and** the database simultaneously, keeping them always in sync — slightly slower writes, but cache is never stale.

### Write Behind (Write Back)
Writes go to the cache first and are asynchronously flushed to the database later — faster writes, but risk of data loss if the cache fails before the write reaches the database.

### Interview Question: Where should cache be placed?
Caching can exist at multiple layers in a microservices system:
- **Client-side/CDN:** For static or rarely-changing public content.
- **API Gateway level:** Caching responses for frequently-requested, cacheable endpoints before they even reach backend services.
- **Service level (application cache):** Each service caches its own frequently-accessed data (e.g., Product Service caching popular product details in Redis) — this is the most common placement, since it respects the "database per service" boundary and lets each service manage its own cache invalidation logic.
- **Database query cache:** Caching expensive query results within a service.

**Best practice:** Cache should generally live **within the owning service**, not shared globally across services, to avoid violating service boundaries and creating hidden coupling (the same reasoning behind "database per service").

---

## 11. Authentication (in Microservices)

### JWT
Stateless tokens (see the Authentication notes for full detail) — a natural fit for microservices, since any service can independently verify a JWT's signature without needing to call a central session store.

### OAuth
A framework for delegated authorization (e.g., "login with Google") — commonly used for both third-party login and securing service-to-service API access via access tokens.

### OpenID Connect (OIDC)
An identity layer built on top of OAuth 2.0, specifically designed for **authentication** (proving who a user is) — OAuth alone is technically about authorization, and OIDC adds the standardized identity/login layer on top.

### API Gateway Authentication
**Common pattern:** The API Gateway validates the client's token (e.g., verifies the JWT signature and expiry) **once**, at the edge, before forwarding the request to internal services — internal services can then trust that requests arriving from the gateway are already authenticated, often via a simpler internal header/token, avoiding the need to re-validate the full auth flow in every single service.

### Refresh Token
Used to obtain a new access token without forcing the user to log in again (full detail in the Authentication notes).

### SSO (Single Sign-On)
**Simple definition:** Lets a user log in once and gain access to multiple related applications/services without needing to log in separately to each one — typically implemented using a central identity provider (IdP) that issues a token trusted across all connected applications.

### Interview Question: How does authentication work in microservices?
Typically, a client authenticates once (often via the API Gateway or a dedicated Auth Service), receiving a JWT access token. The API Gateway validates this token at the edge for every incoming request. The validated identity/claims are then passed along to internal services (e.g., via a lightweight internal header or a re-signed internal token), so downstream services don't need to repeat the full authentication handshake — they can trust the gateway's validation and simply read the user identity/claims to perform their own authorization checks.

---

## 12. Logging

### Centralized Logging
**Simple definition:** Since a single user request can touch many different services, logs from every service need to be collected into **one centralized system**, rather than being scattered across dozens of individual servers/containers where they'd be nearly impossible to correlate manually.

**ELK Stack:** Elasticsearch (storage/search), Logstash (log collection/processing), Kibana (visualization) — a widely used centralized logging stack.

**Grafana + Loki:** Loki is a log aggregation system designed to work efficiently with Grafana for visualization — often considered a lighter-weight alternative to the full ELK stack.

### Correlation ID / Trace ID
**Simple definition:** A unique identifier generated at the **start** of a request (usually at the API Gateway) and passed along through **every** subsequent service call involved in handling that request — allowing you to search your centralized logs for that one ID and see the complete journey of a single request across every service it touched.

```
Client Request → API Gateway generates traceId: "abc-123"
   → Order Service logs: [traceId: abc-123] "Creating order..."
   → Payment Service logs: [traceId: abc-123] "Charging payment..."
   → Inventory Service logs: [traceId: abc-123] "Reserving stock..."

// Searching logs for "abc-123" reveals the ENTIRE request's journey across all 3 services
```

### Interview Question: How do you debug a request across 10 services?
By using a **correlation/trace ID** generated at the entry point (API Gateway) and propagated through every downstream service call (typically via an HTTP header like `X-Trace-Id`). Every service includes this ID in its logs. With centralized logging (ELK/Loki) and this shared trace ID, you can search across all services' logs at once and reconstruct the full path and timing of that specific request, even though it touched many independent services. Distributed tracing tools (like Jaeger or Zipkin) build on this same concept, visualizing the entire call chain and timing as a single trace diagram.

---

## 13. Monitoring

**Prometheus:** An open-source monitoring system that **scrapes metrics** (numeric time-series data — CPU usage, request counts, latency) from services at regular intervals and stores them for querying/alerting.

**Grafana:** A visualization tool, commonly paired with Prometheus, that turns raw metrics into dashboards and graphs.

**Health Check:** An endpoint (e.g., `GET /health`) that a service exposes, reporting whether it's running correctly (and often whether its dependencies, like its database connection, are healthy) — used by load balancers/orchestrators (like Kubernetes) to decide whether to route traffic to that instance.

**Metrics:** Quantitative data about system behavior — request rate, error rate, response time (often referred to as the "RED" metrics: Rate, Errors, Duration), and resource usage (CPU, memory).

**Alerting:** Automatically notifying engineers (via Slack, PagerDuty, email) when metrics cross a defined threshold (e.g., error rate > 5% for 5 minutes), enabling fast response to production issues before they escalate.

---

## 14. Circuit Breaker

**Simple definition:** A resilience pattern that **prevents a service from repeatedly calling a downstream service that's known to be failing** — instead of waiting on (and piling up) doomed requests, the circuit breaker "trips" and immediately returns a fallback response, giving the failing service time to recover.

**Real-world analogy:** Just like an electrical circuit breaker cuts power to prevent damage when there's a fault, a software circuit breaker "cuts off" calls to a failing service to prevent cascading failures across the system.

```
Order Service → Payment Service (down/timing out)

Without circuit breaker: every Order request waits for Payment to time out,
  piling up threads/connections, eventually crashing Order Service too (cascading failure).

With circuit breaker: after enough failures, the circuit "opens" —
  Order Service immediately returns a fallback response (e.g., "Payment temporarily unavailable")
  WITHOUT even attempting to call Payment Service, until it's confirmed healthy again.
```

### The three states of a circuit breaker
| State | Behavior |
|---|---|
| **Closed** | Normal operation — requests pass through to the downstream service |
| **Open** | Downstream service has failed too many times — requests immediately fail/fallback without even attempting the call |
| **Half-Open** | After a cooldown period, a limited number of test requests are allowed through to check if the downstream service has recovered — if they succeed, the circuit closes again; if they fail, it reopens |

### Popular libraries
- **Hystrix** — Netflix's original circuit breaker library (now in maintenance mode/deprecated, but historically very influential).
- **Resilience4j** — a modern, lightweight alternative widely used in Java microservices today.

### Interview Question: Explain Circuit Breaker.
A circuit breaker monitors calls to a downstream service and "trips open" after detecting a certain threshold of failures, causing subsequent calls to fail fast (returning a fallback) instead of continuing to hammer a struggling service — this prevents cascading failures where one slow/down service causes threads/resources in calling services to pile up and crash as well. After a cooldown period, the circuit enters a half-open state, allowing a few test requests through to check if the downstream service has recovered before fully resuming normal traffic.

---

## 15. Retry Pattern

**Simple definition:** Automatically re-attempting a failed operation, since many failures in distributed systems are **transient** (temporary network blips, brief service restarts) rather than permanent — a simple retry often succeeds where the original attempt failed.

- **Retry:** Re-attempt the failed call, typically a limited number of times.
- **Exponential Backoff:** Increase the wait time between each retry attempt exponentially (e.g., 1s, 2s, 4s, 8s...) instead of retrying immediately and repeatedly — this avoids overwhelming an already-struggling service with a rapid burst of retry traffic, and gives it time to recover.
- **Timeout:** Define a maximum time to wait for a response before giving up on that attempt — without a timeout, a single slow call could hang indefinitely, tying up resources.

**Important pairing:** Retry and Circuit Breaker work well together — retries handle brief, transient blips, while the circuit breaker prevents endless retrying against a service that's genuinely down for an extended period.

---

## 16. Load Balancing

**Simple definition:** Distributing incoming requests across **multiple instances** of a service, so no single instance is overwhelmed while others sit idle — essential for both performance and high availability.

- **Round Robin:** Requests are distributed sequentially, one after another, to each available instance in turn (Instance A, B, C, A, B, C...).
- **Least Connection:** Routes each new request to whichever instance currently has the **fewest active connections** — better suited when requests vary significantly in processing time, avoiding overloading an instance that's still busy with previous long-running requests.
- **Sticky Session:** Ensures a specific client's requests are **consistently routed to the same instance** for the duration of their session — useful when session state is stored in-memory on a specific instance (though a more scalable approach is often to store session state externally, e.g., in Redis, removing the need for stickiness at all).

---

## 17. Containerization (Docker)

**Simple definition:** Docker packages an application along with everything it needs to run (code, runtime, libraries, system dependencies) into a **portable, isolated unit called a container** — ensuring it runs identically regardless of the underlying host machine ("works on my machine" problems largely disappear).

| Concept | Meaning |
|---|---|
| **Dockerfile** | A text file with step-by-step instructions for building a Docker image (base OS, dependencies, code, startup command) |
| **Image** | A read-only, packaged template built from a Dockerfile — the "blueprint" for a container |
| **Container** | A running instance of an image — the actual live process, isolated from the host and other containers |
| **Volume** | Persistent storage that survives beyond a container's lifecycle, used for data that shouldn't be lost when a container is removed/restarted |
| **Network** | Virtual networking that lets containers communicate with each other and the outside world |

```dockerfile
# Example Dockerfile for a Node.js service
FROM node:20
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```

### Interview Question: Difference between Docker Image and Container?
An **image** is a static, read-only template/blueprint containing everything needed to run an application — it doesn't run by itself. A **container** is a live, running instance created **from** an image — you can create multiple containers from the same single image, each running independently with its own isolated process, filesystem layer, and network. Analogy: an image is like a class definition; a container is an instantiated object of that class.

---

## 18. Kubernetes Basics

**Simple definition:** Kubernetes (K8s) is a **container orchestration platform** — it automates deploying, scaling, networking, and managing the lifecycle of containerized applications across a cluster of machines, which is essential once you're running many microservices as many containers.

| Concept | Meaning |
|---|---|
| **Pod** | The smallest deployable unit in Kubernetes — typically wraps one (or a few tightly-coupled) container(s) |
| **Deployment** | Describes the desired state for a set of pods (which image, how many replicas) — Kubernetes continuously works to match reality to this desired state |
| **ReplicaSet** | Ensures a specified number of identical pod replicas are running at all times, replacing any that crash |
| **Service** | A stable network endpoint/DNS name that routes traffic to a set of pods, even as individual pods are created/destroyed/rescheduled |
| **ConfigMap** | Stores non-sensitive configuration data (e.g., feature flags, API URLs) separately from application code |
| **Secret** | Like a ConfigMap, but for sensitive data (passwords, API keys, tokens) — stored more securely |
| **Namespace** | A way to logically partition a cluster's resources (e.g., separate `dev`, `staging`, `production` environments within one cluster) |
| **Ingress** | Manages external HTTP(S) access into the cluster, routing incoming traffic to the appropriate internal Services based on hostname/path rules |

### Interview Question: How does Kubernetes scale microservices?
Kubernetes scales microservices primarily through **Deployments** and **ReplicaSets** — you specify a desired number of pod replicas for a service, and Kubernetes ensures that many are always running (recreating any that crash). With **Horizontal Pod Autoscaling (HPA)**, Kubernetes can automatically increase or decrease the number of running pod replicas based on real-time metrics like CPU/memory usage or custom application metrics, scaling out under heavy load and scaling back in during quiet periods — all without manual intervention.

---

## 19. CI/CD

**Simple definition:** CI/CD automates the process of **building, testing, and deploying** code changes — Continuous Integration (CI) automatically builds and tests every code change, while Continuous Deployment/Delivery (CD) automatically ships passing changes toward (or all the way into) production.

- **GitHub Actions / Jenkins:** Tools that define automated pipelines — triggered on events like a code push or pull request — to run tests, build artifacts, and deploy.
- **Docker Build:** As part of the pipeline, the application is packaged into a Docker image.
- **Deployment Pipeline:** The automated sequence of stages a change passes through — build → test → package (Docker image) → deploy to staging → (optionally) deploy to production — with each stage needing to pass before proceeding to the next.

**Why this matters especially for microservices:** With dozens of independently deployable services, manual deployment simply doesn't scale — CI/CD pipelines (often one per service) are what actually make "independent deployment" a practical reality rather than just a theoretical benefit.

---

## 20. Security

Key security concerns specific to (and general practices important for) a microservices architecture:
- **HTTPS** — encrypt all traffic, both external (client-to-gateway) and ideally internal (service-to-service) too.
- **JWT / OAuth** — for authenticating both end-users and service-to-service calls.
- **API Gateway Security** — centralizing authentication, rate limiting, and input validation at the edge before requests reach internal services.
- **Secret Management** — never hardcode credentials/API keys in code or config files; use a dedicated secret store (e.g., HashiCorp Vault, AWS Secrets Manager, Kubernetes Secrets).
- **Rate Limiting** — protect against abuse and denial-of-service attempts.
- **CORS** — properly configure which origins are allowed to call your APIs from a browser.

---

## 21. Node.js Microservices

### Frameworks
- **Express** — minimal, unopinionated, highly flexible — you assemble the structure yourself.
- **NestJS** — an opinionated, structured framework (inspired by Angular) built on top of Express/Fastify, providing built-in support for dependency injection, modules, decorators, and a more enterprise-style architecture out of the box — often favored for larger microservices projects needing more built-in structure and conventions.

### Typical folder structure for a Node.js microservice
```
src/
├── controllers/    # handle incoming HTTP requests, call services, return responses
├── routes/         # define API endpoints and map them to controllers
├── service/        # business logic lives here
├── repository/     # database access logic (queries, ORM calls)
├── events/         # event publishers/consumers (Kafka/RabbitMQ handlers)
├── middlewares/    # auth, validation, error handling, logging
└── config/         # environment variables, database connections, third-party configs
```
This layered structure keeps concerns separated — controllers don't contain business logic, business logic doesn't contain raw SQL/database queries, and event handling is isolated from the HTTP-request-handling code path.

### Communication
- **REST** — for direct synchronous service-to-service or client-to-service calls.
- **Kafka / RabbitMQ** — for asynchronous, event-driven communication between services.

---

## 22. Design Questions to Prepare

For each of these, practice sketching out: the core microservices involved, how they communicate (sync vs async), what data each owns, and how you'd handle failures/consistency.

- Design Food Delivery Application
- Design E-commerce Backend
- Design Payment Service
- Design Notification Service
- Design OTP Service
- Design Chat Application
- Design Order Management System
- Design Inventory Service
- Design Ticket Booking System

**General approach for these interview questions:**
1. Clarify scope and scale (how many users, requests/sec, etc.).
2. Identify the core business domains/entities → map each to a candidate service.
3. Decide sync vs async communication for each interaction (does the caller need an immediate response?).
4. Identify where consistency matters most (e.g., payment, inventory) and discuss the Saga pattern if a multi-service transaction is involved.
5. Mention supporting infrastructure: API Gateway, service discovery, caching, monitoring, and resilience patterns (circuit breaker, retry) where relevant.
6. Discuss potential failure scenarios and how the design handles them (e.g., "what happens if the Notification Service is down when an order is placed?").

---

## 23. Real Project Questions (Frequently Asked)

These are behavioral/experience-based questions interviewers use to verify you've actually worked with microservices in practice, not just studied the theory. Prepare specific, concrete answers from your own project experience for each:

- **How many microservices were in your project?** — Be ready to name a few and briefly describe what each owned.
- **Why did you split the application?** — Discuss the actual pain point (scaling bottleneck, team size, deployment coordination issues) that motivated the split, not just "because microservices are popular."
- **How did services communicate?** — Be specific: which interactions were REST, which were event-driven (Kafka/RabbitMQ), and why each choice was made.
- **How did authentication work?** — Describe your actual JWT/OAuth/API Gateway setup.
- **How did you deploy services?** — Discuss your actual CI/CD pipeline, containerization (Docker), and orchestration (Kubernetes or similar).
- **How did you monitor production?** — Mention specific tools used (Prometheus/Grafana, ELK, etc.) and what you actually tracked/alerted on.
- **How did you handle failures?** — Discuss specific resilience patterns you implemented — circuit breakers, retries, fallback responses.
- **How did you handle database transactions?** — Discuss a concrete example of a multi-service operation and how you used the Saga pattern (choreography or orchestration) or otherwise ensured consistency.
- **How did you avoid duplicate events?** — Discuss idempotency: using unique event IDs, checking if an event's effect was already applied before reprocessing (idempotent consumers), or deduplication logic at the message broker/consumer level.
- **How did you version APIs?** — Discuss your actual versioning strategy (URI versioning like `/v1/`, `/v2/`, or header-based) and how you handled backward compatibility as services evolved independently.

**Interview tip:** For all of these, concrete, specific answers ("we used Kafka with a consumer group per service, and handled duplicate events using an idempotency key stored in Redis with a 24-hour TTL") are far more convincing than generic, textbook-style answers — even if you have to reconstruct the details from a project you worked on some time ago.

---

## Quick Summary Cheat Sheet

| Concept | One-liner |
|---|---|
| Microservices | Independent services, each owning one business capability and its own data |
| Loose Coupling / High Cohesion | Services depend on each other minimally; each service is internally focused on one domain |
| Sync Communication (REST/gRPC) | Caller waits for a response — simple, but creates tight coupling and blocking |
| Async Communication (Kafka/RabbitMQ) | Caller doesn't wait — scalable and decoupled, but eventually consistent and harder to debug |
| API Gateway | Single entry point handling auth, rate limiting, routing, logging for all services |
| Service Discovery | Dynamically finds a service's current network address instead of hardcoding it |
| Database Per Service | Each service owns its data exclusively — no direct cross-service DB access |
| Saga Pattern | Sequence of local transactions with compensating actions to handle distributed transaction failures |
| Event-Driven Architecture | Services react to published events instead of direct calls, enabling loose coupling |
| Kafka | High-throughput distributed event log (topics, partitions, consumer groups) |
| RabbitMQ | Traditional message broker with flexible routing (queues, exchanges, routing keys) |
| Circuit Breaker | Stops calling a failing downstream service to prevent cascading failures |
| Retry + Exponential Backoff | Re-attempt failed transient operations with increasing delay between attempts |
| Load Balancing | Distributes traffic across multiple instances of a service |
| Docker | Packages an app + dependencies into a portable, isolated container |
| Kubernetes | Orchestrates deployment, scaling, and networking of containers at scale |
| CI/CD | Automates building, testing, and deploying each service independently |
| Correlation/Trace ID | Unique ID passed through every service call to trace one request across a distributed system |
