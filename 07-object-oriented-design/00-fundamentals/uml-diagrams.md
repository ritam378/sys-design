# UML Diagrams for Object-Oriented Design

## What is UML?

**UML (Unified Modeling Language)** is a standardized visual language for modeling software systems. It provides a way to visualize system architecture, design, and implementation through diagrams.

**Why UML Matters:**
- **Communication:** Provides common visual language for teams
- **Design:** Helps plan system architecture before coding
- **Documentation:** Documents existing systems clearly
- **Interviews:** Essential for explaining OOD solutions

## UML Diagram Types

UML includes 14 different diagram types, but for **OOD interviews and software design**, you primarily need to know:

### Most Important for Interviews

1. **Class Diagram** - Shows classes, attributes, methods, and relationships (MOST IMPORTANT)
2. **Sequence Diagram** - Shows object interactions over time
3. **Use Case Diagram** - Shows system functionality from user perspective

### Less Common in Interviews

4. **Activity Diagram** - Shows workflow/process flow
5. **State Diagram** - Shows state transitions
6. **Component Diagram** - Shows system components

**Focus Area:** This guide emphasizes **Class Diagrams** as they're most critical for OOD interviews.

---

# Class Diagrams

## Overview

A **Class Diagram** is a structural diagram that shows:
- Classes in the system
- Attributes (fields) of each class
- Methods (operations) of each class
- Relationships between classes

## Basic Class Representation

### Standard UML Notation

```
┌─────────────────────────┐
│      ClassName          │  ← Class name (centered, bold)
├─────────────────────────┤
│ - privateAttribute      │  ← Attributes section
│ + publicAttribute       │
│ # protectedAttribute    │
├─────────────────────────┤
│ + publicMethod()        │  ← Methods section
│ - privateMethod()       │
│ # protectedMethod()     │
└─────────────────────────┘
```

### Visibility Modifiers

| Symbol | Visibility | Python Equivalent |
|--------|------------|-------------------|
| `+` | Public | `attribute` |
| `-` | Private | `__attribute` or `_attribute` |
| `#` | Protected | `_attribute` |
| `~` | Package | (not common in Python) |

### Example: User Class

```
┌─────────────────────────┐
│         User            │
├─────────────────────────┤
│ - user_id: int          │
│ - username: str         │
│ - email: str            │
│ - _password: str        │
├─────────────────────────┤
│ + login(): bool         │
│ + logout(): void        │
│ + update_email(         │
│     email: str): bool   │
│ - _hash_password(       │
│     pwd: str): str      │
└─────────────────────────┘
```

**Python Implementation:**
```python
class User:
    def __init__(self, user_id: int, username: str, email: str, password: str):
        self.user_id = user_id      # Public (- in UML, but Python default)
        self.username = username
        self.email = email
        self._password = self._hash_password(password)  # Private

    def login(self) -> bool:
        """Public method"""
        return True

    def logout(self) -> None:
        """Public method"""
        pass

    def update_email(self, email: str) -> bool:
        """Public method"""
        self.email = email
        return True

    def _hash_password(self, pwd: str) -> str:
        """Private method"""
        return f"hashed_{pwd}"
```

---

## Relationships Between Classes

UML defines several types of relationships. Understanding these is **critical** for OOD interviews.

### 1. Association (Uses/Has-A)

**Definition:** General relationship where one class uses or references another.

**Notation:** Solid line connecting classes

```
┌─────────────┐          ┌─────────────┐
│   Student   │─────────>│   Course    │
└─────────────┘          └─────────────┘
```

**Meaning:** Student enrolls in Course

**When to Use:**
- One class needs to interact with another
- Loose coupling
- Objects exist independently

**Python Example:**
```python
class Course:
    def __init__(self, name: str):
        self.name = name

class Student:
    def __init__(self, name: str):
        self.name = name
        self.courses: list[Course] = []

    def enroll(self, course: Course):
        """Student uses/associates with Course"""
        self.courses.append(course)

# Both objects exist independently
course = Course("Math 101")
student = Student("Alice")
student.enroll(course)  # Association
```

### 2. Aggregation (Has-A, Weak Ownership)

**Definition:** "Has-a" relationship where child can exist independently of parent. Represents a whole-part relationship with weak ownership.

**Notation:** Line with hollow diamond on parent side

```
┌─────────────┐       ┌─────────────┐
│ Department  │◇──────│   Professor │
└─────────────┘       └─────────────┘
```

**Meaning:** Department has Professors, but Professors can exist without Department

**When to Use:**
- Part can exist without whole
- Shared ownership
- Weak lifecycle dependency

**Python Example:**
```python
class Professor:
    def __init__(self, name: str):
        self.name = name

class Department:
    def __init__(self, name: str):
        self.name = name
        self.professors: list[Professor] = []

    def add_professor(self, prof: Professor):
        self.professors.append(prof)

# Professors exist independently
prof1 = Professor("Dr. Smith")
prof2 = Professor("Dr. Jones")

dept = Department("Computer Science")
dept.add_professor(prof1)
dept.add_professor(prof2)

# If department is deleted, professors still exist
del dept
print(prof1.name)  # Still exists
```

### 3. Composition (Has-A, Strong Ownership)

**Definition:** Strong "has-a" relationship where child cannot exist without parent. Child's lifecycle depends on parent.

**Notation:** Line with filled diamond on parent side

```
┌─────────────┐       ┌─────────────┐
│    House    │◆──────│    Room     │
└─────────────┘       └─────────────┘
```

**Meaning:** House owns Rooms; if House is destroyed, Rooms are too

**When to Use:**
- Part cannot exist without whole
- Strong ownership
- Tight lifecycle dependency

**Python Example:**
```python
class Room:
    def __init__(self, name: str, size: int):
        self.name = name
        self.size = size

class House:
    def __init__(self, address: str):
        self.address = address
        # Rooms created with House, owned by House
        self.rooms: list[Room] = [
            Room("Living Room", 300),
            Room("Bedroom", 200),
            Room("Kitchen", 150)
        ]

# Rooms created inside House, lifecycle tied to House
house = House("123 Main St")
# If house is deleted, rooms are implicitly deleted
del house
# Rooms no longer accessible
```

### 4. Inheritance (Is-A)

**Definition:** Subclass inherits from superclass. Represents specialization.

**Notation:** Line with hollow triangle pointing to parent

```
           ┌─────────────┐
           │   Animal    │
           └─────────────┘
                 △
                 │
        ┌────────┴────────┐
        │                 │
┌───────┴──────┐  ┌───────┴──────┐
│     Dog      │  │     Cat      │
└──────────────┘  └──────────────┘
```

**Meaning:** Dog IS-A Animal, Cat IS-A Animal

**When to Use:**
- True specialization relationship
- Shared behavior/attributes
- Polymorphism needed

**Python Example:**
```python
class Animal:
    def __init__(self, name: str):
        self.name = name

    def make_sound(self) -> str:
        return "Some sound"

class Dog(Animal):
    def make_sound(self) -> str:
        return "Woof!"

class Cat(Animal):
    def make_sound(self) -> str:
        return "Meow!"

# IS-A relationship
dog: Animal = Dog("Buddy")  # Dog IS-A Animal
cat: Animal = Cat("Whiskers")  # Cat IS-A Animal

print(dog.make_sound())  # Woof!
print(cat.make_sound())  # Meow!
```

### 5. Realization/Implementation (Implements)

**Definition:** Class implements an interface or abstract class.

**Notation:** Dashed line with hollow triangle pointing to interface

```
       ┌─────────────────┐
       │  <<interface>>  │
       │    Drawable     │
       └─────────────────┘
                △
                ┆
                ┆ (dashed line)
        ┌───────┴────────┐
        │                │
┌───────┴──────┐  ┌──────┴───────┐
│   Circle     │  │   Rectangle  │
└──────────────┘  └──────────────┘
```

**Meaning:** Circle and Rectangle implement Drawable interface

**Python Example:**
```python
from abc import ABC, abstractmethod

class Drawable(ABC):
    """Interface (abstract class in Python)"""

    @abstractmethod
    def draw(self) -> str:
        pass

    @abstractmethod
    def get_area(self) -> float:
        pass

class Circle(Drawable):
    """Implements Drawable interface"""

    def __init__(self, radius: float):
        self.radius = radius

    def draw(self) -> str:
        return "Drawing circle"

    def get_area(self) -> float:
        return 3.14159 * self.radius ** 2

class Rectangle(Drawable):
    """Implements Drawable interface"""

    def __init__(self, width: float, height: float):
        self.width = width
        self.height = height

    def draw(self) -> str:
        return "Drawing rectangle"

    def get_area(self) -> float:
        return self.width * self.height
```

### 6. Dependency (Uses)

**Definition:** One class depends on another temporarily (method parameter, local variable).

**Notation:** Dashed line with arrow

```
┌─────────────┐          ┌─────────────┐
│ OrderService│─ ─ ─ ─>  │   Logger    │
└─────────────┘          └─────────────┘
```

**Meaning:** OrderService depends on Logger

**When to Use:**
- Method parameter
- Local variable
- Temporary usage

**Python Example:**
```python
class Logger:
    def log(self, message: str):
        print(f"LOG: {message}")

class OrderService:
    def process_order(self, order_id: int, logger: Logger):
        """Depends on Logger - passed as parameter"""
        logger.log(f"Processing order {order_id}")
        # Process order...
        logger.log(f"Order {order_id} completed")

# Usage
logger = Logger()
service = OrderService()
service.process_order(123, logger)  # Temporary dependency
```

---

## Relationship Comparison Table

| Relationship | Notation | Strength | Lifecycle | Example |
|-------------|----------|----------|-----------|---------|
| **Association** | Solid line `────>` | Weak | Independent | Student → Course |
| **Aggregation** | Hollow diamond `◇───` | Weak | Independent | Department ◇─ Professor |
| **Composition** | Filled diamond `◆───` | Strong | Dependent | House ◆─ Room |
| **Inheritance** | Hollow triangle `△` | Strong | N/A | Dog △ Animal |
| **Realization** | Dashed triangle `△┆` | Strong | N/A | Circle △┆ Drawable |
| **Dependency** | Dashed arrow `─ ─>` | Weakest | Temporary | Service ─ ─> Logger |

**Memory Aid:**
- **Diamond = Ownership** (hollow = weak, filled = strong)
- **Triangle = Hierarchy** (hollow = inheritance, dashed = interface)
- **Dashed = Weak/Temporary**
- **Solid = Stronger relationship**

---

## Multiplicity (Cardinality)

Multiplicity indicates how many instances of one class relate to instances of another.

### Common Multiplicity Notations

| Notation | Meaning |
|----------|---------|
| `1` | Exactly one |
| `0..1` | Zero or one |
| `*` or `0..*` | Zero or more |
| `1..*` | One or more |
| `n..m` | Between n and m |

### Examples

#### One-to-One (1:1)

```
┌─────────────┐  1       1  ┌─────────────┐
│   Person    │─────────────│   Passport  │
└─────────────┘             └─────────────┘
```

**Meaning:** Each Person has exactly one Passport, each Passport belongs to exactly one Person

```python
class Passport:
    def __init__(self, passport_number: str, person: 'Person'):
        self.passport_number = passport_number
        self.person = person  # 1:1 relationship

class Person:
    def __init__(self, name: str):
        self.name = name
        self.passport: Passport | None = None  # 1:1 relationship

    def issue_passport(self, passport_number: str):
        self.passport = Passport(passport_number, self)
```

#### One-to-Many (1:*)

```
┌─────────────┐  1      *  ┌─────────────┐
│   Library   │────────────│    Book     │
└─────────────┘            └─────────────┘
```

**Meaning:** One Library has many Books, each Book belongs to one Library

```python
class Book:
    def __init__(self, title: str):
        self.title = title

class Library:
    def __init__(self, name: str):
        self.name = name
        self.books: list[Book] = []  # 1:* relationship

    def add_book(self, book: Book):
        self.books.append(book)
```

#### Many-to-Many (*:*)

```
┌─────────────┐  *      *  ┌─────────────┐
│   Student   │────────────│   Course    │
└─────────────┘            └─────────────┘
```

**Meaning:** Students can enroll in many Courses, Courses can have many Students

```python
class Course:
    def __init__(self, name: str):
        self.name = name
        self.students: list['Student'] = []  # *:* relationship

class Student:
    def __init__(self, name: str):
        self.name = name
        self.courses: list[Course] = []  # *:* relationship

    def enroll(self, course: Course):
        self.courses.append(course)
        course.students.append(self)
```

---

## Complete Class Diagram Example: E-Commerce System

Let's see all concepts in action:

```
                    ┌──────────────────┐
                    │   <<abstract>>   │
                    │      User        │
                    ├──────────────────┤
                    │ - userId: int    │
                    │ - email: str     │
                    │ - password: str  │
                    ├──────────────────┤
                    │ + login(): bool  │
                    │ + logout(): void │
                    └──────────────────┘
                            △
                            │ (inheritance)
                ┌───────────┴───────────┐
                │                       │
        ┌───────┴────────┐      ┌──────┴───────┐
        │    Customer    │      │    Admin     │
        ├────────────────┤      ├──────────────┤
        │ - address: str │      │ - role: str  │
        ├────────────────┤      ├──────────────┤
        │ + placeOrder() │      │ + manage()   │
        └────────────────┘      └──────────────┘
                │
                │ 1
                │
                │ *
        ┌───────┴────────┐
        │     Order      │
        ├────────────────┤
        │ - orderId: int │
        │ - date: Date   │
        │ - status: str  │
        ├────────────────┤
        │ + calculate()  │
        │ + ship()       │
        └────────────────┘
                │
                │ ◆ (composition)
                │
                │ *
        ┌───────┴────────┐
        │   OrderItem    │
        ├────────────────┤
        │ - quantity: int│
        │ - price: float │
        ├────────────────┤
        │ + getSubtotal()│
        └────────────────┘
                │
                │ (association)
                │
                ▼
        ┌────────────────┐
        │    Product     │
        ├────────────────┤
        │ - productId    │
        │ - name: str    │
        │ - price: float │
        ├────────────────┤
        │ + getDetails() │
        └────────────────┘
```

**Relationships Explained:**

1. **User → Customer/Admin (Inheritance):** Customer and Admin inherit from User
2. **Customer → Order (1:*):** One Customer can have many Orders
3. **Order → OrderItem (Composition ◆):** Order owns OrderItems (if Order deleted, OrderItems deleted)
4. **OrderItem → Product (Association):** OrderItem references Product

**Python Implementation:**

```python
from abc import ABC, abstractmethod
from datetime import datetime
from typing import List

# Abstract base class
class User(ABC):
    def __init__(self, user_id: int, email: str, password: str):
        self.user_id = user_id
        self.email = email
        self._password = password

    @abstractmethod
    def login(self) -> bool:
        pass

    def logout(self) -> None:
        print(f"User {self.email} logged out")

# Inheritance
class Customer(User):
    def __init__(self, user_id: int, email: str, password: str, address: str):
        super().__init__(user_id, email, password)
        self.address = address
        self.orders: List['Order'] = []  # 1:* relationship

    def login(self) -> bool:
        print(f"Customer {self.email} logged in")
        return True

    def place_order(self, order: 'Order'):
        self.orders.append(order)

class Admin(User):
    def __init__(self, user_id: int, email: str, password: str, role: str):
        super().__init__(user_id, email, password)
        self.role = role

    def login(self) -> bool:
        print(f"Admin {self.email} logged in")
        return True

    def manage(self):
        print(f"Admin managing system with role: {self.role}")

# Product class
class Product:
    def __init__(self, product_id: int, name: str, price: float):
        self.product_id = product_id
        self.name = name
        self.price = price

    def get_details(self) -> str:
        return f"{self.name}: ${self.price}"

# OrderItem - associated with Product
class OrderItem:
    def __init__(self, product: Product, quantity: int):
        self.product = product  # Association
        self.quantity = quantity
        self.price = product.price

    def get_subtotal(self) -> float:
        return self.price * self.quantity

# Order - composes OrderItems
class Order:
    def __init__(self, order_id: int, customer: Customer):
        self.order_id = order_id
        self.customer = customer
        self.date = datetime.now()
        self.status = "Pending"
        self.items: List[OrderItem] = []  # Composition - items owned by order

    def add_item(self, item: OrderItem):
        self.items.append(item)

    def calculate_total(self) -> float:
        return sum(item.get_subtotal() for item in self.items)

    def ship(self):
        self.status = "Shipped"
        print(f"Order {self.order_id} shipped to {self.customer.address}")

# Usage
customer = Customer(1, "alice@example.com", "pass123", "123 Main St")

product1 = Product(101, "Laptop", 999.99)
product2 = Product(102, "Mouse", 29.99)

order = Order(1001, customer)
order.add_item(OrderItem(product1, 1))
order.add_item(OrderItem(product2, 2))

customer.place_order(order)

print(f"Total: ${order.calculate_total()}")
order.ship()
```

---

## Drawing UML Diagrams

### Tools for Creating UML Diagrams

#### For Interviews (Quick Sketching)

1. **Whiteboard/Paper** - Best for in-person interviews
2. **Drawing tools** - Excalidraw, draw.io, Lucidchart
3. **ASCII Art** - For virtual interviews, code documents

#### Professional Tools

1. **PlantUML** - Text-based, version control friendly
2. **Draw.io / diagrams.net** - Free, web-based
3. **Lucidchart** - Professional, collaborative
4. **Visual Paradigm** - Full-featured UML tool

### ASCII UML for Code/Markdown

**Advantages:**
- Works in plain text
- Version control friendly
- No special tools needed

**Characters to use:**
```
Box drawing: ┌ ┐ └ ┘ ├ ┤ ┬ ┴ ┼ ─ │
Arrows: → ← ↑ ↓ ⇒ ⇐
Shapes: △ ▽ ◁ ▷ ◆ ◇
```

**Example:**
```
┌─────────────────┐
│   BankAccount   │
├─────────────────┤
│ - balance       │
│ - accountNumber │
├─────────────────┤
│ + deposit()     │
│ + withdraw()    │
└─────────────────┘
        △
        │
┌───────┴────────┐
│                │
┌─────┴─────┐  ┌─┴──────────┐
│ Checking  │  │  Savings   │
└───────────┘  └────────────┘
```

---

## Interview Tips for UML Diagrams

### During the Interview

**1. Start Simple**
- Begin with main classes
- Add relationships progressively
- Don't overwhelm with details initially

**2. Talk While Drawing**
- Explain your thinking
- Describe relationships as you draw
- Ask clarifying questions

**3. Focus on Key Relationships**
- Emphasize important associations
- Show inheritance hierarchies
- Highlight composition vs aggregation

**4. Use Standard Notation**
- Stick to basic UML symbols
- Explain non-standard notation if used
- Be consistent

**5. Iterate**
- Start high-level
- Add detail as needed
- Be ready to refactor

### Common Interview Scenarios

**Scenario 1: "Design a Parking Lot"**

Quick approach:
1. Identify main entities: ParkingLot, ParkingSpot, Vehicle, Ticket
2. Draw inheritance: Vehicle → Car/Truck/Motorcycle
3. Show composition: ParkingLot ◆─ ParkingSpot
4. Add methods on key classes

**Scenario 2: "Design a Library Management System"**

Quick approach:
1. Main entities: Library, Book, Member, Librarian
2. Show relationships: Library ◇─ Book, Member ─ Book (borrow)
3. Add multiplicity: Member 1 ─ * Book
4. Discuss checkout/return methods

### What Interviewers Look For

- **Correctness:** Appropriate use of UML notation
- **Completeness:** All major entities represented
- **Relationships:** Correct relationship types
- **Clarity:** Easy to understand diagram
- **Communication:** Explaining design decisions
- **Iteration:** Improving design based on feedback

---

## Common Mistakes to Avoid

### 1. Overcomplicating Diagrams

**Bad:**
- Too many classes at once
- Every possible attribute/method
- All edge cases upfront

**Good:**
- Start with 4-6 main classes
- Key attributes/methods only
- Add details when asked

### 2. Wrong Relationship Types

**Bad:**
```
Order ────> OrderItem  (Should be composition ◆)
Dog ────> Animal       (Should be inheritance △)
```

**Good:**
```
Order ◆──── OrderItem  (Composition - strong ownership)
Dog ─────△─ Animal     (Inheritance - is-a relationship)
```

### 3. Ignoring Multiplicity

**Bad:**
```
Student ──── Course  (How many?)
```

**Good:**
```
Student * ──── * Course  (Many-to-many relationship)
```

### 4. Inconsistent Notation

**Bad:**
- Mixing different arrow styles randomly
- Inconsistent visibility markers
- Non-standard symbols

**Good:**
- Stick to standard UML notation
- Consistent style throughout
- Explain any variations

---

## Practice Problems

### Exercise 1: ATM System

**Task:** Draw a class diagram for an ATM system with:
- Users with accounts
- Different account types (Checking, Savings)
- Transactions (Deposit, Withdrawal)
- ATM machine

**Key relationships to show:**
- Inheritance (Account types)
- Composition (Bank owns ATMs)
- Association (User has Accounts)

### Exercise 2: Social Media Platform

**Task:** Draw a class diagram for a social media platform with:
- Users who post content
- Posts with comments
- Friend connections
- Notifications

**Key relationships to show:**
- User creates Posts (1:*)
- Post has Comments (1:*)
- User friends with Users (*:*)
- Self-referential relationships

### Exercise 3: Online Shopping

**Task:** Draw a class diagram for an online shopping system with:
- Products in categories
- Shopping cart
- Orders with items
- Payment methods

**Key relationships to show:**
- Inheritance (Payment types)
- Composition (Order owns OrderItems)
- Aggregation (Cart contains Products)

---

## UML Best Practices

### 1. Keep It Simple

- Don't model everything upfront
- Focus on core entities first
- Add complexity iteratively

### 2. Use Meaningful Names

- Clear, descriptive class names
- Verb phrases for methods
- Noun phrases for attributes

### 3. Show Important Relationships

- Prioritize key associations
- Clearly indicate relationship types
- Use multiplicity where relevant

### 4. Balance Detail and Clarity

- Enough detail to understand design
- Not so much it becomes cluttered
- Use separate diagrams for different aspects

### 5. Validate Your Design

- Does it satisfy requirements?
- Are relationships logical?
- Can it handle edge cases?

---

## Summary

### Key Takeaways

**UML Class Diagrams show:**
1. **Classes:** Boxes with name, attributes, methods
2. **Visibility:** +public, -private, #protected
3. **Relationships:** Association, Aggregation, Composition, Inheritance, Realization, Dependency
4. **Multiplicity:** 1, *, 0..1, 1..*, etc.

**Relationship Memory Aid:**
- **Solid line = Stronger** (association, aggregation, composition)
- **Dashed line = Weaker** (dependency, realization)
- **Diamond = Ownership** (hollow = weak, filled = strong)
- **Triangle = Hierarchy** (inheritance, interface)

**Interview Strategy:**
1. Start simple, iterate
2. Talk through your design
3. Use standard notation
4. Focus on key relationships
5. Be ready to refactor

### Next Steps

1. **Practice drawing:** Sketch diagrams for common systems
2. **Study examples:** Analyze UML diagrams in design pattern books
3. **Implement:** Convert diagrams to code
4. **Review:** Compare your diagrams with reference solutions

### Related Resources

- Design Patterns Overview: `design-patterns-overview.md`
- OOD vs System Design: `ood-vs-system-design.md`
- Interview Approach: `interview-approach.md`
- Practice Problems: `02-beginner-problems/`

---

Remember: **UML is a communication tool. Clarity is more important than perfection.**
