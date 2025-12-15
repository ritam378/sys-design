# Design an LRU Cache

## Problem Statement

Design a **Least Recently Used (LRU) Cache** data structure that supports:
- `get(key)`: Return value if exists, else -1 (O(1))
- `put(key, value)`: Insert/update key-value pair (O(1))
- Evict least recently used item when capacity reached

## Requirements

1. **O(1) time** for get and put operations
2. **Fixed capacity**: Remove LRU item when full
3. **Access order tracking**: Most recently used at front
4. **Thread safety** (optional)

## Solution: DoublyLinkedList + HashMap

```
HashMap: O(1) lookup
DoublyLinkedList: O(1) add/remove

┌─────────────────────────────────────────────────────┐
│                    HashMap                          │
│  key1 → Node1 ──┐                                   │
│  key2 → Node2 ──┼───┐                               │
│  key3 → Node3 ──┼───┼───┐                           │
└─────────────────┼───┼───┼───────────────────────────┘
                  │   │   │
                  ↓   ↓   ↓
┌─────────────────────────────────────────────────────┐
│          Doubly Linked List (LRU Order)             │
│                                                     │
│  HEAD ←→ Node3 ←→ Node2 ←→ Node1 ←→ TAIL           │
│  (dummy) (MRU)              (LRU)    (dummy)        │
└─────────────────────────────────────────────────────┘
```

## Implementation

```python
class Node:
    """Doubly linked list node"""

    def __init__(self, key=0, value=0):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None

class LRUCache:
    """
    LRU Cache using HashMap + Doubly Linked List

    Time: O(1) for get and put
    Space: O(capacity)
    """

    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}  # key -> Node

        # Dummy head and tail for easier operations
        self.head = Node()
        self.tail = Node()
        self.head.next = self.tail
        self.tail.prev = self.head

    def get(self, key: int) -> int:
        """
        Get value for key

        Steps:
        1. Check if key exists
        2. Move node to front (most recently used)
        3. Return value
        """
        if key not in self.cache:
            return -1

        node = self.cache[key]

        # Move to front (most recently used)
        self._remove(node)
        self._add_to_front(node)

        return node.value

    def put(self, key: int, value: int) -> None:
        """
        Put key-value pair

        Steps:
        1. If key exists: update value and move to front
        2. If new key:
           a. Check capacity
           b. If full: evict LRU (node before tail)
           c. Add new node to front
        """
        if key in self.cache:
            # Update existing key
            node = self.cache[key]
            node.value = value
            self._remove(node)
            self._add_to_front(node)
        else:
            # Add new key
            if len(self.cache) >= self.capacity:
                # Evict LRU (node before tail)
                lru_node = self.tail.prev
                self._remove(lru_node)
                del self.cache[lru_node.key]

            # Add new node
            new_node = Node(key, value)
            self.cache[key] = new_node
            self._add_to_front(new_node)

    def _add_to_front(self, node: Node) -> None:
        """Add node right after head (most recently used)"""
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node

    def _remove(self, node: Node) -> None:
        """Remove node from list"""
        node.prev.next = node.next
        node.next.prev = node.prev

    def __str__(self):
        """Display cache state (for debugging)"""
        items = []
        current = self.head.next
        while current != self.tail:
            items.append(f"{current.key}:{current.value}")
            current = current.next
        return f"LRUCache[{' → '.join(items)}]"

# Example Usage
if __name__ == '__main__':
    cache = LRUCache(capacity=3)

    cache.put(1, 1)  # Cache: [1:1]
    cache.put(2, 2)  # Cache: [2:2, 1:1]
    cache.put(3, 3)  # Cache: [3:3, 2:2, 1:1]
    print(cache)

    print(cache.get(1))  # Returns 1, Cache: [1:1, 3:3, 2:2]
    print(cache)

    cache.put(4, 4)  # Evicts 2, Cache: [4:4, 1:1, 3:3]
    print(cache)

    print(cache.get(2))  # Returns -1 (not found)

    cache.put(5, 5)  # Evicts 3, Cache: [5:5, 4:4, 1:1]
    print(cache)
```

## Alternative: Using OrderedDict (Python)

```python
from collections import OrderedDict

class LRUCacheSimple:
    """
    LRU Cache using OrderedDict

    OrderedDict maintains insertion order
    move_to_end() moves key to end (O(1))
    popitem(last=False) removes first item (O(1))
    """

    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1

        # Move to end (most recently used)
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            # Update and move to end
            self.cache.move_to_end(key)

        self.cache[key] = value

        if len(self.cache) > self.capacity:
            # Remove first (least recently used)
            self.cache.popitem(last=False)

# Usage
cache = LRUCacheSimple(capacity=2)
cache.put(1, 1)
cache.put(2, 2)
print(cache.get(1))  # 1
cache.put(3, 3)  # Evicts 2
print(cache.get(2))  # -1
```

## Thread-Safe LRU Cache

```python
import threading

class ThreadSafeLRUCache:
    """Thread-safe LRU Cache"""

    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}
        self.head = Node()
        self.tail = Node()
        self.head.next = self.tail
        self.tail.prev = self.head
        self.lock = threading.RLock()  # Reentrant lock

    def get(self, key: int) -> int:
        with self.lock:
            if key not in self.cache:
                return -1
            node = self.cache[key]
            self._remove(node)
            self._add_to_front(node)
            return node.value

    def put(self, key: int, value: int) -> None:
        with self.lock:
            if key in self.cache:
                node = self.cache[key]
                node.value = value
                self._remove(node)
                self._add_to_front(node)
            else:
                if len(self.cache) >= self.capacity:
                    lru_node = self.tail.prev
                    self._remove(lru_node)
                    del self.cache[lru_node.key]

                new_node = Node(key, value)
                self.cache[key] = new_node
                self._add_to_front(new_node)

    def _add_to_front(self, node: Node) -> None:
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node

    def _remove(self, node: Node) -> None:
        node.prev.next = node.next
        node.next.prev = node.prev
```

## Extended Features

### 1. TTL (Time-To-Live)

```python
import time

class LRUCacheWithTTL:
    """LRU Cache with expiration"""

    class NodeWithTTL(Node):
        def __init__(self, key, value, ttl=None):
            super().__init__(key, value)
            self.expiry = time.time() + ttl if ttl else None

        def is_expired(self):
            if self.expiry is None:
                return False
            return time.time() > self.expiry

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1

        node = self.cache[key]

        # Check expiration
        if node.is_expired():
            self._remove(node)
            del self.cache[key]
            return -1

        self._remove(node)
        self._add_to_front(node)
        return node.value

    def put(self, key: int, value: int, ttl: int = None) -> None:
        # Similar to regular put, but use NodeWithTTL
        pass
```

### 2. LFU Cache (Least Frequently Used)

```python
from collections import defaultdict

class LFUCache:
    """
    Least Frequently Used Cache

    Evict item with lowest access frequency
    Tie-breaker: LRU among same frequency
    """

    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}  # key -> (value, frequency)
        self.freq_map = defaultdict(OrderedDict)  # frequency -> {key: value}
        self.min_freq = 0

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1

        value, freq = self.cache[key]

        # Remove from current frequency
        del self.freq_map[freq][key]
        if not self.freq_map[freq] and freq == self.min_freq:
            self.min_freq += 1

        # Add to next frequency
        self.cache[key] = (value, freq + 1)
        self.freq_map[freq + 1][key] = value

        return value

    def put(self, key: int, value: int) -> None:
        if self.capacity == 0:
            return

        if key in self.cache:
            # Update value
            _, freq = self.cache[key]
            del self.freq_map[freq][key]
            self.cache[key] = (value, freq + 1)
            self.freq_map[freq + 1][key] = value

            if not self.freq_map[freq] and freq == self.min_freq:
                self.min_freq += 1
        else:
            # New key
            if len(self.cache) >= self.capacity:
                # Evict LFU (first item in min_freq)
                evict_key, _ = self.freq_map[self.min_freq].popitem(last=False)
                del self.cache[evict_key]

            # Add new item
            self.cache[key] = (value, 1)
            self.freq_map[1][key] = value
            self.min_freq = 1
```

### 3. Stats and Monitoring

```python
class LRUCacheWithStats(LRUCache):
    """LRU Cache with hit/miss tracking"""

    def __init__(self, capacity: int):
        super().__init__(capacity)
        self.hits = 0
        self.misses = 0
        self.evictions = 0

    def get(self, key: int) -> int:
        result = super().get(key)
        if result == -1:
            self.misses += 1
        else:
            self.hits += 1
        return result

    def put(self, key: int, value: int) -> None:
        if key not in self.cache and len(self.cache) >= self.capacity:
            self.evictions += 1
        super().put(key, value)

    def hit_rate(self):
        total = self.hits + self.misses
        return self.hits / total if total > 0 else 0

    def stats(self):
        return {
            'hits': self.hits,
            'misses': self.misses,
            'evictions': self.evictions,
            'hit_rate': self.hit_rate(),
            'size': len(self.cache)
        }

# Usage
cache = LRUCacheWithStats(capacity=100)
# ... use cache ...
print(cache.stats())
# {'hits': 850, 'misses': 150, 'evictions': 50, 'hit_rate': 0.85, 'size': 100}
```

## Design Patterns

### 1. Decorator Pattern (Caching)

```python
from functools import wraps

def lru_cached(capacity=128):
    """Decorator to cache function results"""

    cache = LRUCache(capacity)

    def decorator(func):
        @wraps(func)
        def wrapper(*args):
            # Use args as cache key
            key = hash(args)

            result = cache.get(key)
            if result != -1:
                return result

            result = func(*args)
            cache.put(key, result)
            return result

        return wrapper
    return decorator

# Usage
@lru_cached(capacity=10)
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

print(fibonacci(100))  # Fast due to caching
```

## Interview Tips

### Common Questions

**Q: Why DoublyLinkedList instead of Array?**
- Array: O(n) to remove item from middle
- DoublyLinkedList: O(1) to add/remove

**Q: Why HashMap + LinkedList?**
- HashMap: O(1) lookup by key
- LinkedList: O(1) reorder (move to front)
- Combined: O(1) for both operations

**Q: How would you make it thread-safe?**
- Add locks (RLock) to get/put methods
- Use concurrent data structures

**Q: LRU vs LFU?**
- LRU: Evict least recently used
- LFU: Evict least frequently used
- LRU simpler, LFU better for skewed access patterns

**Q: How to handle expiration?**
- Add timestamp to Node
- Check on get() if expired
- Background thread to clean up

## Complexity Analysis

| Operation | Time | Space |
|-----------|------|-------|
| get() | O(1) | - |
| put() | O(1) | - |
| Overall | - | O(capacity) |

## Key Takeaways

1. **HashMap + DoublyLinkedList**: Classic combination for O(1) operations
2. **Dummy nodes**: Simplify edge cases (head/tail)
3. **Encapsulation**: Private helper methods (_add_to_front, _remove)
4. **OrderedDict**: Python shortcut but understand the underlying structure
5. **Thread safety**: Use locks for concurrent access

This is a classic interview problem that tests data structure knowledge and O(1) optimization!
