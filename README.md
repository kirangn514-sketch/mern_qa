# MERN Stack Interview Preparation — Notes Index

A structured collection of interview-ready notes covering the MERN stack, TypeScript, databases, system design, and microservices. Every heading below links **directly to that section** inside its file — click any topic to jump straight there.

---

## 📚 Categories

- [JavaScript & Node.js](#javascript--nodejs)
- [Frontend — React, Next.js & State Management](#frontend--react-nextjs--state-management)
- [Styling — CSS](#styling--css)
- [TypeScript](#typescript)
- [Backend — Express.js, REST & GraphQL](#backend--expressjs-rest--graphql)
- [Databases — SQL & MongoDB](#databases--sql--mongodb)
- [Authentication](#authentication)
- [Performance](#performance)
- [System Design & Microservices](#system-design--microservices)
- [Production Engineering & Deployment](#production-engineering--deployment)
- [Data Structures & Coding Practice](#data-structures--coding-practice)
- [Suggested Reading Order](#suggested-reading-order)

---

## JavaScript & Node.js

### [`JavaScript_ES6_Notes.md`](./JavaScript_ES6_Notes.md)
[1. var, let, const](./JavaScript_ES6_Notes.md#1-var-let-const) · [2. Scope](./JavaScript_ES6_Notes.md#2-scope) · [3. Hoisting](./JavaScript_ES6_Notes.md#3-hoisting) · [4. Closures](./JavaScript_ES6_Notes.md#4-closures) · [5. Event Loop](./JavaScript_ES6_Notes.md#5-event-loop) · [6. Call Stack](./JavaScript_ES6_Notes.md#6-call-stack) · [7. Microtasks vs Macrotasks](./JavaScript_ES6_Notes.md#7-microtasks-vs-macrotasks) · [8. Promises](./JavaScript_ES6_Notes.md#8-promises) · [9. async/await](./JavaScript_ES6_Notes.md#9-asyncawait) · [10. Prototype & Inheritance](./JavaScript_ES6_Notes.md#10-prototype-inheritance) · [11. this](./JavaScript_ES6_Notes.md#11-this) · [12. call/apply/bind](./JavaScript_ES6_Notes.md#12-call-apply-bind) · [13. Arrow Functions](./JavaScript_ES6_Notes.md#13-arrow-functions) · [14. Destructuring](./JavaScript_ES6_Notes.md#14-destructuring) · [15. Spread vs Rest](./JavaScript_ES6_Notes.md#15-spread-vs-rest) · [16. Map, Filter, Reduce](./JavaScript_ES6_Notes.md#16-map-filter-reduce) · [17. Debounce](./JavaScript_ES6_Notes.md#17-debounce) · [18. Throttle](./JavaScript_ES6_Notes.md#18-throttle) · [19. Deep vs Shallow Copy](./JavaScript_ES6_Notes.md#19-deep-copy-vs-shallow-copy) · [20. Memory Leaks](./JavaScript_ES6_Notes.md#20-memory-leaks) · [21. Optional Chaining](./JavaScript_ES6_Notes.md#21-optional-chaining) · [22. Nullish Coalescing](./JavaScript_ES6_Notes.md#22-nullish-coalescing) · [23. Modules (import/export)](./JavaScript_ES6_Notes.md#23-modules-importexport) · [🎯 Interview Q&A](./JavaScript_ES6_Notes.md#interview-questions-model-answers)

### [`NodeJS_Notes.md`](./NodeJS_Notes.md)
**🏛️ Architecture:** [1. Event Loop](./NodeJS_Notes.md#1-event-loop) · [2. Non-blocking I/O](./NodeJS_Notes.md#2-non-blocking-io) · [3. Streams](./NodeJS_Notes.md#3-streams) · [4. Buffers](./NodeJS_Notes.md#4-buffers) · [5. Cluster](./NodeJS_Notes.md#5-cluster)
**📦 Core Modules:** [fs](./NodeJS_Notes.md#fs-file-system) · [path](./NodeJS_Notes.md#path) · [http](./NodeJS_Notes.md#http) · [os](./NodeJS_Notes.md#os) · [crypto](./NodeJS_Notes.md#crypto)
**Other:** [⚠️ Error Handling](./NodeJS_Notes.md#error-handling) · [📝 Logging](./NodeJS_Notes.md#logging) · [🔐 JWT Authentication](./NodeJS_Notes.md#jwt-authentication) · [🔄 Refresh Token](./NodeJS_Notes.md#refresh-token) · [🔒 Password Hashing](./NodeJS_Notes.md#password-hashing) · [🧂 Bcrypt](./NodeJS_Notes.md#bcrypt) · [📤 File Upload](./NodeJS_Notes.md#file-upload) · [📁 Multer](./NodeJS_Notes.md#multer) · [📧 Email](./NodeJS_Notes.md#email) · [🔢 OTP](./NodeJS_Notes.md#otp-one-time-password) · [🎯 Interview Q&A](./NodeJS_Notes.md#interview-questions-model-answers)

---

## Frontend — React, Next.js & State Management

### [`reactjs-notes.md`](./reactjs-notes.md)
[1. What is React?](./reactjs-notes.md#1-what-is-react)
**Part 1 — Hooks:** [useState](./reactjs-notes.md#2-usestate) · [useEffect](./reactjs-notes.md#3-useeffect) · [useMemo](./reactjs-notes.md#4-usememo) · [useCallback](./reactjs-notes.md#5-usecallback) · [useRef](./reactjs-notes.md#6-useref) · [useReducer](./reactjs-notes.md#7-usereducer) · [Custom Hooks](./reactjs-notes.md#8-custom-hooks)
**Part 2 — Concepts:** [Virtual DOM](./reactjs-notes.md#9-virtual-dom) · [Reconciliation](./reactjs-notes.md#10-reconciliation) · [Rendering Process](./reactjs-notes.md#11-react-rendering-the-render-process) · [Controlled Components](./reactjs-notes.md#12-controlled-components) · [Uncontrolled Components](./reactjs-notes.md#13-uncontrolled-components) · [Lifting State Up](./reactjs-notes.md#14-lifting-state-up) · [Props Drilling](./reactjs-notes.md#15-props-drilling) · [Context API](./reactjs-notes.md#16-context-api) · [Lazy Loading](./reactjs-notes.md#17-lazy-loading) · [Suspense](./reactjs-notes.md#18-suspense) · [Error Boundary](./reactjs-notes.md#19-error-boundary) · [Memoization](./reactjs-notes.md#20-memoization-in-react-recap)
**End:** [Interview Q&A](./reactjs-notes.md#interview-questions-answers) · [Cheat Sheet](./reactjs-notes.md#quick-summary-cheat-sheet)

### [`nextjs-notes.md`](./nextjs-notes.md)
[1. What is Next.js?](./nextjs-notes.md#1-what-is-nextjs) · [2. App Router](./nextjs-notes.md#2-app-router) · [3. Rendering Strategies (SSR/CSR/SSG/ISR)](./nextjs-notes.md#3-rendering-strategies) · [4. Server vs Client Components](./nextjs-notes.md#4-server-components-vs-client-components) · [5. Data Fetching](./nextjs-notes.md#5-data-fetching) · [6. Routing](./nextjs-notes.md#6-routing) · [7. Middleware](./nextjs-notes.md#7-middleware) · [8. Authentication](./nextjs-notes.md#8-authentication) · [9. Image Optimization](./nextjs-notes.md#9-image-optimization) · [10. SEO](./nextjs-notes.md#10-seo) · [11. Metadata API](./nextjs-notes.md#11-metadata-api) · [12. Environment Variables](./nextjs-notes.md#12-environment-variables) · [13. Deployment](./nextjs-notes.md#13-deployment) · [14. Interview Q&A](./nextjs-notes.md#14-interview-questions-answers) · [Cheat Sheet](./nextjs-notes.md#quick-summary-cheat-sheet)

### [`state-management-notes.md`](./state-management-notes.md)
[1. What is State Management?](./state-management-notes.md#1-what-is-state-management) · [2. Store/Slice/Reducer/Dispatch/Selector](./state-management-notes.md#2-core-concepts-you-must-know) · [3. Redux Toolkit](./state-management-notes.md#3-redux-toolkit) · [4. Zustand](./state-management-notes.md#4-zustand) · [5. Context API](./state-management-notes.md#5-context-api) · [6. Comparison Table](./state-management-notes.md#6-comparison-table) · [Interview Q&A (Redux vs Context, Zustand advantages)](./state-management-notes.md#interview-questions-answers) · [Cheat Sheet](./state-management-notes.md#quick-summary-cheat-sheet)

---

## Styling — CSS

### [`css-notes.md`](./css-notes.md)
[1. What is CSS?](./css-notes.md#1-what-is-css) · [2. Flexbox](./css-notes.md#2-flexbox) · [3. Grid](./css-notes.md#3-grid) · [4. Responsive Design](./css-notes.md#4-responsive-design) · [5. Tailwind CSS](./css-notes.md#5-tailwind-css) · [6. Styled Components](./css-notes.md#6-styled-components) · [7. CSS Modules](./css-notes.md#7-css-modules) · [8. Box Model](./css-notes.md#8-box-model) · [9. Position](./css-notes.md#9-position) · [10. z-index & Stacking Context](./css-notes.md#10-z-index-stacking-context) · [11. Specificity & Cascade](./css-notes.md#11-specificity-the-cascade) · [12. Selectors/Pseudo-classes/elements](./css-notes.md#12-common-selectors-pseudo-classes-pseudo-elements) · [13. CSS Variables](./css-notes.md#13-css-custom-properties-css-variables) · [14. Transitions & Animations](./css-notes.md#14-transitions-animations) · [15. CSS Units](./css-notes.md#15-css-units) · [16. BEM](./css-notes.md#16-bem-naming-convention) · [Cheat Sheet](./css-notes.md#quick-summary-cheat-sheet) · [Decision Guide](./css-notes.md#quick-decision-guide)

---

## TypeScript

### [`typescript-notes.md`](./typescript-notes.md)
[1. What is TypeScript?](./typescript-notes.md#1-what-is-typescript) · [2. Types](./typescript-notes.md#2-types-basic-type-annotations) · [3. Interfaces](./typescript-notes.md#3-interfaces) · [4. Type Alias](./typescript-notes.md#4-type-alias) · [5. Interface vs Type](./typescript-notes.md#5-interface-vs-type-alias-key-differences) · [6. Generics](./typescript-notes.md#6-generics) · [7. Utility Types (Partial/Pick/Omit/Record/Required)](./typescript-notes.md#7-utility-types) · [8. Enums](./typescript-notes.md#8-enums) · [9. Union Types](./typescript-notes.md#9-union-types) · [10. Intersection Types](./typescript-notes.md#10-intersection-types) · [11. Type Guards](./typescript-notes.md#11-type-guards) · [12. Type Assertions](./typescript-notes.md#12-type-assertions) · [13. any vs unknown](./typescript-notes.md#13-any-vs-unknown) · [Cheat Sheet](./typescript-notes.md#quick-summary-cheat-sheet) · [Interview Q&A](./typescript-notes.md#interview-questions-answers)

---

## Backend — Express.js, REST & GraphQL

### [`expressjs-notes.md`](./expressjs-notes.md)
[1. What is Express.js?](./expressjs-notes.md#1-what-is-expressjs) · [2. Middleware](./expressjs-notes.md#2-middleware) · [3. Routing](./expressjs-notes.md#3-routing) · [4. Error Middleware](./expressjs-notes.md#4-error-middleware) · [5. Authentication](./expressjs-notes.md#5-authentication) · [6. Authorization](./expressjs-notes.md#6-authorization) · [7. JWT](./expressjs-notes.md#7-jwt-json-web-token) · [8. CORS](./expressjs-notes.md#8-cors-cross-origin-resource-sharing) · [9. Helmet](./expressjs-notes.md#9-helmet) · [10. Rate Limiter](./expressjs-notes.md#10-rate-limiter) · [11. Validation](./expressjs-notes.md#11-validation) · [12. MVC Structure](./expressjs-notes.md#12-mvc-structure) · [Cheat Sheet](./expressjs-notes.md#quick-summary-cheat-sheet) · [Interview Q&A](./expressjs-notes.md#interview-style-quick-answers)

### [`rest-api-notes.md`](./rest-api-notes.md)
[1. What is a REST API?](./rest-api-notes.md#1-what-is-a-rest-api) · [2. CRUD APIs](./rest-api-notes.md#2-crud-apis) · [3. HTTP Methods](./rest-api-notes.md#3-http-methods) · [4. Status Codes](./rest-api-notes.md#4-status-codes) · [5. Pagination](./rest-api-notes.md#5-pagination) · [6. Filtering](./rest-api-notes.md#6-filtering) · [7. Sorting](./rest-api-notes.md#7-sorting) · [8. Auth in REST](./rest-api-notes.md#8-authentication-authorization-in-rest-context) · [9. Idempotency](./rest-api-notes.md#9-idempotency) · [10. Versioning](./rest-api-notes.md#10-versioning) · [11. Rate Limiting](./rest-api-notes.md#11-rate-limiting) · [Interview Q&A (PUT vs PATCH, 401 vs 403, REST principles)](./rest-api-notes.md#interview-questions-answers) · [Cheat Sheet](./rest-api-notes.md#quick-summary-cheat-sheet)

### [`graphql-notes.md`](./graphql-notes.md)
[1. What is GraphQL?](./graphql-notes.md#1-what-is-graphql) · [2. Schema](./graphql-notes.md#2-schema) · [3. Query](./graphql-notes.md#3-query) · [4. Mutation](./graphql-notes.md#4-mutation) · [5. Resolver](./graphql-notes.md#5-resolver) · [6. Apollo Server](./graphql-notes.md#6-apollo-server) · [7. REST vs GraphQL](./graphql-notes.md#7-rest-vs-graphql) · [Cheat Sheet](./graphql-notes.md#quick-summary-cheat-sheet) · [Interview Q&A](./graphql-notes.md#interview-style-quick-answers)

---

## Databases — SQL & MongoDB

### [`database-notes.md`](./database-notes.md)
**Part 1 — MongoDB:** [1. What is MongoDB?](./database-notes.md#1-what-is-mongodb) · [2. Embedding vs Referencing](./database-notes.md#2-embedding-vs-referencing) · [3. $lookup](./database-notes.md#3-lookup-lookup) · [4. Aggregation](./database-notes.md#4-aggregation) · [5. Indexes](./database-notes.md#5-indexes-mongodb) · [6. Transactions](./database-notes.md#6-transactions-mongodb)
**Part 2 — SQL:** [7. Joins](./database-notes.md#7-joins) · [8. Group By](./database-notes.md#8-group-by) · [9. Having](./database-notes.md#9-having) · [10. Order By](./database-notes.md#10-order-by) · [11. Index (Clustered/Composite)](./database-notes.md#11-index-sql) · [12. Transactions](./database-notes.md#12-transactions-sql) · [13. ACID](./database-notes.md#13-acid) · [14. Normalization](./database-notes.md#14-normalization) · [15. Window Functions](./database-notes.md#15-window-functions) · [16. NOLOCK](./database-notes.md#16-nolock) · [17. SQL Functions](./database-notes.md#17-sql-functions) · [18. Stored Procedure](./database-notes.md#18-stored-procedure)
**End:** [Interview Q&A (LEFT vs INNER JOIN, Index, Query Optimization)](./database-notes.md#interview-questions-answers) · [Cheat Sheet](./database-notes.md#quick-summary-cheat-sheet)

---

## Authentication

### [`authentication-notes.md`](./authentication-notes.md)
[1. What is Authentication?](./authentication-notes.md#1-what-is-authentication) · [2. Session](./authentication-notes.md#2-session) · [3. JWT](./authentication-notes.md#3-jwt-json-web-token) · [4. Access Token](./authentication-notes.md#4-access-token) · [5. Refresh Token](./authentication-notes.md#5-refresh-token) · [6. Cookies](./authentication-notes.md#6-cookies) · [7. HttpOnly](./authentication-notes.md#7-httponly) · [8. SameSite](./authentication-notes.md#8-samesite) · [9. OAuth](./authentication-notes.md#9-oauth) · [How It All Fits Together](./authentication-notes.md#how-it-all-fits-together-a-typical-modern-auth-flow) · [Cheat Sheet](./authentication-notes.md#quick-summary-cheat-sheet) · [Interview Q&A](./authentication-notes.md#interview-style-quick-answers)

*Also see security-specific coverage: [SQL/NoSQL Injection, XSS, CSRF, RBAC, bcrypt →](./system-design-production-notes-part1.md#section-3-security-implementation-questions) and [Auth in Node.js — JWT, Refresh Token, Password Hashing, Bcrypt, OTP →](./NodeJS_Notes.md#jwt-authentication)*

---

## Performance

### [`performance-notes.md`](./performance-notes.md)
[1. Why Performance Matters](./performance-notes.md#1-why-performance-matters) · [2. Code Splitting](./performance-notes.md#2-code-splitting) · [3. Lazy Loading](./performance-notes.md#3-lazy-loading) · [4. Image Optimization](./performance-notes.md#4-image-optimization) · [5. Memoization](./performance-notes.md#5-memoization-general-concept) · [6. React.memo](./performance-notes.md#6-reactmemo) · [7. useMemo](./performance-notes.md#7-usememo) · [8. useCallback](./performance-notes.md#8-usecallback) · [9. Virtualization](./performance-notes.md#9-virtualization) · [10. Bundle Optimization](./performance-notes.md#10-bundle-optimization) · [Cheat Sheet](./performance-notes.md#quick-summary-cheat-sheet) · [Interview Q&A](./performance-notes.md#interview-style-quick-answers)

*Backend/database/infra performance lives in [Section 4](./system-design-production-notes-part1.md#section-4-performance-improvement-questions) and [Section 8](./system-design-production-notes-part2.md#section-8-database-performance-improvement-questions) of the Production notes.*

---

## System Design & Microservices

### [`system-design-basics-notes.md`](./system-design-basics-notes.md)
[1. What is System Design?](./system-design-basics-notes.md#1-what-is-system-design) · [2. Authentication Flow](./system-design-basics-notes.md#2-authentication-flow) · [3. File Upload](./system-design-basics-notes.md#3-file-upload) · [4. Chat Application](./system-design-basics-notes.md#4-chat-application) · [5. URL Shortener](./system-design-basics-notes.md#5-url-shortener) · [6. Notification System](./system-design-basics-notes.md#6-notification-system) · [7. Caching](./system-design-basics-notes.md#7-caching) · [8. Redis Basics](./system-design-basics-notes.md#8-redis-basics) · [Cheat Sheet](./system-design-basics-notes.md#quick-summary-cheat-sheet) · [Interview Tips](./system-design-basics-notes.md#interview-tips-for-system-design-basics)

### [`microservices-interview-notes.md`](./microservices-interview-notes.md)
[1. What are Microservices?](./microservices-interview-notes.md#1-what-are-microservices) · [2. Microservice Architecture](./microservices-interview-notes.md#2-microservice-architecture) · [3. Communication (Sync/Async)](./microservices-interview-notes.md#3-communication-between-services) · [4. API Gateway](./microservices-interview-notes.md#4-api-gateway) · [5. Service Discovery](./microservices-interview-notes.md#5-service-discovery) · [6. Database Per Service](./microservices-interview-notes.md#6-database-per-service) · [7. Distributed Transactions (Saga)](./microservices-interview-notes.md#7-distributed-transactions) · [8. Event Driven Architecture](./microservices-interview-notes.md#8-event-driven-architecture) · [9. Message Broker (Kafka/RabbitMQ)](./microservices-interview-notes.md#9-message-broker) · [10. Caching](./microservices-interview-notes.md#10-caching-redis-in-a-microservices-context) · [11. Authentication](./microservices-interview-notes.md#11-authentication-in-microservices) · [12. Logging](./microservices-interview-notes.md#12-logging) · [13. Monitoring](./microservices-interview-notes.md#13-monitoring) · [14. Circuit Breaker](./microservices-interview-notes.md#14-circuit-breaker) · [15. Retry Pattern](./microservices-interview-notes.md#15-retry-pattern) · [16. Load Balancing](./microservices-interview-notes.md#16-load-balancing) · [17. Docker](./microservices-interview-notes.md#17-containerization-docker) · [18. Kubernetes](./microservices-interview-notes.md#18-kubernetes-basics) · [19. CI/CD](./microservices-interview-notes.md#19-cicd) · [20. Security](./microservices-interview-notes.md#20-security) · [21. Node.js Microservices](./microservices-interview-notes.md#21-nodejs-microservices) · [22. Design Questions to Prepare](./microservices-interview-notes.md#22-design-questions-to-prepare) · [23. Real Project Questions](./microservices-interview-notes.md#23-real-project-questions-frequently-asked) · [Cheat Sheet](./microservices-interview-notes.md#quick-summary-cheat-sheet)

### [`system-design-production-notes-part1.md`](./system-design-production-notes-part1.md)
[Section 1: Design Architecture (25 "Design X" scenarios)](./system-design-production-notes-part1.md#section-1-design-architecture-for-a-new-project) · [Section 2: Architecture Scenario Q&A (30)](./system-design-production-notes-part1.md#section-2-architecture-scenario-based-questions) · [Section 3: Security Implementation (30)](./system-design-production-notes-part1.md#section-3-security-implementation-questions) · [Section 4: Performance Improvement (25)](./system-design-production-notes-part1.md#section-4-performance-improvement-questions)

---

## Production Engineering & Deployment

### [`system-design-production-notes-part2.md`](./system-design-production-notes-part2.md)
[Section 5: Production Issue Investigation (30)](./system-design-production-notes-part2.md#section-5-production-issue-investigation-questions) · [Section 6: Resolving Production Issues (25)](./system-design-production-notes-part2.md#section-6-resolving-production-issues) · [Section 7: Deployment (Docker/K8s/Blue-Green/Canary) (25)](./system-design-production-notes-part2.md#section-7-deployment-related-questions) · [Section 8: Database Performance (25)](./system-design-production-notes-part2.md#section-8-database-performance-improvement-questions) · [Cross-Cutting Themes Reference](./system-design-production-notes-part2.md#overall-quick-reference-cross-cutting-themes)

---

## Data Structures & Coding Practice

### [`coding-practice-notes.md`](./coding-practice-notes.md)
[1. Arrays](./coding-practice-notes.md#1-arrays) · [2. Strings](./coding-practice-notes.md#2-strings) · [3. HashMap](./coding-practice-notes.md#3-hashmap-hash-table-dictionary-object) · [4. Linked List](./coding-practice-notes.md#4-linked-list) · [5. Stack](./coding-practice-notes.md#5-stack) · [6. Queue](./coding-practice-notes.md#6-queue) · [7. Sliding Window](./coding-practice-notes.md#7-sliding-window) · [8. Two Pointer](./coding-practice-notes.md#8-two-pointer) · [Pattern Recognition Cheat Sheet](./coding-practice-notes.md#quick-pattern-recognition-cheat-sheet) · [Complexity Cheat Sheet](./coding-practice-notes.md#complexity-cheat-sheet) · [Problem-Solving Approach](./coding-practice-notes.md#general-interview-problem-solving-approach)

---

## Suggested Reading Order

1. **Fundamentals:** [`JavaScript_ES6_Notes.md`](./JavaScript_ES6_Notes.md) → [`typescript-notes.md`](./typescript-notes.md) → [`NodeJS_Notes.md`](./NodeJS_Notes.md)
2. **Frontend:** [`reactjs-notes.md`](./reactjs-notes.md) → [`state-management-notes.md`](./state-management-notes.md) → [`css-notes.md`](./css-notes.md) → [`nextjs-notes.md`](./nextjs-notes.md) → [`performance-notes.md`](./performance-notes.md)
3. **Backend & APIs:** [`expressjs-notes.md`](./expressjs-notes.md) → [`rest-api-notes.md`](./rest-api-notes.md) → [`graphql-notes.md`](./graphql-notes.md)
4. **Data & Security:** [`database-notes.md`](./database-notes.md) → [`authentication-notes.md`](./authentication-notes.md)
5. **Scale & Architecture:** [`system-design-basics-notes.md`](./system-design-basics-notes.md) → [`microservices-interview-notes.md`](./microservices-interview-notes.md)
6. **Interview Depth Rounds:** [`system-design-production-notes-part1.md`](./system-design-production-notes-part1.md) → [`system-design-production-notes-part2.md`](./system-design-production-notes-part2.md)
7. **Coding Rounds:** [`coding-practice-notes.md`](./coding-practice-notes.md)

---

### Format

Every file follows the same structure: a **simple definition** for each concept, a **detailed explanation** of how/why it works, **code examples**, comparison tables for commonly-confused concepts, and a **Quick Summary Cheat Sheet** plus **Interview Q&A** at the end.
