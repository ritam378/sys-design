# Object-Oriented Design (OOD) - Interview Preparation

Welcome to the Object-Oriented Design module! This section covers low-level design (LLD) problems commonly asked in technical interviews, distinct from high-level distributed system design.

## Overview

**What is OOD?**
Object-Oriented Design focuses on designing **classes, interfaces, and their relationships** to solve specific problems. It emphasizes:
- Class design and responsibilities
- Design patterns and principles
- Code organization and modularity
- Maintainability and extensibility

**OOD vs System Design:**

| Aspect | OOD (Low-Level Design) | System Design (High-Level Design) |
|--------|------------------------|-----------------------------------|
| **Scope** | Single application/component | Distributed systems |
| **Focus** | Classes, interfaces, methods | Servers, databases, APIs |
| **Scale** | Runs on one machine | Runs on many machines |
| **Concerns** | Inheritance, polymorphism, encapsulation | Scalability, availability, consistency |
| **Example** | Design a Chess game | Design Twitter |
| **Interview Round** | L4-L5 coding rounds | L5+ system design rounds |

**When each is asked:**
- **OOD:** Junior/Mid-level (L3-L5), sometimes combined with coding
- **System Design:** Senior+ (L5+), separate dedicated round

## Directory Structure

```
07-object-oriented-design/
├── README.md                          # This file
├── 00-fundamentals/                   # Core concepts
│   ├── solid-principles.md           # SOLID in depth with examples
│   ├── design-patterns-overview.md   # 23 GoF patterns summary
│   ├── uml-diagrams.md               # Class, sequence diagrams
│   ├── ood-vs-system-design.md       # When to use each
│   └── interview-approach.md         # How to approach OOD questions
├── 01-design-patterns/               # 23 Gang of Four patterns
│   ├── creational/                   # 5 creational patterns
│   ├── structural/                   # 7 structural patterns
│   └── behavioral/                   # 11 behavioral patterns
├── 02-beginner-problems/             # 8 beginner OOD problems
├── 03-intermediate-problems/         # 8 intermediate problems
├── 04-advanced-problems/             # 6 advanced problems
└── 05-templates/                     # Problem-solving templates
    └── ood-template.md
```

## Learning Path

### Phase 1: Foundations (Week 1)
1. Read [SOLID Principles](00-fundamentals/solid-principles.md)
2. Study [Design Patterns Overview](00-fundamentals/design-patterns-overview.md)
3. Learn [UML Diagrams](00-fundamentals/uml-diagrams.md)
4. Understand [Interview Approach](00-fundamentals/interview-approach.md)

### Phase 2: Design Patterns (Week 2-3)
1. **Creational** (5 patterns) - Singleton, Factory, Builder, etc.
2. **Structural** (7 patterns) - Adapter, Decorator, Facade, etc.
3. **Behavioral** (11 patterns) - Observer, Strategy, Command, etc.

### Phase 3: Problem Solving (Week 4-6)
1. **Beginner** (8 problems) - Parking Lot, Deck of Cards, Vending Machine
2. **Intermediate** (8 problems) - Elevator, Library, Hotel Booking
3. **Advanced** (6 problems) - Uber App, Amazon Shopping, Chess

## Content Overview

### Beginner Problems (8)

| # | Problem | Key Concepts | Difficulty |
|---|---------|--------------|------------|
| 1 | [Deck of Cards](02-beginner-problems/deck-of-cards/) | Enum, Class hierarchy | ⭐ |
| 2 | [Parking Lot](02-beginner-problems/parking-lot/) | State pattern, Strategy | ⭐⭐ |
| 3 | [Vending Machine](02-beginner-problems/vending-machine/) | State pattern | ⭐⭐ |
| 4 | [Coffee Maker](02-beginner-problems/coffee-maker/) | Builder pattern | ⭐ |
| 5 | [Movie Ticket Booking](02-beginner-problems/movie-ticket-booking/) | Booking system basics | ⭐⭐ |
| 6 | [Stack/Queue](02-beginner-problems/stack-queue/) | Data structure design | ⭐ |
| 7 | [HashMap](02-beginner-problems/hash-map/) | Internal implementation | ⭐⭐ |
| 8 | [LRU Cache](02-beginner-problems/lru-cache/) | LinkedHashMap pattern | ⭐⭐ |

### Intermediate Problems (8)

| # | Problem | Key Concepts | Difficulty |
|---|---------|--------------|------------|
| 1 | [Library Management](03-intermediate-problems/library-management/) | CRUD operations, Search | ⭐⭐⭐ |
| 2 | [Elevator System](03-intermediate-problems/elevator-system/) | Scheduling algorithms | ⭐⭐⭐ |
| 3 | [ATM Machine](03-intermediate-problems/atm-machine/) | State, Command patterns | ⭐⭐⭐ |
| 4 | [Hotel Booking](03-intermediate-problems/hotel-booking/) | Reservation, Availability | ⭐⭐⭐ |
| 5 | [Airline Reservation](03-intermediate-problems/airline-reservation/) | Seat assignment, Booking | ⭐⭐⭐ |
| 6 | [Online Shopping Cart](03-intermediate-problems/shopping-cart/) | Cart, Checkout, Inventory | ⭐⭐⭐ |
| 7 | [Chess Game](03-intermediate-problems/chess-game/) | Game rules, Move validation | ⭐⭐⭐⭐ |
| 8 | [Tic-Tac-Toe](03-intermediate-problems/tic-tac-toe/) | Game logic, AI | ⭐⭐ |

### Advanced Problems (6)

| # | Problem | Key Concepts | Difficulty |
|---|---------|--------------|------------|
| 1 | [Uber App](04-advanced-problems/uber-app/) | Rider-Driver matching, Trip | ⭐⭐⭐⭐ |
| 2 | [Amazon Shopping](04-advanced-problems/amazon-shopping/) | Product catalog, Orders, Reviews | ⭐⭐⭐⭐ |
| 3 | [Netflix Streaming](04-advanced-problems/netflix-streaming/) | Content, User profiles, Recommendations | ⭐⭐⭐⭐ |
| 4 | [LinkedIn Connections](04-advanced-problems/linkedin-connections/) | Social graph, Connections | ⭐⭐⭐⭐ |
| 5 | [Facebook News Feed](04-advanced-problems/facebook-newsfeed/) | Posts, Comments, Likes (OOD view) | ⭐⭐⭐⭐⭐ |
| 6 | [Stack Overflow](04-advanced-problems/stackoverflow/) | Questions, Answers, Voting | ⭐⭐⭐⭐ |

## Design Patterns Covered

### Creational Patterns (5)
1. **Singleton** - Ensure only one instance exists
2. **Factory Method** - Create objects without specifying exact class
3. **Abstract Factory** - Families of related objects
4. **Builder** - Construct complex objects step-by-step
5. **Prototype** - Clone objects

### Structural Patterns (7)
1. **Adapter** - Make incompatible interfaces work together
2. **Decorator** - Add functionality without modifying class
3. **Facade** - Simplified interface to complex subsystem
4. **Proxy** - Control access to an object
5. **Composite** - Tree structures (part-whole hierarchy)
6. **Bridge** - Separate abstraction from implementation
7. **Flyweight** - Share objects to save memory

### Behavioral Patterns (11)
1. **Observer** - Notify multiple objects of changes
2. **Strategy** - Encapsulate algorithms, make them interchangeable
3. **Command** - Encapsulate requests as objects
4. **State** - Change behavior when state changes
5. **Iterator** - Access elements sequentially
6. **Template Method** - Define algorithm skeleton
7. **Chain of Responsibility** - Pass request along chain
8. **Mediator** - Centralize complex communications
9. **Memento** - Capture and restore object state
10. **Visitor** - Add operations without modifying classes
11. **Interpreter** - Define grammar and interpret it

## SOLID Principles

### S - Single Responsibility Principle
A class should have only one reason to change.

**Example:**
```python
# Bad: UserManager does too much
class UserManager:
    def create_user(self): pass
    def send_email(self): pass        # Email responsibility
    def generate_report(self): pass   # Reporting responsibility

# Good: Separate responsibilities
class UserManager:
    def create_user(self): pass

class EmailService:
    def send_email(self): pass

class ReportGenerator:
    def generate_report(self): pass
```

### O - Open/Closed Principle
Open for extension, closed for modification.

**Example:**
```python
# Bad: Modify class for new shapes
class AreaCalculator:
    def calculate(self, shape):
        if shape.type == "circle":
            return 3.14 * shape.radius ** 2
        elif shape.type == "square":
            return shape.side ** 2

# Good: Extend through inheritance
class Shape(ABC):
    @abstractmethod
    def area(self): pass

class Circle(Shape):
    def area(self):
        return 3.14 * self.radius ** 2

class Square(Shape):
    def area(self):
        return self.side ** 2
```

### L - Liskov Substitution Principle
Subtypes must be substitutable for their base types.

**Example:**
```python
# Bad: Square violates LSP (breaks Rectangle's behavior)
class Rectangle:
    def set_width(self, width): self.width = width
    def set_height(self, height): self.height = height

class Square(Rectangle):
    def set_width(self, width):
        self.width = width
        self.height = width  # Breaks expectation!

# Good: Separate hierarchies
class Shape(ABC):
    @abstractmethod
    def area(self): pass

class Rectangle(Shape):
    def area(self): return self.width * self.height

class Square(Shape):
    def area(self): return self.side ** 2
```

### I - Interface Segregation Principle
Clients shouldn't depend on interfaces they don't use.

**Example:**
```python
# Bad: Printer interface too broad
class Printer(ABC):
    @abstractmethod
    def print(self): pass
    @abstractmethod
    def scan(self): pass
    @abstractmethod
    def fax(self): pass

# SimplePrinter forced to implement scan/fax
class SimplePrinter(Printer):
    def print(self): pass
    def scan(self): raise NotImplementedError
    def fax(self): raise NotImplementedError

# Good: Split into specific interfaces
class Printable(ABC):
    @abstractmethod
    def print(self): pass

class Scannable(ABC):
    @abstractmethod
    def scan(self): pass

class SimplePrinter(Printable):
    def print(self): pass

class AllInOnePrinter(Printable, Scannable):
    def print(self): pass
    def scan(self): pass
```

### D - Dependency Inversion Principle
Depend on abstractions, not concretions.

**Example:**
```python
# Bad: High-level depends on low-level
class MySQLDatabase:
    def save(self, data): pass

class UserService:
    def __init__(self):
        self.db = MySQLDatabase()  # Tight coupling!

# Good: Depend on abstraction
class Database(ABC):
    @abstractmethod
    def save(self, data): pass

class MySQLDatabase(Database):
    def save(self, data): pass

class PostgreSQLDatabase(Database):
    def save(self, data): pass

class UserService:
    def __init__(self, db: Database):  # Dependency injection
        self.db = db
```

## Interview Approach

### 4-Step Framework for OOD Questions

#### Step 1: Clarify Requirements (5 min)
```
Ask:
- What are the core features?
- What are the actors/users?
- What are the use cases?
- Any constraints? (single-player vs multiplayer, etc.)
- What should we focus on?
```

#### Step 2: Identify Core Objects (5 min)
```
From requirements, extract:
- Nouns → Potential classes
- Verbs → Potential methods
- Relationships → Inheritance, composition

Example (Parking Lot):
- Nouns: ParkingLot, Floor, Spot, Vehicle, Ticket
- Verbs: park(), unpark(), pay()
- Relationships: ParkingLot HAS-A Floors, Floor HAS-A Spots
```

#### Step 3: Define Classes and Relationships (15 min)
```
For each class:
- Attributes (private fields)
- Methods (public interface)
- Relationships (inheritance, composition)

Draw UML diagram:
- Classes as boxes
- Relationships as arrows
- Methods and attributes listed
```

#### Step 4: Implement and Discuss (15 min)
```
- Write code for 2-3 key classes
- Discuss design patterns used
- Talk about extensibility
- Mention edge cases
- Discuss trade-offs
```

## Common Patterns in OOD Problems

### 1. Booking/Reservation Systems
**Pattern:** Command + State
**Examples:** Hotel Booking, Airline Reservation, Movie Tickets
**Key Classes:** Booking, Reservation, Payment, Customer

### 2. Game Design
**Pattern:** State + Strategy
**Examples:** Chess, Tic-Tac-Toe, Card Games
**Key Classes:** Game, Player, Board, Move, Rules

### 3. Inventory/Catalog Systems
**Pattern:** Factory + Singleton
**Examples:** Vending Machine, Library, Shopping Cart
**Key Classes:** Product, Inventory, Catalog, Order

### 4. Scheduling Systems
**Pattern:** Strategy + Command
**Examples:** Elevator, Parking Lot, ATM
**Key Classes:** Scheduler, Request, Resource, Allocation

## Tips for Success

### Do's ✅
- Ask clarifying questions before designing
- Start with core classes (20% that do 80% of work)
- Use design patterns where appropriate (but don't force them)
- Draw UML diagrams to visualize relationships
- Write clean, readable code
- Discuss extensibility and trade-offs

### Don'ts ❌
- Don't jump to code immediately
- Don't try to design everything (focus on core)
- Don't over-engineer with unnecessary patterns
- Don't forget access modifiers (public/private)
- Don't ignore edge cases
- Don't forget to test your design mentally

## UML Quick Reference

### Class Diagram

```
┌─────────────────────┐
│   ClassName         │
├─────────────────────┤
│ - privateField      │
│ + publicField       │
├─────────────────────┤
│ + publicMethod()    │
│ - privateMethod()   │
└─────────────────────┘

Relationships:
→        Association (uses)
◆────    Composition (has-a, strong ownership)
◇────    Aggregation (has-a, weak ownership)
─────▷   Inheritance (is-a)
- - - ▷  Implementation (interface)
```

### Example: Parking Lot UML

```
         ┌──────────────┐
         │  ParkingLot  │
         ├──────────────┤
         │ - floors     │
         ├──────────────┤
         │ + parkVehicle│
         │ + unpark     │
         └──────┬───────┘
                │
                │ has-a (composition)
                │
         ┌──────▼───────┐
         │    Floor     │
         ├──────────────┤
         │ - spots      │
         ├──────────────┤
         │ + findSpot   │
         └──────┬───────┘
                │
                │ has-a
                │
         ┌──────▼───────┐
         │     Spot     │◁────────────┐
         ├──────────────┤             │
         │ - vehicle    │             │ is-a
         │ - occupied   │             │
         ├──────────────┤      ┌──────┴───────┐
         │ + park       │      │   Vehicle    │
         │ + unpark     │      ├──────────────┤
         └──────────────┘      │ - plate      │
                               │ - type       │
                               └──────┬───────┘
                                      │
                        ┌─────────────┼─────────────┐
                        │             │             │
                   ┌────▼────┐  ┌────▼────┐  ┌────▼────┐
                   │   Car   │  │  Truck  │  │  Bike   │
                   └─────────┘  └─────────┘  └─────────┘
```

## Practice Schedule

### Week 1: Fundamentals
- Day 1-2: SOLID principles
- Day 3-4: Design patterns overview
- Day 5-6: UML diagrams
- Day 7: Practice with Deck of Cards

### Week 2: Beginner Problems
- Day 1: Parking Lot
- Day 2: Vending Machine
- Day 3: Coffee Maker
- Day 4: Movie Ticket Booking
- Day 5: LRU Cache
- Day 6-7: Review and practice

### Week 3: Intermediate Problems
- Day 1: Library Management
- Day 2: Elevator System
- Day 3: ATM Machine
- Day 4: Hotel Booking
- Day 5: Chess Game
- Day 6-7: Review

### Week 4: Advanced Problems
- Day 1-2: Uber App
- Day 3-4: Amazon Shopping
- Day 5: LinkedIn Connections
- Day 6-7: Mock interviews

## Resources

### Books
- **Head First Design Patterns** - Beginner-friendly
- **Design Patterns: Elements of Reusable OO Software** - Gang of Four (GoF)
- **Clean Code** by Robert C. Martin - Code quality

### Online
- [Refactoring Guru](https://refactoring.guru/design-patterns) - Best design patterns resource
- [SourceMaking](https://sourcemaking.com/) - Design patterns and anti-patterns

### Practice Platforms
- LeetCode OOD problems
- Educative.io - Grokking OOD Interview
- InterviewBit - OOD questions

---

## Quick Start

**New to OOD?**
1. Start with [SOLID Principles](00-fundamentals/solid-principles.md)
2. Solve [Deck of Cards](02-beginner-problems/deck-of-cards/)
3. Solve [Parking Lot](02-beginner-problems/parking-lot/)

**Already know basics?**
1. Jump to [Elevator System](03-intermediate-problems/elevator-system/)
2. Study [Design Patterns](01-design-patterns/)
3. Tackle [Advanced Problems](04-advanced-problems/)

**Preparing for interview next week?**
1. Review [Interview Approach](00-fundamentals/interview-approach.md)
2. Practice top 5: Parking Lot, Elevator, Library, Chess, Uber
3. Do mock interviews

---

**Good luck with your OOD interviews!** Remember: clarity of thought and communication matter more than perfect code.

**Next:** Start with [SOLID Principles](00-fundamentals/solid-principles.md) →
