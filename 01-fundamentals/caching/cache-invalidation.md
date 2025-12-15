# Cache Invalidation

## Overview

**Cache invalidation** is the process of removing or updating cached data when the underlying data changes. It's one of the hardest problems in computer science.

> "There are only two hard things in Computer Science: cache invalidation and naming things." - Phil Karlton

**Key Question:** "How do you ensure cached data stays consistent with the source of truth?"

**Short Answer:**
- **TTL (Time-To-Live):** Expire after time period
- **Write-Through:** Update cache immediately on write
- **Write-Behind:** Update cache asynchronously
- **Event-driven:** Invalidate on database changes

---

## The Problem

### Stale Data

```
Time 0: Database has user.age = 25
        Cache stores user.age = 25 ✅

Time 1: User updates age to 26 in database
        Database: user.age = 26
        Cache: user.age = 25 ❌ STALE!

Time 2: Application reads from cache
        Returns: age = 25 (wrong!)
```

**Impact:**
- ❌ Inconsistent data shown to users
- ❌ Business logic errors
- ❌ User confusion

---

## Invalidation Strategies

### 1. TTL (Time-To-Live)

**Expire cache entries after fixed time**

```python
import time

class TTLCache:
    """Cache with time-based expiration"""

    def __init__(self, ttl_seconds: int = 300):
        self.cache = {}  # key → (value, expiry_time)
        self.ttl = ttl_seconds

    def get(self, key: str):
        if key not in self.cache:
            return None

        value, expiry = self.cache[key]

        # Check if expired
        if time.time() > expiry:
            del self.cache[key]
            return None  # Expired

        return value

    def set(self, key: str, value):
        expiry = time.time() + self.ttl
        self.cache[key] = (value, expiry)

# Usage
cache = TTLCache(ttl_seconds=60)  # 1 minute TTL

cache.set("user:123", {"name": "Alice", "age": 25})

time.sleep(30)
cache.get("user:123")  # ✅ Returns data (not expired)

time.sleep(35)
cache.get("user:123")  # ❌ Returns None (expired)
```

**Pros:**
- ✅ Simple to implement
- ✅ Automatic cleanup
- ✅ Works without coordination

**Cons:**
- ❌ May serve stale data within TTL window
- ❌ Choosing TTL is hard (too short = cache misses, too long = stale data)
- ❌ Wastes resources (unnecessary fetches after expiry)

**When to use:**
- Data changes infrequently
- Some staleness acceptable
- Simple caching needs

---

### 2. Write-Through Cache

**Update cache synchronously when writing to database**

```python
class WriteThroughCache:
    """
    Synchronously update both cache and database
    """

    def __init__(self, cache, database):
        self.cache = cache
        self.db = database

    def get(self, key: str):
        """Read from cache first, fallback to DB"""
        # Try cache
        value = self.cache.get(key)
        if value is not None:
            return value

        # Cache miss: read from DB
        value = self.db.get(key)
        if value is not None:
            self.cache.set(key, value)

        return value

    def set(self, key: str, value):
        """Write to both cache and DB synchronously"""
        # 1. Write to database first
        self.db.set(key, value)

        # 2. Update cache
        self.cache.set(key, value)

        # Cache always consistent with DB ✅

# Usage
cache = TTLCache(ttl_seconds=3600)
db = Database()
wt_cache = WriteThroughCache(cache, db)

# Write
wt_cache.set("user:123", {"name": "Alice", "age": 26})
# → Database updated
# → Cache updated
# → Always consistent ✅

# Read
user = wt_cache.get("user:123")  # From cache (fast)
```

**Pros:**
- ✅ Strong consistency (cache always up-to-date)
- ✅ Simple to reason about

**Cons:**
- ❌ Slower writes (must update both cache and DB)
- ❌ Extra load on cache for every write

**When to use:**
- Strong consistency required
- Read-heavy workloads
- Can tolerate slower writes

---

### 3. Write-Behind (Write-Back) Cache

**Update cache immediately, database asynchronously**

```python
import asyncio
from queue import Queue

class WriteBehindCache:
    """
    Update cache immediately, queue DB writes
    """

    def __init__(self, cache, database):
        self.cache = cache
        self.db = database
        self.write_queue = Queue()
        self.is_running = True

    async def start_background_writer(self):
        """Background task to flush writes to DB"""
        while self.is_running:
            if not self.write_queue.empty():
                key, value = self.write_queue.get()
                await self.db.set(key, value)
            await asyncio.sleep(0.1)  # Check every 100ms

    def get(self, key: str):
        """Read from cache (or DB if miss)"""
        value = self.cache.get(key)
        if value is None:
            value = self.db.get(key)
            if value is not None:
                self.cache.set(key, value)
        return value

    def set(self, key: str, value):
        """
        Write to cache immediately, queue DB write
        """
        # 1. Update cache (fast)
        self.cache.set(key, value)

        # 2. Queue database write (async)
        self.write_queue.put((key, value))

        # Returns immediately ✅ (DB write happens later)

# Usage
cache = TTLCache(ttl_seconds=3600)
db = Database()
wb_cache = WriteBehindCache(cache, db)

# Start background writer
asyncio.create_task(wb_cache.start_background_writer())

# Write
wb_cache.set("user:123", {"name": "Alice", "age": 26})
# → Cache updated immediately ✅
# → DB write queued (written within 100ms)

# Read
user = wb_cache.get("user:123")  # From cache (fast + consistent)
```

**Pros:**
- ✅ Fast writes (cache updated immediately)
- ✅ Reduced DB load (batched writes)

**Cons:**
- ❌ Risk of data loss (if cache crashes before DB write)
- ❌ Complexity (background writer, queue management)

**When to use:**
- Write-heavy workloads
- Can tolerate small data loss risk
- Performance critical

---

### 4. Cache-Aside (Lazy Loading)

**Application manages cache manually**

```python
class CacheAside:
    """
    Cache-aside pattern: app controls caching
    """

    def __init__(self, cache, database):
        self.cache = cache
        self.db = database

    def get(self, key: str):
        """
        Read pattern:
        1. Try cache
        2. If miss, read DB and populate cache
        """
        # Try cache
        value = self.cache.get(key)
        if value is not None:
            return value  # Cache hit

        # Cache miss: read from DB
        value = self.db.get(key)
        if value is not None:
            self.cache.set(key, value)  # Populate cache

        return value

    def set(self, key: str, value):
        """
        Write pattern:
        1. Write to DB
        2. Invalidate cache (remove stale entry)
        """
        # Update database
        self.db.set(key, value)

        # Invalidate cache
        self.cache.delete(key)

        # Next read will fetch from DB and populate cache ✅

# Usage
cache = TTLCache(ttl_seconds=3600)
db = Database()
ca_cache = CacheAside(cache, db)

# First read (cache miss)
user = ca_cache.get("user:123")  # Reads from DB, caches

# Second read (cache hit)
user = ca_cache.get("user:123")  # From cache ✅

# Write
ca_cache.set("user:123", {"name": "Alice", "age": 27})
# → DB updated
# → Cache invalidated
# → Next read will get fresh data ✅
```

**Pros:**
- ✅ Simple
- ✅ Application has full control
- ✅ Only caches requested data (lazy loading)

**Cons:**
- ❌ Cache miss penalty (must fetch from DB)
- ❌ Application responsible for consistency

**When to use:**
- Most common pattern
- Read-heavy workloads
- Don't know what data to cache upfront

---

## Advanced Invalidation Techniques

### 5. Event-Driven Invalidation

**Invalidate cache based on database events**

```python
import asyncio

class EventDrivenCache:
    """
    Listen to database changes and invalidate cache
    """

    def __init__(self, cache):
        self.cache = cache

    async def listen_to_db_changes(self):
        """
        Listen to database change events (e.g., PostgreSQL NOTIFY)
        """
        # Pseudo-code (real implementation uses DB-specific APIs)
        async for event in database.listen_for_changes():
            if event.type == "UPDATE" or event.type == "DELETE":
                # Invalidate cached entry
                self.cache.delete(event.key)
                print(f"Invalidated cache for {event.key}")

# PostgreSQL example (using LISTEN/NOTIFY)
"""
-- In database:
CREATE OR REPLACE FUNCTION notify_cache_invalidation()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('cache_invalidation', NEW.user_id::text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER user_update_trigger
AFTER UPDATE ON users
FOR EACH ROW
EXECUTE FUNCTION notify_cache_invalidation();
"""

# Python listener
import asyncpg

async def cache_invalidation_listener():
    conn = await asyncpg.connect("postgresql://...")

    await conn.add_listener('cache_invalidation', lambda conn, pid, channel, payload:
        cache.delete(f"user:{payload}")
    )

    # Keep listening
    await asyncio.Future()
```

**Pros:**
- ✅ Real-time invalidation
- ✅ No stale data
- ✅ Automatic (no manual invalidation)

**Cons:**
- ❌ Complex setup
- ❌ Database-specific
- ❌ Requires reliable event delivery

---

### 6. Versioning

**Use version numbers to invalidate**

```python
class VersionedCache:
    """
    Track version numbers to invalidate stale data
    """

    def __init__(self, cache, database):
        self.cache = cache
        self.db = database
        self.version_tracker = {}  # key → version

    def get(self, key: str):
        cached = self.cache.get(key)
        if cached:
            cached_value, cached_version = cached

            # Check version
            current_version = self.db.get_version(key)
            if cached_version == current_version:
                return cached_value  # Version matches ✅
            else:
                # Version mismatch: refetch
                self.cache.delete(key)

        # Fetch from DB with version
        value = self.db.get(key)
        version = self.db.get_version(key)

        self.cache.set(key, (value, version))
        return value

    def set(self, key: str, value):
        # Increment version
        self.db.set(key, value)
        new_version = self.db.increment_version(key)

        # Update cache with new version
        self.cache.set(key, (value, new_version))

# Database schema:
"""
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    name VARCHAR(100),
    age INT,
    version INT DEFAULT 1  ← Version column
);

-- On update, increment version:
UPDATE users
SET name = 'Alice', age = 26, version = version + 1
WHERE user_id = 123;
"""
```

**Pros:**
- ✅ Precise invalidation
- ✅ No unnecessary cache misses

**Cons:**
- ❌ Extra storage (version column)
- ❌ More complex queries

---

## Comparison Table

| Strategy | Consistency | Write Latency | Complexity | Data Loss Risk |
|----------|-------------|---------------|------------|----------------|
| **TTL** | Eventual | Low | Low | None |
| **Write-Through** | Strong | High | Low | None |
| **Write-Behind** | Eventual | Low | Medium | Yes (on crash) |
| **Cache-Aside** | Eventual | Medium | Low | None |
| **Event-Driven** | Strong | Low | High | None |
| **Versioning** | Strong | Medium | Medium | None |

---

## Common Patterns

### Pattern 1: TTL + Cache-Aside (Most Common)

```python
# Combine TTL for automatic cleanup + manual invalidation
cache = TTLCache(ttl_seconds=3600)  # 1 hour TTL

def get_user(user_id):
    # Try cache
    user = cache.get(f"user:{user_id}")
    if user:
        return user

    # Cache miss: fetch from DB
    user = db.get_user(user_id)
    cache.set(f"user:{user_id}", user)
    return user

def update_user(user_id, data):
    # Update DB
    db.update_user(user_id, data)

    # Invalidate cache
    cache.delete(f"user:{user_id}")

# Benefits:
# - TTL prevents cache growing forever
# - Manual invalidation ensures freshness on updates
```

### Pattern 2: Multi-Level Caching

```python
# L1: Application cache (fast, small)
# L2: Redis cache (medium, large)
# L3: Database (slow, source of truth)

def get_user(user_id):
    # Try L1 (in-memory)
    user = app_cache.get(user_id)
    if user:
        return user

    # Try L2 (Redis)
    user = redis_cache.get(user_id)
    if user:
        app_cache.set(user_id, user)  # Populate L1
        return user

    # L2 miss: fetch from DB (L3)
    user = db.get_user(user_id)
    redis_cache.set(user_id, user)  # Populate L2
    app_cache.set(user_id, user)     # Populate L1
    return user

def update_user(user_id, data):
    db.update_user(user_id, data)

    # Invalidate all levels
    app_cache.delete(user_id)
    redis_cache.delete(user_id)
```

---

## Best Practices

### 1. Use Appropriate TTLs

```python
# Short TTL for frequently changing data
cache.set("stock_price", price, ttl=10)  # 10 seconds

# Long TTL for static data
cache.set("country_list", countries, ttl=86400)  # 1 day

# No TTL for immutable data
cache.set("historical_data", data, ttl=None)  # Never expires
```

### 2. Cache Tags for Bulk Invalidation

```python
# Tag related cache entries
cache.set("product:123", data, tags=["products", "category:5"])
cache.set("product:456", data, tags=["products", "category:5"])

# Invalidate all products in category 5
cache.invalidate_by_tag("category:5")
```

### 3. Graceful Degradation

```python
def get_user(user_id):
    try:
        # Try cache
        return cache.get(user_id)
    except CacheUnavailable:
        # Cache down: fallback to DB
        return db.get_user(user_id)  # Slower but works ✅
```

---

## Summary

**Cache Invalidation** strategies:

1. **TTL** - Time-based expiry (simple, eventual consistency)
2. **Write-Through** - Update cache on write (strong consistency, slower writes)
3. **Write-Behind** - Async DB writes (fast writes, data loss risk)
4. **Cache-Aside** - Manual invalidation (most flexible)
5. **Event-Driven** - Real-time invalidation (complex, precise)
6. **Versioning** - Version tracking (no false invalidations)

**Most common:** TTL + Cache-Aside

**Key Takeaway:** Choose strategy based on:
- Consistency requirements
- Write/read patterns
- Complexity tolerance
- Acceptable staleness

**Next:** Learn about [CDN & Edge Caching](cdn-edge-caching.md) for global distribution.
