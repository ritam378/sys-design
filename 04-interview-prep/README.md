# Interview Preparation Guide

Complete guide to acing system design interviews with strategies, frameworks, and company-specific insights.

## Overview

System design interviews are different from coding interviews. They test your ability to:
- Design large-scale distributed systems
- Make informed trade-off decisions
- Communicate technical ideas clearly
- Think systematically about complexity

## Table of Contents

1. [Alex Xu's 4-Step Framework](#alex-xus-4-step-framework)
2. [Interview Timeline](#interview-timeline)
3. [Common Mistakes](#common-mistakes)
4. [Communication Tips](#communication-tips)
5. [Company-Specific Guides](#company-specific-guides)
6. [Practice Strategy](#practice-strategy)
7. [Resources](#resources)

---

## Alex Xu's 4-Step Framework

### Step 1: Understand the Problem and Establish Design Scope (3-10 min)

**Goal:** Clarify requirements and constraints

**Key Questions to Ask:**

**Functional Requirements:**
- What are the core features we need to build?
- What's the expected user flow?
- Do we need to support mobile, web, or both?
- Any specific features to prioritize?

**Non-Functional Requirements:**
- How many users? Daily active users (DAU)?
- What's the read/write ratio?
- What's the expected latency requirement?
- What's the availability requirement (99.9%? 99.99%)?
- Any data consistency requirements?

**Scope:**
- Are there any features we should NOT focus on?
- What's the time constraint for this interview?

**Example Dialogue:**
```
Interviewer: Design a URL shortener

You: Great! Let me clarify a few things:

Functional:
- Users can create short URLs from long URLs?
- When visiting short URL, redirect to original?
- Should we support custom aliases?
- Any expiration for URLs?

Non-Functional:
- How many URLs created per day?
- What's the read/write ratio? (Assume 100:1)
- Any latency requirements? (Target < 200ms?)
- Availability expectations? (99.99%?)

Scope:
- Should I focus on analytics/tracking?
- User authentication needed?

Interviewer: Focus on core shortening and redirect. 100M URLs/month,
100:1 read/write, < 200ms latency, 99.9% availability.
```

### Step 2: Propose High-Level Design and Get Buy-In (10-15 min)

**Goal:** Present initial architecture, get feedback

**Steps:**

1. **Draw high-level components**
   - Client
   - Load balancer
   - Application servers
   - Database
   - Cache

2. **Explain data flow**
   - Write path (how data enters system)
   - Read path (how data is retrieved)

3. **Present API design**
   - Key endpoints
   - Request/response formats

4. **Discuss database schema**
   - Main tables
   - Key indexes

5. **Get buy-in**
   - "Does this approach make sense?"
   - "Should I dive deeper into any component?"

**Example:**
```
You: Here's my high-level design:

[Draw diagram]
Client → Load Balancer → API Servers → Cache/Database

For URL creation:
- User sends POST request with long URL
- API server generates short code
- Store mapping in database
- Return short URL

For redirection:
- User visits short URL
- Check cache first, then database
- Return 301 redirect

API:
- POST /api/v1/shorten - Create short URL
- GET /{short_code} - Redirect

Database:
- Table: urls (id, short_code, long_url, created_at)
- Index on short_code for fast lookup

Does this make sense? Should I dive into any specific component?
```

### Step 3: Design Deep Dive (10-25 min)

**Goal:** Demonstrate technical depth in 2-3 areas

**Common Deep Dive Topics:**

**For URL Shortener:**
- Short code generation algorithm
- Handling collisions
- Database sharding strategy

**For Chat System:**
- WebSocket vs long polling
- Message delivery guarantee
- Read receipts implementation

**For Video Streaming:**
- Video transcoding pipeline
- Adaptive bitrate streaming
- CDN strategy

**Approach:**
1. Interviewer guides: "How would you generate short codes?"
2. Present algorithm with pseudocode
3. Discuss trade-offs
4. Mention alternatives

**Example:**
```
You: For short code generation, I have two approaches:

Approach 1: Hash-based
- Hash the long URL (MD5/SHA256)
- Take first 7 characters (base62)
- Check for collision, rehash if needed
- Pros: Same URL → same short code
- Cons: Collision handling complexity

Approach 2: Auto-incrementing ID + base62
- Get unique ID from counter
- Encode to base62 (7 chars = 3.5T combinations)
- Pros: No collisions, simple
- Cons: Predictable sequence

I'd choose Approach 2 for simplicity, use ZooKeeper for
distributed ID generation. Add random offset to make
IDs less predictable.

[Draw sequence diagram showing ID generation flow]
```

### Step 4: Wrap Up (3-5 min)

**Goal:** Show completeness and foresight

**Topics to Cover:**

**Bottlenecks and Improvements:**
- "If we had more time, I'd improve..."
- Database is SPOF → add replication
- Cache stampede → use cache warming
- Hot keys → local cache on servers

**Failure Scenarios:**
- What if database goes down?
- What if cache fails?
- How to handle network partitions?

**Monitoring and Operations:**
- Key metrics to track
- Alerting strategy
- Logging approach

**Scaling:**
- How to scale to 10x traffic?
- Multi-region deployment
- Database sharding

**Example:**
```
You: To summarize:

Bottlenecks:
- Single database → add primary-replica setup
- Hot URLs → cache at CDN level
- ID generation → use distributed Snowflake IDs

Monitoring:
- Track: QPS, latency (p99), error rate, cache hit ratio
- Alert on: DB down, high error rate, high latency

Scaling to 10x:
- Shard database by short_code prefix
- Add more read replicas
- Deploy in multiple regions

Error handling:
- If DB down → serve from cache (stale is ok)
- If cache down → fallback to DB
- Circuit breakers to prevent cascade failures
```

---

## Interview Timeline

### 45-Minute Interview Breakdown

```
0-5 min:   Introductions, problem statement
5-10 min:  Clarifying questions, requirements
10-25 min: High-level design, get buy-in
25-40 min: Deep dive (2-3 components)
40-45 min: Wrap up, bottlenecks, Q&A
```

### 60-Minute Interview Breakdown

```
0-5 min:   Introductions, problem statement
5-15 min:  Clarifying questions, estimation
15-30 min: High-level design, API, schema
30-50 min: Deep dive (3-4 components)
50-60 min: Scaling, failures, monitoring, Q&A
```

---

## Common Mistakes

### ❌ Mistake #1: Jumping to Solution Too Quickly

**Problem:** Not asking clarifying questions

**Example:**
```
Bad:
Interviewer: Design Twitter
You: [Immediately starts drawing architecture]

Good:
Interviewer: Design Twitter
You: Let me clarify the requirements first...
- Should I focus on newsfeed, tweeting, or both?
- How many DAU?
- What's more important: consistency or availability?
```

### ❌ Mistake #2: Over-Engineering for Small Scale

**Problem:** Using Kafka for 10 QPS

**Example:**
```
Bad:
100 requests/day → "We need Kafka, Kubernetes, microservices"

Good:
100 requests/day → "A single server with PostgreSQL is sufficient.
As we scale to 100K requests/day, we can add caching and replicas."
```

### ❌ Mistake #3: Not Using Numbers

**Problem:** Vague statements about scale

**Example:**
```
Bad:
"We'll need a lot of storage"

Good:
"100M users × 100 posts each × 1KB per post = 10TB storage.
With 3x replication, we need 30TB total."
```

### ❌ Mistake #4: Ignoring Trade-offs

**Problem:** Presenting only one solution

**Example:**
```
Bad:
"We'll use NoSQL because it scales better"

Good:
"We have two options:
1. SQL: ACID, complex queries, vertical scaling limit
2. NoSQL: Horizontal scaling, eventual consistency

For this use case, I'd choose SQL because we need
transactions for payments. NoSQL would be better for
a social feed where eventual consistency is acceptable."
```

### ❌ Mistake #5: Not Thinking About Failures

**Problem:** Assuming everything works perfectly

**Example:**
```
Bad:
[Design ends with single database, single server]

Good:
"Single points of failure:
- Database → add primary-replica replication
- Application → deploy multiple instances
- Load balancer → active-passive failover
- Cache → Redis cluster with Sentinel"
```

---

## Communication Tips

### Tip #1: Think Out Loud

**Why:** Interviewer wants to see your thought process

**Example:**
```
"I'm thinking about the database choice. For this use case,
we need strong consistency for financial transactions, so
I'm leaning toward PostgreSQL over Cassandra. However,
if we prioritize availability over consistency, Cassandra
might be better. Let me check the requirements again..."
```

### Tip #2: Use the Whiteboard Effectively

**Dos:**
- Draw boxes for components
- Use arrows to show data flow
- Label everything clearly
- Leave space for additions

**Don'ts:**
- Don't cram everything into one corner
- Don't use tiny writing
- Don't erase good work
- Don't draw without explaining

### Tip #3: Engage with the Interviewer

**Questions to Ask:**
- "Does this approach make sense?"
- "Should I dive deeper into X or move on?"
- "Is this the level of detail you're looking for?"
- "Would you like me to discuss Y?"

### Tip #4: Structure Your Answers

**Framework:**
```
1. Restate the question
2. State your approach
3. Explain your reasoning
4. Discuss trade-offs
5. Conclude with recommendation

Example:
"You asked about caching strategy. I'm going to use a
cache-aside pattern because [reasoning]. This gives us
[benefit] but has [trade-off]. An alternative would be
write-through, which [comparison]. For our read-heavy
use case, cache-aside is better."
```

### Tip #5: Be Honest About Unknowns

**Good Responses:**
```
"I'm not sure about the exact implementation of X, but
my understanding is [explain]. I would need to research
the specifics."

"That's a great question. I haven't worked with X before,
but I think the approach would be [reasoning]."

"I don't know off the top of my head. Could we discuss
the trade-offs between X and Y instead?"
```

---

## Company-Specific Guides

### Google
**Focus:** Scalability, reliability, latency
**Common Questions:** YouTube, Gmail, Google Maps
**Tips:** Emphasize distributed systems, use GCP services

### Meta (Facebook)
**Focus:** Social graphs, real-time, scale
**Common Questions:** News Feed, Messenger, Instagram
**Tips:** Discuss graph databases, fan-out strategies

### Amazon
**Focus:** Reliability, customer obsession, cost
**Common Questions:** E-commerce, recommendations, AWS services
**Tips:** Mention CAP theorem, use AWS services, discuss costs

### Microsoft
**Focus:** Enterprise, reliability, hybrid cloud
**Common Questions:** Office 365, Teams, Azure
**Tips:** Consider on-prem + cloud, enterprise features

### Netflix
**Focus:** Availability, chaos engineering, performance
**Common Questions:** Video streaming, recommendations
**Tips:** Discuss CDN, microservices, failover strategies

### Uber
**Focus:** Real-time, geo-location, reliability
**Common Questions:** Ride matching, routing, surge pricing
**Tips:** Geo-hashing, real-time matching algorithms

---

## Practice Strategy

### Phase 1: Learn Fundamentals (2 weeks)
- Complete all beginner designs
- Implement building blocks
- Master estimation techniques

### Phase 2: Pattern Recognition (2 weeks)
- Study intermediate designs
- Identify common patterns
- Practice explaining trade-offs

### Phase 3: Mock Interviews (2+ weeks)
- 2-3 mock interviews per week
- Record and review
- Get feedback from experienced engineers

### Practice Schedule

**Week 1-2: Foundations**
- Day 1-2: URL Shortener
- Day 3-4: Pastebin
- Day 5-6: Key-Value Store
- Day 7: Review and practice

**Week 3-4: Intermediate**
- Day 1-2: News Feed
- Day 3-4: Chat System
- Day 5-6: Rate Limiter
- Day 7: Mock interview

**Week 5-6: Advanced**
- Day 1-2: Video Streaming
- Day 3-4: Ride-Sharing
- Day 5-6: Payment System
- Day 7: Mock interview

**Week 7+: Polish**
- Daily: Review one design
- 2-3x per week: Mock interviews
- Focus on communication

---

## Mock Interview Checklist

### Before the Interview
- [ ] Whiteboard/drawing tool ready
- [ ] Timer set for 45-60 minutes
- [ ] Have questions prepared
- [ ] Review recent designs

### During the Interview
- [ ] Ask clarifying questions (5-10 min)
- [ ] Do back-of-envelope estimation
- [ ] Draw high-level architecture
- [ ] Get buy-in before deep dive
- [ ] Discuss trade-offs
- [ ] Cover failure scenarios
- [ ] Mention monitoring/operations

### After the Interview
- [ ] Ask for feedback
- [ ] Write down what went well
- [ ] Note areas for improvement
- [ ] Review missed concepts

---

## Resources

### Practice Platforms
- **Pramp** - Free peer mock interviews
- **interviewing.io** - Anonymous interviews with engineers
- **Exponent** - System design course + mocks

### YouTube Channels
- **Gaurav Sen** - System design basics
- **Tech Dummies** - Narendra L
- **System Design Interview**

### Books (Priority Order)
1. **System Design Interview Vol 1 & 2** - Alex Xu
2. **Designing Data-Intensive Applications** - Martin Kleppmann
3. **Web Scalability for Startup Engineers** - Artur Ejsmont

---

## Quick Reference: Common Questions

| Question | Focus Areas | Key Concepts |
|----------|------------|--------------|
| **URL Shortener** | Hashing, Caching | Base62, Redis |
| **Pastebin** | Object storage | S3, Expiration |
| **Rate Limiter** | Algorithms | Token bucket, Redis |
| **News Feed** | Ranking, Fan-out | Push vs pull |
| **Chat System** | Real-time | WebSockets, Message queue |
| **Video Streaming** | CDN, Encoding | Adaptive bitrate |
| **Uber** | Geo-location | QuadTree, matching |
| **Instagram** | Media storage | CDN, Image processing |
| **Twitter** | Social graph | Fan-out, Timeline |
| **WhatsApp** | Messaging | End-to-end encryption |

---

**Remember:** System design interviews are about demonstrating:
1. **Systematic thinking** - Follow a framework
2. **Trade-off analysis** - No perfect solution
3. **Communication** - Explain your reasoning
4. **Scale awareness** - Use numbers and estimates

**Good luck!** 🚀
