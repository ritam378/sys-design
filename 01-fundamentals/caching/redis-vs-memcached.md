# Redis vs Memcached

## Overview

Both **Redis** and **Memcached** are in-memory key-value stores used for caching, but they have different features, use cases, and performance characteristics.

**Quick Comparison:**

| Feature | Redis | Memcached |
|---------|-------|-----------|
| **Data Structures** | Rich (strings, lists, sets, hashes, sorted sets) | Simple (strings only) |
| **Persistence** | Optional (RDB, AOF) | None (in-memory only) |
| **Replication** | Yes (master-slave) | No (use client-side) |
| **Clustering** | Native Redis Cluster | Use consistent hashing |
| **Threading** | Single-threaded (event loop) | Multi-threaded |
| **Memory Efficiency** | Less efficient | More efficient |
| **Use Cases** | Cache + data structures + pub/sub | Pure caching only |

---

## Memcached

### What is Memcached?

A **simple, high-performance distributed memory caching system**. Designed for one purpose: caching.

**Philosophy:** Keep it simple, do one thing well.

### Key Characteristics

#### 1. Simple Key-Value Store

```python
import memcache

mc = memcache.Client(['127.0.0.1:11211'])

# Set value
mc.set('user:123', {'name': 'Alice', 'age': 30}, time=3600)

# Get value
user = mc.get('user:123')
print(user)  # {'name': 'Alice', 'age': 30}

# Delete
mc.delete('user:123')
```

**Supported Operations:**
- `set(key, value, time)` - Store value
- `get(key)` - Retrieve value
- `delete(key)` - Remove value
- `add(key, value)` - Add if not exists
- `replace(key, value)` - Replace if exists
- `incr(key, delta)` - Increment counter
- `decr(key, delta)` - Decrement counter

#### 2. Multi-Threaded

```
Memcached process
├─ Thread 1 (handles connections)
├─ Thread 2 (handles connections)
├─ Thread 3 (handles connections)
└─ Thread 4 (handles connections)

Can utilize multiple CPU cores
```

**Performance:**
- Very fast for simple get/set operations
- Better at handling many concurrent connections
- Can saturate network before CPU

#### 3. LRU Eviction

```
Memory: 1GB (full)
New data arrives → Evict least recently used item
```

**Slab Allocator:**
```
Memory divided into slabs (chunks)
- Slab 1: 80 bytes
- Slab 2: 160 bytes
- Slab 3: 320 bytes
...

Reduces fragmentation
```

#### 4. No Persistence

All data lost on restart.

```bash
# Restart memcached
sudo systemctl restart memcached

# All cache data is gone!
```

#### 5. Distributed (Client-Side)

```python
# Multiple memcached servers
mc = memcache.Client([
    '192.168.1.1:11211',
    '192.168.1.2:11211',
    '192.168.1.3:11211'
])

# Client uses consistent hashing to determine which server
mc.set('user:123', data)  # Goes to server based on key hash
```

**Consistent Hashing:**
```
key 'user:123' → hash → server 2
key 'user:456' → hash → server 1
key 'user:789' → hash → server 3
```

### Pros and Cons

**Pros:**
- ✅ Extremely simple
- ✅ Multi-threaded (better CPU utilization)
- ✅ Memory efficient
- ✅ Fast for pure caching
- ✅ Mature and stable

**Cons:**
- ❌ No data structures (strings only)
- ❌ No persistence
- ❌ No replication (client handles distribution)
- ❌ Limited functionality
- ❌ No pub/sub or advanced features

### Use Cases

✅ **Perfect For:**
1. **Simple caching** - Session data, page fragments
2. **High concurrency** - Many simultaneous connections
3. **Pure cache** - Don't need persistence
4. **Horizontal scaling** - Multiple servers with consistent hashing

❌ **Not For:**
1. Data structures (lists, sets)
2. Persistence required
3. Pub/sub messaging
4. Complex operations

---

## Redis

### What is Redis?

A **versatile in-memory data structure store** that can be used as cache, database, message broker, and more.

**Philosophy:** Swiss Army knife of in-memory stores.

### Key Characteristics

#### 1. Rich Data Structures

**Strings:**
```python
import redis

r = redis.Redis(host='localhost', port=6379)

# String operations
r.set('user:123:name', 'Alice')
r.get('user:123:name')  # b'Alice'

# Counters
r.incr('page:views')
r.incrby('page:views', 10)
```

**Lists (FIFO/LIFO queues):**
```python
# Push to list
r.lpush('tasks', 'task1')
r.lpush('tasks', 'task2')

# Pop from list
task = r.rpop('tasks')  # b'task1'

# Queue length
r.llen('tasks')
```

**Sets (unique values):**
```python
# Add to set
r.sadd('tags:post1', 'python', 'redis', 'caching')

# Check membership
r.sismember('tags:post1', 'python')  # True

# Set operations
r.sinter('tags:post1', 'tags:post2')  # Intersection
r.sunion('tags:post1', 'tags:post2')  # Union
```

**Hashes (objects):**
```python
# Store object as hash
r.hset('user:123', mapping={
    'name': 'Alice',
    'age': 30,
    'email': 'alice@example.com'
})

# Get field
r.hget('user:123', 'name')  # b'Alice'

# Get all fields
r.hgetall('user:123')
```

**Sorted Sets (leaderboards):**
```python
# Add scores
r.zadd('leaderboard', {'player1': 100, 'player2': 200, 'player3': 150})

# Get rank
r.zrank('leaderboard', 'player2')  # 2 (highest score)

# Get top 10
r.zrevrange('leaderboard', 0, 9, withscores=True)
```

#### 2. Persistence Options

**RDB (Snapshotting):**
```bash
# redis.conf
save 900 1      # Save if 1 key changed in 900 seconds
save 300 10     # Save if 10 keys changed in 300 seconds
save 60 10000   # Save if 10000 keys changed in 60 seconds
```

**AOF (Append-Only File):**
```bash
# redis.conf
appendonly yes
appendfsync everysec  # Fsync every second (balance safety/performance)
```

**Comparison:**
```
RDB: Fast restart, larger files, data loss possible
AOF: Smaller data loss, slower restart, larger files
Hybrid: RDB + AOF (Redis 4.0+)
```

#### 3. Master-Slave Replication

```
Master (writes)
    ↓ (replicates)
Slave 1 (reads)
Slave 2 (reads)
Slave 3 (reads)
```

**Setup:**
```bash
# On slave server
redis-cli
> REPLICAOF 192.168.1.1 6379
```

**Benefits:**
- Read scaling
- High availability
- Data redundancy

#### 4. Redis Cluster (Sharding)

```
Cluster: 6 nodes (3 masters, 3 slaves)

Master 1 (slots 0-5461)    → Slave 1
Master 2 (slots 5462-10922) → Slave 2
Master 3 (slots 10923-16383) → Slave 3
```

**Client automatically routes to correct node:**
```python
from rediscluster import RedisCluster

rc = RedisCluster(startup_nodes=[
    {"host": "127.0.0.1", "port": "7000"},
    {"host": "127.0.0.1", "port": "7001"},
    {"host": "127.0.0.1", "port": "7002"}
])

rc.set('user:123', 'data')  # Automatically routes to correct shard
```

#### 5. Pub/Sub Messaging

```python
# Publisher
r.publish('notifications', 'New message!')

# Subscriber
pubsub = r.pubsub()
pubsub.subscribe('notifications')

for message in pubsub.listen():
    print(message)
```

#### 6. Lua Scripting

```python
# Atomic operations with Lua
lua_script = """
local current = redis.call('GET', KEYS[1])
if current and tonumber(current) > tonumber(ARGV[1]) then
    return 0
else
    redis.call('SET', KEYS[1], ARGV[1])
    return 1
end
"""

# Execute script
r.eval(lua_script, 1, 'max_value', 100)
```

#### 7. Transactions

```python
# Pipeline multiple commands
pipe = r.pipeline()
pipe.set('user:123:name', 'Alice')
pipe.incr('user:count')
pipe.lpush('recent_users', 'user:123')
pipe.execute()  # All or nothing
```

### Pros and Cons

**Pros:**
- ✅ Rich data structures
- ✅ Persistence (RDB/AOF)
- ✅ Replication (master-slave)
- ✅ Clustering (native)
- ✅ Pub/sub messaging
- ✅ Lua scripting
- ✅ Transactions
- ✅ TTL on keys

**Cons:**
- ❌ Single-threaded (can't use multiple cores for one command)
- ❌ More memory overhead
- ❌ More complex
- ❌ Slower for pure caching vs Memcached (marginal)

### Use Cases

✅ **Perfect For:**
1. **Caching** with data structures
2. **Session store** with persistence
3. **Leaderboards** (sorted sets)
4. **Real-time analytics** (counters, sets)
5. **Message queues** (lists, pub/sub)
6. **Rate limiting** (sorted sets, TTL)

---

## Performance Comparison

### Benchmark Results

**Simple GET/SET operations:**
```
Memcached: ~100,000 ops/sec (multi-threaded, 4 cores)
Redis:     ~80,000 ops/sec (single-threaded)
```

**Why Memcached is faster for simple ops:**
- Multi-threaded (uses all CPU cores)
- Less overhead (no persistence checks)
- Optimized for one thing

**When Redis is faster:**
- Complex data structures (O(1) operations on lists, sets)
- Pipelining (batch commands)
- Lua scripts (atomic complex operations)

### Memory Efficiency

**Storing 1 million small objects:**
```
Memcached: ~50 MB (efficient slab allocator)
Redis:     ~80 MB (more overhead per key)
```

**Memcached advantages:**
- Better memory efficiency for small keys
- Slab allocator reduces fragmentation

**Redis advantages:**
- More efficient for complex data structures
- Better memory utilization for large values

---

## Detailed Comparison Table

| Feature | Redis | Memcached |
|---------|-------|-----------|
| **Performance (simple ops)** | ~80K ops/sec | ~100K ops/sec |
| **Data Types** | Strings, Lists, Sets, Hashes, Sorted Sets, Bitmaps, HyperLogLog, Streams | Strings only |
| **Max Key Size** | 512 MB | 250 bytes |
| **Max Value Size** | 512 MB | 1 MB |
| **Persistence** | RDB, AOF | None |
| **Replication** | Master-Slave | Client-side |
| **Clustering** | Native (Redis Cluster) | Client-side (consistent hashing) |
| **Threading Model** | Single-threaded (event loop) | Multi-threaded |
| **Transactions** | Yes (MULTI/EXEC) | No |
| **Pub/Sub** | Yes | No |
| **Lua Scripting** | Yes | No |
| **Eviction Policies** | 8 policies (LRU, LFU, TTL-based, random) | LRU only |
| **Geospatial** | Yes (GEO commands) | No |
| **TTL** | Per-key TTL | Global expiration |
| **Watch/CAS** | WATCH command | CAS (Compare-And-Swap) |
| **Memory Overhead** | Higher | Lower |
| **Learning Curve** | Steeper | Gentle |

---

## Migration Considerations

### Memcached → Redis

**Reasons to migrate:**
- Need data structures (lists, sets)
- Want persistence
- Need replication
- Want pub/sub

**Migration strategy:**
```python
# Dual-write pattern
def set_cache(key, value, ttl=3600):
    # Write to both
    memcached.set(key, value, ttl)
    redis.setex(key, ttl, value)

# Gradually move reads to Redis
def get_cache(key):
    # Try Redis first
    value = redis.get(key)
    if value:
        return value

    # Fallback to Memcached
    value = memcached.get(key)
    if value:
        # Backfill Redis
        redis.setex(key, 3600, value)

    return value
```

### Redis → Memcached

**Reasons to migrate:**
- Only need simple caching
- Want multi-threaded performance
- Reduce memory usage
- Simplify infrastructure

**Usually not recommended** - Redis can do everything Memcached can, plus more.

---

## Use Case Decision Matrix

| Use Case | Recommended | Why |
|----------|-------------|-----|
| **Simple page caching** | Either | Both work well |
| **Session storage** | Redis | Persistence + data structures |
| **Leaderboards** | Redis | Sorted sets |
| **Rate limiting** | Redis | Sorted sets + TTL |
| **Job queues** | Redis | Lists (LPUSH/RPOP) |
| **Counters** | Either | Both support INCR |
| **Pub/Sub** | Redis | Built-in pub/sub |
| **Geospatial** | Redis | GEO commands |
| **Full-text search** | Redis (RediSearch) | Module support |
| **High concurrency (10K+ conns)** | Memcached | Multi-threaded |
| **Memory constrained** | Memcached | More efficient |
| **Need persistence** | Redis | RDB/AOF |

---

## Real-World Examples

### Companies Using Memcached
- **Facebook:** Session caching (historically)
- **Wikipedia:** Page caching
- **Flickr:** Photo metadata caching
- **Twitter:** Timeline caching (historically)

### Companies Using Redis
- **Twitter:** Timeline, rate limiting
- **GitHub:** Job queues, caching
- **Stack Overflow:** Leaderboards, caching
- **Instagram:** Activity feeds, caching
- **Airbnb:** Session storage, caching
- **Uber:** Geospatial data, caching

### Trend

**Redis is replacing Memcached** in many organizations because:
- Handles simple caching just as well
- Provides additional features when needed
- Easier to operate (one system instead of multiple)

---

## Hybrid Approach

Some companies use **both**:

```
Redis: Complex use cases (leaderboards, queues, pub/sub)
Memcached: Simple caching (session data, page fragments)
```

**Example Architecture:**
```
Application
    ↓
Redis (data structures, pub/sub)
    ↓
Memcached (pure caching layer)
    ↓
Database
```

---

## Code Examples

### Session Storage

**Memcached:**
```python
# Simple string storage
session_data = json.dumps({
    'user_id': 123,
    'cart': ['item1', 'item2']
})
memcached.set(f'session:{session_id}', session_data, time=3600)

# Retrieve
data = json.loads(memcached.get(f'session:{session_id}'))
```

**Redis (better):**
```python
# Store as hash (no JSON serialization needed)
redis.hset(f'session:{session_id}', mapping={
    'user_id': 123,
    'cart': json.dumps(['item1', 'item2'])
})
redis.expire(f'session:{session_id}', 3600)

# Retrieve specific field
user_id = redis.hget(f'session:{session_id}', 'user_id')
```

### Leaderboard

**Memcached (complex, inefficient):**
```python
# Store entire leaderboard as JSON
leaderboard = json.loads(memcached.get('leaderboard'))
leaderboard.append({'player': 'Alice', 'score': 100})
leaderboard.sort(key=lambda x: x['score'], reverse=True)
memcached.set('leaderboard', json.dumps(leaderboard[:100]))
```

**Redis (native support):**
```python
# Add score
redis.zadd('leaderboard', {'Alice': 100})

# Get top 10
top_10 = redis.zrevrange('leaderboard', 0, 9, withscores=True)

# Get player rank
rank = redis.zrevrank('leaderboard', 'Alice')
```

### Rate Limiting

**Memcached (limited):**
```python
key = f'ratelimit:{user_id}:{current_minute}'
count = memcached.get(key) or 0

if count >= 100:
    raise RateLimitExceeded()

memcached.set(key, count + 1, time=60)
```

**Redis (sliding window):**
```python
import time

key = f'ratelimit:{user_id}'
now = time.time()
window = 60  # 60 seconds

# Remove old entries
redis.zremrangebyscore(key, 0, now - window)

# Count requests in window
count = redis.zcard(key)

if count >= 100:
    raise RateLimitExceeded()

# Add current request
redis.zadd(key, {str(now): now})
redis.expire(key, window)
```

---

## Best Practices

### Memcached
1. **Use for simple caching** - Strings only
2. **Consistent hashing** - For distribution
3. **Connection pooling** - Reuse connections
4. **Monitor hit ratio** - Should be > 80%
5. **Right-size memory** - Avoid evictions

### Redis
1. **Choose right data structure** - Use hashes for objects, not serialized strings
2. **Set TTLs** - Prevent memory leaks
3. **Use pipelining** - Batch commands
4. **Monitor slow log** - Identify slow queries
5. **Enable persistence** - RDB or AOF for important data
6. **Use connection pooling** - Reuse connections

---

## Summary

**Memcached:**
- Simple, fast, multi-threaded
- Pure caching only
- Memory efficient
- No persistence or replication
- **Choose when:** Simple caching, high concurrency, memory constrained

**Redis:**
- Rich data structures
- Persistence and replication
- Single-threaded
- More features (pub/sub, Lua, transactions)
- **Choose when:** Need data structures, persistence, or advanced features

**General Recommendation:**
- **Start with Redis** - Handles simple caching well + provides room to grow
- **Use Memcached** only if you specifically need multi-threaded performance or maximum memory efficiency for pure caching

**Interview Tip:**
- Don't say "Redis is always better" - explain trade-offs
- Mention specific use cases for each
- Discuss performance characteristics (single vs multi-threaded)
- Cover when you'd use both together

---

**Next:** Learn about [HTTP/HTTPS/WebSockets](../networking/http-https-websockets.md) protocols.
