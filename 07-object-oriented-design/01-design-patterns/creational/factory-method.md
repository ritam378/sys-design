# Factory Method Pattern

## Intent
Define an interface for creating an object, but let subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses.

## Problem
- Object creation is complex or depends on runtime conditions
- Client shouldn't know about concrete classes
- Need flexibility to add new product types without changing existing code

## Solution
Create objects through a factory method rather than direct constructor calls. Subclasses override the factory method to create different types of objects.

## Structure
```
    ┌──────────────────┐
    │    Creator       │
    ├──────────────────┤
    │ + factory_method()│  (abstract)
    │ + some_operation()│
    └──────────────────┘
            △
            │
    ┌───────┴────────┐
    │                │
┌───┴─────────┐  ┌──┴──────────┐
│ConcreteA    │  │ConcreteB    │
├─────────────┤  ├─────────────┤
│factory_method│ │factory_method│
└─────────────┘  └─────────────┘
    │                │
    │creates         │creates
    ▼                ▼
┌─────────┐      ┌─────────┐
│ProductA │      │ProductB │
└─────────┘      └─────────┘
```

## Python Implementation

### Example 1: Document Creation
```python
from abc import ABC, abstractmethod

# Product interface
class Document(ABC):
    @abstractmethod
    def open(self) -> str:
        pass

    @abstractmethod
    def save(self) -> str:
        pass

# Concrete products
class PDFDocument(Document):
    def open(self) -> str:
        return "Opening PDF document"

    def save(self) -> str:
        return "Saving as PDF"

class WordDocument(Document):
    def open(self) -> str:
        return "Opening Word document"

    def save(self) -> str:
        return "Saving as .docx"

class ExcelDocument(Document):
    def open(self) -> str:
        return "Opening Excel spreadsheet"

    def save(self) -> str:
        return "Saving as .xlsx"

# Creator
class DocumentCreator(ABC):
    @abstractmethod
    def create_document(self) -> Document:
        """Factory method"""
        pass

    def new_document(self) -> str:
        """Template method using factory method"""
        doc = self.create_document()
        return f"{doc.open()}\n{doc.save()}"

# Concrete creators
class PDFCreator(DocumentCreator):
    def create_document(self) -> Document:
        return PDFDocument()

class WordCreator(DocumentCreator):
    def create_document(self) -> Document:
        return WordDocument()

class ExcelCreator(DocumentCreator):
    def create_document(self) -> Document:
        return ExcelDocument()

# Usage
def client_code(creator: DocumentCreator):
    print(creator.new_document())

client_code(PDFCreator())   # Creates PDF
client_code(WordCreator())  # Creates Word doc
```

### Example 2: Payment Processing
```python
from abc import ABC, abstractmethod

class PaymentProcessor(ABC):
    @abstractmethod
    def process_payment(self, amount: float) -> str:
        pass

class CreditCardProcessor(PaymentProcessor):
    def process_payment(self, amount: float) -> str:
        return f"Processing ${amount} via Credit Card"

class PayPalProcessor(PaymentProcessor):
    def process_payment(self, amount: float) -> str:
        return f"Processing ${amount} via PayPal"

class CryptoProcessor(PaymentProcessor):
    def process_payment(self, amount: float) -> str:
        return f"Processing ${amount} via Cryptocurrency"

# Factory
class PaymentFactory(ABC):
    @abstractmethod
    def create_processor(self) -> PaymentProcessor:
        pass

    def process(self, amount: float) -> str:
        processor = self.create_processor()
        return processor.process_payment(amount)

class CreditCardFactory(PaymentFactory):
    def create_processor(self) -> PaymentProcessor:
        return CreditCardProcessor()

class PayPalFactory(PaymentFactory):
    def create_processor(self) -> PaymentProcessor:
        return PayPalProcessor()

# Usage
factory = CreditCardFactory()
print(factory.process(100.00))  # Processing $100.0 via Credit Card

factory = PayPalFactory()
print(factory.process(50.00))   # Processing $50.0 via PayPal
```

### Example 3: Notification System
```python
from abc import ABC, abstractmethod

class Notification(ABC):
    @abstractmethod
    def send(self, message: str) -> None:
        pass

class EmailNotification(Notification):
    def send(self, message: str) -> None:
        print(f"Sending Email: {message}")

class SMSNotification(Notification):
    def send(self, message: str) -> None:
        print(f"Sending SMS: {message}")

class PushNotification(Notification):
    def send(self, message: str) -> None:
        print(f"Sending Push: {message}")

class NotificationService(ABC):
    @abstractmethod
    def create_notification(self) -> Notification:
        pass

    def notify(self, message: str) -> None:
        notification = self.create_notification()
        notification.send(message)

class EmailService(NotificationService):
    def create_notification(self) -> Notification:
        return EmailNotification()

class SMSService(NotificationService):
    def create_notification(self) -> Notification:
        return SMSNotification()

# Usage
service = EmailService()
service.notify("Hello, World!")  # Sends email

service = SMSService()
service.notify("Hello, World!")  # Sends SMS
```

### Example 4: Simple Factory (Not Pure Factory Method, but Common)
```python
# Simple Factory - determines type at runtime
class ShapeFactory:
    @staticmethod
    def create_shape(shape_type: str):
        if shape_type == "circle":
            return Circle()
        elif shape_type == "square":
            return Square()
        elif shape_type == "triangle":
            return Triangle()
        else:
            raise ValueError(f"Unknown shape type: {shape_type}")

class Shape(ABC):
    @abstractmethod
    def draw(self) -> str:
        pass

class Circle(Shape):
    def draw(self) -> str:
        return "Drawing Circle"

class Square(Shape):
    def draw(self) -> str:
        return "Drawing Square"

class Triangle(Shape):
    def draw(self) -> str:
        return "Drawing Triangle"

# Usage
shape = ShapeFactory.create_shape("circle")
print(shape.draw())  # Drawing Circle
```

## When to Use
- ✅ Don't know exact types/dependencies beforehand
- ✅ Want to provide library users extension points
- ✅ Want to reuse existing objects instead of rebuilding
- ✅ Need to decouple object creation from usage

## Advantages
- **Open/Closed Principle:** Add new products without changing existing code
- **Single Responsibility:** Creation logic in one place
- **Loose Coupling:** Client code doesn't depend on concrete classes
- **Flexibility:** Easy to introduce new product types

## Disadvantages
- **Complexity:** More classes and interfaces
- **Overhead:** May be overkill for simple object creation
- **Indirection:** Extra layer makes code harder to trace

## Real-World Examples

### Django/Flask View Handling
```python
class ViewFactory(ABC):
    @abstractmethod
    def create_view(self):
        pass

    def render(self, request):
        view = self.create_view()
        return view.handle(request)

class ListViewFactory(ViewFactory):
    def create_view(self):
        return ListView()

class DetailViewFactory(ViewFactory):
    def create_view(self):
        return DetailView()
```

### Database Connection
```python
class DatabaseFactory(ABC):
    @abstractmethod
    def create_connection(self):
        pass

class PostgreSQLFactory(DatabaseFactory):
    def create_connection(self):
        return PostgreSQLConnection()

class MySQLFactory(DatabaseFactory):
    def create_connection(self):
        return MySQLConnection()

# Usage based on config
db_type = os.getenv("DB_TYPE", "postgresql")
factory = PostgreSQLFactory() if db_type == "postgresql" else MySQLFactory()
conn = factory.create_connection()
```

## vs. Other Patterns

| Pattern | Key Difference |
|---------|---------------|
| **Abstract Factory** | Creates families of related objects |
| **Builder** | Constructs complex objects step-by-step |
| **Prototype** | Creates objects by cloning |
| **Simple Factory** | Not a pattern; just a function/method |

## Interview Tips

**Q: Difference between Factory Method and Abstract Factory?**
A: Factory Method creates one product; Abstract Factory creates families of related products.

**Q: When would you use Factory Method over direct instantiation?**
A: When you need flexibility to add new types without changing existing code, or when client shouldn't know concrete classes.

**Q: How does Factory Method support Open/Closed Principle?**
A: New product types can be added by creating new creator subclasses without modifying existing code.

## Summary
Factory Method promotes loose coupling by eliminating the need for client code to bind to specific classes. Use when you need flexibility in object creation and want to follow the Open/Closed Principle.
