# Airline Reservation System - OOD Design

**Difficulty:** Intermediate
**Interview Frequency:** High
**Key Concepts:** Seat Assignment, Pricing Strategy, Booking Management
**Companies:** Airlines, Travel Sites, Expedia, Google Flights

---

## Problem Statement

Design an airline reservation system handling flights, seat assignments, bookings, and pricing with support for different fare classes.

---

## Implementation

```python
from enum import Enum
from datetime import datetime
from typing import List, Optional


class SeatClass(Enum):
    ECONOMY = "Economy"
    BUSINESS = "Business"
    FIRST_CLASS = "First Class"


class SeatStatus(Enum):
    AVAILABLE = "Available"
    BOOKED = "Booked"
    BLOCKED = "Blocked"


class Seat:
    def __init__(self, seat_number: str, seat_class: SeatClass, price: float):
        self.seat_number = seat_number
        self.seat_class = seat_class
        self.price = price
        self.status = SeatStatus.AVAILABLE

    def book(self) -> bool:
        if self.status == SeatStatus.AVAILABLE:
            self.status = SeatStatus.BOOKED
            return True
        return False

    def release(self):
        self.status = SeatStatus.AVAILABLE


class Passenger:
    def __init__(self, passenger_id: int, name: str, email: str, passport: str):
        self.passenger_id = passenger_id
        self.name = name
        self.email = email
        self.passport = passport


class Flight:
    def __init__(self, flight_number: str, origin: str, destination: str, departure: datetime, duration_minutes: int):
        self.flight_number = flight_number
        self.origin = origin
        self.destination = destination
        self.departure = departure
        self.duration_minutes = duration_minutes
        self.seats: List[Seat] = []

    def add_seat(self, seat: Seat):
        self.seats.append(seat)

    def get_available_seats(self, seat_class: Optional[SeatClass] = None) -> List[Seat]:
        available = [s for s in self.seats if s.status == SeatStatus.AVAILABLE]
        if seat_class:
            available = [s for s in available if s.seat_class == seat_class]
        return available

    def __str__(self) -> str:
        return f"Flight {self.flight_number}: {self.origin} → {self.destination} at {self.departure.strftime('%Y-%m-%d %H:%M')}"


class Reservation:
    _reservation_counter = 1

    def __init__(self, passenger: Passenger, flight: Flight, seat: Seat):
        self.reservation_id = f"RES{Reservation._reservation_counter:06d}"
        Reservation._reservation_counter += 1
        self.passenger = passenger
        self.flight = flight
        self.seat = seat
        self.booking_time = datetime.now()
        self.price = seat.price

    def __str__(self) -> str:
        return (f"Reservation {self.reservation_id}: {self.passenger.name} - "
                f"{self.flight.flight_number} Seat {self.seat.seat_number} ({self.seat.seat_class.value}) - ${self.price:.2f}")


class ReservationSystem:
    def __init__(self):
        self.flights: List[Flight] = []
        self.reservations: List[Reservation] = []

    def add_flight(self, flight: Flight):
        self.flights.append(flight)

    def search_flights(self, origin: str, destination: str, date: datetime) -> List[Flight]:
        return [
            f for f in self.flights
            if f.origin == origin and f.destination == destination and f.departure.date() == date.date()
        ]

    def book_seat(self, passenger: Passenger, flight: Flight, seat_number: str) -> Optional[Reservation]:
        seat = next((s for s in flight.seats if s.seat_number == seat_number), None)

        if not seat:
            print(f"Seat {seat_number} not found")
            return None

        if not seat.book():
            print(f"Seat {seat_number} not available")
            return None

        reservation = Reservation(passenger, flight, seat)
        self.reservations.append(reservation)
        print(f"✓ {reservation}")
        return reservation

    def cancel_reservation(self, reservation_id: str) -> bool:
        reservation = next((r for r in self.reservations if r.reservation_id == reservation_id), None)
        if reservation:
            reservation.seat.release()
            self.reservations.remove(reservation)
            print(f"✓ Reservation {reservation_id} cancelled")
            return True
        return False


def main():
    system = ReservationSystem()

    # Create flight
    flight = Flight("AA123", "NYC", "LAX", datetime(2024, 12, 25, 8, 0), 360)

    # Add seats
    for i in range(1, 4):
        flight.add_seat(Seat(f"1{i}", SeatClass.FIRST_CLASS, 800.0))
    for i in range(1, 11):
        flight.add_seat(Seat(f"2{i}", SeatClass.BUSINESS, 400.0))
    for i in range(1, 31):
        flight.add_seat(Seat(f"3{i}", SeatClass.ECONOMY, 200.0))

    system.add_flight(flight)

    # Search flights
    print(f"\nSearching flights NYC → LAX on 2024-12-25")
    flights = system.search_flights("NYC", "LAX", datetime(2024, 12, 25))
    for f in flights:
        print(f"  {f}")
        available = f.get_available_seats()
        print(f"    Available seats: {len(available)}")

    # Create passengers and book
    passenger1 = Passenger(1, "Alice", "alice@email.com", "P123456")
    passenger2 = Passenger(2, "Bob", "bob@email.com", "P789012")

    print("\n--- Bookings ---")
    reservation1 = system.book_seat(passenger1, flight, "11")
    reservation2 = system.book_seat(passenger2, flight, "21")

    # Try duplicate booking
    reservation3 = system.book_seat(passenger2, flight, "11")


if __name__ == "__main__":
    main()
```

---

## Design Patterns
- **Strategy Pattern:** Pricing strategies for different classes
- **Factory Pattern:** Seat creation
- **State Pattern:** Seat status management

## Extensions
- Add baggage allowance
- Implement waitlist
- Support connecting flights
- Add meal preferences

This tests seat assignment logic and booking conflict handling.
