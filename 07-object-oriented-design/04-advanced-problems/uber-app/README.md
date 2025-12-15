# Uber/Ride-Sharing App - OOD Design

**Difficulty:** Advanced
**Interview Frequency:** Very High
**Key Concepts:** Real-time Matching, Geolocation, State Management, Pricing Strategy
**Companies:** Uber, Lyft, Ola, DoorDash

---

## Problem Statement

Design a ride-sharing application handling riders, drivers, trip matching, real-time tracking, fare calculation, and ratings. This bridges OOD and system design.

---

## Implementation

```python
from enum import Enum
from typing import List, Optional
from datetime import datetime
import math


class RideStatus(Enum):
    REQUESTED = "Requested"
    ACCEPTED = "Accepted"
    IN_PROGRESS = "In Progress"
    COMPLETED = "Completed"
    CANCELLED = "Cancelled"


class VehicleType(Enum):
    ECONOMY = "Economy"
    PREMIUM = "Premium"
    SUV = "SUV"


class Location:
    def __init__(self, latitude: float, longitude: float, address: str = ""):
        self.latitude = latitude
        self.longitude = longitude
        self.address = address

    def distance_to(self, other: 'Location') -> float:
        """Calculate distance in km (simplified)"""
        return math.sqrt((self.latitude - other.latitude)**2 +
                        (self.longitude - other.longitude)**2) * 111  # Approx km per degree


class User:
    def __init__(self, user_id: int, name: str, phone: str):
        self.user_id = user_id
        self.name = name
        self.phone = phone
        self.rating = 5.0


class Rider(User):
    def __init__(self, user_id: int, name: str, phone: str):
        super().__init__(user_id, name, phone)
        self.payment_method: Optional[str] = None


class Driver(User):
    def __init__(self, user_id: int, name: str, phone: str, vehicle_type: VehicleType, license_plate: str):
        super().__init__(user_id, name, phone)
        self.vehicle_type = vehicle_type
        self.license_plate = license_plate
        self.current_location: Optional[Location] = None
        self.is_available = True


class PricingStrategy:
    def calculate_fare(self, distance: float, duration_minutes: int, vehicle_type: VehicleType) -> float:
        base_fare = {
            VehicleType.ECONOMY: 2.5,
            VehicleType.PREMIUM: 5.0,
            VehicleType.SUV: 7.0
        }

        per_km = {
            VehicleType.ECONOMY: 1.0,
            VehicleType.PREMIUM: 1.5,
            VehicleType.SUV: 2.0
        }

        return base_fare[vehicle_type] + (distance * per_km[vehicle_type]) + (duration_minutes * 0.3)


class Ride:
    _ride_counter = 1

    def __init__(self, rider: Rider, pickup: Location, dropoff: Location, vehicle_type: VehicleType):
        self.ride_id = Ride._ride_counter
        Ride._ride_counter += 1
        self.rider = rider
        self.driver: Optional[Driver] = None
        self.pickup = pickup
        self.dropoff = dropoff
        self.vehicle_type = vehicle_type
        self.status = RideStatus.REQUESTED
        self.request_time = datetime.now()
        self.start_time: Optional[datetime] = None
        self.end_time: Optional[datetime] = None
        self.fare: Optional[float] = None

    def assign_driver(self, driver: Driver):
        self.driver = driver
        self.status = RideStatus.ACCEPTED
        driver.is_available = False
        print(f"✓ Driver {driver.name} assigned to Ride #{self.ride_id}")

    def start_ride(self):
        self.status = RideStatus.IN_PROGRESS
        self.start_time = datetime.now()
        print(f"✓ Ride #{self.ride_id} started")

    def complete_ride(self, pricing: PricingStrategy):
        self.status = RideStatus.COMPLETED
        self.end_time = datetime.now()

        distance = self.pickup.distance_to(self.dropoff)
        duration = (self.end_time - self.start_time).seconds // 60 if self.start_time else 0
        self.fare = pricing.calculate_fare(distance, duration, self.vehicle_type)

        if self.driver:
            self.driver.is_available = True

        print(f"✓ Ride #{self.ride_id} completed")
        print(f"  Distance: {distance:.2f} km")
        print(f"  Duration: {duration} minutes")
        print(f"  Fare: ${self.fare:.2f}")

    def cancel_ride(self):
        self.status = RideStatus.CANCELLED
        if self.driver:
            self.driver.is_available = True

    def rate_driver(self, rating: float):
        if self.driver and self.status == RideStatus.COMPLETED:
            # Update driver rating (simplified)
            self.driver.rating = (self.driver.rating + rating) / 2

    def __str__(self) -> str:
        driver_name = self.driver.name if self.driver else "Unassigned"
        return (f"Ride #{self.ride_id}: {self.rider.name} → {driver_name} - "
                f"{self.status.value}")


class RideMatchingService:
    def find_nearest_driver(self, pickup: Location, drivers: List[Driver], vehicle_type: VehicleType) -> Optional[Driver]:
        available_drivers = [
            d for d in drivers
            if d.is_available and d.vehicle_type == vehicle_type and d.current_location
        ]

        if not available_drivers:
            return None

        return min(available_drivers, key=lambda d: d.current_location.distance_to(pickup))


class UberSystem:
    def __init__(self):
        self.riders: List[Rider] = []
        self.drivers: List[Driver] = []
        self.rides: List[Ride] = []
        self.matching_service = RideMatchingService()
        self.pricing = PricingStrategy()

    def register_rider(self, rider: Rider):
        self.riders.append(rider)

    def register_driver(self, driver: Driver):
        self.drivers.append(driver)

    def request_ride(self, rider: Rider, pickup: Location, dropoff: Location, vehicle_type: VehicleType) -> Optional[Ride]:
        ride = Ride(rider, pickup, dropoff, vehicle_type)
        self.rides.append(ride)

        # Find and assign driver
        driver = self.matching_service.find_nearest_driver(pickup, self.drivers, vehicle_type)
        if driver:
            ride.assign_driver(driver)
            return ride
        else:
            print("No drivers available")
            return None

    def start_ride(self, ride_id: int):
        ride = next((r for r in self.rides if r.ride_id == ride_id), None)
        if ride:
            ride.start_ride()

    def complete_ride(self, ride_id: int):
        ride = next((r for r in self.rides if r.ride_id == ride_id), None)
        if ride:
            ride.complete_ride(self.pricing)


def main():
    system = UberSystem()

    # Register drivers
    driver1 = Driver(1, "John", "555-0001", VehicleType.ECONOMY, "ABC123")
    driver1.current_location = Location(37.7749, -122.4194)  # SF
    system.register_driver(driver1)

    driver2 = Driver(2, "Sarah", "555-0002", VehicleType.PREMIUM, "XYZ789")
    driver2.current_location = Location(37.7849, -122.4094)
    system.register_driver(driver2)

    # Register rider
    rider = Rider(101, "Alice", "555-1001")
    system.register_rider(rider)

    # Request ride
    pickup = Location(37.7750, -122.4195, "123 Main St")
    dropoff = Location(37.8044, -122.2712, "456 Oak Ave")

    print("--- Requesting Ride ---")
    ride = system.request_ride(rider, pickup, dropoff, VehicleType.ECONOMY)

    if ride:
        # Start and complete ride
        print("\n--- Starting Ride ---")
        system.start_ride(ride.ride_id)

        print("\n--- Completing Ride ---")
        system.complete_ride(ride.ride_id)

        # Rate driver
        ride.rate_driver(4.5)
        print(f"\nDriver rating: {ride.driver.rating:.2f}")


if __name__ == "__main__":
    main()
```

---

## Design Patterns
- **Strategy Pattern:** Pricing strategies (surge, flat rate)
- **Observer Pattern:** Real-time location updates
- **State Pattern:** Ride status transitions
- **Factory Pattern:** Creating different ride types

## System Design Overlap
- Geospatial indexing for driver matching
- Real-time tracking with WebSockets
- Surge pricing algorithm
- Driver-rider matching optimization

## SOLID Principles
- SRP: Separate matching, pricing, and ride management
- Strategy for different pricing models
- Open for extension (new vehicle types)

This tests real-world app design with location services, matching algorithms, and dynamic pricing.
