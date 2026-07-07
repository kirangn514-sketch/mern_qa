# GraphQL Complete Notes

---

## 1. What is GraphQL?

**Simple definition:** GraphQL is a **query language for APIs** (and a runtime for executing those queries) that lets clients ask for **exactly the data they need** — no more, no less — in a single request, instead of being locked into whatever fixed shape a REST endpoint decides to return.

**Why it exists:** With traditional REST APIs, each endpoint returns a fixed, predetermined data structure. If a mobile app only needs a user's `name` and `avatar`, but the `/users/:id` endpoint always returns the full user object (address, order history, preferences, etc.), the client downloads unnecessary data — this is called **over-fetching**. And if the client needs data from multiple related resources (a user's profile + their recent orders), it often has to make several separate REST calls — this is called **under-fetching**. GraphQL solves both problems by letting the client specify precisely what fields it wants, across related data, in one request.

```graphql
# The client asks for exactly this — nothing more
query {
  user(id: "1") {
    name
    avatar
  }
}
```
```json
// The server returns exactly that shape — no extra fields
{
  "data": {
    "user": { "name": "Alice", "avatar": "https://..." }
  }
}
```

---

## 2. Schema

**Simple definition:** The schema is the **contract/blueprint** of your GraphQL API — it defines every type of data available, what fields each type has, and what queries/mutations clients are allowed to perform. It's written in GraphQL's own type language (**SDL** — Schema Definition Language).

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  age: Int
  posts: [Post!]!  # a user has a list of posts (a relationship to another type)
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!    # each post belongs to a user
}

type Query {
  users: [User!]!
  user(id: ID!): User
}

type Mutation {
  createUser(name: String!, email: String!): User!
}
```

### Key schema concepts
- **Scalar types:** Built-in primitive types — `String`, `Int`, `Float`, `Boolean`, `ID`.
- **`!` (non-null):** Marks a field as **required** — it can never return `null`. Without it, the field is optional/nullable by default.
- **`[Type!]!`:** A non-null list of non-null items — e.g., `[Post!]!` means "a list that's always present, containing only non-null `Post` objects."
- **`Query` type:** Defines every **read** operation clients can perform — the entry point for fetching data.
- **`Mutation` type:** Defines every operation that **changes** data (create, update, delete).
- **Custom object types:** Define the shape of your actual application data (`User`, `Post`, etc.), including relationships between them.

**Why the schema matters so much:** It acts as a single source of truth and **strongly typed contract** between frontend and backend — tools can auto-generate documentation, validate queries before they even run, and provide autocomplete in editors, because the entire structure of the API is explicitly declared upfront.

---

## 3. Query

**Simple definition:** A `Query` is how a client **reads/fetches data** from a GraphQL API — equivalent to a `GET` request in REST, but with the client specifying exactly which fields it wants, including nested/related data, all in one request.

```graphql
# Client's query
query {
  user(id: "1") {
    name
    email
    posts {
      title
    }
  }
}
```
```json
// Server's response — matches the exact shape requested
{
  "data": {
    "user": {
      "name": "Alice",
      "email": "alice@mail.com",
      "posts": [
        { "title": "My First Post" },
        { "title": "GraphQL is Great" }
      ]
    }
  }
}
```
Notice: this single query fetched a user **and** their related posts (something that would typically require two separate REST calls: `GET /users/1` and `GET /users/1/posts`).

### Queries with variables (avoiding hardcoded values)
```graphql
query GetUser($userId: ID!) {
  user(id: $userId) {
    name
    email
  }
}
```
```json
// Variables sent alongside the query
{ "userId": "1" }
```
- Using variables instead of hardcoding values directly in the query string is the standard practice — similar to using parameterized queries in SQL, it keeps queries reusable and reduces risk of injection-style issues.

---

## 4. Mutation

**Simple definition:** A `Mutation` is how a client **modifies data** — creating, updating, or deleting records — equivalent to `POST`/`PUT`/`PATCH`/`DELETE` in REST. Mutations look and behave like queries syntactically, but by convention they're used specifically for operations that change server-side data.

```graphql
mutation {
  createUser(name: "Bob", email: "bob@mail.com") {
    id
    name
    email
  }
}
```
```json
{
  "data": {
    "createUser": { "id": "2", "name": "Bob", "email": "bob@mail.com" }
  }
}
```
- Just like queries, mutations let you specify exactly which fields you want returned **after** the operation completes — e.g., getting back the newly created user's generated `id` immediately, without a separate follow-up request.

### Mutation with variables
```graphql
mutation CreateUser($name: String!, $email: String!) {
  createUser(name: $name, email: $email) {
    id
    name
  }
}
```

### Important note on execution order
Unlike queries (whose fields can be resolved in parallel since reading data has no side effects), **multiple top-level mutations in a single request are executed serially, one after another** — this matters because mutations have side effects, and their order can affect the outcome.

---

## 5. Resolver

**Simple definition:** A resolver is a **function that actually fetches or computes the data** for a specific field in the schema. The schema defines *what* data is available and its shape; resolvers define *how* to actually get that data (from a database, another API, a computed value, etc.).

```js
const resolvers = {
  Query: {
    users: async () => {
      return await db.users.findMany(); // fetch all users from the database
    },
    user: async (parent, args) => {
      return await db.users.findById(args.id); // args.id comes from the query's variables/arguments
    },
  },
  Mutation: {
    createUser: async (parent, args) => {
      return await db.users.create({ name: args.name, email: args.email });
    },
  },
  User: {
    // Resolver for the "posts" field ON a User — runs only if the client actually asked for posts
    posts: async (parent) => {
      return await db.posts.findByUserId(parent.id); // parent = the User object being resolved
    },
  },
};
```

### Resolver function signature
Every resolver receives four standard arguments:
```js
fieldName: (parent, args, context, info) => { ... }
```
| Argument | Meaning |
|---|---|
| `parent` | The result returned by the resolver of the **parent field** (e.g., the `User` object, when resolving its `posts` field) |
| `args` | The arguments passed in the query (e.g., `{ id: "1" }` from `user(id: "1")`) |
| `context` | Shared data available to all resolvers in a request — commonly used for the authenticated user, database connections, or dataloaders |
| `info` | Metadata about the query execution itself (field names, path, schema) — rarely used directly in typical applications |

**Key insight:** GraphQL only calls the resolvers for the **fields the client actually requested** — if a query doesn't ask for `posts`, the `posts` resolver never runs at all, avoiding unnecessary work (this is part of what makes GraphQL efficient compared to REST endpoints that always compute and return everything, whether or not it's used).

### The N+1 problem (a well-known resolver pitfall)
If resolving a list of users' posts naively calls a separate database query for *each* user's posts individually, fetching 100 users results in 1 query for users + 100 separate queries for each user's posts (101 total queries!). This is commonly solved using a **DataLoader** — a utility that batches and caches these lookups into a single, efficient query per request.

---

## 6. Apollo Server

**Simple definition:** Apollo Server is one of the most popular **GraphQL server implementations** for Node.js — it takes your schema and resolvers and turns them into a fully functioning GraphQL API, handling query parsing, validation, and execution for you.

### Basic setup
```js
const { ApolloServer, gql } = require("apollo-server");

// 1. Define the schema
const typeDefs = gql`
  type User {
    id: ID!
    name: String!
    email: String!
  }

  type Query {
    users: [User!]!
    user(id: ID!): User
  }

  type Mutation {
    createUser(name: String!, email: String!): User!
  }
`;

// 2. Define the resolvers
const resolvers = {
  Query: {
    users: () => db.users.findMany(),
    user: (_, { id }) => db.users.findById(id),
  },
  Mutation: {
    createUser: (_, { name, email }) => db.users.create({ name, email }),
  },
};

// 3. Create and start the server
const server = new ApolloServer({ typeDefs, resolvers });

server.listen().then(({ url }) => {
  console.log(`Server ready at ${url}`);
});
```

### Integrating with Express (common in real projects)
```js
const express = require("express");
const { ApolloServer } = require("@apollo/server");
const { expressMiddleware } = require("@apollo/server/express4");

const app = express();
const server = new ApolloServer({ typeDefs, resolvers });

await server.start();
app.use("/graphql", express.json(), expressMiddleware(server, {
  context: async ({ req }) => ({ user: req.user }), // pass request context to resolvers (e.g., authenticated user)
}));

app.listen(4000);
```

### Why Apollo Server is popular
- **Built-in tooling:** Ships with GraphQL Playground/Apollo Sandbox — an interactive UI for exploring the schema and testing queries directly in the browser.
- **Ecosystem:** Pairs naturally with **Apollo Client** on the frontend, providing built-in caching, loading states, and query management in React apps.
- **Middleware/plugin support:** Easy to add authentication, logging, error handling, and performance tracing.
- **Widely adopted:** Large community, extensive documentation, and production-proven at scale.

---

## 7. REST vs GraphQL

### Core conceptual difference
- **REST:** Multiple fixed **endpoints**, each returning a predetermined data shape (`GET /users`, `GET /users/1/posts`).
- **GraphQL:** A **single endpoint** (typically `/graphql`), where the client specifies exactly what data it wants via the query itself.

### Detailed comparison

| Aspect | REST | GraphQL |
|---|---|---|
| Endpoints | Multiple (one per resource/action) | Single endpoint for all operations |
| Data shape | Fixed by the server per endpoint | Defined by the client, per request |
| Over-fetching | Common — endpoints often return more fields than needed | Avoided — client requests only needed fields |
| Under-fetching | Common — related data often needs multiple round-trips | Avoided — related/nested data fetched in one request |
| Versioning | Typically via URL (`/v1/`, `/v2/`) | Usually avoided — fields can be deprecated and added without versioning the whole API |
| Caching | Simple, leverages standard HTTP caching (`GET` + `Cache-Control`) | More complex — typically needs client-side libraries (Apollo Client, Relay) for effective caching, since everything goes through `POST` to one endpoint |
| File uploads | Native support via `multipart/form-data` | Requires extra libraries/conventions (not natively part of the spec) |
| Learning curve | Simpler, widely understood, matches HTTP semantics directly | Steeper — requires learning schema design, resolvers, and a new query syntax |
| Error handling | Uses HTTP status codes (`404`, `500`, etc.) | Typically returns `200 OK` even on errors, with error details inside the response body's `errors` field |
| Tooling | Mature, universal (Postman, curl, browser-native) | Strong tooling too (GraphQL Playground, Apollo tools), but newer/more specialized |

### Example: same task, both approaches

**REST — fetching a user and their recent posts requires 2 calls:**
```
GET /users/1        → { id, name, email, address, ... } (full object, possibly more than needed)
GET /users/1/posts  → [{ id, title, content, ... }, ...]
```

**GraphQL — one call, exact fields:**
```graphql
query {
  user(id: "1") {
    name
    posts {
      title
    }
  }
}
```

### When to choose REST
- Simple CRUD APIs with straightforward, well-understood resource shapes.
- Need to leverage standard HTTP caching easily (CDNs, browser cache).
- Public APIs where broad tooling compatibility and simplicity matter (e.g., third-party integrations, webhooks).
- File upload-heavy APIs.

### When to choose GraphQL
- Complex, deeply nested/related data (e.g., a social media feed combining posts, comments, likes, and author info in one view).
- Multiple client types with very different data needs (mobile app needing minimal data vs. a web dashboard needing rich data) — one flexible API instead of multiple tailored REST endpoints.
- Rapidly evolving frontend requirements, where adding new fields shouldn't require backend versioning.
- Reducing the number of network round-trips matters significantly (e.g., mobile apps on slow networks).

**Balanced takeaway:** GraphQL isn't strictly "better" than REST — it solves specific problems (over/under-fetching, multiple round-trips) at the cost of added complexity (schema design, resolver N+1 issues, harder HTTP-level caching). Many real-world systems use both: REST for simple, cacheable, public-facing endpoints, and GraphQL for complex, client-driven data aggregation needs.

---

## Quick Summary Cheat Sheet

| Concept | One-liner |
|---|---|
| Schema | The typed contract defining all available data and operations |
| Query | Read operation — client specifies exactly which fields to fetch |
| Mutation | Write operation — create/update/delete data, executed serially |
| Resolver | Function that actually fetches/computes the data for a schema field |
| Apollo Server | Popular Node.js library that turns a schema + resolvers into a working GraphQL API |
| REST vs GraphQL | REST = multiple fixed endpoints; GraphQL = one flexible, client-driven endpoint |

## Interview-Style Quick Answers

**Q: What problem does GraphQL solve that REST doesn't handle well?**
It eliminates over-fetching (getting more data than needed) and under-fetching (needing multiple requests to gather related data), by letting the client specify exactly the fields and relationships it needs in a single request.

**Q: Why do resolvers matter, and what's the N+1 problem?**
Resolvers are the functions that actually fetch data for each schema field — GraphQL only runs the resolvers for fields the client requested. The N+1 problem happens when resolving a list of items each triggers a separate, naive database call for related data (e.g., 1 query for users + 1 query per user's posts) — commonly solved with a batching tool like DataLoader to combine those into efficient batched queries.

**Q: Does GraphQL replace REST entirely?**
No — they solve overlapping but distinct problems. GraphQL is well-suited for flexible, client-driven data-fetching in complex applications; REST remains simpler and better-suited for straightforward CRUD, standard HTTP caching, and public APIs. Many production systems use both together, depending on the use case.
