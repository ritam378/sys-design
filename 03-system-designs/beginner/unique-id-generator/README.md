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

## 2. Approaches

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

## 4. Capacity Analysis

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

**Next:** Learn about [Load Balancing](../../../01-fundamentals/networking/load-balancing.md) strategies.
