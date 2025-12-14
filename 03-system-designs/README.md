# System Design Case Studies

This directory contains 35+ system design case studies organized by difficulty level. Each design follows Alex Xu's methodology with detailed explanations, diagrams, and code samples.

## How to Use This Resource

### For Beginners
1. Start with [Fundamentals](../01-fundamentals/) to build foundation
2. Study the [Template](../06-templates/system-design-template.md) to understand the structure
3. Work through **Beginner** designs in order
4. Practice explaining each design out loud

### For Intermediate Learners
1. Review **Beginner** designs quickly
2. Focus on **Intermediate** designs
3. Pay attention to trade-offs and alternatives
4. Try to design before reading the solution

### For Advanced Practitioners
1. Skip to **Advanced** designs
2. Compare your approach with the provided solution
3. Focus on distributed systems concepts
4. Practice under time constraints (45 minutes)

## Directory Structure

```
03-system-designs/
├── beginner/           # 8 foundational designs
├── intermediate/       # 12 moderate complexity designs
└── advanced/          # 15 enterprise-scale designs
```

---

## Beginner Level (8 Designs)

Perfect for those new to system design or preparing for entry-level/mid-level interviews.

| # | System | Difficulty | Key Concepts | Status |
|---|--------|------------|--------------|--------|
| 1 | [URL Shortener](beginner/url-shortener/) | ⭐ | Hashing, Base62, Caching | ✅ Complete |
| 2 | [Pastebin](beginner/pastebin/) | ⭐ | Object storage, Expiration | 📝 In Progress |
| 3 | [Unique ID Generator](beginner/unique-id-generator/) | ⭐⭐ | Snowflake, Distributed systems | 📝 In Progress |
| 4 | [Key-Value Store](beginner/key-value-store/) | ⭐⭐ | Hash tables, CAP theorem | 📝 In Progress |
| 5 | [Parking Lot](beginner/parking-lot/) | ⭐ | OOP + System design | 📝 In Progress |
| 6 | [Library Management](beginner/library-management/) | ⭐ | CRUD, Relationships | 📝 In Progress |
| 7 | [File Storage](beginner/file-storage/) | ⭐⭐ | Chunking, Metadata | 📝 In Progress |
| 8 | [Leaderboard](beginner/leaderboard/) | ⭐ | Sorted sets, Redis | 📝 In Progress |

**Time to Complete:** 2-3 weeks (1-2 designs per day)

---

## Intermediate Level (12 Designs)

For those comfortable with basics and preparing for senior engineer interviews (L4-L5).

| # | System | Difficulty | Key Concepts | Status |
|---|--------|------------|--------------|--------|
| 1 | [Rate Limiter](intermediate/rate-limiter/) | ⭐⭐⭐ | Token bucket, Sliding window | 📝 In Progress |
| 2 | [News Feed](intermediate/news-feed/) | ⭐⭐⭐ | Fan-out, Ranking algorithms | 📝 In Progress |
| 3 | [Notification System](intermediate/notification-system/) | ⭐⭐⭐ | Multi-channel, Priority queues | 📝 In Progress |
| 4 | [Web Crawler](intermediate/web-crawler/) | ⭐⭐⭐ | BFS, Politeness, Deduplication | 📝 In Progress |
| 5 | [Search Autocomplete](intermediate/search-autocomplete/) | ⭐⭐⭐ | Trie, Prefix search | 📝 In Progress |
| 6 | [Chat System](intermediate/chat-system/) | ⭐⭐⭐ | WebSockets, Message queues | 📝 In Progress |
| 7 | [Photo Sharing](intermediate/photo-sharing/) | ⭐⭐⭐ | CDN, Image processing | 📝 In Progress |
| 8 | [E-commerce Catalog](intermediate/ecommerce-catalog/) | ⭐⭐⭐ | Search, Filtering, Inventory | 📝 In Progress |
| 9 | [Booking System](intermediate/booking-system/) | ⭐⭐⭐ | Concurrency, Locking | 📝 In Progress |
| 10 | [Online Voting](intermediate/online-voting/) | ⭐⭐⭐ | Idempotency, Security | 📝 In Progress |
| 11 | [Collaborative Docs](intermediate/collaborative-docs/) | ⭐⭐⭐⭐ | OT, CRDT, WebSockets | 📝 In Progress |
| 12 | [API Gateway](intermediate/api-gateway/) | ⭐⭐⭐ | Routing, Auth, Rate limiting | 📝 In Progress |

**Time to Complete:** 3-4 weeks (1 design per day)

---

## Advanced Level (15 Designs)

For experienced engineers targeting staff/principal roles or FAANG interviews.

| # | System | Difficulty | Key Concepts | Status |
|---|--------|------------|--------------|--------|
| 1 | [Video Streaming](advanced/video-streaming/) | ⭐⭐⭐⭐⭐ | CDN, Adaptive bitrate, Transcoding | 📝 In Progress |
| 2 | [Cloud Storage](advanced/cloud-storage/) | ⭐⭐⭐⭐⭐ | Sync, Conflict resolution, Versioning | 📝 In Progress |
| 3 | [Ride-Sharing](advanced/ride-sharing/) | ⭐⭐⭐⭐⭐ | Geo-location, Matching, Real-time | 📝 In Progress |
| 4 | [Food Delivery](advanced/food-delivery/) | ⭐⭐⭐⭐⭐ | Multi-party matching, Routing | 📝 In Progress |
| 5 | [Distributed Message Queue](advanced/distributed-message-queue/) | ⭐⭐⭐⭐⭐ | Partitioning, Replication, Ordering | 📝 In Progress |
| 6 | [Payment System](advanced/payment-system/) | ⭐⭐⭐⭐⭐ | Transactions, Ledger, Idempotency | 📝 In Progress |
| 7 | [Social Network](advanced/social-network/) | ⭐⭐⭐⭐⭐ | Graph database, Feed, Recommendations | 📝 In Progress |
| 8 | [Search Engine](advanced/search-engine/) | ⭐⭐⭐⭐⭐ | Inverted index, PageRank, Crawling | 📝 In Progress |
| 9 | [Metrics Monitoring](advanced/metrics-monitoring/) | ⭐⭐⭐⭐⭐ | Time-series, Aggregation, Alerting | 📝 In Progress |
| 10 | [Ad Click Aggregation](advanced/ad-click-aggregation/) | ⭐⭐⭐⭐⭐ | Stream processing, Real-time analytics | 📝 In Progress |
| 11 | [Stock Trading](advanced/stock-trading/) | ⭐⭐⭐⭐⭐ | Low latency, Order matching | 📝 In Progress |
| 12 | [CDN](advanced/cdn/) | ⭐⭐⭐⭐ | Edge caching, Geo-routing | 📝 In Progress |
| 13 | [Distributed Cache](advanced/distributed-cache/) | ⭐⭐⭐⭐ | Consistent hashing, Eviction | 📝 In Progress |
| 14 | [Email Service](advanced/email-service/) | ⭐⭐⭐⭐ | Spam detection, Attachments, SMTP | 📝 In Progress |
| 15 | [Recommendation System](advanced/recommendation-system/) | ⭐⭐⭐⭐⭐ | ML, Collaborative filtering, Real-time | 📝 In Progress |

**Time to Complete:** 4-6 weeks (1 design per day)

---

## Learning Path

### Week 1-2: Beginner Foundations
- Complete all 8 beginner designs
- Focus on understanding basic patterns
- Practice drawing architecture diagrams
- Master back-of-envelope calculations

### Week 3-4: Intermediate Challenges
- Complete 6-8 intermediate designs
- Study trade-offs deeply
- Implement core algorithms (rate limiter, autocomplete)
- Practice explaining designs out loud

### Week 5-6: Advanced Mastery
- Complete 5-7 advanced designs
- Focus on distributed systems concepts
- Study real-world implementations
- Mock interviews with peers

### Week 7+: Interview Preparation
- Review all designs
- Practice with time constraints (45 min each)
- Focus on communication skills
- Study company-specific patterns

## Interview Preparation Strategy

### 1. Understand the Framework (Week 1)
- Learn Alex Xu's 4-step approach
- Practice with beginner designs
- Master the template structure

### 2. Build Pattern Recognition (Week 2-4)
- Identify common patterns (caching, sharding, queues)
- Understand when to apply each pattern
- Study trade-offs between alternatives

### 3. Practice Communication (Week 5-6)
- Explain designs to friends/colleagues
- Record yourself and review
- Practice whiteboarding (physical or digital)

### 4. Mock Interviews (Week 7+)
- Use platforms like Pramp or interviewing.io
- Get feedback from experienced engineers
- Time yourself (45-60 minutes per design)

## Common Patterns Across Designs

### Scalability Patterns
- **Load Balancing** - Used in: All designs
- **Caching** - Used in: URL Shortener, News Feed, Video Streaming
- **Database Sharding** - Used in: Social Network, Chat System
- **Message Queues** - Used in: Notification, Email, Food Delivery
- **CDN** - Used in: Video Streaming, Photo Sharing, CDN

### Data Patterns
- **Eventual Consistency** - Used in: News Feed, Social Network
- **Strong Consistency** - Used in: Payment System, Stock Trading
- **Replication** - Used in: Cloud Storage, Distributed Cache
- **Partitioning** - Used in: Message Queue, Metrics Monitoring

### Architecture Patterns
- **Microservices** - Used in: E-commerce, Ride-Sharing
- **Event-Driven** - Used in: Notification, Food Delivery
- **CQRS** - Used in: E-commerce, Booking System
- **Saga Pattern** - Used in: Payment, Food Delivery

## Tips for Success

### Do's ✅
- Ask clarifying questions upfront
- Start with high-level design before diving deep
- Use numbers (back-of-envelope calculations)
- Draw diagrams (architecture, sequence, data flow)
- Discuss trade-offs and alternatives
- Consider both functional and non-functional requirements
- Talk about monitoring and operations

### Don'ts ❌
- Don't jump to implementation details too quickly
- Don't over-engineer for small scale
- Don't ignore edge cases
- Don't forget about failures and fault tolerance
- Don't skip estimation - it shows rigor
- Don't memorize designs - understand principles

## Recommended Study Order

### For FAANG Interviews
1. **Must-Know (Top 10):**
   - URL Shortener
   - News Feed System
   - Chat System
   - Video Streaming (YouTube)
   - Ride-Sharing (Uber)
   - Cloud Storage (Dropbox)
   - Rate Limiter
   - Search Autocomplete
   - Notification System
   - Web Crawler

2. **Should-Know (Next 10):**
   - Distributed Message Queue
   - Payment System
   - Metrics Monitoring
   - Social Network
   - E-commerce Catalog
   - Booking System
   - Photo Sharing
   - API Gateway
   - CDN
   - Distributed Cache

3. **Nice-to-Know (Remaining 15):**
   - All other designs

### For Startups/Scale-ups
Focus on practical designs:
- API Gateway
- Rate Limiter
- Notification System
- E-commerce Catalog
- Booking System
- Payment System

### For Infrastructure Roles
Focus on platform designs:
- Distributed Cache
- Message Queue
- CDN
- Metrics Monitoring
- API Gateway
- Search Engine

## Additional Resources

### Books
- **System Design Interview Vol 1 & 2** by Alex Xu
- **Designing Data-Intensive Applications** by Martin Kleppmann
- **Web Scalability for Startup Engineers** by Artur Ejsmont

### Online Courses
- [Grokking the System Design Interview](https://www.educative.io/courses/grokking-the-system-design-interview)
- [ByteByteGo](https://bytebytego.com/) by Alex Xu
- [System Design Primer](https://github.com/donnemartin/system-design-primer)

### Practice Platforms
- [Pramp](https://www.pramp.com/) - Free mock interviews
- [interviewing.io](https://interviewing.io/) - Anonymous interviews
- [Exponent](https://www.tryexponent.com/) - System design courses

### Engineering Blogs
- **Netflix Tech Blog** - https://netflixtechblog.com/
- **Uber Engineering** - https://eng.uber.com/
- **Meta Engineering** - https://engineering.fb.com/
- **AWS Architecture** - https://aws.amazon.com/architecture/
- **High Scalability** - http://highscalability.com/

---

**Good luck with your system design interviews!** Remember: consistent practice and understanding principles > memorizing designs.

**Questions or feedback?** Open an issue or contribute your own design!
