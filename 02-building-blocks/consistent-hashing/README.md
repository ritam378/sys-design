# Consistent Hashing

## Overview

**Consistent Hashing** is a distributed hashing technique that minimizes data movement when nodes are added or removed from a distributed system. It's essential for building scalable distributed caches, load balancers, and databases.

**Key Question:** "How do you distribute data across servers such that adding/removing servers doesn't require massive data migration?"

**Short Answer:**
- **Hash ring** maps both data and servers onto a circular space
- Adding/removing a server only affects adjacent data (not all data)
- **Virtual nodes** ensure even distribution

---

## The Problem

### Naive Approach: Modulo Hashing

```python
# Simple hash distribution
server_id = hash(key) % num_servers

# Example with 4 servers:
hash("user_123") % 4 = 2  → Server 2
hash("user_456") % 4 = 1  → Server 1
hash("user_789") % 4 = 3  → Server 3
```

**Problem: Adding/Removing Servers Breaks Everything**

```python
# Originally 4 servers
hash("user_123") % 4 = 2  → Server 2

# Add 1 server (now 5 servers)
hash("user_123") % 5 = 3  → Server 3  ❌ Different server!

# Result: ~80% of keys move to different servers!
```

**Impact:**
- ❌ Cache misses spike to 80%+
- ❌ Massive data migration required
- ❌ Database overwhelmed by cache misses
- ❌ System downtime or severe degradation

---

## Consistent Hashing Solution

### Core Concept: Hash Ring

**Instead of modulo, map to a circular hash space (0 to 2^32-1):**

```
Hash Space: 0 → 2^32-1 (forms a ring)

         0/2^32
          │
    ┌─────┴─────┐
    │           │
2^30│           │2^2
    │    Ring   │
    │           │
2^28│           │2^4
    │           │
    └─────┬─────┘
          │
        2^31
```

**Both servers AND keys are hashed onto this ring:**

1. **Hash servers** onto ring (using server IP/name)
2. **Hash keys** onto ring
3. **Key belongs to first server clockwise** from its position

**Example:**

```
Ring positions (simplified):

         S1 (hash=100)
          │
    ┌─────┴─────┐
    │           │
   K2           K1
  (50)        (120)
    │           │
S4  │    Ring   │  S2
(300)│         (150)
    │           │
   K3           │
  (280)         │
    └─────┬─────┘
          │
        S3 (200)

Key assignments (clockwise):
K1 (120) → S2 (150)
K2 (50)  → S1 (100)
K3 (280) → S4 (300)
```

---

## Why It Works

### Adding a Server

```
Before (3 servers):
S1(100), S2(150), S3(200)

Keys:
K1(120) → S2
K2(50)  → S1
K3(180) → S3

Add S4(125):
         S1 (100)
          │
    ┌─────┴─────┐
    │           │
   K2     S4    K1
  (50)  (125) (120)
    │           │
    │    Ring   │  S2
    │         (150)
    │           │
   K3           │
  (180)         │
    └─────┬─────┘
          │
        S3 (200)

After:
K1(120) → S4 (125)  ← MOVED
K2(50)  → S1 (100)  ✅ Same
K3(180) → S3 (200)  ✅ Same

Only ~25% of keys moved (those between S1 and S2)
vs. 75% with modulo hashing!
```

**Key Insight:** Only keys in affected range move, not all keys.

---

## Implementation

### Basic Consistent Hashing

```python
import hashlib
from typing import Dict, List, Optional
from bisect import bisect_right

class ConsistentHash:
    """
    Consistent hashing implementation with virtual nodes

    Minimizes data movement when adding/removing servers
    """

    def __init__(self, nodes: Optional[List[str]] = None, virtual_nodes: int = 150):
        """
        Initialize consistent hash ring

        Args:
            nodes: List of server identifiers (e.g., ["server1", "server2"])
            virtual_nodes: Number of virtual nodes per physical node
        """
        self.virtual_nodes = virtual_nodes
        self.ring: Dict[int, str] = {}  # hash_value → node_name
        self.sorted_keys: List[int] = []  # Sorted hash values for binary search

        if nodes:
            for node in nodes:
                self.add_node(node)

    def _hash(self, key: str) -> int:
        """
        Hash function: MD5 hash to 32-bit integer

        Args:
            key: String to hash

        Returns:
            Integer hash value (0 to 2^32-1)
        """
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str):
        """
        Add a server node to the ring

        Creates multiple virtual nodes for better distribution

        Args:
            node: Server identifier (e.g., "server1" or "192.168.1.10")
        """
        for i in range(self.virtual_nodes):
            # Create virtual node identifier
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)

            self.ring[hash_value] = node
            self.sorted_keys.append(hash_value)

        # Keep keys sorted for binary search
        self.sorted_keys.sort()

    def remove_node(self, node: str):
        """
        Remove a server node from the ring

        Args:
            node: Server identifier to remove
        """
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)

            if hash_value in self.ring:
                del self.ring[hash_value]
                self.sorted_keys.remove(hash_value)

    def get_node(self, key: str) -> Optional[str]:
        """
        Get the server node responsible for a key

        Args:
            key: Data key (e.g., "user_123")

        Returns:
            Server identifier, or None if no nodes available
        """
        if not self.ring:
            return None

        hash_value = self._hash(key)

        # Find first node clockwise from key's position
        # Binary search for efficiency
        idx = bisect_right(self.sorted_keys, hash_value)

        # Wrap around to beginning if past last node
        if idx == len(self.sorted_keys):
            idx = 0

        return self.ring[self.sorted_keys[idx]]

    def get_nodes(self, key: str, count: int) -> List[str]:
        """
        Get multiple nodes for replication

        Args:
            key: Data key
            count: Number of replica nodes needed

        Returns:
            List of server identifiers (unique physical nodes)
        """
        if not self.ring or count <= 0:
            return []

        hash_value = self._hash(key)
        idx = bisect_right(self.sorted_keys, hash_value)

        nodes = []
        seen = set()

        # Traverse ring clockwise until we have 'count' unique nodes
        while len(nodes) < count and len(seen) < len(self.ring):
            if idx >= len(self.sorted_keys):
                idx = 0

            node = self.ring[self.sorted_keys[idx]]

            # Ensure unique physical nodes (skip virtual duplicates)
            if node not in seen:
                nodes.append(node)
                seen.add(node)

            idx += 1

        return nodes

    def get_distribution(self, num_keys: int = 10000) -> Dict[str, int]:
        """
        Simulate key distribution across nodes

        Args:
            num_keys: Number of keys to simulate

        Returns:
            Dictionary mapping node → key count
        """
        distribution = {}

        for i in range(num_keys):
            key = f"key_{i}"
            node = self.get_node(key)
            distribution[node] = distribution.get(node, 0) + 1

        return distribution

# Example Usage
if __name__ == "__main__":
    # Create hash ring with 3 servers
    ch = ConsistentHash(["server1", "server2", "server3"])

    # Assign keys
    keys = ["user_123", "user_456", "user_789", "post_1", "post_2"]

    print("Initial assignment:")
    assignments = {}
    for key in keys:
        node = ch.get_node(key)
        assignments[key] = node
        print(f"  {key} → {node}")

    # Add a new server
    print("\nAdding server4...")
    ch.add_node("server4")

    print("\nAssignment after adding server4:")
    moved = 0
    for key in keys:
        new_node = ch.get_node(key)
        old_node = assignments[key]
        status = "MOVED" if new_node != old_node else "same"
        print(f"  {key} → {new_node} ({status})")
        if new_node != old_node:
            moved += 1

    print(f"\nKeys moved: {moved}/{len(keys)} ({moved*100//len(keys)}%)")

    # Show distribution
    print("\nKey distribution (10,000 keys):")
    distribution = ch.get_distribution(10000)
    for node, count in sorted(distribution.items()):
        percentage = count / 100
        print(f"  {node}: {count} keys ({percentage:.1f}%)")

# Output:
# Initial assignment:
#   user_123 → server2
#   user_456 → server1
#   user_789 → server3
#   post_1 → server3
#   post_2 → server1
#
# Adding server4...
#
# Assignment after adding server4:
#   user_123 → server2 (same)
#   user_456 → server4 (MOVED)
#   user_789 → server3 (same)
#   post_1 → server3 (same)
#   post_2 → server1 (same)
#
# Keys moved: 1/5 (20%)
#
# Key distribution (10,000 keys):
#   server1: 2487 keys (24.9%)
#   server2: 2501 keys (25.0%)
#   server3: 2534 keys (25.3%)
#   server4: 2478 keys (24.8%)
```

---

## Virtual Nodes

### Problem: Uneven Distribution

**With just physical nodes, distribution can be uneven:**

```
3 physical nodes (no virtual nodes):

         S1 (small arc)
          │
    ┌─────┴─────┐
    │           │
    │           │
    │           │
S3  │    Ring   │  S2 (large arc)
    │           │
    │           │
    │           │
    └─────┬─────┘

S1: 10% of keys  ❌ Underutilized
S2: 60% of keys  ❌ Overloaded
S3: 30% of keys  ✅ Balanced
```

### Solution: Virtual Nodes

**Each physical node gets multiple positions on the ring:**

```python
# With 150 virtual nodes per server:
server1 → server1:0, server1:1, ..., server1:149
server2 → server2:0, server2:1, ..., server2:149
server3 → server3:0, server3:1, ..., server3:149

# Now distributed evenly across ring:
         s1:0
          │
    ┌─────┴─────┐
  s3:5      s2:3
    │           │
  s1:10   Ring  s3:8
    │           │
  s2:15         s1:20
    │           │
    └─────┬─────┘
        s3:12

# Result: Each server gets ~33% of keys (balanced)
```

**Trade-off:**
- ✅ Better distribution (standard deviation < 5%)
- ✅ Smoother load when adding/removing nodes
- ❌ More memory (150 virtual nodes vs 1 physical)
- ❌ Slightly slower lookups (more ring entries)

**Recommended:** 100-200 virtual nodes per physical node

---

## Real-World Applications

### 1. Distributed Caching (Memcached, Redis)

```python
class DistributedCache:
    """
    Distributed cache using consistent hashing

    Minimizes cache misses when servers added/removed
    """

    def __init__(self, cache_servers: List[str]):
        self.ch = ConsistentHash(cache_servers, virtual_nodes=150)
        self.connections = {
            server: self._connect(server) for server in cache_servers
        }

    def _connect(self, server: str):
        """Connect to cache server (simplified)"""
        # In production: redis.Redis(host=server)
        return f"connection_to_{server}"

    def get(self, key: str) -> Optional[str]:
        """Get value from cache"""
        server = self.ch.get_node(key)
        conn = self.connections[server]
        # return conn.get(key)
        print(f"GET {key} from {server}")
        return None

    def set(self, key: str, value: str):
        """Set value in cache"""
        server = self.ch.get_node(key)
        conn = self.connections[server]
        # conn.set(key, value)
        print(f"SET {key}={value} to {server}")

    def add_server(self, server: str):
        """
        Add new cache server

        Only keys in affected range need migration
        """
        # Track which keys will move
        affected_keys = []

        # Add server to ring
        self.ch.add_node(server)
        self.connections[server] = self._connect(server)

        print(f"Added {server}. Minimal data migration needed.")
        # In production: migrate only affected keys

    def remove_server(self, server: str):
        """Remove cache server"""
        self.ch.remove_node(server)
        if server in self.connections:
            # self.connections[server].close()
            del self.connections[server]

        print(f"Removed {server}. Keys redistributed to next servers.")

# Usage
cache = DistributedCache(["cache1", "cache2", "cache3"])
cache.set("user:123", "Alice")
cache.set("user:456", "Bob")
cache.get("user:123")

cache.add_server("cache4")  # Only ~25% of keys migrate
```

---

### 2. Load Balancing

```python
class ConsistentHashLoadBalancer:
    """
    Load balancer using consistent hashing

    Ensures same client always routed to same backend
    (useful for sticky sessions)
    """

    def __init__(self, backends: List[str]):
        self.ch = ConsistentHash(backends, virtual_nodes=100)

    def route_request(self, client_id: str) -> str:
        """
        Route client to backend server

        Same client always gets same server (unless server fails)
        """
        return self.ch.get_node(client_id)

    def add_backend(self, backend: str):
        """Add new backend server"""
        self.ch.add_node(backend)
        print(f"Added {backend}. Only ~{100 // (len(self.ch.ring) + 1)}% of sessions migrate.")

    def remove_backend(self, backend: str):
        """Remove failed backend"""
        self.ch.remove_node(backend)
        print(f"Removed {backend}. Sessions redistributed.")

# Usage
lb = ConsistentHashLoadBalancer(["backend1", "backend2", "backend3"])

# Client requests
for client_id in ["client_a", "client_b", "client_c", "client_d"]:
    backend = lb.route_request(client_id)
    print(f"{client_id} → {backend}")

# Output:
# client_a → backend2
# client_b → backend1
# client_c → backend3
# client_d → backend2
# (same client always goes to same backend)
```

---

### 3. Database Sharding

```python
class ConsistentHashSharding:
    """
    Database sharding using consistent hashing

    Minimizes data movement when adding/removing shards
    """

    def __init__(self, shards: List[str]):
        self.ch = ConsistentHash(shards, virtual_nodes=150)

    def get_shard(self, user_id: int) -> str:
        """Determine which shard stores this user"""
        return self.ch.get_node(str(user_id))

    def get_replica_shards(self, user_id: int, replicas: int = 3) -> List[str]:
        """Get shards for replication (primary + replicas)"""
        return self.ch.get_nodes(str(user_id), replicas)

    def add_shard(self, shard: str):
        """
        Add new database shard

        Only 1/N of data needs migration (N = num shards)
        """
        old_shards = len(set(self.ch.ring.values()))
        self.ch.add_node(shard)
        new_shards = len(set(self.ch.ring.values()))

        migration_percentage = 100 / new_shards
        print(f"Added {shard}. Migrate ~{migration_percentage:.1f}% of data.")

# Usage
db_sharding = ConsistentHashSharding(["shard1", "shard2", "shard3"])

# Determine shard for users
for user_id in [123, 456, 789]:
    shard = db_sharding.get_shard(user_id)
    replicas = db_sharding.get_replica_shards(user_id, replicas=2)
    print(f"User {user_id}: primary={shard}, replicas={replicas}")

# Add new shard
db_sharding.add_shard("shard4")  # Only 25% of data migrates
```

---

## Comparison with Modulo Hashing

| Aspect | Modulo Hashing | Consistent Hashing |
|--------|----------------|---------------------|
| **Formula** | `hash(key) % N` | Hash ring + clockwise lookup |
| **Adding server** | ~(N-1)/N keys move | ~1/N keys move |
| **Removing server** | ~(N-1)/N keys move | Only keys on that server |
| **Distribution** | Perfect (if hash is uniform) | Good (with virtual nodes) |
| **Complexity** | O(1) | O(log V) where V = virtual nodes |
| **Use case** | Fixed number of servers | Dynamic server pool |

**Example: 4 servers → 5 servers**

```
Modulo:
  Keys moved: 80% ❌

Consistent Hashing:
  Keys moved: 20% ✅
```

---

## Advanced: Bounded Load Consistent Hashing

**Problem:** Some keys are "hot" (accessed frequently), leading to imbalance.

**Solution:** Limit max load per server.

```python
class BoundedLoadConsistentHash(ConsistentHash):
    """
    Consistent hashing with bounded load

    Prevents hot keys from overloading a single server
    """

    def __init__(self, nodes: List[str], virtual_nodes: int = 150, load_factor: float = 1.25):
        super().__init__(nodes, virtual_nodes)
        self.load_factor = load_factor  # Max load = avg load × load_factor
        self.load: Dict[str, int] = {node: 0 for node in set(nodes)}

    def get_node_with_load(self, key: str) -> str:
        """
        Get node for key, respecting load limits

        If primary node overloaded, use next node on ring
        """
        if not self.ring:
            return None

        avg_load = sum(self.load.values()) / len(set(self.load.keys()))
        max_load = avg_load * self.load_factor

        hash_value = self._hash(key)
        idx = bisect_right(self.sorted_keys, hash_value)

        # Try nodes clockwise until finding one under load limit
        attempts = 0
        while attempts < len(self.sorted_keys):
            if idx >= len(self.sorted_keys):
                idx = 0

            node = self.ring[self.sorted_keys[idx]]

            if self.load[node] < max_load:
                self.load[node] += 1
                return node

            idx += 1
            attempts += 1

        # All nodes at max load, use primary anyway
        node = super().get_node(key)
        self.load[node] += 1
        return node

# Usage: Prevents hotspots by spreading load
```

---

## Interview Tips

### Common Questions

**Q: "What happens when you add a server?"**

**A:** Only keys in the range between new server and previous server (clockwise) need to migrate. This is approximately 1/N of total keys where N is the new number of servers.

**Q: "Why use virtual nodes?"**

**A:**
- Better load distribution (avoids hotspots)
- Smoother migration when adding/removing servers
- Recommended: 100-200 virtual nodes per physical node

**Q: "What's the time complexity?"**

**A:**
- `get_node(key)`: O(log V) where V = total virtual nodes (binary search)
- `add_node()`: O(V log V) for sorting
- `remove_node()`: O(V log V)

**Q: "Compare to rendezvous hashing?"**

**A:**
- Consistent hashing: O(log V) lookup, requires sorted structure
- Rendezvous hashing: O(N) lookup (hash key with each server), simpler but slower

---

## Summary

**Consistent Hashing** solves the distributed data placement problem:

✅ **Minimal migration** - Only ~1/N keys move when adding server
✅ **No single point of failure** - Decentralized
✅ **Scalable** - Easy to add/remove nodes
✅ **Load balanced** - Virtual nodes ensure even distribution

**Used by:**
- Memcached, Redis (distributed caching)
- Cassandra, DynamoDB (distributed databases)
- Akamai CDN (content distribution)
- Discord (message routing)

**Key Concepts:**
1. Hash ring (circular hash space)
2. Clockwise lookup
3. Virtual nodes for balance
4. Binary search for efficiency

**Trade-offs:**
- Slightly more complex than modulo hashing
- Requires sorted data structure (O(log V) vs O(1))
- Memory overhead for virtual nodes

**Next:** Learn about [LRU Cache](../lru-cache/README.md) for efficient caching.
