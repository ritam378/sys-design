# Caching Strategies

## Overview

Caching is storing frequently accessed data in a fast storage layer (memory) to reduce latency and database load. Choosing the right caching strategy is critical for system performance.

**Why Cache?**
- **Reduce latency:** RAM is 100x faster than disk
- **Reduce load:** Fewer database queries
- **Reduce costs:** Less compute resources needed
- **Improve UX:** Faster page loads

**Typical Performance:**
- RAM (Cache): 1-10 microseconds
- SSD (Database): 50-150 microseconds
- HDD (Database): 5-15 milliseconds
- Network (API call): 10-100 milliseconds

---

## Cache Reading Strategies

### 1. Cache-Aside (Lazy Loading)

**How It Works:**
1. Application checks cache first
2. If **cache hit** → return data
3. If **cache miss** → query database, store in cache, return data

**Flow:**
```
Application → Check Cache
    ↓ Miss
Query Database → Store in Cache → Return to App
```

**Implementation:**
```python
def get_user(user_id):
    # 1. Try cache first
    cache_key = f"user:{user_id}"
    user = cache.get(cache_key)

    if user is not None:
        # Cache hit
        return user

    # 2. Cache miss - query database
    user = database.query("SELECT * FROM users WHERE id = ?", user_id)

    if user:
        # 3. Store in cache
        cache.set(cache_key, user, ttl=3600)  # 1 hour TTL

    return user
```

**Pros:**
- ✅ Only requested data is cached (efficient memory usage)
- ✅ Cache failures don't break application (fallback to DB)
- ✅ Simple to implement

**Cons:**
- ❌ Initial requests are slow (cold start)
- ❌ Risk of stale data
- ❌ Cache miss penalty (latency spike)

**Use Cases:**
- Read-heavy applications
- User profiles, product catalogs
- Session data

**Real-World Example:**
```python
# Reddit - caching hot posts
def get_hot_posts(subreddit, limit=25):
    cache_key = f"hot_posts:{subreddit}:{limit}"
    posts = redis.get(cache_key)

    if posts:
        return json.loads(posts)

    # Query database
    posts = db.query(
        "SELECT * FROM posts WHERE subreddit = ? ORDER BY score DESC LIMIT ?",
        subreddit, limit
    )

    # Cache for 5 minutes
    redis.setex(cache_key, 300, json.dumps(posts))
    return posts
```

---

### 2. Read-Through Cache

**How It Works:**
1. Application always queries cache
2. Cache automatically loads data from database on miss
3. Cache manages database interaction (not application)

**Flow:**
```
Application → Cache
              ↓ (cache handles DB query transparently)
           Database
```

**Implementation:**
```python
# Cache library handles read-through automatically
cache = ReadThroughCache(
    backend=redis_client,
    data_source=lambda key: database.get(key),
    ttl=3600
)

# Application code is simple
def get_user(user_id):
    # Cache handles miss automatically
    return cache.get(f"user:{user_id}")
```

**With cachetools library:**
```python
from cachetools import TTLCache, cached

# Database function
def get_user_from_db(user_id):
    return database.query("SELECT * FROM users WHERE id = ?", user_id)

# Read-through cache decorator
cache = TTLCache(maxsize=1000, ttl=3600)

@cached(cache, key=lambda user_id: user_id)
def get_user(user_id):
    # Automatically cached, loads from DB on miss
    return get_user_from_db(user_id)
```

**Pros:**
- ✅ Simplified application code
- ✅ Consistent caching logic
- ✅ Cache manages lifecycle

**Cons:**
- ❌ Tight coupling with cache library
- ❌ Limited flexibility
- ❌ Cold start problem remains

**Use Cases:**
- When cache library supports read-through
- Simplified application logic preferred
- Consistent data access pattern

---

### 3. Refresh-Ahead (Pre-emptive Refresh)

**How It Works:**
1. Cache automatically refreshes data **before** it expires
2. Prevents cache misses for popular data
3. Based on access patterns

**Flow:**
```
Cache → Detect TTL expiring soon
      → Pre-fetch from DB
      → Update cache
      → User never experiences miss
```

**Implementation:**
```python
class RefreshAheadCache:
    def __init__(self, redis_client, db, ttl=3600, refresh_threshold=0.8):
        self.redis = redis_client
        self.db = db
        self.ttl = ttl
        self.refresh_threshold = refresh_threshold  # Refresh at 80% TTL

    def get(self, key):
        # Get value and TTL
        value = self.redis.get(key)
        ttl_remaining = self.redis.ttl(key)

        if value is None:
            # Cache miss - load from DB
            value = self.db.get(key)
            self.redis.setex(key, self.ttl, value)
            return value

        # Check if needs refresh
        if ttl_remaining < (self.ttl * self.refresh_threshold):
            # Asynchronously refresh (don't block)
            self._refresh_async(key)

        return value

    def _refresh_async(self, key):
        # Background task
        threading.Thread(target=self._refresh, args=(key,)).start()

    def _refresh(self, key):
        value = self.db.get(key)
        self.redis.setex(key, self.ttl, value)

# Usage
cache = RefreshAheadCache(redis_client, database, ttl=3600)
user = cache.get("user:123")  # Refreshes at ~48min if accessed
```

**Pros:**
- ✅ No cache miss latency for hot data
- ✅ Always fresh data
- ✅ Better user experience

**Cons:**
- ❌ Wasted refreshes for unpopular data
- ❌ Increased database load
- ❌ Complex implementation

**Use Cases:**
- High-traffic applications (Reddit front page)
- Predictable access patterns
- Latency-sensitive applications

---

## Cache Writing Strategies

### 1. Write-Through Cache

**How It Works:**
1. Application writes to cache
2. Cache **synchronously** writes to database
3. Write completes when both succeed

**Flow:**
```
Application → Cache → Database (sync)
              ↓
         Return to App (after DB write)
```

**Implementation:**
```python
def update_user(user_id, data):
    cache_key = f"user:{user_id}"

    # 1. Write to cache
    cache.set(cache_key, data, ttl=3600)

    # 2. Write to database (synchronous)
    database.update("users", user_id, data)

    # 3. Return after both complete
    return data
```

**Pros:**
- ✅ Cache and database always in sync
- ✅ No stale reads
- ✅ Data durability guaranteed

**Cons:**
- ❌ Higher write latency (two writes)
- ❌ Wasted cache space (write-once data)
- ❌ Cache failure breaks writes

**Use Cases:**
- Strong consistency required
- Read-heavy with occasional writes
- Critical data (user profiles)

---

### 2. Write-Behind (Write-Back) Cache

**How It Works:**
1. Application writes to cache
2. Cache **asynchronously** writes to database (later)
3. Write returns immediately

**Flow:**
```
Application → Cache → Return immediately
              ↓
          Queue/Buffer
              ↓ (async, batched)
          Database
```

**Implementation:**
```python
import queue
import threading

class WriteBehindCache:
    def __init__(self, cache, db):
        self.cache = cache
        self.db = db
        self.write_queue = queue.Queue()

        # Background worker
        self.worker = threading.Thread(target=self._process_writes)
        self.worker.daemon = True
        self.worker.start()

    def set(self, key, value, ttl=3600):
        # 1. Write to cache immediately
        self.cache.set(key, value, ttl)

        # 2. Queue database write
        self.write_queue.put((key, value))

        # 3. Return immediately (don't wait for DB)
        return True

    def _process_writes(self):
        while True:
            # Batch writes for efficiency
            batch = []
            while len(batch) < 100:  # Batch size
                try:
                    item = self.write_queue.get(timeout=1)
                    batch.append(item)
                except queue.Empty:
                    break

            if batch:
                # Batch write to database
                self.db.batch_write(batch)

# Usage
cache = WriteBehindCache(redis_client, database)
cache.set("user:123", user_data)  # Returns immediately
```

**Pros:**
- ✅ Very low write latency
- ✅ Batch writes (better database performance)
- ✅ Reduces database load

**Cons:**
- ❌ Risk of data loss (if cache crashes before DB write)
- ❌ Eventual consistency
- ❌ Complex to implement

**Use Cases:**
- High-write applications (gaming leaderboards)
- Can tolerate data loss
- Database is write bottleneck

**Real-World Example:**
- **Write counters in Redis:** Page views, likes (eventually synced to DB)

---

### 3. Write-Around Cache

**How It Works:**
1. Application writes directly to database (bypass cache)
2. Read operations use cache-aside pattern
3. Cache populated on first read

**Flow:**
```
Write: Application → Database (skip cache)
Read:  Application → Cache → Database (cache-aside)
```

**Implementation:**
```python
def update_user(user_id, data):
    # Write directly to database
    database.update("users", user_id, data)

    # Optionally invalidate cache
    cache.delete(f"user:{user_id}")

    return data

def get_user(user_id):
    # Cache-aside for reads
    cache_key = f"user:{user_id}"
    user = cache.get(cache_key)

    if user is None:
        user = database.query("SELECT * FROM users WHERE id = ?", user_id)
        cache.set(cache_key, user, ttl=3600)

    return user
```

**Pros:**
- ✅ No wasted cache space (write-once data)
- ✅ Simple implementation
- ✅ Cache and DB can't be out of sync on write

**Cons:**
- ❌ Cache miss after every write
- ❌ Read latency after writes
- ❌ Cache might be stale

**Use Cases:**
- Write-once, read-rarely data
- Large data objects
- Log data, analytics events

---

## Cache Invalidation Strategies

> "There are only two hard things in Computer Science: cache invalidation and naming things." - Phil Karlton

### 1. Time-Based (TTL)

**How It Works:** Cache entries expire after fixed time.

```python
# Set TTL when caching
cache.setex("user:123", 3600, user_data)  # Expires in 1 hour

# Redis automatically deletes after TTL
```

**Pros:**
- ✅ Simple
- ✅ Automatic cleanup
- ✅ Predictable

**Cons:**
- ❌ Stale data until expiration
- ❌ Wasted memory for inactive data
- ❌ Hard to choose right TTL

**TTL Guidelines:**
- **Static data:** 24 hours - 7 days
- **User profiles:** 1-4 hours
- **Product catalogs:** 15 minutes - 1 hour
- **Real-time data:** 1-5 minutes
- **Frequently updated:** 30 seconds - 2 minutes

### 2. Event-Based Invalidation

**How It Works:** Invalidate cache when data changes.

```python
def update_user(user_id, data):
    # Update database
    database.update("users", user_id, data)

    # Invalidate cache
    cache.delete(f"user:{user_id}")

    # Or update cache with new data
    cache.set(f"user:{user_id}", data, ttl=3600)
```

**Pros:**
- ✅ No stale data
- ✅ Always fresh
- ✅ Efficient

**Cons:**
- ❌ Must track all writes
- ❌ Complex with multiple writers
- ❌ Race conditions possible

**Advanced: Pub/Sub Invalidation**
```python
# Service A updates user
def update_user(user_id, data):
    database.update("users", user_id, data)

    # Publish invalidation event
    redis.publish("cache:invalidate", json.dumps({
        "key": f"user:{user_id}"
    }))

# Service B subscribes
pubsub = redis.pubsub()
pubsub.subscribe("cache:invalidate")

for message in pubsub.listen():
    data = json.loads(message['data'])
    local_cache.delete(data['key'])
```

### 3. Version-Based (Versioned Keys)

**How It Works:** Include version in cache key.

```python
# Version in application config or database
USER_SCHEMA_VERSION = "v2"

def get_user(user_id):
    cache_key = f"user:{USER_SCHEMA_VERSION}:{user_id}"
    # ...

# When schema changes, increment version
# Old cache entries automatically obsolete
USER_SCHEMA_VERSION = "v3"
```

**Pros:**
- ✅ No invalidation needed
- ✅ Zero-downtime deploys
- ✅ Can compare versions

**Cons:**
- ❌ Wasted memory (old versions)
- ❌ Requires cleanup
- ❌ More complex keys

### 4. Cache Tagging

**How It Works:** Tag related cache entries, invalidate by tag.

```python
# Redis example with sets
def set_with_tag(key, value, tags, ttl=3600):
    # Store value
    redis.setex(key, ttl, value)

    # Add to tag sets
    for tag in tags:
        redis.sadd(f"tag:{tag}", key)

def invalidate_by_tag(tag):
    # Get all keys with tag
    keys = redis.smembers(f"tag:{tag}")

    # Delete all
    for key in keys:
        redis.delete(key)

    # Remove tag
    redis.delete(f"tag:{tag}")

# Usage
set_with_tag("user:123", user_data, tags=["users", "active_users"])
set_with_tag("user:456", user_data, tags=["users", "premium_users"])

# Invalidate all premium users
invalidate_by_tag("premium_users")
```

**Pros:**
- ✅ Bulk invalidation
- ✅ Flexible grouping
- ✅ Efficient

**Cons:**
- ❌ More complex
- ❌ Memory overhead (tags)

---

## Cache Eviction Policies

When cache is full, which data to remove?

### 1. LRU (Least Recently Used)
Remove data not accessed for longest time.

**Best for:** General-purpose caching

```python
# Redis
maxmemory-policy allkeys-lru
```

### 2. LFU (Least Frequently Used)
Remove data accessed least often.

**Best for:** Long-running cache, popularity-based

```python
# Redis
maxmemory-policy allkeys-lfu
```

### 3. FIFO (First In, First Out)
Remove oldest data.

**Best for:** Time-series data

### 4. Random
Remove random entry.

**Best for:** Uniform access patterns

### 5. TTL-based
Remove entries closest to expiration.

**Best for:** Time-sensitive data

---

## Advanced Caching Patterns

### 1. Multi-Level Caching

```
Application
    ↓
L1: In-Memory Cache (local, fastest)
    ↓ miss
L2: Redis Cache (shared, fast)
    ↓ miss
L3: CDN Cache (global, medium)
    ↓ miss
Database (slowest)
```

**Implementation:**
```python
def get_user(user_id):
    # L1: Check local cache (dict, LRU cache)
    if user_id in local_cache:
        return local_cache[user_id]

    # L2: Check Redis
    cache_key = f"user:{user_id}"
    user = redis.get(cache_key)

    if user:
        local_cache[user_id] = user  # Populate L1
        return user

    # L3: Database
    user = database.query("SELECT * FROM users WHERE id = ?", user_id)

    # Populate caches
    redis.setex(cache_key, 3600, user)
    local_cache[user_id] = user

    return user
```

### 2. Cache Stampede Prevention

**Problem:** Cache expires, many requests hit database simultaneously.

```
Cache expires
    ↓
1000 requests → Database (overload!)
```

**Solution: Locking**
```python
import threading

locks = {}

def get_user_safe(user_id):
    cache_key = f"user:{user_id}"
    user = cache.get(cache_key)

    if user is not None:
        return user

    # Acquire lock
    lock = locks.setdefault(user_id, threading.Lock())

    with lock:
        # Double-check (maybe another thread loaded it)
        user = cache.get(cache_key)
        if user is not None:
            return user

        # Load from database (only one thread does this)
        user = database.query("SELECT * FROM users WHERE id = ?", user_id)
        cache.set(cache_key, user, ttl=3600)

    return user
```

**Solution: Probabilistic Early Expiration**
```python
import random

def get_user_probabilistic(user_id):
    cache_key = f"user:{user_id}"
    user, ttl = cache.get_with_ttl(cache_key)

    if user is None:
        user = database.query("SELECT * FROM users WHERE id = ?", user_id)
        cache.setex(cache_key, 3600, user)
        return user

    # Probabilistically refresh early
    # More likely to refresh as TTL decreases
    if random.random() < (1 - ttl / 3600):
        # Refresh asynchronously
        threading.Thread(target=refresh_cache, args=(user_id,)).start()

    return user
```

### 3. Cache Warming

**Preload cache before traffic arrives.**

```python
def warm_cache():
    # Popular products
    popular_product_ids = get_trending_products()

    for product_id in popular_product_ids:
        product = database.query("SELECT * FROM products WHERE id = ?", product_id)
        cache.setex(f"product:{product_id}", 3600, product)

# Run on application startup
warm_cache()

# Or scheduled job
schedule.every().day.at("02:00").do(warm_cache)
```

---

## Caching Best Practices

### 1. What to Cache
✅ **Good candidates:**
- Expensive database queries
- API responses
- Computed results
- User sessions
- Static content

❌ **Poor candidates:**
- Frequently changing data
- User-specific sensitive data
- Large objects (> 1MB)
- Rarely accessed data

### 2. Cache Key Design

```python
# Good: Hierarchical, readable
f"user:{user_id}:profile"
f"post:{post_id}:comments:page:{page_num}"
f"product:{category}:{subcategory}:list"

# Bad: Unclear, hard to debug
f"{user_id}_data"
f"cache123"
```

### 3. Monitoring

```python
# Track hit/miss ratio
cache_hits = metrics.counter('cache.hits')
cache_misses = metrics.counter('cache.misses')

def get_with_metrics(key):
    value = cache.get(key)
    if value:
        cache_hits.inc()
    else:
        cache_misses.inc()
    return value

# Alert if hit ratio < 70%
if (cache_hits / (cache_hits + cache_misses)) < 0.7:
    alert("Low cache hit ratio")
```

### 4. Failover

```python
def get_user_resilient(user_id):
    try:
        # Try cache
        cache_key = f"user:{user_id}"
        user = cache.get(cache_key)

        if user:
            return user
    except CacheConnectionError:
        # Cache down, log and continue
        logger.warning("Cache unavailable, falling back to database")

    # Always fallback to database
    return database.query("SELECT * FROM users WHERE id = ?", user_id)
```

---

## Summary

**Reading Strategies:**
- **Cache-Aside:** Most common, application manages cache
- **Read-Through:** Cache manages database reads
- **Refresh-Ahead:** Proactive refresh for hot data

**Writing Strategies:**
- **Write-Through:** Sync write to cache and DB (consistent)
- **Write-Behind:** Async write to DB (fast, risky)
- **Write-Around:** Bypass cache on writes

**Invalidation:**
- **TTL:** Time-based expiration
- **Event-Based:** Invalidate on change
- **Versioning:** Version in key

**Eviction Policies:**
- **LRU:** Most common
- **LFU:** Frequency-based
- **TTL:** Time-based

**Key Metrics:**
- Cache hit ratio (target: > 80%)
- Average latency
- Memory usage
- Eviction rate

**Interview Tip:** Always discuss trade-offs (consistency vs latency, complexity vs performance) and mention specific use cases.

---

**Next:** Learn about [CDN Caching](cdn.md) and [Redis vs Memcached](redis-vs-memcached.md).
