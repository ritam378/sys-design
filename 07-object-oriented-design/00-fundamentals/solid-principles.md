# SOLID Principles

## Overview

SOLID is an acronym for five design principles that make software designs more understandable, flexible, and maintainable. Introduced by Robert C. Martin (Uncle Bob), these principles are fundamental to object-oriented design.

**Why SOLID matters:**
- ✅ Makes code easier to understand and maintain
- ✅ Reduces coupling between components
- ✅ Makes code more testable
- ✅ Facilitates code reuse
- ✅ Makes systems more robust to change

---

## S - Single Responsibility Principle (SRP)

### Definition
**"A class should have one, and only one, reason to change."**

Each class should have a single, well-defined responsibility. If a class has multiple responsibilities, changes to one responsibility may affect the others.

### Example: Bad Design

```python
class UserManager:
    """
    VIOLATION: This class has multiple responsibilities:
    1. User data management
    2. Email sending
    3. Report generation
    """

    def __init__(self):
        self.users = []

    def create_user(self, name, email):
        """User management responsibility"""
        user = {'name': name, 'email': email}
        self.users.append(user)
        return user

    def send_welcome_email(self, user):
        """Email responsibility - SHOULD BE SEPARATE!"""
        print(f"Sending email to {user['email']}")
        # Email sending logic...

    def generate_user_report(self):
        """Reporting responsibility - SHOULD BE SEPARATE!"""
        print("Generating report...")
        # Report generation logic...

    def save_to_database(self, user):
        """Persistence responsibility - SHOULD BE SEPARATE!"""
        print(f"Saving {user['name']} to database")
        # Database logic...
```

**Problems:**
- Changes to email format require modifying UserManager
- Changes to report format require modifying UserManager
- Changes to database require modifying UserManager
- Hard to test (need to mock email, database, etc.)
- Violates SRP - too many reasons to change

### Example: Good Design

```python
class User:
    """Represents a user entity"""
    def __init__(self, name, email):
        self.name = name
        self.email = email


class UserRepository:
    """SINGLE RESPONSIBILITY: User data persistence"""
    def __init__(self):
        self.users = []

    def create(self, user):
        self.users.append(user)
        return user

    def find_by_email(self, email):
        return next((u for u in self.users if u.email == email), None)


class EmailService:
    """SINGLE RESPONSIBILITY: Email sending"""
    def send_welcome_email(self, user):
        print(f"Sending welcome email to {user.email}")
        # Email logic here


class UserReportGenerator:
    """SINGLE RESPONSIBILITY: Report generation"""
    def __init__(self, user_repository):
        self.user_repository = user_repository

    def generate_report(self):
        users = self.user_repository.users
        print(f"Report: {len(users)} total users")
        # Report logic here


# Usage:
user_repo = UserRepository()
email_service = EmailService()
report_gen = UserReportGenerator(user_repo)

# Create user
user = User("Alice", "alice@example.com")
user_repo.create(user)

# Send email
email_service.send_welcome_email(user)

# Generate report
report_gen.generate_report()
```

**Benefits:**
- Each class has one reason to change
- Easy to test (mock individual services)
- Can modify email logic without touching UserRepository
- Promotes reusability (EmailService can be used elsewhere)

### Real-World Examples

**Bad (SRP Violation):**
- A `Product` class that handles business logic, database access, and UI rendering
- A `Controller` that processes HTTP requests, validates data, and sends emails

**Good (SRP Compliance):**
- MVC pattern: Model (data), View (presentation), Controller (logic)
- Microservices: Each service has single responsibility

---

## O - Open/Closed Principle (OCP)

### Definition
**"Software entities should be open for extension, but closed for modification."**

You should be able to add new functionality without changing existing code. Achieve this through abstraction and polymorphism.

### Example: Bad Design

```python
class PaymentProcessor:
    """
    VIOLATION: Adding new payment methods requires modifying this class
    """
    def process_payment(self, amount, payment_type):
        if payment_type == "credit_card":
            print(f"Processing ${amount} via Credit Card")
            # Credit card logic

        elif payment_type == "paypal":
            print(f"Processing ${amount} via PayPal")
            # PayPal logic

        elif payment_type == "crypto":
            # New payment method - had to modify existing code!
            print(f"Processing ${amount} via Cryptocurrency")
            # Crypto logic

        else:
            raise ValueError(f"Unknown payment type: {payment_type}")
```

**Problems:**
- Adding new payment methods requires modifying `process_payment`
- Risk of breaking existing functionality
- Violates OCP - not closed for modification

### Example: Good Design

```python
from abc import ABC, abstractmethod

class PaymentMethod(ABC):
    """Abstract base class - defines contract"""
    @abstractmethod
    def process_payment(self, amount):
        pass


class CreditCardPayment(PaymentMethod):
    """Concrete implementation for credit cards"""
    def process_payment(self, amount):
        print(f"Processing ${amount} via Credit Card")
        # Credit card specific logic


class PayPalPayment(PaymentMethod):
    """Concrete implementation for PayPal"""
    def process_payment(self, amount):
        print(f"Processing ${amount} via PayPal")
        # PayPal specific logic


class CryptoPayment(PaymentMethod):
    """New payment method - NO MODIFICATION to existing code!"""
    def process_payment(self, amount):
        print(f"Processing ${amount} via Cryptocurrency")
        # Crypto specific logic


class PaymentProcessor:
    """OPEN/CLOSED: Open for extension, closed for modification"""
    def process(self, payment_method: PaymentMethod, amount):
        payment_method.process_payment(amount)


# Usage:
processor = PaymentProcessor()

# Use credit card
processor.process(CreditCardPayment(), 100)

# Use PayPal
processor.process(PayPalPayment(), 200)

# Use crypto (NEW! No changes to PaymentProcessor)
processor.process(CryptoPayment(), 300)
```

**Benefits:**
- Add new payment methods by creating new classes (extension)
- No changes to PaymentProcessor (closed for modification)
- Existing payment methods unaffected by new additions
- Easy to test new methods in isolation

### Real-World Examples

**Strategy Pattern:**
```python
# Different sorting algorithms without modifying sorter
class SortStrategy(ABC):
    @abstractmethod
    def sort(self, data): pass

class QuickSort(SortStrategy):
    def sort(self, data): ...

class MergeSort(SortStrategy):
    def sort(self, data): ...

class Sorter:
    def __init__(self, strategy: SortStrategy):
        self.strategy = strategy

    def sort_data(self, data):
        return self.strategy.sort(data)
```

---

## L - Liskov Substitution Principle (LSP)

### Definition
**"Objects of a superclass should be replaceable with objects of a subclass without breaking the application."**

Subtypes must be behavioral substitutes for their base types. If S is a subtype of T, then objects of type T can be replaced with objects of type S.

### Example: Bad Design (Classic LSP Violation)

```python
class Rectangle:
    """Base class"""
    def __init__(self, width, height):
        self._width = width
        self._height = height

    def set_width(self, width):
        self._width = width

    def set_height(self, height):
        self._height = height

    def area(self):
        return self._width * self._height


class Square(Rectangle):
    """
    VIOLATION: Square violates LSP because it breaks Rectangle's behavior
    """
    def set_width(self, width):
        # Square must have equal width and height
        self._width = width
        self._height = width  # Unexpected side effect!

    def set_height(self, height):
        # Square must have equal width and height
        self._width = height  # Unexpected side effect!
        self._height = height


# Test that demonstrates LSP violation:
def test_rectangle(rect: Rectangle):
    """This function expects Rectangle behavior"""
    rect.set_width(5)
    rect.set_height(10)
    assert rect.area() == 50, "Area should be 50"


# Works with Rectangle
rectangle = Rectangle(0, 0)
test_rectangle(rectangle)  # ✅ PASS: area = 50

# Fails with Square (LSP violation!)
square = Square(0, 0)
test_rectangle(square)  # ❌ FAIL: area = 100 (not 50!)
# Square changed height when we set width!
```

**Problem:** Square can't be substituted for Rectangle because it breaks the expected behavior.

### Example: Good Design

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    """Base abstraction for all shapes"""
    @abstractmethod
    def area(self):
        pass


class Rectangle(Shape):
    """Rectangle implementation"""
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def set_width(self, width):
        self.width = width

    def set_height(self, height):
        self.height = height

    def area(self):
        return self.width * self.height


class Square(Shape):
    """Square implementation - NO LONGER inherits from Rectangle"""
    def __init__(self, side):
        self.side = side

    def set_side(self, side):
        self.side = side

    def area(self):
        return self.side ** 2


# Both Rectangle and Square are Shapes, but independent
# No LSP violation because Square doesn't pretend to be a Rectangle
```

### Real-World LSP Guidelines

**Subclasses should:**
- ✅ Not strengthen preconditions (make inputs more restrictive)
- ✅ Not weaken postconditions (make outputs less reliable)
- ✅ Preserve invariants (maintain class guarantees)
- ✅ Not throw new exceptions for methods in base class

**Example:**
```python
class Bird(ABC):
    @abstractmethod
    def move(self): pass

class Sparrow(Bird):
    def move(self):
        print("Flying")  # ✅ Valid

class Penguin(Bird):
    def move(self):
        print("Walking")  # ✅ Valid, different but still "move"

# Bad: Penguin with fly() method would violate LSP
# class Penguin(Bird):
#     def fly(self):
#         raise Exception("Penguins can't fly!")  # ❌ LSP violation
```

---

## I - Interface Segregation Principle (ISP)

### Definition
**"Clients should not be forced to depend on interfaces they don't use."**

Many specific interfaces are better than one general-purpose interface. Don't force classes to implement methods they don't need.

### Example: Bad Design

```python
from abc import ABC, abstractmethod

class Printer(ABC):
    """
    VIOLATION: Too broad, forces all implementations to have all methods
    """
    @abstractmethod
    def print(self, document):
        pass

    @abstractmethod
    def scan(self, document):
        pass

    @abstractmethod
    def fax(self, document):
        pass

    @abstractmethod
    def staple(self, document):
        pass


class SimplePrinter(Printer):
    """
    BAD: Forced to implement methods it doesn't support
    """
    def print(self, document):
        print(f"Printing: {document}")

    def scan(self, document):
        raise NotImplementedError("SimplePrinter doesn't support scanning")

    def fax(self, document):
        raise NotImplementedError("SimplePrinter doesn't support faxing")

    def staple(self, document):
        raise NotImplementedError("SimplePrinter doesn't support stapling")


class AllInOnePrinter(Printer):
    """This works, but requires simple printers to implement everything"""
    def print(self, document):
        print(f"Printing: {document}")

    def scan(self, document):
        print(f"Scanning: {document}")

    def fax(self, document):
        print(f"Faxing: {document}")

    def staple(self, document):
        print(f"Stapling: {document}")
```

**Problem:** SimplePrinter is forced to implement methods it doesn't support, resulting in runtime errors.

### Example: Good Design

```python
from abc import ABC, abstractmethod

# Segregated interfaces - each focused on specific functionality

class Printable(ABC):
    """Interface for printing capability"""
    @abstractmethod
    def print(self, document):
        pass


class Scannable(ABC):
    """Interface for scanning capability"""
    @abstractmethod
    def scan(self, document):
        pass


class Faxable(ABC):
    """Interface for faxing capability"""
    @abstractmethod
    def fax(self, document):
        pass


class Stapleable(ABC):
    """Interface for stapling capability"""
    @abstractmethod
    def staple(self, document):
        pass


# Implementations choose which interfaces to implement

class SimplePrinter(Printable):
    """Only implements what it supports"""
    def print(self, document):
        print(f"Printing: {document}")


class ScannerPrinter(Printable, Scannable):
    """Supports printing and scanning"""
    def print(self, document):
        print(f"Printing: {document}")

    def scan(self, document):
        print(f"Scanning: {document}")


class AllInOnePrinter(Printable, Scannable, Faxable, Stapleable):
    """Supports everything"""
    def print(self, document):
        print(f"Printing: {document}")

    def scan(self, document):
        print(f"Scanning: {document}")

    def fax(self, document):
        print(f"Faxing: {document}")

    def staple(self, document):
        print(f"Stapling: {document}")


# Usage:
def print_document(printer: Printable, doc):
    """Function only depends on Printable interface"""
    printer.print(doc)

simple = SimplePrinter()
advanced = AllInOnePrinter()

print_document(simple, "Report.pdf")    # ✅ Works
print_document(advanced, "Report.pdf")  # ✅ Works
```

**Benefits:**
- SimplePrinter only implements what it needs
- No NotImplementedError exceptions
- Functions can depend on minimal interfaces
- Easy to add new capabilities

---

## D - Dependency Inversion Principle (DIP)

### Definition
**"High-level modules should not depend on low-level modules. Both should depend on abstractions."**

**Also:** "Abstractions should not depend on details. Details should depend on abstractions."

This principle promotes dependency injection and programming to interfaces, not implementations.

### Example: Bad Design

```python
class MySQLDatabase:
    """Low-level module (concrete implementation)"""
    def connect(self):
        print("Connecting to MySQL database")

    def save(self, data):
        print(f"Saving '{data}' to MySQL")


class UserService:
    """
    VIOLATION: High-level module depends on low-level concrete class
    Hard to switch databases or test
    """
    def __init__(self):
        self.database = MySQLDatabase()  # ❌ Tight coupling!

    def create_user(self, name):
        user_data = f"User: {name}"
        self.database.save(user_data)


# Problem: Can't easily switch to PostgreSQL or use a mock for testing
service = UserService()
service.create_user("Alice")
```

**Problems:**
- UserService is tightly coupled to MySQLDatabase
- Can't switch to PostgreSQL without changing UserService code
- Hard to test (need actual MySQL connection)
- Violates DIP - depends on concrete implementation

### Example: Good Design

```python
from abc import ABC, abstractmethod

# Abstraction (high-level interface)
class Database(ABC):
    """Abstract interface that both modules depend on"""
    @abstractmethod
    def connect(self):
        pass

    @abstractmethod
    def save(self, data):
        pass


# Low-level modules (concrete implementations)
class MySQLDatabase(Database):
    """MySQL implementation"""
    def connect(self):
        print("Connecting to MySQL database")

    def save(self, data):
        print(f"Saving '{data}' to MySQL")


class PostgreSQLDatabase(Database):
    """PostgreSQL implementation"""
    def connect(self):
        print("Connecting to PostgreSQL database")

    def save(self, data):
        print(f"Saving '{data}' to PostgreSQL")


class MockDatabase(Database):
    """Mock for testing"""
    def connect(self):
        print("Mock connection")

    def save(self, data):
        print(f"Mock save: {data}")
        self.last_saved = data


# High-level module (depends on abstraction)
class UserService:
    """
    GOOD: Depends on Database abstraction, not concrete class
    """
    def __init__(self, database: Database):  # Dependency injection
        self.database = database
        self.database.connect()

    def create_user(self, name):
        user_data = f"User: {name}"
        self.database.save(user_data)


# Usage: Easily switch implementations
mysql_service = UserService(MySQLDatabase())
mysql_service.create_user("Alice")

postgres_service = UserService(PostgreSQLDatabase())
postgres_service.create_user("Bob")

# Testing: Use mock
mock_db = MockDatabase()
test_service = UserService(mock_db)
test_service.create_user("Charlie")
assert mock_db.last_saved == "User: Charlie"  # ✅ Easy to test
```

**Benefits:**
- UserService doesn't care which database is used
- Easy to switch databases (just pass different instance)
- Easy to test (use MockDatabase)
- Follows DIP - both depend on Database abstraction

### Dependency Injection Patterns

```python
# 1. Constructor Injection (Recommended)
class Service:
    def __init__(self, dependency: Interface):
        self.dependency = dependency

# 2. Setter Injection
class Service:
    def set_dependency(self, dependency: Interface):
        self.dependency = dependency

# 3. Interface Injection
class DependencyInjectable(ABC):
    @abstractmethod
    def inject_dependency(self, dependency: Interface):
        pass
```

---

## SOLID in Practice: Complete Example

Let's design a **Notification System** using all SOLID principles:

```python
from abc import ABC, abstractmethod
from typing import List

# ========== S - Single Responsibility ==========

class User:
    """Represents a user - SINGLE RESPONSIBILITY: User data"""
    def __init__(self, name, email, phone):
        self.name = name
        self.email = email
        self.phone = phone


class Message:
    """Represents a message - SINGLE RESPONSIBILITY: Message data"""
    def __init__(self, title, body):
        self.title = title
        self.body = body


# ========== O/I - Open/Closed + Interface Segregation ==========

class NotificationChannel(ABC):
    """Interface for notification channels"""
    @abstractmethod
    def send(self, user: User, message: Message):
        pass


class EmailChannel(NotificationChannel):
    """Email notification - EXTENDS without modifying existing code"""
    def send(self, user: User, message: Message):
        print(f"Sending email to {user.email}: {message.title}")


class SMSChannel(NotificationChannel):
    """SMS notification - EXTENDS without modifying existing code"""
    def send(self, user: User, message: Message):
        print(f"Sending SMS to {user.phone}: {message.body}")


class PushChannel(NotificationChannel):
    """Push notification - NEW channel, NO changes to existing code"""
    def send(self, user: User, message: Message):
        print(f"Sending push to {user.name}: {message.title}")


# ========== D - Dependency Inversion ==========

class NotificationService:
    """
    High-level module depends on NotificationChannel abstraction
    Uses Dependency Injection
    """
    def __init__(self, channels: List[NotificationChannel]):
        self.channels = channels

    def notify(self, user: User, message: Message):
        """Send notification via all configured channels"""
        for channel in self.channels:
            channel.send(user, message)


# ========== L - Liskov Substitution ==========

def send_urgent_alert(channel: NotificationChannel, user: User):
    """
    This function works with ANY NotificationChannel
    All channels are substitutable
    """
    urgent_message = Message("URGENT", "System alert!")
    channel.send(user, urgent_message)


# Usage demonstrating all SOLID principles:
if __name__ == "__main__":
    # Create user
    alice = User("Alice", "alice@example.com", "+1234567890")

    # Create message
    message = Message("Welcome", "Welcome to our service!")

    # Configure channels (Dependency Injection)
    email = EmailChannel()
    sms = SMSChannel()
    push = PushChannel()

    # Create service with injected dependencies
    notification_service = NotificationService([email, sms, push])

    # Send notification
    notification_service.notify(alice, message)

    # Liskov Substitution: any channel works
    send_urgent_alert(email, alice)
    send_urgent_alert(sms, alice)
    send_urgent_alert(push, alice)  # New channel, same function

    # Easy to add new channel (Open/Closed)
    class SlackChannel(NotificationChannel):
        def send(self, user: User, message: Message):
            print(f"Sending Slack message to {user.name}: {message.body}")

    slack = SlackChannel()
    send_urgent_alert(slack, alice)  # ✅ Works without any changes!
```

**SOLID Principles Applied:**
- **S:** User, Message, EmailChannel each have single responsibility
- **O:** Can add new channels (SlackChannel) without modifying existing code
- **L:** All NotificationChannel implementations are substitutable
- **I:** NotificationChannel interface is focused (only `send` method)
- **D:** NotificationService depends on abstraction, not concrete classes

---

## Summary

| Principle | Key Question | Guideline |
|-----------|-------------|-----------|
| **Single Responsibility** | "How many reasons does this class have to change?" | One class, one responsibility |
| **Open/Closed** | "Can I add features without changing existing code?" | Extend, don't modify |
| **Liskov Substitution** | "Can I replace parent with child without breaking?" | Subtypes must behave like supertypes |
| **Interface Segregation** | "Am I forced to implement methods I don't need?" | Many small interfaces > one large |
| **Dependency Inversion** | "Do I depend on concrete classes?" | Depend on abstractions, inject dependencies |

**Remember:** SOLID principles are guidelines, not strict rules. Apply them pragmatically based on your specific needs. Sometimes violating a principle is acceptable if it simplifies the design.

---

**Next:** Study [Design Patterns Overview](design-patterns-overview.md) to see how design patterns implement SOLID principles.
