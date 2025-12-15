# Design a Distributed Key-Value Store

## Table of Contents
- [Problem Statement](#problem-statement)
- [Requirements](#requirements)
- [Capacity Estimation](#capacity-estimation)
- [High-Level Design](#high-level-design)
- [Core Components](#core-components)
- [Data Partitioning](#data-partitioning)
- [Replication](#replication)
- [CAP Theorem Trade-offs](#cap-theorem-trade-offs)
- [Conflict Resolution](#conflict-resolution)
- [Data Persistence](#data-persistence)
- [API Design](#api-design)
- [Implementation](#implementation)
- [Failure Handling](#failure-handling)
- [Optimizations](#optimizations)
- [Trade-offs](#trade-offs)
- [Real-World Examples](#real-world-examples)
- [Interview Tips](#interview-tips)

---

## Problem Statement

Design a **distributed key-value store** that can:
- Store and retrieve data using a key-value interface
- Handle billions of key-value pairs
- Provide high availability and fault tolerance
- Scale horizontally across multiple nodes
- Ensure data durability and consistency

**Similar to**: Amazon DynamoDB, Redis Cluster, Apache Cassandra, Riak

---

## Requirements

### Functional Requirements

1. **put(key, value)**: Store a value associated with a key
2. **get(key)**: Retrieve the value for a given key
3. **delete(key)**: Remove a key-value pair
4. Support for large datasets (billions of keys)
5. Support for various data types (strings, numbers, objects)

### Non-Functional Requirements

1. **Availability**: Always able to accept reads/writes (99.99% uptime)
2. **Scalability**: Horizontal scaling to handle growth
3. **Fault Tolerance**: Survive node failures
4. **Durability**: Data persists even with failures
5. **Low Latency**: < 10ms for reads, < 20ms for writes
6. **Partition Tolerance**: Continue operating during network partitions

### Extended Requirements

1. **Configurable consistency**: Trade-off between consistency and availability
2. **Data versioning**: Track multiple versions of values
3. **Expiration/TTL**: Automatic cleanup of old data
4. **Atomic operations**: Increment, decrement, compare-and-swap

---

## Capacity Estimation

### Assumptions

- **Total keys**: 10 billion
- **Average key size**: 20 bytes
- **Average value size**: 1 KB
- **Total storage**: 10B × (20B + 1KB) ≈ 10 TB
- **QPS**: 100,000 requests/second
- **Read:Write ratio**: 80:20

### Storage Calculation

```
Total data = 10 billion × 1 KB = 10 TB
With replication factor 3 = 30 TB
With overhead (metadata, tombstones) = 40 TB
```

### Server Calculation

```
Assuming 1 TB per server (for performance)
Servers needed = 40 TB / 1 TB = 40 servers

For load balancing:
100K QPS / 10K QPS per server = 10 servers minimum
Use 40+ servers for both storage and throughput
```

---

## High-Level Design

```
┌──────────────────────────────────────────────────────────────┐
│                         Client Layer                          │
└────────────────┬─────────────────────────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────────────────────────┐
│                      Load Balancer / API Gateway               │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 ↓
┌────────────────────────────────────────────────────────────────┐
│                     Coordinator Nodes                          │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                  │
│  │  Node 1  │   │  Node 2  │   │  Node 3  │                  │
│  │ (Leader) │   │          │   │          │                  │
│  └─────┬────┘   └─────┬────┘   └─────┬────┘                  │
└────────┼──────────────┼──────────────┼────────────────────────┘
         │              │              │
         ↓              ↓              ↓
┌────────────────────────────────────────────────────────────────┐
│                  Consistent Hash Ring                          │
│         (Determines which nodes store which keys)              │
└────────┬───────────────────────────────────────────────────────┘
         │
         ↓
┌────────────────────────────────────────────────────────────────┐
│                      Storage Nodes                             │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐              │
│  │ Node A │  │ Node B │  │ Node C │  │ Node D │  ...         │
│  │        │  │        │  │        │  │        │              │
│  │ Primary│  │ Replica│  │ Replica│  │ Primary│              │
│  └────────┘  └────────┘  └────────┘  └────────┘              │
└────────────────────────────────────────────────────────────────┘
         │              │
         ↓              ↓
   ┌─────────┐    ┌─────────┐
   │ Memtable│    │ SSTables│
   │  (RAM)  │    │ (Disk)  │
   └─────────┘    └─────────┘
```

---

## Core Components

### 1. Client Library/SDK

Handles request routing and failover.

```python
class KVStoreClient:
    def __init__(self, coordinator_nodes):
        self.coordinators = coordinator_nodes
        self.consistent_hash = ConsistentHash()

    def put(self, key, value, consistency_level='QUORUM'):
        """Store key-value pair"""
        nodes = self.consistent_hash.get_nodes(key, n=3)
        responses = []

        for node in nodes:
            try:
                response = node.put(key, value)
                responses.append(response)
            except Exception as e:
                print(f"Node {node} failed: {e}")

        # Wait for quorum
        if consistency_level == 'QUORUM':
            required = len(nodes) // 2 + 1
            return len(responses) >= required
        elif consistency_level == 'ONE':
            return len(responses) >= 1
        elif consistency_level == 'ALL':
            return len(responses) == len(nodes)

    def get(self, key, consistency_level='QUORUM'):
        """Retrieve value for key"""
        nodes = self.consistent_hash.get_nodes(key, n=3)
        responses = []

        for node in nodes:
            try:
                value, version = node.get(key)
                responses.append((value, version))
            except Exception:
                continue

        # Resolve conflicts if multiple versions exist
        return self.resolve_conflicts(responses)
```

### 2. Coordinator Node

Routes requests to appropriate storage nodes.

```python
class CoordinatorNode:
    def __init__(self, node_id, hash_ring):
        self.node_id = node_id
        self.hash_ring = hash_ring

    def handle_put(self, key, value, consistency_level='QUORUM'):
        """
        1. Hash the key to determine storage nodes
        2. Send write requests to N replicas
        3. Wait for W acknowledgments (based on consistency level)
        4. Return success/failure
        """
        storage_nodes = self.hash_ring.get_nodes(key, n=3)

        # Sloppy quorum: Use next available nodes if preferred nodes down
        write_count = 0
        for node in storage_nodes:
            try:
                success = node.store(key, value)
                if success:
                    write_count += 1
                    if self.satisfies_consistency(write_count, consistency_level):
                        return True
            except NodeUnavailable:
                # Try next node (hinted handoff)
                continue

        return False

    def handle_get(self, key, consistency_level='QUORUM'):
        """
        1. Hash the key to find storage nodes
        2. Send read requests to N replicas
        3. Wait for R responses
        4. Resolve conflicts (if any)
        5. Return latest value
        """
        storage_nodes = self.hash_ring.get_nodes(key, n=3)

        responses = []
        for node in storage_nodes:
            try:
                data = node.retrieve(key)
                responses.append(data)
                if len(responses) >= self.get_read_quorum(consistency_level):
                    break
            except NodeUnavailable:
                continue

        # Return reconciled value (vector clock comparison)
        return self.reconcile(responses)
```

### 3. Storage Node

Stores actual key-value pairs.

```python
class StorageNode:
    def __init__(self, node_id):
        self.node_id = node_id
        self.memtable = {}  # In-memory cache
        self.sstables = []  # On-disk sorted string tables
        self.wal = WriteAheadLog()  # For durability
        self.bloom_filter = BloomFilter()

    def store(self, key, value, vector_clock=None):
        """
        1. Write to WAL for durability
        2. Write to memtable (in-memory)
        3. When memtable full, flush to SSTable (disk)
        """
        # Durability: Write-ahead log
        self.wal.append(key, value, vector_clock)

        # Write to memtable
        self.memtable[key] = {
            'value': value,
            'timestamp': time.time(),
            'vector_clock': vector_clock or self.increment_clock(key)
        }

        # Flush if memtable exceeds threshold
        if len(self.memtable) > MEMTABLE_THRESHOLD:
            self.flush_to_disk()

        # Update bloom filter
        self.bloom_filter.add(key)

        return True

    def retrieve(self, key):
        """
        1. Check memtable
        2. Check bloom filter (avoid disk reads)
        3. Search SSTables (newest to oldest)
        """
        # Check memtable first (fastest)
        if key in self.memtable:
            return self.memtable[key]

        # Check bloom filter before disk access
        if not self.bloom_filter.may_contain(key):
            return None

        # Search SSTables (newest to oldest)
        for sstable in reversed(self.sstables):
            value = sstable.get(key)
            if value is not None:
                return value

        return None

    def flush_to_disk(self):
        """Flush memtable to disk as SSTable"""
        sstable = SSTable(sorted(self.memtable.items()))
        self.sstables.append(sstable)
        self.memtable.clear()

    def compact(self):
        """Merge multiple SSTables to remove duplicates/tombstones"""
        if len(self.sstables) < COMPACTION_THRESHOLD:
            return

        # Merge SSTables (keep latest version of each key)
        merged = self.merge_sstables(self.sstables)
        self.sstables = [merged]
```

---

## Data Partitioning

### Consistent Hashing

Distribute data evenly across nodes and minimize data movement when nodes are added/removed.

```python
import hashlib

class ConsistentHash:
    def __init__(self, virtual_nodes=150):
        self.virtual_nodes = virtual_nodes
        self.ring = {}  # hash_value -> node
        self.sorted_keys = []

    def add_node(self, node):
        """Add a node with virtual nodes"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{node.id}:{i}"
            hash_val = self.hash(virtual_key)
            self.ring[hash_val] = node
            self.sorted_keys.append(hash_val)

        self.sorted_keys.sort()

    def remove_node(self, node):
        """Remove a node and its virtual nodes"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{node.id}:{i}"
            hash_val = self.hash(virtual_key)
            del self.ring[hash_val]
            self.sorted_keys.remove(hash_val)

    def get_node(self, key):
        """Find the node responsible for this key"""
        if not self.ring:
            return None

        hash_val = self.hash(key)

        # Find first node clockwise from hash position
        for ring_hash in self.sorted_keys:
            if hash_val <= ring_hash:
                return self.ring[ring_hash]

        # Wrap around to first node
        return self.ring[self.sorted_keys[0]]

    def get_nodes(self, key, n=3):
        """Get N nodes for replication"""
        if not self.ring:
            return []

        hash_val = self.hash(key)
        nodes = []
        idx = bisect.bisect(self.sorted_keys, hash_val)

        # Collect unique nodes
        seen_nodes = set()
        while len(nodes) < n:
            if idx >= len(self.sorted_keys):
                idx = 0
            node = self.ring[self.sorted_keys[idx]]
            if node not in seen_nodes:
                nodes.append(node)
                seen_nodes.add(node)
            idx += 1

        return nodes

    def hash(self, key):
        """Hash function (MD5)"""
        return int(hashlib.md5(key.encode()).hexdigest(), 16)
```

**Benefits:**
- Minimal data movement when nodes added/removed
- Even distribution with virtual nodes
- Automatic load balancing

---

## Replication

### Replication Factor

Store data on multiple nodes for availability and durability.

```
Replication Factor N = 3 (typical)

Key "user:123" hashed to position X on ring
Stored on nodes: A, B, C (next 3 nodes clockwise)

Node A: Primary replica
Node B: Secondary replica
Node C: Tertiary replica
```

### Replication Strategies

#### 1. Synchronous Replication
```python
def synchronous_replication(key, value, replicas):
    """Wait for all replicas to acknowledge"""
    for replica in replicas:
        replica.write(key, value)  # Blocking
    return True
```

**Pros**: Strong consistency
**Cons**: High latency, low availability

#### 2. Asynchronous Replication
```python
def asynchronous_replication(key, value, replicas):
    """Write to primary, replicate in background"""
    primary = replicas[0]
    primary.write(key, value)

    # Replicate asynchronously
    for replica in replicas[1:]:
        threading.Thread(target=replica.write, args=(key, value)).start()

    return True
```

**Pros**: Low latency, high availability
**Cons**: Eventual consistency

#### 3. Quorum-based Replication (Dynamo-style)

```python
def quorum_write(key, value, replicas, W):
    """Wait for W replicas to acknowledge"""
    acks = 0
    for replica in replicas:
        try:
            if replica.write(key, value):
                acks += 1
                if acks >= W:
                    return True
        except:
            continue
    return False

def quorum_read(key, replicas, R):
    """Read from R replicas and resolve conflicts"""
    responses = []
    for replica in replicas:
        try:
            value = replica.read(key)
            responses.append(value)
            if len(responses) >= R:
                break
        except:
            continue

    return resolve_conflicts(responses)
```

**Configuration:**
- N = 3 (replication factor)
- W = 2 (write quorum)
- R = 2 (read quorum)

**Consistency guarantee:** If R + W > N, you get strong consistency

**Examples:**
- N=3, W=2, R=2 → Strong consistency
- N=3, W=1, R=1 → Eventual consistency (high availability)
- N=3, W=3, R=1 → Strong consistency for writes

---

## CAP Theorem Trade-offs

The CAP theorem states you can only have 2 of 3:
- **Consistency**: All nodes see the same data at the same time
- **Availability**: Every request receives a response
- **Partition Tolerance**: System continues despite network partitions

### Trade-off Spectrum

```
CP (Consistency + Partition Tolerance)
├─ HBase, MongoDB (strong consistency)
├─ Sacrifice availability during partitions
└─ Wait for quorum before responding

AP (Availability + Partition Tolerance)
├─ Cassandra, DynamoDB (eventual consistency)
├─ Always respond (even with stale data)
└─ Resolve conflicts later
```

### DynamoDB's Approach (AP with tunable consistency)

```python
class DynamoConfig:
    def __init__(self):
        self.N = 3  # Replication factor
        self.W = 2  # Write quorum
        self.R = 2  # Read quorum

    def strong_consistency(self):
        """R + W > N"""
        self.W = 2
        self.R = 2  # 2 + 2 > 3 ✓

    def eventual_consistency(self):
        """Optimize for availability"""
        self.W = 1
        self.R = 1  # 1 + 1 < 3 (may read stale data)

    def write_optimized(self):
        """Fast writes, slower reads"""
        self.W = 1
        self.R = 3  # 1 + 3 > 3 ✓

    def read_optimized(self):
        """Fast reads, slower writes"""
        self.W = 3
        self.R = 1  # 3 + 1 > 3 ✓
```

---

## Conflict Resolution

### Vector Clocks

Track causality and detect conflicts.

```python
class VectorClock:
    def __init__(self):
        self.clock = {}  # node_id -> counter

    def increment(self, node_id):
        """Increment counter for this node"""
        self.clock[node_id] = self.clock.get(node_id, 0) + 1

    def merge(self, other):
        """Merge two vector clocks"""
        merged = VectorClock()
        all_nodes = set(self.clock.keys()) | set(other.clock.keys())
        for node in all_nodes:
            merged.clock[node] = max(
                self.clock.get(node, 0),
                other.clock.get(node, 0)
            )
        return merged

    def compare(self, other):
        """
        Returns:
        - 'before' if self happened before other
        - 'after' if self happened after other
        - 'concurrent' if concurrent (conflict!)
        """
        self_greater = False
        other_greater = False

        all_nodes = set(self.clock.keys()) | set(other.clock.keys())
        for node in all_nodes:
            self_val = self.clock.get(node, 0)
            other_val = other.clock.get(node, 0)

            if self_val > other_val:
                self_greater = True
            elif other_val > self_val:
                other_greater = True

        if self_greater and not other_greater:
            return 'after'
        elif other_greater and not self_greater:
            return 'before'
        else:
            return 'concurrent'

# Example usage
def resolve_with_vector_clock(responses):
    """
    Responses: [(value, vector_clock), ...]
    """
    if len(responses) == 1:
        return responses[0][0]

    # Find all concurrent versions
    concurrent_values = []
    for value, clock in responses:
        is_concurrent = False
        for other_value, other_clock in responses:
            if clock.compare(other_clock) == 'concurrent':
                is_concurrent = True
                break
        if is_concurrent:
            concurrent_values.append(value)

    if len(concurrent_values) == 1:
        return concurrent_values[0]

    # Multiple concurrent versions - need application-level resolution
    # Options:
    # 1. Last-write-wins (based on timestamp)
    # 2. Return all versions to client
    # 3. Custom merge logic

    return concurrent_values  # Return all for client to resolve
```

### Last-Write-Wins (LWW)

Simpler but can lose updates.

```python
def last_write_wins(responses):
    """Choose value with latest timestamp"""
    latest = max(responses, key=lambda x: x['timestamp'])
    return latest['value']
```

### Read Repair

Fix inconsistencies during reads.

```python
def read_with_repair(key, storage_nodes):
    """
    1. Read from multiple replicas
    2. Detect inconsistencies
    3. Update stale replicas
    """
    responses = []
    for node in storage_nodes:
        value, version = node.get(key)
        responses.append((node, value, version))

    # Find latest version
    latest_value, latest_version = resolve_conflicts(
        [(v, ver) for _, v, ver in responses]
    )

    # Repair stale replicas
    for node, value, version in responses:
        if version < latest_version:
            node.put(key, latest_value, latest_version)

    return latest_value
```

---

## Data Persistence

### LSM Tree (Log-Structured Merge Tree)

Used by Cassandra, RocksDB, LevelDB.

```
Writes:
1. Write to WAL (disk, sequential)
2. Write to Memtable (memory, fast)
3. When memtable full → flush to SSTable (disk)
4. Periodically compact SSTables

Reads:
1. Check memtable (memory)
2. Check bloom filters (avoid disk reads)
3. Search SSTables (disk, newest to oldest)
```

```python
class LSMTree:
    def __init__(self):
        self.wal = WriteAheadLog('wal.log')
        self.memtable = {}  # Key -> (value, timestamp, vector_clock)
        self.sstables = []
        self.bloom_filter = BloomFilter(expected_items=1000000)

    def put(self, key, value):
        """Write operation"""
        timestamp = time.time()
        vector_clock = self.get_vector_clock(key)

        # 1. Write to WAL (durability)
        self.wal.append({
            'key': key,
            'value': value,
            'timestamp': timestamp,
            'vector_clock': vector_clock
        })

        # 2. Write to memtable
        self.memtable[key] = (value, timestamp, vector_clock)

        # 3. Flush if needed
        if len(self.memtable) > 10000:  # Threshold
            self.flush()

    def get(self, key):
        """Read operation"""
        # 1. Check memtable
        if key in self.memtable:
            return self.memtable[key][0]

        # 2. Check bloom filter
        if not self.bloom_filter.may_contain(key):
            return None  # Definitely not present

        # 3. Search SSTables (newest to oldest)
        for sstable in reversed(self.sstables):
            value = sstable.get(key)
            if value is not None:
                return value

        return None

    def flush(self):
        """Flush memtable to SSTable"""
        if not self.memtable:
            return

        # Create new SSTable
        sstable = SSTable.create(
            sorted(self.memtable.items()),
            filename=f'sstable_{len(self.sstables)}.db'
        )
        self.sstables.append(sstable)

        # Update bloom filter
        for key in self.memtable.keys():
            self.bloom_filter.add(key)

        # Clear memtable
        self.memtable.clear()

        # Trigger compaction if needed
        if len(self.sstables) > 5:
            self.compact()

    def compact(self):
        """Merge SSTables (remove duplicates, tombstones)"""
        # Merge all SSTables
        merged_data = {}
        for sstable in self.sstables:
            for key, (value, ts, vc) in sstable.items():
                if key not in merged_data or ts > merged_data[key][1]:
                    merged_data[key] = (value, ts, vc)

        # Create new compacted SSTable
        new_sstable = SSTable.create(
            sorted(merged_data.items()),
            filename='sstable_compacted.db'
        )

        # Replace old SSTables
        for sstable in self.sstables:
            sstable.delete()
        self.sstables = [new_sstable]
```

### Write-Ahead Log (WAL)

Ensures durability.

```python
class WriteAheadLog:
    def __init__(self, filename):
        self.file = open(filename, 'ab')  # Append mode

    def append(self, entry):
        """Append entry to log"""
        serialized = json.dumps(entry) + '\n'
        self.file.write(serialized.encode())
        self.file.flush()  # Force write to disk
        os.fsync(self.file.fileno())  # Ensure durability

    def replay(self):
        """Replay log after crash"""
        entries = []
        with open(self.file.name, 'r') as f:
            for line in f:
                entries.append(json.loads(line))
        return entries
```

---

## API Design

### REST API

```
PUT    /api/v1/keys/{key}           # Create/Update key
GET    /api/v1/keys/{key}           # Retrieve value
DELETE /api/v1/keys/{key}           # Delete key
GET    /api/v1/keys?prefix={prefix} # List keys with prefix
```

### Example Requests

```bash
# Put a value
curl -X PUT http://api.kvstore.com/v1/keys/user:123 \
  -H "Content-Type: application/json" \
  -d '{"value": {"name": "Alice", "age": 30}}'

# Get a value
curl http://api.kvstore.com/v1/keys/user:123

# Response:
{
  "key": "user:123",
  "value": {"name": "Alice", "age": 30},
  "version": "v1-node1-1234567890",
  "timestamp": 1234567890
}

# Delete a value
curl -X DELETE http://api.kvstore.com/v1/keys/user:123
```

### Client Library API

```python
class KVStoreClient:
    def put(self, key, value, ttl=None, consistency='QUORUM'):
        """
        Store a key-value pair

        Args:
            key: String key
            value: Any serializable value
            ttl: Time-to-live in seconds (optional)
            consistency: 'ONE', 'QUORUM', 'ALL'

        Returns:
            bool: Success status
        """
        pass

    def get(self, key, consistency='QUORUM'):
        """
        Retrieve value for key

        Args:
            key: String key
            consistency: 'ONE', 'QUORUM', 'ALL'

        Returns:
            value: Stored value or None
        """
        pass

    def delete(self, key, consistency='QUORUM'):
        """
        Delete a key-value pair

        Args:
            key: String key
            consistency: 'ONE', 'QUORUM', 'ALL'

        Returns:
            bool: Success status
        """
        pass

    def increment(self, key, delta=1):
        """Atomic increment operation"""
        pass

    def compare_and_swap(self, key, expected_value, new_value):
        """Atomic compare-and-swap"""
        pass

# Usage
client = KVStoreClient(['node1:8080', 'node2:8080', 'node3:8080'])

# Store data
client.put('user:123', {'name': 'Alice', 'age': 30})

# Retrieve data
user = client.get('user:123')
print(user)  # {'name': 'Alice', 'age': 30}

# Delete data
client.delete('user:123')

# Atomic increment
client.put('counter:views', 0)
client.increment('counter:views')  # 1
client.increment('counter:views')  # 2
```

---

## Implementation

### Complete Minimal Key-Value Store

```python
import hashlib
import time
import json
from collections import defaultdict
from threading import Lock

class Node:
    def __init__(self, node_id):
        self.id = node_id
        self.data = {}  # key -> (value, timestamp, vector_clock)
        self.vector_clock = defaultdict(int)
        self.lock = Lock()

    def put(self, key, value, vector_clock=None):
        with self.lock:
            if vector_clock is None:
                self.vector_clock[self.id] += 1
                vector_clock = dict(self.vector_clock)

            self.data[key] = {
                'value': value,
                'timestamp': time.time(),
                'vector_clock': vector_clock
            }
            return True

    def get(self, key):
        with self.lock:
            return self.data.get(key)

    def delete(self, key):
        with self.lock:
            if key in self.data:
                # Tombstone (don't actually delete, mark as deleted)
                self.data[key] = {
                    'value': None,
                    'timestamp': time.time(),
                    'vector_clock': self.vector_clock,
                    'deleted': True
                }
                return True
            return False

class ConsistentHashRing:
    def __init__(self, nodes, virtual_nodes=150):
        self.virtual_nodes = virtual_nodes
        self.ring = {}
        self.sorted_keys = []

        for node in nodes:
            self.add_node(node)

    def add_node(self, node):
        for i in range(self.virtual_nodes):
            key = self._hash(f"{node.id}:{i}")
            self.ring[key] = node
            self.sorted_keys.append(key)
        self.sorted_keys.sort()

    def get_nodes(self, key, n=3):
        if not self.ring:
            return []

        hash_val = self._hash(key)
        nodes = []
        seen = set()

        # Binary search for starting position
        idx = self._binary_search(hash_val)

        # Collect n unique nodes
        while len(nodes) < n and len(seen) < len(set(self.ring.values())):
            if idx >= len(self.sorted_keys):
                idx = 0
            node = self.ring[self.sorted_keys[idx]]
            if node.id not in seen:
                nodes.append(node)
                seen.add(node.id)
            idx += 1

        return nodes

    def _hash(self, key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def _binary_search(self, hash_val):
        left, right = 0, len(self.sorted_keys) - 1
        while left <= right:
            mid = (left + right) // 2
            if self.sorted_keys[mid] == hash_val:
                return mid
            elif self.sorted_keys[mid] < hash_val:
                left = mid + 1
            else:
                right = mid - 1
        return left if left < len(self.sorted_keys) else 0

class DistributedKVStore:
    def __init__(self, nodes):
        self.hash_ring = ConsistentHashRing(nodes)
        self.N = 3  # Replication factor
        self.W = 2  # Write quorum
        self.R = 2  # Read quorum

    def put(self, key, value):
        """Write with quorum"""
        nodes = self.hash_ring.get_nodes(key, self.N)
        successful_writes = 0

        for node in nodes:
            try:
                if node.put(key, value):
                    successful_writes += 1
                    if successful_writes >= self.W:
                        return True
            except Exception as e:
                print(f"Write failed on node {node.id}: {e}")
                continue

        return successful_writes >= self.W

    def get(self, key):
        """Read with quorum and conflict resolution"""
        nodes = self.hash_ring.get_nodes(key, self.N)
        responses = []

        for node in nodes:
            try:
                data = node.get(key)
                if data:
                    responses.append(data)
                    if len(responses) >= self.R:
                        break
            except Exception as e:
                print(f"Read failed on node {node.id}: {e}")
                continue

        if not responses:
            return None

        # Resolve conflicts using vector clocks
        return self._resolve_conflicts(responses)

    def delete(self, key):
        """Delete with quorum"""
        nodes = self.hash_ring.get_nodes(key, self.N)
        successful_deletes = 0

        for node in nodes:
            try:
                if node.delete(key):
                    successful_deletes += 1
                    if successful_deletes >= self.W:
                        return True
            except Exception:
                continue

        return successful_deletes >= self.W

    def _resolve_conflicts(self, responses):
        """Use timestamps for last-write-wins"""
        latest = max(responses, key=lambda x: x['timestamp'])
        return latest['value']

# Usage example
if __name__ == '__main__':
    # Create cluster
    nodes = [Node(f"node{i}") for i in range(5)]
    kv_store = DistributedKVStore(nodes)

    # Write data
    kv_store.put('user:123', {'name': 'Alice', 'age': 30})
    kv_store.put('user:456', {'name': 'Bob', 'age': 25})

    # Read data
    user1 = kv_store.get('user:123')
    print(f"User 123: {user1}")  # {'name': 'Alice', 'age': 30}

    user2 = kv_store.get('user:456')
    print(f"User 456: {user2}")  # {'name': 'Bob', 'age': 25}

    # Delete data
    kv_store.delete('user:123')
    print(kv_store.get('user:123'))  # None
```

---

## Failure Handling

### 1. Node Failures

**Detection:**
- Heartbeat mechanism (gossip protocol)
- Health checks every N seconds

```python
class FailureDetector:
    def __init__(self, nodes, timeout=5):
        self.nodes = nodes
        self.timeout = timeout
        self.last_heartbeat = {}

    def heartbeat(self, node_id):
        """Record heartbeat from node"""
        self.last_heartbeat[node_id] = time.time()

    def get_alive_nodes(self):
        """Return list of alive nodes"""
        now = time.time()
        alive = []
        for node in self.nodes:
            last_hb = self.last_heartbeat.get(node.id, 0)
            if now - last_hb < self.timeout:
                alive.append(node)
        return alive
```

**Recovery:**
- **Hinted Handoff**: Store writes for down nodes, replay when recovered
- **Anti-entropy**: Background sync to repair inconsistencies
- **Merkle Trees**: Efficiently detect data differences between replicas

### 2. Hinted Handoff

```python
class HintedHandoff:
    def __init__(self):
        self.hints = defaultdict(list)  # target_node -> [(key, value, timestamp)]

    def store_hint(self, target_node, key, value):
        """Store write for unavailable node"""
        self.hints[target_node].append({
            'key': key,
            'value': value,
            'timestamp': time.time()
        })

    def replay_hints(self, node):
        """Replay stored hints when node recovers"""
        if node.id in self.hints:
            for hint in self.hints[node.id]:
                node.put(hint['key'], hint['value'])
            del self.hints[node.id]
```

### 3. Network Partitions

**Sloppy Quorum:**
- If preferred nodes unavailable, write to next N healthy nodes
- Temporary inconsistency, resolved later

**Split-Brain Prevention:**
- Use majority quorum (W + R > N)
- Leader election (Paxos, Raft)

---

## Optimizations

### 1. Bloom Filters

Reduce disk reads by quickly checking if key exists.

```python
from bitarray import bitarray
import mmh3  # MurmurHash3

class BloomFilter:
    def __init__(self, size=1000000, hash_count=7):
        self.size = size
        self.hash_count = hash_count
        self.bit_array = bitarray(size)
        self.bit_array.setall(0)

    def add(self, key):
        """Add key to bloom filter"""
        for i in range(self.hash_count):
            index = mmh3.hash(key, i) % self.size
            self.bit_array[index] = 1

    def may_contain(self, key):
        """Check if key might exist (no false negatives)"""
        for i in range(self.hash_count):
            index = mmh3.hash(key, i) % self.size
            if not self.bit_array[index]:
                return False  # Definitely not present
        return True  # Might be present
```

### 2. Caching

```python
from functools import lru_cache

class CachedKVStore:
    def __init__(self, kv_store, cache_size=10000):
        self.kv_store = kv_store
        self.cache = {}  # LRU cache
        self.cache_size = cache_size

    def get(self, key):
        # Check cache first
        if key in self.cache:
            return self.cache[key]

        # Fetch from store
        value = self.kv_store.get(key)

        # Update cache
        if len(self.cache) >= self.cache_size:
            self.cache.popitem()  # Remove oldest
        self.cache[key] = value

        return value
```

### 3. Compression

```python
import zlib

def compress_value(value):
    """Compress large values"""
    serialized = json.dumps(value).encode()
    if len(serialized) > 1024:  # Only compress if > 1KB
        return zlib.compress(serialized)
    return serialized

def decompress_value(compressed):
    """Decompress value"""
    try:
        return json.loads(zlib.decompress(compressed))
    except:
        return json.loads(compressed)
```

---

## Trade-offs

### 1. Consistency vs Availability

| Configuration | Consistency | Availability | Use Case |
|---------------|-------------|--------------|----------|
| **R=1, W=1** | Weak | High | Session data, caching |
| **R=2, W=2, N=3** | Strong | Medium | User profiles, settings |
| **R=3, W=3, N=3** | Strong | Low | Financial data, inventory |

### 2. Read vs Write Performance

| Optimization | Read Speed | Write Speed | Storage | Use Case |
|--------------|------------|-------------|---------|----------|
| **Write-through cache** | Fast | Slow | Medium | Read-heavy |
| **Write-back cache** | Fast | Fast | Medium | Mixed workload |
| **LSM Tree** | Medium | Fast | High | Write-heavy |
| **B-Tree** | Fast | Medium | Medium | Read-heavy |

### 3. Storage vs Performance

| Strategy | Storage | Read Perf | Write Perf |
|----------|---------|-----------|------------|
| **No compression** | High | Fast | Fast |
| **Compression** | Low | Slower | Slower |
| **Bloom filters** | Low | Fast | Same |
| **Caching** | High | Fast | Same |

---

## Real-World Examples

### Amazon DynamoDB

**Architecture:**
- Consistent hashing for partitioning
- Replication across 3 AZs
- Configurable consistency (eventual or strong)
- LSM-based storage

**Key Features:**
- Auto-scaling
- Global tables (multi-region)
- DynamoDB Streams (change data capture)
- TTL for automatic expiration

**Example:**
```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

# Put item
table.put_item(Item={'userId': '123', 'name': 'Alice', 'age': 30})

# Get item (eventually consistent)
response = table.get_item(Key={'userId': '123'})

# Get item (strongly consistent)
response = table.get_item(
    Key={'userId': '123'},
    ConsistentRead=True
)
```

### Redis Cluster

**Architecture:**
- Hash slots (16384 slots)
- Master-slave replication
- Automatic failover

**Example:**
```python
from redis.cluster import RedisCluster

cluster = RedisCluster(host='localhost', port=6379)

# Set value
cluster.set('user:123', json.dumps({'name': 'Alice'}))

# Get value
user = json.loads(cluster.get('user:123'))
```

### Apache Cassandra

**Architecture:**
- Wide-column store
- Tunable consistency (ONE, QUORUM, ALL)
- Multi-datacenter replication
- CQL (Cassandra Query Language)

**Example:**
```python
from cassandra.cluster import Cluster

cluster = Cluster(['127.0.0.1'])
session = cluster.connect('mykeyspace')

# Write
session.execute("""
    INSERT INTO users (user_id, name, age)
    VALUES (%s, %s, %s)
""", ('123', 'Alice', 30))

# Read
rows = session.execute("SELECT * FROM users WHERE user_id = '123'")
for row in rows:
    print(row.name, row.age)
```

---

## Interview Tips

### Key Points to Remember

1. **Consistent Hashing**: Use for partitioning, explain virtual nodes
2. **Replication**: N copies, quorum (R+W>N), trade-offs
3. **CAP Theorem**: Can't have all three, explain trade-offs
4. **Vector Clocks**: Conflict detection, causality tracking
5. **LSM Trees**: Write-optimized storage, compaction

### Common Follow-up Questions

**Q: How do you handle node failures?**
- Replication (N=3)
- Failure detection (heartbeat, gossip)
- Hinted handoff for temporary failures
- Anti-entropy for permanent inconsistencies

**Q: How do you ensure data durability?**
- Write-ahead log (WAL)
- Replication to multiple nodes
- Periodic snapshots
- Backup to durable storage (S3)

**Q: How do you scale reads?**
- Read replicas
- Caching (Redis, Memcached)
- Eventually consistent reads
- Bloom filters to reduce disk I/O

**Q: How do you scale writes?**
- Sharding/partitioning
- Asynchronous replication
- LSM trees (batch writes)
- Write-back caching

**Q: How do you handle hotspots?**
- Consistent hashing with virtual nodes
- Adaptive partitioning
- Caching popular keys
- Rate limiting

### Design Interview Template

1. **Clarify requirements** (consistency vs availability, read/write ratio)
2. **High-level design** (consistent hashing, replication)
3. **API design** (put, get, delete)
4. **Data model** (key-value pairs, metadata)
5. **Storage** (LSM tree, WAL)
6. **Replication** (N, R, W values)
7. **Failure handling** (hinted handoff, anti-entropy)
8. **Optimizations** (bloom filters, caching, compression)

---

## Summary

**Distributed key-value stores** are foundational to modern cloud infrastructure, powering applications from session storage to real-time analytics.

**Key Design Decisions:**
1. **Partitioning**: Consistent hashing with virtual nodes
2. **Replication**: N=3, configurable quorum (R, W)
3. **Consistency**: Tunable (CAP theorem trade-offs)
4. **Conflict Resolution**: Vector clocks, last-write-wins
5. **Storage**: LSM trees for write performance
6. **Failure Handling**: Hinted handoff, anti-entropy

**Production Systems:**
- **DynamoDB**: Auto-scaling, multi-region, strong/eventual consistency
- **Cassandra**: Wide-column, tunable consistency, multi-datacenter
- **Redis**: In-memory, persistence options, clustering

This design demonstrates core distributed systems concepts essential for system design interviews and real-world applications.
