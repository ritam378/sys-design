# System Design Interview Preparation

A comprehensive repository of system design case studies following **Alex Xu's methodology** for technical interview preparation. This repository contains detailed breakdowns of 35+ real-world system designs with step-by-step analysis, capacity estimations, and architectural diagrams.

## 📚 What's Inside

- **35+ System Design Case Studies** organized by difficulty level
- **Core Fundamentals** - Scalability, databases, caching, networking concepts
- **Reusable Building Blocks** - Rate limiters, consistent hashing, distributed locks
- **Interview Framework** - Alex Xu's 4-step approach with templates
- **Python Implementations** - Code samples for key algorithms
- **Visual Diagrams** - Architecture and data flow diagrams using Mermaid

## 🎯 Who Is This For?

- Software engineers preparing for system design interviews (L4-L6)
- Students learning distributed systems and scalable architecture
- Developers wanting to understand real-world system design patterns
- Anyone studying for FAANG/MANGA interviews

## 🗂️ Repository Structure

```
sys-design/
├── 01-fundamentals/          # Core concepts and building blocks
├── 02-building-blocks/       # Reusable components with code
├── 03-system-designs/        # Main case studies
│   ├── beginner/            # 8 foundational designs
│   ├── intermediate/        # 12 moderate complexity designs
│   └── advanced/            # 15 enterprise-scale designs
├── 04-interview-prep/        # Interview strategies and tips
├── 05-references/           # Curated learning resources
└── 06-templates/            # Standard templates for designs
```

## 🚀 Quick Start

### Learning Path

**For Beginners:**
1. Start with [Fundamentals](01-fundamentals/) - understand core concepts
2. Study [Building Blocks](02-building-blocks/) - learn reusable patterns
3. Practice with [Beginner Designs](03-system-designs/beginner/) - URL Shortener, Pastebin
4. Use the [Interview Framework](04-interview-prep/framework-cheatsheet.md)

**For Intermediate:**
1. Review [Fundamentals](01-fundamentals/) if needed
2. Dive into [Intermediate Designs](03-system-designs/intermediate/) - News Feed, Chat System
3. Focus on trade-offs and bottleneck analysis
4. Practice with [Mock Scenarios](04-interview-prep/mock-interviews/)

**For Advanced:**
1. Tackle [Advanced Designs](03-system-designs/advanced/) - YouTube, Uber, Payment Systems
2. Study multiple approaches for each problem
3. Deep dive into distributed systems patterns
4. Practice explaining designs clearly and concisely

## 📖 Featured Case Studies

### Beginner Level
| System | Key Concepts | Difficulty |
|--------|-------------|------------|
| [URL Shortener](03-system-designs/beginner/url-shortener/) | Hashing, Base62 encoding, Database design | ⭐ |
| [Pastebin](03-system-designs/beginner/pastebin/) | Object storage, Expiration policies | ⭐ |
| [Unique ID Generator](03-system-designs/beginner/unique-id-generator/) | Distributed systems, Snowflake algorithm | ⭐⭐ |
| [Key-Value Store](03-system-designs/beginner/key-value-store/) | CAP theorem, Consistent hashing | ⭐⭐ |

### Intermediate Level
| System | Key Concepts | Difficulty |
|--------|-------------|------------|
| [News Feed System](03-system-designs/intermediate/news-feed/) | Fan-out patterns, Ranking algorithms | ⭐⭐⭐ |
| [Chat System](03-system-designs/intermediate/chat-system/) | WebSockets, Message queues, Read receipts | ⭐⭐⭐ |
| [Notification System](03-system-designs/intermediate/notification-system/) | Multi-channel delivery, Rate limiting | ⭐⭐⭐ |
| [Web Crawler](03-system-designs/intermediate/web-crawler/) | Distributed crawling, Politeness | ⭐⭐⭐ |

### Advanced Level
| System | Key Concepts | Difficulty |
|--------|-------------|------------|
| [Video Streaming (YouTube)](03-system-designs/advanced/video-streaming/) | CDN, Adaptive bitrate, Recommendation | ⭐⭐⭐⭐⭐ |
| [Ride-Sharing (Uber)](03-system-designs/advanced/ride-sharing/) | Geo-location, Matching algorithms, Real-time tracking | ⭐⭐⭐⭐⭐ |
| [Payment System](03-system-designs/advanced/payment-system/) | Transactions, Idempotency, Double-entry ledger | ⭐⭐⭐⭐⭐ |
| [Distributed Message Queue](03-system-designs/advanced/distributed-message-queue/) | Partitioning, Replication, Exactly-once delivery | ⭐⭐⭐⭐⭐ |

## 🎓 Alex Xu's 4-Step Framework

Each design follows this proven interview approach:

### 1. Understand the Problem and Establish Design Scope (3-10 min)
- Ask clarifying questions
- Define functional and non-functional requirements
- Establish constraints and assumptions
- Identify users and scale

### 2. Propose High-Level Design and Get Buy-In (10-15 min)
- Sketch initial architecture
- Identify major components
- Present API design
- Get interviewer feedback

### 3. Design Deep Dive (10-25 min)
- Dive into 2-3 critical components
- Discuss trade-offs and alternatives
- Show technical depth
- Address bottlenecks

### 4. Wrap Up (3-5 min)
- Identify potential improvements
- Discuss error cases and edge scenarios
- Talk about monitoring and operations
- Recap key decisions

## 🛠️ Technologies Covered

- **Databases:** PostgreSQL, MySQL, MongoDB, Cassandra, Redis
- **Message Queues:** Kafka, RabbitMQ, AWS SQS
- **Caching:** Redis, Memcached, CDN
- **Storage:** S3, HDFS, Object Storage
- **Search:** Elasticsearch, Solr
- **Monitoring:** Prometheus, Grafana, ELK Stack
- **Load Balancing:** NGINX, HAProxy, AWS ELB
- **Protocols:** HTTP/HTTPS, WebSockets, gRPC, TCP/UDP

## 📊 Estimation Techniques

Learn back-of-the-envelope calculations for:
- **Traffic estimation:** QPS, peak load, DAU/MAU
- **Storage estimation:** Data size, growth rate, retention
- **Bandwidth estimation:** Read/write throughput
- **Memory estimation:** Cache sizing, in-memory data

### Standard Assumptions
```
1 million requests/day ≈ 12 requests/second
1 billion requests/day ≈ 12,000 requests/second
100 million DAU with 10 actions/day ≈ 12,000 QPS
```

## 🎯 Interview Tips

1. **Ask Before You Design** - Clarify requirements upfront
2. **Start Simple** - Begin with basic design, then iterate
3. **Think Out Loud** - Explain your reasoning
4. **Consider Trade-offs** - No perfect solution exists
5. **Use Numbers** - Back-of-envelope calculations show rigor
6. **Draw Diagrams** - Visual communication is key
7. **Know Your Bottlenecks** - Identify single points of failure
8. **Scale Gradually** - Don't over-engineer from the start

## 📚 Recommended Resources

### Books
- **System Design Interview Vol 1 & 2** by Alex Xu (primary reference)
- **Designing Data-Intensive Applications** by Martin Kleppmann
- **Web Scalability for Startup Engineers** by Artur Ejsmont

### Online Courses
- Grokking the System Design Interview (educative.io)
- System Design Primer (GitHub)
- ByteByteGo (Alex Xu's platform)

### Websites
- [High Scalability](http://highscalability.com/)
- [System Design Primer](https://github.com/donnemartin/system-design-primer)
- Engineering blogs: Netflix, Uber, Airbnb, Meta

## 🤝 Contributing

Contributions are welcome! Please see [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

### How to Contribute
- Add new system design case studies
- Improve existing designs with diagrams
- Fix errors or clarify explanations
- Add code implementations
- Share interview experiences

## 📝 License

This repository is for educational purposes. System design concepts are based on publicly available information and standard industry practices.

## 🙏 Acknowledgments

- **Alex Xu** - For the excellent System Design Interview books and methodology
- **Martin Kleppmann** - For Designing Data-Intensive Applications
- The open-source community for sharing knowledge

## 📧 Contact

Have questions or suggestions? Feel free to open an issue or reach out!

---

**Happy Learning! Good luck with your interviews!** 🚀

Star ⭐ this repo if you find it helpful!
