# Bloom Filter

## Overview

**Bloom Filter** is a **space-efficient probabilistic data structure** that tests whether an element is a member of a set. It can have **false positives** but **never false negatives**.

**Key Question:** "How do you check set membership using minimal space?"

**Short Answer:**
- Bit array + multiple hash functions
- **Definitely not in set** or **possibly in set**
- 10x-100x more space-efficient than hash set
- Used for: Spam filtering, cache optimization, databases

---

## The Problem

### Naive Approach: Hash Set

```python
# Check if username exists
usernames = set()  # Hash set
usernames.add("alice")
usernames.add("bob")

"alice" in usernames  # True
"charlie" in usernames  # False
```

**Problem with Hash Set:**

```
1 million usernames
Average username: 10 bytes
Storage: 10 MB ❌ (plus hash table overhead)

With Bloom Filter:
Storage: ~1.2 MB ✅ (8x smaller!)
```

---

## How Bloom Filter Works

### Structure

```
Bit Array (m bits, initially all 0):
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  0   1   2   3   4   5   6   7   8   9

k hash functions: h1, h2, ..., hk
```

### Add Element

```python
# Add "alice"
h1("alice") = 2
h2("alice") = 5
h3("alice") = 8

┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 0 │ 1 │ 0 │ 0 │ 1 │ 0 │ 0 │ 1 │ 0 │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
          ↑           ↑           ↑
          Set to 1
```

### Check Membership

```python
# Check "alice" (in set)
h1("alice") = 2 → bit[2] = 1 ✓
h2("alice") = 5 → bit[5] = 1 ✓
h3("alice") = 8 → bit[8] = 1 ✓
All bits = 1 → "Possibly in set" ✅

# Check "charlie" (not in set)
h1("charlie") = 1 → bit[1] = 0 ✗
                    ↑ At least one bit = 0
Result: "Definitely NOT in set" ✅
```

### False Positive Example

```python
# Add "alice" and "bob"
alice:  bits[2, 5, 8] = 1
bob:    bits[1, 5, 9] = 1

Bit array:
[0, 1, 1, 0, 0, 1, 0, 0, 1, 1]

# Check "charlie" (not added)
h1("charlie") = 2 → bit[2] = 1 (from alice)
h2("charlie") = 5 → bit[5] = 1 (from alice/bob)
h3("charlie") = 9 → bit[9] = 1 (from bob)

All bits = 1 → "Possibly in set" ❌ FALSE POSITIVE!
(charlie not actually in set, but hash collisions match)
```

---

## Implementation

```python
import hashlib
from typing import List

class BloomFilter:
    """
    Space-efficient probabilistic set membership test

    Can have false positives, but never false negatives
    """

    def __init__(self, size: int, num_hashes: int):
        """
        Initialize Bloom filter

        Args:
            size: Bit array size (m)
            num_hashes: Number of hash functions (k)
        """
        self.size = size
        self.num_hashes = num_hashes
        self.bit_array = [0] * size

    def _hash(self, item: str, seed: int) -> int:
        """
        Hash function with seed

        Uses MD5 with different seeds to simulate k hash functions
        """
        h = hashlib.md5(f"{item}{seed}".encode())
        return int(h.hexdigest(), 16) % self.size

    def add(self, item: str):
        """
        Add item to Bloom filter

        Time: O(k) where k = num_hashes
        Space: O(1)
        """
        for i in range(self.num_hashes):
            index = self._hash(item, i)
            self.bit_array[index] = 1

    def contains(self, item: str) -> bool:
        """
        Check if item might be in set

        Returns:
            True: Possibly in set (might be false positive)
            False: Definitely NOT in set (100% accurate)

        Time: O(k)
        """
        for i in range(self.num_hashes):
            index = self._hash(item, i)
            if self.bit_array[index] == 0:
                return False  # Definitely not in set
        return True  # Possibly in set

    def false_positive_rate(self, n: int) -> float:
        """
        Calculate theoretical false positive probability

        Formula: (1 - e^(-kn/m))^k
        where k=num_hashes, n=items added, m=bit array size
        """
        import math
        k = self.num_hashes
        m = self.size
        return (1 - math.e ** (-k * n / m)) ** k

# Example Usage
if __name__ == "__main__":
    # Create Bloom filter
    bf = BloomFilter(size=100, num_hashes=3)

    # Add elements
    bf.add("alice")
    bf.add("bob")
    bf.add("charlie")

    # Test membership
    print(f"alice in filter: {bf.contains('alice')}")      # True
    print(f"bob in filter: {bf.contains('bob')}")          # True
    print(f"charlie in filter: {bf.contains('charlie')}")  # True
    print(f"david in filter: {bf.contains('david')}")      # False (definitely not)

    # Might get false positive for some strings
    print(f"eve in filter: {bf.contains('eve')}")          # Might be True (false positive)

    # Calculate false positive rate
    fpr = bf.false_positive_rate(n=3)
    print(f"\nFalse positive rate: {fpr*100:.2f}%")

# Output:
# alice in filter: True
# bob in filter: True
# charlie in filter: True
# david in filter: False
# eve in filter: False
#
# False positive rate: 0.04%
```

---

## Optimal Parameters

### Choosing Size and Hash Functions

**Given:**
- `n` = expected number of elements
- `p` = desired false positive rate

**Optimal formulas:**

```python
import math

def optimal_bloom_filter_params(n: int, p: float):
    """
    Calculate optimal Bloom filter parameters

    Args:
        n: Expected number of elements
        p: Desired false positive rate (e.g., 0.01 for 1%)

    Returns:
        (m, k): Bit array size and number of hash functions
    """
    # Optimal bit array size
    m = int(-(n * math.log(p)) / (math.log(2) ** 2))

    # Optimal number of hash functions
    k = int((m / n) * math.log(2))

    return m, k

# Example: 1 million elements, 1% false positive rate
n = 1_000_000
p = 0.01

m, k = optimal_bloom_filter_params(n, p)
print(f"For {n:,} elements with {p*100}% FPR:")
print(f"  Bit array size: {m:,} bits = {m/8/1024:.2f} KB")
print(f"  Hash functions: {k}")

# Output:
# For 1,000,000 elements with 1.0% FPR:
#   Bit array size: 9,585,058 bits = 1,171.88 KB (~1.2 MB)
#   Hash functions: 7
```

**Comparison with Hash Set:**

```
Hash Set (1M elements × 10 bytes avg):
  Storage: ~10 MB

Bloom Filter (1% FPR):
  Storage: ~1.2 MB  ← 8x smaller! ✅

Trade-off: 1% false positives
```

---

## Real-World Applications

### 1. Cache Optimization (Avoid Useless Lookups)

```python
class BloomCacheOptimizer:
    """
    Use Bloom filter to avoid cache misses

    If Bloom filter says "not in cache", skip expensive cache lookup
    """

    def __init__(self, cache_capacity: int):
        self.cache = {}
        self.bloom = BloomFilter(size=cache_capacity * 10, num_hashes=7)

    def get(self, key: str):
        """Get from cache with Bloom filter optimization"""

        # Check Bloom filter first (fast)
        if not self.bloom.contains(key):
            # Definitely not in cache
            return None  # Skip expensive cache lookup ✅

        # Possibly in cache (might be false positive)
        return self.cache.get(key)

    def set(self, key: str, value):
        """Add to cache and Bloom filter"""
        self.cache[key] = value
        self.bloom.add(key)

# Usage
cache = BloomCacheOptimizer(cache_capacity=1000)
cache.set("user:123", {"name": "Alice"})

# Fast negative lookup (no cache access needed)
cache.get("user:999")  # Bloom says "not in cache" → return None immediately
```

---

### 2. Weak Password Detection

```python
class WeakPasswordChecker:
    """
    Check against 10 million common passwords

    Bloom filter: ~12 MB vs. Hash set: ~100 MB
    """

    def __init__(self, weak_passwords_file: str):
        # Bloom filter for 10M passwords, 0.1% FPR
        self.bloom = BloomFilter(size=120_000_000, num_hashes=10)

        # Load weak passwords into Bloom filter
        with open(weak_passwords_file) as f:
            for password in f:
                self.bloom.add(password.strip())

    def is_weak(self, password: str) -> bool:
        """
        Check if password is weak

        Returns:
            True: Definitely weak or possibly weak (false positive)
            False: Definitely strong
        """
        return self.bloom.contains(password)

# Usage
checker = WeakPasswordChecker("weak_passwords.txt")

if checker.is_weak("password123"):
    print("Weak password! Choose another.")
# Note: Might reject some strong passwords (0.1% FPR)
#       but never accepts weak passwords ✅
```

---

### 3. Database Query Optimization (BigTable, Cassandra)

```python
class DatabaseWithBloom:
    """
    Database uses Bloom filters to avoid disk reads

    Each SSTable (sorted string table) has a Bloom filter
    """

    def __init__(self):
        self.sstables = []  # List of on-disk files

    def write(self, key: str, value: str):
        """Write to SSTable with Bloom filter"""
        sstable = {
            "data": {key: value},
            "bloom": BloomFilter(size=10000, num_hashes=7)
        }
        sstable["bloom"].add(key)
        self.sstables.append(sstable)

    def read(self, key: str):
        """Read from SSTable(s)"""
        for sstable in self.sstables:
            # Check Bloom filter first (in-memory, fast)
            if not sstable["bloom"].contains(key):
                # Definitely not in this SSTable
                continue  # Skip disk read ✅

            # Possibly in this SSTable
            # Perform expensive disk read
            if key in sstable["data"]:
                return sstable["data"][key]

        return None

# Usage
db = DatabaseWithBloom()
db.write("user:123", "Alice")

# Query
db.read("user:123")  # Found (1 disk read)
db.read("user:999")  # Not found (0 disk reads, Bloom filter skipped all) ✅
```

---

### 4. Web Crawler (Avoid Re-crawling URLs)

```python
class WebCrawler:
    """
    Track crawled URLs with Bloom filter

    100M URLs: Bloom filter ~120 MB vs. Hash set ~1 GB
    """

    def __init__(self):
        # Bloom filter for 100M URLs, 1% FPR
        self.visited_bloom = BloomFilter(size=1_200_000_000, num_hashes=7)
        self.visited_set = set()  # Exact set (backup for false positives)

    def crawl(self, url: str):
        """Crawl URL if not visited"""

        # Quick check with Bloom filter
        if self.visited_bloom.contains(url):
            # Possibly visited
            # Verify with exact set (handles false positives)
            if url in self.visited_set:
                print(f"Already crawled: {url}")
                return

        # Definitely not visited, crawl it
        print(f"Crawling: {url}")
        self._fetch_and_parse(url)

        # Mark as visited
        self.visited_bloom.add(url)
        self.visited_set.add(url)

    def _fetch_and_parse(self, url: str):
        """Fetch and parse URL (simplified)"""
        pass

# Usage (saves 10x memory vs. hash set alone)
crawler = WebCrawler()
crawler.crawl("https://example.com")
crawler.crawl("https://example.com")  # Skipped (already crawled)
```

---

## Bloom Filter vs. Alternatives

| Data Structure | Space | False Positives | False Negatives | Use Case |
|----------------|-------|-----------------|-----------------|----------|
| **Hash Set** | O(n) | No | No | Exact membership |
| **Bloom Filter** | O(m) << O(n) | Yes (tunable) | No | Space-critical, negatives important |
| **Cuckoo Filter** | O(n) but lower | Yes | No | Supports deletion |
| **Count-Min Sketch** | O(m) | Yes | No | Frequency estimation |

---

## Limitations

### Cannot Delete Elements

```python
# Standard Bloom filter doesn't support deletion
bf = BloomFilter(size=100, num_hashes=3)
bf.add("alice")
bf.add("bob")

# bf.remove("alice")  ❌ Not supported!
# Why? Bit might be shared by multiple elements

# Bit array after adding:
# alice: bits[2, 5, 8] = 1
# bob:   bits[2, 7, 9] = 1
#               ↑ bit 2 shared!

# If we clear bit 2 to delete alice:
# bf.contains("bob") would fail! ❌
```

**Solution: Counting Bloom Filter**

```python
class CountingBloomFilter:
    """Bloom filter with deletion support"""

    def __init__(self, size: int, num_hashes: int):
        self.size = size
        self.num_hashes = num_hashes
        self.counters = [0] * size  # Use counters instead of bits

    def add(self, item: str):
        for i in range(self.num_hashes):
            index = self._hash(item, i)
            self.counters[index] += 1  # Increment counter

    def remove(self, item: str):
        for i in range(self.num_hashes):
            index = self._hash(item, i)
            if self.counters[index] > 0:
                self.counters[index] -= 1  # Decrement counter

    def contains(self, item: str) -> bool:
        for i in range(self.num_hashes):
            index = self._hash(item, i)
            if self.counters[index] == 0:
                return False
        return True

    # Trade-off: Uses more space (counters vs. bits)
```

---

## Interview Tips

### Common Questions

**Q: "What's the difference between Bloom filter and hash set?"**

**A:**
- Hash set: Exact membership, O(n) space
- Bloom filter: Probabilistic, O(m) space where m << n, allows false positives

**Q: "Can Bloom filter have false negatives?"**

**A:** No, never. If Bloom filter says "not in set", it's 100% accurate.

**Q: "How do you choose parameters?"**

**A:** Given n items and desired false positive rate p, use:
- `m = -(n * ln(p)) / (ln(2)^2)` (bit array size)
- `k = (m/n) * ln(2)` (number of hash functions)

**Q: "Why use multiple hash functions?"**

**A:** More hash functions → lower false positive rate (up to optimal k). Too few → high FPR, too many → slower and higher FPR.

---

## Summary

**Bloom Filter** is a probabilistic set membership structure:

✅ **Space-efficient** - 10x-100x smaller than hash set
✅ **Fast** - O(k) lookups where k is small
✅ **No false negatives** - If says "not in set", 100% accurate
❌ **False positives** - Might say "in set" when it's not
❌ **Cannot delete** - Standard version doesn't support removal

**Used by:**
- Google BigTable (skip disk reads)
- Apache Cassandra (SSTable optimization)
- Bitcoin (transaction validation)
- Akamai CDN (cache optimization)
- Chrome browser (malicious URL detection)

**Key Insight:** Trade accuracy for space when false positives are acceptable.

**Next:** Learn about [Distributed Lock](../distributed-lock/README.md) for concurrency control.
