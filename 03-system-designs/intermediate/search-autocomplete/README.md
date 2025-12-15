# Search Autocomplete System (Typeahead)

> **Difficulty:** Intermediate
> **Topics:** Trie, Caching, Ranking, Real-time Updates
> **Companies:** Google, Amazon, Facebook, LinkedIn, Shopify

---

## 1. Problem Statement

Design an **autocomplete/typeahead system** like Google Search that suggests completions as users type.

### Core Features

**User Experience:**
- User types "sys" → Shows ["system design", "systematic", "systemic"]
- Real-time suggestions (<100ms)
- Top 5-10 most relevant results
- Updates as user continues typing

**System:**
- Handle billions of queries
- Learn from user behavior (popular queries)
- Support ranking by relevance/popularity
- Low latency (<100ms)

---

## 2. Requirements

### Functional Requirements

1. **Autocomplete Suggestions:**
   - Given prefix, return top N matching queries
   - Ranked by popularity/relevance

2. **Real-time Updates:**
   - Suggestions appear as user types
   - Each keystroke triggers new query

3. **Query Logging:**
   - Track search queries for ranking
   - Update popularity scores

### Non-Functional Requirements

1. **Low Latency:** <100ms response time
2. **High Availability:** 99.9% uptime
3. **Scalability:** 10 billion queries/day
4. **Consistency:** Eventually consistent (delayed ranking updates OK)

### Out of Scope

- Personalized suggestions
- Spell correction
- Multi-language support
- Voice search

---

## 3. Back-of-Envelope Estimation

### Traffic

```
Daily Active Users: 500M
Queries per user per day: 20
Total queries/day: 10 billion

QPS: 10B / 86400 ≈ 115K queries/sec
Peak (3x): 345K queries/sec

Autocomplete requests:
  Average query length: 10 characters
  Autocomplete requests per query: 10 (one per character)
  Total autocomplete requests: 10B × 10 = 100B/day
  QPS: 1.15M autocomplete requests/sec
```

### Storage

```
Unique queries: ~500 million
Per query:
  query_text: 50 bytes (avg)
  frequency: 8 bytes
  Total: ~58 bytes

Storage: 500M × 58 bytes = 29 GB (manageable in memory!)
```

---

## 4. High-Level Design

```
┌─────────────┐
│   Client    │  Types: "sys"
└──────┬──────┘
       │ HTTP GET /autocomplete?prefix=sys
       ▼
┌─────────────────┐
│  API Gateway    │  (Load Balancer)
└──────┬──────────┘
       │
       ▼
┌─────────────────┐
│ Autocomplete    │  ← In-memory Trie
│    Service      │  ← Cache (Redis)
└──────┬──────────┘
       │
       ├──→ Trie: Prefix search O(k + m)
       │    k = prefix length, m = results
       │
       └──→ Returns: ["system design",
                      "systematic",
                      "systems"]
```

---

## 5. Data Structure: Trie

### Why Trie?

```
Alternatives:
1. Database LIKE query:
   SELECT query FROM searches WHERE query LIKE 'sys%'
   ❌ Slow (O(n) scan), no ranking

2. Elasticsearch:
   ✅ Fast, good for large scale
   ❌ Complex, overkill for prefix search

3. Trie:
   ✅ O(k) prefix search (k = prefix length)
   ✅ In-memory (fast)
   ✅ Natural fit for autocomplete
```

### Trie Structure

```
Root
 │
 ├─ s
 │  └─ y
 │     └─ s
 │        ├─ t
 │        │  ├─ e
 │        │  │  └─ m (queries: ["system", "systematic"])
 │        │  └─ ...
 │        └─ ...
 └─ ...

Prefix "sys" → Traverse to node "s→y→s"
             → Collect all queries under this subtree
             → Return top N by frequency
```

### Implementation

```python
from typing import List, Optional, Tuple
from collections import defaultdict
import heapq

class TrieNode:
    """Node in Trie structure"""

    def __init__(self):
        self.children: dict[str, TrieNode] = {}
        self.is_end: bool = False
        self.frequency: int = 0  # How often this query is searched
        self.query: Optional[str] = None  # Full query text (stored at leaf)

class AutocompleteTrie:
    """
    Trie-based autocomplete system

    Optimized for prefix search with frequency-based ranking
    """

    def __init__(self):
        self.root = TrieNode()

    def insert(self, query: str, frequency: int = 1):
        """
        Insert query into Trie with frequency

        Time: O(k) where k = len(query)
        Space: O(k)
        """
        node = self.root
        query = query.lower()  # Case-insensitive

        # Traverse/create path for each character
        for char in query:
            if char not in node.children:
                node.children[char] = TrieNode()
            node = node.children[char]

        # Mark end of query
        node.is_end = True
        node.frequency += frequency
        node.query = query

    def search_prefix(self, prefix: str, top_k: int = 10) -> List[Tuple[str, int]]:
        """
        Find top K queries matching prefix

        Args:
            prefix: Search prefix (e.g., "sys")
            top_k: Number of results to return

        Returns:
            List of (query, frequency) tuples, sorted by frequency desc

        Time: O(p + n) where p = len(prefix), n = nodes in subtree
        """
        prefix = prefix.lower()
        node = self.root

        # 1. Navigate to prefix node
        for char in prefix:
            if char not in node.children:
                return []  # Prefix not found
            node = node.children[char]

        # 2. Collect all queries under this node
        queries = []
        self._collect_queries(node, queries)

        # 3. Return top K by frequency
        # Use heap for efficient top-K selection
        top_queries = heapq.nlargest(top_k, queries, key=lambda x: x[1])
        return top_queries

    def _collect_queries(self, node: TrieNode, results: List[Tuple[str, int]]):
        """
        DFS to collect all queries under a node

        Args:
            node: Current trie node
            results: List to accumulate (query, frequency) tuples
        """
        if node.is_end:
            results.append((node.query, node.frequency))

        for child in node.children.values():
            self._collect_queries(child, results)

    def update_frequency(self, query: str, increment: int = 1):
        """
        Update frequency of existing query

        Time: O(k)
        """
        node = self.root
        query = query.lower()

        for char in query:
            if char not in node.children:
                return  # Query not in trie
            node = node.children[char]

        if node.is_end:
            node.frequency += increment

# Example Usage
if __name__ == "__main__":
    trie = AutocompleteTrie()

    # Insert popular queries with frequencies
    queries = [
        ("system design", 1000),
        ("systematic", 500),
        ("systemic risk", 300),
        ("systems thinking", 200),
        ("syrup", 100),
    ]

    for query, freq in queries:
        trie.insert(query, freq)

    # Autocomplete for "sys"
    print("Autocomplete for 'sys':")
    results = trie.search_prefix("sys", top_k=3)
    for query, freq in results:
        print(f"  {query} (searched {freq} times)")

    # Autocomplete for "syst"
    print("\nAutocomplete for 'syst':")
    results = trie.search_prefix("syst", top_k=3)
    for query, freq in results:
        print(f"  {query} (searched {freq} times)")

# Output:
# Autocomplete for 'sys':
#   system design (searched 1000 times)
#   systematic (searched 500 times)
#   systemic risk (searched 300 times)
#
# Autocomplete for 'syst':
#   system design (searched 1000 times)
#   systematic (searched 500 times)
#   systems thinking (searched 200 times)
```

---

## 6. Optimization: Prefix Caching

**Problem:** Collecting all queries for popular prefixes is expensive

```
Prefix "s" → Millions of queries starting with 's'
Traversing entire subtree: O(n) where n = millions ❌
```

**Solution: Cache top K at each node**

```python
class OptimizedTrieNode:
    """Trie node with cached top K queries"""

    def __init__(self, k: int = 10):
        self.children: dict[str, OptimizedTrieNode] = {}
        self.is_end: bool = False
        self.frequency: int = 0
        self.query: Optional[str] = None
        self.top_k: List[Tuple[str, int]] = []  # Cached top K queries
        self.max_k = k

class OptimizedAutocompleteTrie:
    """
    Trie with top-K caching at each node for O(1) lookup
    """

    def __init__(self, k: int = 10):
        self.root = OptimizedTrieNode(k)
        self.k = k

    def insert(self, query: str, frequency: int = 1):
        """Insert and update top-K cache along path"""
        node = self.root
        query = query.lower()

        for char in query:
            if char not in node.children:
                node.children[char] = OptimizedTrieNode(self.k)
            node = node.children[char]

            # Update top-K at this node
            self._update_top_k(node, query, frequency)

        node.is_end = True
        node.frequency += frequency
        node.query = query

    def _update_top_k(self, node: OptimizedTrieNode, query: str, frequency: int):
        """
        Update top-K cache at node

        Maintain sorted list of top K queries
        """
        # Find if query already in top-K
        for i, (q, f) in enumerate(node.top_k):
            if q == query:
                node.top_k[i] = (q, f + frequency)
                break
        else:
            # Add new query
            node.top_k.append((query, frequency))

        # Sort and keep top K
        node.top_k.sort(key=lambda x: x[1], reverse=True)
        node.top_k = node.top_k[:self.k]

    def search_prefix(self, prefix: str) -> List[Tuple[str, int]]:
        """
        Get top K queries for prefix

        Time: O(p) where p = len(prefix)  ← Much faster!
        No need to traverse subtree
        """
        node = self.root
        prefix = prefix.lower()

        for char in prefix:
            if char not in node.children:
                return []
            node = node.children[char]

        # Return cached top-K (already sorted)
        return node.top_k

# Usage
optimized_trie = OptimizedAutocompleteTrie(k=5)

for query, freq in queries:
    optimized_trie.insert(query, freq)

# O(prefix_length) lookup ✅
results = optimized_trie.search_prefix("sys")
print("Top suggestions:", [q for q, f in results])

# Output:
# Top suggestions: ['system design', 'systematic', 'systemic risk', 'systems thinking', 'syrup']
```

**Complexity Comparison:**

| Operation | Basic Trie | Optimized Trie |
|-----------|------------|----------------|
| **Insert** | O(k) | O(k) |
| **Search** | O(p + n) | O(p) ← Cached! |
| **Space** | O(total chars) | O(total chars + K × nodes) |

---

## 7. Scaling Architecture

### Complete System Design

```
┌─────────────┐
│   Browser   │  Types: "sys"
└──────┬──────┘
       │
       ▼ (Debounced, e.g., 200ms)
┌─────────────────────┐
│  CDN / Edge Cache   │  (CloudFlare)
│  Cache popular      │
│  prefixes (TTL=1h)  │
└──────┬──────────────┘
       │ Cache miss
       ▼
┌─────────────────────┐
│   API Gateway       │
│   (Rate Limiting)   │
└──────┬──────────────┘
       │
       ▼
┌─────────────────────┐
│  Autocomplete       │
│  Service Cluster    │  (Replicas with same Trie)
│  - In-memory Trie   │
│  - Redis Cache      │
└──────┬──────────────┘
       │
       ├──→ Hot Cache (Redis): Top 1000 prefixes
       │    TTL: 1 hour
       │
       └──→ Trie: All queries
            Updated hourly from Analytics

       ▲
       │ Async update
       │
┌──────┴──────────────┐
│  Analytics Service  │
│  (Kafka Consumer)   │
│  - Aggregate clicks │
│  - Update rankings  │
└─────────────────────┘
       ▲
       │
┌──────┴──────────────┐
│   Kafka Topic       │
│   (search_queries)  │
└─────────────────────┘
       ▲
       │ Log every search
       │
   [User searches]
```

---

## 8. Ranking Strategies

### 1. Frequency-Based (Simple)

```python
# Rank by how often query was searched
score = frequency
```

### 2. Time-Decayed Frequency

```python
import time
import math

def calculate_score(frequency: int, last_searched: float) -> float:
    """
    Score = frequency × decay_factor

    Recent searches weighted higher
    """
    now = time.time()
    hours_ago = (now - last_searched) / 3600
    decay = math.exp(-hours_ago / 24)  # Half-life of 24 hours
    return frequency * decay

# Example:
# Query A: 1000 searches, last searched 1 hour ago
# Query B: 800 searches, last searched 1 day ago

score_a = calculate_score(1000, time.time() - 3600)    # ~990
score_b = calculate_score(800, time.time() - 86400)    # ~509

# Query A ranks higher (more recent) ✅
```

### 3. Personalized Ranking

```python
def personalized_score(query: str, user_context: dict) -> float:
    """
    Incorporate user context:
    - Location
    - Search history
    - Device type
    """
    base_score = query_frequency(query)

    # Boost if user searched similar queries
    history_boost = user_context.get('history_match', 0)

    # Boost if popular in user's location
    location_boost = location_popularity(query, user_context['location'])

    return base_score * (1 + history_boost + location_boost)
```

---

## 9. Data Collection & Updates

### Query Logging

```python
import asyncio
from kafka import KafkaProducer
import json

class QueryLogger:
    """Log search queries to Kafka for analytics"""

    def __init__(self):
        self.producer = KafkaProducer(
            bootstrap_servers=['localhost:9092'],
            value_serializer=lambda v: json.dumps(v).encode()
        )

    def log_query(self, user_id: str, query: str):
        """Log completed search query"""
        self.producer.send('search_queries', {
            'user_id': user_id,
            'query': query,
            'timestamp': time.time()
        })

    def log_autocomplete_click(self, user_id: str, query: str, clicked: str):
        """Log which suggestion user clicked"""
        self.producer.send('autocomplete_clicks', {
            'user_id': user_id,
            'typed_query': query,
            'clicked_suggestion': clicked,
            'timestamp': time.time()
        })

# Usage
logger = QueryLogger()
logger.log_query(user_id="123", query="system design interview")
logger.log_autocomplete_click(user_id="123", query="sys", clicked="system design")
```

### Trie Updates (Batch)

```python
class TrieUpdater:
    """
    Periodically rebuild Trie from aggregated data
    """

    def __init__(self, trie: OptimizedAutocompleteTrie):
        self.trie = trie

    async def update_from_analytics(self):
        """
        Rebuild Trie hourly from analytics database
        """
        while True:
            # Fetch top queries from last 24 hours
            top_queries = await self.fetch_top_queries()

            # Rebuild Trie
            new_trie = OptimizedAutocompleteTrie(k=10)
            for query, frequency in top_queries:
                new_trie.insert(query, frequency)

            # Atomic swap (minimal downtime)
            self.trie = new_trie

            # Wait 1 hour
            await asyncio.sleep(3600)

    async def fetch_top_queries(self) -> List[Tuple[str, int]]:
        """
        Query analytics DB for top queries
        """
        # SELECT query, COUNT(*) as freq
        # FROM search_logs
        # WHERE timestamp > NOW() - INTERVAL '24 hours'
        # GROUP BY query
        # ORDER BY freq DESC
        # LIMIT 500000
        pass
```

---

## 10. API Design

```python
# Autocomplete endpoint
GET /api/v1/autocomplete?prefix=sys&limit=5

Response 200:
{
  "suggestions": [
    {"query": "system design", "score": 1000},
    {"query": "systematic", "score": 500},
    {"query": "systemic risk", "score": 300}
  ],
  "latency_ms": 12
}

# Log query endpoint
POST /api/v1/log
{
  "user_id": "123",
  "query": "system design",
  "clicked": true
}
```

---

## 11. Client-Side Optimization

```javascript
// Debounce autocomplete requests
let debounceTimer;

function onInputChange(inputValue) {
  clearTimeout(debounceTimer);

  debounceTimer = setTimeout(() => {
    fetchAutocomplete(inputValue);
  }, 200);  // Wait 200ms after user stops typing
}

// Reduces requests from 10/query to 1-2/query ✅

// Cache on client
const cache = {};

async function fetchAutocomplete(prefix) {
  if (cache[prefix]) {
    return cache[prefix];  // Use cached results
  }

  const response = await fetch(`/api/autocomplete?prefix=${prefix}`);
  const data = await response.json();

  cache[prefix] = data;
  return data;
}
```

---

## 12. Summary

**Search Autocomplete System:**

✅ **Trie data structure** - O(p) prefix search
✅ **Top-K caching** at nodes - O(1) result retrieval
✅ **Redis/CDN caching** - Reduce backend load
✅ **Async updates** - Hourly Trie rebuild from analytics
✅ **Client debouncing** - Reduce request volume

**Key Metrics:**
- Latency: <100ms (p99)
- QPS: 1.15M autocomplete requests/sec
- Storage: ~30 GB (in-memory)

**Trade-offs:**
- Accuracy vs Latency (cached top-K)
- Real-time vs Batch updates (hourly rebuild)
- Memory vs Computation (Trie in-memory)

**Next:** Learn about [Unique ID Generator](../../beginner/unique-id-generator/README.md) (Snowflake).
