# LRU Cache (Least Recently Used)

## Overview

**LRU Cache** is a data structure that stores a limited number of items and automatically evicts the **least recently used** item when the cache is full. It's one of the most common interview questions and a fundamental building block in system design.

**Key Question:** "How do you implement a cache with O(1) access and O(1) eviction?"

**Short Answer:**
- **Hash map** for O(1) access by key
- **Doubly linked list** for O(1) eviction (track access order)
- Move accessed items to front (most recent)
- Evict from tail (least recent)

---

## The Problem

### Cache Basics

A cache stores frequently accessed data for fast retrieval:

```python
# Without cache: Query database every time (slow)
user = database.query("SELECT * FROM users WHERE id = 123")  # 50ms

# With cache: Check cache first (fast)
user = cache.get(123)  # 0.1ms if in cache
if user is None:
    user = database.query("SELECT * FROM users WHERE id = 123")
    cache.set(123, user)
```

### Why LRU?

**Problem:** Limited cache size (memory constraints)

**Cache Eviction Policies:**

| Policy | Description | Pros | Cons |
|--------|-------------|------|------|
| **FIFO** | Evict oldest inserted item | Simple | Doesn't consider usage |
| **LFU** | Evict least frequently used | Good for repeated access | Complex, doesn't adapt quickly |
| **Random** | Evict random item | Very simple | Unpredictable, may evict hot data |
| **LRU** | Evict least recently used | Balances recency & frequency | Need to track access order |

**LRU Assumption:** Recently accessed items likely to be accessed again soon (temporal locality)

**Example:**

```
Cache (capacity = 3):

Access pattern: A, B, C, A, D
                          ↑ Cache full, must evict

LRU order: [A, C, B]  ← B is least recently used
           (most recent) → (least recent)

Evict B, insert D: [D, A, C]
```

---

## LRU Cache Requirements

### Operations

1. **get(key)** - Get value if exists, mark as recently used
2. **put(key, value)** - Insert/update, mark as recently used, evict LRU if full

### Constraints

- **O(1) time complexity** for both get and put
- **Fixed capacity** - Evict when full
- **Track access order** - Most recent to least recent

---

## Implementation

### Approach: Hash Map + Doubly Linked List

**Data Structures:**

1. **Hash Map** - O(1) access: `key → Node`
2. **Doubly Linked List** - O(1) move to front, O(1) remove from tail

**Why Doubly Linked List?**

```
Singly Linked List:
  head → A → B → C → tail
  To remove C (tail), need to traverse from head ❌ O(n)

Doubly Linked List:
  head ⇄ A ⇄ B ⇄ C ⇄ tail
  Can remove C directly via tail pointer ✅ O(1)
  Can move node to head in O(1) ✅
```

**Structure:**

```
Hash Map:
┌─────┬──────┐
│ key │ node │
├─────┼──────┤
│  1  │  →A  │
│  2  │  →B  │
│  3  │  →C  │
└─────┴──────┘

Doubly Linked List (access order):
  dummy_head ⇄ A ⇄ B ⇄ C ⇄ dummy_tail
  (most recent) → → → (least recent)
```

### Complete Implementation

```python
from typing import Optional

class Node:
    """Doubly linked list node"""

    def __init__(self, key: int = 0, value: int = 0):
        self.key = key
        self.value = value
        self.prev: Optional[Node] = None
        self.next: Optional[Node] = None

class LRUCache:
    """
    LRU Cache with O(1) get and put operations

    Uses hash map + doubly linked list for efficient operations
    """

    def __init__(self, capacity: int):
        """
        Initialize LRU cache with fixed capacity

        Args:
            capacity: Maximum number of items in cache
        """
        self.capacity = capacity
        self.cache = {}  # key → Node

        # Dummy head and tail for easier list manipulation
        self.head = Node()  # dummy head (most recent)
        self.tail = Node()  # dummy tail (least recent)
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove_node(self, node: Node):
        """
        Remove node from linked list

        Time: O(1)
        """
        prev_node = node.prev
        next_node = node.next
        prev_node.next = next_node
        next_node.prev = prev_node

    def _add_to_head(self, node: Node):
        """
        Add node right after dummy head (most recent position)

        Time: O(1)
        """
        node.prev = self.head
        node.next = self.head.next
        self.head.next.prev = node
        self.head.next = node

    def _move_to_head(self, node: Node):
        """
        Move existing node to head (mark as recently used)

        Time: O(1)
        """
        self._remove_node(node)
        self._add_to_head(node)

    def _pop_tail(self) -> Node:
        """
        Remove and return least recently used node (before dummy tail)

        Time: O(1)
        """
        lru_node = self.tail.prev
        self._remove_node(lru_node)
        return lru_node

    def get(self, key: int) -> int:
        """
        Get value by key

        If key exists, mark as recently used and return value
        If not exists, return -1

        Time: O(1)
        Space: O(1)
        """
        if key not in self.cache:
            return -1

        # Move to head (mark as recently used)
        node = self.cache[key]
        self._move_to_head(node)

        return node.value

    def put(self, key: int, value: int):
        """
        Insert or update key-value pair

        If key exists, update value and mark as recently used
        If not exists:
          - Add to cache
          - If cache full, evict LRU item first

        Time: O(1)
        Space: O(1)
        """
        if key in self.cache:
            # Update existing key
            node = self.cache[key]
            node.value = value
            self._move_to_head(node)
        else:
            # Insert new key
            new_node = Node(key, value)
            self.cache[key] = new_node
            self._add_to_head(new_node)

            # Check capacity
            if len(self.cache) > self.capacity:
                # Evict LRU item
                lru_node = self._pop_tail()
                del self.cache[lru_node.key]

    def __str__(self) -> str:
        """String representation showing access order"""
        items = []
        current = self.head.next
        while current != self.tail:
            items.append(f"{current.key}:{current.value}")
            current = current.next
        return f"LRUCache({self.capacity}) [MRU → LRU]: {' → '.join(items)}"

# Example Usage
if __name__ == "__main__":
    cache = LRUCache(capacity=3)

    # Insert items
    cache.put(1, 100)
    print(cache)  # [1:100]

    cache.put(2, 200)
    print(cache)  # [2:200 → 1:100]

    cache.put(3, 300)
    print(cache)  # [3:300 → 2:200 → 1:100]

    # Access item (moves to front)
    print(f"get(1) = {cache.get(1)}")  # 100
    print(cache)  # [1:100 → 3:300 → 2:200]

    # Insert 4th item (evicts LRU = 2)
    cache.put(4, 400)
    print(cache)  # [4:400 → 1:100 → 3:300]

    print(f"get(2) = {cache.get(2)}")  # -1 (evicted)

    # Update existing key
    cache.put(1, 999)
    print(cache)  # [1:999 → 4:400 → 3:300]

# Output:
# LRUCache(3) [MRU → LRU]: 1:100
# LRUCache(3) [MRU → LRU]: 2:200 → 1:100
# LRUCache(3) [MRU → LRU]: 3:300 → 2:200 → 1:100
# get(1) = 100
# LRUCache(3) [MRU → LRU]: 1:100 → 3:300 → 2:200
# LRUCache(3) [MRU → LRU]: 4:400 → 1:100 → 3:300
# get(2) = -1
# LRUCache(3) [MRU → LRU]: 1:999 → 4:400 → 3:300
```

---

## Complexity Analysis

| Operation | Time | Space | Explanation |
|-----------|------|-------|-------------|
| **get(key)** | O(1) | O(1) | Hash map lookup + move to head |
| **put(key, value)** | O(1) | O(1) | Hash map insert + add to head + possible eviction |
| **Overall** | O(1) | O(capacity) | Store at most `capacity` items |

**Why O(1)?**

- **Hash map:** O(1) lookup, insert, delete
- **Doubly linked list:** O(1) remove, add to head, pop tail
- **No iteration needed!**

---

## Variations

### 1. LRU Cache with TTL (Time-To-Live)

```python
import time

class LRUCacheWithTTL(LRUCache):
    """LRU Cache with expiration time"""

    def __init__(self, capacity: int, ttl_seconds: int):
        super().__init__(capacity)
        self.ttl = ttl_seconds
        self.timestamps = {}  # key → insert_time

    def put(self, key: int, value: int):
        super().put(key, value)
        self.timestamps[key] = time.time()

    def get(self, key: int) -> int:
        if key in self.timestamps:
            # Check expiration
            if time.time() - self.timestamps[key] > self.ttl:
                # Expired, remove
                del self.cache[key]
                del self.timestamps[key]
                return -1

        return super().get(key)

# Usage
cache = LRUCacheWithTTL(capacity=100, ttl_seconds=300)  # 5 min TTL
cache.put(1, 100)
# After 5 minutes:
cache.get(1)  # -1 (expired)
```

---

### 2. Thread-Safe LRU Cache

```python
import threading

class ThreadSafeLRUCache(LRUCache):
    """Thread-safe LRU cache using locks"""

    def __init__(self, capacity: int):
        super().__init__(capacity)
        self.lock = threading.Lock()

    def get(self, key: int) -> int:
        with self.lock:
            return super().get(key)

    def put(self, key: int, value: int):
        with self.lock:
            super().put(key, value)

# Usage in multi-threaded environment
cache = ThreadSafeLRUCache(capacity=1000)

def worker(cache, key, value):
    cache.put(key, value)
    result = cache.get(key)

threads = [
    threading.Thread(target=worker, args=(cache, i, i*100))
    for i in range(100)
]

for t in threads:
    t.start()
for t in threads:
    t.join()
```

---

### 3. LRU Cache with Stats

```python
class LRUCacheWithStats(LRUCache):
    """LRU cache with hit/miss tracking"""

    def __init__(self, capacity: int):
        super().__init__(capacity)
        self.hits = 0
        self.misses = 0

    def get(self, key: int) -> int:
        result = super().get(key)
        if result == -1:
            self.misses += 1
        else:
            self.hits += 1
        return result

    def hit_rate(self) -> float:
        """Calculate cache hit rate"""
        total = self.hits + self.misses
        return self.hits / total if total > 0 else 0.0

    def stats(self) -> dict:
        """Get cache statistics"""
        return {
            "hits": self.hits,
            "misses": self.misses,
            "hit_rate": f"{self.hit_rate()*100:.2f}%",
            "size": len(self.cache),
            "capacity": self.capacity
        }

# Usage
cache = LRUCacheWithStats(capacity=100)

for i in range(200):
    cache.put(i, i*10)

for i in range(50):
    cache.get(i)  # First 50 evicted (miss)

for i in range(100, 150):
    cache.get(i)  # Last 100 still in cache (hit)

print(cache.stats())
# {
#   "hits": 50,
#   "misses": 50,
#   "hit_rate": "50.00%",
#   "size": 100,
#   "capacity": 100
# }
```

---

## Real-World Applications

### 1. Web Browser Cache

```python
class BrowserCache:
    """
    Browser caches recently visited pages

    LRU ensures frequently visited pages stay cached
    """

    def __init__(self, max_pages: int = 50):
        self.cache = LRUCache(max_pages)

    def visit_page(self, url: str, html_content: str):
        """Cache page content"""
        url_hash = hash(url)
        self.cache.put(url_hash, html_content)
        print(f"Cached: {url}")

    def get_cached_page(self, url: str) -> Optional[str]:
        """Get page from cache (fast)"""
        url_hash = hash(url)
        content = self.cache.get(url_hash)

        if content != -1:
            print(f"Cache HIT: {url}")
            return content
        else:
            print(f"Cache MISS: {url} (fetch from server)")
            return None

# Usage
browser = BrowserCache(max_pages=3)

browser.visit_page("google.com", "<html>Google</html>")
browser.visit_page("github.com", "<html>GitHub</html>")
browser.visit_page("stackoverflow.com", "<html>SO</html>")

# Revisit (cache hit)
browser.get_cached_page("google.com")  # HIT

# Visit new page (evicts stackoverflow.com)
browser.visit_page("reddit.com", "<html>Reddit</html>")

browser.get_cached_page("stackoverflow.com")  # MISS (evicted)
```

---

### 2. Database Query Cache

```python
class DatabaseQueryCache:
    """
    Cache database query results

    Avoids expensive repeated queries
    """

    def __init__(self, cache_size: int = 1000):
        self.cache = LRUCache(cache_size)

    def query(self, sql: str, params: tuple = ()):
        """Execute query with caching"""
        # Create cache key from query + params
        cache_key = hash((sql, params))

        # Check cache first
        result = self.cache.get(cache_key)
        if result != -1:
            print(f"Cache HIT: {sql}")
            return result

        # Cache miss: execute query
        print(f"Cache MISS: {sql} (executing...)")
        result = self._execute_query(sql, params)

        # Cache result
        self.cache.put(cache_key, result)

        return result

    def _execute_query(self, sql: str, params: tuple):
        """Simulate database query (slow)"""
        # In production: conn.execute(sql, params)
        return f"result_for_{sql}"

# Usage
db_cache = DatabaseQueryCache(cache_size=100)

# First call: cache miss
db_cache.query("SELECT * FROM users WHERE id = ?", (123,))

# Second call: cache hit (fast!)
db_cache.query("SELECT * FROM users WHERE id = ?", (123,))

# Output:
# Cache MISS: SELECT * FROM users WHERE id = ? (executing...)
# Cache HIT: SELECT * FROM users WHERE id = ?
```

---

### 3. Operating System Page Cache

```python
class PageCache:
    """
    OS page cache for disk blocks

    LRU eviction policy for memory management
    """

    def __init__(self, num_pages: int):
        self.cache = LRUCache(num_pages)

    def read_page(self, page_number: int):
        """Read page from cache or disk"""
        page_data = self.cache.get(page_number)

        if page_data != -1:
            # Cache hit (fast)
            print(f"Page {page_number}: Memory (0.1μs)")
            return page_data
        else:
            # Cache miss (slow disk read)
            print(f"Page {page_number}: Disk (5ms)")
            page_data = self._load_from_disk(page_number)
            self.cache.put(page_number, page_data)
            return page_data

    def _load_from_disk(self, page_number: int):
        """Simulate disk I/O (slow)"""
        return f"page_data_{page_number}"

# Usage
os_cache = PageCache(num_pages=4)

# Access pattern: 1, 2, 3, 4, 1, 5
for page in [1, 2, 3, 4, 1, 5]:
    os_cache.read_page(page)

# Output:
# Page 1: Disk (5ms)
# Page 2: Disk (5ms)
# Page 3: Disk (5ms)
# Page 4: Disk (5ms)
# Page 1: Memory (0.1μs)  ← Cache hit!
# Page 5: Disk (5ms)      ← Evicts page 2 (LRU)
```

---

## Alternative: Using OrderedDict (Python)

**Python's `collections.OrderedDict` provides similar functionality:**

```python
from collections import OrderedDict

class LRUCacheOrderedDict:
    """LRU Cache using Python's OrderedDict"""

    def __init__(self, capacity: int):
        self.cache = OrderedDict()
        self.capacity = capacity

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1

        # Move to end (mark as recently used)
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key: int, value: int):
        if key in self.cache:
            # Update and move to end
            self.cache.move_to_end(key)

        self.cache[key] = value

        # Evict LRU if over capacity
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)  # Remove first (LRU)

# Simpler implementation, but same O(1) complexity
```

**Note:** For interviews, implement from scratch (hash map + doubly linked list) to demonstrate understanding.

---

## Interview Tips

### Common Questions

**Q: "Why doubly linked list instead of singly linked list?"**

**A:** Doubly linked list allows O(1) removal from middle. With singly linked list, removing a node requires traversing from head to find previous node (O(n)).

**Q: "Can you use an array instead?"**

**A:** No. Array insertion/deletion from middle is O(n). We need O(1) removal.

**Q: "What if two items have same access time?"**

**A:** Order doesn't matter. Any FIFO order among equally recent items is fine.

**Q: "How would you make this thread-safe?"**

**A:** Add a lock (`threading.Lock()`) and wrap all operations in `with self.lock:`.

**Q: "LRU vs LFU?"**

**A:**
- **LRU (Least Recently Used):** Evicts item not accessed for longest time
- **LFU (Least Frequently Used):** Evicts item accessed fewest times
- LRU is simpler and more common

### Common Mistakes

❌ Forgetting to update access order on `get()`
❌ Not handling update (put existing key) correctly
❌ Incorrect eviction (evicting wrong item)
❌ Not using dummy head/tail (makes code complex)

✅ Always move accessed node to head
✅ Check if key exists in `put()` before inserting
✅ Use dummy nodes for cleaner code

---

## Summary

**LRU Cache** is a fundamental data structure combining:
- **Hash map** for O(1) key lookup
- **Doubly linked list** for O(1) eviction order tracking

**Operations:**
- `get(key)`: O(1) - Return value, move to front
- `put(key, value)`: O(1) - Insert/update, evict LRU if full

**Used by:**
- Web browsers (page cache)
- Databases (query result cache)
- Operating systems (page cache)
- CDNs (content cache)
- Redis (eviction policy)

**Key Concepts:**
1. Temporal locality (recently used → likely to be used again)
2. Fixed capacity (memory constraints)
3. O(1) operations (hash map + doubly linked list)

**Trade-offs:**
- More complex than simple hash map
- Extra memory for linked list pointers
- Not optimal if access pattern is random

**Next:** Learn about [Bloom Filter](../bloom-filter/README.md) for space-efficient set membership testing.
