# Database Complete Notes (MongoDB + SQL)

---

# Part 1: MongoDB

## 1. What is MongoDB?

**Simple definition:** MongoDB is a **NoSQL, document-oriented database** — instead of storing data in rigid tables and rows (like SQL databases), it stores data as flexible, JSON-like documents (called **BSON** internally) grouped into **collections**.

```js
// A MongoDB document — flexible, nested structure
{
  _id: ObjectId("64a1..."),
  name: "Alice",
  email: "alice@mail.com",
  address: { city: "New York", zip: "10001" }, // nested object, no separate table needed
  tags: ["admin", "verified"]                   // array field
}
```
**Why it's popular:** No fixed schema means documents in the same collection can have different fields, making it flexible for rapidly evolving applications. It also scales horizontally well and handles nested/hierarchical data naturally.

---

## 2. Embedding vs Referencing

**Simple definition:** These are the two core strategies for modeling **relationships** between data in MongoDB, since there are no traditional SQL-style joins built into the core query engine.

### Embedding
**Definition:** Storing related data **directly inside** the parent document, as a nested object or array.
```js
// User with embedded address
{
  _id: 1,
  name: "Alice",
  address: { street: "123 Main St", city: "New York" }
}
```
**Best for:** Data that's almost always accessed together, has a "contains" relationship (one-to-few), and doesn't need to be queried independently.
- **Pros:** Single query retrieves everything — fast reads, no joins needed.
- **Cons:** Can lead to large, unwieldy documents if the embedded data grows unbounded (e.g., embedding thousands of comments inside a blog post document); duplicated data if the same info is embedded in multiple places.

### Referencing
**Definition:** Storing just an **ID reference** to a document in another collection, similar to a foreign key in SQL.
```js
// User document
{ _id: 1, name: "Alice" }

// Order document referencing the user by ID
{ _id: 101, userId: 1, product: "Laptop", amount: 999 }
```
**Best for:** Data that's large, frequently changing, needs to be queried independently, or has a one-to-many/many-to-many relationship (e.g., a user with thousands of orders).
- **Pros:** Keeps documents smaller; avoids data duplication.
- **Cons:** Requires an extra query (or a `$lookup`, see below) to fetch related data — more like a traditional join.

### Quick decision guide
| Use Embedding when... | Use Referencing when... |
|---|---|
| Related data is small and bounded | Related data is large or unbounded |
| Data is always read together | Data is queried independently |
| One-to-one or one-to-few relationships | One-to-many or many-to-many relationships |
| Data rarely changes independently | Related data changes frequently on its own |

---

## 3. Lookup (`$lookup`)

**Simple definition:** MongoDB's equivalent of a SQL **JOIN** — used within an aggregation pipeline to pull in related documents from another collection based on a matching field, when you've chosen the **referencing** strategy.

```js
db.orders.aggregate([
  {
    $lookup: {
      from: "users",          // the collection to join with
      localField: "userId",   // field in "orders"
      foreignField: "_id",    // field in "users"
      as: "userDetails",      // output array field name
    },
  },
]);
```
**Result:** Each order document now has a `userDetails` array field containing the matching user document(s).

```json
{
  "_id": 101,
  "userId": 1,
  "product": "Laptop",
  "userDetails": [{ "_id": 1, "name": "Alice" }]
}
```
**Note:** Unlike SQL joins which are core to the relational engine, `$lookup` is a specific aggregation stage — it's powerful but generally more expensive than embedding data directly, so it should be used thoughtfully.

---

## 4. Aggregation

**Simple definition:** The **aggregation pipeline** is MongoDB's framework for processing and transforming data through a sequence of **stages** — filtering, grouping, sorting, reshaping documents — similar to a Unix pipe, where each stage's output feeds into the next.

```js
db.orders.aggregate([
  { $match: { status: "completed" } },              // stage 1: filter documents
  { $group: {                                        // stage 2: group and aggregate
      _id: "$userId",
      totalSpent: { $sum: "$amount" },
      orderCount: { $count: {} }
  }},
  { $sort: { totalSpent: -1 } },                     // stage 3: sort results
  { $limit: 10 },                                     // stage 4: top 10 only
]);
```

### Common aggregation stages
| Stage | Purpose |
|---|---|
| `$match` | Filters documents (like SQL `WHERE`) |
| `$group` | Groups documents and computes aggregates (like SQL `GROUP BY`) |
| `$sort` | Orders results |
| `$project` | Reshapes documents — include/exclude/compute fields |
| `$limit` / `$skip` | Pagination |
| `$lookup` | Joins with another collection |
| `$unwind` | Flattens an array field into separate documents |

```js
// $unwind example — flattening an array
db.orders.aggregate([
  { $unwind: "$items" }, // if an order has items: [item1, item2], produces 2 separate documents
]);
```
**Why it matters:** Aggregation pipelines let you do complex data processing (analytics, reporting, joins, transformations) directly inside the database, rather than pulling raw data into your application and processing it there — much more efficient for large datasets.

---

## 5. Indexes (MongoDB)

**Simple definition:** An index is a **special data structure** that stores a small, sorted subset of a collection's data, allowing MongoDB to find matching documents **without scanning every document** — dramatically speeding up queries at the cost of extra storage and slightly slower writes.

```js
db.users.createIndex({ email: 1 }); // 1 = ascending, -1 = descending

// This query now uses the index instead of scanning the whole collection
db.users.find({ email: "alice@mail.com" });
```
### Types of indexes in MongoDB
- **Single field index:** Index on one field (as above).
- **Compound index:** Index on multiple fields together — order matters, and it supports queries that use a prefix of the indexed fields.
```js
db.orders.createIndex({ userId: 1, status: 1 });
// Efficiently supports queries filtering on: { userId } alone, OR { userId, status } together
// Does NOT efficiently support a query filtering on { status } alone
```
- **Text index:** Enables full-text search on string fields.
- **Unique index:** Prevents duplicate values in that field (e.g., ensuring emails are unique).
```js
db.users.createIndex({ email: 1 }, { unique: true });
```

**Trade-off to remember:** Indexes speed up reads but slow down writes (every insert/update must also update the index) and consume additional storage — don't index every field blindly.

---

## 6. Transactions (MongoDB)

**Simple definition:** A way to group **multiple operations** (across one or more documents/collections) into a single atomic unit — either **all** the operations succeed, or **none** of them do, keeping data consistent.

**Why it matters in a document database:** Since MongoDB documents can contain nested/related data (via embedding), single-document updates are already atomic by default. But when an operation needs to touch **multiple documents or collections together** (e.g., transferring money between two user accounts), you need multi-document transactions.

```js
const session = client.startSession();

try {
  session.startTransaction();

  await accounts.updateOne(
    { _id: senderId },
    { $inc: { balance: -100 } },
    { session }
  );
  await accounts.updateOne(
    { _id: receiverId },
    { $inc: { balance: 100 } },
    { session }
  );

  await session.commitTransaction(); // both succeed together
} catch (error) {
  await session.abortTransaction(); // rollback both if anything fails
} finally {
  session.endSession();
}
```
**Note:** Multi-document transactions in MongoDB are supported (since v4.0 for replica sets, v4.2 for sharded clusters) but come with a performance cost compared to single-document atomic operations — use them only when genuinely needed across multiple documents.

---

# Part 2: SQL

## 7. Joins

**Simple definition:** A `JOIN` combines rows from two or more tables based on a **related column** between them — the foundation of working with normalized, relational data spread across multiple tables.

### Example tables
```sql
-- users table
id | name
1  | Alice
2  | Bob

-- orders table
id | user_id | product
1  | 1        | Laptop
2  | 1        | Mouse
3  | 3        | Keyboard   -- note: user_id 3 doesn't exist in users
```

### `INNER JOIN`
Returns only rows where there's a **match in both tables**.
```sql
SELECT users.name, orders.product
FROM users
INNER JOIN orders ON users.id = orders.user_id;
```
Result: Alice/Laptop, Alice/Mouse (Bob has no orders, so he's excluded; the orphaned order with user_id 3 is excluded too).

### `LEFT JOIN` (LEFT OUTER JOIN)
Returns **all rows from the left table**, plus matching rows from the right table — unmatched right-side columns show `NULL`.
```sql
SELECT users.name, orders.product
FROM users
LEFT JOIN orders ON users.id = orders.user_id;
```
Result: Alice/Laptop, Alice/Mouse, Bob/NULL (Bob is included even without any orders).

### `RIGHT JOIN` (RIGHT OUTER JOIN)
The mirror of `LEFT JOIN` — all rows from the right table, matched rows from the left (rarely used in practice; usually rewritten as a `LEFT JOIN` by swapping table order).

### `FULL OUTER JOIN`
Returns all rows from **both** tables, with `NULL`s where there's no match on either side (not supported directly in MySQL — usually emulated with `UNION` of a `LEFT JOIN` and `RIGHT JOIN`).

### Quick visual summary
| Join Type | Returns |
|---|---|
| `INNER JOIN` | Only matching rows in both tables |
| `LEFT JOIN` | All rows from left table + matches from right (NULL if no match) |
| `RIGHT JOIN` | All rows from right table + matches from left (NULL if no match) |
| `FULL OUTER JOIN` | All rows from both tables, matched where possible |

---

## 8. Group By

**Simple definition:** Groups rows that share the same value in specified column(s) into summary rows — typically used with aggregate functions (`COUNT`, `SUM`, `AVG`, `MAX`, `MIN`) to compute per-group statistics.

```sql
SELECT user_id, COUNT(*) AS order_count, SUM(amount) AS total_spent
FROM orders
GROUP BY user_id;
```
**Key rule:** Every column in the `SELECT` list must either be part of the `GROUP BY` clause or wrapped in an aggregate function — you can't select a raw, non-grouped, non-aggregated column.

---

## 9. Having

**Simple definition:** Filters **groups** created by `GROUP BY` — similar to `WHERE`, but `HAVING` works on aggregated results, since `WHERE` runs before grouping happens and can't reference aggregate values.

```sql
SELECT user_id, SUM(amount) AS total_spent
FROM orders
GROUP BY user_id
HAVING SUM(amount) > 500; -- only groups with total spending over 500
```
| | `WHERE` | `HAVING` |
|---|---|---|
| Filters | Individual rows | Grouped/aggregated results |
| Runs | Before `GROUP BY` | After `GROUP BY` |
| Can use aggregate functions | ❌ No | ✅ Yes |

---

## 10. Order By

**Simple definition:** Sorts the final result set by one or more columns, ascending (`ASC`, default) or descending (`DESC`).

```sql
SELECT name, age FROM users ORDER BY age DESC; -- oldest first

SELECT name, age FROM users ORDER BY age DESC, name ASC; -- multiple sort keys
```
- `ORDER BY` is always applied **last**, after filtering (`WHERE`), grouping (`GROUP BY`/`HAVING`), and before `LIMIT`.

---

## 11. Index (SQL)

**Simple definition:** A database structure (usually a **B-Tree**) that stores a sorted, searchable copy of one or more columns, allowing the database to find rows much faster than scanning the entire table (a "full table scan").

```sql
CREATE INDEX idx_users_email ON users(email);

-- This query can now use the index to jump directly to matching rows
SELECT * FROM users WHERE email = 'alice@mail.com';
```

### Types of index

| Type | Description |
|---|---|
| **Primary key index** | Automatically created on the primary key; uniquely identifies each row |
| **Unique index** | Ensures all values in the column(s) are distinct |
| **Composite (compound) index** | Index across multiple columns together |
| **Clustered index** | Determines the **physical storage order** of table rows — a table can only have ONE clustered index (often the primary key) |
| **Non-clustered index** | A separate structure that stores pointers to the actual row data — a table can have MANY non-clustered indexes |
| **Full-text index** | Optimized for searching within text content |

### Clustered vs Composite Index

**Clustered Index:**
- Physically orders the actual table data on disk according to the indexed column(s).
- Because data can only be physically sorted one way, there can be **only one clustered index per table** — usually the primary key by default.
- Extremely fast for range queries and lookups by the clustered key, since related data is physically stored together.

**Composite (Compound) Index:**
- An index built on **multiple columns**, in a specific order.
```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```
- **Leftmost prefix rule:** This index efficiently supports queries filtering on `user_id` alone, or `user_id AND status` together — but NOT a query filtering on `status` alone, since the index is sorted primarily by `user_id` first.
```sql
-- Uses the index efficiently
SELECT * FROM orders WHERE user_id = 5;
SELECT * FROM orders WHERE user_id = 5 AND status = 'shipped';

-- Does NOT use this index efficiently (status isn't the leftmost column)
SELECT * FROM orders WHERE status = 'shipped';
```

**Trade-off:** Indexes speed up `SELECT` queries but slow down `INSERT`/`UPDATE`/`DELETE` operations (since indexes must be updated too), and consume additional disk space — index thoughtfully based on actual query patterns, not every column.

---

## 12. Transactions (SQL)

**Simple definition:** A transaction groups multiple SQL statements into a single all-or-nothing unit of work — either every statement succeeds and is saved (**committed**), or if anything fails, all changes are undone (**rolled back**), leaving the database as if nothing happened.

```sql
BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 100 WHERE id = 1; -- debit sender
UPDATE accounts SET balance = balance + 100 WHERE id = 2; -- credit receiver

COMMIT; -- both succeed together

-- If something went wrong instead:
-- ROLLBACK; -- undoes both updates
```
**Why it matters:** Without transactions, if the second `UPDATE` failed after the first one succeeded, you'd end up with money deducted from one account but never credited to the other — a corrupted, inconsistent state. Transactions guarantee this can't happen.

---

## 13. ACID

**Simple definition:** ACID is the set of four guarantees that a reliable database transaction system provides, ensuring data stays consistent and correct even in the face of errors, crashes, or concurrent access.

| Property | Meaning |
|---|---|
| **Atomicity** | All operations in a transaction succeed together, or none do — no partial updates |
| **Consistency** | A transaction moves the database from one valid state to another, never violating defined rules/constraints (e.g., foreign keys, unique constraints) |
| **Isolation** | Concurrent transactions don't interfere with each other — each transaction behaves as if it were running alone, even when others run at the same time |
| **Durability** | Once a transaction is committed, the changes are permanently saved, even if the system crashes immediately afterward |

**Example tying it together:** A bank transfer transaction — Atomicity ensures both the debit and credit happen together or not at all; Consistency ensures account balances never violate rules like "balance can't go below zero" (if that's a defined constraint); Isolation ensures another transaction reading the account balance mid-transfer doesn't see a half-completed state; Durability ensures that once you get a "transfer successful" confirmation, that result survives even a server crash a second later.

---

## 14. Normalization

**Simple definition:** Normalization is the process of **organizing tables to reduce data duplication and avoid update anomalies**, by splitting data into multiple related tables based on logical dependencies, connected via foreign keys.

### Example: Unnormalized vs Normalized

**Unnormalized (data duplication problem):**
```
orders table:
id | customer_name | customer_email      | product
1  | Alice          | alice@mail.com      | Laptop
2  | Alice          | alice@mail.com      | Mouse
```
Problem: Alice's email is duplicated. If her email changes, you must update every single order row — miss one, and now you have inconsistent data.

**Normalized:**
```
customers table:              orders table:
id | name  | email             id | customer_id | product
1  | Alice | alice@mail.com    1  | 1            | Laptop
                                2  | 1            | Mouse
```
Now Alice's email exists in exactly one place — update it once, and every order automatically reflects the correct value via the relationship.

### Normal forms (simplified)
| Form | Rule |
|---|---|
| **1NF** | Each column holds atomic (indivisible) values; no repeating groups |
| **2NF** | Must be in 1NF, and every non-key column depends on the *entire* primary key (relevant for composite keys) |
| **3NF** | Must be in 2NF, and no non-key column depends on another non-key column (eliminate transitive dependencies) |

**Trade-off — Normalization vs Denormalization:** Highly normalized schemas minimize duplication and update anomalies but require more `JOIN`s to reassemble related data (potentially slower reads). Denormalization (intentionally duplicating some data) can improve read performance at the cost of more complex updates — a common trade-off decision in real-world schema design, especially for read-heavy analytics systems.

---

## 15. Window Functions

**Simple definition:** Window functions perform calculations **across a set of related rows** (a "window") **without collapsing them into a single row** like `GROUP BY` does — each row keeps its individual identity while also getting access to aggregate/ranking information about its group.

```sql
SELECT
  name,
  department,
  salary,
  RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank,
  AVG(salary) OVER (PARTITION BY department) AS dept_avg_salary
FROM employees;
```
- `PARTITION BY department` — divides rows into groups (windows) by department, similar to `GROUP BY`, but doesn't collapse rows.
- `ORDER BY salary DESC` (inside `OVER`) — determines the order used for ranking within each partition.
- Unlike `GROUP BY`, you still get **one output row per input row**, each annotated with its rank/average — perfect for "show me each employee alongside their rank within their department."

### Common window functions
| Function | Purpose |
|---|---|
| `ROW_NUMBER()` | Assigns a unique sequential number to each row within its partition |
| `RANK()` | Assigns a rank, with gaps after ties (1, 2, 2, 4) |
| `DENSE_RANK()` | Assigns a rank, without gaps after ties (1, 2, 2, 3) |
| `LAG()` / `LEAD()` | Accesses a previous/next row's value within the partition |
| `SUM()`/`AVG()`/`COUNT() OVER (...)` | Running totals, moving averages, group aggregates without collapsing rows |

```sql
-- Running total example
SELECT
  order_date,
  amount,
  SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM orders;
```

---

## 16. NOLOCK

**Simple definition:** `NOLOCK` (a SQL Server-specific query hint) tells the database to read data **without acquiring shared locks**, allowing the query to read rows even if they're currently being modified by another uncommitted transaction — essentially reading "dirty" (possibly-uncommitted, possibly-about-to-be-rolled-back) data.

```sql
SELECT * FROM orders WITH (NOLOCK) WHERE status = 'pending';
```

### Why/when it's used
- **Pro:** Improves read performance and reduces blocking in high-concurrency systems, since readers don't wait for writers to finish and vice versa.
- **Con — the real risk:** You might read **uncommitted data** that later gets rolled back (a "dirty read"), leading to inconsistent or incorrect results — generally unsafe for financial data, inventory counts, or anything requiring accuracy.

**Best practice:** Use `NOLOCK` sparingly and only for scenarios where slightly stale or approximate data is acceptable (e.g., dashboards, reporting queries) — never for critical transactional reads (e.g., checking account balance before a withdrawal).

---

## 17. SQL Functions

**Simple definition:** Built-in or user-defined routines that take input(s) and return a single value, used to transform, calculate, or format data within a query.

### Categories

**Aggregate functions** — operate across multiple rows, return one value:
```sql
SELECT COUNT(*), SUM(amount), AVG(amount), MAX(amount), MIN(amount) FROM orders;
```

**Scalar functions** — operate on a single value per row:
```sql
SELECT UPPER(name), LEN(name), ROUND(price, 2), CONCAT(first_name, ' ', last_name)
FROM users;
```

**Date functions:**
```sql
SELECT GETDATE(), DATEDIFF(day, order_date, GETDATE()) AS days_ago FROM orders;
```

**User-Defined Functions (UDFs)** — custom, reusable logic you define yourself:
```sql
CREATE FUNCTION dbo.GetFullName (@first VARCHAR(50), @last VARCHAR(50))
RETURNS VARCHAR(101)
AS
BEGIN
  RETURN @first + ' ' + @last;
END;

SELECT dbo.GetFullName(first_name, last_name) FROM users;
```

---

## 18. Stored Procedure

**Simple definition:** A **precompiled, named block of SQL code** stored in the database itself, which can accept parameters and be called repeatedly — bundling multiple statements/logic into a single reusable, callable unit.

```sql
CREATE PROCEDURE usp_GetUserOrders
  @UserId INT
AS
BEGIN
  SELECT o.id, o.product, o.amount
  FROM orders o
  WHERE o.user_id = @UserId
  ORDER BY o.order_date DESC;
END;

-- Calling it:
EXEC usp_GetUserOrders @UserId = 5;
```

### Why use stored procedures?
- **Performance:** Precompiled execution plans can be faster than repeatedly parsing/planning the same raw SQL from the application.
- **Reusability:** Multiple applications/services can call the same procedure instead of duplicating SQL logic in application code.
- **Security:** You can grant permission to execute a procedure without granting direct access to the underlying tables, reducing the attack surface (e.g., protecting against certain SQL injection patterns since parameters are handled safely).
- **Encapsulation:** Complex, multi-step business logic (e.g., "process an order": check inventory, deduct stock, create an order record, log the transaction) can live in one place in the database rather than scattered across application code.

**Trade-off to mention:** Stored procedures can make logic harder to version-control and test compared to application-layer code, and can tie your business logic tightly to a specific database vendor — many modern teams prefer keeping business logic in the application layer and using stored procedures more sparingly (e.g., for performance-critical or tightly-controlled data operations).

---

## Interview Questions & Answers

### Q1. `LEFT JOIN` explained
`LEFT JOIN` returns **every row from the left (first) table**, along with matching data from the right table where it exists — if there's no match, the right table's columns simply show `NULL` instead of excluding the row entirely.
```sql
SELECT u.name, o.product
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
-- Every user appears at least once, even users with zero orders (product = NULL for them)
```
**Use case:** "Show me all customers, and their orders if they have any" — you want to see everyone, not just those with a match.

### Q2. `INNER JOIN` explained
`INNER JOIN` returns **only rows where there's a match in both tables** — anything without a corresponding match on either side is excluded from the results entirely.
```sql
SELECT u.name, o.product
FROM users u
INNER JOIN orders o ON u.id = o.user_id;
-- Only users who have placed at least one order appear in the results
```
**Use case:** "Show me only customers who have actually placed an order" — you only care about matched pairs.

### Q3. What is an Index, and how does it improve performance?
An index is a separate, sorted data structure (typically a B-Tree) built on one or more columns, allowing the database engine to locate matching rows via a fast lookup (similar to using a book's index instead of reading every page) instead of scanning the entire table row by row (a full table scan). This turns an O(n) search into something closer to O(log n).

**The trade-off to always mention:** Indexes aren't free — every `INSERT`/`UPDATE`/`DELETE` must also update the relevant indexes, adding write overhead, and indexes consume additional disk space. The right approach is to index columns that are frequently used in `WHERE`, `JOIN`, and `ORDER BY` clauses — not every column indiscriminately.

### Q4. How would you approach query optimization / explaining a query?

**Step 1: Use `EXPLAIN` (or `EXPLAIN ANALYZE`) to see the execution plan.**
```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 5 AND status = 'shipped';
```
This reveals how the database engine intends to execute the query — whether it's doing a full table scan, using an index, the estimated number of rows examined, join strategies used, and the estimated cost.

**Step 2: Look for red flags in the plan:**
- **Full table scans** on large tables (instead of an index seek/lookup) — often means a missing or unused index.
- **High estimated row counts** flowing into later stages of the plan — suggests filtering isn't happening early enough.
- **Type mismatches** in `WHERE` clauses (e.g., comparing a string column to a number) — this can silently prevent the database from using an existing index.
- **Nested loop joins on large tables** — can be far slower than hash/merge joins for big datasets, depending on the engine.

**Step 3: Common optimization techniques:**
- Add appropriate indexes on columns used in `WHERE`, `JOIN`, and `ORDER BY` — paying attention to composite index column order (leftmost prefix rule).
- Avoid `SELECT *` — only select the columns you actually need, reducing data transfer and potentially allowing "covering indexes" (where the index alone contains all needed columns, avoiding a lookup into the actual table).
- Rewrite subqueries as `JOIN`s where possible, since some database engines optimize joins more efficiently than nested subqueries.
- Ensure `WHERE` clause conditions aren't wrapped in functions on the indexed column (e.g., `WHERE YEAR(order_date) = 2024` prevents index usage — rewrite as a date range instead: `WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01'`).
- Consider denormalization or materialized views for expensive, frequently-run aggregate queries in read-heavy systems.

**One-line summary for interviews:** "I'd run `EXPLAIN` to see the actual execution plan, look for full table scans or unused indexes, check if the `WHERE`/`JOIN` columns are properly indexed, and verify the query isn't preventing index usage through functions or type mismatches on those columns."

---

## Quick Summary Cheat Sheet

### MongoDB
| Concept | One-liner |
|---|---|
| Embedding | Nesting related data directly inside a document |
| Referencing | Storing an ID pointer to a document in another collection |
| `$lookup` | MongoDB's join, used within aggregation pipelines |
| Aggregation | Multi-stage pipeline for filtering, grouping, transforming data |
| Indexes | Speed up queries by avoiding full collection scans |
| Transactions | All-or-nothing operations across multiple documents/collections |

### SQL
| Concept | One-liner |
|---|---|
| Joins | Combine rows from multiple tables based on a related column |
| Group By | Groups rows for aggregate calculations |
| Having | Filters grouped/aggregated results (unlike WHERE, which filters raw rows) |
| Order By | Sorts the final result set |
| Index | Speeds up lookups via a sorted, searchable structure |
| Clustered Index | Physically orders table data; only one per table |
| Composite Index | Index across multiple columns, following the leftmost prefix rule |
| Transactions | All-or-nothing group of SQL statements |
| ACID | Atomicity, Consistency, Isolation, Durability — reliability guarantees |
| Normalization | Organizing tables to reduce duplication and update anomalies |
| Window Functions | Per-row calculations across a group, without collapsing rows |
| NOLOCK | Reads data without locks — faster, but risks dirty reads |
| SQL Functions | Aggregate, scalar, date, and user-defined functions for data transformation |
| Stored Procedure | Precompiled, reusable, callable block of SQL logic |
