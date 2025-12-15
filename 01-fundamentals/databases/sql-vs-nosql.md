# SQL vs NoSQL Databases

## Overview

One of the most fundamental decisions in system design is choosing between SQL (relational) and NoSQL (non-relational) databases. This choice impacts scalability, consistency, query capabilities, and development complexity.

**Key Question:** "When should I use SQL vs NoSQL?"

**Short Answer:**
- **SQL** when you need ACID guarantees, complex queries, and structured data
- **NoSQL** when you need horizontal scalability, flexible schema, and high write throughput

---

## SQL Databases (Relational)

### What is SQL?

SQL (Structured Query Language) databases store data in **tables with fixed schemas**. Data is organized in rows and columns with relationships between tables.

**Popular SQL Databases:**
- **PostgreSQL** - Open-source, feature-rich, ACID compliant
- **MySQL** - Fast, reliable, widely used (owned by Oracle)
- **Oracle Database** - Enterprise-grade, expensive
- **Microsoft SQL Server** - Windows ecosystem integration
- **SQLite** - Embedded, serverless, file-based

### Key Characteristics

#### 1. ACID Compliance

**ACID** guarantees strong consistency:

- **Atomicity:** Transactions are all-or-nothing
- **Consistency:** Data always moves from one valid state to another
- **Isolation:** Concurrent transactions don't interfere
- **Durability:** Committed data persists even after crashes

**Example:**
```sql
-- Bank transfer (atomic transaction)
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;
COMMIT;

-- If anything fails, entire transaction rolls back
-- Money never disappears or duplicates
```

#### 2. Fixed Schema

Tables have predefined structure:

```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
```

**Pros:**
- ✅ Data integrity enforced at database level
- ✅ Clear structure and documentation
- ✅ Prevents invalid data insertion

**Cons:**
- ❌ Schema changes can be expensive (ALTER TABLE)
- ❌ All records must fit the schema
- ❌ Less flexible for rapidly evolving data

#### 3. Powerful Query Language

SQL provides rich querying capabilities:

```sql
-- Complex JOIN query
SELECT u.username, COUNT(p.post_id) as post_count
FROM users u
LEFT JOIN posts p ON u.user_id = p.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.user_id
HAVING COUNT(p.post_id) > 10
ORDER BY post_count DESC;
```

**Capabilities:**
- ✅ JOINs (combine data from multiple tables)
- ✅ Aggregations (SUM, COUNT, AVG)
- ✅ Subqueries and CTEs (Common Table Expressions)
- ✅ Transactions spanning multiple tables

#### 4. Normalization

SQL encourages **normalized** data (avoiding duplication):

```sql
-- Normalized design (good for SQL)
users table:
  user_id | username | email

posts table:
  post_id | user_id | content | created_at

comments table:
  comment_id | post_id | user_id | content
```

**Benefits:**
- ✅ No data duplication
- ✅ Easy to update (change email in one place)
- ✅ Data consistency

**Trade-off:**
- ❌ Requires JOINs to fetch related data
- ❌ JOINs can be slow for large datasets

---

## NoSQL Databases (Non-Relational)

### What is NoSQL?

NoSQL databases store data in flexible formats: **documents, key-value pairs, wide columns, or graphs**. Designed for horizontal scalability and high throughput.

**Types of NoSQL Databases:**

| Type | Examples | Use Case |
|------|----------|----------|
| **Document** | MongoDB, CouchDB | Flexible schema, nested data |
| **Key-Value** | Redis, DynamoDB | Simple lookups, caching |
| **Wide-Column** | Cassandra, HBase | Time-series, high writes |
| **Graph** | Neo4j, Amazon Neptune | Social networks, recommendations |

### Key Characteristics

#### 1. BASE Properties

NoSQL often follows **BASE** instead of ACID:

- **Basically Available:** System appears to work most of the time
- **Soft state:** State may change without input (eventual consistency)
- **Eventual consistency:** System becomes consistent over time

**Example:**
```javascript
// Write to Cassandra (immediate success)
INSERT INTO posts (post_id, user_id, content)
VALUES (uuid(), 123, 'Hello World');

// Read immediately after might not see the post yet
// (replication lag)
// But eventually (milliseconds later), it will appear
```

#### 2. Flexible Schema

Documents can have different structures:

```javascript
// MongoDB example
// User 1 (has address)
{
  "_id": "user_1",
  "name": "Alice",
  "email": "alice@example.com",
  "address": {
    "city": "New York",
    "zip": "10001"
  }
}

// User 2 (no address - perfectly fine!)
{
  "_id": "user_2",
  "name": "Bob",
  "email": "bob@example.com",
  "interests": ["coding", "music"]
}
```

**Pros:**
- ✅ No schema migrations needed
- ✅ Can add fields anytime
- ✅ Different records can have different fields

**Cons:**
- ❌ No schema enforcement (application must validate)
- ❌ Can lead to inconsistent data
- ❌ Harder to query across varying structures

#### 3. Denormalization

NoSQL encourages **denormalized** data (duplicate for speed):

```javascript
// Denormalized (good for NoSQL)
// Post document includes user info
{
  "post_id": "post_1",
  "content": "Hello World",
  "user": {
    "user_id": "user_123",
    "username": "alice",
    "avatar_url": "https://..."
  },
  "comments": [
    {
      "comment_id": "c1",
      "user": {"user_id": "user_456", "username": "bob"},
      "content": "Great post!"
    }
  ]
}

// Everything in one document - fast read, no JOINs
```

**Benefits:**
- ✅ Fast reads (no JOINs needed)
- ✅ Single query to get all data
- ✅ Better for read-heavy workloads

**Trade-offs:**
- ❌ Data duplication (more storage)
- ❌ Updates must happen in multiple places
- ❌ Risk of inconsistency

#### 4. Horizontal Scalability

NoSQL databases are designed to scale horizontally:

```
SQL (Vertical Scaling):
  Single powerful server (16 cores, 128GB RAM)
  Expensive, hardware limits

NoSQL (Horizontal Scaling):
  10 commodity servers (4 cores, 16GB RAM each)
  Cheaper, unlimited scaling

Cassandra cluster:
  Node 1 | Node 2 | Node 3 | ... | Node N
  Data sharded automatically across nodes
```

**Benefits:**
- ✅ Add more servers to increase capacity
- ✅ No single point of failure
- ✅ Cost-effective at large scale

---

## Comparison Table

| Aspect | SQL | NoSQL |
|--------|-----|-------|
| **Data Model** | Tables with rows/columns | Documents, key-value, wide-column, graph |
| **Schema** | Fixed, predefined | Flexible, schema-less |
| **Consistency** | ACID (strong) | BASE (eventual) |
| **Scalability** | Vertical (scale up) | Horizontal (scale out) |
| **Query Language** | SQL (standardized) | Database-specific (MongoDB Query Language, CQL) |
| **Transactions** | Multi-row ACID transactions | Limited (often single-document) |
| **JOINs** | Powerful JOINs | Limited or no JOINs |
| **Best For** | Complex queries, transactions | High throughput, flexible data |
| **Examples** | PostgreSQL, MySQL | MongoDB, Cassandra, DynamoDB |

---

## When to Use SQL

### Use Cases

#### 1. Financial Applications (Banking, Payments)
**Why:** ACID transactions critical for money transfers

```sql
-- Transfer must be atomic
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
  INSERT INTO transactions (from, to, amount) VALUES (1, 2, 100);
COMMIT;
```

**Examples:** Banking apps, payment processors (Stripe uses PostgreSQL)

#### 2. E-commerce (Inventory, Orders)
**Why:** Need accurate stock counts, order consistency

```sql
-- Must ensure product in stock before order
BEGIN TRANSACTION;
  UPDATE products SET stock = stock - 1 WHERE id = 123 AND stock > 0;
  INSERT INTO orders (user_id, product_id, quantity) VALUES (456, 123, 1);
COMMIT;
```

**Examples:** Amazon, Shopify (use PostgreSQL/MySQL for core transactions)

#### 3. Enterprise Applications (ERP, CRM)
**Why:** Complex relationships, reporting, data integrity

```sql
-- Complex reporting query
SELECT
    c.company_name,
    COUNT(DISTINCT o.order_id) as total_orders,
    SUM(oi.quantity * oi.price) as revenue
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
WHERE o.created_at > '2024-01-01'
GROUP BY c.company_name
HAVING revenue > 10000;
```

#### 4. Applications Requiring Complex Queries
**Why:** SQL excels at JOINs, aggregations, filtering

### Advantages of SQL

✅ **Strong consistency** - No stale data
✅ **Data integrity** - Foreign keys, constraints
✅ **Mature ecosystem** - Tools, ORMs, expertise
✅ **Powerful queries** - Complex JOINs and aggregations
✅ **Transactions** - ACID guarantees

### Limitations of SQL

❌ **Vertical scaling limits** - Can't infinitely scale up
❌ **Schema changes** - ALTER TABLE can be slow
❌ **Write throughput** - Limited by single primary
❌ **Sharding complexity** - Not designed for horizontal scaling

---

## When to Use NoSQL

### Use Cases

#### 1. Social Media Feeds (Facebook, Twitter)
**Why:** High write volume, denormalized data, eventual consistency OK

```javascript
// MongoDB - Store entire feed item
{
  "_id": "post_123",
  "user": {
    "id": "user_456",
    "name": "Alice",
    "avatar": "url"
  },
  "content": "Hello World!",
  "likes": 42,
  "comments": [...],  // Embedded
  "created_at": ISODate("2024-01-15")
}

// Single query gets everything - fast!
```

**Examples:** Facebook (uses Cassandra), Twitter (uses Manhattan)

#### 2. Real-Time Analytics (Logging, Metrics)
**Why:** Massive write throughput, time-series data

```sql
-- Cassandra example (wide-column)
CREATE TABLE metrics (
    metric_name text,
    timestamp timestamp,
    value double,
    PRIMARY KEY (metric_name, timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);

-- Optimized for time-series inserts and range queries
```

**Examples:** Uber (uses Cassandra for trips), Netflix (uses Cassandra for viewing data)

#### 3. Caching Layer (Sessions, Temporary Data)
**Why:** Fast key-value lookups, in-memory speed

```python
# Redis example
redis.set("user:session:abc123", json.dumps(user_data), ex=3600)
session = redis.get("user:session:abc123")
```

**Examples:** Twitter (uses Redis), Pinterest (uses Redis)

#### 4. Content Management (Blogs, Catalogs)
**Why:** Flexible schema for different content types

```javascript
// MongoDB - Different product types
// Book
{
  "_id": "prod_1",
  "type": "book",
  "title": "Design Patterns",
  "author": "GoF",
  "pages": 395
}

// T-Shirt
{
  "_id": "prod_2",
  "type": "tshirt",
  "size": "M",
  "color": "blue",
  "material": "cotton"
}
```

### Advantages of NoSQL

✅ **Horizontal scalability** - Add nodes, increase capacity
✅ **Flexible schema** - Evolve data model easily
✅ **High write throughput** - Designed for massive writes
✅ **Low latency** - Optimized for speed
✅ **Handles big data** - Petabytes scale

### Limitations of NoSQL

❌ **Eventual consistency** - Stale reads possible
❌ **Limited transactions** - Usually single-document
❌ **No JOINs** - Must denormalize or multiple queries
❌ **Learning curve** - Each database has own query language
❌ **Data duplication** - Denormalization increases storage

---

## Real-World Examples

### Companies Using SQL

| Company | Database | Use Case |
|---------|----------|----------|
| **Stripe** | PostgreSQL | Payment processing (ACID critical) |
| **Shopify** | MySQL | E-commerce transactions |
| **GitHub** | MySQL | Code repositories metadata |
| **Discord** | PostgreSQL (+ Cassandra) | User accounts, servers |

### Companies Using NoSQL

| Company | Database | Use Case |
|---------|----------|----------|
| **Facebook** | Cassandra | Messages, inbox |
| **Netflix** | Cassandra | Viewing history, recommendations |
| **Uber** | Cassandra | Trip data, locations |
| **Twitter** | Manhattan (custom) | Tweets, timelines |
| **LinkedIn** | Espresso (custom) | Social graph |
| **Amazon** | DynamoDB | Shopping cart, session data |

### Hybrid Approach (Most Common!)

Most large companies use **both**:

**Instagram Example:**
- **PostgreSQL** - User accounts, authentication (need ACID)
- **Cassandra** - Photos metadata, feed (need scale)
- **Redis** - Caching, sessions (need speed)

**Uber Example:**
- **PostgreSQL** - User profiles, payments (need ACID)
- **Cassandra** - Trip data, locations (need scale)
- **Redis** - Geo-location cache (need speed)

---

## Decision Matrix

### Choose SQL When:

✅ You need **ACID transactions**
✅ Your data has **complex relationships**
✅ You need **complex queries** (JOINs, aggregations)
✅ Your data is **structured and predictable**
✅ You value **data integrity** over flexibility
✅ **Consistency** is more important than availability
✅ Your data fits on a **single powerful server** (or can be replicated)

### Choose NoSQL When:

✅ You need **massive scalability** (billions of records)
✅ You have **flexible/evolving schema**
✅ You need **high write throughput**
✅ Your data is **denormalized**
✅ You can tolerate **eventual consistency**
✅ **Availability** is more important than consistency
✅ You're doing **simple key-value lookups**

---

## Migration Considerations

### SQL → NoSQL Migration

**When to migrate:**
- SQL database can't handle write load
- Need to scale horizontally
- Sharding SQL database is too complex

**Challenges:**
- Lose ACID guarantees
- Must handle consistency in application
- Rewrite queries (no SQL)
- Learn new database paradigm

**Example:** Instagram migrated from PostgreSQL to Cassandra for photo feeds

### NoSQL → SQL Migration

**When to migrate:**
- Need complex querying (JOINs, aggregations)
- Need strong consistency guarantees
- Data has become more structured
- Dealing with too much data duplication

**Challenges:**
- Design normalized schema
- Migrate large volumes of data
- May lose horizontal scalability

**Example:** Segment migrated from MongoDB to PostgreSQL for better reliability

---

## Common Myths Debunked

### Myth 1: "NoSQL is always faster"
**Reality:** SQL can be faster for complex queries with proper indexing. NoSQL is faster for simple key-value lookups.

### Myth 2: "SQL doesn't scale"
**Reality:** SQL scales vertically very well. With read replicas and sharding, can handle billions of records (Instagram uses PostgreSQL).

### Myth 3: "NoSQL means no schema"
**Reality:** Most NoSQL databases support optional schemas. Schema-less means flexible, not chaotic.

### Myth 4: "You must choose one"
**Reality:** Most companies use both! Use the right tool for each use case.

---

## Summary

**SQL (Relational):**
- Use for: Transactions, complex queries, structured data
- Examples: Banking, e-commerce, enterprise apps
- Databases: PostgreSQL, MySQL, Oracle

**NoSQL (Non-Relational):**
- Use for: Scale, flexibility, high throughput
- Examples: Social feeds, analytics, caching
- Databases: MongoDB, Cassandra, Redis, DynamoDB

**Best Practice:** Use both in a hybrid approach
- SQL for critical transactional data
- NoSQL for high-scale, flexible data
- Redis for caching

**Interview Tip:** Always ask clarifying questions about:
- Data consistency requirements
- Expected scale (millions vs billions)
- Query patterns (simple lookups vs complex JOINs)
- Read/write ratio

Then choose the appropriate database based on requirements, not preferences!

---

**Next:** Learn about [Database Replication](database-replication.md) for scaling reads and high availability.
