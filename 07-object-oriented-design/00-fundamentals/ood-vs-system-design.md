# Object-Oriented Design vs System Design

## Overview

Many candidates confuse **Object-Oriented Design (OOD)** and **System Design** because both involve designing software systems. However, they operate at different levels of abstraction and focus on different concerns.

This guide clarifies the differences, helps you identify which type of interview you're in, and provides strategies for each.

---

## Quick Comparison

| Aspect | Object-Oriented Design (OOD) | System Design |
|--------|------------------------------|---------------|
| **Scope** | Single component, class structure | Entire system, multiple services |
| **Scale** | Hundreds to thousands of users | Millions to billions of users |
| **Focus** | Classes, objects, relationships | Services, databases, infrastructure |
| **Time** | 35-45 minutes | 45-60 minutes |
| **Output** | Class diagram + code | Architecture diagram + trade-offs |
| **Questions** | "Design a parking lot" | "Design Netflix" |
| **Depth** | Deep dive into code | Broad system architecture |
| **Patterns** | Design patterns (Singleton, Factory) | Architectural patterns (Microservices, Sharding) |

---

## Object-Oriented Design (OOD)

### What is OOD?

OOD interviews focus on designing the **internal structure** of a system using object-oriented principles. You're expected to:
- Identify classes and their responsibilities
- Define relationships between classes
- Apply SOLID principles
- Implement design patterns
- Write clean, maintainable code

### Characteristics

**Scope:**
- Single application or component
- Focus on code-level design
- Typically runs on one machine

**Scale:**
- Hundreds to thousands of users
- No distributed systems
- No scalability concerns

**Key Topics:**
- Classes and objects
- Inheritance, polymorphism, encapsulation
- Design patterns (Gang of Four)
- SOLID principles
- UML class diagrams
- Code implementation

### Example OOD Questions

1. **Design a Parking Lot**
   - Classes: ParkingLot, ParkingSpot, Vehicle, Ticket
   - Focus: Object modeling, state management

2. **Design a Deck of Cards**
   - Classes: Deck, Card, Suit, Rank
   - Focus: Inheritance, encapsulation

3. **Design an Elevator System**
   - Classes: Elevator, Floor, Request, Controller
   - Focus: State pattern, strategy pattern

4. **Design a Vending Machine**
   - Classes: VendingMachine, Product, Coin, State
   - Focus: State pattern, finite state machines

5. **Design a Chess Game**
   - Classes: Board, Piece, Player, Move
   - Focus: Inheritance hierarchy, polymorphism

### OOD Interview Structure

**Time Allocation (45 minutes):**

1. **Requirements Gathering (5 min)**
   - Clarify use cases
   - Define functional requirements
   - Identify constraints

2. **Core Objects Identification (5 min)**
   - List main entities (nouns)
   - Identify actions (verbs → methods)
   - Define relationships

3. **Class Diagram (10 min)**
   - Draw UML diagram
   - Show inheritance/composition
   - Add key methods

4. **Implementation (20 min)**
   - Write code for core classes
   - Demonstrate functionality
   - Handle edge cases

5. **Discussion (5 min)**
   - Design patterns used
   - SOLID principles applied
   - Extensions and improvements

### What Interviewers Look For (OOD)

- Clean code organization
- Proper encapsulation
- Appropriate use of inheritance vs composition
- Design pattern application
- SOLID principle adherence
- Extensibility considerations
- Edge case handling

### Example: Parking Lot (OOD)

**Classes:**
```python
from enum import Enum
from typing import Optional

class VehicleType(Enum):
    CAR = 1
    MOTORCYCLE = 2
    TRUCK = 3

class ParkingSpotType(Enum):
    COMPACT = 1
    REGULAR = 2
    LARGE = 3

class Vehicle:
    def __init__(self, license_plate: str, vehicle_type: VehicleType):
        self.license_plate = license_plate
        self.vehicle_type = vehicle_type

class ParkingSpot:
    def __init__(self, spot_id: int, spot_type: ParkingSpotType):
        self.spot_id = spot_id
        self.spot_type = spot_type
        self.vehicle: Optional[Vehicle] = None
        self.is_available = True

    def park_vehicle(self, vehicle: Vehicle) -> bool:
        if self.is_available and self._can_fit(vehicle):
            self.vehicle = vehicle
            self.is_available = False
            return True
        return False

    def remove_vehicle(self) -> Optional[Vehicle]:
        vehicle = self.vehicle
        self.vehicle = None
        self.is_available = True
        return vehicle

    def _can_fit(self, vehicle: Vehicle) -> bool:
        # Logic to check if vehicle fits in spot
        return True

class ParkingLot:
    def __init__(self):
        self.spots: list[ParkingSpot] = []
        self.parked_vehicles: dict[str, ParkingSpot] = {}

    def add_spot(self, spot: ParkingSpot):
        self.spots.append(spot)

    def park_vehicle(self, vehicle: Vehicle) -> Optional[ParkingSpot]:
        for spot in self.spots:
            if spot.park_vehicle(vehicle):
                self.parked_vehicles[vehicle.license_plate] = spot
                return spot
        return None

    def remove_vehicle(self, license_plate: str) -> bool:
        if license_plate in self.parked_vehicles:
            spot = self.parked_vehicles[license_plate]
            spot.remove_vehicle()
            del self.parked_vehicles[license_plate]
            return True
        return False
```

**Focus:** Code structure, class relationships, design patterns

---

## System Design

### What is System Design?

System Design interviews focus on designing **large-scale distributed systems**. You're expected to:
- Design architecture for millions/billions of users
- Handle scalability, availability, consistency
- Choose appropriate databases and services
- Consider trade-offs and bottlenecks
- Design for failure and monitoring

### Characteristics

**Scope:**
- Entire system end-to-end
- Multiple services and components
- Distributed infrastructure

**Scale:**
- Millions to billions of users
- Distributed across data centers
- Global availability

**Key Topics:**
- Load balancing
- Caching strategies
- Database sharding/replication
- Microservices architecture
- Message queues
- CDNs
- Scalability patterns
- CAP theorem
- Consistency vs availability trade-offs

### Example System Design Questions

1. **Design Netflix**
   - Video streaming at scale
   - Focus: CDN, encoding, recommendations

2. **Design Twitter**
   - Real-time feed generation
   - Focus: Fanout, caching, sharding

3. **Design Uber**
   - Real-time matching
   - Focus: Geospatial indexing, matching algorithms

4. **Design WhatsApp**
   - Messaging at scale
   - Focus: WebSockets, message delivery, presence

5. **Design URL Shortener**
   - High-throughput read/write
   - Focus: Hashing, database choice, caching

### System Design Interview Structure

**Time Allocation (60 minutes):**

1. **Requirements Gathering (5-10 min)**
   - Functional requirements
   - Non-functional requirements (scale, latency, availability)
   - Back-of-envelope calculations

2. **High-Level Design (10-15 min)**
   - Architecture diagram (clients, servers, databases)
   - API design
   - Data models

3. **Detailed Design (20-25 min)**
   - Database schema
   - Caching strategy
   - Scaling approach
   - Handling bottlenecks

4. **Trade-offs & Bottlenecks (10-15 min)**
   - CAP theorem implications
   - Consistency models
   - Failure scenarios
   - Monitoring and alerting

### What Interviewers Look For (System Design)

- Structured thinking
- Scalability considerations
- Trade-off analysis
- Understanding of distributed systems
- Database choice justification
- Caching strategies
- Handling failure scenarios
- Practical experience with real systems

### Example: URL Shortener (System Design)

**Architecture:**

```
                     ┌──────────────┐
                     │  CDN / Cache │
                     └──────┬───────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
    ┌─────────▼────────┐        ┌────────▼────────┐
    │  Load Balancer   │        │  Load Balancer  │
    └─────────┬────────┘        └────────┬────────┘
              │                           │
    ┌─────────▼────────┐        ┌────────▼────────┐
    │  Web Servers     │        │  Web Servers    │
    │  (Create URLs)   │        │  (Redirect)     │
    └─────────┬────────┘        └────────┬────────┘
              │                           │
              │         ┌─────────────────┘
              │         │
    ┌─────────▼─────────▼────┐
    │  Redis Cache            │
    │  (Hot URLs)             │
    └─────────┬───────────────┘
              │
    ┌─────────▼───────────────┐
    │  Database Cluster       │
    │  (Sharded by URL hash)  │
    │  - Shard 1 - Shard N    │
    └─────────────────────────┘
```

**Key Decisions:**

1. **URL Generation:**
   - Base62 encoding of auto-increment ID
   - Or hash-based (MD5 → Base62)

2. **Database:**
   - NoSQL (Cassandra) for horizontal scaling
   - Sharding by URL hash
   - Replication for availability

3. **Caching:**
   - Redis for hot URLs
   - CDN for geographic distribution
   - Cache-aside pattern

4. **Scalability:**
   - Horizontal scaling of web servers
   - Database sharding
   - Read replicas for redirects

5. **API Design:**
   ```
   POST /api/shorten
   Body: { "long_url": "https://example.com/very/long/url" }
   Response: { "short_url": "https://short.ly/abc123" }

   GET /abc123
   Response: 302 Redirect to original URL
   ```

**Focus:** Architecture, scalability, trade-offs

---

## Key Differences Explained

### 1. Abstraction Level

**OOD:**
```
"How should I structure my classes?"
"What methods should this object have?"
"Should I use inheritance or composition here?"
```

**System Design:**
```
"How do I handle 100M requests per day?"
"Should I use SQL or NoSQL?"
"How do I ensure 99.99% uptime?"
```

### 2. Code vs Architecture

**OOD:**
- You **write actual code**
- Focus on implementation details
- Class methods, attributes, visibility
- Code runs on single machine

**System Design:**
- You **draw architecture diagrams**
- Focus on components and interfaces
- Services, databases, message queues
- Distributed across many machines

### 3. Patterns Used

**OOD Patterns (Gang of Four):**
- Singleton
- Factory Method
- Observer
- Strategy
- Decorator
- State

**System Design Patterns:**
- Microservices
- Event sourcing
- CQRS (Command Query Responsibility Segregation)
- Saga pattern
- Circuit breaker
- Database sharding

### 4. Scaling Concerns

**OOD:**
- Algorithm complexity (O(n) vs O(log n))
- Memory usage of objects
- Code maintainability
- Hundreds to thousands of users

**System Design:**
- Horizontal vs vertical scaling
- Database partitioning
- Load distribution
- Millions to billions of users

### 5. Questions Asked

**OOD:**
- "What design patterns did you use?"
- "Why did you use composition over inheritance?"
- "How would you extend this to support X?"
- "Show me the code for this method"

**System Design:**
- "How would you handle 10x traffic?"
- "What if the database goes down?"
- "How do you ensure data consistency?"
- "What's your caching strategy?"

---

## When OOD and System Design Overlap

Some questions can be approached from **both** perspectives:

### Example: Design a Chat Application

**OOD Perspective (Class-level):**
```python
class User:
    def send_message(self, message: Message, recipient: User)
    def receive_message(self, message: Message)

class Message:
    def __init__(self, sender: User, content: str, timestamp: datetime)

class ChatRoom:
    def add_user(self, user: User)
    def broadcast_message(self, message: Message)

class ChatService:
    def create_room(self, name: str) -> ChatRoom
    def send_message(self, message: Message)
```

**System Design Perspective (Architecture):**
```
Users → Load Balancer → WebSocket Servers → Message Queue (Kafka)
                                          → Message DB (Cassandra)
                                          → Presence Service (Redis)
                                          → Notification Service
```

**Which to use?**
- Listen to interviewer's focus
- "Design the classes for..." → OOD
- "Design the architecture for..." → System Design
- "How would you handle millions of users?" → System Design

---

## How to Identify Interview Type

### Clues it's OOD:

- **Question phrasing:**
  - "Design a parking lot system"
  - "Implement a deck of cards"
  - "Model a chess game"

- **Interviewer focus:**
  - Asks about classes and objects
  - Wants to see UML diagrams
  - Requests code implementation
  - Discusses design patterns

- **Scale hints:**
  - No mention of millions of users
  - Single instance assumed
  - No distributed system concerns

### Clues it's System Design:

- **Question phrasing:**
  - "Design Netflix"
  - "Build a URL shortener like bit.ly"
  - "Design Twitter's feed system"

- **Interviewer focus:**
  - Asks about scalability
  - Wants architecture diagrams
  - Discusses databases and caching
  - Mentions millions/billions of users

- **Scale hints:**
  - "Handle 100M daily active users"
  - "Ensure 99.99% uptime"
  - "Low latency globally"

### When in Doubt: ASK!

If unclear, clarify with interviewer:
- "Should I focus on the class structure or the overall architecture?"
- "Are we designing for single-instance or distributed deployment?"
- "How many users are we expecting?"

---

## Transition Between OOD and System Design

Some interviews start with OOD and expand to system design:

### Example Interview Flow: Design a Library System

**Phase 1: OOD (20 min)**
- Interviewer: "Design a library management system"
- You: Design classes (Book, Member, Librarian, Checkout)
- Draw UML diagram
- Implement core functionality

**Phase 2: Transition (5 min)**
- Interviewer: "Great! Now let's say this is for a national library network with 10M users..."

**Phase 3: System Design (20 min)**
- You: Transition to distributed architecture
- Multiple library branches
- Central catalog database
- Reservation system
- Sharding by library branch or user ID

**Key Skill:** Ability to switch contexts smoothly

---

## Preparation Strategy

### For OOD Interviews

**Study:**
1. SOLID principles
2. Gang of Four design patterns
3. UML class diagrams
4. Object-oriented programming concepts

**Practice:**
1. Implement common OOD problems (15-20 problems)
2. Draw UML diagrams for each
3. Code complete solutions
4. Review design pattern applications

**Resources:**
- "Design Patterns" (Gang of Four book)
- "Head First Design Patterns"
- LeetCode OOD section
- This repository's `02-beginner-problems/`

### For System Design Interviews

**Study:**
1. Scalability patterns
2. Database types and trade-offs
3. Caching strategies
4. Load balancing
5. CAP theorem

**Practice:**
1. Design popular systems (20-30 problems)
2. Calculate capacity requirements
3. Draw architecture diagrams
4. Discuss trade-offs

**Resources:**
- "Designing Data-Intensive Applications" (Kleppmann)
- "System Design Interview" (Alex Xu)
- Grokking the System Design Interview
- This repository's system design section

---

## Combined Preparation

Since both types can appear in interviews, prepare for both:

### Week 1-2: OOD Foundations
- SOLID principles
- Design patterns (Creational)
- UML diagrams
- 5 beginner OOD problems

### Week 3-4: OOD Practice
- Design patterns (Structural, Behavioral)
- 10 intermediate OOD problems
- Code implementation practice

### Week 5-6: System Design Foundations
- Scalability concepts
- Database fundamentals
- Caching and load balancing
- 5 basic system design problems

### Week 7-8: System Design Practice
- Advanced distributed systems
- Trade-off analysis
- 10 complex system design problems

### Week 9+: Mixed Practice
- Both OOD and system design
- Mock interviews
- Review and refine

---

## Common Mistakes

### Mistake 1: Treating OOD as System Design

**Wrong:**
```
Interviewer: "Design a parking lot"
Candidate: "First, we'll use a load balancer to distribute requests across
multiple parking lot servers, then we'll shard our database by parking spot ID..."
```

**Right:**
```
Interviewer: "Design a parking lot"
Candidate: "Let me identify the core objects: ParkingLot, ParkingSpot, Vehicle, Ticket.
Let me draw the class relationships..."
```

### Mistake 2: Treating System Design as OOD

**Wrong:**
```
Interviewer: "Design Netflix"
Candidate: "Here's my Video class with methods play(), pause(), rewind()..."
```

**Right:**
```
Interviewer: "Design Netflix"
Candidate: "For streaming to millions of users, we'll need a CDN for video delivery,
origin servers for encoding, and a recommendation service..."
```

### Mistake 3: Going Too Deep Too Fast

**OOD:**
- Don't implement every method before discussing design
- Start with high-level class structure

**System Design:**
- Don't dive into database schema before architecture
- Start with high-level components

### Mistake 4: Ignoring Interviewer Signals

- Interviewer asks about classes → Stay in OOD
- Interviewer asks about scale → Transition to system design
- Interviewer asks for code → Write implementation
- Interviewer asks for architecture → Draw components

---

## Real Interview Examples

### Example 1: Amazon OOD Interview

**Question:** "Design an online shopping cart"

**Approach:**
1. Classes: ShoppingCart, Product, CartItem, User
2. Relationships: User has-a ShoppingCart, ShoppingCart has-many CartItems
3. Methods: add_item(), remove_item(), calculate_total(), checkout()
4. Design patterns: Observer (notify on price changes), Strategy (pricing strategies)

**Focus:** Class design, not distributed architecture

### Example 2: Google System Design Interview

**Question:** "Design Google Drive"

**Approach:**
1. Requirements: File upload, sync, sharing (1B users)
2. Architecture: Upload service, metadata DB, file storage (S3), sync service
3. Scalability: CDN for downloads, chunking for uploads, delta sync
4. Trade-offs: Consistency vs availability, sync conflicts

**Focus:** Distributed architecture, not class methods

### Example 3: Facebook Hybrid Interview

**Question:** "Design a notification system"

**Part 1 (OOD):**
- Classes: Notification, User, NotificationSender
- Factory pattern for different notification types
- Observer pattern for subscriptions

**Part 2 (System Design):**
- Push notification service
- Message queue (Kafka)
- Database for notification history
- Real-time delivery (WebSockets)

**Focus:** Both perspectives required

---

## Summary

### Object-Oriented Design (OOD)

**Focus:** Classes, objects, design patterns
**Scale:** Single machine, thousands of users
**Output:** UML diagrams + code
**Examples:** Parking lot, deck of cards, elevator

### System Design

**Focus:** Architecture, scalability, distributed systems
**Scale:** Multiple machines, millions of users
**Output:** Architecture diagrams + trade-offs
**Examples:** Netflix, Twitter, Uber

### Key Differences

| OOD | System Design |
|-----|---------------|
| Code-level | Architecture-level |
| Design patterns | Architectural patterns |
| Class diagrams | Component diagrams |
| Single machine | Distributed system |
| Hundreds-thousands | Millions-billions |
| "How to implement?" | "How to scale?" |

### Interview Strategy

1. **Identify the type** from question phrasing and scale
2. **Clarify when uncertain** - ask interviewer
3. **Start appropriately** - classes for OOD, architecture for system design
4. **Be ready to transition** between both in some interviews
5. **Practice both** - they test different skills

### Next Steps

- **OOD Practice:** Work through `02-beginner-problems/` folder
- **Learn Patterns:** Study `design-patterns-overview.md`
- **Master UML:** Review `uml-diagrams.md`
- **Interview Prep:** Read `interview-approach.md`
- **System Design:** Move to system design section of this repo

---

**Remember:** Both OOD and System Design are essential skills for software engineers. Master both to succeed in technical interviews!
