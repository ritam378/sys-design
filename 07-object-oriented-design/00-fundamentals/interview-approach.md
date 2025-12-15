# Object-Oriented Design Interview Approach

## Overview

This guide provides a **step-by-step framework** for tackling Object-Oriented Design (OOD) interviews. Following a structured approach helps you:

- Communicate your thought process clearly
- Cover all important aspects of design
- Manage time effectively (typically 35-45 minutes)
- Demonstrate systematic thinking
- Impress interviewers with your methodology

**Key Principle:** OOD interviews test your ability to translate real-world problems into well-designed code. The process is as important as the solution.

---

## The 5-Step Framework

### Overview

```
1. Requirements Gathering (5 min)
   └─> Clarify ambiguities, define scope

2. Core Objects Identification (5 min)
   └─> Identify entities and relationships

3. Class Diagram Design (10 min)
   └─> Create UML diagram with relationships

4. Implementation (20 min)
   └─> Write clean, working code

5. Review & Extension (5 min)
   └─> Discuss patterns, improvements, trade-offs
```

**Total Time:** 45 minutes

Let's dive deep into each step with examples.

---

## Step 1: Requirements Gathering (5 minutes)

### Goals

- Understand the problem fully
- Identify core use cases
- Define scope and constraints
- Clarify ambiguities

### What to Do

#### 1.1 Ask Clarifying Questions

**Never jump straight to coding!** Always ask questions first.

**Example Problem:** "Design a parking lot system"

**Good Questions:**
```
- "What types of vehicles should the system support?" (Cars, motorcycles, trucks?)
- "How should we handle parking spots of different sizes?"
- "Should we track entry/exit times?"
- "Is payment processing in scope?"
- "Do we need to handle multiple floors or levels?"
- "Should we support reservations?"
- "What happens when the lot is full?"
```

**Why these questions matter:**
- Shows you don't make assumptions
- Demonstrates systematic thinking
- Helps you avoid over-engineering or under-engineering
- Clarifies what interviewer wants to see

#### 1.2 Define Functional Requirements

List what the system **must do**.

**Format:** Use checkboxes for clarity

**Example: Parking Lot**

**Functional Requirements:**
- ✅ Support multiple vehicle types (car, motorcycle, truck)
- ✅ Support different spot sizes (compact, regular, large)
- ✅ Park and remove vehicles
- ✅ Track available spots
- ✅ Issue parking tickets with entry time
- ✅ Calculate parking fee based on duration
- ✅ Handle multiple floors/levels

#### 1.3 Define Non-Functional Requirements

List quality attributes and constraints.

**Example: Parking Lot**

**Non-Functional Requirements:**
- Fast spot allocation (< 100ms)
- Simple, maintainable code
- Extensible for future features
- Thread-safe (if applicable)

#### 1.4 Define Out of Scope

Explicitly state what you're **not** building.

**Example: Parking Lot**

**Out of Scope:**
- ❌ Payment gateway integration
- ❌ User authentication
- ❌ Mobile app interface
- ❌ Distributed system (single parking lot only)
- ❌ Analytics and reporting

### Tips for Step 1

**Do:**
- Ask 5-7 clarifying questions
- Write down requirements on whiteboard
- Get interviewer's confirmation
- Timebox to 5 minutes

**Don't:**
- Spend more than 5 minutes
- Ask questions you can reasonably assume
- Over-complicate scope
- Skip this step

### Example Script

```
You: "Before I start designing, let me clarify a few things..."

[Ask 5-7 questions]

You: "Based on our discussion, here are the key requirements:
     [List functional requirements]

     These are out of scope:
     [List out of scope items]

     Does this align with what you're looking for?"

Interviewer: "Yes, that looks good."

You: "Great! Let me move to identifying the core objects..."
```

---

## Step 2: Core Objects Identification (5 minutes)

### Goals

- Identify main entities (nouns → classes)
- Identify actions (verbs → methods)
- Define relationships between entities

### What to Do

#### 2.1 Identify Nouns (Potential Classes)

**Technique:** Underline nouns in problem statement and requirements

**Example: Parking Lot**

Problem: "Design a **parking lot** system that manages **vehicles** parking in **spots** and issues **tickets**."

**Nouns → Classes:**
- ParkingLot
- Vehicle
- ParkingSpot
- Ticket
- Floor/Level
- Payment

**Tip:** Look for entities with:
- State (attributes)
- Behavior (methods)
- Identity (unique instance)

#### 2.2 Identify Verbs (Potential Methods)

**Technique:** Underline verbs to find methods

**Example: Parking Lot**

Actions: "**Park** vehicle, **remove** vehicle, **find** available spot, **issue** ticket, **calculate** fee"

**Verbs → Methods:**
- `park_vehicle()`
- `remove_vehicle()`
- `find_available_spot()`
- `issue_ticket()`
- `calculate_fee()`

#### 2.3 Define Relationships

**Types of Relationships:**
- **Inheritance (IS-A):** Car IS-A Vehicle
- **Composition (HAS-A, strong):** ParkingLot HAS-A ParkingSpot (spots can't exist without lot)
- **Aggregation (HAS-A, weak):** ParkingSpot HAS-A Vehicle (vehicle exists independently)
- **Association (USES):** Ticket USES Vehicle

**Example: Parking Lot Relationships**

```
Inheritance:
- Car IS-A Vehicle
- Motorcycle IS-A Vehicle
- Truck IS-A Vehicle

Composition:
- ParkingLot HAS-A ParkingSpot (strong ownership)
- ParkingLot HAS-A Floor

Aggregation:
- ParkingSpot HAS-A Vehicle (weak - vehicle independent)

Association:
- Ticket REFERENCES Vehicle
- Ticket REFERENCES ParkingSpot
```

### Output Format

Document your findings clearly:

```
CORE OBJECTS:

Classes (Nouns):
- ParkingLot
- ParkingSpot
- Vehicle (abstract)
  - Car (extends Vehicle)
  - Motorcycle (extends Vehicle)
  - Truck (extends Vehicle)
- Ticket
- Floor

Methods (Verbs):
- park_vehicle(vehicle: Vehicle) -> Ticket
- remove_vehicle(ticket: Ticket) -> Vehicle
- find_available_spot(vehicle_type: VehicleType) -> ParkingSpot
- calculate_fee(ticket: Ticket) -> float

Relationships:
- Car, Motorcycle, Truck INHERIT FROM Vehicle
- ParkingLot OWNS (composition) ParkingSpots
- ParkingSpot CONTAINS (aggregation) Vehicle
- Ticket REFERENCES Vehicle and ParkingSpot
```

### Tips for Step 2

**Do:**
- Write down all entities
- Group related entities
- Identify obvious inheritance hierarchies
- Think about what "has-a" vs "is-a"

**Don't:**
- Skip relationship identification
- Confuse composition and aggregation
- Over-complicate with too many classes
- Spend more than 5 minutes

---

## Step 3: Class Diagram Design (10 minutes)

### Goals

- Create visual UML class diagram
- Show class attributes and methods
- Display relationships clearly
- Get interviewer feedback

### What to Do

#### 3.1 Draw Basic Class Boxes

**Format:**
```
┌─────────────────────┐
│   ClassName         │
├─────────────────────┤
│ - attribute1: type  │
│ + attribute2: type  │
├─────────────────────┤
│ + method1(): type   │
│ - method2(): type   │
└─────────────────────┘
```

**Visibility:**
- `+` = public
- `-` = private
- `#` = protected

#### 3.2 Add Relationships

**Symbols:**
- `─────>` = Association
- `◇────` = Aggregation (hollow diamond)
- `◆────` = Composition (filled diamond)
- `△` = Inheritance (hollow triangle)
- `△┆` = Realization/Interface (dashed hollow triangle)

#### 3.3 Complete Example: Parking Lot

```
                    ┌──────────────────────┐
                    │   <<enumeration>>    │
                    │    VehicleType       │
                    ├──────────────────────┤
                    │ CAR                  │
                    │ MOTORCYCLE           │
                    │ TRUCK                │
                    └──────────────────────┘


                    ┌──────────────────────┐
                    │   <<abstract>>       │
                    │      Vehicle         │
                    ├──────────────────────┤
                    │ - license_plate: str │
                    │ - type: VehicleType  │
                    ├──────────────────────┤
                    │ + get_type()         │
                    └──────────────────────┘
                            △
                            │
              ┌─────────────┼─────────────┐
              │             │             │
      ┌───────┴──────┐ ┌────┴─────┐ ┌────┴──────┐
      │     Car      │ │Motorcycle│ │   Truck   │
      └──────────────┘ └──────────┘ └───────────┘


┌─────────────────────────┐         ┌─────────────────────────┐
│     ParkingLot          │         │    ParkingSpot          │
├─────────────────────────┤         ├─────────────────────────┤
│ - name: str             │   1   * │ - spot_id: int          │
│ - floors: List[Floor]   │◆────────│ - spot_type: SpotType   │
│ - tickets: Dict         │         │ - is_available: bool    │
├─────────────────────────┤         │ - vehicle: Vehicle      │
│ + park_vehicle()        │         ├─────────────────────────┤
│ + remove_vehicle()      │         │ + park_vehicle()        │
│ + get_available_spots() │         │ + remove_vehicle()      │
│ + issue_ticket()        │         │ + is_available()        │
└─────────────────────────┘         └─────────────────────────┘
                                                │
                                                │ ◇ (aggregation)
                                                │
                                         ┌──────┴──────┐
                                         │   Vehicle   │
                                         └─────────────┘


┌─────────────────────────┐
│       Ticket            │
├─────────────────────────┤
│ - ticket_id: int        │───────> Vehicle
│ - entry_time: DateTime  │
│ - spot: ParkingSpot     │───────> ParkingSpot
├─────────────────────────┤
│ + calculate_fee()       │
│ + get_duration()        │
└─────────────────────────┘
```

### Tips for Step 3

**Do:**
- Start with main classes
- Add relationships progressively
- Use standard UML notation
- Discuss diagram with interviewer
- Make corrections if needed

**Don't:**
- Include every possible attribute
- Over-complicate relationships
- Spend more than 10 minutes
- Skip getting feedback

### Presentation Tips

**Talk through your diagram:**

```
You: "Here's the class structure I'm proposing:

1. Vehicle is an abstract base class with three concrete types: Car, Motorcycle, and Truck.

2. ParkingLot owns ParkingSpots (composition) - spots don't exist without the lot.

3. ParkingSpot can contain a Vehicle (aggregation) - the vehicle is independent.

4. Ticket references both Vehicle and ParkingSpot for tracking.

5. Main operations are park_vehicle() and remove_vehicle() on the ParkingLot.

Does this design make sense? Any feedback before I start coding?"
```

---

## Step 4: Implementation (20 minutes)

### Goals

- Write clean, working code
- Implement core functionality
- Demonstrate SOLID principles
- Handle edge cases

### What to Do

#### 4.1 Start with Enums and Simple Classes

**Why:** Build foundation first

**Example:**
```python
from enum import Enum
from typing import Optional
from datetime import datetime

class VehicleType(Enum):
    CAR = 1
    MOTORCYCLE = 2
    TRUCK = 3

class SpotType(Enum):
    COMPACT = 1
    REGULAR = 2
    LARGE = 3
```

#### 4.2 Implement Base Classes and Inheritance

```python
class Vehicle:
    """Abstract base class for vehicles"""

    def __init__(self, license_plate: str, vehicle_type: VehicleType):
        self.license_plate = license_plate
        self.vehicle_type = vehicle_type

    def get_type(self) -> VehicleType:
        return self.vehicle_type

class Car(Vehicle):
    def __init__(self, license_plate: str):
        super().__init__(license_plate, VehicleType.CAR)

class Motorcycle(Vehicle):
    def __init__(self, license_plate: str):
        super().__init__(license_plate, VehicleType.MOTORCYCLE)

class Truck(Vehicle):
    def __init__(self, license_plate: str):
        super().__init__(license_plate, VehicleType.TRUCK)
```

#### 4.3 Implement Core Component Classes

```python
class ParkingSpot:
    """Represents a single parking spot"""

    def __init__(self, spot_id: int, spot_type: SpotType):
        self.spot_id = spot_id
        self.spot_type = spot_type
        self.vehicle: Optional[Vehicle] = None
        self.is_available = True

    def park_vehicle(self, vehicle: Vehicle) -> bool:
        """Park a vehicle in this spot"""
        if not self.is_available:
            return False

        if not self._can_fit(vehicle):
            return False

        self.vehicle = vehicle
        self.is_available = False
        return True

    def remove_vehicle(self) -> Optional[Vehicle]:
        """Remove vehicle from spot and return it"""
        if self.is_available:
            return None

        vehicle = self.vehicle
        self.vehicle = None
        self.is_available = True
        return vehicle

    def _can_fit(self, vehicle: Vehicle) -> bool:
        """Check if vehicle can fit in this spot"""
        # Motorcycle can fit in any spot
        if vehicle.vehicle_type == VehicleType.MOTORCYCLE:
            return True

        # Car needs REGULAR or LARGE
        if vehicle.vehicle_type == VehicleType.CAR:
            return self.spot_type in [SpotType.REGULAR, SpotType.LARGE]

        # Truck needs LARGE
        if vehicle.vehicle_type == VehicleType.TRUCK:
            return self.spot_type == SpotType.LARGE

        return False

class Ticket:
    """Represents a parking ticket"""

    _ticket_counter = 1

    def __init__(self, vehicle: Vehicle, spot: ParkingSpot):
        self.ticket_id = Ticket._ticket_counter
        Ticket._ticket_counter += 1
        self.vehicle = vehicle
        self.spot = spot
        self.entry_time = datetime.now()

    def calculate_fee(self, hourly_rate: float = 5.0) -> float:
        """Calculate parking fee based on duration"""
        duration_hours = (datetime.now() - self.entry_time).total_seconds() / 3600
        return max(1.0, duration_hours) * hourly_rate  # Minimum 1 hour

    def get_duration(self) -> float:
        """Get parking duration in hours"""
        return (datetime.now() - self.entry_time).total_seconds() / 3600
```

#### 4.4 Implement Main System Class

```python
class ParkingLot:
    """Main parking lot system"""

    def __init__(self, name: str):
        self.name = name
        self.spots: list[ParkingSpot] = []
        self.active_tickets: dict[int, Ticket] = {}  # ticket_id -> Ticket

    def add_spot(self, spot: ParkingSpot):
        """Add a parking spot to the lot"""
        self.spots.append(spot)

    def park_vehicle(self, vehicle: Vehicle) -> Optional[Ticket]:
        """
        Park a vehicle and return a ticket.
        Returns None if no suitable spot available.
        """
        # Find first available suitable spot
        for spot in self.spots:
            if spot.park_vehicle(vehicle):
                ticket = Ticket(vehicle, spot)
                self.active_tickets[ticket.ticket_id] = ticket
                return ticket

        return None  # No spot available

    def remove_vehicle(self, ticket: Ticket) -> Optional[float]:
        """
        Remove vehicle using ticket and return parking fee.
        Returns None if ticket invalid.
        """
        if ticket.ticket_id not in self.active_tickets:
            return None

        # Calculate fee before removing
        fee = ticket.calculate_fee()

        # Remove vehicle from spot
        ticket.spot.remove_vehicle()

        # Remove ticket from active tickets
        del self.active_tickets[ticket.ticket_id]

        return fee

    def get_available_spots(self, spot_type: Optional[SpotType] = None) -> int:
        """Get count of available spots, optionally filtered by type"""
        count = 0
        for spot in self.spots:
            if spot.is_available:
                if spot_type is None or spot.spot_type == spot_type:
                    count += 1
        return count

    def display_status(self):
        """Display current parking lot status"""
        print(f"\n{self.name} Status:")
        print(f"Total spots: {len(self.spots)}")
        print(f"Available spots: {self.get_available_spots()}")
        print(f"Active tickets: {len(self.active_tickets)}")
```

#### 4.5 Add Usage Example

```python
def main():
    """Demonstrate parking lot system"""

    # Create parking lot
    parking_lot = ParkingLot("Downtown Parking")

    # Add parking spots
    parking_lot.add_spot(ParkingSpot(1, SpotType.COMPACT))
    parking_lot.add_spot(ParkingSpot(2, SpotType.COMPACT))
    parking_lot.add_spot(ParkingSpot(3, SpotType.REGULAR))
    parking_lot.add_spot(ParkingSpot(4, SpotType.REGULAR))
    parking_lot.add_spot(ParkingSpot(5, SpotType.LARGE))

    # Park vehicles
    car1 = Car("ABC-123")
    motorcycle1 = Motorcycle("XYZ-789")
    truck1 = Truck("TRK-456")

    ticket1 = parking_lot.park_vehicle(car1)
    ticket2 = parking_lot.park_vehicle(motorcycle1)
    ticket3 = parking_lot.park_vehicle(truck1)

    if ticket1:
        print(f"Car parked: Ticket #{ticket1.ticket_id}")
    if ticket2:
        print(f"Motorcycle parked: Ticket #{ticket2.ticket_id}")
    if ticket3:
        print(f"Truck parked: Ticket #{ticket3.ticket_id}")

    parking_lot.display_status()

    # Remove vehicle
    if ticket1:
        fee = parking_lot.remove_vehicle(ticket1)
        print(f"\nCar removed. Fee: ${fee:.2f}")

    parking_lot.display_status()

if __name__ == "__main__":
    main()
```

### Coding Best Practices

**Do:**
- Use type hints
- Add docstrings
- Handle edge cases
- Use descriptive names
- Follow SOLID principles
- Write clean, readable code

**Don't:**
- Over-engineer
- Skip error handling
- Use magic numbers
- Write overly complex code

### Time Management

**20-minute breakdown:**
- 0-5 min: Enums, base classes, inheritance
- 5-10 min: Component classes (ParkingSpot, Ticket)
- 10-15 min: Main system class (ParkingLot)
- 15-20 min: Usage example, testing edge cases

---

## Step 5: Review & Extension (5 minutes)

### Goals

- Highlight design patterns used
- Discuss SOLID principles applied
- Handle follow-up questions
- Suggest extensions

### What to Do

#### 5.1 Design Patterns Used

**Identify patterns in your code:**

**Example: Parking Lot**

```
You: "I used several design patterns in this design:

1. **Strategy Pattern**: The _can_fit() method implements different
   strategies for different vehicle types.

2. **Singleton Pattern**: (if applicable) The ParkingLot could be
   implemented as a singleton to ensure only one instance.

3. **Factory Pattern**: (if implemented) We could add a VehicleFactory
   to create vehicles based on type.
```

#### 5.2 SOLID Principles Applied

**Explain how your design follows SOLID:**

```
You: "This design adheres to SOLID principles:

**Single Responsibility Principle (SRP):**
- ParkingSpot: Only manages a single spot
- Ticket: Only tracks parking session
- ParkingLot: Coordinates overall system

**Open/Closed Principle (OCP):**
- Easy to add new vehicle types by extending Vehicle class
- No need to modify existing code

**Liskov Substitution Principle (LSP):**
- Car, Motorcycle, Truck can substitute Vehicle base class

**Interface Segregation Principle (ISP):**
- Each class has minimal, focused interface

**Dependency Inversion Principle (DIP):**
- ParkingLot depends on Vehicle abstraction, not concrete types
```

#### 5.3 Handle Follow-up Questions

**Common Follow-ups:**

**Q: "How would you add payment processing?"**
```python
class Payment(ABC):
    @abstractmethod
    def process(self, amount: float) -> bool:
        pass

class CreditCardPayment(Payment):
    def process(self, amount: float) -> bool:
        # Process credit card payment
        return True

class CashPayment(Payment):
    def process(self, amount: float) -> bool:
        # Process cash payment
        return True

# In ParkingLot.remove_vehicle():
def remove_vehicle(self, ticket: Ticket, payment: Payment) -> bool:
    fee = ticket.calculate_fee()
    if payment.process(fee):
        # Remove vehicle
        return True
    return False
```

**Q: "How would you handle multiple floors?"**
```python
class Floor:
    def __init__(self, floor_number: int):
        self.floor_number = floor_number
        self.spots: list[ParkingSpot] = []

class ParkingLot:
    def __init__(self, name: str):
        self.name = name
        self.floors: list[Floor] = []

    def add_floor(self, floor: Floor):
        self.floors.append(floor)

    def park_vehicle(self, vehicle: Vehicle) -> Optional[Ticket]:
        for floor in self.floors:
            for spot in floor.spots:
                if spot.park_vehicle(vehicle):
                    return Ticket(vehicle, spot)
        return None
```

**Q: "How would you make this thread-safe?"**
```python
import threading

class ParkingLot:
    def __init__(self, name: str):
        self.name = name
        self.spots: list[ParkingSpot] = []
        self.active_tickets: dict[int, Ticket] = {}
        self._lock = threading.Lock()

    def park_vehicle(self, vehicle: Vehicle) -> Optional[Ticket]:
        with self._lock:
            # Thread-safe parking
            for spot in self.spots:
                if spot.park_vehicle(vehicle):
                    ticket = Ticket(vehicle, spot)
                    self.active_tickets[ticket.ticket_id] = ticket
                    return ticket
            return None
```

#### 5.4 Suggest Extensions

**Proactively mention improvements:**

```
You: "Some possible extensions:

1. **Reservation System**: Allow users to reserve spots in advance
2. **Pricing Strategies**: Different rates for different spot types or times
3. **Event System**: Observer pattern for monitoring (e.g., notify when lot full)
4. **Persistence**: Add database layer for saving/loading state
5. **Metrics**: Track utilization, revenue, average duration
6. **Display System**: Show available spots at entrance
```

### Tips for Step 5

**Do:**
- Proactively mention patterns
- Show awareness of trade-offs
- Be ready for extensions
- Demonstrate depth of knowledge

**Don't:**
- Wait for interviewer to ask about patterns
- Over-promise on extensions
- Dismiss interviewer's suggestions
- Get defensive about design choices

---

## Common Mistakes to Avoid

### Mistake 1: Jumping to Code Too Quickly

**Bad:**
```
Interviewer: "Design a parking lot"
Candidate: [Immediately starts coding]
```

**Good:**
```
Interviewer: "Design a parking lot"
Candidate: "Let me ask a few clarifying questions first..."
[Asks questions, gathers requirements, then designs]
```

### Mistake 2: Over-Engineering

**Bad:**
```python
# Adding unnecessary complexity
class ParkingLotBuilder:
    # Complex builder pattern for simple object
    pass

class ParkingLotFactory:
    # Unnecessary factory
    pass

class ParkingLotSingleton:
    # Premature optimization
    pass
```

**Good:**
```python
# Simple, clean design
class ParkingLot:
    def __init__(self, name: str):
        self.name = name
        self.spots: list[ParkingSpot] = []
```

### Mistake 3: Ignoring Edge Cases

**Bad:**
```python
def park_vehicle(self, vehicle: Vehicle) -> Ticket:
    spot = self.spots[0]  # Assumes spot exists and is available
    spot.park_vehicle(vehicle)
    return Ticket(vehicle, spot)
```

**Good:**
```python
def park_vehicle(self, vehicle: Vehicle) -> Optional[Ticket]:
    for spot in self.spots:
        if spot.park_vehicle(vehicle):
            return Ticket(vehicle, spot)
    return None  # No spot available
```

### Mistake 4: Poor Naming

**Bad:**
```python
class PL:  # Unclear abbreviation
    def p(self, v):  # What does 'p' mean?
        pass
```

**Good:**
```python
class ParkingLot:
    def park_vehicle(self, vehicle: Vehicle) -> Optional[Ticket]:
        pass
```

### Mistake 5: Not Talking Through Code

**Bad:**
[Silently codes for 20 minutes]

**Good:**
```
"I'm starting with the base Vehicle class..."
"Now implementing ParkingSpot with composition..."
"Adding validation for vehicle fit..."
[Explains while coding]
```

### Mistake 6: Ignoring Interviewer Hints

**Bad:**
```
Interviewer: "How would you handle different parking spot sizes?"
Candidate: "I don't think that's necessary."
```

**Good:**
```
Interviewer: "How would you handle different parking spot sizes?"
Candidate: "Good point! Let me add a SpotType enum and validation logic..."
```

---

## Interview Cheat Sheet

### Pre-Interview Checklist

- [ ] Review SOLID principles
- [ ] Review common design patterns
- [ ] Practice drawing UML diagrams
- [ ] Practice 10+ OOD problems
- [ ] Prepare questions to ask
- [ ] Set up whiteboard/environment

### During Interview: Quick Reference

**Step 1: Requirements (5 min)**
- Ask 5-7 clarifying questions
- List functional requirements
- Define out of scope
- Get confirmation

**Step 2: Objects (5 min)**
- Identify nouns → classes
- Identify verbs → methods
- Define relationships
- Document findings

**Step 3: Diagram (10 min)**
- Draw class boxes
- Add attributes/methods
- Show relationships
- Get feedback

**Step 4: Implementation (20 min)**
- Start with enums/simple classes
- Implement inheritance hierarchy
- Build component classes
- Create main system class
- Add usage example

**Step 5: Review (5 min)**
- Mention design patterns
- Discuss SOLID principles
- Handle follow-ups
- Suggest extensions

### Time Checkpoints

- **10 min:** Should have requirements and objects identified
- **20 min:** Should have class diagram complete
- **40 min:** Should have code implementation complete
- **45 min:** Wrap up with review and extensions

---

## Practice Problems by Difficulty

### Beginner (Start Here)

1. **Deck of Cards** - Simple inheritance
2. **Stack/Queue** - Basic data structures
3. **Vending Machine** - State pattern
4. **ATM Machine** - State transitions

### Intermediate

5. **Parking Lot** - Composition, multiple entities
6. **Library Management** - Many-to-many relationships
7. **Elevator System** - State pattern, strategy
8. **Hotel Reservation** - Booking systems

### Advanced

9. **Chess Game** - Complex inheritance, move validation
10. **Airline Reservation** - Seat assignment, booking
11. **Online Shopping Cart** - Payment integration
12. **Social Media Platform** - Complex relationships

---

## Post-Interview Reflection

### After Each Practice/Interview

**What went well?**
- Which steps did you execute smoothly?
- What patterns did you apply correctly?
- Where did you demonstrate good design?

**What to improve?**
- Where did you spend too much/too little time?
- What concepts were unclear?
- What mistakes did you make?

**Action items:**
- Specific topics to review
- Patterns to practice more
- Time management adjustments

---

## Summary

### The 5-Step Framework

1. **Requirements Gathering (5 min)** - Ask, clarify, define scope
2. **Core Objects Identification (5 min)** - Nouns, verbs, relationships
3. **Class Diagram (10 min)** - UML, relationships, feedback
4. **Implementation (20 min)** - Clean code, SOLID, edge cases
5. **Review & Extension (5 min)** - Patterns, SOLID, follow-ups

### Keys to Success

- **Structure:** Follow the framework consistently
- **Communication:** Talk through your thinking
- **Clarity:** Use standard notation and naming
- **Flexibility:** Adapt based on interviewer feedback
- **Practice:** Do 10-20 problems before interviews

### Essential Skills

- SOLID principles (solid-principles.md)
- Design patterns (design-patterns-overview.md)
- UML diagrams (uml-diagrams.md)
- OOD vs System Design (ood-vs-system-design.md)

### Next Steps

1. Review fundamentals in `00-fundamentals/`
2. Practice beginner problems in `02-beginner-problems/`
3. Study design patterns in `01-design-patterns/`
4. Do mock interviews
5. Iterate and improve

---

**Remember: The interview is a conversation, not a test. Communicate your thought process, ask questions, and adapt based on feedback. Good luck!**
