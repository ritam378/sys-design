# Database Replication

## Overview

Database replication is the process of **copying and maintaining database data across multiple servers**. It's one of the most fundamental techniques for improving availability, scalability, and disaster recovery.

**Key Question:** "How do you scale database reads and ensure high availability?"

**Short Answer:**
- **Replication** creates copies of data on multiple servers
- **Read replicas** handle read traffic, reducing load on primary
- **Failover** to replica if primary fails

---

## What is Database Replication?

Replication involves maintaining **identical copies of data** on multiple database servers (nodes).

**Basic Concept:**

```
┌─────────────────┐
│  Primary (M)    │  ← Writes go here
│  (Master)       │
└────────┬────────┘
         │
         │ Replicates
         │
    ┌────┴────┬─────────┬──────────┐
    │         │         │          │
    ▼         ▼         ▼          ▼
┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
│ R1    │ │ R2    │ │ R3    │ │ R4    │  ← Reads from here
│Replica│ │Replica│ │Replica│ │Replica│
└───────┘ └───────┘ └───────┘ └───────┘
```

**Why Replication?**

1. **High Availability** - If primary fails, promote replica
2. **Read Scalability** - Distribute reads across replicas
3. **Disaster Recovery** - Geographic redundancy
4. **Data Locality** - Serve users from nearby replicas
5. **Analytics** - Run heavy queries on replica without impacting primary

---

## Replication Architectures

### 1. Master-Slave (Primary-Replica)

**Most common pattern.** Single primary handles writes, multiple replicas handle reads.

```
Application
    │
    ├──── Writes ────────────────┐
    │                             │
    │                             ▼
    │                      ┌─────────────┐
    │                      │  PRIMARY    │
    │                      │  (Master)   │
    │                      └──────┬──────┘
    │                             │
    │                    Replication Log
    │                             │
    │              ┌──────────────┼──────────────┐
    │              │              │              │
    ├── Reads ───► ▼              ▼              ▼
    │         ┌─────────┐    ┌─────────┐    ┌─────────┐
    │         │Replica 1│    │Replica 2│    │Replica 3│
    └────────►│ (Slave) │    │ (Slave) │    │ (Slave) │
              └─────────┘    └─────────┘    └─────────┘
```

**Characteristics:**

✅ **Pros:**
- Simple to implement and understand
- Clear separation: primary for writes, replicas for reads
- No write conflicts (single write source)
- Widely supported (MySQL, PostgreSQL, MongoDB)

❌ **Cons:**
- Single point of failure for writes (until failover)
- Replicas can lag behind primary (replication lag)
- Read-your-writes issues (write to primary, immediately read from replica might not see it)

**Example (PostgreSQL):**

```sql
-- On Primary Server
-- Enable write-ahead logging
wal_level = replica
max_wal_senders = 3
wal_keep_size = 64MB

-- Create replication user
CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'secret';
```

```sql
-- On Replica Server
-- Configure standby mode
primary_conninfo = 'host=primary_host port=5432 user=replicator password=secret'
hot_standby = on
```

**Application Code Pattern:**

```python
import asyncpg

class DatabasePool:
    def __init__(self):
        self.primary = None
        self.replicas = []

    async def initialize(self):
        # Primary for writes
        self.primary = await asyncpg.create_pool(
            "postgresql://primary_host:5432/db"
        )

        # Replicas for reads
        self.replicas = [
            await asyncpg.create_pool(f"postgresql://replica{i}:5432/db")
            for i in range(1, 4)
        ]

    async def execute_write(self, query: str, *args):
        """All writes go to primary"""
        async with self.primary.acquire() as conn:
            return await conn.execute(query, *args)

    async def execute_read(self, query: str, *args):
        """Reads go to random replica (load balancing)"""
        import random
        replica = random.choice(self.replicas)
        async with replica.acquire() as conn:
            return await conn.fetch(query, *args)
```

**Usage:**

```python
db = DatabasePool()
await db.initialize()

# Write goes to primary
await db.execute_write(
    "INSERT INTO users (name, email) VALUES ($1, $2)",
    "Alice", "alice@example.com"
)

# Reads from replicas (load balanced)
users = await db.execute_read("SELECT * FROM users WHERE active = true")
```

---

### 2. Master-Master (Multi-Primary)

**Two or more primaries**, each accepting writes. Replicate to each other.

```
Application
    │
    ├────────┬────────────┐
    │        │            │
    ▼        ▼            ▼
┌─────────────┐      ┌─────────────┐
│  PRIMARY 1  │◄────►│  PRIMARY 2  │
│  (Master)   │      │  (Master)   │
└──────┬──────┘      └──────┬──────┘
       │                    │
       │ Replicate          │ Replicate
       │                    │
       ▼                    ▼
   ┌─────────┐          ┌─────────┐
   │Replica 1│          │Replica 2│
   └─────────┘          └─────────┘
```

**Characteristics:**

✅ **Pros:**
- No single point of failure for writes
- Can write to nearest primary (low latency)
- Load balance writes across primaries

❌ **Cons:**
- **Write conflicts** - Same row updated on two primaries
- Complex conflict resolution needed
- Higher complexity, harder to debug
- Not all databases support well

**Write Conflict Example:**

```
Time    Primary 1                Primary 2
----    ---------                ---------
t0      balance = $100           balance = $100
t1      UPDATE: balance -= $50
t2                               UPDATE: balance -= $30
t3      balance = $50            balance = $70
t4      ← Replication →          ← Replication →
t5      ❌ CONFLICT! Which wins?
```

**Conflict Resolution Strategies:**

1. **Last Write Wins (LWW)**
   ```python
   # Use timestamp to determine winner
   if update1.timestamp > update2.timestamp:
       apply(update1)
   else:
       apply(update2)
   # Problem: Lost update! (one change discarded)
   ```

2. **Version Vectors**
   ```python
   # Track causality
   record = {
       "balance": 100,
       "version": {"primary1": 5, "primary2": 3}
   }
   # Can detect concurrent writes and require manual resolution
   ```

3. **Application-level Resolution**
   ```python
   # E.g., for banking: always conservative (reject if conflict)
   if conflict_detected:
       raise ConflictError("Manual resolution required")
   ```

**When to Use:**
- Geographic distribution (write to local primary)
- High write throughput requirements
- Can tolerate/resolve conflicts
- Example: Cassandra, CouchDB (designed for this)

---

### 3. Cascading Replication

**Replicas themselves have replicas**, forming a tree.

```
       ┌─────────────┐
       │  PRIMARY    │
       └──────┬──────┘
              │
        ┌─────┴─────┐
        │           │
        ▼           ▼
   ┌─────────┐ ┌─────────┐
   │Replica 1│ │Replica 2│
   └────┬────┘ └────┬────┘
        │           │
   ┌────┴───┐  ┌────┴───┐
   ▼        ▼  ▼        ▼
┌──────┐ ┌──────┐ ┌──────┐
│Rep1.1│ │Rep1.2│ │Rep2.1│
└──────┘ └──────┘ └──────┘
```

**Why:**
- Reduce load on primary (fewer direct connections)
- Geographic hierarchy (primary → regional replicas → local replicas)

**Downside:**
- Increased replication lag down the chain
- More points of failure

---

## Synchronous vs Asynchronous Replication

Critical trade-off between **consistency** and **performance**.

### Synchronous Replication

**Primary waits** for replica to confirm write before returning success.

```
Client                Primary              Replica
  │                     │                    │
  │──── WRITE ─────────►│                    │
  │                     │──── Replicate ────►│
  │                     │                    │ (Write to disk)
  │                     │◄─── ACK ───────────│
  │◄─── SUCCESS ────────│                    │
  │                     │                    │
```

**Characteristics:**

✅ **Pros:**
- **Strong consistency** - Replica guaranteed to have latest data
- **Durability** - Data on multiple servers before "success"
- **No data loss on primary failure** (if ≥1 replica)

❌ **Cons:**
- **Slower writes** - Must wait for network round-trip + replica write
- **Reduced availability** - If replica unreachable, writes block
- **Latency sensitive** - Bad if replica is geographically distant

**Example Latency:**

```
Same datacenter:   +1-5ms per write
Cross-region:      +50-200ms per write
Cross-continent:   +100-300ms per write
```

**When to Use:**
- **Financial transactions** (banking, payments)
- **Critical data** that cannot be lost
- Replicas in same datacenter (low latency)

**Configuration (PostgreSQL):**

```sql
-- synchronous_standby_names controls behavior
synchronous_standby_names = 'replica1, replica2'

-- Now writes wait for replica1 and replica2 to confirm
```

---

### Asynchronous Replication

**Primary returns success immediately**, replicates in background.

```
Client                Primary              Replica
  │                     │                    │
  │──── WRITE ─────────►│                    │
  │◄─── SUCCESS ────────│                    │
  │                     │                    │
  │                     │──── Replicate ────►│
  │                     │                    │ (Eventually)
```

**Characteristics:**

✅ **Pros:**
- **Fast writes** - No waiting for replicas
- **High availability** - Primary works even if replicas down
- **Geographic flexibility** - Replicas can be far away

❌ **Cons:**
- **Replication lag** - Replicas behind primary (seconds to minutes)
- **Data loss risk** - If primary crashes before replicating
- **Stale reads** - Reading from replica might return old data

**Replication Lag Example:**

```
Time    Primary        Replica (Async)
----    -------        ---------------
10:00   balance=$100   balance=$100
10:01   balance=$50    balance=$100   ← Lag!
10:02   balance=$50    balance=$100   ← Still lagging
10:03   balance=$50    balance=$50    ← Caught up
```

**Data Loss Scenario:**

```
Primary: [txn1] [txn2] [txn3] [CRASH!]
                              ↑
                      Not yet replicated
Replica: [txn1] [txn2]        ← Missing txn3!

After failover: txn3 is LOST
```

**When to Use:**
- **Social media feeds** (eventual consistency OK)
- **Analytics data** (don't need real-time)
- **Geographic replicas** (cross-continent)
- **Read-heavy workloads** (reduce load on primary)

**Most systems use asynchronous by default** (performance > consistency).

---

### Semi-Synchronous Replication

**Hybrid approach:** Wait for at least **one** replica, but not all.

```sql
-- PostgreSQL
synchronous_standby_names = 'FIRST 1 (replica1, replica2, replica3)'

-- Waits for any 1 replica to ACK
-- Other 2 replicas catch up asynchronously
```

**Benefits:**
- ✅ Durability (data on 2+ servers)
- ✅ Faster than full synchronous (only wait for 1)
- ✅ Less data loss than pure async

---

## Replication Lag

**Replication lag** is the delay between primary write and replica update.

### Measuring Lag

**PostgreSQL:**

```sql
-- On replica, check how far behind
SELECT
    EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))
    AS lag_seconds;

-- Example output: 2.5 (replica is 2.5 seconds behind)
```

**MySQL:**

```sql
SHOW SLAVE STATUS;
-- Look at Seconds_Behind_Master column
```

**Application-level:**

```python
import time

# Write timestamp to primary
await primary.execute(
    "INSERT INTO heartbeat (ts) VALUES ($1)",
    time.time()
)

# Read from replica
replica_ts = await replica.fetchval("SELECT MAX(ts) FROM heartbeat")
lag = time.time() - replica_ts
print(f"Replication lag: {lag:.2f} seconds")
```

---

### Problems Caused by Lag

#### 1. Read-Your-Writes Inconsistency

**User writes data, then reads immediately but doesn't see it.**

```python
# User updates profile
await db.execute_write("UPDATE users SET bio = 'New bio' WHERE id = 123")

# Immediately read from replica (which is lagging)
user = await db.execute_read("SELECT * FROM users WHERE id = 123")
print(user['bio'])  # ❌ Shows old bio! (Lag)

# User thinks update failed, tries again → confusion
```

**Solution: Read-Your-Writes Consistency**

```python
class DatabasePool:
    def __init__(self):
        self.primary = None
        self.replicas = []
        self.recent_writes = {}  # user_id → timestamp

    async def execute_write(self, query: str, user_id: int, *args):
        """Track when user last wrote"""
        result = await self.primary.execute(query, *args)
        self.recent_writes[user_id] = time.time()
        return result

    async def execute_read(self, query: str, user_id: int, *args):
        """Route to primary if user recently wrote"""
        WRITE_WINDOW = 5  # seconds

        last_write = self.recent_writes.get(user_id, 0)
        if time.time() - last_write < WRITE_WINDOW:
            # Read from primary (guaranteed to have user's writes)
            async with self.primary.acquire() as conn:
                return await conn.fetch(query, *args)
        else:
            # Safe to read from replica
            replica = random.choice(self.replicas)
            async with replica.acquire() as conn:
                return await conn.fetch(query, *args)
```

#### 2. Moving Backwards in Time

**User refreshes page, sees older data** (hits different replicas with different lag).

```
Request 1 → Replica A (lag: 1s)  → Shows post at 10:00:05
Request 2 → Replica B (lag: 10s) → Shows post at 10:00:00  ❌ Went back in time!
```

**Solution: Monotonic Reads**

```python
# Use sticky sessions (same user → same replica)
def get_replica_for_user(user_id: int, replicas: List):
    """Consistent hashing to same replica"""
    return replicas[hash(user_id) % len(replicas)]

# Now user always hits same replica → sees monotonic progression
```

#### 3. Causality Violations

**User A posts, User B comments, User C sees comment but not original post!**

```
Time    Primary       Replica 1     Replica 2
----    -------       ---------     ---------
10:00   Post created
10:01                 Post created
10:02   Comment added
10:03                               Comment added
10:04                               ← User C reads here
                                     Sees comment but no post! ❌
```

**Solution: Use logical timestamps or consistent prefix reads.**

---

## Failover

**Failover** is promoting a replica to primary when primary fails.

### Automatic Failover Process

```
1. Detect Primary Failure
   ↓
2. Elect New Primary (consensus algorithm)
   ↓
3. Promote Replica → New Primary
   ↓
4. Redirect Writes to New Primary
   ↓
5. Reconfigure Other Replicas
   ↓
6. Monitor for Old Primary Recovery
```

### Example (PostgreSQL + Patroni)

**Patroni** is a popular tool for HA PostgreSQL with automatic failover.

```yaml
# Patroni configuration
scope: postgres-cluster
name: node1

etcd:
  host: 127.0.0.1:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576

postgresql:
  use_pg_rewind: true
  parameters:
    max_connections: 100
    wal_level: replica
```

**Failover triggered when:**
- Primary unreachable for >30 seconds
- Automatically elects new primary via etcd consensus
- Applications automatically connect to new primary

---

### Failover Challenges

#### 1. Split-Brain

**Two nodes both think they're primary** (network partition).

```
            Network Partition
                    │
    ┌───────────────┼───────────────┐
    │               │               │
┌───▼────┐          │          ┌────▼───┐
│Primary1│          │          │Primary2│
│"I'm PM"│          │          │"I'm PM"│
└────────┘          │          └────────┘
    │               │               │
Both accept writes! │          Conflict!
```

**Solution: Use consensus (Paxos, Raft) to ensure only one primary.**

```python
# Use a distributed lock
def become_primary():
    lock = etcd_client.lock('/postgres/leader', ttl=30)
    if lock.acquire(blocking=False):
        promote_to_primary()
        # Keep renewing lock
        while True:
            lock.refresh()
            time.sleep(10)
    else:
        stay_as_replica()
```

#### 2. Data Loss on Failover

**If async replication, promoted replica might be missing recent transactions.**

```
Primary before crash: [txn1] [txn2] [txn3] [txn4]
Replica before failover: [txn1] [txn2]

After failover to replica:
  - txn3 and txn4 are LOST!
```

**Mitigation:**
- Use synchronous replication for critical data
- Set `maximum_lag_on_failover` limit
- Application-level tracking (transaction IDs)

---

## Replication Topologies

### Star Topology

```
         ┌─────────┐
         │ Primary │
         └────┬────┘
         ┌────┴────┬────┬────┐
         │         │    │    │
         ▼         ▼    ▼    ▼
     ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
     │ Rep 1 │ │ Rep 2 │ │ Rep 3 │ │ Rep 4 │
     └───────┘ └───────┘ └───────┘ └───────┘
```

**Most common.** All replicas connect directly to primary.

### Chain Topology

```
┌─────────┐   ┌───────┐   ┌───────┐   ┌───────┐
│ Primary │──►│ Rep 1 │──►│ Rep 2 │──►│ Rep 3 │
└─────────┘   └───────┘   └───────┘   └───────┘
```

**Linear chain.** Reduces load on primary but increases lag.

### Tree Topology

```
         ┌─────────┐
         │ Primary │
         └────┬────┘
         ┌────┴────┐
         │         │
         ▼         ▼
     ┌───────┐ ┌───────┐
     │ Rep 1 │ │ Rep 2 │
     └───┬───┘ └───┬───┘
       ┌─┴─┐     ┌─┴─┐
       ▼   ▼     ▼   ▼
     R1.1 R1.2 R2.1 R2.2
```

**Hierarchical.** Good for geo-distribution (regional → local).

---

## Real-World Examples

### Instagram (PostgreSQL)

**Architecture:**
- **Primary:** Single PostgreSQL primary for writes
- **Replicas:** 12+ read replicas across regions
- **Failover:** Automated with custom tooling
- **Lag:** Target <1 second

**Why:**
- Reads >> Writes (users browse more than post)
- Geographic read replicas serve users faster
- Critical writes (payments) use synchronous replication

### GitHub (MySQL)

**Architecture:**
- **Primary:** MySQL primary in US East
- **Replicas:** Regional replicas (EU, Asia, US West)
- **Orchestrator:** Automatic failover tool
- **Lag monitoring:** Alert if >5 seconds

**Incident (2018):**
- Network partition caused split-brain
- Two primaries briefly accepted writes
- Manual intervention required to reconcile

**Lesson:** Proper consensus critical for automatic failover.

---

## Best Practices

### 1. Monitor Replication Lag

```python
import prometheus_client

lag_gauge = prometheus_client.Gauge(
    'postgres_replication_lag_seconds',
    'Replication lag in seconds'
)

async def monitor_lag():
    while True:
        lag = await replica.fetchval(
            "SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))"
        )
        lag_gauge.set(lag)

        if lag > 10:
            alert("High replication lag: {lag}s")

        await asyncio.sleep(5)
```

### 2. Use Connection Pooling

```python
# Bad: Create new connection per query
async def get_user(user_id):
    conn = await asyncpg.connect("postgresql://replica/db")
    user = await conn.fetchrow("SELECT * FROM users WHERE id = $1", user_id)
    await conn.close()
    return user

# Good: Use connection pool
pool = await asyncpg.create_pool("postgresql://replica/db", min_size=10, max_size=100)

async def get_user(user_id):
    async with pool.acquire() as conn:
        return await conn.fetchrow("SELECT * FROM users WHERE id = $1", user_id)
```

### 3. Graceful Degradation

```python
async def get_user(user_id: int):
    """Try replicas first, fall back to primary"""
    try:
        # Try replica (fast, but might lag)
        return await replica_pool.fetchrow(
            "SELECT * FROM users WHERE id = $1", user_id
        )
    except Exception as e:
        logger.warning(f"Replica failed: {e}, falling back to primary")
        # Fall back to primary
        return await primary_pool.fetchrow(
            "SELECT * FROM users WHERE id = $1", user_id
        )
```

### 4. Test Failover Regularly

**Chaos engineering:** Regularly kill primary to ensure failover works.

```bash
# Simulate primary failure
docker stop postgres-primary

# Verify:
# - Replica promoted to primary within 30s
# - Applications automatically connect to new primary
# - Zero or minimal downtime
```

---

## Decision Matrix

### When to Use Replication

✅ **Use replication when:**
- Read traffic >> write traffic (10:1 or higher)
- Need high availability (99.9%+)
- Need geographic distribution
- Running analytics on replica (don't impact primary)

❌ **Don't rely solely on replication when:**
- Write-heavy workload (need sharding instead)
- Need strong consistency everywhere (use synchronous or Spanner-like)
- Single-server is sufficient (keep it simple)

### Synchronous vs Asynchronous

| Requirement | Synchronous | Asynchronous |
|-------------|-------------|--------------|
| Zero data loss | ✅ | ❌ |
| Fast writes | ❌ | ✅ |
| Strong consistency | ✅ | ❌ |
| Geographic replicas | ❌ | ✅ |
| High availability | ⚠️ (replica must be up) | ✅ |

---

## Summary

**Replication** is essential for:
- ✅ Scaling reads (distribute across replicas)
- ✅ High availability (failover to replica)
- ✅ Disaster recovery (geographic redundancy)

**Key Patterns:**
- **Master-Slave** (most common) - Single primary, multiple replicas
- **Master-Master** (complex) - Multiple primaries, conflict resolution
- **Cascading** (hierarchical) - Replicas of replicas

**Replication Modes:**
- **Synchronous** - Strong consistency, slower writes
- **Asynchronous** - Fast writes, replication lag

**Challenges:**
- Replication lag → stale reads, read-your-writes issues
- Failover complexity → split-brain, data loss risks
- Monitoring critical → lag, health checks, alerting

**Real-World:**
- Instagram: 12+ read replicas for global scale
- GitHub: Regional replicas with automated failover
- Most systems: Async replication by default, sync for critical data

**Next:** Learn about [Database Sharding](database-sharding.md) for scaling writes horizontally.
