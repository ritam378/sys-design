# System Design & OOD Interview Preparation - Study Plan

**Goal:** Master system design and object-oriented design concepts for technical interviews
**Duration:** 8-12 weeks (adjustable based on your pace)
**Time Commitment:** 10-15 hours/week recommended

---

## 📋 Table of Contents

1. [Week 1-2: Foundations](#week-1-2-foundations)
2. [Week 3-4: OOD Beginner & Intermediate](#week-3-4-ood-beginner--intermediate)
3. [Week 5-6: OOD Advanced & Building Blocks](#week-5-6-ood-advanced--building-blocks)
4. [Week 7-8: System Design Beginner & Intermediate](#week-7-8-system-design-beginner--intermediate)
5. [Week 9-10: System Design Advanced](#week-9-10-system-design-advanced)
6. [Week 11-12: Practice & Mock Interviews](#week-11-12-practice--mock-interviews)
7. [Daily Study Routine](#daily-study-routine)
8. [Progress Tracking](#progress-tracking)
9. [Interview Tips](#interview-tips)

---

## Week 1-2: Foundations

### Goal
Build strong foundation in design patterns and core system design concepts.

### Topics to Cover

#### Design Patterns (6 days)
Study the 23 design patterns in this order:

**Creational (Day 1-2):**
- Singleton Pattern (most common in interviews)
- Factory Pattern (essential for OOD)
- Abstract Factory Pattern
- Builder Pattern (complex object creation)
- Prototype Pattern

**Structural (Day 3-4):**
- Adapter Pattern (interface compatibility)
- Decorator Pattern (dynamic feature addition)
- Facade Pattern (simplify complexity)
- Proxy Pattern (control access)
- Composite Pattern (tree structures)
- Bridge Pattern
- Flyweight Pattern

**Behavioral (Day 5-6):**
- Strategy Pattern (MOST IMPORTANT - used everywhere)
- Observer Pattern (event systems)
- State Pattern (state machines)
- Command Pattern (undo/redo)
- Template Method Pattern
- Iterator Pattern
- Chain of Responsibility
- Mediator Pattern
- Memento Pattern
- Visitor Pattern
- Interpreter Pattern

#### Building Blocks (Days 7-14)
Study in this order:

1. **Load Balancer** (Day 7) - Foundation for distributed systems
2. **Rate Limiter** (Day 8) - API protection
3. **Consistent Hashing** (Day 9) - Data distribution
4. **CDN** (Day 10) - Content delivery
5. **API Gateway** (Day 11) - Service routing
6. **Message Queue** (Day 12) - Async communication
7. **Service Discovery** (Day 13) - Dynamic service location
8. **Caching** (Day 14) - Performance optimization
9. **Database Sharding** (review)
10. **Merkle Tree** (review) - Data integrity

### Study Method
- **Time:** 1.5 hours/day
- **Read:** Pattern/building block README (30 min)
- **Sketch:** Draw class diagrams and flow diagrams (30 min)
- **Write:** Summarize key points in your own words (30 min)
- **Review:** Previous day's topics (10 min)

### Weekly Checklist
- [ ] Complete all 23 design patterns
- [ ] Complete 10 building blocks
- [ ] Create personal cheat sheet for top 10 patterns
- [ ] Can explain Strategy, Observer, Factory patterns clearly

---

## Week 3-4: OOD Beginner & Intermediate

### Goal
Master common OOD interview problems with clean implementation approach.

### Week 3: Beginner OOD (8 problems)

Study **2 problems per day** in this order:

**Day 1-2: Games (Easy warm-up)**
1. Deck of Cards (Day 1 AM) - 30 min
2. Tic-Tac-Toe (Day 1 PM) - 1 hour
3. Chess Game (Day 2) - 2 hours (more complex)

**Day 3-4: Systems**
4. Parking Lot (Day 3 AM) - 1.5 hours
5. Vending Machine (Day 3 PM) - 1 hour
6. LRU Cache (Day 4 AM) - 1 hour
7. Library Management (Day 4 PM) - 1.5 hours

**Day 5: Banking**
8. ATM Machine (Day 5) - 2 hours

**Day 6-7: Review & Practice**
- Revisit challenging problems
- Practice drawing class diagrams from memory
- Identify common patterns across problems

### Week 4: Intermediate OOD (7 problems)

Study **1-2 problems per day**:

**Day 1: Booking Systems**
1. Hotel Booking System (AM) - 2 hours
   - Focus: Date overlap detection, concurrency
2. Airline Reservation (PM) - 1.5 hours
   - Focus: Seat assignment, overbooking

**Day 2: E-commerce**
3. Shopping Cart (Full day) - 2.5 hours
   - Focus: Strategy pattern for discounts
   - Critical for Amazon, e-commerce interviews

**Day 3: Real-time Systems**
4. Elevator System (Full day) - 2.5 hours
   - Focus: SCAN/LOOK algorithms, scheduling
   - Common at Google, Microsoft

**Day 4: Games (Advanced)**
5. Chess Game (revisit with advanced features) - 2 hours
6. Tic-Tac-Toe (add Minimax AI) - 1 hour

**Day 5-7: Review & Mock Interviews**
- Practice explaining solutions out loud
- Time yourself: 35-40 min per problem
- Focus on clarifying requirements first (5 min)

### Study Method for Each Problem

**Step 1: Read (15 min)**
- Problem statement
- Requirements
- Core concepts

**Step 2: Design (30 min)**
- Draw class diagram on paper/whiteboard
- Identify design patterns
- List key classes and relationships

**Step 3: Deep Dive (45 min)**
- Study implementation approach
- Understand trade-offs
- Review common pitfalls

**Step 4: Practice (30 min)**
- Explain solution out loud (as if in interview)
- Write out key classes and methods
- Review follow-up questions

**Step 5: Solidify (15 min)**
- Create 1-page summary
- Note key takeaways
- Add to your pattern library

### Weekly Checklist
- [ ] Complete all 15 OOD problems (8 beginner + 7 intermediate)
- [ ] Can draw class diagrams for each problem in < 10 min
- [ ] Understand when to use Strategy, State, Observer patterns
- [ ] Practiced 3 problems out loud (mock interview style)

---

## Week 5-6: OOD Advanced & Building Blocks

### Goal
Master complex OOD problems and understand distributed system building blocks.

### Week 5: Advanced OOD (6 problems)

**Day 1-2: Ride-Sharing**
1. **Uber/Lyft App** (2 days) - Most important advanced problem
   - Day 1: Core design (matching, pricing, states)
   - Day 2: Advanced features (geospatial, surge pricing)
   - Focus: Real-time systems, state machines
   - Companies: Uber, Lyft, DoorDash, Instacart

**Day 3-4: E-commerce Platform**
2. **Amazon Shopping** (2 days)
   - Day 3: Catalog, cart, inventory management
   - Day 4: Search, recommendations, reviews
   - Focus: Inventory reservation, search optimization
   - Companies: Amazon, eBay, Shopify

**Day 5-6: Streaming Platform**
3. **Netflix/YouTube** (2 days)
   - Day 5: Content hierarchy, user profiles
   - Day 6: Recommendations, continue watching
   - Focus: Content organization, recommendation systems
   - Companies: Netflix, YouTube, Spotify

**Day 7: Review**
- Compare all three platforms
- Identify common patterns (State, Strategy, Observer)
- Practice system design aspects (scalability, real-time)

### Week 6: Deep Dive on Building Blocks

**Critical Building Blocks for Interviews:**

**Day 1-2: Distributed Data**
1. **Consistent Hashing** (Day 1 AM)
   - How data is distributed across servers
   - Why: Essential for distributed caching, databases
2. **Database Sharding** (Day 1 PM)
   - Horizontal partitioning strategies
3. **Merkle Tree** (Day 2)
   - Data integrity verification
   - Applications: Git, blockchain, distributed databases

**Day 3-4: Communication & Coordination**
4. **Message Queue** (Day 3 AM) - CRITICAL
   - Point-to-point vs pub-sub
   - Kafka, RabbitMQ, SQS comparison
5. **Service Discovery** (Day 3 PM)
   - Dynamic service registration
   - Consul, Eureka, Kubernetes DNS
6. **API Gateway** (Day 4)
   - Routing, authentication, rate limiting

**Day 5-6: Performance & Reliability**
7. **Load Balancer** (Day 5 AM) - CRITICAL
   - L4 vs L7, algorithms (round-robin, least connections)
8. **Rate Limiter** (Day 5 PM) - CRITICAL
   - Token bucket, leaky bucket algorithms
9. **Caching** (Day 6 AM) - CRITICAL
   - Write-through, write-back, cache-aside
   - Eviction policies (LRU, LFU)
10. **CDN** (Day 6 PM)
    - Content delivery, edge caching

**Day 7: Integration**
- How building blocks work together
- Draw architecture diagrams using multiple blocks

### Study Method for Building Blocks

**For Each Building Block (90 min):**

1. **Read Concept (20 min)**
   - What problem does it solve?
   - How does it work?
   - When to use it?

2. **Study Implementations (30 min)**
   - Algorithms (e.g., token bucket for rate limiter)
   - Trade-offs (e.g., L4 vs L7 load balancing)
   - Real-world systems (e.g., Redis for caching)

3. **Draw Diagrams (20 min)**
   - Architecture diagram
   - Request flow
   - Data structures

4. **Practice Explaining (20 min)**
   - Explain to imaginary interviewer
   - Answer "why" questions
   - Discuss trade-offs

### Weekly Checklist
- [ ] Complete 3 advanced OOD problems (Uber, Amazon, Netflix)
- [ ] Master 10 critical building blocks
- [ ] Can explain each building block in 2 minutes
- [ ] Drew architecture diagrams for each building block
- [ ] Practiced 2 advanced OOD problems out loud

---

## Week 7-8: System Design Beginner & Intermediate

### Goal
Apply building blocks to design real-world systems at scale.

### Week 7: Beginner System Design (12 problems)

**Strategy:** Study 2 problems per day, focus on architecture patterns.

**Day 1: URL/Link Systems**
1. URL Shortener (AM) - Classic interview problem
2. Pastebin (PM) - Similar pattern

**Day 2: Social Features**
3. Rate Limiter (AM) - API protection
4. Notification Service (PM) - Multi-channel delivery

**Day 3: Content Systems**
5. News Feed (AM) - Fan-out patterns
6. Search Autocomplete (PM) - Trie data structure

**Day 4: Messaging**
7. Chat Application (AM) - WebSocket, real-time
8. Email Service (PM) - Queue-based processing

**Day 5: Monitoring**
9. Metrics/Monitoring System (AM) - Time-series data
10. Web Crawler (PM) - Distributed crawling

**Day 6: Storage & Media**
11. File Storage (AM) - Object storage patterns
12. Image Hosting (PM) - CDN, compression

**Day 7: Review**
- Common patterns across problems
- Practice whiteboard sessions (45 min each)

### Week 8: Intermediate System Design (12 problems)

**Day 1: E-commerce**
1. Food Delivery (AM) - Real-time matching, routing
2. Payment System (PM) - Transactions, idempotency

**Day 2: Ride-Sharing**
3. Ride-Sharing Service (AM) - Geospatial indexing
4. Location Services (PM) - QuadTree, Geohash

**Day 3: Social Networks**
5. Instagram (AM) - Media storage, feed generation
6. Twitter (PM) - Timeline, hashtags, trending

**Day 4: Collaboration**
7. Google Docs (AM) - Operational Transform, CRDT
8. Dropbox (PM) - File sync, deduplication

**Day 5: Booking**
9. Hotel Booking (AM) - Inventory management
10. Ticket Booking (PM) - Concurrency, overbooking

**Day 6: Analytics**
11. Analytics Platform (AM) - Data pipeline
12. A/B Testing (PM) - Statistical analysis

**Day 7: Review & Integration**
- Practice 3 full problems (45 min each)
- Focus on scaling discussions

### System Design Interview Framework

**Use this for EVERY system design problem (45 min total):**

**1. Clarify Requirements (5 min)**
- Functional requirements (what features?)
- Non-functional requirements (scale, performance, availability)
- Constraints (read-heavy? write-heavy?)

Questions to ask:
- "How many users?"
- "Read/write ratio?"
- "Latency requirements?"
- "Data consistency needs?"

**2. High-Level Design (10 min)**
- Draw basic architecture
- Identify major components (API, DB, cache, queue)
- Data flow diagram

**3. Deep Dive (20 min)**
Choose 2-3 areas based on interviewer interest:
- Database schema
- API design
- Scaling strategy
- Caching strategy
- Data partitioning

**4. Bottlenecks & Trade-offs (10 min)**
- Identify bottlenecks
- Discuss trade-offs
- How to scale to 10x, 100x users

### Weekly Checklist
- [ ] Complete 24 system design problems (12 beginner + 12 intermediate)
- [ ] Can draw high-level architecture in < 5 min
- [ ] Understand when to use each building block
- [ ] Practiced 5 problems using the interview framework
- [ ] Can discuss trade-offs for CAP theorem, consistency patterns

---

## Week 9-10: System Design Advanced

### Goal
Master complex, large-scale distributed systems.

### Week 9: Advanced System Design (12 problems)

**Day 1: Video Streaming**
1. **YouTube/Netflix** (Full day) - 3 hours
   - CDN, adaptive bitrate, recommendations
   - Most comprehensive advanced problem

**Day 2: Search Engines**
2. **Google Search** (Full day) - 3 hours
   - Web crawling, indexing, ranking
   - Distributed search infrastructure

**Day 3: Data-Intensive**
3. **Distributed Cache** (AM) - 2 hours
   - Consistent hashing, replication
4. **Key-Value Store** (PM) - 2 hours
   - Dynamo-style, eventual consistency

**Day 4: Real-time Systems**
5. **Stock Exchange** (AM) - 2 hours
   - Ultra-low latency, FIFO matching
6. **Real-time Gaming** (PM) - 2 hours
   - State synchronization, cheating prevention

**Day 5: Ad Tech**
7. **Ad Click Aggregator** (AM) - 2 hours
   - Stream processing, Lambda architecture
8. **Ad Serving** (PM) - 2 hours
   - Real-time bidding, targeting

**Day 6: Infrastructure**
9. **Distributed Task Scheduler** (AM) - 2 hours
10. **API Rate Limiter** (PM) - 2 hours

**Day 7: Practice**
- Pick 2 problems and practice full interview (45 min each)

### Week 10: Specialized Topics

**Day 1-2: Distributed Systems Patterns**
- CAP theorem deep dive
- Consistency models (strong, eventual, causal)
- Consensus algorithms (Paxos, Raft) - overview
- Replication strategies (master-slave, multi-master)

**Day 3-4: Data Engineering**
- Lambda architecture
- Kappa architecture
- Stream processing (Kafka Streams, Flink)
- Data lake vs data warehouse

**Day 5-6: Advanced Scaling**
- Microservices patterns
- Service mesh (Istio, Linkerd)
- Circuit breakers, bulkheads
- Saga pattern for distributed transactions

**Day 7: Integration Day**
- Design a system combining multiple advanced concepts
- Example: "Design a real-time multiplayer game platform"

### Study Method for Advanced Problems

**For Each Problem (2-3 hours):**

1. **Requirements (15 min)**
   - Read problem statement
   - Note scale requirements
   - Identify critical features

2. **Architecture Study (45 min)**
   - Study provided architecture
   - Understand each component's role
   - Draw diagram from memory

3. **Deep Dives (60 min)**
   - Pick 2-3 interesting aspects
   - Understand algorithms/data structures
   - Research real-world implementations

4. **Practice (30 min)**
   - Explain solution out loud
   - Practice whiteboard drawing
   - Answer follow-up questions

### Weekly Checklist
- [ ] Complete 12 advanced system design problems
- [ ] Understand distributed systems concepts (CAP, consistency)
- [ ] Can design scalable systems for millions of users
- [ ] Practiced 4 advanced problems using interview framework
- [ ] Comfortable discussing trade-offs at scale

---

## Week 11-12: Practice & Mock Interviews

### Goal
Solidify knowledge through practice and simulate real interview conditions.

### Week 11: Intensive Practice

**Daily Schedule (2-3 hours/day):**

**Monday: OOD Day**
- Morning: 1 intermediate OOD problem (40 min)
- Afternoon: 1 advanced OOD problem (60 min)
- Evening: Review and notes (20 min)

**Tuesday: System Design Day**
- Morning: 1 beginner system design (45 min)
- Afternoon: 1 intermediate system design (45 min)
- Evening: Review building blocks (30 min)

**Wednesday: Advanced Day**
- Morning: 1 advanced OOD (Netflix/Uber/Amazon) (60 min)
- Afternoon: 1 advanced system design (YouTube/Google) (60 min)

**Thursday: Pattern Review**
- Review all design patterns
- Create pattern selection flowchart
- Practice pattern identification

**Friday: Building Blocks**
- Review all 10 building blocks
- Practice drawing architecture diagrams
- Focus on load balancer, cache, message queue

**Saturday: Mock Interview Day 1**
- OOD problem (45 min) - time yourself strictly
- System design problem (45 min) - draw on whiteboard
- Self-review: What went well? What to improve?

**Sunday: Mock Interview Day 2**
- Different OOD problem (45 min)
- Different system design problem (45 min)
- Record yourself or practice with friend

### Week 12: Final Preparation

**Strategy:** Focus on weak areas, polish strong areas.

**Day 1-2: Weak Area Deep Dive**
- Identify your 3 weakest topics
- Spend 2 hours on each
- Redo problems you struggled with

**Day 3-4: Strong Area Polish**
- Pick your strongest problems
- Practice explaining clearly and concisely
- Aim for confident, smooth delivery

**Day 5: Common Interview Problems**
Practice these MUST-KNOW problems:
- OOD: Parking Lot, LRU Cache, Elevator System
- System Design: URL Shortener, Twitter, Netflix

**Day 6: Final Mock Interview**
- Complete 2 OOD problems (90 min)
- Complete 2 system design problems (90 min)
- Simulate real interview pressure

**Day 7: Rest & Review**
- Light review of cheat sheets
- Read your notes
- Relax and stay confident

### Mock Interview Structure

**OOD Mock Interview (45 min):**
1. Clarify requirements (5 min)
2. List core classes (5 min)
3. Draw class diagram (10 min)
4. Discuss design patterns (5 min)
5. Deep dive on 1-2 classes (15 min)
6. Follow-up questions (5 min)

**System Design Mock Interview (45 min):**
1. Clarify requirements (5 min)
2. High-level architecture (10 min)
3. Component deep dive (15 min)
4. Scaling discussion (10 min)
5. Trade-offs and alternatives (5 min)

### Practice Resources

**Find Practice Partners:**
- Pramp (pramp.com) - free mock interviews
- Interviewing.io - anonymous technical interviews
- LeetCode mock interview feature
- Friends/colleagues preparing for interviews

**Self-Practice:**
- Record yourself explaining solutions
- Use timer strictly
- Practice on whiteboard (not computer)
- Verbalize your thought process

### Weekly Checklist
- [ ] Completed 10+ mock interviews (self or with partner)
- [ ] Practiced 20+ problems total (mix of OOD and system design)
- [ ] Identified and improved weak areas
- [ ] Can confidently complete OOD problem in 40 min
- [ ] Can confidently complete system design in 45 min
- [ ] Created personal cheat sheets for quick review
- [ ] Ready for real interviews!

---

## Daily Study Routine

### Recommended Schedule

**Weekdays (2 hours/day):**

**Morning Session (60 min)** - Before work/school
- Review previous day's topic (10 min)
- New topic study (50 min)

**Evening Session (60 min)** - After work/school
- Practice problems (40 min)
- Create notes/summaries (20 min)

**Weekends (4-5 hours/day):**

**Morning (2.5 hours)**
- Deep dive on complex topic (2 hours)
- Break (15 min)
- Practice problem (30 min)

**Afternoon (2 hours)**
- Mock interview (45 min)
- Review and improve (45 min)
- Weekly summary (30 min)

### Study Techniques

**Active Learning:**
1. **Don't just read - draw**
   - Sketch class diagrams
   - Draw architecture diagrams
   - Visualize data flow

2. **Explain out loud**
   - Pretend you're in an interview
   - Talk through your thought process
   - Practice answering "why" questions

3. **Write summaries**
   - One-page summary per topic
   - Key points in your own words
   - Common pitfalls to avoid

4. **Spaced repetition**
   - Review topics after 1 day, 3 days, 7 days
   - Use flashcards for design patterns
   - Revisit challenging problems

**Focus Strategies:**
- Use Pomodoro technique (25 min focus, 5 min break)
- Remove distractions (phone, social media)
- Study in same place/time (build habit)
- Take notes by hand (better retention)

---

## Progress Tracking

### Weekly Review Template

**Copy this template and fill out every Sunday:**

```
## Week [X] Review - [Date]

### Topics Covered
- [ ] Topic 1
- [ ] Topic 2
- [ ] Topic 3

### Problems Completed
1. [Problem Name] - [Time Taken] - [Difficulty] - [Confidence: 1-5]
2. [Problem Name] - [Time Taken] - [Difficulty] - [Confidence: 1-5]
3. ...

### What Went Well
-

### What Needs Improvement
-

### Key Learnings This Week
1.
2.
3.

### Next Week Focus
-
```

### Progress Metrics

Track these metrics:

**Quantitative:**
- Problems completed per week
- Time per problem (aim to decrease)
- Mock interviews completed
- Concepts mastered

**Qualitative:**
- Confidence level (1-5) per topic
- Ability to explain clearly
- Speed of diagram drawing
- Handling follow-up questions

### Milestone Checkpoints

**End of Week 2:**
- [ ] Can name and explain 10+ design patterns
- [ ] Understand all building blocks conceptually
- [ ] Created personal cheat sheet

**End of Week 4:**
- [ ] Completed all beginner/intermediate OOD
- [ ] Can draw class diagrams quickly
- [ ] Comfortable with Strategy, State, Observer patterns

**End of Week 6:**
- [ ] Completed advanced OOD problems
- [ ] Deep understanding of building blocks
- [ ] Can explain system components

**End of Week 8:**
- [ ] Completed beginner/intermediate system design
- [ ] Can use interview framework effectively
- [ ] Understand scaling strategies

**End of Week 10:**
- [ ] Completed advanced system design
- [ ] Comfortable with distributed systems concepts
- [ ] Can handle complex scaling discussions

**End of Week 12:**
- [ ] 10+ mock interviews completed
- [ ] Confident in both OOD and system design
- [ ] Ready for real interviews!

---

## Interview Tips

### Before the Interview

**1-2 Days Before:**
- Review your cheat sheets
- Do 1 practice problem (don't overdo it)
- Get good sleep
- Prepare questions to ask interviewer

**Day Of:**
- Light review only (no new learning)
- Have paper and pen ready (for virtual)
- Test video/audio setup (virtual interviews)
- Stay hydrated, eat well

### During the Interview

**First 5 Minutes (Critical):**
1. **Listen carefully** to the problem
2. **Ask clarifying questions**
   - "How many users?"
   - "What's the scale?"
   - "Any specific constraints?"
3. **State assumptions explicitly**
   - "I'm assuming..."
   - "For this design, let's say..."
4. **Confirm understanding** before starting

**Design Phase:**
- **Think out loud** - show your thought process
- **Start simple** - basic design first, then enhance
- **Draw clearly** - neat diagrams matter
- **Explain as you draw** - narrate your decisions

**Deep Dive Phase:**
- **Ask what interests them** - "Would you like me to focus on X or Y?"
- **Discuss trade-offs** - "Approach A is better for X, but B is better for Y"
- **Acknowledge limitations** - "This design assumes X, but could be adapted for Y"

**Red Flags to Avoid:**
- ❌ Jumping straight to solution without clarifying
- ❌ Staying silent for long periods
- ❌ Being defensive about your design
- ❌ Ignoring interviewer hints
- ❌ Not discussing trade-offs

**Green Flags to Demonstrate:**
- ✅ Structured thinking (use framework)
- ✅ Clear communication
- ✅ Discussing alternatives
- ✅ Considering edge cases
- ✅ Open to feedback

### After the Interview

**Immediately After:**
- Write down what you remember
- Note questions you struggled with
- Identify areas to improve

**Follow-up:**
- Send thank you email (within 24 hours)
- Mention specific discussion points
- Reiterate your interest

### Common Interview Problems by Company

**FAANG Favorites:**

**Meta/Facebook:**
- News Feed Design
- Instagram
- WhatsApp/Messenger
- Notification Service

**Amazon:**
- Shopping Cart (OOD)
- Amazon E-commerce Platform (OOD)
- Product Catalog
- Inventory Management

**Apple:**
- Music Streaming Service
- Photo Library
- iCloud Storage
- Notification System

**Netflix:**
- Video Streaming Platform (OOD)
- Content Recommendation
- CDN Design

**Google:**
- URL Shortener
- Google Docs
- Search Engine
- YouTube
- Google Maps

**Microsoft:**
- Elevator System (OOD)
- Calendar System
- OneDrive/Dropbox
- Outlook/Email Service

**Uber/Lyft:**
- Ride-Sharing App (OOD) - extremely common
- Location Services
- Real-time Tracking
- Dynamic Pricing

**Startups:**
- Often ask about their specific domain
- Focus on scalability and quick iteration
- May emphasize trade-offs and practical constraints

---

## Quick Reference Cheat Sheets

### Top 10 Design Patterns for Interviews

1. **Strategy** - Interchangeable algorithms (pricing, discounts, AI difficulty)
2. **Observer** - Event notification (UI updates, pub-sub)
3. **Factory** - Object creation (different order types, vehicles)
4. **Singleton** - Single instance (config, logger, DB connection)
5. **State** - State-dependent behavior (order states, game states)
6. **Decorator** - Add features dynamically (pizza toppings, gift wrap)
7. **Command** - Encapsulate actions (undo/redo, remote control)
8. **Adapter** - Interface compatibility (payment gateways)
9. **Facade** - Simplify complexity (checkout process)
10. **Template Method** - Algorithm skeleton (game player types)

### Top 10 Building Blocks

1. **Load Balancer** - Distribute traffic (L4/L7, round-robin)
2. **Cache** - Fast data access (Redis, LRU eviction)
3. **Message Queue** - Async communication (Kafka, RabbitMQ)
4. **CDN** - Content delivery (edge caching)
5. **Database Sharding** - Horizontal partitioning
6. **Rate Limiter** - API protection (token bucket)
7. **API Gateway** - Request routing (auth, rate limiting)
8. **Service Discovery** - Dynamic service location (Consul)
9. **Consistent Hashing** - Data distribution
10. **Merkle Tree** - Data integrity verification

### System Design Decision Tree

**Need to serve millions of users?**
- → Load Balancer + Multiple servers

**Database becoming slow?**
- Read-heavy → Read replicas + Caching
- Write-heavy → Sharding + Message queues

**API getting hammered?**
- → Rate Limiter + API Gateway

**Users worldwide?**
- → CDN for static content
- → Regional data centers

**Services need to communicate?**
- Sync → API calls with circuit breaker
- Async → Message queues

**Need high availability?**
- → Redundancy (multi-AZ, multi-region)
- → Circuit breakers, fallbacks

### OOD Problem Checklist

**Every OOD Problem Should Have:**
1. [ ] Core classes identified
2. [ ] Relationships defined (composition, inheritance)
3. [ ] At least 2-3 design patterns
4. [ ] Clear responsibilities (SRP)
5. [ ] Extensibility considered
6. [ ] Edge cases handled

**Common OOD Patterns:**
- Games → State pattern (game states)
- Pricing → Strategy pattern (different pricing)
- Events → Observer pattern (notifications)
- Complex creation → Factory/Builder pattern
- Undo/Redo → Command pattern

---

## Additional Resources

### Books (Optional but Recommended)
1. **"Designing Data-Intensive Applications"** by Martin Kleppmann
   - Best for: Deep understanding of distributed systems
   - Read: Chapters 1-3, 5-9 (core concepts)

2. **"System Design Interview"** by Alex Xu (Volume 1 & 2)
   - Best for: Interview-specific preparation
   - Read: All chapters, great diagrams

3. **"Head First Design Patterns"** by Freeman & Robson
   - Best for: Understanding design patterns intuitively
   - Read: All patterns with examples

### Online Resources
- **System Design Primer** (GitHub) - Comprehensive free resource
- **Grokking the System Design Interview** (Educative.io) - Structured course
- **High Scalability Blog** - Real-world architectures
- **Engineering blogs**: Netflix, Uber, Airbnb, Discord

### YouTube Channels
- **System Design Interview** - Problem walkthroughs
- **Gaurav Sen** - Clear system design explanations
- **Tech Dummies** - Simplified concepts
- **Success in Tech** - Interview tips

---

## Motivation & Mindset

### Remember:

**It's a Journey:**
- Don't try to memorize everything
- Understanding > memorization
- Quality > quantity

**Interview is a Conversation:**
- Not about perfect solution
- Show your thinking process
- Ask questions, discuss trade-offs
- Be open to feedback

**Practice Makes Progress:**
- First mock interview will feel awkward - that's normal
- You'll improve dramatically with each practice
- Confidence comes from repetition

**You've Got This:**
- You have comprehensive materials (all 13 files!)
- You have a structured plan
- You have 8-12 weeks to master this
- Thousands have done this before you

### When You Feel Overwhelmed:

1. **Take a break** - Walk, exercise, rest
2. **Revisit basics** - Review fundamentals
3. **Do an easy problem** - Build confidence
4. **Talk to others** - Study groups help
5. **Remember why** - Keep end goal in mind

---

## Final Thoughts

This study plan is **comprehensive but flexible**. Adjust based on:
- Your current knowledge level
- Time until interviews
- Your learning pace
- Specific companies you're targeting

**Most Important:**
- **Consistency** beats intensity
- **Understanding** beats memorization
- **Practice** beats passive reading
- **Communication** is as important as technical knowledge

**You have all the materials you need in this repository. Now it's about putting in the focused practice. Good luck! 🚀**

---

## Quick Start Checklist

**Today (Day 1):**
- [ ] Read this entire study plan
- [ ] Set up your study space
- [ ] Create a calendar with study schedule
- [ ] Start Week 1, Day 1 (Creational Patterns)

**This Week:**
- [ ] Complete first 5 design patterns
- [ ] Create your first cheat sheet
- [ ] Establish daily study routine

**This Month:**
- [ ] Complete all design patterns and building blocks
- [ ] Complete beginner OOD problems
- [ ] Do your first mock interview

**Ready to start? Go to Week 1, Day 1 and begin with the Singleton pattern! 💪**
