# Movie Ticket Booking System - OOD Design

**Difficulty:** Beginner-Intermediate
**Interview Frequency:** High
**Key Concepts:** Factory Pattern, State Pattern, Strategy Pattern, Concurrency
**Companies:** BookMyShow, Fandango, AMC, Ticketmaster, Amazon

---

## Problem Statement

Design a movie ticket booking system that allows users to browse movies, select showtimes, choose seats, and book tickets. The system should handle multiple theaters, concurrent bookings, seat selection, and payment processing.

**Core Features:**
1. Browse movies currently showing
2. View available showtimes for movies
3. Select seats in a theater
4. Handle concurrent seat booking (prevent double booking)
5. Process payments
6. Generate and manage tickets
7. Support different seat types and pricing

---

## 1. Requirements Gathering (Interview Step 1)

### Clarifying Questions to Ask

**Q1: What information should we track for movies and showtimes?**
A: Movies have title, duration, genre, rating. Showtimes link movies to specific theaters at specific times.

**Q2: How should we handle seat selection?**
A: Theaters have different layouts with various seat types (Regular, Premium, VIP). Users select specific seats which must be locked during booking to prevent conflicts.

**Q3: Should we support multiple theaters/locations?**
A: Yes, support multiple theaters (e.g., Theater 1, Theater 2) within a cinema complex, each with different capacities.

**Q4: How do we prevent double booking of seats?**
A: Implement seat locking mechanism - when a user selects a seat, it's temporarily locked (e.g., 10 minutes) to complete payment.

**Q5: What payment methods should we support?**
A: Support multiple payment methods (credit card, debit card, mobile wallet) using a strategy pattern.

**Q6: Should we handle ticket cancellation?**
A: Yes, allow cancellation with refund based on timing (e.g., full refund if >24 hours before show).

**Q7: Do we need user authentication?**
A: Yes, basic user management with login to track booking history.

### Functional Requirements

- ✅ **Movie Management:** Add, view, search movies
- ✅ **Showtime Management:** Schedule movies in theaters at specific times
- ✅ **Seat Selection:** View seat map, select available seats
- ✅ **Seat Locking:** Prevent concurrent booking conflicts
- ✅ **Booking Creation:** Reserve seats and generate tickets
- ✅ **Payment Processing:** Support multiple payment methods
- ✅ **Ticket Management:** View, cancel, refund tickets
- ✅ **User Management:** Register, login, view booking history

### Non-Functional Requirements

- Thread-safe seat locking (handle concurrent bookings)
- Seat lock timeout (10 minutes)
- Fast seat availability queries
- Simple, extensible architecture

### Out of Scope

- ❌ Food/beverage ordering
- ❌ Loyalty programs or discounts
- ❌ Movie recommendations
- ❌ Reviews and ratings
- ❌ Integration with external payment gateways
- ❌ Mobile app UI
- ❌ Email/SMS notifications

---

## 2. Core Objects Identification (Interview Step 2)

### Nouns (Potential Classes)

- **Movie** - Film being shown
- **Theater** - Physical auditorium/screen
- **Showtime** - Movie screening at specific time
- **Seat** - Individual seat in theater
- **Booking** - Reservation made by user
- **Ticket** - Proof of booking for a seat
- **User** - Customer booking tickets
- **Payment** - Transaction for booking
- **Cinema** - Complex containing multiple theaters

### Verbs (Potential Methods)

- `search_movies()` - Find movies
- `get_showtimes()` - Get available showtimes
- `select_seat()` - Choose seat
- `lock_seat()` - Reserve seat temporarily
- `unlock_seat()` - Release locked seat
- `create_booking()` - Make reservation
- `process_payment()` - Handle payment
- `cancel_booking()` - Cancel and refund
- `generate_ticket()` - Create ticket

### Relationships

**Inheritance (IS-A):**
- `CreditCardPayment IS-A Payment`
- `MobileWalletPayment IS-A Payment`

**Composition (HAS-A, strong):**
- `Theater HAS-A Seats` (seats owned by theater)
- `Showtime HAS-A Theater` (showtime bound to theater)
- `Booking HAS-A Tickets` (tickets owned by booking)

**Aggregation (HAS-A, weak):**
- `Showtime HAS-A Movie` (movie exists independently)
- `Booking HAS-A User` (user exists independently)

**Association (USES):**
- `Ticket REFERENCES Seat`
- `Ticket REFERENCES Showtime`

---

## 3. Class Diagram (Interview Step 3)

```
┌──────────────────┐         ┌──────────────────┐
│ <<enumeration>>  │         │ <<enumeration>>  │
│    SeatType      │         │   SeatStatus     │
├──────────────────┤         ├──────────────────┤
│ REGULAR          │         │ AVAILABLE        │
│ PREMIUM          │         │ BOOKED           │
│ VIP              │         │ LOCKED           │
└──────────────────┘         └──────────────────┘

┌──────────────────┐         ┌──────────────────┐
│ <<enumeration>>  │         │ <<enumeration>>  │
│  BookingStatus   │         │   MovieGenre     │
├──────────────────┤         ├──────────────────┤
│ PENDING          │         │ ACTION           │
│ CONFIRMED        │         │ COMEDY           │
│ CANCELLED        │         │ DRAMA            │
└──────────────────┘         │ HORROR           │
                             └──────────────────┘


        ┌─────────────────────────┐
        │        Movie            │
        ├─────────────────────────┤
        │ - movie_id: int         │
        │ - title: str            │
        │ - duration: int         │
        │ - genre: MovieGenre     │
        │ - rating: str           │
        ├─────────────────────────┤
        │ + get_details(): str    │
        └─────────────────────────┘
                    │
                    │ (used by)
                    ▼
        ┌─────────────────────────┐         ┌─────────────────────────┐
        │      Showtime           │    1  1 │       Theater           │
        ├─────────────────────────┤◆────────├─────────────────────────┤
        │ - showtime_id: int      │         │ - theater_id: int       │
        │ - movie: Movie          │         │ - name: str             │
        │ - theater: Theater      │         │ - seats: List[Seat]     │
        │ - start_time: DateTime  │         ├─────────────────────────┤
        │ - price: float          │         │ + get_available_seats() │
        ├─────────────────────────┤         │ + get_seat_map()        │
        │ + get_available_seats() │         └─────────────────────────┘
        │ + is_available(): bool  │                     │
        └─────────────────────────┘                     │ 1
                                                        │
                                                        │ *
                                            ┌───────────┴──────────┐
                                            │       Seat           │
                                            ├──────────────────────┤
                                            │ - seat_id: str       │
                                            │ - row: str           │
                                            │ - number: int        │
                                            │ - seat_type: SeatType│
                                            │ - status: SeatStatus │
                                            │ - locked_until: time │
                                            ├──────────────────────┤
                                            │ + lock(): bool       │
                                            │ + unlock()           │
                                            │ + book()             │
                                            │ + is_available()     │
                                            └──────────────────────┘


┌─────────────────────────┐         ┌─────────────────────────┐
│        User             │    1  * │       Booking           │
├─────────────────────────┤◇────────├─────────────────────────┤
│ - user_id: int          │         │ - booking_id: int       │
│ - name: str             │         │ - user: User            │
│ - email: str            │         │ - showtime: Showtime    │
│ - phone: str            │         │ - seats: List[Seat]     │
├─────────────────────────┤         │ - status: BookingStatus │
│ + create_booking()      │         │ - total_amount: float   │
│ + get_bookings()        │         │ - created_at: DateTime  │
│ + cancel_booking()      │         ├─────────────────────────┤
└─────────────────────────┘         │ + confirm()             │
                                    │ + cancel()              │
                                    │ + get_tickets()         │
                                    └─────────────────────────┘
                                                │
                                                │ ◆
                                                │
                                                │ *
                                    ┌───────────┴──────────┐
                                    │       Ticket         │
                                    ├──────────────────────┤
                                    │ - ticket_id: str     │
                                    │ - booking: Booking   │
                                    │ - seat: Seat         │
                                    │ - showtime: Showtime │
                                    ├──────────────────────┤
                                    │ + print_ticket()     │
                                    └──────────────────────┘


        ┌─────────────────────────┐
        │    <<abstract>>         │
        │       Payment           │
        ├─────────────────────────┤
        │ - amount: float         │
        │ - payment_id: str       │
        ├─────────────────────────┤
        │ + process(): bool       │
        │ + refund(): bool        │
        └─────────────────────────┘
                    △
                    │
        ┌───────────┴───────────┐
        │                       │
┌───────┴──────────┐  ┌─────────┴─────────┐
│ CreditCardPayment│  │ MobileWalletPayment│
├──────────────────┤  ├───────────────────┤
│ - card_number    │  │ - wallet_id       │
│ - cvv            │  │ - provider        │
└──────────────────┘  └───────────────────┘


┌─────────────────────────────────┐
│     BookingSystem               │
├─────────────────────────────────┤
│ - movies: List[Movie]           │
│ - theaters: List[Theater]       │
│ - showtimes: List[Showtime]     │
│ - bookings: List[Booking]       │
│ - users: Dict[int, User]        │
├─────────────────────────────────┤
│ + add_movie(movie: Movie)       │
│ + add_showtime(showtime)        │
│ + search_movies(query): List    │
│ + get_showtimes(movie): List    │
│ + create_booking(): Booking     │
│ + cancel_booking(id): bool      │
└─────────────────────────────────┘
```

---

## 4. Python Implementation (Interview Step 4)

### Step 1: Define Enums and Basic Classes

```python
from enum import Enum
from typing import Optional, List, Dict
from datetime import datetime, timedelta
from abc import ABC, abstractmethod
import threading
import uuid


class SeatType(Enum):
    """Types of seats available"""
    REGULAR = "Regular"
    PREMIUM = "Premium"
    VIP = "VIP"


class SeatStatus(Enum):
    """Current status of a seat"""
    AVAILABLE = "Available"
    LOCKED = "Locked"
    BOOKED = "Booked"


class BookingStatus(Enum):
    """Status of booking"""
    PENDING = "Pending"
    CONFIRMED = "Confirmed"
    CANCELLED = "Cancelled"


class MovieGenre(Enum):
    """Movie genres"""
    ACTION = "Action"
    COMEDY = "Comedy"
    DRAMA = "Drama"
    HORROR = "Horror"
    SCIFI = "Sci-Fi"
    ROMANCE = "Romance"
```

### Step 2: Movie and Theater Implementation

```python
class Movie:
    """Represents a movie"""

    _movie_counter = 1

    def __init__(self, title: str, duration: int, genre: MovieGenre, rating: str):
        self.movie_id = Movie._movie_counter
        Movie._movie_counter += 1
        self.title = title
        self.duration = duration  # in minutes
        self.genre = genre
        self.rating = rating  # e.g., "PG-13", "R"

    def get_details(self) -> str:
        """Return formatted movie details"""
        return f"{self.title} ({self.rating}) - {self.genre.value} - {self.duration} mins"

    def __str__(self) -> str:
        return self.get_details()


class Seat:
    """Represents a single seat in a theater"""

    def __init__(self, seat_id: str, row: str, number: int, seat_type: SeatType):
        self.seat_id = seat_id
        self.row = row
        self.number = number
        self.seat_type = seat_type
        self.status = SeatStatus.AVAILABLE
        self.locked_until: Optional[datetime] = None
        self._lock = threading.Lock()  # Thread-safe operations

    def lock(self, duration_minutes: int = 10) -> bool:
        """
        Lock seat for booking process.
        Thread-safe to prevent concurrent booking conflicts.
        """
        with self._lock:
            # Check if seat is available or lock expired
            if self.status == SeatStatus.BOOKED:
                return False

            if self.status == SeatStatus.LOCKED:
                if self.locked_until and datetime.now() < self.locked_until:
                    return False  # Still locked
                # Lock expired, can re-lock

            self.status = SeatStatus.LOCKED
            self.locked_until = datetime.now() + timedelta(minutes=duration_minutes)
            return True

    def unlock(self):
        """Release seat lock"""
        with self._lock:
            if self.status == SeatStatus.LOCKED:
                self.status = SeatStatus.AVAILABLE
                self.locked_until = None

    def book(self) -> bool:
        """Mark seat as booked"""
        with self._lock:
            if self.status == SeatStatus.LOCKED:
                self.status = SeatStatus.BOOKED
                self.locked_until = None
                return True
            return False

    def release(self):
        """Release booked seat (for cancellation)"""
        with self._lock:
            if self.status == SeatStatus.BOOKED:
                self.status = SeatStatus.AVAILABLE

    def is_available(self) -> bool:
        """Check if seat is available for booking"""
        with self._lock:
            if self.status == SeatStatus.AVAILABLE:
                return True
            if self.status == SeatStatus.LOCKED:
                # Check if lock expired
                if self.locked_until and datetime.now() >= self.locked_until:
                    self.status = SeatStatus.AVAILABLE
                    self.locked_until = None
                    return True
            return False

    def get_price_multiplier(self) -> float:
        """Get price multiplier based on seat type"""
        multipliers = {
            SeatType.REGULAR: 1.0,
            SeatType.PREMIUM: 1.5,
            SeatType.VIP: 2.0
        }
        return multipliers[self.seat_type]

    def __str__(self) -> str:
        return f"{self.row}{self.number} ({self.seat_type.value}) - {self.status.value}"


class Theater:
    """Represents a theater/auditorium"""

    def __init__(self, theater_id: int, name: str, rows: int = 10, seats_per_row: int = 15):
        self.theater_id = theater_id
        self.name = name
        self.seats: List[Seat] = []
        self._initialize_seats(rows, seats_per_row)

    def _initialize_seats(self, rows: int, seats_per_row: int):
        """Create seat layout"""
        row_letters = [chr(65 + i) for i in range(rows)]  # A, B, C, ...

        for row_idx, row in enumerate(row_letters):
            for seat_num in range(1, seats_per_row + 1):
                # Determine seat type based on position
                if row_idx < 2:  # First 2 rows are VIP
                    seat_type = SeatType.VIP
                elif row_idx < 5:  # Next 3 rows are Premium
                    seat_type = SeatType.PREMIUM
                else:  # Rest are Regular
                    seat_type = SeatType.REGULAR

                seat_id = f"{self.name}-{row}{seat_num}"
                seat = Seat(seat_id, row, seat_num, seat_type)
                self.seats.append(seat)

    def get_seat(self, row: str, number: int) -> Optional[Seat]:
        """Find seat by row and number"""
        for seat in self.seats:
            if seat.row == row and seat.number == number:
                return seat
        return None

    def get_available_seats(self) -> List[Seat]:
        """Get all available seats"""
        return [seat for seat in self.seats if seat.is_available()]

    def get_seat_map(self) -> str:
        """Display seat map"""
        seat_map = f"\n{self.name} Seat Map:\n"
        seat_map += "=" * 60 + "\n"

        current_row = None
        for seat in self.seats:
            if seat.row != current_row:
                if current_row is not None:
                    seat_map += "\n"
                seat_map += f"Row {seat.row}: "
                current_row = seat.row

            # Display seat status
            if seat.status == SeatStatus.AVAILABLE:
                symbol = "🟢"
            elif seat.status == SeatStatus.LOCKED:
                symbol = "🟡"
            else:  # BOOKED
                symbol = "🔴"

            seat_map += f"{symbol}{seat.number:2d} "

        seat_map += "\n" + "=" * 60
        seat_map += "\n🟢 Available  🟡 Locked  🔴 Booked\n"
        return seat_map

    def __str__(self) -> str:
        available = len(self.get_available_seats())
        total = len(self.seats)
        return f"{self.name} - {available}/{total} seats available"
```

### Step 3: Showtime Implementation

```python
class Showtime:
    """Represents a movie screening at a specific time"""

    _showtime_counter = 1

    def __init__(self, movie: Movie, theater: Theater, start_time: datetime, base_price: float):
        self.showtime_id = Showtime._showtime_counter
        Showtime._showtime_counter += 1
        self.movie = movie
        self.theater = theater
        self.start_time = start_time
        self.base_price = base_price

    def get_available_seats(self) -> List[Seat]:
        """Get available seats for this showtime"""
        return self.theater.get_available_seats()

    def is_available(self) -> bool:
        """Check if showtime has available seats"""
        return len(self.get_available_seats()) > 0

    def get_seat_price(self, seat: Seat) -> float:
        """Calculate price for specific seat"""
        return self.base_price * seat.get_price_multiplier()

    def __str__(self) -> str:
        time_str = self.start_time.strftime("%I:%M %p")
        return f"{self.movie.title} - {time_str} at {self.theater.name}"
```

### Step 4: Payment and Ticket Implementation

```python
class Payment(ABC):
    """Abstract base class for payment methods"""

    def __init__(self, amount: float):
        self.amount = amount
        self.payment_id = str(uuid.uuid4())
        self.processed = False

    @abstractmethod
    def process(self) -> bool:
        """Process payment - to be implemented by subclasses"""
        pass

    @abstractmethod
    def refund(self) -> bool:
        """Refund payment"""
        pass


class CreditCardPayment(Payment):
    """Credit card payment implementation"""

    def __init__(self, amount: float, card_number: str, cvv: str):
        super().__init__(amount)
        self.card_number = card_number[-4:]  # Store last 4 digits only
        self.cvv = cvv  # In real system, never store CVV

    def process(self) -> bool:
        """Simulate credit card processing"""
        print(f"Processing credit card payment of ${self.amount:.2f}...")
        print(f"Card ending in {self.card_number}")
        # Simulate payment gateway API call
        self.processed = True
        return True

    def refund(self) -> bool:
        """Simulate credit card refund"""
        if self.processed:
            print(f"Refunding ${self.amount:.2f} to card ending in {self.card_number}")
            return True
        return False


class MobileWalletPayment(Payment):
    """Mobile wallet payment implementation"""

    def __init__(self, amount: float, wallet_id: str, provider: str):
        super().__init__(amount)
        self.wallet_id = wallet_id
        self.provider = provider  # e.g., "PayPal", "Venmo", "Google Pay"

    def process(self) -> bool:
        """Simulate mobile wallet processing"""
        print(f"Processing {self.provider} payment of ${self.amount:.2f}...")
        print(f"Wallet ID: {self.wallet_id}")
        self.processed = True
        return True

    def refund(self) -> bool:
        """Simulate mobile wallet refund"""
        if self.processed:
            print(f"Refunding ${self.amount:.2f} to {self.provider} wallet")
            return True
        return False


class Ticket:
    """Represents a ticket for a booked seat"""

    def __init__(self, booking: 'Booking', seat: Seat, showtime: Showtime):
        self.ticket_id = str(uuid.uuid4())
        self.booking = booking
        self.seat = seat
        self.showtime = showtime

    def print_ticket(self) -> str:
        """Generate printable ticket"""
        ticket = "\n" + "=" * 50 + "\n"
        ticket += "            MOVIE TICKET\n"
        ticket += "=" * 50 + "\n"
        ticket += f"Movie:    {self.showtime.movie.title}\n"
        ticket += f"Theater:  {self.showtime.theater.name}\n"
        ticket += f"Time:     {self.showtime.start_time.strftime('%Y-%m-%d %I:%M %p')}\n"
        ticket += f"Seat:     Row {self.seat.row}, Seat {self.seat.number} ({self.seat.seat_type.value})\n"
        ticket += f"Ticket ID: {self.ticket_id[:8]}\n"
        ticket += "=" * 50 + "\n"
        return ticket

    def __str__(self) -> str:
        return f"Ticket {self.ticket_id[:8]} - {self.seat.row}{self.seat.number}"
```

### Step 5: Booking and User Implementation

```python
class User:
    """Represents a user of the booking system"""

    _user_counter = 1

    def __init__(self, name: str, email: str, phone: str):
        self.user_id = User._user_counter
        User._user_counter += 1
        self.name = name
        self.email = email
        self.phone = phone
        self.bookings: List['Booking'] = []

    def add_booking(self, booking: 'Booking'):
        """Add booking to user's history"""
        self.bookings.append(booking)

    def get_bookings(self) -> List['Booking']:
        """Get all user bookings"""
        return self.bookings

    def __str__(self) -> str:
        return f"User: {self.name} ({self.email})"


class Booking:
    """Represents a booking for a showtime"""

    _booking_counter = 1

    def __init__(self, user: User, showtime: Showtime, seats: List[Seat]):
        self.booking_id = Booking._booking_counter
        Booking._booking_counter += 1
        self.user = user
        self.showtime = showtime
        self.seats = seats
        self.status = BookingStatus.PENDING
        self.total_amount = self._calculate_total()
        self.created_at = datetime.now()
        self.tickets: List[Ticket] = []
        self.payment: Optional[Payment] = None

    def _calculate_total(self) -> float:
        """Calculate total booking amount"""
        total = sum(self.showtime.get_seat_price(seat) for seat in self.seats)
        return round(total, 2)

    def confirm(self, payment: Payment) -> bool:
        """Confirm booking after successful payment"""
        # Process payment
        if not payment.process():
            return False

        self.payment = payment
        self.status = BookingStatus.CONFIRMED

        # Book all seats
        for seat in self.seats:
            seat.book()

        # Generate tickets
        for seat in self.seats:
            ticket = Ticket(self, seat, self.showtime)
            self.tickets.append(ticket)

        # Add to user's booking history
        self.user.add_booking(self)

        return True

    def cancel(self) -> bool:
        """Cancel booking and refund"""
        if self.status != BookingStatus.CONFIRMED:
            return False

        # Check if cancellation is allowed (e.g., before showtime)
        time_until_show = self.showtime.start_time - datetime.now()
        if time_until_show.total_seconds() < 0:
            print("Cannot cancel - show has already started")
            return False

        # Process refund
        if self.payment and self.payment.refund():
            self.status = BookingStatus.CANCELLED

            # Release seats
            for seat in self.seats:
                seat.release()

            return True

        return False

    def get_tickets(self) -> List[Ticket]:
        """Get all tickets for this booking"""
        return self.tickets

    def __str__(self) -> str:
        seats_str = ", ".join([f"{s.row}{s.number}" for s in self.seats])
        return (f"Booking #{self.booking_id} - {self.showtime.movie.title} - "
                f"Seats: {seats_str} - ${self.total_amount:.2f} - {self.status.value}")
```

### Step 6: Main Booking System

```python
class BookingSystem:
    """Main movie ticket booking system"""

    def __init__(self, name: str):
        self.name = name
        self.movies: List[Movie] = []
        self.theaters: List[Theater] = []
        self.showtimes: List[Showtime] = []
        self.bookings: List[Booking] = []
        self.users: Dict[int, User] = {}

    def add_movie(self, movie: Movie):
        """Add a movie to the system"""
        self.movies.append(movie)

    def add_theater(self, theater: Theater):
        """Add a theater"""
        self.theaters.append(theater)

    def add_showtime(self, showtime: Showtime):
        """Schedule a showtime"""
        self.showtimes.append(showtime)

    def register_user(self, name: str, email: str, phone: str) -> User:
        """Register a new user"""
        user = User(name, email, phone)
        self.users[user.user_id] = user
        return user

    def search_movies(self, query: str = "") -> List[Movie]:
        """Search movies by title"""
        if not query:
            return self.movies

        query = query.lower()
        return [m for m in self.movies if query in m.title.lower()]

    def get_showtimes(self, movie: Movie, date: Optional[datetime] = None) -> List[Showtime]:
        """Get showtimes for a movie"""
        showtimes = [s for s in self.showtimes if s.movie.movie_id == movie.movie_id]

        if date:
            # Filter by date
            showtimes = [s for s in showtimes if s.start_time.date() == date.date()]

        return sorted(showtimes, key=lambda x: x.start_time)

    def create_booking(self, user: User, showtime: Showtime, seat_selections: List[tuple]) -> Optional[Booking]:
        """
        Create a booking for selected seats.
        seat_selections: List of (row, number) tuples
        """
        # Find and lock seats
        seats = []
        for row, number in seat_selections:
            seat = showtime.theater.get_seat(row, number)
            if not seat:
                print(f"Seat {row}{number} not found")
                self._unlock_seats(seats)
                return None

            if not seat.lock():
                print(f"Seat {row}{number} is not available")
                self._unlock_seats(seats)
                return None

            seats.append(seat)

        # Create booking
        booking = Booking(user, showtime, seats)
        self.bookings.append(booking)

        return booking

    def _unlock_seats(self, seats: List[Seat]):
        """Helper to unlock seats"""
        for seat in seats:
            seat.unlock()

    def cancel_booking(self, booking_id: int) -> bool:
        """Cancel a booking by ID"""
        for booking in self.bookings:
            if booking.booking_id == booking_id:
                return booking.cancel()
        return False

    def get_user_bookings(self, user: User) -> List[Booking]:
        """Get all bookings for a user"""
        return user.get_bookings()

    def display_movies(self):
        """Display all available movies"""
        print("\n" + "=" * 60)
        print("NOW SHOWING")
        print("=" * 60)
        for movie in self.movies:
            print(f"{movie.movie_id}. {movie}")
        print("=" * 60)
```

---

## 5. Complete Usage Example

```python
def main():
    """Demonstrate movie ticket booking system"""

    # Create booking system
    cinema = BookingSystem("CineMax Cinema")

    # Add movies
    movie1 = Movie("Inception", 148, MovieGenre.SCIFI, "PG-13")
    movie2 = Movie("The Dark Knight", 152, MovieGenre.ACTION, "PG-13")
    movie3 = Movie("Interstellar", 169, MovieGenre.SCIFI, "PG-13")

    cinema.add_movie(movie1)
    cinema.add_movie(movie2)
    cinema.add_movie(movie3)

    # Add theaters
    theater1 = Theater(1, "Screen 1", rows=8, seats_per_row=12)
    theater2 = Theater(2, "Screen 2", rows=10, seats_per_row=15)

    cinema.add_theater(theater1)
    cinema.add_theater(theater2)

    # Add showtimes
    now = datetime.now()
    showtime1 = Showtime(movie1, theater1, now + timedelta(hours=2), base_price=12.00)
    showtime2 = Showtime(movie1, theater1, now + timedelta(hours=5), base_price=15.00)
    showtime3 = Showtime(movie2, theater2, now + timedelta(hours=3), base_price=12.00)

    cinema.add_showtime(showtime1)
    cinema.add_showtime(showtime2)
    cinema.add_showtime(showtime3)

    # Display movies
    cinema.display_movies()

    # Register users
    user1 = cinema.register_user("Alice Johnson", "alice@example.com", "555-1234")
    user2 = cinema.register_user("Bob Smith", "bob@example.com", "555-5678")

    print(f"\nRegistered: {user1}")
    print(f"Registered: {user2}")

    # Browse showtimes
    print("\n--- Showtimes for Inception ---")
    inception_showtimes = cinema.get_showtimes(movie1)
    for showtime in inception_showtimes:
        print(f"{showtime.showtime_id}. {showtime} - ${showtime.base_price}")

    # Display seat map
    print(theater1.get_seat_map())

    # Create booking 1
    print("\n--- User 1: Creating Booking ---")
    selected_seats = [("A", 5), ("A", 6)]  # VIP seats
    booking1 = cinema.create_booking(user1, showtime1, selected_seats)

    if booking1:
        print(f"\nBooking created: {booking1}")
        print(f"Total amount: ${booking1.total_amount:.2f}")

        # Process payment
        payment = CreditCardPayment(booking1.total_amount, "4532123456789012", "123")
        if booking1.confirm(payment):
            print("\n✓ Booking confirmed!")

            # Print tickets
            for ticket in booking1.get_tickets():
                print(ticket.print_ticket())

    # Display updated seat map
    print("\n--- Updated Seat Map ---")
    print(theater1.get_seat_map())

    # Create booking 2 (concurrent attempt)
    print("\n--- User 2: Trying to Book Same Seats (Should Fail) ---")
    booking2 = cinema.create_booking(user2, showtime1, [("A", 5)])  # Same seat
    if not booking2:
        print("✗ Booking failed - seats already booked")

    # Create different booking
    print("\n--- User 2: Booking Different Seats ---")
    booking3 = cinema.create_booking(user2, showtime1, [("B", 3), ("B", 4)])
    if booking3:
        payment2 = MobileWalletPayment(booking3.total_amount, "bob@wallet.com", "PayPal")
        booking3.confirm(payment2)
        print(f"✓ {booking3}")

    # View user bookings
    print("\n--- User 1 Booking History ---")
    for booking in cinema.get_user_bookings(user1):
        print(f"  {booking}")

    # Cancel booking
    print("\n--- Cancelling Booking ---")
    if booking1.cancel():
        print(f"✓ Booking #{booking1.booking_id} cancelled successfully")
        print("Seats released")

    # Final seat map
    print(theater1.get_seat_map())


if __name__ == "__main__":
    main()
```

---

## 6. Design Patterns Used

### Factory Pattern

**Where:** Can be used for creating different payment types

**Code:**
```python
class PaymentFactory:
    @staticmethod
    def create_payment(payment_type: str, amount: float, **kwargs) -> Payment:
        if payment_type == "credit_card":
            return CreditCardPayment(amount, kwargs['card_number'], kwargs['cvv'])
        elif payment_type == "wallet":
            return MobileWalletPayment(amount, kwargs['wallet_id'], kwargs['provider'])
        raise ValueError(f"Unknown payment type: {payment_type}")

# Usage
payment = PaymentFactory.create_payment("credit_card", 24.00,
                                        card_number="4532...", cvv="123")
```

**Why:** Encapsulates object creation logic, easy to add new payment methods.

### State Pattern

**Where:** `BookingStatus` and `SeatStatus` enums represent states

**Why:** Bookings and seats transition through defined states (Pending → Confirmed → Cancelled).

### Strategy Pattern

**Where:** `Payment` abstract class with different payment implementations

**Code:**
```python
# Different payment strategies can be swapped at runtime
payment1 = CreditCardPayment(amount, card, cvv)
payment2 = MobileWalletPayment(amount, wallet, provider)

booking.confirm(payment1)  # or payment2
```

**Why:** Allows runtime selection of payment method without changing booking logic.

### Singleton Pattern (Optional)

**Where:** `BookingSystem` could be a singleton

**Code:**
```python
class BookingSystem:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

**Why:** Ensures only one booking system instance exists.

---

## 7. SOLID Principles Applied

### Single Responsibility Principle (SRP)

Each class has one clear responsibility:
- `Seat`: Manages individual seat state and locking
- `Theater`: Manages collection of seats and layout
- `Booking`: Handles reservation and payment
- `Payment`: Processes transactions
- `BookingSystem`: Coordinates overall system

### Open/Closed Principle (OCP)

Easy to extend without modification:
- Add new payment methods by extending `Payment` class
- Add new seat types to `SeatType` enum
- Add new movie genres to `MovieGenre` enum

### Liskov Substitution Principle (LSP)

All payment types can substitute `Payment` base class:
```python
def process_booking(booking: Booking, payment: Payment):
    # Works with any Payment subclass
    payment.process()
```

### Interface Segregation Principle (ISP)

Classes have minimal, focused interfaces:
- `Seat` interface: lock, unlock, book, release
- `Payment` interface: process, refund
- No unnecessary methods

### Dependency Inversion Principle (DIP)

High-level modules depend on abstractions:
- `Booking.confirm()` depends on `Payment` abstraction, not concrete payment types

---

## 8. Extensions and Follow-up Questions

### Q1: How would you handle seat hold timeout?

**Answer:** Implement background task to check and release expired locks:

```python
import threading
import time

class SeatLockManager:
    def __init__(self, booking_system: BookingSystem):
        self.booking_system = booking_system
        self.running = False

    def start(self):
        """Start background thread to release expired locks"""
        self.running = True
        thread = threading.Thread(target=self._cleanup_expired_locks, daemon=True)
        thread.start()

    def _cleanup_expired_locks(self):
        """Periodically check and release expired locks"""
        while self.running:
            time.sleep(30)  # Check every 30 seconds
            for theater in self.booking_system.theaters:
                for seat in theater.seats:
                    if seat.status == SeatStatus.LOCKED:
                        if seat.locked_until and datetime.now() >= seat.locked_until:
                            seat.unlock()
                            print(f"Released expired lock on seat {seat.seat_id}")

# Usage
lock_manager = SeatLockManager(cinema)
lock_manager.start()
```

### Q2: How would you add notifications (email/SMS)?

**Answer:** Use Observer pattern:

```python
from abc import ABC, abstractmethod

class BookingObserver(ABC):
    @abstractmethod
    def notify(self, booking: Booking, event: str):
        pass

class EmailNotifier(BookingObserver):
    def notify(self, booking: Booking, event: str):
        if event == "confirmed":
            print(f"Sending confirmation email to {booking.user.email}")
        elif event == "cancelled":
            print(f"Sending cancellation email to {booking.user.email}")

class SMSNotifier(BookingObserver):
    def notify(self, booking: Booking, event: str):
        print(f"Sending SMS to {booking.user.phone}: Booking {event}")

class Booking:
    def __init__(self, user, showtime, seats):
        # ... existing init
        self.observers: List[BookingObserver] = []

    def attach_observer(self, observer: BookingObserver):
        self.observers.append(observer)

    def _notify_observers(self, event: str):
        for observer in self.observers:
            observer.notify(self, event)

    def confirm(self, payment: Payment) -> bool:
        # ... existing confirm logic
        self._notify_observers("confirmed")
        return True

    def cancel(self) -> bool:
        # ... existing cancel logic
        self._notify_observers("cancelled")
        return True

# Usage
booking.attach_observer(EmailNotifier())
booking.attach_observer(SMSNotifier())
```

### Q3: How would you implement dynamic pricing?

**Answer:** Use Strategy pattern for pricing:

```python
class PricingStrategy(ABC):
    @abstractmethod
    def calculate_price(self, base_price: float, seat: Seat, showtime: Showtime) -> float:
        pass

class StandardPricing(PricingStrategy):
    def calculate_price(self, base_price: float, seat: Seat, showtime: Showtime) -> float:
        return base_price * seat.get_price_multiplier()

class DynamicPricing(PricingStrategy):
    def calculate_price(self, base_price: float, seat: Seat, showtime: Showtime) -> float:
        price = base_price * seat.get_price_multiplier()

        # Weekend premium
        if showtime.start_time.weekday() >= 5:  # Saturday or Sunday
            price *= 1.2

        # Peak hours (6pm-10pm)
        hour = showtime.start_time.hour
        if 18 <= hour < 22:
            price *= 1.15

        # Occupancy-based pricing
        available = len(showtime.get_available_seats())
        total = len(showtime.theater.seats)
        occupancy = 1 - (available / total)
        if occupancy > 0.8:
            price *= 1.1  # High demand

        return round(price, 2)

class Showtime:
    def __init__(self, movie, theater, start_time, base_price):
        # ... existing init
        self.pricing_strategy: PricingStrategy = StandardPricing()

    def get_seat_price(self, seat: Seat) -> float:
        return self.pricing_strategy.calculate_price(self.base_price, seat, self)

# Usage
showtime.pricing_strategy = DynamicPricing()
```

### Q4: How would you handle group bookings with discounts?

**Answer:** Add group booking class with discount logic:

```python
class GroupBooking(Booking):
    def __init__(self, user, showtime, seats, group_size_threshold: int = 5):
        super().__init__(user, showtime, seats)
        self.group_size_threshold = group_size_threshold
        self.total_amount = self._calculate_total_with_discount()

    def _calculate_total_with_discount(self) -> float:
        total = super()._calculate_total()

        if len(self.seats) >= self.group_size_threshold:
            discount = 0.15  # 15% discount for groups
            total *= (1 - discount)
            print(f"Applied {discount*100}% group discount")

        return round(total, 2)

# Usage
group_booking = GroupBooking(user, showtime, seats)  # Auto discount if >= 5 seats
```

---

## 9. Interview Tips

### What Interviewers Look For

- ✅ **Thread safety:** Proper seat locking to prevent double booking
- ✅ **State management:** Correct handling of booking and seat states
- ✅ **Design patterns:** Factory, Strategy, Observer patterns
- ✅ **Concurrency handling:** Thread-safe operations with locks
- ✅ **Edge cases:** Expired locks, cancelled bookings, payment failures
- ✅ **Extensibility:** Easy to add theaters, movies, payment methods
- ✅ **Real-world considerations:** Timeout logic, refund policies

### Common Mistakes

- ❌ **No concurrency control:** Allowing double booking of seats
- ❌ **Missing timeout logic:** Seats locked forever
- ❌ **Tight coupling:** Payment logic embedded in booking
- ❌ **No state validation:** Allowing invalid state transitions
- ❌ **Missing edge cases:** Not handling show start time for cancellations
- ❌ **Poor separation:** Mixing UI, business logic, data layers

### Time Management (45 min interview)

- **0-5 min:** Requirements and clarification (concurrency is key!)
- **5-10 min:** Identify classes and relationships
- **10-15 min:** Draw class diagram with emphasis on Seat locking
- **15-35 min:** Implement core classes (Seat with thread safety, Booking, Payment)
- **35-40 min:** Demonstrate booking flow and concurrent access
- **40-45 min:** Discuss patterns, scaling, extensions

---

## 10. Summary

### Key Takeaways

1. **Thread Safety** is critical for preventing double booking
2. **State machines** manage booking and seat lifecycles
3. **Strategy pattern** enables flexible payment processing
4. **Seat locking** with timeouts handles concurrent users
5. **Composition** models real-world relationships (Theater owns Seats)
6. **Factory pattern** can simplify object creation
7. **Observer pattern** enables notifications and event handling

### Design Highlights

- **Concurrent access:** Thread-safe seat locking with `threading.Lock`
- **Timeout mechanism:** Automatic lock expiration
- **Payment abstraction:** Easy to add new payment methods
- **State management:** Clear state transitions for bookings and seats
- **Realistic modeling:** Reflects actual cinema booking systems

### Related Problems

- **Restaurant Table Booking:** Similar seat/table locking
- **Flight Reservation System:** Seat selection and overbooking
- **Hotel Room Booking:** Room availability and locking
- **Concert Ticket Booking:** Similar seat map and pricing

This problem demonstrates real-world system design with concurrency challenges!
