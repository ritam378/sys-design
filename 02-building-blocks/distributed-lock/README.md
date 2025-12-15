# Distributed Lock

## Overview

A **distributed lock** is a synchronization mechanism that ensures only one process can access a shared resource across multiple machines/nodes.

## Use Cases

1. **Prevent duplicate job execution**: Cron jobs in multiple servers
2. **Leader election**: Designate one coordinator
3. **Resource allocation**: Exclusive access to shared resource
4. **Data consistency**: Prevent concurrent modifications

## Requirements

1. **Mutual Exclusion**: Only one client holds lock at a time
2. **Deadlock Free**: Always possible to acquire lock eventually
3. **Fault Tolerance**: Handle process crashes, network partitions
4. **Fair**: No starvation

## Implementation Approaches

### 1. Redis-Based Distributed Lock (Redlock)

```python
import redis
import time
import uuid

class RedisDistributedLock:
    """
    Distributed lock using Redis

    Uses SET with NX (not exists) and PX (expiration)
    """

    def __init__(self, redis_client, resource_name, ttl=30000):
        """
        Args:
            redis_client: Redis connection
            resource_name: Name of resource to lock
            ttl: Time-to-live in milliseconds
        """
        self.redis = redis_client
        self.resource_name = f"lock:{resource_name}"
        self.ttl = ttl
        self.lock_id = str(uuid.uuid4())  # Unique ID for this lock instance
        self.is_locked = False

    def acquire(self, timeout=10):
        """
        Acquire lock with timeout

        Returns True if lock acquired, False otherwise
        """
        end_time = time.time() + timeout

        while time.time() < end_time:
            # SET key value NX PX milliseconds
            # NX: Only set if key doesn't exist
            # PX: Set expiration in milliseconds
            result = self.redis.set(
                self.resource_name,
                self.lock_id,
                nx=True,  # Only if not exists
                px=self.ttl  # Expiration in ms
            )

            if result:
                self.is_locked = True
                return True

            # Lock held by another process - wait a bit
            time.sleep(0.1)

        return False

    def release(self):
        """
        Release lock (only if we own it)

        Use Lua script for atomicity:
        1. Check if lock value matches our ID
        2. Delete lock if match
        """
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """

        result = self.redis.eval(lua_script, 1, self.resource_name, self.lock_id)

        if result:
            self.is_locked = False
            return True
        return False

    def extend(self, additional_time=30000):
        """
        Extend lock TTL (if we own it)
        """
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("pexpire", KEYS[1], ARGV[2])
        else
            return 0
        end
        """

        return self.redis.eval(
            lua_script,
            1,
            self.resource_name,
            self.lock_id,
            additional_time
        )

    def __enter__(self):
        """Context manager support"""
        if not self.acquire():
            raise RuntimeError("Failed to acquire lock")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        """Release lock on exit"""
        self.release()

# Usage Example
redis_client = redis.Redis(host='localhost', port=6379)

# Using context manager
with RedisDistributedLock(redis_client, "my_resource") as lock:
    # Critical section
    print("Doing work with exclusive lock")
    time.sleep(1)
# Lock automatically released

# Manual acquire/release
lock = RedisDistributedLock(redis_client, "my_resource")
if lock.acquire(timeout=5):
    try:
        # Critical section
        print("Doing work")
    finally:
        lock.release()
```

### 2. Redlock Algorithm (Multiple Redis Instances)

```python
import redis
import time

class Redlock:
    """
    Redlock algorithm for distributed lock across multiple Redis instances

    More fault-tolerant than single Redis instance
    Requires majority (N/2 + 1) of instances to acquire lock
    """

    def __init__(self, redis_instances, resource_name, ttl=30000):
        """
        Args:
            redis_instances: List of Redis connections
            resource_name: Resource to lock
            ttl: Lock expiration in ms
        """
        self.redis_instances = redis_instances
        self.resource_name = f"lock:{resource_name}"
        self.ttl = ttl
        self.lock_id = str(uuid.uuid4())
        self.quorum = len(redis_instances) // 2 + 1

    def acquire(self, timeout=10):
        """
        Acquire lock from majority of Redis instances
        """
        start_time = time.time()
        end_time = start_time + timeout

        while time.time() < end_time:
            locked_instances = 0
            lock_start = time.time()

            # Try to acquire lock on all instances
            for redis_client in self.redis_instances:
                try:
                    result = redis_client.set(
                        self.resource_name,
                        self.lock_id,
                        nx=True,
                        px=self.ttl
                    )
                    if result:
                        locked_instances += 1
                except:
                    # Instance unavailable
                    pass

            # Calculate elapsed time
            elapsed = (time.time() - lock_start) * 1000  # ms

            # Check if we got majority and lock is still valid
            if locked_instances >= self.quorum and elapsed < self.ttl:
                return True

            # Failed to acquire - release locks we did get
            self._release_all()

            # Wait before retrying
            time.sleep(0.1)

        return False

    def release(self):
        """Release lock from all instances"""
        self._release_all()

    def _release_all(self):
        """Release lock from all Redis instances"""
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """

        for redis_client in self.redis_instances:
            try:
                redis_client.eval(lua_script, 1, self.resource_name, self.lock_id)
            except:
                # Instance unavailable - best effort
                pass

# Usage
redis_instances = [
    redis.Redis(host='redis1.example.com', port=6379),
    redis.Redis(host='redis2.example.com', port=6379),
    redis.Redis(host='redis3.example.com', port=6379),
    redis.Redis(host='redis4.example.com', port=6379),
    redis.Redis(host='redis5.example.com', port=6379),
]

lock = Redlock(redis_instances, "critical_resource")
if lock.acquire():
    try:
        # Critical section
        print("Acquired lock across majority of Redis instances")
    finally:
        lock.release()
```

### 3. ZooKeeper-Based Lock

```python
from kazoo.client import KazooClient
from kazoo.exceptions import NodeExistsError

class ZooKeeperLock:
    """
    Distributed lock using ZooKeeper

    Uses ephemeral sequential nodes for lock queue
    """

    def __init__(self, zk_hosts, lock_path):
        """
        Args:
            zk_hosts: ZooKeeper connection string
            lock_path: Path for lock (e.g., /locks/my_resource)
        """
        self.zk = KazooClient(hosts=zk_hosts)
        self.lock_path = lock_path
        self.node_path = None

    def acquire(self, timeout=10):
        """
        Acquire lock using ZooKeeper

        Algorithm:
        1. Create ephemeral sequential node
        2. Get all children
        3. If we have smallest number, we have lock
        4. Otherwise, watch next smaller node
        """
        self.zk.start()

        # Ensure lock path exists
        self.zk.ensure_path(self.lock_path)

        # Create ephemeral sequential node
        self.node_path = self.zk.create(
            f"{self.lock_path}/lock_",
            ephemeral=True,
            sequence=True
        )

        while True:
            # Get all lock nodes
            children = sorted(self.zk.get_children(self.lock_path))

            # Extract our sequence number
            our_node = self.node_path.split('/')[-1]

            # If we have smallest number, we have lock
            if children[0] == our_node:
                return True

            # Otherwise, watch next smaller node
            index = children.index(our_node)
            prev_node = children[index - 1]

            # Wait for previous node to be deleted
            @self.zk.DataWatch(f"{self.lock_path}/{prev_node}")
            def watch_prev_node(data, stat):
                if stat is None:  # Node deleted
                    return False  # Stop watching
                return True  # Continue watching

            # Check again (prev node might have been deleted already)
            if our_node == sorted(self.zk.get_children(self.lock_path))[0]:
                return True

    def release(self):
        """Release lock by deleting our node"""
        if self.node_path:
            try:
                self.zk.delete(self.node_path)
            except:
                pass  # Node might already be deleted

        self.zk.stop()

# Usage
lock = ZooKeeperLock("localhost:2181", "/locks/my_resource")
if lock.acquire():
    try:
        print("Acquired ZooKeeper lock")
    finally:
        lock.release()
```

### 4. Database-Based Lock

```python
import psycopg2
import time

class DatabaseLock:
    """
    Distributed lock using PostgreSQL

    Uses advisory locks or row-level locking
    """

    def __init__(self, db_connection, lock_name):
        self.conn = db_connection
        self.lock_id = hash(lock_name) % (2**31)  # Convert to int
        self.is_locked = False

    def acquire(self, timeout=10):
        """
        Acquire lock using PostgreSQL advisory lock

        pg_try_advisory_lock: Non-blocking
        pg_advisory_lock: Blocking
        """
        cursor = self.conn.cursor()
        end_time = time.time() + timeout

        while time.time() < end_time:
            cursor.execute(
                "SELECT pg_try_advisory_lock(%s)",
                (self.lock_id,)
            )
            result = cursor.fetchone()[0]

            if result:
                self.is_locked = True
                return True

            time.sleep(0.1)

        return False

    def release(self):
        """Release advisory lock"""
        if not self.is_locked:
            return

        cursor = self.conn.cursor()
        cursor.execute(
            "SELECT pg_advisory_unlock(%s)",
            (self.lock_id,)
        )
        self.is_locked = False

    def __enter__(self):
        if not self.acquire():
            raise RuntimeError("Failed to acquire database lock")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.release()

# Usage
import psycopg2

conn = psycopg2.connect("dbname=mydb user=user password=pass")

with DatabaseLock(conn, "my_resource") as lock:
    print("Doing work with database lock")
```

## Common Pitfalls

### 1. Lock Expiration

**Problem**: Process holds lock, then becomes slow. Lock expires while still working.

**Solution**:
- Use reasonable TTL
- Extend lock if work takes longer
- Check if we still own lock before critical operations

```python
class LockWithExtension:
    def do_long_work(self):
        lock = RedisDistributedLock(redis_client, "resource", ttl=5000)

        if lock.acquire():
            try:
                for i in range(100):
                    # Do work
                    process_item(i)

                    # Extend lock every 10 iterations
                    if i % 10 == 0:
                        lock.extend(additional_time=5000)
            finally:
                lock.release()
```

### 2. Deadlock

**Problem**: Two processes waiting for each other's locks.

**Solution**:
- Always acquire locks in same order
- Use timeout
- Deadlock detection

### 3. Fencing Tokens

**Problem**: Process thinks it has lock after it expired.

**Solution**: Use monotonically increasing token

```python
class LockWithFencing:
    """
    Lock with fencing token to prevent stale operations
    """

    def __init__(self, redis_client, resource_name):
        self.redis = redis_client
        self.resource_name = resource_name
        self.token_key = f"fence:{resource_name}"
        self.lock_id = str(uuid.uuid4())

    def acquire(self):
        # Increment fencing token
        token = self.redis.incr(self.token_key)

        result = self.redis.set(
            f"lock:{self.resource_name}",
            self.lock_id,
            nx=True,
            px=30000
        )

        if result:
            self.fencing_token = token
            return token

        return None

    def execute_with_token(self, operation, token):
        """
        Execute operation only if token is current

        Resource checks token before accepting operation
        """
        if token != self.get_current_token():
            raise ValueError("Stale token - lock expired")

        operation()

    def get_current_token(self):
        return int(self.redis.get(self.token_key))
```

## Comparison

| Approach | Pros | Cons | Use When |
|----------|------|------|----------|
| **Redis** | Fast, simple | Single point of failure | Low criticality, high performance |
| **Redlock** | More fault-tolerant | Complex, requires multiple Redis | Higher reliability needs |
| **ZooKeeper** | Strong consistency, built-in | Heavyweight, steep learning curve | Critical locks, need ordering |
| **Database** | Simple, existing infrastructure | Slower, DB load | Moderate load, existing DB |

## Interview Tips

### Common Questions

**Q: How do you prevent deadlock?**
- Acquire locks in consistent order
- Use timeouts
- Deadlock detection algorithm

**Q: What if lock holder crashes?**
- Use TTL/expiration
- Ephemeral nodes (ZooKeeper)
- Health checks

**Q: How to ensure fairness?**
- FIFO queue (ZooKeeper sequential nodes)
- Timestamp-based ordering
- Fair queuing in application layer

**Q: Redis vs ZooKeeper for locks?**
- Redis: Faster, simpler, less consistent
- ZooKeeper: Slower, complex, strongly consistent
- Choose based on CAP requirements

## Key Takeaways

1. **Expiration is critical**: Prevent deadlocks from crashes
2. **Unique lock IDs**: Prevent releasing others' locks
3. **Atomic operations**: Use Lua scripts for Redis
4. **Fencing tokens**: Prevent stale lock operations
5. **Trade-offs**: Consistency vs availability vs performance

Distributed locks are fundamental for coordination in distributed systems!
