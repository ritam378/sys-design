# Fundamentals of System Design

This section covers the core concepts and building blocks that form the foundation of scalable system design. Master these fundamentals before diving into specific system designs.

## Overview

Understanding these concepts is crucial for:
- Making informed architectural decisions
- Discussing trade-offs in interviews
- Identifying bottlenecks and solutions
- Scaling systems effectively

## Topics Covered

### 1. [Scalability](scalability/)
Learn how to scale systems to handle millions of users

- [Horizontal vs Vertical Scaling](scalability/horizontal-vs-vertical-scaling.md)
- [Load Balancing](scalability/load-balancing.md)
- [Stateless vs Stateful Architecture](scalability/stateless-architecture.md)
- [Microservices vs Monolith](scalability/microservices.md)

**Key Concepts:** Horizontal scaling, load balancers (L4/L7), service discovery, auto-scaling

### 2. [Databases](databases/)
Understand data storage strategies and patterns

- [SQL vs NoSQL](databases/sql-vs-nosql.md)
- [Database Replication](databases/database-replication.md)
- [Database Sharding](databases/database-sharding.md)
- [Indexing Strategies](databases/indexing.md)
- [CAP Theorem](databases/cap-theorem.md)

**Key Concepts:** ACID, BASE, eventual consistency, master-slave, sharding keys

### 3. [Caching](caching/)
Optimize performance with effective caching strategies

- [Caching Strategies](caching/caching-strategies.md) - Cache-aside, write-through, write-back
- [Cache Invalidation](caching/cache-invalidation.md)
- [Content Delivery Networks (CDN)](caching/cdn.md)
- [Redis vs Memcached](caching/redis-vs-memcached.md)

**Key Concepts:** Cache hit ratio, eviction policies (LRU, LFU), cache stampede, thundering herd

### 4. [Message Queues](message-queues/)
Enable asynchronous communication and decoupling

- [Message Queue Patterns](message-queues/patterns.md)
- [Kafka vs RabbitMQ vs SQS](message-queues/kafka-rabbitmq-sqs.md)
- [Pub-Sub Pattern](message-queues/pub-sub-pattern.md)
- [Event-Driven Architecture](message-queues/event-driven.md)

**Key Concepts:** Producers, consumers, topics, partitions, message ordering, at-least-once/exactly-once delivery

### 5. [Networking](networking/)
Understand network protocols and communication patterns

- [HTTP/HTTPS/HTTP2](networking/http-https-websockets.md)
- [WebSockets](networking/websockets.md)
- [TCP vs UDP](networking/tcp-vs-udp.md)
- [DNS](networking/dns.md)
- [gRPC](networking/grpc.md)

**Key Concepts:** Request-response, long-polling, server-sent events, connection pooling

### 6. [Estimation](estimation/)
Master back-of-the-envelope calculations

- [Back-of-Envelope Calculations](estimation/back-of-envelope-calculations.md)
- [Capacity Planning](estimation/capacity-planning.md)
- [QPS and Traffic Estimation](estimation/qps-estimation.md)
- [Storage Estimation](estimation/storage-estimation.md)
- [Bandwidth Estimation](estimation/bandwidth-estimation.md)

**Key Concepts:** QPS, throughput, latency, availability (9s), storage multipliers

## Standard Numbers to Remember

### Latency Numbers Every Programmer Should Know

```
L1 cache reference                           0.5 ns
Branch mispredict                            5   ns
L2 cache reference                           7   ns
Mutex lock/unlock                           25   ns
Main memory reference                      100   ns
Compress 1K bytes with Snappy            3,000   ns  =   3 µs
Send 1K bytes over 1 Gbps network       10,000   ns  =  10 µs
Read 4K randomly from SSD              150,000   ns  = 150 µs
Read 1 MB sequentially from memory     250,000   ns  = 250 µs
Round trip within same datacenter      500,000   ns  = 500 µs
Read 1 MB sequentially from SSD      1,000,000   ns  =   1 ms
Disk seek                           10,000,000   ns  =  10 ms
Read 1 MB sequentially from disk    20,000,000   ns  =  20 ms
Send packet CA->Netherlands->CA    150,000,000   ns  = 150 ms
```

### Power of Two Table

```
Power           Exact Value         Approx Value
-----------------------------------------------
7               128
8               256
10              1,024               ~1 thousand
16              65,536              ~65 thousand
20              1,048,576           ~1 million
30              1,073,741,824       ~1 billion
32              4,294,967,296       ~4 billion
40              1,099,511,627,776   ~1 trillion
```

### Availability Numbers (The 9s)

```
Availability    Downtime per Year    Downtime per Month
-----------------------------------------------------
99%             3.65 days            7.20 hours
99.9%           8.76 hours           43.2 minutes
99.99%          52.56 minutes        4.32 minutes
99.999%         5.26 minutes         25.9 seconds
99.9999%        31.5 seconds         2.59 seconds
```

### Traffic Estimation Rules of Thumb

```
1 million requests/day ≈ 12 requests/second
100 million requests/day ≈ 1,200 requests/second
1 billion requests/day ≈ 12,000 requests/second

Read-heavy systems: 100:1 read/write ratio
Social networks: 10:1 read/write ratio
Write-heavy systems: 1:1 or more writes
```

### Storage Multipliers

```
1 Byte = 8 bits
1 KB = 1,000 bytes (or 1,024 bytes)
1 MB = 1,000 KB
1 GB = 1,000 MB
1 TB = 1,000 GB
1 PB = 1,000 TB
```

## Learning Path

### For Interview Preparation

1. **Start with Scalability** - Understand how systems grow
2. **Learn Database Patterns** - Master data storage strategies
3. **Study Caching** - Know when and how to cache
4. **Understand Message Queues** - Learn async patterns
5. **Practice Estimation** - Do calculations for every design
6. **Review Networking** - Know your protocols

### Practice Exercise

For each fundamental concept:
1. Read the detailed explanation
2. Understand when to use it
3. Know the trade-offs
4. Apply it to a real system
5. Practice explaining it clearly

## Common Interview Questions

- "How would you scale a database that's reaching its limits?"
- "When would you use a message queue instead of direct API calls?"
- "What caching strategy would you use for a read-heavy system?"
- "How do you ensure data consistency across distributed systems?"
- "Calculate the storage needed for X users over Y years"

## Additional Resources

### Books
- **Designing Data-Intensive Applications** by Martin Kleppmann (Chapter 1-3)
- **System Design Interview Vol 1** by Alex Xu (Chapter 1)

### Articles
- [System Design Primer](https://github.com/donnemartin/system-design-primer)
- [High Scalability Blog](http://highscalability.com/)

### Videos
- [MIT 6.824: Distributed Systems](https://www.youtube.com/playlist?list=PLrw6a1wE39_tb2fErI4-WkMbsvGQk9_UB)

---

**Next Steps:** After mastering these fundamentals, move on to [Building Blocks](../02-building-blocks/) to learn reusable system components.
