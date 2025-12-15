# CAP Theorem

## Overview

The **CAP Theorem** states that in a **distributed database**, you can only guarantee **two out of three** properties: **Consistency**, **Availability**, and **Partition Tolerance**.

**Key Question:** "In a distributed system with network failures, what guarantees can you make?"

**Short Answer:**
- **C**onsistency: All nodes see the same data at the same time
- **A**vailability: Every request gets a response (success or failure)
- **P**artition Tolerance: System works despite network failures
- **Pick 2:** You cannot have all three during a network partition

---

## The Three Properties

### Consistency (C)

**All nodes see the same data at the same time.**

```
Client writes: balance = $100

┌─────────┐      ┌─────────┐      ┌─────────┐
│ Node 1  │      │ Node 2  │      │ Node 3  │
│ $100    │  =   │ $100    │  =   │ $100    │
└─────────┘      └─────────┘      └─────────┘

✅ Consistent: All reads return $100
```

**Inconsistent Example:**

```
Client writes $100 to Node 1 (not yet replicated)

┌─────────┐      ┌─────────┐      ┌─────────┐
│ Node 1  │      │ Node 2  │      │ Node 3  │
│ $100    │  ≠   │ $50     │  ≠   │ $50     │
└─────────┘      └─────────┘      └─────────┘
    ↑
  Write here

Read from Node 2: $50  ❌ Stale data!
Read from Node 1: $100 ✅ Latest data

❌ Inconsistent: Reads return different values
```

**Strong Consistency:**

- All reads return the most recent write
- No stale data
- Example: Traditional SQL databases (PostgreSQL with synchronous replication)

**Eventual Consistency:**

- Reads may return stale data temporarily
- All nodes eventually converge to same value
- Example: DynamoDB, Cassandra, DNS

---

### Availability (A)

**Every request gets a response** (even if data is stale).

```
Client requests data

┌─────────┐
│ Node 1  │  ← Working
│ $100    │  → Returns $100  ✅
└─────────┘

┌─────────┐
│ Node 2  │  ← Working
│ $100    │  → Returns $100  ✅
└─────────┘

┌─────────┐
│ Node 3  │  ← CRASHED
│   💥    │  → (Unreachable)
└─────────┘

System still available (2/3 nodes respond)
```

**Unavailable Example:**

```
Client writes to Node 1
System waits for Node 3 to confirm (synchronous replication)

┌─────────┐      ┌─────────┐      ┌─────────┐
│ Node 1  │      │ Node 2  │      │ Node 3  │
│ Write   │ ───► │ Replicate│ ───► │  💥    │
└─────────┘      └─────────┘      └─────────┘
     ↓                                  ↑
     └─────── Waiting forever ──────────┘

Client: ⏳ Timeout after 30s  ❌ Unavailable
```

**High Availability:**

- System remains operational despite failures
- Requests don't timeout or block
- Example: Cassandra, DynamoDB (keep responding even if nodes fail)

---

### Partition Tolerance (P)

**System continues to work despite network partitions** (nodes can't communicate).

```
Normal Network:
┌─────────┐      ┌─────────┐      ┌─────────┐
│ Node 1  │◄────►│ Node 2  │◄────►│ Node 3  │
└─────────┘      └─────────┘      └─────────┘

Network Partition (split-brain):
┌─────────┐      ┌─────────┐      ┌─────────┐
│ Node 1  │      │ Node 2  │◄────►│ Node 3  │
└─────────┘      └─────────┘      └─────────┘
     ↑                |                 |
     |     ❌ Network partition        |
     └──────────────────────────────────┘
     (Node 1 can't reach Node 2/3)

Partition Tolerant:
  ✅ System keeps working on both sides
```

**Why Partition Tolerance is Non-Negotiable:**

In distributed systems, **network failures are inevitable**:
- Switch failures
- Cable cuts
- Datacenter connectivity issues
- Cloud provider outages

**Therefore, in practice:**

```
CAP Theorem in Reality:
  Since P (Partition Tolerance) is required,
  You must choose between C and A:

  CP (Consistency + Partition Tolerance)
    or
  AP (Availability + Partition Tolerance)
```

---

## CP Systems (Consistency + Partition Tolerance)

**During a network partition, sacrifice availability to maintain consistency.**

### Example: Banking System

```
Scenario: Money transfer
  Account A: $100
  Account B: $50
  Transfer $30 from A → B

Network partition occurs:

┌─────────┐            ❌           ┌─────────┐
│ Node 1  │      Network Partition  │ Node 2  │
│ A: $100 │            ❌           │ A: $100 │
│ B: $50  │                         │ B: $50  │
└─────────┘                         └─────────┘

Client writes: Transfer $30

Node 1 Decision (CP System):
  "I can't reach Node 2 to ensure consistency.
   I will REJECT this write to avoid inconsistency."

Response: ❌ "Service Temporarily Unavailable" (503 error)

✅ Consistency maintained (no split-brain)
❌ Availability sacrificed (request rejected)
```

**CP Databases:**

| Database | Type | Use Case |
|----------|------|----------|
| **PostgreSQL** (with sync replication) | SQL | Financial transactions |
| **MongoDB** (with majority writes) | Document | Critical data |
| **HBase** | Wide-column | Strong consistency required |
| **Redis** (with replication) | Key-value | Caching with consistency |
| **ZooKeeper** | Coordination | Distributed locks, config |
| **etcd** | Key-value | Kubernetes, service discovery |

**Characteristics:**

✅ **Strong consistency** - Reads always return latest write
✅ **ACID transactions** - Banking, payments
✅ **No data loss** - Every write confirmed before success

❌ **Lower availability** - Rejects requests during partitions
❌ **Higher latency** - Must wait for consensus (Paxos/Raft)
❌ **Single point of failure** (if no partition tolerance design)

**Example Code (MongoDB):**

```python
import pymongo

# CP system: Require majority write concern
client = pymongo.MongoClient("mongodb://localhost:27017/")
db = client["banking"]
accounts = db["accounts"]

# Transfer $30 from A → B
def transfer(from_account, to_account, amount):
    # Use write concern: majority (CP behavior)
    result = accounts.update_one(
        {"account_id": from_account, "balance": {"$gte": amount}},
        {"$inc": {"balance": -amount}},
        write_concern=pymongo.WriteConcern(w="majority")  # Wait for majority
    )

    if result.modified_count == 0:
        raise Exception("Insufficient funds or partition!")

    # Second write also requires majority
    accounts.update_one(
        {"account_id": to_account},
        {"$inc": {"balance": amount}},
        write_concern=pymongo.WriteConcern(w="majority")
    )

# During partition:
# If majority of nodes unreachable → Exception raised ❌ (Unavailable)
# But consistency guaranteed ✅
```

---

## AP Systems (Availability + Partition Tolerance)

**During a network partition, sacrifice consistency to maintain availability.**

### Example: Social Media Feed

```
Scenario: User posts "Hello World"

Network partition occurs:

┌─────────┐            ❌           ┌─────────┐
│ Node 1  │      Network Partition  │ Node 2  │
│ (US)    │            ❌           │ (EU)    │
└─────────┘                         └─────────┘

US Client writes to Node 1: "Hello World"

Node 1 Decision (AP System):
  "I can't reach Node 2, but I'll accept the write anyway.
   Node 2 will catch up eventually (eventual consistency)."

Response: ✅ "Post created successfully!"

EU Client reads from Node 2:
  (Doesn't see "Hello World" yet... will appear in a few seconds)

✅ Availability maintained (write accepted)
❌ Consistency sacrificed (temporary inconsistency)
```

**AP Databases:**

| Database | Type | Use Case |
|----------|------|----------|
| **Cassandra** | Wide-column | Social media, IoT |
| **DynamoDB** | Key-value | Session storage, carts |
| **CouchDB** | Document | Offline-first apps |
| **Riak** | Key-value | High availability storage |
| **Voldemort** | Key-value | LinkedIn's data store |

**Characteristics:**

✅ **High availability** - Always accepts reads/writes
✅ **Low latency** - No waiting for consensus
✅ **Partition tolerant** - Works during network failures

❌ **Eventual consistency** - Stale reads possible
❌ **Conflict resolution needed** - Concurrent writes may conflict
❌ **No ACID guarantees** - Not suitable for banking

**Example Code (Cassandra):**

```python
from cassandra.cluster import Cluster

# AP system: Write to any node, eventual consistency
cluster = Cluster(['node1', 'node2', 'node3'])
session = cluster.connect('social_media')

# Post creation (AP behavior)
def create_post(user_id, content):
    # Write to any available node
    # Doesn't wait for all replicas
    session.execute(
        "INSERT INTO posts (post_id, user_id, content, created_at) VALUES (uuid(), %s, %s, toTimestamp(now()))",
        (user_id, content)
    )
    # ✅ Returns immediately (Available)
    # ❌ Other nodes might not see it yet (Eventually Consistent)

# During partition:
# Write succeeds on Node 1 ✅ (Available)
# Node 2 doesn't see it yet ❌ (Inconsistent)
# After partition heals: Nodes sync (Eventual Consistency) ✅
```

---

## The Trade-Off in Action

### Scenario: E-commerce Shopping Cart

**Option 1: CP System (Strong Consistency)**

```python
# Add item to cart
def add_to_cart(user_id, item_id):
    try:
        # Wait for majority of nodes to confirm
        db.execute(
            "INSERT INTO cart_items (user_id, item_id) VALUES (%s, %s)",
            (user_id, item_id),
            consistency_level="QUORUM"  # Majority
        )
        return "Item added"
    except Exception:
        # Partition: Can't reach majority
        return "Service unavailable, try again"  ❌

# ✅ Guarantee: Cart is consistent across all nodes
# ❌ Downside: During partition, users can't add to cart (poor UX)
```

**Option 2: AP System (High Availability)**

```python
# Add item to cart
def add_to_cart(user_id, item_id):
    # Write to any available node
    db.execute(
        "INSERT INTO cart_items (user_id, item_id) VALUES (%s, %s)",
        (user_id, item_id),
        consistency_level="ONE"  # Just 1 node
    )
    return "Item added"  ✅ Always succeeds

# During partition:
# User adds item on Node 1 (writes to Node 1)
# User refreshes, routed to Node 2 (doesn't see item yet)
# → Confusing UX, but at least system is available

# After partition heals: Nodes sync, item appears ✅
```

**Hybrid Approach:**

```python
# Best of both worlds:
# - Writes: AP (high availability)
# - Reads: CP (strong consistency for checkout)

def add_to_cart(user_id, item_id):
    # Write with eventual consistency (fast)
    db.execute(..., consistency_level="ONE")
    return "Item added"  ✅

def get_cart_for_checkout(user_id):
    # Read with strong consistency (accurate)
    return db.execute(..., consistency_level="QUORUM")

# Users can always add items (AP)
# But checkout reads latest data (CP)
```

---

## PACELC Theorem (Extended CAP)

**CAP only describes behavior during partitions.** What about normal operation?

**PACELC:**

```
If Partition (P):
  Choose between Availability (A) or Consistency (C)
Else (E):
  Choose between Latency (L) or Consistency (C)
```

**In normal operation (no partition):**

- **Low latency** → Accept stale reads (eventual consistency)
- **Strong consistency** → Wait for all replicas (higher latency)

**Examples:**

| Database | During Partition | Normal Operation | Classification |
|----------|------------------|------------------|----------------|
| **DynamoDB** | A (available) | L (low latency) | PA/EL |
| **Cassandra** | A (available) | L (low latency) | PA/EL |
| **MongoDB** | C (consistent) | L (low latency) | PC/EL |
| **PostgreSQL** | C (consistent) | C (consistent) | PC/EC |
| **Cosmos DB** | Tunable | Tunable | Configurable |

---

## Real-World Examples

### Amazon DynamoDB (AP)

**Design Goal:** Shopping cart must always be available (Black Friday!)

**Behavior:**

```
Black Friday: Millions of users adding items to cart

Network partition between US-East and US-West:

US-East Node:
  User adds item A → ✅ Success

US-West Node:
  User adds item B → ✅ Success

Temporary inconsistency:
  US-East sees: [A]
  US-West sees: [B]

After partition heals:
  Merge: [A, B]  ✅ Both items in cart

✅ High availability (both writes succeeded)
❌ Temporary inconsistency (brief split view)
```

**Why AP:**
- ❌ Can't afford downtime during peak sales
- ✅ Eventual consistency acceptable (extra items in cart OK, just remove later)

---

### Google Spanner (CP... sort of)

**Design Goal:** Global consistency for financial transactions

**Behavior:**

```
Write to Spanner:
  1. Propose transaction to majority (Paxos)
  2. Wait for majority to ack
  3. Return success to client

During partition:
  If can't reach majority → ❌ Reject write (CP)

But: Uses atomic clocks (TrueTime) to achieve near-global consistency
```

**Why CP:**
- ✅ Financial data requires strong consistency
- ❌ Cannot tolerate split-brain (double spending)

**Innovation:**
- Uses GPS + atomic clocks to reduce latency of consistency
- Still CP, but much faster than traditional CP systems

---

### Instagram (Hybrid)

**Posts: AP (Cassandra)**

```python
# User posts photo → Available
# Even during partition, post accepted
# Eventual consistency OK (feed doesn't need instant consistency)
```

**Payments: CP (PostgreSQL)**

```python
# User pays for ad → Consistent
# Must wait for majority to confirm
# Strong consistency required (no double-charging)
```

**Strategy:** Use different databases for different use cases.

---

## Choosing Between CP and AP

### Choose CP When:

✅ **Correctness > Availability**
- Banking, payments (no double-spending)
- Inventory management (no overselling)
- Voting systems (no double-voting)
- Critical data (medical records)

✅ **Strong consistency required**
- Transactions across accounts
- Stock trading
- Reservation systems

**Example Use Cases:**
- Stripe (payment processing)
- Stock exchanges
- Airlines (seat reservations)

---

### Choose AP When:

✅ **Availability > Correctness**
- Social media feeds (stale data OK)
- Product catalog (brief inconsistency OK)
- Session storage (can tolerate loss)
- Logging, metrics (eventual consistency OK)

✅ **Eventual consistency acceptable**
- DNS (propagation delay OK)
- Content delivery (CDN)
- Shopping cart (can merge conflicts)

**Example Use Cases:**
- Facebook (posts, likes)
- Amazon (shopping cart)
- Netflix (viewing history)

---

## Implementing CAP Trade-offs

### Tunable Consistency (Cassandra)

**Flexibility:** Choose consistency level per query.

```python
from cassandra.cluster import Cluster, ConsistencyLevel

session = cluster.connect()

# AP behavior: Write to 1 node (fast, available)
session.execute(
    "INSERT INTO posts (id, content) VALUES (uuid(), %s)",
    (content,),
    consistency_level=ConsistencyLevel.ONE  # AP
)

# CP behavior: Write to majority (consistent)
session.execute(
    "INSERT INTO payments (id, amount) VALUES (uuid(), %s)",
    (amount,),
    consistency_level=ConsistencyLevel.QUORUM  # CP
)

# Strong consistency: Write to ALL nodes (slowest)
session.execute(
    "INSERT INTO critical_data (...) VALUES (...)",
    consistency_level=ConsistencyLevel.ALL  # Strong CP
)
```

**Consistency Levels:**

| Level | Replicas | Behavior | CAP |
|-------|----------|----------|-----|
| ONE | 1 | Fastest, least consistent | AP |
| QUORUM | Majority (N/2 + 1) | Balanced | CP |
| ALL | All replicas | Slowest, most consistent | Strong CP |

---

### Read Repair (Eventual Consistency)

**How AP systems converge:**

```python
# Read from 3 replicas
Node 1: balance = $100  (latest)
Node 2: balance = $90   (stale)
Node 3: balance = $90   (stale)

# Read repair process:
1. Return majority value to client: $90
2. Background: Update Node 1 → $90 (or update Node 2/3 → $100 based on timestamp)
3. Eventually all nodes consistent
```

**Anti-Entropy:**

```python
# Periodic background sync
async def anti_entropy():
    while True:
        for replica in all_replicas:
            for other_replica in all_replicas:
                if replica != other_replica:
                    sync_data(replica, other_replica)
        await asyncio.sleep(3600)  # Every hour
```

---

## Summary

**CAP Theorem:**
- **C**onsistency: All nodes see same data
- **A**vailability: Every request gets response
- **P**artition Tolerance: System works despite network failures

**Key Insight:**
- ❌ Cannot have all three during partition
- ✅ In distributed systems, P is required
- ⚖️ Choose between **CP** or **AP**

**CP Systems (Consistency + Partition Tolerance):**
- Reject requests during partition to maintain consistency
- Examples: PostgreSQL, MongoDB, HBase, ZooKeeper
- Use cases: Banking, payments, critical data

**AP Systems (Availability + Partition Tolerance):**
- Accept requests during partition, eventual consistency
- Examples: Cassandra, DynamoDB, CouchDB
- Use cases: Social media, shopping cart, logs

**Real-World Strategy:**
- Use **both** CP and AP databases for different use cases
- Tunable consistency (Cassandra, Cosmos DB)
- PACELC: Consider latency vs consistency trade-off in normal operation

**Next Steps:**
- Understand your application's consistency requirements
- Choose appropriate database based on CAP needs
- Monitor and tune consistency levels based on usage

---

**Related Topics:**
- [SQL vs NoSQL](sql-vs-nosql.md) - Database paradigms
- [Database Replication](database-replication.md) - How nodes sync
- [Database Sharding](database-sharding.md) - Horizontal scaling
