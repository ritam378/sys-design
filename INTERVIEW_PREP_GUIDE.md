# System Design & OOD Interview Preparation Guide

**Last Updated:** January 8, 2026
**Purpose:** Comprehensive interview preparation guide with primary focus on HLD (High-Level Design) and critical LLD (Low-Level Design) problems

---

## Table of Contents

1. [Overview](#overview)
2. [The RESHADED Framework](#the-reshaded-framework)
3. [HLD Interview Mastery](#hld-interview-mastery)
4. [LLD Interview Essentials](#lld-interview-essentials)
5. [Interview Strategy & Communication](#interview-strategy--communication)
6. [8-Week Study Plan](#8-week-study-plan)
7. [Common Pitfalls & How to Avoid Them](#common-pitfalls--how-to-avoid-them)
8. [Company-Specific Preparation](#company-specific-preparation)
9. [Self-Assessment & Practice](#self-assessment--practice)

---

## Overview

### What This Guide Covers

This guide is designed to help you prepare for system design and object-oriented design interviews at top tech companies (FAANG and beyond). It focuses on:

- **Primary Focus (70%):** High-Level Design (HLD) - Distributed systems, scalability, architecture
- **Secondary Focus (30%):** Low-Level Design (LLD) - Object-oriented design, design patterns, class structures

### Interview Format

**Typical Interview Breakdown:**
- **HLD Round:** 45-60 minutes, design a large-scale distributed system
- **LLD Round:** 45-60 minutes, design classes and relationships for a specific feature
- **Coding Round:** May include implementing small parts of your design

**What Interviewers Look For:**
1. **Communication** (30%): Can you explain your thoughts clearly?
2. **Problem-solving** (25%): Do you ask clarifying questions? Handle trade-offs?
3. **Technical depth** (25%): Do you understand the technologies you propose?
4. **Scalability thinking** (20%): Can you design for millions/billions of users?

---

## The RESHADED Framework

Use this framework for every HLD interview. Practice until it becomes second nature.

### R - Requirements (5 minutes)

**What to do:**
- Clarify functional requirements (core features)
- Clarify non-functional requirements (scale, performance, availability)
- Define scope (what's in, what's out)

**Example Questions:**
- "How many daily active users do we expect?"
- "What's the read-to-write ratio?"
- "Do we need real-time updates or is eventual consistency okay?"
- "What's more important: availability or consistency?"
- "Are there any specific features we should prioritize?"

**Red Flags to Avoid:**
❌ Jumping straight into design without asking questions
❌ Making assumptions without stating them
❌ Trying to build everything (feature creep)

### E - Estimation (5 minutes)

**What to calculate:**
- **Traffic estimates:** QPS (queries per second), DAU (daily active users)
- **Storage estimates:** Data size per user, total storage over time
- **Bandwidth estimates:** Ingress and egress bandwidth
- **Memory estimates:** For caching, in-memory processing

**Back-of-Envelope Math:**

```
Daily Active Users (DAU): 100M
Average requests per user per day: 50
Total requests per day: 100M * 50 = 5B
Requests per second (QPS): 5B / 86400 ≈ 58K QPS
Peak QPS (3x average): 174K QPS

Storage per user: 1KB
Total users: 500M
Total storage: 500M * 1KB = 500GB
Growth per year (20%): 100GB/year
```

**Practice These Numbers:**
- 1 million = 10^6
- 1 billion = 10^9
- 1 day = 86,400 seconds (~100K for quick math)
- 1 month = 2.5M seconds
- Peak traffic = 2-3x average

### S - Storage (Part of detailed design)

**What to decide:**
- Database choice (SQL vs NoSQL)
- Data model / schema
- Sharding strategy
- Replication approach

**Decision Framework:**

| Use SQL When | Use NoSQL When |
|--------------|----------------|
| Need ACID transactions | Need horizontal scalability |
| Complex queries with JOINs | Simple key-value access |
| Structured, relational data | Unstructured or semi-structured data |
| Data integrity is critical | High write throughput needed |

### H - High-level Design (10 minutes)

**What to draw:**
- Main components (Load Balancer, API Gateway, App Servers, Databases, Cache, CDN)
- Data flow (arrows showing request/response paths)
- External services (if any)

**Standard Components:**

```
Client → CDN (for static content)
       ↓
Client → DNS → Load Balancer → API Gateway → App Servers → Cache (Redis)
                                                          → Database (Primary)
                                                          → Database (Replica)
                                                          → Message Queue
                                                          → Background Workers
```

**Pro Tip:** Draw this on the left side of the whiteboard, leave right side for deep dives.

### A - API Design (5-10 minutes)

**What to define:**
- Key API endpoints
- Request/response formats
- Authentication approach

**RESTful API Template:**

```
POST   /api/v1/users                 # Create user
GET    /api/v1/users/{userId}        # Get user
PUT    /api/v1/users/{userId}        # Update user
DELETE /api/v1/users/{userId}        # Delete user

POST   /api/v1/posts                 # Create post
GET    /api/v1/posts/{postId}        # Get post
GET    /api/v1/feed?userId={id}      # Get news feed
```

**Include:**
- HTTP methods
- Endpoint paths
- Key parameters
- Expected responses

### D - Deep-dive (15-20 minutes)

**What to cover:**
- 2-3 critical components in detail
- Data structures and algorithms used
- How to handle bottlenecks
- Trade-offs for key decisions

**Common Deep-dive Topics:**
- **Caching strategy:** What to cache? When to invalidate?
- **Database schema:** Show tables, indexes, relationships
- **Sharding logic:** How to partition data?
- **Consistency:** How to handle distributed transactions?
- **Real-time updates:** WebSockets vs Long Polling?

**Example Deep-dive: Newsfeed Ranking**

```python
# Fan-out on write vs fan-out on read
def generate_feed(user_id):
    # Pull model (fan-out on read)
    following = get_following(user_id)  # Get list of users I follow
    posts = []
    for followee_id in following[:100]:  # Limit to top 100
        recent_posts = get_recent_posts(followee_id, limit=10)
        posts.extend(recent_posts)

    # Rank by algorithm (engagement, recency, etc.)
    ranked_posts = rank_posts(posts, user_id)
    return ranked_posts[:20]

# Trade-off: Fan-out on read is slower but handles celebrity users better
```

### E - Extend & Edge Cases (5 minutes)

**What to discuss:**
- How would you scale to 10x users?
- What if a celebrity user has 100M followers?
- How to handle network partitions?
- What about data consistency across regions?
- Security considerations?

**Examples:**
- "For celebrity users, we could use a separate fan-out strategy"
- "To scale 10x, we'd need to shard the database and use a CDN for static content"
- "For cross-region consistency, we could use multi-master replication with conflict resolution"

### D - Discuss Trade-offs (Throughout)

**What to explain:**
- Why you chose one approach over another
- What are the downsides of your choices?
- What would you do differently at different scales?

**Common Trade-offs:**

| Decision | Option A | Option B | When to Use Each |
|----------|----------|----------|------------------|
| **Consistency** | Strong (ACID) | Eventual (BASE) | Banking vs Social Media |
| **Storage** | SQL (relational) | NoSQL (document) | Structured vs Unstructured |
| **Caching** | Write-through | Write-behind | Consistency vs Performance |
| **Communication** | Sync (REST) | Async (Message Queue) | Real-time vs Eventual |
| **Data** | Normalized | Denormalized | Storage vs Read Speed |

---

## HLD Interview Mastery

### Top 10 Must-Know Problems (90% Interview Frequency)

These problems cover all major system design concepts. Master these first.

#### 1. URL Shortener (Beginner)

**Why it's asked:** Tests hashing, database design, scalability basics
**Key concepts:** Base62 encoding, distributed ID generation, caching
**Time to master:** 2-3 hours

**Core Challenge:**
- Generate short unique IDs for long URLs
- Handle billions of URLs
- High read-to-write ratio (100:1)

**Key Design Points:**
```
Encoding: base62 (a-z, A-Z, 0-9) = 62^7 = 3.5 trillion URLs
Database: NoSQL (key-value store) - fast lookups
Caching: Cache popular URLs (80/20 rule)
```

**Must Discuss:**
- MD5 hash collision handling vs auto-incrementing IDs
- Custom short URL support
- Analytics tracking
- Expiration policy

#### 2. Rate Limiter (Intermediate)

**Why it's asked:** Critical for API design, tests algorithmic thinking
**Key concepts:** Token bucket, sliding window, distributed rate limiting
**Time to master:** 3-4 hours

**Algorithms to Know:**

```python
# Token Bucket (most common)
class TokenBucket:
    def __init__(self, capacity, refill_rate):
        self.capacity = capacity
        self.tokens = capacity
        self.refill_rate = refill_rate  # tokens per second
        self.last_refill = time.time()

    def allow_request(self):
        self._refill()
        if self.tokens >= 1:
            self.tokens -= 1
            return True
        return False

    def _refill(self):
        now = time.time()
        tokens_to_add = (now - self.last_refill) * self.refill_rate
        self.tokens = min(self.capacity, self.tokens + tokens_to_add)
        self.last_refill = now
```

**Must Discuss:**
- Token bucket vs leaky bucket vs fixed window vs sliding window
- Distributed rate limiting with Redis
- Rate limiting at different layers (user, IP, API key)
- Handling edge cases (DDoS, burst traffic)

#### 3. News Feed (Intermediate)

**Why it's asked:** Tests fan-out patterns, ranking algorithms
**Key concepts:** Fan-out on write vs read, timeline caching
**Time to master:** 4-5 hours

**Two Approaches:**

```
Fan-out on Write (Push model):
- When user posts → write to all followers' feeds
- Pros: Fast reads
- Cons: Slow writes, celebrity problem

Fan-out on Read (Pull model):
- When user requests feed → gather from all followees
- Pros: Handles celebrities well
- Cons: Slow reads

Hybrid (Best):
- Fan-out on write for normal users
- Fan-out on read for celebrities (>100K followers)
```

**Must Discuss:**
- Feed ranking algorithm (ML-based vs simple time-based)
- Caching strategy for hot posts
- Pagination and cursor-based scrolling
- Handling deleted posts

#### 4. Notification System (Intermediate)

**Why it's asked:** Tests multi-channel design, message queuing
**Key concepts:** Pub/Sub, fan-out, message prioritization
**Time to master:** 3-4 hours

**Architecture:**

```
Event → API Server → Message Queue → Workers (by channel)
                                   → Email Service
                                   → SMS Service
                                   → Push Notification Service
                                   → In-App Notification
```

**Must Discuss:**
- User preferences (which channels, frequency)
- De-duplication (same event shouldn't spam)
- Priority queue for urgent notifications
- Rate limiting per channel
- Retry logic and dead letter queue

#### 5. Chat System (Intermediate)

**Why it's asked:** Tests real-time communication, WebSockets
**Key concepts:** WebSocket management, message persistence, presence
**Time to master:** 4-5 hours

**Real-time Communication Options:**

| Approach | Pros | Cons | Use Case |
|----------|------|------|----------|
| WebSockets | True bidirectional, low latency | Complex to scale | WhatsApp, Slack |
| Long Polling | Simple, firewall-friendly | Higher latency, server load | Legacy systems |
| Server-Sent Events | One-way push, simple | No bidirectional | Live scores, stock prices |

**Must Discuss:**
- Message persistence (SQL vs NoSQL)
- Read receipts and typing indicators
- Group chat (fan-out to all members)
- Media messages (upload to S3, share link)
- Message encryption (end-to-end)

#### 6. Distributed Cache (Advanced)

**Why it's asked:** Redis/Memcached are everywhere
**Key concepts:** Consistent hashing, eviction policies, replication
**Time to master:** 5-6 hours

**Core Components:**

```python
# LRU Cache Implementation (must code from scratch)
class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}  # key -> node
        self.head = Node(0, 0)  # dummy head
        self.tail = Node(0, 0)  # dummy tail
        self.head.next = self.tail
        self.tail.prev = self.head

    def get(self, key):
        if key in self.cache:
            node = self.cache[key]
            self._remove(node)
            self._add_to_head(node)
            return node.value
        return -1

    def put(self, key, value):
        if key in self.cache:
            self._remove(self.cache[key])
        node = Node(key, value)
        self._add_to_head(node)
        self.cache[key] = node
        if len(self.cache) > self.capacity:
            lru = self.tail.prev
            self._remove(lru)
            del self.cache[lru.key]
```

**Must Discuss:**
- Consistent hashing for sharding
- Cache invalidation strategies
- Write-through vs write-behind
- Cache stampede problem and solutions
- Redis vs Memcached

#### 7. Key-Value Store (Beginner)

**Why it's asked:** Foundation for distributed systems, CAP theorem
**Key concepts:** Consistent hashing, replication, CAP theorem
**Time to master:** 3-4 hours

**CAP Theorem:**

```
Consistency: All nodes see same data at same time
Availability: Every request gets a response
Partition Tolerance: System works despite network failures

You can only have 2 out of 3!

CP systems (Consistency + Partition Tolerance):
- MongoDB, HBase, Redis (in cluster mode)
- Use case: Banking, inventory

AP systems (Availability + Partition Tolerance):
- Cassandra, DynamoDB, CouchDB
- Use case: Social media, analytics

CA systems (Consistency + Availability):
- Traditional RDBMS (no partition tolerance)
- Use case: Single datacenter only
```

**Must Discuss:**
- Vector clocks for conflict resolution
- Quorum reads/writes (W + R > N)
- Merkle trees for replica synchronization
- Gossip protocol for node discovery

#### 8. Web Crawler (Intermediate)

**Why it's asked:** Tests BFS/DFS, politeness, distributed processing
**Key concepts:** URL frontier, politeness, deduplication
**Time to master:** 3-4 hours

**Architecture:**

```
URL Frontier (Priority Queue) → Fetcher Workers → Parser → Storage
                                      ↓
                                 Politeness (rate limit per domain)
                                      ↓
                                 Deduplication (Bloom filter + URL hash)
```

**Must Discuss:**
- URL frontier (priority queue for important pages)
- Politeness policy (respect robots.txt, rate limit)
- Distributed crawling (consistent hashing by domain)
- Handling dynamic content (JavaScript rendering)
- Detecting spam and duplicate content

#### 9. Video Streaming (Advanced)

**Why it's asked:** Tests CDN knowledge, adaptive streaming
**Key concepts:** HLS/DASH, CDN, video encoding
**Time to master:** 5-6 hours

**Video Processing Pipeline:**

```
Upload → Transcoding (multiple resolutions) → Storage (S3) → CDN → Client
         |
         └─> 1080p, 720p, 480p, 360p (adaptive bitrate)
```

**Must Discuss:**
- Adaptive Bitrate Streaming (HLS, DASH)
- CDN for edge caching
- Video encoding formats (H.264, H.265)
- Handling live streaming vs VOD
- DRM and content protection
- Buffer management on client

#### 10. Search Autocomplete (Intermediate)

**Why it's asked:** Tests Trie data structure, caching
**Key concepts:** Trie, prefix search, ranking
**Time to master:** 2-3 hours

**Data Structure:**

```python
class TrieNode:
    def __init__(self):
        self.children = {}
        self.is_end = False
        self.frequency = 0  # For ranking popular searches
        self.top_searches = []  # Cache top 10 at each node

class Autocomplete:
    def search(self, prefix):
        node = self._find_prefix_node(prefix)
        if not node:
            return []
        return node.top_searches  # Pre-computed top 10
```

**Must Discuss:**
- Trie vs database for prefix search
- Ranking by popularity/recency
- Handling typos (fuzzy search)
- Caching at different levels
- Personalized suggestions

---

### Interview Frameworks for HLD

#### 1. The 45-Minute Breakdown

```
Minutes 0-5:   Requirements clarification
Minutes 5-10:  Back-of-envelope estimation
Minutes 10-15: High-level design (draw boxes)
Minutes 15-35: Deep dive (2-3 components)
Minutes 35-40: Bottlenecks and scaling
Minutes 40-45: Q&A and edge cases
```

**Pro Tips:**
- Always ask "Is this the right level of detail?" around minute 20
- If stuck, say "Let me think out loud..." and verbalize your thought process
- Draw first, explain second (visual communication is key)

#### 2. The "Levels of Scale" Approach

Always discuss your design at different scales:

```
Level 1: Single server (1-1000 users)
→ Simple monolith, single database

Level 2: Separated tiers (1K-100K users)
→ Web tier, app tier, database tier
→ Add load balancer, caching

Level 3: Horizontal scaling (100K-1M users)
→ Multiple app servers
→ Database replication (master-slave)
→ CDN for static content

Level 4: Microservices (1M-10M users)
→ Break into services
→ Message queues for async
→ Distributed cache

Level 5: Global scale (10M+ users)
→ Multi-region deployment
→ Sharded databases
→ Geo-distributed CDN
```

**In the interview:** Start at Level 3-4, then discuss how to scale to Level 5.

#### 3. The "Trade-offs" Checklist

For every major decision, be ready to explain trade-offs:

**Database Choice:**
```
SQL:
✅ ACID transactions, complex queries, data integrity
❌ Harder to scale horizontally, schema rigidity

NoSQL:
✅ Horizontal scalability, flexible schema, high throughput
❌ No ACID (usually), limited query capabilities
```

**Caching Strategy:**
```
Cache-Aside (lazy loading):
✅ Only cache what's needed
❌ Initial cache miss penalty

Write-Through:
✅ Always in sync
❌ Write latency, cache all data

Write-Behind:
✅ Fast writes, batch updates
❌ Risk of data loss, eventual consistency
```

**Communication Pattern:**
```
Synchronous (REST, gRPC):
✅ Simple, immediate response
❌ Tight coupling, cascade failures

Asynchronous (Message Queue):
✅ Decoupling, fault tolerance
❌ Complexity, eventual consistency
```

---

## LLD Interview Essentials

### Top 8 Must-Know LLD Problems (80% Interview Frequency)

#### 1. LRU Cache

**Why it's critical:** Tests data structures (HashMap + DoublyLinkedList)
**Difficulty:** Medium
**Time to master:** 2-3 hours
**YOU MUST CODE THIS FROM MEMORY**

**Key Requirements:**
- `get(key)`: Return value if exists, else -1. Move to front.
- `put(key, value)`: Insert/update. Evict LRU if capacity exceeded.
- Both operations must be O(1)

**Data Structures:**
```
HashMap: key → DoublyLinkedNode (O(1) lookup)
DoublyLinkedList: maintains LRU order (O(1) add/remove)
```

**Full Implementation:**

```python
class Node:
    def __init__(self, key, value):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None

class LRUCache:
    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}  # key -> Node
        # Dummy head and tail for easier add/remove
        self.head = Node(0, 0)
        self.tail = Node(0, 0)
        self.head.next = self.tail
        self.tail.prev = self.head

    def get(self, key):
        if key in self.cache:
            node = self.cache[key]
            self._remove(node)
            self._add_to_head(node)
            return node.value
        return -1

    def put(self, key, value):
        if key in self.cache:
            # Update existing key
            self._remove(self.cache[key])

        # Create new node
        node = Node(key, value)
        self._add_to_head(node)
        self.cache[key] = node

        # Evict LRU if over capacity
        if len(self.cache) > self.capacity:
            lru = self.tail.prev
            self._remove(lru)
            del self.cache[lru.key]

    def _remove(self, node):
        """Remove node from linked list"""
        prev_node = node.prev
        next_node = node.next
        prev_node.next = next_node
        next_node.prev = prev_node

    def _add_to_head(self, node):
        """Add node right after head (most recently used)"""
        node.prev = self.head
        node.next = self.head.next
        self.head.next.prev = node
        self.head.next = node
```

**Follow-up Questions:**
- "How would you make this thread-safe?" → Add locks
- "How to implement LFU (Least Frequently Used)?" → Add frequency counter
- "How to handle TTL (time-to-live)?" → Add timestamp to Node

#### 2. Parking Lot

**Why it's critical:** Tests OOP principles, design patterns
**Difficulty:** Medium
**Time to master:** 3-4 hours

**Key Requirements:**
- Support multiple vehicle types (Car, Truck, Motorcycle)
- Different spot sizes (Compact, Large, Handicapped)
- Track availability
- Calculate fees

**Design Patterns Used:**
- **Factory Pattern:** Create vehicles
- **Strategy Pattern:** Different parking strategies
- **Singleton Pattern:** ParkingLot instance

**Class Diagram:**

```python
from enum import Enum
from datetime import datetime
from abc import ABC, abstractmethod

class VehicleType(Enum):
    CAR = 1
    TRUCK = 2
    MOTORCYCLE = 3

class SpotSize(Enum):
    COMPACT = 1
    LARGE = 2
    HANDICAPPED = 3

class Vehicle(ABC):
    def __init__(self, license_plate):
        self.license_plate = license_plate
        self.vehicle_type = None

    @abstractmethod
    def can_fit_in_spot(self, spot):
        pass

class Car(Vehicle):
    def __init__(self, license_plate):
        super().__init__(license_plate)
        self.vehicle_type = VehicleType.CAR

    def can_fit_in_spot(self, spot):
        return spot.size in [SpotSize.COMPACT, SpotSize.LARGE, SpotSize.HANDICAPPED]

class Truck(Vehicle):
    def __init__(self, license_plate):
        super().__init__(license_plate)
        self.vehicle_type = VehicleType.TRUCK

    def can_fit_in_spot(self, spot):
        return spot.size == SpotSize.LARGE

class Motorcycle(Vehicle):
    def __init__(self, license_plate):
        super().__init__(license_plate)
        self.vehicle_type = VehicleType.MOTORCYCLE

    def can_fit_in_spot(self, spot):
        return True  # Can fit anywhere

class ParkingSpot:
    def __init__(self, spot_id, size):
        self.spot_id = spot_id
        self.size = size
        self.vehicle = None
        self.is_available = True

    def park_vehicle(self, vehicle):
        if not self.is_available:
            return False
        if not vehicle.can_fit_in_spot(self):
            return False

        self.vehicle = vehicle
        self.is_available = False
        return True

    def remove_vehicle(self):
        vehicle = self.vehicle
        self.vehicle = None
        self.is_available = True
        return vehicle

class Ticket:
    def __init__(self, vehicle, spot, entry_time):
        self.vehicle = vehicle
        self.spot = spot
        self.entry_time = entry_time
        self.exit_time = None

    def calculate_fee(self):
        if not self.exit_time:
            return 0
        hours = (self.exit_time - self.entry_time).total_seconds() / 3600
        rate = 5  # $5 per hour
        return hours * rate

class ParkingLot:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self):
        if not hasattr(self, 'spots'):
            self.spots = {}  # spot_id -> ParkingSpot
            self.tickets = {}  # license_plate -> Ticket

    def add_spot(self, spot):
        self.spots[spot.spot_id] = spot

    def park_vehicle(self, vehicle):
        # Find available spot
        for spot in self.spots.values():
            if spot.is_available and vehicle.can_fit_in_spot(spot):
                if spot.park_vehicle(vehicle):
                    ticket = Ticket(vehicle, spot, datetime.now())
                    self.tickets[vehicle.license_plate] = ticket
                    return ticket
        return None  # No spot available

    def remove_vehicle(self, license_plate):
        if license_plate not in self.tickets:
            return None

        ticket = self.tickets[license_plate]
        ticket.exit_time = datetime.now()
        ticket.spot.remove_vehicle()
        del self.tickets[license_plate]
        return ticket

    def get_available_spots(self, size=None):
        available = [s for s in self.spots.values() if s.is_available]
        if size:
            available = [s for s in available if s.size == size]
        return len(available)
```

**Follow-up Questions:**
- "How to handle payment processing?" → Add PaymentProcessor interface
- "How to support multiple floors?" → Add Floor class, organize spots by floor
- "How to reserve spots in advance?" → Add Reservation class with time slots

#### 3. Elevator System

**Why it's critical:** Tests state machines, scheduling algorithms
**Difficulty:** Hard
**Time to master:** 4-5 hours

**Key Concepts:**
- **State Pattern:** Elevator states (Idle, MovingUp, MovingDown)
- **Strategy Pattern:** Different scheduling algorithms (FCFS, SCAN, LOOK)

**Core Classes:**

```python
from enum import Enum
from collections import deque

class Direction(Enum):
    UP = 1
    DOWN = 2
    IDLE = 3

class ElevatorState(Enum):
    IDLE = 1
    MOVING_UP = 2
    MOVING_DOWN = 3

class Request:
    def __init__(self, floor, direction):
        self.floor = floor
        self.direction = direction

class Elevator:
    def __init__(self, elevator_id, num_floors):
        self.elevator_id = elevator_id
        self.current_floor = 1
        self.state = ElevatorState.IDLE
        self.up_requests = set()
        self.down_requests = set()

    def add_request(self, floor, direction):
        if direction == Direction.UP:
            self.up_requests.add(floor)
        else:
            self.down_requests.add(floor)

    def move(self):
        if self.state == ElevatorState.MOVING_UP:
            self.current_floor += 1
            if self.current_floor in self.up_requests:
                self.up_requests.remove(self.current_floor)
                print(f"Elevator {self.elevator_id} stopped at floor {self.current_floor}")

            # Check if should change direction
            if not self.up_requests or self.current_floor == max(self.up_requests):
                if self.down_requests:
                    self.state = ElevatorState.MOVING_DOWN
                else:
                    self.state = ElevatorState.IDLE

        elif self.state == ElevatorState.MOVING_DOWN:
            self.current_floor -= 1
            if self.current_floor in self.down_requests:
                self.down_requests.remove(self.current_floor)
                print(f"Elevator {self.elevator_id} stopped at floor {self.current_floor}")

            # Check if should change direction
            if not self.down_requests or self.current_floor == min(self.down_requests):
                if self.up_requests:
                    self.state = ElevatorState.MOVING_UP
                else:
                    self.state = ElevatorState.IDLE

        else:  # IDLE
            if self.up_requests:
                self.state = ElevatorState.MOVING_UP
            elif self.down_requests:
                self.state = ElevatorState.MOVING_DOWN

class ElevatorController:
    def __init__(self, num_elevators, num_floors):
        self.elevators = [Elevator(i, num_floors) for i in range(num_elevators)]
        self.num_floors = num_floors

    def request_elevator(self, floor, direction):
        # Strategy: Assign to closest elevator moving in same direction
        best_elevator = None
        min_distance = float('inf')

        for elevator in self.elevators:
            if elevator.state == ElevatorState.IDLE:
                distance = abs(elevator.current_floor - floor)
                if distance < min_distance:
                    min_distance = distance
                    best_elevator = elevator
            elif (elevator.state == ElevatorState.MOVING_UP and
                  direction == Direction.UP and
                  elevator.current_floor <= floor):
                distance = floor - elevator.current_floor
                if distance < min_distance:
                    min_distance = distance
                    best_elevator = elevator
            elif (elevator.state == ElevatorState.MOVING_DOWN and
                  direction == Direction.DOWN and
                  elevator.current_floor >= floor):
                distance = elevator.current_floor - floor
                if distance < min_distance:
                    min_distance = distance
                    best_elevator = elevator

        if best_elevator:
            best_elevator.add_request(floor, direction)
            return best_elevator

        # Fallback: assign to first idle elevator
        return self.elevators[0]
```

**Must Discuss:**
- SCAN algorithm (elevator moves in one direction, services all requests)
- Energy optimization (don't move if no requests)
- Priority handling (emergency vs normal requests)

#### 4-8. Other Must-Know LLD Problems

**4. Deck of Cards** - Factory pattern, Shuffle algorithm
**5. Vending Machine** - State pattern, Inventory management
**6. Chess Game** - Command pattern, Move validation
**7. ATM Machine** - State pattern, Chain of Responsibility
**8. Movie Ticket Booking** - Concurrency control, Seat reservation

(Detailed implementations available in repository)

---

### SOLID Principles (Memorize These)

**S - Single Responsibility Principle**
```
A class should have only one reason to change.

❌ Bad: class UserManager handles user CRUD + email sending + logging
✅ Good: class UserRepository (CRUD), EmailService (email), Logger (logging)
```

**O - Open/Closed Principle**
```
Open for extension, closed for modification.

❌ Bad: Modify existing class to add new feature
✅ Good: Extend via inheritance or composition
```

**L - Liskov Substitution Principle**
```
Derived classes must be substitutable for their base classes.

❌ Bad: Square extends Rectangle, but breaks area calculation
✅ Good: Square and Rectangle both implement Shape interface
```

**I - Interface Segregation Principle**
```
Don't force clients to depend on interfaces they don't use.

❌ Bad: interface Worker { work(); eat(); sleep(); } # Robots don't eat!
✅ Good: interface Workable { work(); }, interface Eatable { eat(); }
```

**D - Dependency Inversion Principle**
```
Depend on abstractions, not concretions.

❌ Bad: class Service { private MySQLDatabase db; }
✅ Good: class Service { private IDatabase db; } # Works with any DB
```

---

### Common Design Patterns for Interviews

**Creational Patterns:**
1. **Singleton:** Only one instance (e.g., ParkingLot, Database connection)
2. **Factory:** Create objects without specifying exact class (e.g., VehicleFactory)
3. **Builder:** Construct complex objects step by step (e.g., SQL query builder)

**Structural Patterns:**
4. **Adapter:** Make incompatible interfaces work together
5. **Decorator:** Add behavior to objects dynamically (e.g., Java I/O streams)
6. **Facade:** Simplified interface to complex subsystem

**Behavioral Patterns:**
7. **Observer:** Notify dependents of state changes (e.g., Event listeners)
8. **Strategy:** Select algorithm at runtime (e.g., Payment methods)
9. **State:** Object behavior changes with state (e.g., Vending machine)
10. **Command:** Encapsulate request as object (e.g., Undo/Redo)

**Most Common in Interviews:** Singleton, Factory, Strategy, Observer, State

---

## Interview Strategy & Communication

### The 3 C's of Interview Success

#### 1. Clarification (First 5 minutes)

**Always ask these questions:**

```
Functional Requirements:
- "What are the core features we must support?"
- "What's out of scope for this design?"
- "Are there any specific features we should prioritize?"

Non-Functional Requirements:
- "How many users do we expect?"
- "What's the read-to-write ratio?"
- "What's more important: consistency or availability?"
- "Do we need to support multiple regions?"
- "What's the expected latency requirement?"

Constraints:
- "Are there any specific technologies we must use?"
- "What's our budget for infrastructure?"
- "Do we need to integrate with existing systems?"
```

**Pro Tip:** Write down requirements on the whiteboard. It shows you're organized.

#### 2. Communication (Throughout)

**Think out loud:**
```
❌ Silent for 2 minutes, then "I think we should use Redis."
✅ "Let me think about caching options... We have high read traffic, so
   caching makes sense. Redis gives us data structures like sorted sets
   which could be useful for leaderboards. Memcached is simpler but
   less flexible. Given our use case, I'll go with Redis."
```

**Use the right vocabulary:**
- Don't say "database thingy" → Say "database" or "data store"
- Don't say "make it faster" → Say "reduce latency" or "improve throughput"
- Don't say "lots of users" → Say "millions of daily active users"

**Check in regularly:**
```
At minute 10: "Does this high-level design make sense? Should I go deeper?"
At minute 20: "Is this the right level of detail, or should I zoom out?"
At minute 30: "What component would you like me to deep dive into?"
```

#### 3. Confidence (The right amount)

**Be confident, not arrogant:**
```
❌ "This is definitely the best approach."
✅ "I think this approach works well because X, but there's a trade-off with Y."

❌ "I don't know."
✅ "I haven't worked with this exact technology, but my understanding is..."
```

**Handle uncertainty:**
```
If you don't know something:
1. Say what you do know
2. Make an educated guess
3. State your assumptions

Example: "I haven't used Kafka directly, but I know it's a distributed
message queue that supports high throughput. I'd use it here for its
ability to handle millions of events per second."
```

---

### Common Pitfalls & How to Avoid Them

#### Pitfall 1: Jumping to Solution Too Quickly

❌ **Wrong:**
```
Interviewer: "Design a URL shortener."
You: "We'll use MongoDB to store URLs and Redis for caching!"
```

✅ **Right:**
```
Interviewer: "Design a URL shortener."
You: "Let me clarify the requirements first. How many URLs are we
     shortening per day? What's the read-to-write ratio? Do we need
     analytics? How long should we keep URLs?"
```

**Why:** Shows you gather requirements before diving into solutions (like a real engineer).

#### Pitfall 2: Over-engineering

❌ **Wrong:**
```
You: "For 1000 users, we'll use Kubernetes with 10 microservices,
     Kafka for messaging, Elasticsearch for search, and multi-region
     deployment."
```

✅ **Right:**
```
You: "For 1000 users, a simple monolith with a single database would
     work fine. As we scale to 100K+ users, we can separate into
     microservices and add caching."
```

**Why:** Scale your solution to the problem size. Don't use a cannon to kill a fly.

#### Pitfall 3: Not Discussing Trade-offs

❌ **Wrong:**
```
You: "We'll use NoSQL for the database."
Interviewer: "Why NoSQL over SQL?"
You: "Because it's faster."
```

✅ **Right:**
```
You: "I'm choosing NoSQL because:
     ✅ We need horizontal scalability for billions of records
     ✅ Our access pattern is simple key-value lookups
     ✅ We can tolerate eventual consistency

     Trade-off: We lose ACID transactions and complex joins, but
     those aren't needed for this use case."
```

**Why:** Shows you understand there's no perfect solution, only trade-offs.

#### Pitfall 4: Ignoring the Interviewer's Hints

❌ **Wrong:**
```
Interviewer: "How would this handle a celebrity with 100M followers?"
You: "Uh, it should be fine."
```

✅ **Right:**
```
Interviewer: "How would this handle a celebrity with 100M followers?"
You: "Good point! Fan-out on write would be too slow. For celebrity
     users, I'd switch to fan-out on read, or use a hybrid approach
     where we fan out to active followers only."
```

**Why:** The interviewer is giving you a clue. Take it!

#### Pitfall 5: Weak Drawings

❌ **Wrong:**
- Messy arrows everywhere
- No labels
- Boxes overlapping
- Can't tell what connects to what

✅ **Right:**
- Clean boxes with clear labels
- Arrows show data flow direction
- Components grouped logically
- Legend if needed

**Whiteboarding Tip:** Use left side for high-level design, right side for deep dives.

---

## 8-Week Study Plan

See [PROGRESS_SUMMARY.md](progress_summary.md) for the complete week-by-week breakdown.

**Quick Summary:**

| Week | Focus | Hours | Key Deliverables |
|------|-------|-------|------------------|
| 1-2 | Fundamentals + Easy HLD | 20-25 | Master networking, caching, 3 beginner HLD |
| 3-4 | Core HLD Interview Problems | 25-30 | Master rate limiter, news feed, chat, notification |
| 5-6 | Advanced HLD + Critical LLD | 25-30 | Distributed cache, video streaming, LRU, parking lot |
| 7-8 | Important Problems + Mocks | 20-30 | Fill gaps, 4-5 full mock interviews |

**Total Time: 90-115 hours over 8 weeks**

---

## Company-Specific Preparation

### Meta (Facebook)

**Focus Areas:**
- Social networking (News Feed, Friend Recommendations)
- Messaging systems (WhatsApp, Messenger)
- Ads infrastructure

**Common Questions:**
- Design Facebook News Feed
- Design Instagram
- Design Facebook Messenger
- Design Live Comments

**What They Value:**
- Scale (billions of users)
- Real-time updates
- Graph algorithms (social graph)

**Prep Tips:**
- Study fan-out patterns deeply
- Understand WebSockets and push notifications
- Practice graph-based problems

### Google

**Focus Areas:**
- Infrastructure (GFS, BigTable, MapReduce)
- Search-related systems
- Global scale

**Common Questions:**
- Design Google Search
- Design Google Drive
- Design YouTube
- Design Google Maps

**What They Value:**
- Handling billions of queries
- Distributed systems theory
- Consistency and reliability

**Prep Tips:**
- Read Google papers (GFS, BigTable, MapReduce)
- Focus on sharding and replication
- Practice estimation heavily

### Amazon

**Focus Areas:**
- E-commerce systems
- Inventory management
- Distributed systems (Dynamo paper)

**Common Questions:**
- Design Amazon product catalog
- Design shopping cart
- Design order management system
- Design recommendation engine

**What They Value:**
- Availability over consistency
- Trade-off discussions
- Cost optimization

**Prep Tips:**
- Read Dynamo paper
- Study CAP theorem scenarios
- Focus on high availability designs

### Netflix

**Focus Areas:**
- Video streaming
- Content delivery (CDN)
- Recommendation systems

**Common Questions:**
- Design Netflix video streaming
- Design content recommendation system
- Design Netflix homepage

**What They Value:**
- CDN knowledge
- Adaptive bitrate streaming
- Personalization

**Prep Tips:**
- Study video encoding formats (HLS, DASH)
- Understand CDN architecture deeply
- Practice ML-based ranking problems

### Uber

**Focus Areas:**
- Geospatial systems
- Real-time matching
- Dispatch systems

**Common Questions:**
- Design Uber ride-sharing
- Design Uber Eats
- Design surge pricing

**What They Value:**
- Geospatial indexing (Geohash, S2)
- Real-time processing
- Matching algorithms

**Prep Tips:**
- Study geospatial data structures
- Understand WebSockets for real-time updates
- Practice supply-demand matching problems

---

## Self-Assessment & Practice

### How to Know You're Ready

Use this checklist before your interview:

#### HLD Readiness ✅

- [ ] Can explain any Must-Know problem in <40 minutes
- [ ] Can draw clean architecture diagrams on whiteboard
- [ ] Can estimate QPS, storage, bandwidth in <5 minutes
- [ ] Can discuss trade-offs for SQL vs NoSQL
- [ ] Can discuss trade-offs for Sync vs Async communication
- [ ] Can discuss trade-offs for different caching strategies
- [ ] Know when to use Redis vs Memcached
- [ ] Know when to use REST vs gRPC vs WebSockets
- [ ] Can explain CAP theorem with examples
- [ ] Can explain consistent hashing
- [ ] Can explain sharding strategies
- [ ] Can handle "what if 10x users?" questions

#### LLD Readiness ✅

- [ ] Can code LRU Cache from memory in <20 minutes
- [ ] Can explain all 5 SOLID principles with examples
- [ ] Can identify design patterns in code
- [ ] Can design class hierarchies for new problems
- [ ] Can discuss composition vs inheritance trade-offs
- [ ] Can write thread-safe code
- [ ] Can handle concurrent access scenarios
- [ ] Know when to use interface vs abstract class

### Practice Resources

**Mock Interview Platforms:**

1. **Pramp.com** (Free)
   - Peer-to-peer practice
   - Both HLD and LLD available
   - Good for getting comfortable

2. **interviewing.io** (Paid)
   - Practice with real engineers from FAANG
   - Anonymous interviews
   - Detailed feedback

3. **LeetCode System Design** (Paid)
   - 30+ curated problems
   - Video solutions
   - Discuss section is gold

**Study Groups:**
- Find 2-3 peers preparing for same interviews
- Weekly mock interview sessions
- Review each other's designs

**Solo Practice:**
- Set 45-minute timer
- Pick random problem
- Design on paper/whiteboard (NOT computer)
- Record yourself explaining it
- Review recording next day

### Self-Grading Rubric

After each mock interview, rate yourself (1-5):

**Communication (30%):**
- [ ] Did I ask clarifying questions? (5 = asked 5+ questions)
- [ ] Did I think out loud? (5 = constant communication)
- [ ] Did I check in with interviewer? (5 = checked in 3+ times)

**Technical Depth (25%):**
- [ ] Did I justify my choices? (5 = explained every decision)
- [ ] Did I discuss trade-offs? (5 = discussed 3+ trade-offs)
- [ ] Did I handle follow-ups? (5 = handled all without help)

**Problem-Solving (25%):**
- [ ] Did I start with requirements? (5 = wrote them down)
- [ ] Did I do estimation? (5 = calculated QPS, storage, bandwidth)
- [ ] Did I iterate on design? (5 = improved design based on feedback)

**System Design Skills (20%):**
- [ ] Was my high-level design sound? (5 = all components correct)
- [ ] Did I deep dive appropriately? (5 = 2-3 components in detail)
- [ ] Did I discuss scalability? (5 = discussed multiple scale levels)

**Scoring:**
- 90-100: Excellent, you'd likely pass
- 75-89: Good, but needs minor improvements
- 60-74: Okay, needs practice in weak areas
- <60: More preparation needed

---

## Final Tips for Interview Day

### The Night Before

- [ ] Review your top 3 strongest designs
- [ ] Practice 2-3 estimations
- [ ] Review RESHADED framework
- [ ] Get 8 hours of sleep
- [ ] Don't cram new topics

### 30 Minutes Before

- [ ] Review clarifying questions template
- [ ] Review SOLID principles
- [ ] Do breathing exercises
- [ ] Test your setup (camera, mic, whiteboard tool)

### During the Interview

**First 30 seconds:**
1. Greet interviewer warmly
2. Ask if you can take 30 seconds to think
3. Take a breath, organize thoughts

**If you get stuck:**
1. Say "Let me think out loud..."
2. State what you know
3. Make an educated guess
4. Ask for a hint if needed

**If you realize a mistake:**
1. Don't panic
2. Say "Actually, I think there's an issue with..."
3. Explain the issue
4. Propose a fix
5. Move on

### After the Interview

- [ ] Write down all questions asked
- [ ] Note what went well
- [ ] Note what to improve
- [ ] Don't obsess over mistakes
- [ ] Update your study plan

---

## Additional Resources

### Must-Read Papers

1. **Google Papers:**
   - Google File System (GFS)
   - BigTable
   - MapReduce
   - Spanner

2. **Amazon Papers:**
   - Dynamo (key-value store)
   - Aurora (database)

3. **Facebook Papers:**
   - TAO (social graph)
   - Memcached at Facebook

4. **Other:**
   - Kafka (LinkedIn)
   - Raft Consensus Algorithm

### Recommended Books

1. **"Designing Data-Intensive Applications"** by Martin Kleppmann
   - Best book for understanding distributed systems
   - Read chapters 5-9 at minimum

2. **"System Design Interview Vol 1 & 2"** by Alex Xu
   - Great for interview-specific prep
   - Visual diagrams help a lot

3. **"Design Patterns"** by Gang of Four
   - Classic for LLD prep
   - Focus on patterns listed in this guide

### YouTube Channels

1. **Gaurav Sen**
   - Clear explanations, good for beginners
   - Covers popular interview problems

2. **Tech Dummies Narendra L**
   - Deep technical dives
   - More advanced topics

3. **System Design Interview**
   - Interview-focused content
   - Mock interview walkthroughs

---

## Conclusion

**Remember:**

1. **Breadth first, then depth:** Master Must-Know problems before Good-to-Know
2. **Practice out loud:** Explaining is different from understanding
3. **Draw everything:** Visual communication is key
4. **Discuss trade-offs:** There's no perfect solution
5. **Stay calm:** Interviews are conversations, not interrogations

**You've got this!** With 90-115 hours of focused preparation using this guide and the repository materials, you'll be ready to ace your system design interviews.

**Good luck!** 🚀

---

**Questions or need clarification?** Review the specific problem READMEs in this repository for detailed implementations and examples.

**Track your progress** using [PROGRESS_SUMMARY.md](progress_summary.md) and update checkboxes as you study.
