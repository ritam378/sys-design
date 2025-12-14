# Building Blocks for System Design

Reusable components and patterns that appear across multiple system designs. Understanding these building blocks helps you quickly assemble complex systems in interviews.

## Overview

These are the "Lego pieces" of system design. Master these components and you can mix and match them to solve most system design problems.

## Components

### 1. [Rate Limiter](rate-limiter/)
Control the rate of requests to prevent abuse and ensure fair usage.

**Algorithms:**
- Token Bucket (Recommended)
- Leaky Bucket
- Fixed Window Counter
- Sliding Window Log
- Sliding Window Counter

**Use Cases:**
- API rate limiting (GitHub, Stripe)
- DDoS protection
- Fair resource allocation
- Cost control for paid APIs

**Implementation:** Python code with Redis

---

### 2. [Consistent Hashing](consistent-hashing/)
Distribute data across nodes minimizing rehashing when nodes are added/removed.

**Key Concepts:**
- Hash ring
- Virtual nodes
- Replication factor
- Load distribution

**Use Cases:**
- Database sharding (Cassandra, DynamoDB)
- Distributed caching (Memcached)
- Load balancing
- CDN routing

**Implementation:** Python code with visualization

---

### 3. [Bloom Filter](bloom-filter/)
Space-efficient probabilistic data structure to test set membership.

**Characteristics:**
- False positives possible
- False negatives impossible
- Extremely space-efficient

**Use Cases:**
- Check if username exists (Medium, Quora)
- Avoid disk lookups (Cassandra, HBase)
- Malicious URL detection (Chrome)
- Deduplication (web crawlers)

**Implementation:** Python with hash functions

---

### 4. [Distributed Lock](distributed-lock/)
Coordinate access to shared resources in distributed systems.

**Approaches:**
- Redis-based (Redlock)
- ZooKeeper-based
- Database-based
- Etcd-based

**Use Cases:**
- Prevent double booking
- Leader election
- Job scheduling (cron)
- Distributed transactions

**Implementation:** Python with Redis

---

### 5. [LRU Cache](lru-cache/)
Least Recently Used cache eviction policy.

**Data Structures:**
- HashMap + Doubly Linked List
- OrderedDict (Python)

**Complexity:**
- Get: O(1)
- Put: O(1)

**Use Cases:**
- In-memory caching
- Browser cache
- CPU cache
- Database query cache

**Implementation:** Python with full LRU implementation

---

### 6. [Merkle Tree](merkle-tree/)
Efficiently verify and sync data in distributed systems.

**Key Concepts:**
- Hash tree
- Root hash
- Leaf nodes

**Use Cases:**
- Data synchronization (Cassandra, DynamoDB)
- Version control (Git)
- Blockchain (Bitcoin)
- File integrity (IPFS)

**Implementation:** Python with hashing

---

### 7. [Trie (Prefix Tree)](trie/)
Tree data structure for efficient string prefix operations.

**Complexity:**
- Insert: O(m) where m = string length
- Search: O(m)
- Prefix search: O(p + n) where p = prefix length, n = results

**Use Cases:**
- Autocomplete (Google Search)
- Spell checker
- IP routing (longest prefix match)
- Word games

**Implementation:** Python with full Trie class

---

### 8. [Circuit Breaker](circuit-breaker/)
Prevent cascading failures in distributed systems.

**States:**
- Closed (normal)
- Open (failing)
- Half-Open (testing)

**Use Cases:**
- Microservices communication
- Third-party API calls
- Database connections
- Prevent cascading failures

**Implementation:** Python with state machine

---

### 9. [Service Discovery](service-discovery/)
Dynamically locate services in distributed systems.

**Approaches:**
- Client-side (Netflix Eureka)
- Server-side (Consul, etcd)
- DNS-based

**Use Cases:**
- Microservices architecture
- Load balancing
- Health checking
- Dynamic scaling

**Tools:** Consul, etcd, ZooKeeper

---

### 10. [Message Queue](message-queue/)
Asynchronous communication between services.

**Patterns:**
- Point-to-Point
- Pub-Sub
- Request-Reply
- Priority Queue

**Use Cases:**
- Decouple services
- Load leveling
- Async processing
- Event-driven architecture

**Technologies:** Kafka, RabbitMQ, SQS

---

## Quick Reference Table

| Component | Time Complexity | Space Complexity | Best Use Case |
|-----------|----------------|------------------|---------------|
| **Rate Limiter** | O(1) | O(n users) | API throttling |
| **Consistent Hashing** | O(log n) | O(n nodes) | Data distribution |
| **Bloom Filter** | O(k) k=hashes | O(m bits) | Membership test |
| **Distributed Lock** | O(1) | O(n locks) | Coordination |
| **LRU Cache** | O(1) | O(capacity) | Fast retrieval |
| **Merkle Tree** | O(log n) | O(n) | Data sync |
| **Trie** | O(m) m=length | O(alphabet × n) | Prefix search |
| **Circuit Breaker** | O(1) | O(1) | Fault tolerance |

## How to Use These Building Blocks

### In Interviews

1. **Identify the pattern** - What problem are you solving?
2. **Choose the right block** - Which component fits best?
3. **Explain trade-offs** - Why this over alternatives?
4. **Discuss implementation** - How would you build it?

### Example: Designing a Chat System

```
Problem: Need to handle rate limiting for messages

Building Block: Rate Limiter (Token Bucket)
Why: Prevents spam, ensures fair usage
Trade-off: Token bucket vs fixed window (token bucket is smoother)
Implementation: Redis-based with user_id as key

Problem: Need to distribute users across chat servers

Building Block: Consistent Hashing
Why: Minimize rehashing when adding/removing servers
Trade-off: Consistent hashing vs simple hash (better for dynamic scaling)
Implementation: Hash ring with virtual nodes
```

## Common Combinations

### Cache + Bloom Filter
```
Use Case: Check if key exists before expensive cache lookup
Example: Before querying distributed cache, use bloom filter
Benefit: Avoid cache stampede on non-existent keys
```

### Consistent Hashing + Replication
```
Use Case: Distribute data with fault tolerance
Example: Cassandra uses both for data placement
Benefit: Even distribution + high availability
```

### Rate Limiter + Circuit Breaker
```
Use Case: Protect downstream services
Example: API Gateway combining both patterns
Benefit: Prevent overload + graceful degradation
```

### Trie + Caching
```
Use Case: Fast autocomplete with popular queries cached
Example: Google Search combining both
Benefit: O(1) for popular, O(m) for long tail
```

## Implementation Priority

### Must Implement (Practice These First)
1. **Rate Limiter** - Extremely common in interviews
2. **LRU Cache** - Classic algorithm question
3. **Consistent Hashing** - Distributed systems fundamental
4. **Trie** - Search and autocomplete systems

### Should Implement
5. **Bloom Filter** - Shows advanced knowledge
6. **Circuit Breaker** - Reliability pattern
7. **Distributed Lock** - Coordination primitive

### Nice to Know
8. **Merkle Tree** - Data sync pattern
9. **Service Discovery** - Microservices pattern
10. **Message Queue** - Usually use existing (Kafka)

## Learning Path

### Week 1: Fundamentals
- Study rate limiter algorithms
- Implement LRU cache from scratch
- Understand consistent hashing theory

### Week 2: Advanced Structures
- Implement Trie for autocomplete
- Build bloom filter
- Create circuit breaker state machine

### Week 3: Distributed Primitives
- Distributed lock with Redis
- Service discovery patterns
- Message queue patterns

### Week 4: Integration
- Combine building blocks in designs
- Practice explaining trade-offs
- Mock interviews using these patterns

## Code Examples

All building blocks include:
- ✅ Full Python implementation
- ✅ Unit tests
- ✅ Usage examples
- ✅ Time/space complexity analysis
- ✅ Visual diagrams

## Interview Tips

### Common Questions

**"How would you implement X?"**
- Start with interface/API
- Discuss data structures
- Analyze complexity
- Mention trade-offs

**"When would you use X over Y?"**
- Compare use cases
- Discuss trade-offs
- Give real-world examples
- Mention constraints

**"What are the limitations of X?"**
- Discuss edge cases
- Mention failure modes
- Explain workarounds
- Suggest alternatives

### How to Present

1. **Start with the problem** - Why do we need this?
2. **Explain the concept** - How does it work?
3. **Show the implementation** - Key data structures
4. **Discuss trade-offs** - Pros and cons
5. **Give examples** - Real-world usage

## Additional Resources

### Books
- **Designing Data-Intensive Applications** (Chapter 3, 6, 7)
- **Elements of Programming Interviews** (Chapters on data structures)
- **Cracking the Coding Interview** (System design chapter)

### Articles
- [Consistent Hashing Explained](https://www.toptal.com/big-data/consistent-hashing)
- [Bloom Filters by Example](https://llimllib.github.io/bloomfilter-tutorial/)
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)

### Videos
- [MIT 6.824: Distributed Systems](https://www.youtube.com/playlist?list=PLrw6a1wE39_tb2fErI4-WkMbsvGQk9_UB)
- [System Design Interview YouTube Channel](https://www.youtube.com/c/SystemDesignInterview)

---

**Next Steps:** Pick a building block, implement it from scratch, then find it in a system design (e.g., rate limiter in API Gateway, consistent hashing in URL Shortener).
