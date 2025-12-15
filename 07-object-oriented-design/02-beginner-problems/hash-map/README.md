# Hash Map - OOD Design

**Difficulty:** Beginner-Intermediate
**Interview Frequency:** Very High
**Key Concepts:** Hashing, Collision Resolution, Generics, Encapsulation
**Companies:** Google, Amazon, Facebook, Microsoft, Apple, LinkedIn

---

## Problem Statement

Design and implement a hash map (hash table) from scratch with support for collision resolution, dynamic resizing, and generic key-value pairs. Demonstrate understanding of hashing principles and trade-offs.

**Core Features:**
1. Put/Get/Remove operations
2. Collision resolution (chaining and open addressing)
3. Dynamic resizing when load factor exceeds threshold
4. Generic type support for keys and values
5. Iterator support
6. Handle edge cases (null keys, collisions, resizing)

---

## Implementation

```python
from typing import Generic, TypeVar, Optional, List, Tuple, Iterator
from abc import ABC, abstractmethod

K = TypeVar('K')
V = TypeVar('V')


class HashEntry(Generic[K, V]):
    """Represents a key-value pair in the hash map"""

    def __init__(self, key: K, value: V):
        self.key = key
        self.value = value
        self.next: Optional['HashEntry[K, V]'] = None  # For chaining

    def __str__(self) -> str:
        return f"{self.key}: {self.value}"


class HashMap(Generic[K, V]):
    """
    Hash Map implementation with chaining for collision resolution.
    Features:
    - Dynamic resizing when load factor > 0.75
    - Generic key-value support
    - O(1) average case for put/get/remove
    """

    DEFAULT_CAPACITY = 16
    LOAD_FACTOR_THRESHOLD = 0.75

    def __init__(self, initial_capacity: int = DEFAULT_CAPACITY):
        self._capacity = initial_capacity
        self._buckets: List[Optional[HashEntry[K, V]]] = [None] * self._capacity
        self._size = 0

    def _hash(self, key: K) -> int:
        """
        Hash function to convert key to bucket index.
        Uses Python's built-in hash() and modulo.
        """
        return hash(key) % self._capacity

    def _resize(self) -> None:
        """
        Double the capacity and rehash all entries.
        Called when load factor exceeds threshold.
        """
        old_buckets = self._buckets
        self._capacity *= 2
        self._buckets = [None] * self._capacity
        self._size = 0

        # Rehash all entries
        for bucket in old_buckets:
            entry = bucket
            while entry:
                self.put(entry.key, entry.value)
                entry = entry.next

    def put(self, key: K, value: V) -> None:
        """
        Insert or update key-value pair.
        O(1) average case, O(n) worst case with collisions.
        """
        # Check load factor and resize if necessary
        if self._size / self._capacity > self.LOAD_FACTOR_THRESHOLD:
            self._resize()

        index = self._hash(key)
        entry = self._buckets[index]

        # Check if key already exists (update value)
        current = entry
        while current:
            if current.key == key:
                current.value = value
                return
            current = current.next

        # Add new entry at beginning of chain
        new_entry = HashEntry(key, value)
        new_entry.next = entry
        self._buckets[index] = new_entry
        self._size += 1

    def get(self, key: K) -> Optional[V]:
        """
        Retrieve value by key.
        Returns None if key not found.
        O(1) average case.
        """
        index = self._hash(key)
        entry = self._buckets[index]

        while entry:
            if entry.key == key:
                return entry.value
            entry = entry.next

        return None

    def remove(self, key: K) -> bool:
        """
        Remove key-value pair by key.
        Returns True if removed, False if not found.
        O(1) average case.
        """
        index = self._hash(key)
        entry = self._buckets[index]

        if entry is None:
            return False

        # Check if first entry matches
        if entry.key == key:
            self._buckets[index] = entry.next
            self._size -= 1
            return True

        # Search in chain
        prev = entry
        current = entry.next
        while current:
            if current.key == key:
                prev.next = current.next
                self._size -= 1
                return True
            prev = current
            current = current.next

        return False

    def contains_key(self, key: K) -> bool:
        """Check if key exists"""
        return self.get(key) is not None

    def size(self) -> int:
        """Return number of key-value pairs"""
        return self._size

    def is_empty(self) -> bool:
        """Check if map is empty"""
        return self._size == 0

    def clear(self) -> None:
        """Remove all entries"""
        self._buckets = [None] * self._capacity
        self._size = 0

    def keys(self) -> List[K]:
        """Return list of all keys"""
        result = []
        for bucket in self._buckets:
            entry = bucket
            while entry:
                result.append(entry.key)
                entry = entry.next
        return result

    def values(self) -> List[V]:
        """Return list of all values"""
        result = []
        for bucket in self._buckets:
            entry = bucket
            while entry:
                result.append(entry.value)
                entry = entry.next
        return result

    def items(self) -> List[Tuple[K, V]]:
        """Return list of (key, value) tuples"""
        result = []
        for bucket in self._buckets:
            entry = bucket
            while entry:
                result.append((entry.key, entry.value))
                entry = entry.next
        return result

    def __getitem__(self, key: K) -> Optional[V]:
        """Support dict-style access: map[key]"""
        return self.get(key)

    def __setitem__(self, key: K, value: V) -> None:
        """Support dict-style assignment: map[key] = value"""
        self.put(key, value)

    def __delitem__(self, key: K) -> None:
        """Support dict-style deletion: del map[key]"""
        if not self.remove(key):
            raise KeyError(key)

    def __contains__(self, key: K) -> bool:
        """Support 'in' operator: key in map"""
        return self.contains_key(key)

    def __len__(self) -> int:
        """Support len(map)"""
        return self._size

    def __iter__(self) -> Iterator[K]:
        """Support iteration: for key in map"""
        return iter(self.keys())

    def __str__(self) -> str:
        """String representation"""
        items = [f"{k}: {v}" for k, v in self.items()]
        return "{" + ", ".join(items) + "}"

    def get_load_factor(self) -> float:
        """Get current load factor"""
        return self._size / self._capacity

    def get_stats(self) -> dict:
        """Get hash map statistics for debugging"""
        chain_lengths = []
        for bucket in self._buckets:
            length = 0
            entry = bucket
            while entry:
                length += 1
                entry = entry.next
            if length > 0:
                chain_lengths.append(length)

        return {
            "size": self._size,
            "capacity": self._capacity,
            "load_factor": self.get_load_factor(),
            "num_buckets_used": len(chain_lengths),
            "max_chain_length": max(chain_lengths) if chain_lengths else 0,
            "avg_chain_length": sum(chain_lengths) / len(chain_lengths) if chain_lengths else 0
        }


# Alternative: Open Addressing Implementation
class OpenAddressingHashMap(Generic[K, V]):
    """
    Hash Map using open addressing (linear probing).
    Alternative to chaining for collision resolution.
    """

    class _Entry(Generic[K, V]):
        def __init__(self, key: K, value: V):
            self.key = key
            self.value = value
            self.is_deleted = False

    DEFAULT_CAPACITY = 16
    LOAD_FACTOR_THRESHOLD = 0.5  # Lower threshold for open addressing

    def __init__(self, initial_capacity: int = DEFAULT_CAPACITY):
        self._capacity = initial_capacity
        self._table: List[Optional[OpenAddressingHashMap._Entry[K, V]]] = [None] * self._capacity
        self._size = 0

    def _hash(self, key: K, attempt: int = 0) -> int:
        """Linear probing: try successive slots"""
        return (hash(key) + attempt) % self._capacity

    def put(self, key: K, value: V) -> None:
        """Insert with linear probing"""
        if self._size / self._capacity > self.LOAD_FACTOR_THRESHOLD:
            self._resize()

        for attempt in range(self._capacity):
            index = self._hash(key, attempt)
            entry = self._table[index]

            if entry is None or entry.is_deleted:
                self._table[index] = self._Entry(key, value)
                self._size += 1
                return
            elif entry.key == key:
                entry.value = value  # Update
                return

        raise Exception("Hash table is full")

    def get(self, key: K) -> Optional[V]:
        """Retrieve with linear probing"""
        for attempt in range(self._capacity):
            index = self._hash(key, attempt)
            entry = self._table[index]

            if entry is None:
                return None
            elif not entry.is_deleted and entry.key == key:
                return entry.value

        return None

    def remove(self, key: K) -> bool:
        """Remove with lazy deletion"""
        for attempt in range(self._capacity):
            index = self._hash(key, attempt)
            entry = self._table[index]

            if entry is None:
                return False
            elif not entry.is_deleted and entry.key == key:
                entry.is_deleted = True
                self._size -= 1
                return True

        return False

    def _resize(self) -> None:
        """Resize and rehash"""
        old_table = self._table
        self._capacity *= 2
        self._table = [None] * self._capacity
        self._size = 0

        for entry in old_table:
            if entry and not entry.is_deleted:
                self.put(entry.key, entry.value)

    def size(self) -> int:
        return self._size


def main():
    """Demonstrate hash map implementations"""

    print("=" * 60)
    print("HASH MAP WITH CHAINING")
    print("=" * 60)

    # Create hash map
    map1 = HashMap[str, int]()

    # Put operations
    print("\n--- Inserting entries ---")
    map1["apple"] = 5
    map1["banana"] = 3
    map1["orange"] = 8
    map1.put("grape", 12)
    map1.put("mango", 7)

    print(f"Map: {map1}")
    print(f"Size: {len(map1)}")

    # Get operations
    print("\n--- Getting values ---")
    print(f"apple: {map1['apple']}")
    print(f"banana: {map1.get('banana')}")
    print(f"kiwi: {map1.get('kiwi')}")  # None

    # Contains
    print(f"\n'apple' in map: {'apple' in map1}")
    print(f"'kiwi' in map: {'kiwi' in map1}")

    # Update
    print("\n--- Updating value ---")
    map1["apple"] = 10
    print(f"Updated apple: {map1['apple']}")

    # Iteration
    print("\n--- Iterating through keys ---")
    for key in map1:
        print(f"  {key}: {map1[key]}")

    # Keys, values, items
    print(f"\nKeys: {map1.keys()}")
    print(f"Values: {map1.values()}")

    # Remove
    print("\n--- Removing entries ---")
    map1.remove("banana")
    del map1["grape"]
    print(f"After removals: {map1}")

    # Stats
    print("\n--- Statistics ---")
    stats = map1.get_stats()
    for key, value in stats.items():
        print(f"  {key}: {value}")

    # Test resizing
    print("\n--- Testing dynamic resizing ---")
    map2 = HashMap[int, str](initial_capacity=4)

    for i in range(20):
        map2[i] = f"value_{i}"
        if i % 5 == 0:
            stats = map2.get_stats()
            print(f"After {i+1} inserts: capacity={stats['capacity']}, "
                  f"load_factor={stats['load_factor']:.2f}")

    print(f"\nFinal map size: {map2.size()}")
    print(f"Final capacity: {map2.get_stats()['capacity']}")

    # Test collisions
    print("\n" + "=" * 60)
    print("TESTING COLLISION HANDLING")
    print("=" * 60)

    class BadHash:
        """Class with poor hash function to force collisions"""
        def __init__(self, value: int):
            self.value = value

        def __hash__(self) -> int:
            return self.value % 3  # Force collisions

        def __eq__(self, other) -> bool:
            return isinstance(other, BadHash) and self.value == other.value

        def __str__(self) -> str:
            return f"BadHash({self.value})"

    collision_map = HashMap[BadHash, str](initial_capacity=8)

    # Insert keys that will collide
    for i in range(10):
        key = BadHash(i)
        collision_map[key] = f"value_{i}"

    stats = collision_map.get_stats()
    print(f"\nWith forced collisions:")
    print(f"  Size: {stats['size']}")
    print(f"  Buckets used: {stats['num_buckets_used']}")
    print(f"  Max chain length: {stats['max_chain_length']}")
    print(f"  Avg chain length: {stats['avg_chain_length']:.2f}")

    # Verify all can be retrieved
    print("\nRetrieving collided entries:")
    for i in range(10):
        key = BadHash(i)
        value = collision_map[key]
        print(f"  {key}: {value}")

    print("\n" + "=" * 60)
    print("OPEN ADDRESSING HASH MAP")
    print("=" * 60)

    oa_map = OpenAddressingHashMap[str, int]()
    oa_map.put("red", 1)
    oa_map.put("green", 2)
    oa_map.put("blue", 3)

    print(f"\nSize: {oa_map.size()}")
    print(f"Get 'red': {oa_map.get('red')}")
    print(f"Get 'green': {oa_map.get('green')}")

    oa_map.remove("green")
    print(f"After removing 'green': {oa_map.get('green')}")
    print(f"Size: {oa_map.size()}")


if __name__ == "__main__":
    main()
```

---

## Design Patterns & SOLID

### Patterns Used
1. **Iterator Pattern:** `__iter__()` enables traversal
2. **Template Method:** Abstract collision resolution strategy
3. **Strategy Pattern:** Different collision resolution approaches (chaining vs open addressing)

### SOLID Principles
- **SRP:** HashEntry handles entry, HashMap handles operations
- **OCP:** Easy to add new hashing strategies
- **LSP:** Both implementations can substitute abstract hash table interface

---

## Complexity Analysis

| Operation | Average | Worst Case |
|-----------|---------|------------|
| Put | O(1) | O(n) |
| Get | O(1) | O(n) |
| Remove | O(1) | O(n) |
| Space | O(n) | O(n) |

**Worst case** occurs with many collisions (poor hash function or high load factor).

---

## Interview Tips

### What Interviewers Look For
- ✅ Understanding of hashing and collision resolution
- ✅ Dynamic resizing implementation
- ✅ Load factor management
- ✅ Generic type support
- ✅ Handle edge cases (null, collisions, resize)

### Common Mistakes
- ❌ Not handling collisions
- ❌ Poor hash function
- ❌ Not resizing (causes performance degradation)
- ❌ Memory leaks in chaining
- ❌ Incorrect rehashing logic

### Key Topics to Discuss
- Chaining vs Open Addressing trade-offs
- Load factor and performance impact
- Hash function quality
- Resize strategy (when and by how much)
- Thread safety considerations

This problem demonstrates core CS fundamentals essential for technical interviews!
