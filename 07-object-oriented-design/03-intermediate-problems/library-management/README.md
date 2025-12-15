# Library Management System - OOD Design

**Difficulty:** Intermediate
**Interview Frequency:** High
**Key Concepts:** Many-to-Many Relationships, State Pattern, Observer Pattern, Fine Management
**Companies:** Amazon, Google, Microsoft, Apple

---

## Problem Statement

Design a library management system that handles books, members, librarians, checkouts, and fines. The system should track book availability, manage loans, calculate fines for late returns, and support reservations.

**Core Features:**
1. Manage books (add, remove, search)
2. Member registration and management
3. Check out and return books
4. Reserve books
5. Calculate and collect fines
6. Track book loan history
7. Support different member types with different limits

---

## Requirements Gathering

### Functional Requirements
- ✅ Add/remove books from catalog
- ✅ Register members (student, faculty, public)
- ✅ Check out books (max limits per member type)
- ✅ Return books and calculate fines
- ✅ Reserve books when unavailable
- ✅ Search books by title, author, ISBN
- ✅ Track loan history
- ✅ Renew loans (if no reservations)

### Non-Functional Requirements
- Handle concurrent checkouts
- Fast search operations
- Accurate fine calculations
- Data persistence considerations

### Out of Scope
- ❌ Payment processing integration
- ❌ Digital books/e-books
- ❌ Inter-library loans
- ❌ Book recommendations

---

## Implementation

```python
from enum import Enum
from datetime import datetime, timedelta
from typing import Optional, List, Dict
from abc import ABC, abstractmethod


class BookStatus(Enum):
    AVAILABLE = "Available"
    CHECKED_OUT = "Checked Out"
    RESERVED = "Reserved"
    LOST = "Lost"


class MemberType(Enum):
    STUDENT = "Student"
    FACULTY = "Faculty"
    PUBLIC = "Public"


class Book:
    def __init__(self, isbn: str, title: str, author: str, publication_year: int):
        self.isbn = isbn
        self.title = title
        self.author = author
        self.publication_year = publication_year
        self.status = BookStatus.AVAILABLE
        self.borrowed_by: Optional['Member'] = None
        self.reserved_by: List['Member'] = []

    def is_available(self) -> bool:
        return self.status == BookStatus.AVAILABLE

    def checkout(self, member: 'Member') -> bool:
        if self.is_available():
            self.status = BookStatus.CHECKED_OUT
            self.borrowed_by = member
            return True
        return False

    def return_book(self):
        self.status = BookStatus.AVAILABLE
        self.borrowed_by = None
        if self.reserved_by:
            self.status = BookStatus.RESERVED

    def reserve(self, member: 'Member'):
        if member not in self.reserved_by:
            self.reserved_by.append(member)
            if self.is_available():
                self.status = BookStatus.RESERVED

    def cancel_reservation(self, member: 'Member'):
        if member in self.reserved_by:
            self.reserved_by.remove(member)
            if not self.reserved_by:
                self.status = BookStatus.AVAILABLE

    def __str__(self) -> str:
        return f"{self.title} by {self.author} ({self.isbn}) - {self.status.value}"


class Member(ABC):
    def __init__(self, member_id: int, name: str, email: str):
        self.member_id = member_id
        self.name = name
        self.email = email
        self.checked_out_books: List['LoanRecord'] = []
        self.reservations: List[Book] = []
        self.total_fines = 0.0

    @abstractmethod
    def get_max_books(self) -> int:
        pass

    @abstractmethod
    def get_loan_period_days(self) -> int:
        pass

    @abstractmethod
    def get_daily_fine(self) -> float:
        pass

    def can_checkout(self) -> bool:
        return len(self.checked_out_books) < self.get_max_books()

    def add_fine(self, amount: float):
        self.total_fines += amount

    def pay_fine(self, amount: float) -> float:
        paid = min(amount, self.total_fines)
        self.total_fines -= paid
        return paid

    def __str__(self) -> str:
        return f"{self.name} ({self.member_id}) - {self.__class__.__name__}"


class StudentMember(Member):
    def get_max_books(self) -> int:
        return 3

    def get_loan_period_days(self) -> int:
        return 14

    def get_daily_fine(self) -> float:
        return 0.50


class FacultyMember(Member):
    def get_max_books(self) -> int:
        return 10

    def get_loan_period_days(self) -> int:
        return 30

    def get_daily_fine(self) -> float:
        return 1.00


class PublicMember(Member):
    def get_max_books(self) -> int:
        return 5

    def get_loan_period_days(self) -> int:
        return 21

    def get_daily_fine(self) -> float:
        return 0.75


class LoanRecord:
    _record_counter = 1

    def __init__(self, book: Book, member: Member):
        self.loan_id = LoanRecord._record_counter
        LoanRecord._record_counter += 1
        self.book = book
        self.member = member
        self.checkout_date = datetime.now()
        self.due_date = self.checkout_date + timedelta(days=member.get_loan_period_days())
        self.return_date: Optional[datetime] = None
        self.fine_amount = 0.0

    def return_book(self) -> float:
        self.return_date = datetime.now()
        if self.return_date > self.due_date:
            days_late = (self.return_date - self.due_date).days
            self.fine_amount = days_late * self.member.get_daily_fine()
            self.member.add_fine(self.fine_amount)
        return self.fine_amount

    def is_overdue(self) -> bool:
        return datetime.now() > self.due_date and self.return_date is None

    def days_overdue(self) -> int:
        if self.is_overdue():
            return (datetime.now() - self.due_date).days
        return 0

    def __str__(self) -> str:
        status = "Returned" if self.return_date else "Active"
        return f"Loan #{self.loan_id}: {self.book.title} - {status}"


class Library:
    def __init__(self, name: str):
        self.name = name
        self.books: Dict[str, Book] = {}  # ISBN -> Book
        self.members: Dict[int, Member] = {}  # member_id -> Member
        self.loans: List[LoanRecord] = []

    def add_book(self, book: Book):
        self.books[book.isbn] = book
        print(f"Added: {book}")

    def remove_book(self, isbn: str) -> bool:
        if isbn in self.books:
            book = self.books[isbn]
            if book.is_available():
                del self.books[isbn]
                return True
        return False

    def register_member(self, member: Member):
        self.members[member.member_id] = member
        print(f"Registered: {member}")

    def search_by_title(self, title: str) -> List[Book]:
        return [b for b in self.books.values() if title.lower() in b.title.lower()]

    def search_by_author(self, author: str) -> List[Book]:
        return [b for b in self.books.values() if author.lower() in b.author.lower()]

    def checkout_book(self, isbn: str, member_id: int) -> Optional[LoanRecord]:
        if isbn not in self.books or member_id not in self.members:
            print("Book or member not found")
            return None

        book = self.books[isbn]
        member = self.members[member_id]

        if not member.can_checkout():
            print(f"Member has reached checkout limit ({member.get_max_books()})")
            return None

        if member.total_fines > 0:
            print(f"Member has outstanding fines: ${member.total_fines:.2f}")
            return None

        if not book.checkout(member):
            print(f"Book is not available. Current status: {book.status.value}")
            return None

        loan = LoanRecord(book, member)
        self.loans.append(loan)
        member.checked_out_books.append(loan)

        print(f"✓ Checked out: {book.title} to {member.name}")
        print(f"  Due date: {loan.due_date.strftime('%Y-%m-%d')}")
        return loan

    def return_book(self, isbn: str, member_id: int) -> Optional[float]:
        book = self.books.get(isbn)
        member = self.members.get(member_id)

        if not book or not member:
            return None

        # Find active loan
        loan = None
        for l in member.checked_out_books:
            if l.book.isbn == isbn and l.return_date is None:
                loan = l
                break

        if not loan:
            print("No active loan found")
            return None

        fine = loan.return_book()
        book.return_book()
        member.checked_out_books.remove(loan)

        print(f"✓ Returned: {book.title}")
        if fine > 0:
            print(f"  Fine assessed: ${fine:.2f}")

        return fine

    def reserve_book(self, isbn: str, member_id: int) -> bool:
        book = self.books.get(isbn)
        member = self.members.get(member_id)

        if book and member:
            book.reserve(member)
            member.reservations.append(book)
            print(f"✓ Reserved: {book.title} for {member.name}")
            return True
        return False

    def get_overdue_books(self) -> List[LoanRecord]:
        return [loan for loan in self.loans if loan.is_overdue()]

    def display_status(self):
        print(f"\n{'='*60}")
        print(f"{self.name} - Library Status")
        print(f"{'='*60}")
        print(f"Total Books: {len(self.books)}")
        print(f"Available: {sum(1 for b in self.books.values() if b.is_available())}")
        print(f"Checked Out: {sum(1 for b in self.books.values() if b.status == BookStatus.CHECKED_OUT)}")
        print(f"Total Members: {len(self.members)}")
        print(f"Active Loans: {sum(1 for l in self.loans if l.return_date is None)}")
        print(f"Overdue Loans: {len(self.get_overdue_books())}")
        print(f"{'='*60}\n")


def main():
    """Demonstrate library management system"""

    # Create library
    lib = Library("City Public Library")

    # Add books
    lib.add_book(Book("978-0134685991", "Effective Java", "Joshua Bloch", 2017))
    lib.add_book(Book("978-0132350884", "Clean Code", "Robert Martin", 2008))
    lib.add_book(Book("978-0201633610", "Design Patterns", "Gang of Four", 1994))
    lib.add_book(Book("978-0136291558", "Object Oriented Analysis", "Grady Booch", 2007))

    # Register members
    lib.register_member(StudentMember(1, "Alice", "alice@university.edu"))
    lib.register_member(FacultyMember(2, "Dr. Bob", "bob@university.edu"))
    lib.register_member(PublicMember(3, "Charlie", "charlie@email.com"))

    lib.display_status()

    # Search books
    print("--- Searching for 'Java' ---")
    results = lib.search_by_title("Java")
    for book in results:
        print(f"  {book}")

    # Checkout books
    print("\n--- Checkouts ---")
    lib.checkout_book("978-0134685991", 1)  # Alice checks out Effective Java
    lib.checkout_book("978-0132350884", 2)  # Dr. Bob checks out Clean Code

    lib.display_status()

    # Try to checkout already checked out book
    print("--- Attempting duplicate checkout ---")
    lib.checkout_book("978-0134685991", 3)  # Charlie tries to check out same book

    # Reserve book
    print("\n--- Reservation ---")
    lib.reserve_book("978-0134685991", 3)  # Charlie reserves it

    # Return book
    print("\n--- Returns ---")
    fine = lib.return_book("978-0134685991", 1)  # Alice returns
    if fine and fine > 0:
        print(f"Fine to pay: ${fine:.2f}")

    lib.display_status()

    # Show overdue books
    overdue = lib.get_overdue_books()
    if overdue:
        print("--- Overdue Books ---")
        for loan in overdue:
            print(f"  {loan} - {loan.days_overdue()} days overdue")


if __name__ == "__main__":
    main()
```

---

## Design Patterns Used

### Strategy Pattern
**Where:** Different member types with different rules
```python
class Member(ABC):
    @abstractmethod
    def get_max_books(self) -> int:
        pass  # Each type implements own strategy
```

### State Pattern
**Where:** Book status (Available, Checked Out, Reserved)

### Template Method
**Where:** Member base class defines loan workflow, subclasses customize limits

---

## SOLID Principles

- **SRP:** Each class has single responsibility (Book manages state, LoanRecord tracks loans)
- **OCP:** Easy to add new member types without modifying existing code
- **LSP:** All member types can substitute Member base class
- **ISP:** Focused interfaces, no unnecessary methods
- **DIP:** Library depends on Member abstraction, not concrete types

---

## Extensions

### Q1: How would you add book categories and search filters?
```python
class Category(Enum):
    FICTION = "Fiction"
    NON_FICTION = "Non-Fiction"
    SCIENCE = "Science"

class Book:
    def __init__(self, ...):
        # ... existing
        self.category = Category.FICTION
        self.tags: List[str] = []

class Library:
    def search_by_category(self, category: Category) -> List[Book]:
        return [b for b in self.books.values() if b.category == category]
```

### Q2: How would you implement notification system for due dates?
Use Observer pattern:
```python
class NotificationObserver(ABC):
    @abstractmethod
    def notify(self, loan: LoanRecord):
        pass

class EmailNotifier(NotificationObserver):
    def notify(self, loan: LoanRecord):
        print(f"Sending email to {loan.member.email}: Book due soon")

class LoanRecord:
    def __init__(self, ...):
        self.observers: List[NotificationObserver] = []

    def check_due_soon(self):
        if (self.due_date - datetime.now()).days <= 2:
            for observer in self.observers:
                observer.notify(self)
```

---

## Interview Tips

### What Interviewers Look For
- ✅ Many-to-many relationships (books ↔ members)
- ✅ State management (book status transitions)
- ✅ Business rules (loan limits, fines)
- ✅ Inheritance hierarchy for member types
- ✅ Date/time handling for due dates

### Common Mistakes
- ❌ Not enforcing checkout limits
- ❌ Incorrect fine calculations
- ❌ Missing reservation system
- ❌ No validation for member fines
- ❌ Tight coupling between book and member

---

## Summary

Library management demonstrates:
- Complex entity relationships
- Business rule enforcement
- State management
- Strategy pattern for member types
- Fine calculation logic
- Reservation queuing

This problem tests understanding of real-world business logic, state machines, and policy enforcement through OOD.
