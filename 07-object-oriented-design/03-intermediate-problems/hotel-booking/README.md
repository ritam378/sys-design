# Hotel Booking System - OOD Design

**Difficulty:** Intermediate
**Interview Frequency:** High
**Key Concepts:** Booking Management, Search Filters, Pricing Strategy
**Companies:** Booking.com, Airbnb, Expedia, Google

---

## Problem Statement

Design a hotel booking system that manages rooms, reservations, pricing, and availability across multiple hotels.

---

## Implementation

```python
from enum import Enum
from datetime import datetime, timedelta
from typing import List, Optional


class RoomType(Enum):
    SINGLE = "Single"
    DOUBLE = "Double"
    SUITE = "Suite"


class RoomStatus(Enum):
    AVAILABLE = "Available"
    BOOKED = "Booked"
    MAINTENANCE = "Maintenance"


class Room:
    def __init__(self, room_number: str, room_type: RoomType, price_per_night: float):
        self.room_number = room_number
        self.room_type = room_type
        self.price_per_night = price_per_night
        self.status = RoomStatus.AVAILABLE

    def is_available(self, check_in: datetime, check_out: datetime, bookings: List['Booking']) -> bool:
        if self.status == RoomStatus.MAINTENANCE:
            return False

        for booking in bookings:
            if booking.room == self and not (
                check_out <= booking.check_in or check_in >= booking.check_out
            ):
                return False
        return True


class Guest:
    def __init__(self, guest_id: int, name: str, email: str, phone: str):
        self.guest_id = guest_id
        self.name = name
        self.email = email
        self.phone = phone


class Booking:
    _booking_counter = 1

    def __init__(self, guest: Guest, room: Room, check_in: datetime, check_out: datetime):
        self.booking_id = Booking._booking_counter
        Booking._booking_counter += 1
        self.guest = guest
        self.room = room
        self.check_in = check_in
        self.check_out = check_out
        self.total_price = self._calculate_price()

    def _calculate_price(self) -> float:
        nights = (self.check_out - self.check_in).days
        return nights * self.room.price_per_night

    def get_nights(self) -> int:
        return (self.check_out - self.check_in).days

    def __str__(self) -> str:
        return (f"Booking #{self.booking_id}: {self.guest.name} - "
                f"Room {self.room.room_number} ({self.room.room_type.value}) - "
                f"{self.get_nights()} nights - ${self.total_price:.2f}")


class Hotel:
    def __init__(self, hotel_id: int, name: str, location: str):
        self.hotel_id = hotel_id
        self.name = name
        self.location = location
        self.rooms: List[Room] = []
        self.bookings: List[Booking] = []

    def add_room(self, room: Room):
        self.rooms.append(room)

    def search_available_rooms(
        self,
        check_in: datetime,
        check_out: datetime,
        room_type: Optional[RoomType] = None
    ) -> List[Room]:
        available = []
        for room in self.rooms:
            if room_type and room.room_type != room_type:
                continue
            if room.is_available(check_in, check_out, self.bookings):
                available.append(room)
        return available

    def book_room(
        self,
        guest: Guest,
        room_number: str,
        check_in: datetime,
        check_out: datetime
    ) -> Optional[Booking]:
        # Find room
        room = next((r for r in self.rooms if r.room_number == room_number), None)
        if not room:
            print(f"Room {room_number} not found")
            return None

        # Check availability
        if not room.is_available(check_in, check_out, self.bookings):
            print(f"Room {room_number} not available for selected dates")
            return None

        # Create booking
        booking = Booking(guest, room, check_in, check_out)
        self.bookings.append(booking)
        print(f"✓ {booking}")
        return booking

    def cancel_booking(self, booking_id: int) -> bool:
        booking = next((b for b in self.bookings if b.booking_id == booking_id), None)
        if booking:
            self.bookings.remove(booking)
            print(f"✓ Booking #{booking_id} cancelled")
            return True
        return False


def main():
    # Create hotel
    hotel = Hotel(1, "Grand Plaza Hotel", "New York")

    # Add rooms
    hotel.add_room(Room("101", RoomType.SINGLE, 100.0))
    hotel.add_room(Room("102", RoomType.SINGLE, 100.0))
    hotel.add_room(Room("201", RoomType.DOUBLE, 150.0))
    hotel.add_room(Room("202", RoomType.DOUBLE, 150.0))
    hotel.add_room(Room("301", RoomType.SUITE, 300.0))

    # Create guest
    guest1 = Guest(1, "Alice Johnson", "alice@email.com", "555-1234")
    guest2 = Guest(2, "Bob Smith", "bob@email.com", "555-5678")

    # Search available rooms
    check_in = datetime.now() + timedelta(days=7)
    check_out = check_in + timedelta(days=3)

    print(f"\nSearching for available rooms from {check_in.date()} to {check_out.date()}")
    available = hotel.search_available_rooms(check_in, check_out, RoomType.DOUBLE)
    print(f"Found {len(available)} available double rooms:")
    for room in available:
        print(f"  Room {room.room_number} - ${room.price_per_night}/night")

    # Book room
    print("\n--- Making Bookings ---")
    booking1 = hotel.book_room(guest1, "201", check_in, check_out)
    booking2 = hotel.book_room(guest2, "301", check_in, check_out)

    # Try to book already booked room
    print("\n--- Attempting Double Booking ---")
    booking3 = hotel.book_room(guest2, "201", check_in, check_out)

    # Cancel booking
    print("\n--- Cancellation ---")
    if booking1:
        hotel.cancel_booking(booking1.booking_id)


if __name__ == "__main__":
    main()
```

---

## Design Patterns
- **Strategy Pattern:** Pricing strategies (seasonal, dynamic)
- **Factory Pattern:** Room creation
- **Observer Pattern:** Booking notifications

## Extensions
- Add payment processing
- Implement room amenities filtering
- Add hotel ratings and reviews
- Support multiple hotels in chain

This tests availability checking, date range handling, and booking conflict resolution.
