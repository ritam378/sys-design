# Database Indexing

## Overview

**Indexing** is a data structure technique that **dramatically improves query speed** by creating efficient lookup paths to data. Think of it like a book's index: instead of reading every page to find a topic, you check the index and jump directly to the right page.

**Key Question:** "How do you make database queries fast?"

**Short Answer:**
- **Indexes** create efficient lookup structures (usually B+ trees or hash tables)
- Trade-off: **Faster reads** vs **Slower writes** (must update index)
- Critical for performance at scale

---

## What is an Index?

An index is a **separate data structure** that stores a subset of table data in a way optimized for quick lookups.

**Without Index (Full Table Scan):**

```
Table: users (1 million rows)
┌────────┬──────────┬─────────────────────┐
│ user_id│   name   │       email         │
├────────┼──────────┼─────────────────────┤
│   1    │  Alice   │ alice@example.com   │
│   2    │  Bob     │ bob@example.com     │
│   3    │  Carol   │ carol@example.com   │
│  ...   │   ...    │       ...           │
│ 1000000│  Zoe     │ zoe@example.com     │
└────────┴──────────┴─────────────────────┘

Query: SELECT * FROM users WHERE email = 'bob@example.com';

❌ Must scan all 1M rows (slow!)
```

**With Index on `email`:**

```
Index on email (B+ tree):
           [m@...]
          /       \
    [d@...]       [s@...]
    /    \        /     \
[a@...] [e@...] [n@...] [z@...]
  |       |       |       |
 Row 1  Row 2   Row 3   Row 1M

Query: SELECT * FROM users WHERE email = 'bob@example.com';

✅ Index lookup: O(log n) = ~20 comparisons
✅ Jump directly to Row 2
✅ 50,000x faster!
```

**Time Complexity:**

| Operation | Without Index | With Index (B+ tree) |
|-----------|---------------|----------------------|
| **Exact match** | O(n) | O(log n) |
| **Range query** | O(n) | O(log n + k) |
| **Order by** | O(n log n) | O(k) if sorted |

---

## How Indexes Work

### B+ Tree (Most Common)

**B+ trees** are balanced tree structures optimized for disk-based storage.

**Structure:**

```
                    Root Node
                   [50 | 100]
                  /     |     \
                /       |       \
         [10|25|40]  [60|80]  [110|150]
          /  |  |  \   / |  \   /  |   \
        /    |  |   \ /  |   \ /   |    \
   Leaf Nodes (contain actual data/pointers):
   [1,5,8]  [10,15,20]  [25,30,35]  [40,45,48]  ...

   Each leaf points to actual table rows
```

**Characteristics:**

✅ **Balanced** - All leaf nodes at same depth (consistent performance)
✅ **Range queries** - Leaf nodes linked (easy to scan range)
✅ **Sorted** - Data in order (supports ORDER BY without sorting)
✅ **Disk-friendly** - Nodes fit in disk pages (few I/O operations)

**Example Query:**

```sql
SELECT * FROM users WHERE age = 25;

-- Traversal:
Root: 25 < 50, go left
Level 2: 25 >= 10 and 25 < 40, go middle
Leaf: Found 25 → Fetch rows [101, 305, 782]

-- Only 3 disk reads (vs. 1 million for table scan)
```

**Time Complexity:**

- **Search:** O(log n)
- **Insert:** O(log n)
- **Delete:** O(log n)
- **Range query:** O(log n + k) where k = results

---

### Hash Index

**Hash table** for exact-match lookups.

**Structure:**

```
Hash(email) → Bucket → Row pointer

Hash("alice@example.com") = 12345 → [Row 1]
Hash("bob@example.com")   = 67890 → [Row 2]
Hash("carol@example.com") = 11111 → [Row 3]
```

**Characteristics:**

✅ **O(1) lookups** - Constant time for exact match
❌ **No range queries** - Can't do `age > 25`
❌ **No sorting** - Can't support ORDER BY
❌ **Collisions** - Multiple keys hash to same bucket

**When to Use:**

- Exact-match queries only (e.g., `WHERE email = 'bob@example.com'`)
- In-memory databases (Redis, Memcached)
- Not common in traditional SQL databases

**Example (PostgreSQL):**

```sql
CREATE INDEX idx_email_hash ON users USING HASH (email);

-- Good:
SELECT * FROM users WHERE email = 'bob@example.com';  ✅ O(1)

-- Bad:
SELECT * FROM users WHERE email LIKE 'bob%';  ❌ Can't use hash index
SELECT * FROM users WHERE email > 'bob@example.com';  ❌ No range support
```

---

### Bitmap Index

**Bitmap** for low-cardinality columns (few distinct values).

**Example:**

```
Table: users (1M rows)
Column: gender (only 3 values: M, F, Other)

Bitmap index on gender:
M:     [1, 0, 1, 1, 0, 0, 1, ...]  ← 1 = row has value 'M'
F:     [0, 1, 0, 0, 1, 1, 0, ...]
Other: [0, 0, 0, 0, 0, 0, 0, ...]

Query: SELECT * FROM users WHERE gender = 'M';
→ Scan bitmap, fetch rows where bit = 1
```

**Benefits:**

✅ **Space-efficient** - 1 bit per row per value
✅ **Fast AND/OR** - Bitwise operations
✅ **Low cardinality** - Few distinct values (gender, boolean, status)

**Example (Boolean Queries):**

```sql
-- Find active male users
SELECT * FROM users WHERE gender = 'M' AND status = 'active';

Bitmap for gender=M:    [1, 0, 1, 1, 0, ...]
Bitmap for status=active: [1, 1, 0, 1, 0, ...]
Bitwise AND:            [1, 0, 0, 1, 0, ...]  ← Result
```

**When to Use:**

- Low-cardinality columns (< 100 distinct values)
- Data warehouses (OLAP)
- Read-heavy workloads

**Not Recommended:**

- High-cardinality columns (user_id, email)
- Write-heavy workloads (updates are expensive)

---

## Types of Indexes

### 1. Single-Column Index

Index on one column.

```sql
CREATE INDEX idx_email ON users(email);
```

**Good for:**

```sql
SELECT * FROM users WHERE email = 'bob@example.com';  ✅ Uses index
```

**Not used:**

```sql
SELECT * FROM users WHERE name = 'Bob';  ❌ No index on name
```

---

### 2. Composite (Multi-Column) Index

Index on multiple columns.

```sql
CREATE INDEX idx_name_age ON users(name, age);
```

**Index structure:**

```
(name, age) pairs sorted:
[('Alice', 25), ('Alice', 30), ('Bob', 20), ('Bob', 35), ('Carol', 28), ...]
```

**Key Rule: Left-to-Right Prefix**

```sql
-- ✅ Uses index (matches left prefix)
SELECT * FROM users WHERE name = 'Bob';
SELECT * FROM users WHERE name = 'Bob' AND age = 30;

-- ❌ Does NOT use index (missing left prefix)
SELECT * FROM users WHERE age = 30;  -- 'name' is missing
```

**Think of it like a phone book:**

- Sorted by (last_name, first_name)
- Can find "Smith, John" efficiently
- Can find all "Smith" efficiently
- **Cannot** find all "John" efficiently (need first_name index)

**Optimal Order:**

Put **most selective column first** (column that filters most rows).

```sql
-- Bad: age first (low selectivity)
CREATE INDEX idx_age_email ON users(age, email);
-- age=25 might match 100K users (still slow)

-- Good: email first (high selectivity)
CREATE INDEX idx_email_age ON users(email, age);
-- email='bob@example.com' matches 1 user (very fast)
```

---

### 3. Unique Index

Enforces uniqueness, also speeds up lookups.

```sql
CREATE UNIQUE INDEX idx_unique_email ON users(email);

-- Now:
INSERT INTO users (email) VALUES ('bob@example.com');  ✅
INSERT INTO users (email) VALUES ('bob@example.com');  ❌ Error: Duplicate
```

**Benefit:**

- ✅ Enforces constraint at database level
- ✅ Faster than regular index (knows to stop after 1 result)

---

### 4. Partial Index

Index only a **subset** of rows.

```sql
-- Only index active users
CREATE INDEX idx_active_users ON users(email) WHERE status = 'active';
```

**Benefits:**

✅ **Smaller index** - Faster, less storage
✅ **Faster updates** - Inactive users don't update index
✅ **Targeted queries** - Perfect for `WHERE status = 'active'`

**Example:**

```sql
-- ✅ Uses partial index
SELECT * FROM users WHERE email = 'bob@example.com' AND status = 'active';

-- ❌ Does NOT use partial index (includes inactive)
SELECT * FROM users WHERE email = 'bob@example.com';
```

**When to Use:**

- Queries filter on same condition repeatedly
- Large table but only care about subset (e.g., recent orders, active users)

---

### 5. Covering Index

Index that **includes all columns needed** for a query (no table lookup needed).

```sql
-- Regular index
CREATE INDEX idx_email ON users(email);

-- Query:
SELECT name FROM users WHERE email = 'bob@example.com';

-- Steps:
-- 1. Lookup email in index → Row pointer
-- 2. Fetch row from table to get 'name'  ← Extra disk read!
```

```sql
-- Covering index (includes 'name')
CREATE INDEX idx_email_name ON users(email, name);

-- Same query:
SELECT name FROM users WHERE email = 'bob@example.com';

-- Steps:
-- 1. Lookup email in index → 'name' is right there! ✅
-- No table lookup needed (faster!)
```

**PostgreSQL Syntax:**

```sql
CREATE INDEX idx_email_covering ON users(email) INCLUDE (name, created_at);
-- Index on 'email', also stores 'name' and 'created_at' for covering
```

**Benefit:**

- ✅ **Index-only scan** - No table access (much faster)

**Trade-off:**

- ❌ Larger index size
- ❌ Slower writes (more data to update)

---

### 6. Full-Text Index

For **text search** queries.

```sql
-- PostgreSQL
CREATE INDEX idx_fulltext ON posts USING GIN (to_tsvector('english', content));

-- Search:
SELECT * FROM posts
WHERE to_tsvector('english', content) @@ to_tsquery('database & indexing');

-- ✅ Finds posts containing "database" AND "indexing" (in any order)
```

**Features:**

✅ **Stemming** - "run", "running", "runs" all match
✅ **Stop words** - Ignores "the", "a", "is"
✅ **Ranking** - Orders results by relevance

**When to Use:**

- Blog posts, articles, comments
- "Search within text" features

**Better Alternatives for Complex Search:**

- Elasticsearch
- Algolia
- Meilisearch

---

## Indexing Best Practices

### 1. Index Columns in WHERE, JOIN, ORDER BY

```sql
-- Index columns used in filters
SELECT * FROM users WHERE email = 'bob@example.com';
→ CREATE INDEX idx_email ON users(email);

-- Index columns in JOINs
SELECT u.name, p.title
FROM users u
JOIN posts p ON u.user_id = p.user_id;
→ CREATE INDEX idx_posts_user_id ON posts(user_id);

-- Index columns in ORDER BY
SELECT * FROM posts ORDER BY created_at DESC LIMIT 10;
→ CREATE INDEX idx_created_at ON posts(created_at);
```

---

### 2. Don't Over-Index

**Each index has cost:**

- ❌ Slower writes (must update all indexes)
- ❌ More storage (indexes take space)
- ❌ Query planner overhead (more options to consider)

**Example:**

```sql
-- Table with 10 columns, 10 indexes (1 per column)

INSERT INTO users (...) VALUES (...);
-- Must update:
-- 1. Table itself
-- 2-11. All 10 indexes
-- Result: 11x slower writes! ❌
```

**Rule of Thumb:**

- ✅ Index columns used in 90%+ of queries
- ❌ Don't index columns rarely queried
- ✅ Remove unused indexes (monitor with pg_stat_user_indexes)

---

### 3. Choose Selectivity

**Selectivity** = uniqueness of values.

```
High selectivity (good):
  email: 1M distinct values / 1M rows = 100%  ✅
  user_id: 1M distinct values / 1M rows = 100%  ✅

Low selectivity (bad):
  gender: 3 distinct values / 1M rows = 0.0003%  ❌
  is_active: 2 distinct values / 1M rows = 0.0002%  ❌
```

**Index high-selectivity columns first in composite indexes.**

---

### 4. Monitor Index Usage

**PostgreSQL:**

```sql
-- Find unused indexes
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan AS index_scans,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Output:
-- indexname         | index_scans | index_size
-- idx_old_column    |      0      | 500 MB      ← Drop this!
-- idx_email         | 1000000     | 100 MB      ← Keep this
```

**Drop unused indexes:**

```sql
DROP INDEX idx_old_column;
-- Reclaim 500 MB, faster writes
```

---

### 5. Use EXPLAIN to Verify

**Always check if query uses index:**

```sql
EXPLAIN SELECT * FROM users WHERE email = 'bob@example.com';

-- Good output:
-- Index Scan using idx_email on users  (cost=0.42..8.44 rows=1)
-- ✅ Using index

-- Bad output:
-- Seq Scan on users  (cost=0.00..18334.00 rows=1)
-- ❌ Full table scan (index not used!)
```

**EXPLAIN ANALYZE (run query, show actual time):**

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'bob@example.com';

-- Output:
-- Index Scan using idx_email on users  (cost=0.42..8.44 rows=1 width=128) (actual time=0.015..0.016 rows=1)
-- ✅ Only 0.016ms (fast!)
```

---

## Common Pitfalls

### 1. Functions on Indexed Columns

```sql
-- Bad: Function on indexed column (index not used!)
SELECT * FROM users WHERE LOWER(email) = 'bob@example.com';
❌ Seq Scan (slow!)

-- Good: Use functional index
CREATE INDEX idx_email_lower ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'bob@example.com';
✅ Uses idx_email_lower
```

---

### 2. Implicit Type Conversion

```sql
-- email column is VARCHAR
CREATE INDEX idx_email ON users(email);

-- Bad: Passing INT (implicit conversion)
SELECT * FROM users WHERE email = 123;
-- Interpreted as: WHERE CAST(email AS INT) = 123
❌ Index not used (function on column!)

-- Good: Use correct type
SELECT * FROM users WHERE email = '123';
✅ Uses index
```

---

### 3. OR Conditions

```sql
-- Bad: OR across different columns (index might not be used)
SELECT * FROM users WHERE email = 'bob@example.com' OR name = 'Bob';
❌ Might do full table scan

-- Good: Separate queries with UNION
SELECT * FROM users WHERE email = 'bob@example.com'
UNION
SELECT * FROM users WHERE name = 'Bob';
✅ Each query uses its index
```

---

### 4. Leading Wildcards

```sql
-- Bad: Leading wildcard (index not used)
SELECT * FROM users WHERE email LIKE '%@example.com';
❌ Full table scan

-- Good: Prefix search
SELECT * FROM users WHERE email LIKE 'bob%';
✅ Uses index

-- Better: Use GIN index for pattern matching
CREATE INDEX idx_email_gin ON users USING GIN (email gin_trgm_ops);
-- Now LIKE '%@example.com' can use index
```

---

## Real-World Example

### Slow Query

```sql
-- Users table: 10M rows
-- Query:
SELECT * FROM users
WHERE status = 'active'
  AND created_at > '2024-01-01'
ORDER BY created_at DESC
LIMIT 10;

-- Without index:
-- Seq Scan: 15 seconds ❌
```

### Optimization Steps

**Step 1: EXPLAIN**

```sql
EXPLAIN ANALYZE SELECT ...;

-- Output:
-- Seq Scan on users  (cost=0.00..250000.00 rows=100000)
-- Filter: (status = 'active' AND created_at > '2024-01-01')
-- Sort: created_at DESC
-- ❌ Full table scan + sorting
```

**Step 2: Add Index**

```sql
-- Composite index: created_at first (for ORDER BY), status second
CREATE INDEX idx_created_status ON users(created_at DESC, status);
```

**Step 3: Verify**

```sql
EXPLAIN ANALYZE SELECT ...;

-- Output:
-- Index Scan Backward using idx_created_status on users  (cost=0.43..150.00 rows=10)
-- Index Cond: (created_at > '2024-01-01' AND status = 'active')
-- ✅ 0.05 seconds (300x faster!)
```

**Alternative: Partial Index**

```sql
-- If most queries filter on status = 'active'
CREATE INDEX idx_active_created ON users(created_at DESC) WHERE status = 'active';

-- Even smaller, faster index (only indexes active users)
```

---

## Index Maintenance

### Rebuilding Indexes

Over time, indexes can become **fragmented** (inefficient).

**PostgreSQL:**

```sql
-- Check index bloat
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
    pg_size_pretty(pg_relation_size(indexrelid) - pg_relation_size(indexrelid, 'main')) AS bloat
FROM pg_stat_user_indexes;

-- Rebuild fragmented index
REINDEX INDEX idx_email;
-- or
DROP INDEX idx_email;
CREATE INDEX idx_email ON users(email);  -- Recreate
```

**MySQL:**

```sql
OPTIMIZE TABLE users;
-- Rebuilds indexes
```

---

### Monitoring Index Performance

```sql
-- PostgreSQL: Index hit rate (should be > 99%)
SELECT
    sum(idx_blks_hit) / NULLIF(sum(idx_blks_hit + idx_blks_read), 0) AS index_hit_rate
FROM pg_statio_user_indexes;

-- If < 95%, indexes not cached well (increase shared_buffers)
```

---

## Summary

**Indexing** is essential for query performance:
- ✅ **Speeds up queries** (O(log n) vs O(n))
- ❌ **Slows down writes** (must update indexes)
- ⚖️ **Trade-off:** Read speed vs Write speed

**Index Types:**
- **B+ Tree** (default) - Balanced, supports range queries, ORDER BY
- **Hash** - O(1) exact match, no range queries
- **Bitmap** - Low-cardinality columns, space-efficient
- **Full-Text** - Text search with stemming/ranking

**Best Practices:**
- Index columns in WHERE, JOIN, ORDER BY
- Composite indexes: left-to-right prefix rule
- High selectivity columns first
- Use EXPLAIN to verify
- Monitor and drop unused indexes
- Avoid functions on indexed columns

**Real-World:**
- Instagram: Composite indexes on (user_id, created_at) for feed queries
- GitHub: Partial indexes on `status = 'open'` for issue queries
- Every major system: Heavy use of indexes for performance

**Next:** Learn about [CAP Theorem](cap-theorem.md) for distributed database trade-offs.
