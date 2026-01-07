# Unique ID Generator (Snowflake)

> **Difficulty:** Beginner
> **Topics:** Distributed Systems, ID Generation, Timestamp-based IDs
> **Companies:** Twitter, Instagram, Discord, Shopify

---

## 1. Problem Statement

Design a **distributed unique ID generator** that creates globally unique IDs across multiple servers without coordination.

### Requirements

**Functional:**
- Generate unique 64-bit integer IDs
- IDs must be globally unique (no duplicates)
- IDs should be sortable by time (roughly)
- High throughput (100K IDs/sec per server)

**Non-Functional:**
- No central coordination (each server independent)
- Low latency (<1ms)
- Scalable to thousands of servers
- IDs fit in 64 bits (JavaScript safe integer)

---

## 2. Capacity Estimation

### Traffic Estimates

```python
# Assumptions:
Daily Active Users (DAU): 500 million
Posts per user per day: 2
Daily posts: 500M × 2 = 1 billion posts/day

# ID Generation Rate:
IDs per second: 1B / 86,400 = ~11,600 IDs/sec
Peak QPS (3x average): ~35,000 IDs/sec

# Storage (IDs only):
ID size: 8 bytes (64 bits)
Daily storage: 1B × 8 bytes = 8 GB/day
Annual storage: 8 GB × 365 = 2.9 TB/year
```

### Latency Requirements

```python
p50 latency: < 1ms
p99 latency: < 5ms
Availability: 99.99% (52 minutes downtime/year)
```

---

## 3. Approaches

### Approach 1: UUID (128-bit)

```python
import uuid

id = uuid.uuid4()  # "550e8400-e29b-41d4-a716-446655440000"
```

**Pros:**
- ✅ Guaranteed unique (collision probability ~0)
- ✅ No coordination needed
- ✅ Works offline

**Cons:**
- ❌ 128 bits (too large, doesn't fit in 64-bit integer)
- ❌ Not sortable by time
- ❌ String format inefficient

### Approach 2: Database Auto-Increment

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,  -- Auto-increment
    name VARCHAR(100)
);
```

**Pros:**
- ✅ Simple, built-in
- ✅ Sortable by time

**Cons:**
- ❌ Single point of failure (database)
- ❌ Difficult to shard
- ❌ Performance bottleneck

### Approach 3: Snowflake (Twitter's Solution) ✅

**64-bit ID structure:**

```
┌────────────────────────────────────────────────────┬─────────┬──────────┐
│         Timestamp (41 bits)                         │Machine │ Sequence │
│         Milliseconds since epoch                    │ID (10b)│  (12b)   │
└────────────────────────────────────────────────────┴─────────┴──────────┘
 63                                                  22        12          0

Total: 41 + 10 + 12 = 63 bits (sign bit = 0)
```

**Breakdown:**
- **41 bits - Timestamp:** Milliseconds since custom epoch (~69 years)
- **10 bits - Machine ID:** 1024 unique machines
- **12 bits - Sequence:** 4096 IDs per millisecond per machine

**Pros:**
- ✅ 64-bit integer (fits everywhere)
- ✅ Sortable by time
- ✅ No coordination needed
- ✅ 4M IDs/sec per machine
- ✅ Distributed and scalable

---

## 3. Snowflake Implementation

```python
import time
import threading

class SnowflakeIDGenerator:
    """
    Twitter Snowflake ID Generator

    Generates unique 64-bit IDs:
    - 41 bits: timestamp (milliseconds)
    - 10 bits: machine ID (datacenter + worker)
    - 12 bits: sequence number
    """

    # Custom epoch (January 1, 2020)
    EPOCH = 1577836800000  # Milliseconds

    # Bit lengths
    TIMESTAMP_BITS = 41
    MACHINE_ID_BITS = 10
    SEQUENCE_BITS = 12

    # Max values
    MAX_MACHINE_ID = (1 << MACHINE_ID_BITS) - 1  # 1023
    MAX_SEQUENCE = (1 << SEQUENCE_BITS) - 1      # 4095

    # Bit shifts
    TIMESTAMP_SHIFT = MACHINE_ID_BITS + SEQUENCE_BITS  # 22
    MACHINE_ID_SHIFT = SEQUENCE_BITS                    # 12

    def __init__(self, machine_id: int):
        """
        Initialize ID generator

        Args:
            machine_id: Unique machine identifier (0-1023)
        """
        if machine_id < 0 or machine_id > self.MAX_MACHINE_ID:
            raise ValueError(f"Machine ID must be 0-{self.MAX_MACHINE_ID}")

        self.machine_id = machine_id
        self.sequence = 0
        self.last_timestamp = -1
        self.lock = threading.Lock()

    def _current_millis(self) -> int:
        """Get current timestamp in milliseconds"""
        return int(time.time() * 1000)

    def _wait_next_millis(self, last_timestamp: int) -> int:
        """
        Wait until next millisecond

        Called when sequence overflow in same millisecond
        """
        timestamp = self._current_millis()
        while timestamp <= last_timestamp:
            timestamp = self._current_millis()
        return timestamp

    def generate_id(self) -> int:
        """
        Generate unique 64-bit ID

        Thread-safe, handles clock going backwards

        Returns:
            64-bit integer ID

        Raises:
            RuntimeError: If clock moves backwards
        """
        with self.lock:
            timestamp = self._current_millis()

            # Clock moved backwards (server time adjusted)
            if timestamp < self.last_timestamp:
                raise RuntimeError(
                    f"Clock moved backwards. "
                    f"Refusing to generate ID for {self.last_timestamp - timestamp}ms"
                )

            if timestamp == self.last_timestamp:
                # Same millisecond: increment sequence
                self.sequence = (self.sequence + 1) & self.MAX_SEQUENCE

                if self.sequence == 0:
                    # Sequence overflow: wait for next millisecond
                    timestamp = self._wait_next_millis(self.last_timestamp)
            else:
                # New millisecond: reset sequence
                self.sequence = 0

            self.last_timestamp = timestamp

            # Construct 64-bit ID
            id = (
                ((timestamp - self.EPOCH) << self.TIMESTAMP_SHIFT) |
                (self.machine_id << self.MACHINE_ID_SHIFT) |
                self.sequence
            )

            return id

    def parse_id(self, id: int) -> dict:
        """
        Parse Snowflake ID into components

        Args:
            id: 64-bit Snowflake ID

        Returns:
            Dictionary with timestamp, machine_id, sequence
        """
        timestamp_ms = (id >> self.TIMESTAMP_SHIFT) + self.EPOCH
        machine_id = (id >> self.MACHINE_ID_SHIFT) & self.MAX_MACHINE_ID
        sequence = id & self.MAX_SEQUENCE

        return {
            "timestamp_ms": timestamp_ms,
            "timestamp": time.strftime(
                "%Y-%m-%d %H:%M:%S",
                time.localtime(timestamp_ms / 1000)
            ),
            "machine_id": machine_id,
            "sequence": sequence
        }

# Example Usage
if __name__ == "__main__":
    # Create generator for machine 42
    generator = SnowflakeIDGenerator(machine_id=42)

    # Generate 10 IDs
    print("Generated IDs:")
    ids = []
    for i in range(10):
        id = generator.generate_id()
        ids.append(id)
        print(f"  {i+1}. {id}")

    # IDs are sortable by time
    print(f"\nIDs sorted: {ids == sorted(ids)}")

    # Parse an ID
    sample_id = ids[0]
    parsed = generator.parse_id(sample_id)
    print(f"\nParsed ID {sample_id}:")
    print(f"  Timestamp: {parsed['timestamp']}")
    print(f"  Machine ID: {parsed['machine_id']}")
    print(f"  Sequence: {parsed['sequence']}")

# Output:
# Generated IDs:
#   1. 123456789012345678
#   2. 123456789012345679
#   3. 123456789012345680
#   ...
#
# IDs sorted: True
#
# Parsed ID 123456789012345678:
#   Timestamp: 2024-12-15 10:30:45
#   Machine ID: 42
#   Sequence: 0
```

---

## 4. High-Level Design

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Applications                       │
│         (Mobile Apps, Web Servers, Backend Services)            │
└────────────┬──────────────┬──────────────┬──────────────────────┘
             │              │              │
             ▼              ▼              ▼
      ┌───────────┐   ┌───────────┐   ┌───────────┐
      │   ID Gen  │   │   ID Gen  │   │   ID Gen  │
      │  Server 1 │   │  Server 2 │   │  Server N │
      │           │   │           │   │           │
      │ Machine   │   │ Machine   │   │ Machine   │
      │   ID: 1   │   │   ID: 2   │   │   ID: N   │
      └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
            │               │               │
            │         ┌─────┴─────┐         │
            │         │           │         │
            └────────►│    NTP    │◄────────┘
                      │  Servers  │
                      │           │
                      └───────────┘
                 (Clock Synchronization)

┌───────────────────────────────────────────────────────────────────┐
│                    Configuration Management                        │
│              (ZooKeeper / etcd / Consul)                          │
│                                                                   │
│  - Machine ID Registry                                            │
│  - Server Health Monitoring                                       │
│  - Configuration Distribution                                     │
└───────────────────────────────────────────────────────────────────┘
```

### Components

#### 1. ID Generator Service

```python
# Stateless service running on multiple machines
# Each machine has unique machine_id
# Generates IDs independently without coordination

class IDGeneratorService:
    def __init__(self):
        self.machine_id = self._register_machine()
        self.generator = SnowflakeIDGenerator(self.machine_id)

    def generate(self) -> int:
        """Generate single ID"""
        return self.generator.generate_id()

    def generate_batch(self, count: int) -> List[int]:
        """Generate batch of IDs for efficiency"""
        return [self.generator.generate_id() for _ in range(count)]
```

#### 2. Load Balancer

```python
# Distributes requests across ID generator servers
# Any server can handle any request (stateless)

┌────────────┐
│   Nginx    │  Round-robin or least-connections
│    Load    │  No sticky sessions needed
│  Balancer  │
└────────────┘
```

#### 3. Clock Synchronization (NTP)

```python
# All servers sync with NTP servers
# Keeps clock drift < 1ms
# Critical for maintaining time-sortability

Configuration:
server time.google.com
server time.cloudflare.com
server time.apple.com
```

#### 4. Machine ID Registry (ZooKeeper/etcd)

```python
# Centralized registry for machine IDs
# Handles machine registration and deregistration
# Recycles IDs when machines go offline

/snowflake/
  /machines/
    /1 -> {"host": "server1.example.com", "status": "active"}
    /2 -> {"host": "server2.example.com", "status": "active"}
    /3 -> {"host": "server3.example.com", "status": "inactive"}
```

---

## 5. Capacity Analysis

### Throughput

```python
# Per machine:
Sequence bits: 12 bits = 4096 values
IDs per millisecond: 4096
IDs per second: 4096 × 1000 = 4,096,000 IDs/sec

# 1024 machines (10-bit machine ID):
Total: 4,096,000 × 1024 = 4.2 billion IDs/sec
```

### Time Range

```python
# Timestamp bits: 41 bits
Max timestamp: (2^41 - 1) milliseconds

Years: (2^41) / (1000 × 60 × 60 × 24 × 365)
     ≈ 69.7 years

# With custom epoch (2020-01-01):
Valid until: 2020 + 69 = 2089 ✅
```

---

## 5. Machine ID Assignment

**Challenge:** How to assign unique machine IDs?

### Option 1: Configuration File

```python
# Each server has config file with machine_id
# config.yaml:
machine_id: 42

generator = SnowflakeIDGenerator(machine_id=config['machine_id'])
```

**Pros:** ✅ Simple
**Cons:** ❌ Manual assignment, error-prone

### Option 2: ZooKeeper / etcd

```python
import kazoo.client

def get_machine_id_from_zookeeper():
    """
    Atomically claim next available machine ID from ZooKeeper
    """
    zk = kazoo.client.KazooClient(hosts='zookeeper:2181')
    zk.start()

    # Atomic counter in ZooKeeper
    counter_path = "/snowflake/machine_id_counter"

    # Increment and get
    value, stat = zk.get(counter_path)
    current_id = int(value.decode())

    if current_id >= 1024:
        raise ValueError("No more machine IDs available")

    # Increment
    zk.set(counter_path, str(current_id + 1).encode())

    return current_id

# Usage
machine_id = get_machine_id_from_zookeeper()
generator = SnowflakeIDGenerator(machine_id)
```

### Option 3: Datacenter + Worker ID

```python
class EnhancedSnowflake:
    """
    Split 10-bit machine ID into:
    - 5 bits: Datacenter ID (32 datacenters)
    - 5 bits: Worker ID (32 workers per datacenter)
    """

    DATACENTER_ID_BITS = 5
    WORKER_ID_BITS = 5

    def __init__(self, datacenter_id: int, worker_id: int):
        if datacenter_id < 0 or datacenter_id >= (1 << self.DATACENTER_ID_BITS):
            raise ValueError("Invalid datacenter ID")
        if worker_id < 0 or worker_id >= (1 << self.WORKER_ID_BITS):
            raise ValueError("Invalid worker ID")

        # Combine into machine_id
        machine_id = (datacenter_id << self.WORKER_ID_BITS) | worker_id
        super().__init__(machine_id)

# Usage
generator = EnhancedSnowflake(datacenter_id=2, worker_id=15)
```

---

## 6. Clock Synchronization

**Problem:** Server clocks drift

```python
# Server A clock: 10:00:00.500
# Server B clock: 10:00:00.450  (50ms behind)

# Server A generates: ID with timestamp 500
# Server B generates: ID with timestamp 450

# ❌ B's ID appears older than A's despite being generated later!
```

**Solution: NTP (Network Time Protocol)**

```bash
# Install NTP
sudo apt-get install ntp

# Configure NTP servers
# /etc/ntp.conf:
server 0.pool.ntp.org
server 1.pool.ntp.org
server 2.pool.ntp.org

# NTP keeps clock drift < 1ms ✅
```

**Handling Clock Going Backwards:**

```python
def generate_id(self) -> int:
    timestamp = self._current_millis()

    if timestamp < self.last_timestamp:
        # Clock moved backwards!
        drift = self.last_timestamp - timestamp

        if drift < 5:
            # Small drift: wait it out
            time.sleep(drift / 1000)
            timestamp = self._current_millis()
        else:
            # Large drift: error
            raise RuntimeError(f"Clock moved backwards by {drift}ms")

    # Continue...
```

---

## 7. Alternative: Sonyflake (Sony's Variation)

```
┌────────────────────┬──────────┬───────────┬──────────┐
│  Timestamp (39b)   │ Sequence │ Machine   │Reserved  │
│  10ms precision    │  (8b)    │ ID (16b)  │ (1b)     │
└────────────────────┴──────────┴───────────┴──────────┘
 63                 24         16          0

Differences:
- 10ms precision (vs 1ms) → Lasts 174 years
- 16-bit machine ID → 65,536 machines
- 8-bit sequence → 256 IDs per 10ms
```

**Trade-off:** More machines, less throughput per machine

---

## 8. Comparison Table

| Approach | Uniqueness | Sortable | Size | Coordination | Throughput |
|----------|------------|----------|------|--------------|------------|
| **Auto-increment** | ✅ | ✅ | 64-bit | ❌ DB required | Low |
| **UUID v4** | ✅ | ❌ | 128-bit | ✅ None | High |
| **Snowflake** | ✅ | ✅ | 64-bit | ⚠️ Machine ID | Very High |
| **Sonyflake** | ✅ | ✅ | 64-bit | ⚠️ Machine ID | High |

---

## 9. Real-World Usage

**Twitter (Original Snowflake):**
- Tweet IDs, User IDs
- Billions of IDs generated daily

**Instagram:**
- Modified Snowflake for photo IDs
- 41 bits timestamp + 13 bits shard ID + 10 bits sequence

**Discord:**
- Uses Snowflake for all IDs (messages, channels, users)
- Parseable to extract timestamp

```javascript
// Discord: Extract timestamp from Snowflake ID
const DISCORD_EPOCH = 1420070400000;  // 2015-01-01

function getTimestamp(snowflake) {
  return Number(BigInt(snowflake) >> 22n) + DISCORD_EPOCH;
}

const messageId = "175928847299117063";
const timestamp = getTimestamp(messageId);
console.log(new Date(timestamp));  // Message creation time
```

---

## 10. Summary

**Snowflake ID Generator:**

✅ **Distributed** - No coordination between machines
✅ **64-bit** - Fits in standard integer types
✅ **Sortable** - IDs increase over time
✅ **High throughput** - 4M IDs/sec per machine
✅ **Parseable** - Extract timestamp, machine ID

**Key Components:**
1. Timestamp (41 bits) - Sortability
2. Machine ID (10 bits) - Uniqueness across machines
3. Sequence (12 bits) - Uniqueness within millisecond

**Trade-offs:**
- Requires machine ID assignment
- Relies on clock synchronization (NTP)
- Reveals timestamp (privacy consideration)

---

## 11. Deep Dive: Additional Approaches

### Approach 4: ULID (Universally Unique Lexicographically Sortable ID)

```python
import time
import random

class ULIDGenerator:
    """
    ULID Format (128 bits):
    - 48 bits: Timestamp (milliseconds)
    - 80 bits: Random

    Advantages:
    - Lexicographically sortable (can be stored as string)
    - 128-bit random component reduces collision risk
    - Case-insensitive Base32 encoding
    """

    BASE32 = "0123456789ABCDEFGHJKMNPQRSTVWXYZ"

    def generate(self) -> str:
        """Generate ULID string (26 characters)"""
        timestamp_ms = int(time.time() * 1000)

        # Encode timestamp (10 chars)
        timestamp_part = self._encode_time(timestamp_ms, 10)

        # Encode random (16 chars)
        random_part = self._encode_random(16)

        return timestamp_part + random_part

    def _encode_time(self, timestamp_ms: int, length: int) -> str:
        result = ""
        for _ in range(length):
            result = self.BASE32[timestamp_ms % 32] + result
            timestamp_ms //= 32
        return result

    def _encode_random(self, length: int) -> str:
        return "".join(random.choice(self.BASE32) for _ in range(length))

# Example
generator = ULIDGenerator()
ulid = generator.generate()
print(ulid)  # "01ARZ3NDEKTSV4RRFFQ69G5FAV"
```

**ULID vs Snowflake:**

| Feature | ULID | Snowflake |
|---------|------|-----------|
| Size | 128 bits | 64 bits |
| Format | String (26 chars) | Integer |
| Sortable | ✅ Yes | ✅ Yes |
| Random component | 80 bits | 12 bits (sequence) |
| Machine ID needed | ❌ No | ✅ Yes |
| Collision risk | Very low | Zero (with proper machine ID) |

---

### Approach 5: MongoDB ObjectId

```python
import time
import os
import random

class ObjectIdGenerator:
    """
    MongoDB ObjectId (96 bits / 12 bytes):
    - 4 bytes: Timestamp (seconds)
    - 5 bytes: Random value (process + machine)
    - 3 bytes: Incrementing counter
    """

    def __init__(self):
        self.counter = random.randint(0, 0xFFFFFF)
        self.random_value = os.urandom(5)

    def generate(self) -> bytes:
        timestamp = int(time.time()).to_bytes(4, 'big')
        counter_bytes = (self.counter % 0xFFFFFF).to_bytes(3, 'big')
        self.counter += 1

        return timestamp + self.random_value + counter_bytes

    def to_hex(self, object_id: bytes) -> str:
        """Convert to hex string (24 characters)"""
        return object_id.hex()

# Example
generator = ObjectIdGenerator()
object_id = generator.generate()
print(generator.to_hex(object_id))  # "507f1f77bcf86cd799439011"
```

---

### Approach 6: Instagram's Sharded ID

```python
"""
Instagram uses modified Snowflake with PostgreSQL:

64-bit ID structure:
- 41 bits: Timestamp (milliseconds)
- 13 bits: Shard ID (8192 shards)
- 10 bits: Auto-increment sequence (1024 per millisecond)

Key difference: Shard ID is part of the ID itself
This allows Instagram to know which database shard to query
just by looking at the ID!
"""

class InstagramIDGenerator:
    def __init__(self, shard_id: int):
        self.shard_id = shard_id  # 0-8191
        self.sequence = 0
        self.last_timestamp = -1

        # PostgreSQL sequence per shard
        self.postgres_sequence = f"CREATE SEQUENCE shard_{shard_id}_seq"

    def generate_id(self) -> int:
        timestamp = int(time.time() * 1000)

        if timestamp == self.last_timestamp:
            self.sequence = (self.sequence + 1) & 0x3FF
        else:
            self.sequence = 0

        self.last_timestamp = timestamp

        # Construct ID
        return ((timestamp - EPOCH) << 23) | (self.shard_id << 10) | self.sequence

    def extract_shard(self, id: int) -> int:
        """Extract shard ID from Instagram ID"""
        return (id >> 10) & 0x1FFF
```

**Why Instagram chose this:**
- Each photo belongs to a user
- User IDs are hashed to shards
- Photo IDs embed the shard ID
- Query: "Get photo 12345" → Extract shard → Query correct database

---

## 12. Trade-offs & Design Decisions

### 1. Integer vs String IDs

**Integer (Snowflake):**
- ✅ Space efficient (8 bytes)
- ✅ Fast comparisons
- ✅ Database indexing optimized
- ❌ Not URL-safe without encoding

**String (UUID, ULID):**
- ✅ URL-safe
- ✅ Human-readable
- ❌ Larger (16-36 bytes)
- ❌ Slower comparisons

**Decision:** Use integers for internal IDs, encode to Base62 for URLs

```python
def encode_base62(num: int) -> str:
    """Encode integer ID to Base62 for URLs"""
    alphabet = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
    if num == 0:
        return alphabet[0]

    result = ""
    while num:
        result = alphabet[num % 62] + result
        num //= 62
    return result

# Example
internal_id = 123456789012345678  # Snowflake ID
url_id = encode_base62(internal_id)  # "1T7kM7v9b4"
```

---

### 2. Embedded Metadata vs Pure Random

**Embedded Metadata (Snowflake, Instagram):**
- ✅ Can extract timestamp, shard, machine
- ✅ Useful for debugging
- ✅ Can route based on ID
- ❌ Privacy concern (reveals timing)
- ❌ Less flexibility (bits are allocated)

**Pure Random (UUID v4):**
- ✅ No information leakage
- ✅ Maximum entropy
- ❌ Can't extract metadata
- ❌ Not sortable

**Decision:** Depends on use case
- **Public IDs** (user-facing): Use pure random or ULID
- **Internal IDs** (database): Use Snowflake for sortability

---

### 3. Centralized vs Decentralized

**Centralized (Database auto-increment):**
- ✅ Guaranteed uniqueness
- ✅ Simple logic
- ❌ Single point of failure
- ❌ Performance bottleneck
- ❌ Hard to scale

**Decentralized (Snowflake):**
- ✅ No coordination needed
- ✅ High availability
- ✅ Linear scalability
- ❌ Requires machine ID management
- ❌ Clock synchronization needed

**Hybrid (Ticket Server):**
```python
"""
Flickr's approach: Pre-allocate ID ranges

Ticket Server allocates ranges:
- Server A: 1-1000
- Server B: 1001-2000
- Server C: 2001-3000

Each server generates IDs from its range
No coordination needed until range exhausted
"""

class TicketServerClient:
    def __init__(self):
        self.current_id = 0
        self.max_id = 0

    def get_id(self) -> int:
        if self.current_id >= self.max_id:
            # Request new range from ticket server
            self.current_id, self.max_id = self._request_range()

        self.current_id += 1
        return self.current_id

    def _request_range(self) -> tuple:
        """Request ID range from central ticket server"""
        response = requests.post("http://ticket-server/allocate", json={"size": 1000})
        return response.json()["start"], response.json()["end"]
```

---

## 13. Monitoring & Observability

### Key Metrics to Monitor

```python
# 1. ID Generation Rate
ids_generated_per_second = Counter("snowflake_ids_generated_total")

# 2. Sequence Overflow Events
sequence_overflows = Counter("snowflake_sequence_overflows_total")
# When sequence hits 4096 in single millisecond

# 3. Clock Drift Events
clock_drift_events = Counter("snowflake_clock_backwards_total")
# When clock moves backwards

# 4. Generation Latency
id_generation_latency = Histogram("snowflake_generation_latency_seconds")

# 5. Machine ID Conflicts
machine_id_conflicts = Counter("snowflake_machine_id_conflicts_total")
```

### Alerting Rules

```yaml
# Alert if clock moves backwards significantly
- alert: ClockMovedBackwards
  expr: increase(snowflake_clock_backwards_total[5m]) > 0
  severity: critical
  annotations:
    description: "Server clock moved backwards - check NTP"

# Alert if sequence overflows frequently (high load)
- alert: HighSequenceOverflow
  expr: rate(snowflake_sequence_overflows_total[1m]) > 100
  severity: warning
  annotations:
    description: "Sequence overflowing frequently - consider scaling"

# Alert if generation latency is high
- alert: HighIDGenerationLatency
  expr: histogram_quantile(0.99, snowflake_generation_latency_seconds) > 0.005
  severity: warning
  annotations:
    description: "p99 ID generation latency > 5ms"
```

---

## 14. Testing Strategies

### 1. Uniqueness Testing

```python
import unittest
from concurrent.futures import ThreadPoolExecutor

class TestSnowflakeUniqueness(unittest.TestCase):
    def test_single_threaded_uniqueness(self):
        """Test IDs are unique in single thread"""
        generator = SnowflakeIDGenerator(machine_id=1)
        ids = set(generator.generate_id() for _ in range(100000))

        # All IDs should be unique
        self.assertEqual(len(ids), 100000)

    def test_multi_threaded_uniqueness(self):
        """Test IDs are unique across threads"""
        generator = SnowflakeIDGenerator(machine_id=1)

        def generate_batch():
            return [generator.generate_id() for _ in range(10000)]

        with ThreadPoolExecutor(max_workers=10) as executor:
            results = list(executor.map(lambda _: generate_batch(), range(10)))

        all_ids = [id for batch in results for id in batch]

        # All IDs should be unique even across threads
        self.assertEqual(len(set(all_ids)), len(all_ids))

    def test_multi_machine_uniqueness(self):
        """Test IDs are unique across machines"""
        generators = [SnowflakeIDGenerator(machine_id=i) for i in range(10)]

        all_ids = []
        for gen in generators:
            all_ids.extend([gen.generate_id() for _ in range(10000)])

        # All IDs should be unique across machines
        self.assertEqual(len(set(all_ids)), len(all_ids))
```

### 2. Sortability Testing

```python
def test_sortability(self):
    """Test IDs are roughly sorted by time"""
    generator = SnowflakeIDGenerator(machine_id=1)

    ids = []
    for _ in range(1000):
        ids.append(generator.generate_id())
        time.sleep(0.001)  # 1ms delay

    # IDs should be in ascending order (roughly)
    self.assertEqual(ids, sorted(ids))
```

### 3. Clock Backwards Testing

```python
def test_clock_backwards(self):
    """Test handling of clock moving backwards"""
    generator = SnowflakeIDGenerator(machine_id=1)

    # Generate ID
    id1 = generator.generate_id()

    # Simulate clock going backwards
    generator.last_timestamp = generator._current_millis() + 1000

    # Should raise error
    with self.assertRaises(RuntimeError):
        generator.generate_id()
```

---

## 15. Interview Tips & Common Questions

### Q1: "Why not use UUID?"

**Answer:**
"UUID v4 is 128 bits, which doesn't fit in a 64-bit integer. Many systems (JavaScript, some databases) have issues with 128-bit values. Additionally, UUIDs are not sortable by time, which means:
- Poor database index locality (scattered inserts)
- Can't easily query 'IDs created today'
- Random I/O instead of sequential I/O

Snowflake gives us 64-bit integers that are sortable, which leads to better database performance."

### Q2: "What if two machines generate the same ID?"

**Answer:**
"This can't happen if machine IDs are unique. The 10-bit machine ID space allows 1024 machines, each with a unique identifier. Even if two machines generate an ID at the exact same millisecond with the same sequence number, the machine ID portion will differ, ensuring uniqueness.

The critical requirement is proper machine ID assignment and management."

### Q3: "How do you handle machine ID exhaustion?"

**Answer:**
"Several strategies:
1. **Recycle IDs**: When a machine goes offline, recycle its ID after a grace period
2. **Adjust bit allocation**: If you need more machines, reduce timestamp or sequence bits
3. **Hierarchical IDs**: Split machine ID into datacenter (5 bits) + worker (5 bits) for better organization
4. **Multiple ID pools**: Use different machine ID pools for different services"

### Q4: "What happens during clock synchronization?"

**Answer:**
"NTP synchronization is usually gradual (slewing), not sudden jumps. However, if the clock jumps backwards:
1. **Small jumps (<5ms)**: Wait it out (block until clock catches up)
2. **Large jumps**: Raise an error and alert operations
3. **Prevention**: Use multiple NTP sources and monitor for anomalies

In production, we'd also monitor clock drift and alert if it exceeds thresholds."

### Q5: "Can Snowflake IDs be used for distributed transactions?"

**Answer:**
"Snowflake IDs are great for entity identification but not for transaction ordering because:
- Clock skew between machines means IDs aren't perfectly ordered globally
- For transaction ordering, you need consensus (e.g., Paxos, Raft)
- For causal ordering, use vector clocks or hybrid logical clocks

Snowflake is best for: User IDs, Post IDs, Order IDs (entity identifiers), not for distributed transaction sequencing."

---

## 16. Summary & Key Takeaways

### Core Concepts

1. **Distributed ID Generation** requires careful balance of:
   - Uniqueness (no duplicates)
   - Sortability (time-ordering)
   - Performance (low latency, high throughput)
   - Scalability (no single point of failure)

2. **Snowflake's elegance** comes from:
   - Embedding timestamp → Sortability
   - Embedding machine ID → No coordination
   - Embedding sequence → High throughput per machine

3. **Trade-offs to consider**:
   - Privacy (timestamp is embedded in ID)
   - Clock dependency (NTP required)
   - Machine ID management (operational complexity)

### When to Use What

| Use Case | Recommended Approach | Reason |
|----------|---------------------|---------|
| **User IDs** | Snowflake | Need sortability, 64-bit, high volume |
| **Session IDs** | UUID v4 | Security (random), no sortability needed |
| **URLs (public)** | ULID or Custom | URL-safe, lexicographically sortable |
| **Database PKs** | Snowflake | Index locality, range queries |
| **API Request IDs** | UUID v4 | No central system, stateless |
| **Sharded Systems** | Instagram-style | Need to embed shard ID for routing |

### Production Checklist

- [ ] Machine ID assignment strategy defined
- [ ] NTP configured on all servers
- [ ] Clock drift monitoring in place
- [ ] Alerting for clock backwards events
- [ ] Load testing completed (verify throughput)
- [ ] Uniqueness testing across multiple machines
- [ ] Disaster recovery plan (machine ID conflicts)
- [ ] Documentation for on-call engineers

**Next Steps:**
- Learn about [Load Balancing](../../../01-fundamentals/networking/load-balancing.md)
- Study [Database Sharding](../../intermediate/database-sharding/README.md)
- Explore [Distributed Systems Patterns](../../advanced/distributed-patterns/README.md)
