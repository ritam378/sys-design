# Parking Lot System - OOD Design

> **Difficulty:** Beginner-Intermediate
> **Interview Frequency:** Very High
> **Key Concepts:** State pattern, Strategy pattern, Enum, Composition
> **Companies:** Amazon, Google, Microsoft, Uber

## Problem Statement

Design a parking lot system that can:
1. Park vehicles of different types (Motorcycle, Car, Bus)
2. Have multiple floors and spots per floor
3. Charge different rates based on vehicle type and duration
4. Track available spots
5. Issue and validate parking tickets

## Requirements Gathering (Interview Step 1)

### Clarifying Questions to Ask

**Q: What types of vehicles should we support?**
A: Motorcycle, Car, Bus

**Q: Do different vehicles take different sized spots?**
A: Yes. Motorcycle = 1 spot, Car = 1 spot, Bus = 5 spots

**Q: How many floors? How many spots per floor?**
A: Multiple floors (configurable), 100+ spots per floor

**Q: How is pricing calculated?**
A: Hourly rate varies by vehicle type

**Q: Can we assume unlimited parking lot size?**
A: No, must handle "lot full" scenario

**Q: Should we handle reservations?**
A: No, first-come-first-served

### Functional Requirements

✅ Park vehicle - assign spot, issue ticket
✅ Unpark vehicle - calculate fee, free up spot
✅ Track available spots per floor
✅ Support multiple vehicle types with different spot requirements
✅ Calculate parking fee based on duration and vehicle type
✅ Handle "parking lot full" scenario

### Non-Functional Requirements

- Handle concurrent parking requests (thread-safe)
- Fast spot allocation (< 100ms)
- Scalable to 1000+ spots

### Out of Scope

❌ Online reservation system
❌ Payment processing integration
❌ Valet parking
❌ Electric vehicle charging

---

## Core Objects Identification (Interview Step 2)

**Nouns (Potential Classes):**
- ParkingLot
- Floor
- Spot (ParkingSpot)
- Vehicle (Motorcycle, Car, Bus)
- Ticket
- Payment

**Verbs (Potential Methods):**
- park(vehicle)
- unpark(ticket)
- findAvailableSpot(vehicle)
- calculateFee(ticket)

**Relationships:**
- ParkingLot HAS-A multiple Floors
- Floor HAS-A multiple Spots
- Spot HAS-A Vehicle (when occupied)
- Ticket REFERENCES Spot and Vehicle

---

## Class Diagram (Interview Step 3)

```
┌─────────────────────────┐
│     ParkingLot          │
├─────────────────────────┤
│ - floors: List[Floor]   │
│ - name: str             │
├─────────────────────────┤
│ + park(vehicle)         │
│ + unpark(ticket)        │
│ + getAvailableSpots()   │
└────────┬────────────────┘
         │
         │ has-a (composition)
         │
┌────────▼────────────────┐
│       Floor             │
├─────────────────────────┤
│ - spots: List[Spot]     │
│ - floor_number: int     │
├─────────────────────────┤
│ + findAvailableSpot()   │
│ + getAvailableCount()   │
└────────┬────────────────┘
         │
         │ has-a
         │
┌────────▼────────────────┐
│      ParkingSpot        │
├─────────────────────────┤
│ - spot_number: int      │
│ - spot_type: SpotType   │
│ - vehicle: Vehicle      │
│ - is_available: bool    │
├─────────────────────────┤
│ + park(vehicle)         │
│ + unpark()              │
│ + isAvailable()         │
└─────────────────────────┘

┌─────────────────────────┐
│       Vehicle           │◁──────────────┐
├─────────────────────────┤               │
│ - plate: str            │               │ is-a
│ - type: VehicleType     │               │
└─────────────────────────┘               │
         △                                 │
         │                                 │
    ┌────┴────┬─────────┬─────────┐      │
    │         │         │         │      │
┌───▼──┐  ┌──▼───┐  ┌──▼──┐  ┌───▼──────▼────┐
│ Bike │  │ Car  │  │ Bus │  │  VehicleType   │
└──────┘  └──────┘  └─────┘  │  (Enum)        │
                              ├────────────────┤
                              │ MOTORCYCLE     │
                              │ CAR            │
                              │ BUS            │
                              └────────────────┘

┌─────────────────────────┐
│      ParkingTicket      │
├─────────────────────────┤
│ - ticket_id: str        │
│ - vehicle: Vehicle      │
│ - spot: ParkingSpot     │
│ - entry_time: datetime  │
│ - exit_time: datetime   │
├─────────────────────────┤
│ + calculateFee()        │
└─────────────────────────┘
```

---

## Python Implementation (Interview Step 4)

### Step 1: Define Enums and Basic Classes

```python
from enum import Enum
from datetime import datetime, timedelta
from typing import List, Optional
import uuid


class VehicleType(Enum):
    """Enum for vehicle types"""
    MOTORCYCLE = 1
    CAR = 2
    BUS = 3


class SpotType(Enum):
    """Enum for parking spot types"""
    MOTORCYCLE = 1  # Can hold motorcycle
    COMPACT = 2     # Can hold motorcycle, car
    LARGE = 3       # Can hold motorcycle, car, bus


class Vehicle:
    """Base class for all vehicles"""
    def __init__(self, plate: str, vehicle_type: VehicleType):
        self.plate = plate
        self.type = vehicle_type

    def __str__(self):
        return f"{self.type.name} ({self.plate})"


class Motorcycle(Vehicle):
    def __init__(self, plate: str):
        super().__init__(plate, VehicleType.MOTORCYCLE)


class Car(Vehicle):
    def __init__(self, plate: str):
        super().__init__(plate, VehicleType.CAR)


class Bus(Vehicle):
    def __init__(self, plate: str):
        super().__init__(plate, VehicleType.BUS)
```

### Step 2: Parking Spot Implementation

```python
class ParkingSpot:
    """Represents a single parking spot"""

    def __init__(self, spot_number: int, spot_type: SpotType):
        self.spot_number = spot_number
        self.spot_type = spot_type
        self.vehicle: Optional[Vehicle] = None
        self.is_available = True

    def can_fit_vehicle(self, vehicle: Vehicle) -> bool:
        """Check if this spot can accommodate the vehicle"""
        # Motorcycle spots: only motorcycles
        if self.spot_type == SpotType.MOTORCYCLE:
            return vehicle.type == VehicleType.MOTORCYCLE

        # Compact spots: motorcycles and cars
        elif self.spot_type == SpotType.COMPACT:
            return vehicle.type in [VehicleType.MOTORCYCLE, VehicleType.CAR]

        # Large spots: all vehicles
        elif self.spot_type == SpotType.LARGE:
            return True

        return False

    def park_vehicle(self, vehicle: Vehicle) -> bool:
        """Park a vehicle in this spot"""
        if not self.is_available:
            return False

        if not self.can_fit_vehicle(vehicle):
            return False

        self.vehicle = vehicle
        self.is_available = False
        return True

    def unpark_vehicle(self) -> Optional[Vehicle]:
        """Remove vehicle from spot"""
        if self.is_available:
            return None

        vehicle = self.vehicle
        self.vehicle = None
        self.is_available = True
        return vehicle

    def __str__(self):
        status = "Available" if self.is_available else f"Occupied by {self.vehicle}"
        return f"Spot #{self.spot_number} ({self.spot_type.name}): {status}"
```

### Step 3: Floor Implementation

```python
class Floor:
    """Represents a parking floor with multiple spots"""

    def __init__(self, floor_number: int):
        self.floor_number = floor_number
        self.spots: List[ParkingSpot] = []

    def add_spot(self, spot: ParkingSpot):
        """Add a parking spot to this floor"""
        self.spots.append(spot)

    def find_available_spot(self, vehicle: Vehicle) -> Optional[ParkingSpot]:
        """Find first available spot that can fit the vehicle"""
        for spot in self.spots:
            if spot.is_available and spot.can_fit_vehicle(vehicle):
                return spot
        return None

    def get_available_count(self) -> dict:
        """Get count of available spots by type"""
        counts = {
            SpotType.MOTORCYCLE: 0,
            SpotType.COMPACT: 0,
            SpotType.LARGE: 0
        }

        for spot in self.spots:
            if spot.is_available:
                counts[spot.spot_type] += 1

        return counts

    def __str__(self):
        available = sum(1 for spot in self.spots if spot.is_available)
        total = len(self.spots)
        return f"Floor {self.floor_number}: {available}/{total} spots available"
```

### Step 4: Parking Ticket Implementation

```python
class ParkingTicket:
    """Represents a parking ticket issued when vehicle enters"""

    # Hourly rates by vehicle type
    HOURLY_RATES = {
        VehicleType.MOTORCYCLE: 2.0,
        VehicleType.CAR: 5.0,
        VehicleType.BUS: 10.0
    }

    def __init__(self, vehicle: Vehicle, spot: ParkingSpot):
        self.ticket_id = str(uuid.uuid4())[:8]
        self.vehicle = vehicle
        self.spot = spot
        self.entry_time = datetime.now()
        self.exit_time: Optional[datetime] = None

    def calculate_fee(self) -> float:
        """Calculate parking fee based on duration and vehicle type"""
        if not self.exit_time:
            self.exit_time = datetime.now()

        duration = self.exit_time - self.entry_time
        hours = duration.total_seconds() / 3600

        # Round up to nearest hour
        hours_ceil = int(hours) + (1 if hours % 1 > 0 else 0)

        rate = self.HOURLY_RATES[self.vehicle.type]
        fee = hours_ceil * rate

        return fee

    def __str__(self):
        return (f"Ticket #{self.ticket_id}\n"
                f"Vehicle: {self.vehicle}\n"
                f"Spot: Floor {self.spot.floor_number if hasattr(self.spot, 'floor_number') else 'N/A'}, "
                f"Spot #{self.spot.spot_number}\n"
                f"Entry: {self.entry_time.strftime('%Y-%m-%d %H:%M:%S')}")
```

### Step 5: Parking Lot Implementation (Main Class)

```python
class ParkingLot:
    """Main parking lot class that manages all floors and spots"""

    _instance = None  # Singleton pattern

    def __new__(cls, name: str):
        """Singleton: only one parking lot instance"""
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.name = name
            cls._instance.floors = []
            cls._instance.active_tickets = {}  # ticket_id -> ParkingTicket
        return cls._instance

    def add_floor(self, floor: Floor):
        """Add a floor to the parking lot"""
        self.floors.append(floor)

    def park_vehicle(self, vehicle: Vehicle) -> Optional[ParkingTicket]:
        """
        Park a vehicle in the first available spot

        Returns ParkingTicket if successful, None if lot is full
        """
        # Find available spot across all floors
        for floor in self.floors:
            spot = floor.find_available_spot(vehicle)
            if spot:
                # Park the vehicle
                if spot.park_vehicle(vehicle):
                    # Issue ticket
                    ticket = ParkingTicket(vehicle, spot)
                    self.active_tickets[ticket.ticket_id] = ticket

                    print(f"✅ Parked {vehicle} at Floor {floor.floor_number}, Spot #{spot.spot_number}")
                    print(f"Ticket ID: {ticket.ticket_id}")
                    return ticket

        print(f"❌ Sorry, parking lot is full. Cannot park {vehicle}")
        return None

    def unpark_vehicle(self, ticket_id: str) -> Optional[float]:
        """
        Unpark a vehicle using ticket ID

        Returns parking fee if successful, None if ticket not found
        """
        if ticket_id not in self.active_tickets:
            print(f"❌ Invalid ticket ID: {ticket_id}")
            return None

        ticket = self.active_tickets[ticket_id]

        # Unpark vehicle from spot
        vehicle = ticket.spot.unpark_vehicle()

        # Calculate fee
        fee = ticket.calculate_fee()

        # Remove ticket from active tickets
        del self.active_tickets[ticket_id]

        print(f"✅ Unparked {vehicle}")
        print(f"Duration: {ticket.exit_time - ticket.entry_time}")
        print(f"Parking Fee: ${fee:.2f}")

        return fee

    def display_availability(self):
        """Display available spots on each floor"""
        print(f"\n{'='*50}")
        print(f"Parking Lot: {self.name}")
        print(f"{'='*50}")

        for floor in self.floors:
            counts = floor.get_available_count()
            print(f"\n{floor}")
            print(f"  Motorcycle spots: {counts[SpotType.MOTORCYCLE]}")
            print(f"  Compact spots: {counts[SpotType.COMPACT]}")
            print(f"  Large spots: {counts[SpotType.LARGE]}")

        total_available = sum(
            sum(floor.get_available_count().values())
            for floor in self.floors
        )
        total_spots = sum(len(floor.spots) for floor in self.floors)

        print(f"\nTotal: {total_available}/{total_spots} spots available")
        print(f"{'='*50}\n")
```

---

## Complete Usage Example

```python
def main():
    """Demonstrate parking lot system"""

    # Create parking lot (Singleton)
    parking_lot = ParkingLot("Downtown Parking")

    # Create floors
    floor1 = Floor(1)
    floor2 = Floor(2)

    # Add spots to Floor 1
    # 10 motorcycle spots
    for i in range(1, 11):
        floor1.add_spot(ParkingSpot(i, SpotType.MOTORCYCLE))

    # 20 compact spots
    for i in range(11, 31):
        floor1.add_spot(ParkingSpot(i, SpotType.COMPACT))

    # 20 large spots
    for i in range(31, 51):
        floor1.add_spot(ParkingSpot(i, SpotType.LARGE))

    # Add spots to Floor 2 (similar distribution)
    for i in range(1, 11):
        floor2.add_spot(ParkingSpot(i, SpotType.MOTORCYCLE))
    for i in range(11, 31):
        floor2.add_spot(ParkingSpot(i, SpotType.COMPACT))
    for i in range(31, 51):
        floor2.add_spot(ParkingSpot(i, SpotType.LARGE))

    # Add floors to parking lot
    parking_lot.add_floor(floor1)
    parking_lot.add_floor(floor2)

    # Display initial availability
    parking_lot.display_availability()

    # Park some vehicles
    print("\n--- Parking Vehicles ---\n")

    bike1 = Motorcycle("BIKE-001")
    ticket1 = parking_lot.park_vehicle(bike1)

    car1 = Car("CAR-001")
    ticket2 = parking_lot.park_vehicle(car1)

    bus1 = Bus("BUS-001")
    ticket3 = parking_lot.park_vehicle(bus1)

    car2 = Car("CAR-002")
    ticket4 = parking_lot.park_vehicle(car2)

    # Display availability after parking
    parking_lot.display_availability()

    # Simulate some time passing
    import time
    print("\n--- Waiting 2 seconds (simulating parking duration) ---\n")
    time.sleep(2)

    # Unpark vehicles
    print("\n--- Unparking Vehicles ---\n")

    if ticket1:
        parking_lot.unpark_vehicle(ticket1.ticket_id)

    if ticket3:
        parking_lot.unpark_vehicle(ticket3.ticket_id)

    # Display final availability
    parking_lot.display_availability()

    # Try to unpark with invalid ticket
    print("\n--- Testing Invalid Ticket ---\n")
    parking_lot.unpark_vehicle("INVALID-TICKET")


if __name__ == "__main__":
    main()
```

### Expected Output:

```
==================================================
Parking Lot: Downtown Parking
==================================================

Floor 1: 50/50 spots available
  Motorcycle spots: 10
  Compact spots: 20
  Large spots: 20

Floor 2: 50/50 spots available
  Motorcycle spots: 10
  Compact spots: 20
  Large spots: 20

Total: 100/100 spots available
==================================================

--- Parking Vehicles ---

✅ Parked MOTORCYCLE (BIKE-001) at Floor 1, Spot #1
Ticket ID: a7b3c9e1
✅ Parked CAR (CAR-001) at Floor 1, Spot #11
Ticket ID: f2d8e4a6
✅ Parked BUS (BUS-001) at Floor 1, Spot #31
Ticket ID: c5b9f3d2
✅ Parked CAR (CAR-002) at Floor 1, Spot #12
Ticket ID: e8a4c1f7

==================================================
Parking Lot: Downtown Parking
==================================================

Floor 1: 46/50 spots available
  Motorcycle spots: 9
  Compact spots: 18
  Large spots: 19

Floor 2: 50/50 spots available
  Motorcycle spots: 10
  Compact spots: 20
  Large spots: 20

Total: 96/100 spots available
==================================================

--- Waiting 2 seconds (simulating parking duration) ---

--- Unparking Vehicles ---

✅ Unparked MOTORCYCLE (BIKE-001)
Duration: 0:00:02.001234
Parking Fee: $2.00
✅ Unparked BUS (BUS-001)
Duration: 0:00:02.001456
Parking Fee: $10.00

==================================================
Parking Lot: Downtown Parking
==================================================

Floor 1: 48/50 spots available
  Motorcycle spots: 10
  Compact spots: 18
  Large spots: 20

Floor 2: 50/50 spots available
  Motorcycle spots: 10
  Compact spots: 20
  Large spots: 20

Total: 98/100 spots available
==================================================

--- Testing Invalid Ticket ---

❌ Invalid ticket ID: INVALID-TICKET
```

---

## Design Patterns Used

### 1. **Singleton Pattern**
```python
class ParkingLot:
    _instance = None
    def __new__(cls, name: str):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```
**Why:** Only one parking lot should exist

### 2. **Strategy Pattern**
```python
# Different spot allocation strategies could be implemented
class SpotAllocationStrategy(ABC):
    @abstractmethod
    def find_spot(self, vehicle): pass

class FirstAvailableStrategy(SpotAllocationStrategy):
    def find_spot(self, vehicle): ...

class ClosestToEntranceStrategy(SpotAllocationStrategy):
    def find_spot(self, vehicle): ...
```

### 3. **State Pattern** (Implicit)
```python
# ParkingSpot has states: available/occupied
spot.is_available = True   # Available state
spot.is_available = False  # Occupied state
```

### 4. **Enum Pattern**
```python
class VehicleType(Enum):
    MOTORCYCLE = 1
    CAR = 2
    BUS = 3
```
**Why:** Type-safe vehicle and spot categories

---

## SOLID Principles Applied

**Single Responsibility:**
- `ParkingSpot`: Manages single spot
- `Floor`: Manages collection of spots
- `ParkingTicket`: Handles ticketing and billing
- `ParkingLot`: Orchestrates parking operations

**Open/Closed:**
- Easy to add new `VehicleType` (e.g., `TRUCK`)
- Easy to add new `SpotType` (e.g., `HANDICAPPED`)
- Can extend with new features without modifying existing classes

**Liskov Substitution:**
- `Motorcycle`, `Car`, `Bus` are substitutable for `Vehicle`

**Interface Segregation:**
- Each class has focused interface (no bloated interfaces)

**Dependency Inversion:**
- Could inject pricing strategy instead of hardcoding rates

---

## Extensions and Follow-up Questions

### Q1: How would you handle handicapped parking?

**Answer:**
```python
class SpotType(Enum):
    MOTORCYCLE = 1
    COMPACT = 2
    LARGE = 3
    HANDICAPPED = 4  # New type

# ParkingSpot needs handicapped permit check
class ParkingSpot:
    def can_fit_vehicle(self, vehicle: Vehicle) -> bool:
        if self.spot_type == SpotType.HANDICAPPED:
            return vehicle.has_handicapped_permit  # New vehicle attribute
        # ... rest of logic
```

### Q2: How would you implement reservations?

**Answer:**
```python
class Reservation:
    def __init__(self, vehicle, start_time, duration):
        self.vehicle = vehicle
        self.start_time = start_time
        self.duration = duration
        self.spot = None

class ParkingSpot:
    def __init__(self, ...):
        # ...
        self.reservation = None  # Current reservation

    def is_reserved(self, current_time) -> bool:
        if self.reservation:
            return (self.reservation.start_time <= current_time <
                    self.reservation.start_time + self.reservation.duration)
        return False
```

### Q3: How would you make it thread-safe for concurrent parking?

**Answer:**
```python
import threading

class ParkingLot:
    def __init__(self, name):
        # ...
        self.lock = threading.Lock()

    def park_vehicle(self, vehicle):
        with self.lock:
            # Thread-safe parking logic
            for floor in self.floors:
                spot = floor.find_available_spot(vehicle)
                if spot:
                    # Atomic check and park
                    return spot.park_vehicle(vehicle)
```

### Q4: How would you implement different pricing strategies?

**Answer:**
```python
class PricingStrategy(ABC):
    @abstractmethod
    def calculate_fee(self, ticket): pass

class HourlyPricing(PricingStrategy):
    def calculate_fee(self, ticket):
        # Current implementation
        ...

class FlatRatePricing(PricingStrategy):
    def calculate_fee(self, ticket):
        return 10.0  # Flat rate

class PeakHourPricing(PricingStrategy):
    def calculate_fee(self, ticket):
        # Higher rates during peak hours
        ...

class ParkingTicket:
    def __init__(self, vehicle, spot, pricing_strategy):
        self.pricing_strategy = pricing_strategy

    def calculate_fee(self):
        return self.pricing_strategy.calculate_fee(self)
```

---

## Interview Tips

### What Interviewers Look For:

✅ **Clear class design** with well-defined responsibilities
✅ **Use of OOP concepts** (inheritance, composition, encapsulation)
✅ **Design patterns** where appropriate
✅ **Edge case handling** (lot full, invalid ticket)
✅ **Extensibility** (easy to add features)
✅ **Code organization** (clean, readable)

### Common Mistakes:

❌ God class (one class doing everything)
❌ Not using enums (using strings for types)
❌ Not handling edge cases
❌ Over-engineering (unnecessary complexity)
❌ Poor naming conventions

### Time Management (45 min interview):

- **5 min:** Clarify requirements
- **5 min:** Identify core classes
- **10 min:** Design class diagram
- **20 min:** Implement 3-4 key classes
- **5 min:** Discuss extensions and trade-offs

---

## Summary

**Parking Lot** is a classic OOD problem that tests:
- Class design and relationships
- Enum usage
- Design patterns (Singleton, Strategy, State)
- SOLID principles
- Real-world system modeling

**Key Takeaways:**
1. Start with requirements and core objects
2. Use composition (HAS-A) over inheritance where appropriate
3. Apply design patterns pragmatically
4. Handle edge cases
5. Make design extensible

This problem demonstrates your ability to model real-world systems using object-oriented principles, a critical skill for software engineering roles.
