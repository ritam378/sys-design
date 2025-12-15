# Database Sharding

## Overview

**Sharding** (horizontal partitioning) is the process of **splitting data across multiple databases** to scale beyond the capacity of a single server. Each shard holds a subset of the total data.

**Key Question:** "How do you scale a database to handle billions of records and millions of writes per second?"

**Short Answer:**
- **Sharding** distributes data across multiple database servers
- Each shard is independent (own CPU, RAM, disk)
- Enables horizontal scaling for both reads AND writes

---

## What is Sharding?

Sharding splits a large dataset into smaller chunks (**shards**), each stored on a separate database server.

**Before Sharding (Single Database):**

```
┌─────────────────────────────────┐
│      Single Database            │
│  10 TB data, 100K writes/sec    │
│                                 │
│  users table (1 billion rows)   │
│  posts table (10 billion rows)  │
│                                 │
│  ❌ CPU maxed out               │
│  ❌ Disk I/O bottleneck         │
│  ❌ Cannot scale further        │
└─────────────────────────────────┘
```

**After Sharding (4 Shards):**

```
Application Layer (knows which shard to query)
         │
    ┌────┼────┬────────┬────────┐
    │    │    │        │        │
    ▼    ▼    ▼        ▼        ▼
┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
│Shard 0│ │Shard 1│ │Shard 2│ │Shard 3│
├───────┤ ├───────┤ ├───────┤ ├───────┤
│user_id│ │user_id│ │user_id│ │user_id│
│ 0-24% │ │25-49% │ │50-74% │ │75-99% │
│       │ │       │ │       │ │       │
│2.5TB  │ │2.5TB  │ │2.5TB  │ │2.5TB  │
│25K w/s│ │25K w/s│ │25K w/s│ │25K w/s│
└───────┘ └───────┘ └───────┘ └───────┘

✅ Each shard handles 1/4 of load
✅ Can add more shards to scale
✅ Independent failures
```

**Sharding vs Replication:**

| Aspect | Replication | Sharding |
|--------|-------------|----------|
| **Data** | Full copy on each node | Different data on each shard |
| **Purpose** | Availability, read scaling | Write scaling, storage capacity |
| **Queries** | Any replica has all data | Must route to correct shard |
| **Scalability** | Scales reads | Scales reads AND writes |

---

## Sharding Strategies

Choosing **how to distribute data** is critical. Different strategies have different trade-offs.

### 1. Hash-Based Sharding

**Most common.** Hash the shard key, use modulo to determine shard.

**Algorithm:**

```python
def get_shard_id(user_id: int, num_shards: int) -> int:
    """Distribute users evenly across shards"""
    return hash(user_id) % num_shards

# Example:
get_shard_id(123, 4)   # → 3  (user 123 goes to shard 3)
get_shard_id(456, 4)   # → 0  (user 456 goes to shard 0)
get_shard_id(789, 4)   # → 1  (user 789 goes to shard 1)
```

**Visual Distribution:**

```
Users: [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12]

hash(user_id) % 4:

Shard 0: [4, 8, 12]      ← 25%
Shard 1: [1, 5, 9]       ← 25%
Shard 2: [2, 6, 10]      ← 25%
Shard 3: [3, 7, 11]      ← 25%

✅ Even distribution
```

**Pros:**
- ✅ **Even distribution** - Load balanced across shards
- ✅ **Simple to implement**
- ✅ **No hotspots** (assuming good hash function)

**Cons:**
- ❌ **Resharding is hard** - Changing num_shards requires moving data
- ❌ **No range queries** - Can't query "users 1-1000" (scattered across shards)
- ❌ **Related data scattered** - User's posts spread across shards

**When to Use:**
- Even distribution more important than range queries
- Known number of shards (or willing to reshard)
- Accessing data by key (e.g., user_id lookup)

**Example (Instagram):**

Instagram shards users by `user_id`:

```python
class ShardedDatabase:
    def __init__(self, num_shards: int):
        self.shards = [
            asyncpg.create_pool(f"postgresql://shard{i}:5432/instagram")
            for i in range(num_shards)
        ]
        self.num_shards = num_shards

    def get_shard(self, user_id: int):
        """Get database connection for user"""
        shard_id = hash(user_id) % self.num_shards
        return self.shards[shard_id]

    async def get_user(self, user_id: int):
        """Fetch user from correct shard"""
        shard = self.get_shard(user_id)
        async with shard.acquire() as conn:
            return await conn.fetchrow(
                "SELECT * FROM users WHERE user_id = $1",
                user_id
            )

    async def create_post(self, user_id: int, content: str):
        """Create post in user's shard"""
        shard = self.get_shard(user_id)
        async with shard.acquire() as conn:
            return await conn.fetchrow(
                "INSERT INTO posts (user_id, content) VALUES ($1, $2) RETURNING *",
                user_id, content
            )
```

---

### 2. Range-Based Sharding

**Partition by ranges** of shard key values.

**Example:**

```
Shard 0: user_id 1       - 250,000,000
Shard 1: user_id 250,000,001 - 500,000,000
Shard 2: user_id 500,000,001 - 750,000,000
Shard 3: user_id 750,000,001 - 1,000,000,000
```

**Implementation:**

```python
class RangeShardRouter:
    def __init__(self):
        self.ranges = [
            (1, 250_000_000, "shard0"),
            (250_000_001, 500_000_000, "shard1"),
            (500_000_001, 750_000_000, "shard2"),
            (750_000_001, 1_000_000_000, "shard3"),
        ]

    def get_shard(self, user_id: int) -> str:
        """Find shard based on range"""
        for start, end, shard_name in self.ranges:
            if start <= user_id <= end:
                return shard_name
        raise ValueError(f"No shard for user_id {user_id}")

    async def get_users_in_range(self, start_id: int, end_id: int):
        """Efficient range query"""
        # Determine which shards contain this range
        shards_to_query = []
        for range_start, range_end, shard_name in self.ranges:
            if not (end_id < range_start or start_id > range_end):
                shards_to_query.append(shard_name)

        # Query only relevant shards
        results = []
        for shard_name in shards_to_query:
            shard = self.get_connection(shard_name)
            async with shard.acquire() as conn:
                rows = await conn.fetch(
                    "SELECT * FROM users WHERE user_id BETWEEN $1 AND $2",
                    start_id, end_id
                )
                results.extend(rows)
        return results
```

**Pros:**
- ✅ **Range queries efficient** - Query only relevant shards
- ✅ **Related data co-located** - Sequential IDs on same shard
- ✅ **Easy to add shards** - Just split a range

**Cons:**
- ❌ **Hotspots possible** - New users all go to last shard
- ❌ **Uneven distribution** - Some ranges more popular
- ❌ **Rebalancing needed** - Must split hot shards

**Hotspot Example:**

```
Shard 0: users 1M - 2M      (old users, low activity)    10 QPS
Shard 1: users 2M - 3M      (old users, low activity)    15 QPS
Shard 2: users 3M - 4M      (medium activity)            50 QPS
Shard 3: users 4M+          (NEW users, high activity)   500 QPS ❌ HOTSPOT!
```

**Solution: Split hot shard**

```
Shard 3a: users 4M - 4.5M   (250 QPS)
Shard 3b: users 4.5M+       (250 QPS)
```

**When to Use:**
- Need range queries (e.g., "users created in 2024")
- Can predict/manage hotspots
- Data has natural ordering

**Example (Time-series data):**

```python
# Shard logs by date
def get_shard_for_log(timestamp: datetime) -> str:
    if timestamp < datetime(2024, 1, 1):
        return "shard_2023"
    elif timestamp < datetime(2024, 4, 1):
        return "shard_2024_q1"
    elif timestamp < datetime(2024, 7, 1):
        return "shard_2024_q2"
    else:
        return "shard_2024_q3"

# Benefit: Old shards can be archived, only recent shards are hot
```

---

### 3. Geographic Sharding

**Shard by user location** for low latency.

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  US Shard   │    │  EU Shard   │    │ Asia Shard  │
│             │    │             │    │             │
│ US users    │    │ EU users    │    │ Asia users  │
│             │    │             │    │             │
│ US region   │    │ EU region   │    │ Asia region │
└─────────────┘    └─────────────┘    └─────────────┘
```

**Implementation:**

```python
class GeoShardRouter:
    def __init__(self):
        self.shard_map = {
            "US": "postgresql://us-east-1/db",
            "EU": "postgresql://eu-west-1/db",
            "ASIA": "postgresql://ap-southeast-1/db",
        }

    def get_shard(self, user_region: str):
        """Route to geographically close shard"""
        return self.shard_map.get(user_region, "US")  # Default to US

    async def get_user(self, user_id: int, user_region: str):
        """Fetch from local shard (low latency)"""
        shard = self.get_shard(user_region)
        async with shard.acquire() as conn:
            return await conn.fetchrow(
                "SELECT * FROM users WHERE user_id = $1",
                user_id
            )
```

**Pros:**
- ✅ **Low latency** - Users access local shard
- ✅ **Compliance** - GDPR (EU data stays in EU)
- ✅ **Fault isolation** - US outage doesn't affect EU

**Cons:**
- ❌ **Cross-region queries expensive** - US user viewing EU user's profile
- ❌ **Uneven distribution** - Some regions bigger
- ❌ **Complex routing** - Must know user's region

**When to Use:**
- Global user base
- Latency critical
- Regulatory requirements (data residency)

**Example (Netflix):**

Netflix shards by region to comply with content licensing and reduce latency.

---

### 4. Directory-Based Sharding

**Lookup table** maps shard key to shard.

```
┌─────────────────────────────┐
│   Shard Directory (Lookup)  │
├─────────────┬───────────────┤
│  user_id    │   shard       │
├─────────────┼───────────────┤
│     123     │   shard_a     │
│     456     │   shard_b     │
│     789     │   shard_a     │
│    1011     │   shard_c     │
└─────────────┴───────────────┘
```

**Implementation:**

```python
class DirectoryShardRouter:
    def __init__(self):
        # In-memory cache of directory (loaded from Redis/DB)
        self.directory = {}  # user_id → shard_name

    async def get_shard(self, user_id: int) -> str:
        """Lookup shard in directory"""
        if user_id not in self.directory:
            # Load from directory database
            shard = await self.directory_db.fetchval(
                "SELECT shard_name FROM shard_directory WHERE user_id = $1",
                user_id
            )
            self.directory[user_id] = shard
        return self.directory[user_id]

    async def move_user_to_shard(self, user_id: int, new_shard: str):
        """Migrate user to different shard (flexible!)"""
        old_shard = await self.get_shard(user_id)

        # Copy data to new shard
        user_data = await self.get_connection(old_shard).fetch(
            "SELECT * FROM users WHERE user_id = $1", user_id
        )
        await self.get_connection(new_shard).execute(
            "INSERT INTO users (...) VALUES (...)", *user_data
        )

        # Update directory
        await self.directory_db.execute(
            "UPDATE shard_directory SET shard_name = $1 WHERE user_id = $2",
            new_shard, user_id
        )

        # Delete from old shard
        await self.get_connection(old_shard).execute(
            "DELETE FROM users WHERE user_id = $1", user_id
        )

        # Update cache
        self.directory[user_id] = new_shard
```

**Pros:**
- ✅ **Flexible** - Can move users between shards easily
- ✅ **No resharding** - Add/remove shards without moving everyone
- ✅ **Custom placement** - VIP users on special shards

**Cons:**
- ❌ **Extra lookup** - Must query directory first
- ❌ **Directory is bottleneck** - Single point of failure
- ❌ **Complexity** - Need to maintain directory

**When to Use:**
- Need flexibility to rebalance
- Willing to pay cost of extra lookup
- Complex sharding rules (e.g., VIP users)

---

## Sharding Key Selection

**Most critical decision.** Choosing the wrong shard key causes hotspots and inefficiency.

### Good Shard Keys

✅ **High cardinality** - Many unique values (user_id ✅, country ❌)
✅ **Evenly distributed** - No single value dominates
✅ **Queried frequently** - Most queries include shard key
✅ **Unchanging** - Shard key shouldn't change (avoid resharding)

**Examples:**

| Shard Key | Cardinality | Distribution | Good? |
|-----------|-------------|--------------|-------|
| `user_id` | Billions | Even | ✅ Excellent |
| `email` | Billions | Even | ✅ Good (but can change) |
| `country` | ~200 | Uneven (US >> others) | ❌ Bad |
| `created_date` | 365/year | Uneven (recent dates hot) | ⚠️ OK for time-series |
| `tenant_id` (SaaS) | Thousands | Varies | ✅ Good for multi-tenancy |

### Bad Shard Key Example

**Sharding by `status` (active/inactive):**

```
Shard 0: status = 'active'       ← 90% of users  ❌ HOTSPOT!
Shard 1: status = 'inactive'     ← 10% of users  Underutilized
```

All queries for active users hit one shard → no benefit!

**Fix: Use `user_id` instead.**

---

## Cross-Shard Queries

**Challenge:** Querying data across multiple shards is expensive.

### Example Problem

```sql
-- Single database (easy)
SELECT * FROM users WHERE country = 'US' ORDER BY created_at LIMIT 10;

-- Sharded database (hard!)
-- Must query ALL shards, merge results
```

**Implementation:**

```python
async def cross_shard_query(query: str, *args):
    """Query all shards and merge results"""
    tasks = []
    for shard in all_shards:
        tasks.append(shard.fetch(query, *args))

    # Execute in parallel
    results = await asyncio.gather(*tasks)

    # Merge results from all shards
    merged = []
    for shard_results in results:
        merged.extend(shard_results)

    return merged

# Usage
all_us_users = await cross_shard_query(
    "SELECT * FROM users WHERE country = 'US'"
)
# ❌ Queried all 100 shards! Expensive!
```

**Optimization: Denormalize**

```python
# Instead of querying shards, maintain a separate index
# E.g., Elasticsearch index of all users
es_results = await es_client.search(
    index="users",
    body={"query": {"term": {"country": "US"}}}
)
# ✅ Single query, no cross-shard join
```

### Joins Across Shards

**Very expensive.** Avoid if possible.

```sql
-- Single DB (easy)
SELECT u.name, p.title
FROM users u
JOIN posts p ON u.user_id = p.user_id
WHERE u.country = 'US';

-- Sharded DB (nightmare!)
-- Users and posts on different shards
-- Must fetch from shard A, then shard B, join in application
```

**Solution: Co-locate related data**

```python
# Shard both users and posts by user_id
# Now user and their posts always on same shard

shard = get_shard(user_id)
async with shard.acquire() as conn:
    # Single shard query (fast!)
    result = await conn.fetch("""
        SELECT u.name, p.title
        FROM users u
        JOIN posts p ON u.user_id = p.user_id
        WHERE u.user_id = $1
    """, user_id)
```

---

## Resharding (Changing Number of Shards)

**Problem:** App grows, need to add more shards. But hash-based sharding breaks!

**Before (4 shards):**

```python
shard_id = hash(user_id) % 4
```

**After adding 1 shard (5 shards):**

```python
shard_id = hash(user_id) % 5  # ❌ Most users now on different shard!
```

**Example:**

```
user_id=123:
  hash(123) % 4 = 3  (before) ← Shard 3
  hash(123) % 5 = 3  (after)  ← Still Shard 3 ✅

user_id=456:
  hash(456) % 4 = 0  (before) ← Shard 0
  hash(456) % 5 = 1  (after)  ← Now Shard 1! ❌ Moved!

Result: 80% of data must be migrated!
```

### Solution 1: Consistent Hashing

**Minimize data movement** when adding/removing shards.

```python
import hashlib

class ConsistentHash:
    def __init__(self, num_shards: int, virtual_nodes: int = 150):
        self.ring = {}  # hash → shard
        self.shards = []
        self.virtual_nodes = virtual_nodes

        for i in range(num_shards):
            self.add_shard(f"shard_{i}")

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_shard(self, shard_name: str):
        """Add shard to ring"""
        self.shards.append(shard_name)
        # Add virtual nodes to distribute evenly
        for i in range(self.virtual_nodes):
            virtual_key = f"{shard_name}:{i}"
            hash_val = self._hash(virtual_key)
            self.ring[hash_val] = shard_name

    def get_shard(self, key: str) -> str:
        """Find shard for key"""
        if not self.ring:
            raise ValueError("No shards available")

        hash_val = self._hash(str(key))

        # Find first shard clockwise from hash
        for ring_hash in sorted(self.ring.keys()):
            if hash_val <= ring_hash:
                return self.ring[ring_hash]

        # Wrap around to first shard
        return self.ring[min(self.ring.keys())]
```

**Benefit:**

```
Adding shard_4:
  Only ~20% of keys move (from neighbor shards)
  vs. 80% with modulo hashing
```

### Solution 2: Logical Shards

**Use many logical shards**, map to fewer physical servers.

```python
# 1024 logical shards (fixed)
logical_shard = hash(user_id) % 1024

# Map to 4 physical servers (can change)
physical_shard_map = {
    range(0, 256): "server_0",      # Logical 0-255 → Server 0
    range(256, 512): "server_1",    # Logical 256-511 → Server 1
    range(512, 768): "server_2",    # Logical 512-767 → Server 2
    range(768, 1024): "server_3",   # Logical 768-1023 → Server 3
}

def get_physical_shard(user_id: int) -> str:
    logical = hash(user_id) % 1024
    for range_obj, server in physical_shard_map.items():
        if logical in range_obj:
            return server
```

**Adding server:**

```python
# Remap some logical shards to new server
physical_shard_map = {
    range(0, 200): "server_0",      # Moved 56 logical shards to server_4
    range(200, 400): "server_1",
    range(400, 600): "server_2",
    range(600, 800): "server_3",
    range(800, 1024): "server_4",   # NEW SERVER
}

# Only ~20% of data moves
```

---

## Challenges and Solutions

### 1. Auto-Incrementing IDs

**Problem:** Each shard generates IDs independently → collisions!

```
Shard 0: user_id = 1, 2, 3, 4, ...
Shard 1: user_id = 1, 2, 3, 4, ...  ❌ Collision!
```

**Solution A: UUID**

```python
import uuid

user_id = uuid.uuid4()  # Globally unique
# e.g., "550e8400-e29b-41d4-a716-446655440000"

# Pros: ✅ No coordination needed
# Cons: ❌ Large (128 bits), not sortable, not user-friendly
```

**Solution B: Snowflake IDs (Twitter)**

```python
import time

def generate_snowflake_id(shard_id: int) -> int:
    """
    64-bit ID:
    - 41 bits: timestamp (milliseconds since epoch)
    - 10 bits: shard_id (supports 1024 shards)
    - 13 bits: sequence (4096 IDs/ms per shard)
    """
    timestamp = int(time.time() * 1000) - EPOCH  # 41 bits
    sequence = get_next_sequence()  # 13 bits (increments)

    return (timestamp << 23) | (shard_id << 13) | sequence

# Example:
# Shard 0: 1234567890123456
# Shard 1: 1234567890234567
# ✅ Globally unique, sortable, compact
```

**Solution C: ID Generation Service**

```python
# Centralized service generates unique IDs
id = await id_service.generate_id()

# Pros: ✅ Sequential, compact
# Cons: ❌ Single point of failure, extra network call
```

### 2. Transactions Across Shards

**Problem:** ACID transactions don't work across shards.

```python
# Shard 0: user_id = 123, balance = $100
# Shard 1: user_id = 456, balance = $50

# Transfer $30 from user 123 → user 456
BEGIN TRANSACTION;
  UPDATE users SET balance = balance - 30 WHERE user_id = 123;  # Shard 0
  UPDATE users SET balance = balance + 30 WHERE user_id = 456;  # Shard 1
COMMIT;

# ❌ Not atomic across shards!
# Shard 0 commits, Shard 1 fails → money lost!
```

**Solution A: Two-Phase Commit (2PC)**

```python
# Phase 1: Prepare
shard_0.execute("PREPARE TRANSACTION 'txn_123'")
shard_1.execute("PREPARE TRANSACTION 'txn_123'")

# Phase 2: Commit
if both_prepared:
    shard_0.execute("COMMIT PREPARED 'txn_123'")
    shard_1.execute("COMMIT PREPARED 'txn_123'")
else:
    shard_0.execute("ROLLBACK PREPARED 'txn_123'")
    shard_1.execute("ROLLBACK PREPARED 'txn_123'")

# Pros: ✅ Atomic
# Cons: ❌ Slow (2 round trips), blocking, coordinator failure
```

**Solution B: Saga Pattern (Eventual Consistency)**

```python
# Step 1: Deduct from user 123 (Shard 0)
await shard_0.execute("UPDATE users SET balance = balance - 30 WHERE user_id = 123")

try:
    # Step 2: Add to user 456 (Shard 1)
    await shard_1.execute("UPDATE users SET balance = balance + 30 WHERE user_id = 456")
except Exception:
    # Compensating transaction: refund user 123
    await shard_0.execute("UPDATE users SET balance = balance + 30 WHERE user_id = 123")
    raise

# Pros: ✅ No locks, non-blocking
# Cons: ❌ Not atomic (brief inconsistency), complex compensating logic
```

**Solution C: Avoid Cross-Shard Transactions**

```python
# Redesign: Keep user balance in a separate "wallet service"
# Wallet service is sharded by user_id (same as users)
# Now both updates on same shard → local transaction works!
```

---

## Real-World Examples

### Instagram (PostgreSQL)

**Strategy:** Hash-based sharding by `user_id`

**Details:**
- Thousands of PostgreSQL shards
- Snowflake IDs for globally unique IDs
- Users and their posts co-located on same shard
- Replication within each shard for HA

**Why:**
- Billions of users (can't fit on one server)
- User-centric queries (fetch user's posts)
- Hash ensures even distribution

### Discord (Cassandra)

**Strategy:** Compound shard key `(channel_id, timestamp)`

**Details:**
- Messages sharded by channel
- All messages in a channel on same shard
- Time-based for range queries (recent messages)

**Why:**
- Queries are "get messages for channel"
- Co-location for efficient joins
- Time-based allows archiving old shards

### Uber (MySQL)

**Strategy:** Geographic sharding by city

**Details:**
- Each city has own shard
- Trips, drivers, riders for that city co-located

**Why:**
- Trips are city-specific (no cross-city queries)
- Low latency (local shard)
- Fault isolation (NYC outage doesn't affect SF)

---

## Best Practices

### 1. Design Schema for Sharding from Day 1

**Even if starting with single DB**, choose shard key and include in queries.

```sql
-- Bad: Hard to shard later
SELECT * FROM posts WHERE created_at > '2024-01-01';

-- Good: Includes shard key
SELECT * FROM posts WHERE user_id = 123 AND created_at > '2024-01-01';
```

### 2. Monitor Shard Imbalance

```python
async def monitor_shards():
    for shard in all_shards:
        row_count = await shard.fetchval("SELECT COUNT(*) FROM users")
        qps = get_qps(shard)

        print(f"{shard.name}: {row_count} rows, {qps} QPS")

        if row_count > THRESHOLD or qps > QPS_LIMIT:
            alert(f"Shard {shard.name} is hot!")

# Output:
# shard_0: 10M rows, 100 QPS  ✅
# shard_1: 10M rows, 95 QPS   ✅
# shard_2: 10M rows, 105 QPS  ✅
# shard_3: 50M rows, 500 QPS  ❌ HOTSPOT! (split needed)
```

### 3. Use Connection Pooling Per Shard

```python
class ShardManager:
    def __init__(self, num_shards: int):
        self.pools = [
            asyncpg.create_pool(
                f"postgresql://shard{i}:5432/db",
                min_size=10,
                max_size=50
            )
            for i in range(num_shards)
        ]

    def get_pool(self, user_id: int):
        shard_id = hash(user_id) % len(self.pools)
        return self.pools[shard_id]
```

### 4. Test Resharding Process

**Before production**, practice adding/removing shards.

```bash
# Simulate adding shard
1. Add new shard to cluster
2. Update routing logic (gradual rollout)
3. Migrate data (in background, no downtime)
4. Verify data consistency
5. Switch traffic to new shard
6. Remove old data

# Ensure zero downtime!
```

---

## Decision Matrix

### When to Shard

✅ **Shard when:**
- Single DB can't handle write load (>10K writes/sec)
- Data size exceeds single server (>1 TB)
- Need to scale beyond vertical limits
- Geographic distribution required

❌ **Don't shard if:**
- Current DB handles load fine (premature optimization!)
- Can scale vertically (simpler)
- Complexity not worth it (sharding adds significant complexity)

### Sharding Strategy

| Requirement | Strategy |
|-------------|----------|
| Even distribution | Hash-based |
| Range queries | Range-based |
| Low latency (global) | Geographic |
| Flexibility | Directory-based |
| Time-series data | Range by timestamp |

---

## Summary

**Sharding** distributes data across multiple databases for:
- ✅ **Write scalability** (beyond single server)
- ✅ **Storage capacity** (petabytes)
- ✅ **Fault isolation** (independent failures)

**Sharding Strategies:**
- **Hash-based** (most common) - Even distribution, simple
- **Range-based** - Efficient range queries, hotspot risk
- **Geographic** - Low latency, compliance
- **Directory-based** - Flexible, extra lookup overhead

**Challenges:**
- Choosing shard key (user_id usually best)
- Cross-shard queries (avoid or denormalize)
- Distributed transactions (2PC or Saga)
- Resharding (consistent hashing helps)
- Auto-increment IDs (use Snowflake or UUID)

**Real-World:**
- Instagram: Shards PostgreSQL by user_id, thousands of shards
- Discord: Shards Cassandra by channel_id
- Uber: Geographic sharding by city

**Best Practice:** Design for sharding early, monitor imbalance, test resharding.

**Next:** Learn about [Database Indexing](database-indexing.md) for query performance.
