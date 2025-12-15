# Design Patterns Overview

## What Are Design Patterns?

Design patterns are **reusable solutions to commonly occurring problems** in software design. They represent best practices evolved over time by experienced software developers. Think of them as templates or blueprints that you can customize to solve a particular design problem in your code.

**Key Points:**
- Design patterns are **not finished code** - they are descriptions or templates for how to solve a problem
- They are **language-independent** concepts that can be implemented in any object-oriented language
- They promote **code reuse, flexibility, and maintainability**
- They provide a **common vocabulary** for developers to communicate design ideas

## Why Design Patterns Matter

Understanding design patterns is crucial for several reasons:

- **Interview Success:** Most tech companies (Google, Amazon, Meta, etc.) expect candidates to know common patterns
- **Code Quality:** Patterns help you write more maintainable, flexible, and scalable code
- **Communication:** Patterns provide a shared language among developers ("Let's use the Observer pattern here")
- **Problem-Solving:** Patterns help you recognize and solve common problems quickly
- **Learning from Experts:** Patterns embody decades of collective wisdom from the software community

## The Gang of Four (GoF)

The most influential work on design patterns is the 1994 book "Design Patterns: Elements of Reusable Object-Oriented Software" by Erich Gamma, Richard Helm, Ralph Johnson, and John Vlissides (collectively known as the "Gang of Four" or GoF).

They identified **23 classic design patterns** organized into three categories:

1. **Creational Patterns (5)** - Object creation mechanisms
2. **Structural Patterns (7)** - Object composition and relationships
3. **Behavioral Patterns (11)** - Object interaction and responsibilities

---

# The 23 Gang of Four Patterns

## Category 1: Creational Patterns (5 Patterns)

Creational patterns deal with **object creation mechanisms**, trying to create objects in a manner suitable to the situation. They help make a system independent of how its objects are created, composed, and represented.

### 1. Singleton Pattern

**Purpose:** Ensure a class has only one instance and provide a global point of access to it.

**When to Use:**
- Database connection managers
- Configuration managers
- Logging services
- Thread pools

**Example:**
```python
class DatabaseConnection:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            # Initialize connection
        return cls._instance

# Both variables point to the same instance
db1 = DatabaseConnection()
db2 = DatabaseConnection()
assert db1 is db2  # True
```

**Pros:**
- Controlled access to sole instance
- Reduced namespace pollution
- Lazy initialization possible

**Cons:**
- Difficult to test (global state)
- Violates Single Responsibility Principle
- Can hide dependencies

---

### 2. Factory Method Pattern

**Purpose:** Define an interface for creating an object, but let subclasses decide which class to instantiate.

**When to Use:**
- A class can't anticipate the type of objects it needs to create
- You want to delegate object creation to subclasses
- You want to provide a library of products

**Example:**
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
result = factory.process(100.0)
```

**Pros:**
- Follows Open/Closed Principle
- Decouples client code from concrete classes
- Easy to add new product types

**Cons:**
- Can lead to many subclasses
- More complex than direct instantiation

---

### 3. Abstract Factory Pattern

**Purpose:** Provide an interface for creating families of related or dependent objects without specifying their concrete classes.

**When to Use:**
- System needs to be independent of how its products are created
- System needs to work with multiple families of related products
- You want to enforce that products from the same family are used together

**Example:**
```python
from abc import ABC, abstractmethod

# Abstract Products
class Button(ABC):
    @abstractmethod
    def render(self) -> str:
        pass

class Checkbox(ABC):
    @abstractmethod
    def render(self) -> str:
        pass

# Concrete Products - Windows
class WindowsButton(Button):
    def render(self) -> str:
        return "Rendering Windows Button"

class WindowsCheckbox(Checkbox):
    def render(self) -> str:
        return "Rendering Windows Checkbox"

# Concrete Products - Mac
class MacButton(Button):
    def render(self) -> str:
        return "Rendering Mac Button"

class MacCheckbox(Checkbox):
    def render(self) -> str:
        return "Rendering Mac Checkbox"

# Abstract Factory
class GUIFactory(ABC):
    @abstractmethod
    def create_button(self) -> Button:
        pass

    @abstractmethod
    def create_checkbox(self) -> Checkbox:
        pass

# Concrete Factories
class WindowsFactory(GUIFactory):
    def create_button(self) -> Button:
        return WindowsButton()

    def create_checkbox(self) -> Checkbox:
        return WindowsCheckbox()

class MacFactory(GUIFactory):
    def create_button(self) -> Button:
        return MacButton()

    def create_checkbox(self) -> Checkbox:
        return MacCheckbox()

# Client code
def create_ui(factory: GUIFactory):
    button = factory.create_button()
    checkbox = factory.create_checkbox()
    print(button.render())
    print(checkbox.render())

# Usage
create_ui(WindowsFactory())  # Creates consistent Windows UI
create_ui(MacFactory())      # Creates consistent Mac UI
```

**Pros:**
- Ensures product family compatibility
- Isolates concrete classes
- Easy to exchange product families

**Cons:**
- Complex to implement
- Adding new products requires changing all factories

---

### 4. Builder Pattern

**Purpose:** Separate the construction of a complex object from its representation, allowing the same construction process to create different representations.

**When to Use:**
- Object has many optional parameters (avoids telescoping constructors)
- Object creation requires multiple steps
- Different representations of an object are needed

**Example:**
```python
class Pizza:
    def __init__(self):
        self.size: str = ""
        self.crust: str = ""
        self.toppings: list[str] = []
        self.cheese: bool = False

    def __str__(self) -> str:
        toppings_str = ", ".join(self.toppings) if self.toppings else "none"
        return (f"{self.size} pizza with {self.crust} crust, "
                f"toppings: {toppings_str}, cheese: {self.cheese}")

class PizzaBuilder:
    def __init__(self):
        self._pizza = Pizza()

    def set_size(self, size: str) -> 'PizzaBuilder':
        self._pizza.size = size
        return self

    def set_crust(self, crust: str) -> 'PizzaBuilder':
        self._pizza.crust = crust
        return self

    def add_topping(self, topping: str) -> 'PizzaBuilder':
        self._pizza.toppings.append(topping)
        return self

    def add_cheese(self) -> 'PizzaBuilder':
        self._pizza.cheese = True
        return self

    def build(self) -> Pizza:
        return self._pizza

# Usage - Method chaining
pizza = (PizzaBuilder()
         .set_size("Large")
         .set_crust("Thin")
         .add_topping("Pepperoni")
         .add_topping("Mushrooms")
         .add_cheese()
         .build())
```

**Pros:**
- Clear, readable object construction
- Avoids telescoping constructors
- Can create different representations

**Cons:**
- More verbose than simple constructors
- Requires creating separate builder class

---

### 5. Prototype Pattern

**Purpose:** Specify the kinds of objects to create using a prototypical instance, and create new objects by copying this prototype.

**When to Use:**
- Object creation is expensive
- You want to avoid subclasses of object creator
- Runtime object configuration is needed

**Example:**
```python
import copy
from abc import ABC, abstractmethod

class Prototype(ABC):
    @abstractmethod
    def clone(self) -> 'Prototype':
        pass

class Document(Prototype):
    def __init__(self, content: str, formatting: dict):
        self.content = content
        self.formatting = formatting
        self.metadata = {"created": "2024-01-01", "version": 1}

    def clone(self) -> 'Document':
        # Deep copy to avoid shared references
        return copy.deepcopy(self)

    def __str__(self) -> str:
        return f"Document(content='{self.content[:20]}...', formatting={self.formatting})"

# Usage
original = Document(
    content="This is a template document with lots of content...",
    formatting={"font": "Arial", "size": 12, "bold": False}
)

# Clone and modify
doc1 = original.clone()
doc1.content = "Modified content for document 1"
doc1.metadata["version"] = 2

doc2 = original.clone()
doc2.content = "Different content for document 2"
doc2.formatting["bold"] = True

# Original remains unchanged
print(original)  # Original template unchanged
print(doc1)      # Modified copy 1
print(doc2)      # Modified copy 2
```

**Pros:**
- Avoids expensive object creation
- Reduces subclassing
- Runtime object configuration

**Cons:**
- Deep copying can be complex
- Circular references can be problematic

---

## Category 2: Structural Patterns (7 Patterns)

Structural patterns deal with **object composition**, creating relationships between objects to form larger structures while keeping them flexible and efficient.

### 6. Adapter Pattern

**Purpose:** Convert the interface of a class into another interface clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces.

**When to Use:**
- You want to use an existing class with an incompatible interface
- You want to create a reusable class that cooperates with unrelated classes
- You need to use several existing subclasses, but it's impractical to adapt their interface by subclassing

**Real-World Analogy:** Power adapter for international travel - converts one plug type to another

**Example:**
```python
# Legacy system - incompatible interface
class OldPaymentSystem:
    def make_payment(self, account_num: str, amount: float):
        print(f"Old system: Processing ${amount} from account {account_num}")

# New interface expected by client
class PaymentProcessor(ABC):
    @abstractmethod
    def process(self, user_id: int, amount: float):
        pass

# Adapter - makes old system compatible with new interface
class PaymentAdapter(PaymentProcessor):
    def __init__(self, old_system: OldPaymentSystem):
        self.old_system = old_system

    def process(self, user_id: int, amount: float):
        # Convert new interface to old interface
        account_num = f"ACC{user_id:06d}"
        self.old_system.make_payment(account_num, amount)

# Client code expects PaymentProcessor interface
def charge_customer(processor: PaymentProcessor, user_id: int, amount: float):
    processor.process(user_id, amount)

# Usage
old_system = OldPaymentSystem()
adapter = PaymentAdapter(old_system)
charge_customer(adapter, 12345, 99.99)  # Works seamlessly
```

**Pros:**
- Follows Single Responsibility Principle
- Follows Open/Closed Principle
- Reuses existing functionality

**Cons:**
- Increases overall code complexity
- Sometimes simpler to just change the service class

---

### 7. Decorator Pattern

**Purpose:** Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.

**When to Use:**
- Add responsibilities to individual objects dynamically
- Withdraw responsibilities from objects
- Extension by subclassing is impractical

**Real-World Analogy:** Adding toppings to coffee - each topping "wraps" the base coffee

**Example:**
```python
from abc import ABC, abstractmethod

# Component interface
class Coffee(ABC):
    @abstractmethod
    def cost(self) -> float:
        pass

    @abstractmethod
    def description(self) -> str:
        pass

# Concrete component
class SimpleCoffee(Coffee):
    def cost(self) -> float:
        return 2.0

    def description(self) -> str:
        return "Simple coffee"

# Decorator base class
class CoffeeDecorator(Coffee):
    def __init__(self, coffee: Coffee):
        self._coffee = coffee

    def cost(self) -> float:
        return self._coffee.cost()

    def description(self) -> str:
        return self._coffee.description()

# Concrete decorators
class MilkDecorator(CoffeeDecorator):
    def cost(self) -> float:
        return self._coffee.cost() + 0.5

    def description(self) -> str:
        return self._coffee.description() + ", milk"

class SugarDecorator(CoffeeDecorator):
    def cost(self) -> float:
        return self._coffee.cost() + 0.2

    def description(self) -> str:
        return self._coffee.description() + ", sugar"

class WhipDecorator(CoffeeDecorator):
    def cost(self) -> float:
        return self._coffee.cost() + 0.7

    def description(self) -> str:
        return self._coffee.description() + ", whip"

# Usage - wrap decorators dynamically
coffee = SimpleCoffee()
print(f"{coffee.description()}: ${coffee.cost()}")  # Simple coffee: $2.0

coffee = MilkDecorator(coffee)
print(f"{coffee.description()}: ${coffee.cost()}")  # Simple coffee, milk: $2.5

coffee = SugarDecorator(coffee)
print(f"{coffee.description()}: ${coffee.cost()}")  # Simple coffee, milk, sugar: $2.7

coffee = WhipDecorator(coffee)
print(f"{coffee.description()}: ${coffee.cost()}")  # Simple coffee, milk, sugar, whip: $3.4
```

**Pros:**
- More flexible than inheritance
- Responsibilities can be added/removed at runtime
- Follows Single Responsibility Principle

**Cons:**
- Can result in many small objects
- Decorators can be hard to debug
- Order of decoration matters

---

### 8. Facade Pattern

**Purpose:** Provide a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use.

**When to Use:**
- You want to provide a simple interface to a complex subsystem
- There are many dependencies between clients and implementation classes
- You want to layer your subsystems

**Real-World Analogy:** Restaurant waiter - simplified interface to complex kitchen operations

**Example:**
```python
# Complex subsystem classes
class VideoFile:
    def __init__(self, filename: str):
        self.filename = filename

class AudioExtractor:
    def extract(self, video: VideoFile) -> str:
        return f"Extracting audio from {video.filename}"

class VideoDecoder:
    def decode(self, video: VideoFile, codec: str) -> str:
        return f"Decoding {video.filename} using {codec}"

class CompressionCodec:
    def compress(self, data: str) -> str:
        return f"Compressing {data}"

class AudioMixer:
    def mix(self, audio: str, video: str) -> str:
        return f"Mixing {audio} with {video}"

# Facade - simplifies complex subsystem
class VideoConverter:
    """Simple interface to complex video conversion system"""

    def convert(self, filename: str, format: str) -> str:
        video = VideoFile(filename)

        # Complex series of operations hidden behind simple interface
        audio_extractor = AudioExtractor()
        audio = audio_extractor.extract(video)

        decoder = VideoDecoder()
        video_data = decoder.decode(video, format)

        codec = CompressionCodec()
        compressed = codec.compress(video_data)

        mixer = AudioMixer()
        result = mixer.mix(audio, compressed)

        return f"Conversion complete: {result}"

# Client code - simple usage
converter = VideoConverter()
result = converter.convert("vacation.mp4", "avi")
print(result)  # Complex operations hidden behind simple call
```

**Pros:**
- Isolates clients from subsystem complexity
- Promotes loose coupling
- Follows Dependency Inversion Principle

**Cons:**
- Facade can become a god object
- May limit access to advanced features

---

### 9. Proxy Pattern

**Purpose:** Provide a surrogate or placeholder for another object to control access to it.

**When to Use:**
- Lazy initialization (virtual proxy)
- Access control (protection proxy)
- Remote object representation (remote proxy)
- Logging/caching (smart reference)

**Types:**
- **Virtual Proxy:** Delays expensive object creation
- **Protection Proxy:** Controls access based on permissions
- **Remote Proxy:** Represents remote object locally
- **Cache Proxy:** Caches results

**Example:**
```python
from abc import ABC, abstractmethod

class Image(ABC):
    @abstractmethod
    def display(self) -> str:
        pass

class RealImage(Image):
    def __init__(self, filename: str):
        self.filename = filename
        self._load_from_disk()

    def _load_from_disk(self):
        print(f"Loading image from disk: {self.filename}")
        # Expensive operation - simulate delay

    def display(self) -> str:
        return f"Displaying {self.filename}"

class ImageProxy(Image):
    """Virtual Proxy - delays loading until needed"""

    def __init__(self, filename: str):
        self.filename = filename
        self._real_image: RealImage | None = None

    def display(self) -> str:
        # Lazy initialization - only load when needed
        if self._real_image is None:
            self._real_image = RealImage(self.filename)
        return self._real_image.display()

# Usage
print("Creating proxy...")
image = ImageProxy("large_photo.jpg")  # No loading yet
print("Proxy created")

print("\nFirst display:")
print(image.display())  # Loading happens here

print("\nSecond display:")
print(image.display())  # No loading - already loaded
```

**Pros:**
- Controls access to objects
- Can add functionality without changing object
- Follows Open/Closed Principle

**Cons:**
- Increased complexity
- Response time may increase

---

### 10. Composite Pattern

**Purpose:** Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions uniformly.

**When to Use:**
- You want to represent part-whole hierarchies
- You want clients to treat individual and composite objects uniformly
- Tree structures are natural for your domain

**Real-World Analogy:** File system - folders contain files and other folders

**Example:**
```python
from abc import ABC, abstractmethod
from typing import List

class FileSystemComponent(ABC):
    @abstractmethod
    def get_size(self) -> int:
        pass

    @abstractmethod
    def display(self, indent: int = 0) -> str:
        pass

class File(FileSystemComponent):
    """Leaf - cannot contain other components"""

    def __init__(self, name: str, size: int):
        self.name = name
        self.size = size

    def get_size(self) -> int:
        return self.size

    def display(self, indent: int = 0) -> str:
        return " " * indent + f"File: {self.name} ({self.size} bytes)"

class Folder(FileSystemComponent):
    """Composite - can contain files and other folders"""

    def __init__(self, name: str):
        self.name = name
        self.children: List[FileSystemComponent] = []

    def add(self, component: FileSystemComponent):
        self.children.append(component)

    def remove(self, component: FileSystemComponent):
        self.children.remove(component)

    def get_size(self) -> int:
        # Recursively sum sizes of all children
        return sum(child.get_size() for child in self.children)

    def display(self, indent: int = 0) -> str:
        result = " " * indent + f"Folder: {self.name}\n"
        for child in self.children:
            result += child.display(indent + 2) + "\n"
        return result.rstrip()

# Usage - build tree structure
root = Folder("root")
documents = Folder("documents")
photos = Folder("photos")

documents.add(File("resume.pdf", 500))
documents.add(File("letter.txt", 100))

photos.add(File("vacation.jpg", 2000))
photos.add(File("family.jpg", 1500))

root.add(documents)
root.add(photos)
root.add(File("readme.txt", 50))

# Treat individual files and folders uniformly
print(root.display())
print(f"\nTotal size: {root.get_size()} bytes")
```

**Pros:**
- Simplifies client code
- Easy to add new component types
- Follows Open/Closed Principle

**Cons:**
- Can make design overly general
- Hard to restrict component types

---

### 11. Bridge Pattern

**Purpose:** Decouple an abstraction from its implementation so that the two can vary independently.

**When to Use:**
- You want to avoid permanent binding between abstraction and implementation
- Both abstraction and implementation should be extensible by subclassing
- Changes in implementation shouldn't affect clients

**Real-World Analogy:** Remote control (abstraction) can work with any TV (implementation)

**Example:**
```python
from abc import ABC, abstractmethod

# Implementation interface
class Device(ABC):
    @abstractmethod
    def turn_on(self):
        pass

    @abstractmethod
    def turn_off(self):
        pass

    @abstractmethod
    def set_volume(self, volume: int):
        pass

# Concrete implementations
class TV(Device):
    def __init__(self):
        self.volume = 10

    def turn_on(self):
        print("TV: Turning on")

    def turn_off(self):
        print("TV: Turning off")

    def set_volume(self, volume: int):
        self.volume = volume
        print(f"TV: Volume set to {volume}")

class Radio(Device):
    def __init__(self):
        self.volume = 5

    def turn_on(self):
        print("Radio: Turning on")

    def turn_off(self):
        print("Radio: Turning off")

    def set_volume(self, volume: int):
        self.volume = volume
        print(f"Radio: Volume set to {volume}")

# Abstraction
class RemoteControl:
    def __init__(self, device: Device):
        self.device = device

    def toggle_power(self):
        # Can call either implementation
        self.device.turn_on()

    def volume_up(self):
        self.device.set_volume(15)

# Extended abstraction
class AdvancedRemote(RemoteControl):
    def mute(self):
        self.device.set_volume(0)

# Usage - abstraction and implementation vary independently
tv = TV()
tv_remote = RemoteControl(tv)
tv_remote.toggle_power()  # TV: Turning on
tv_remote.volume_up()     # TV: Volume set to 15

radio = Radio()
radio_remote = AdvancedRemote(radio)
radio_remote.toggle_power()  # Radio: Turning on
radio_remote.mute()          # Radio: Volume set to 0
```

**Pros:**
- Decouples abstraction and implementation
- Follows Open/Closed Principle
- Improves extensibility

**Cons:**
- Increased complexity
- May be overkill for simple scenarios

---

### 12. Flyweight Pattern

**Purpose:** Use sharing to support large numbers of fine-grained objects efficiently.

**When to Use:**
- Application uses a large number of objects
- Storage costs are high due to object quantity
- Most object state can be made extrinsic
- Many objects can be replaced by few shared objects

**Real-World Analogy:** Text editor - characters share glyph data, only position varies

**Example:**
```python
class TreeType:
    """Flyweight - shared intrinsic state"""

    def __init__(self, name: str, color: str, texture: str):
        self.name = name      # Shared state
        self.color = color    # Shared state
        self.texture = texture  # Shared state

    def render(self, x: int, y: int):
        """Render tree at specific position (extrinsic state)"""
        print(f"Drawing {self.color} {self.name} tree at ({x}, {y})")

class TreeFactory:
    """Manages flyweight pool"""

    _tree_types: dict[str, TreeType] = {}

    @classmethod
    def get_tree_type(cls, name: str, color: str, texture: str) -> TreeType:
        key = f"{name}_{color}_{texture}"
        if key not in cls._tree_types:
            cls._tree_types[key] = TreeType(name, color, texture)
            print(f"Creating new TreeType: {key}")
        return cls._tree_types[key]

    @classmethod
    def get_total_types(cls) -> int:
        return len(cls._tree_types)

class Tree:
    """Context - stores extrinsic state"""

    def __init__(self, x: int, y: int, tree_type: TreeType):
        self.x = x  # Unique state
        self.y = y  # Unique state
        self.type = tree_type  # Shared state (flyweight)

    def render(self):
        self.type.render(self.x, self.y)

# Usage - create 1000 trees with only a few shared types
forest = []
for i in range(1000):
    if i % 3 == 0:
        tree_type = TreeFactory.get_tree_type("Oak", "green", "rough")
    elif i % 3 == 1:
        tree_type = TreeFactory.get_tree_type("Pine", "dark green", "smooth")
    else:
        tree_type = TreeFactory.get_tree_type("Birch", "white", "papery")

    tree = Tree(i * 10, i * 10, tree_type)
    forest.append(tree)

# Render first 5 trees
for tree in forest[:5]:
    tree.render()

print(f"\nTotal trees: {len(forest)}")
print(f"Total tree types (flyweights): {TreeFactory.get_total_types()}")
# Only 3 TreeType objects created for 1000 trees!
```

**Pros:**
- Reduces memory usage dramatically
- Improves performance with large object counts

**Cons:**
- Increased code complexity
- Runtime costs for computing extrinsic state

---

## Category 3: Behavioral Patterns (11 Patterns)

Behavioral patterns focus on **communication between objects**, defining how objects interact and distribute responsibility.

### 13. Observer Pattern

**Purpose:** Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified automatically.

**When to Use:**
- Change to one object requires changing others (unknown number)
- Object should notify others without knowing who they are
- Event handling systems

**Real-World Analogy:** Newsletter subscription - publisher notifies all subscribers

**Example:**
```python
from abc import ABC, abstractmethod
from typing import List

class Observer(ABC):
    @abstractmethod
    def update(self, temperature: float, humidity: float, pressure: float):
        pass

class Subject(ABC):
    @abstractmethod
    def attach(self, observer: Observer):
        pass

    @abstractmethod
    def detach(self, observer: Observer):
        pass

    @abstractmethod
    def notify(self):
        pass

class WeatherStation(Subject):
    def __init__(self):
        self._observers: List[Observer] = []
        self._temperature: float = 0
        self._humidity: float = 0
        self._pressure: float = 0

    def attach(self, observer: Observer):
        self._observers.append(observer)

    def detach(self, observer: Observer):
        self._observers.remove(observer)

    def notify(self):
        for observer in self._observers:
            observer.update(self._temperature, self._humidity, self._pressure)

    def set_measurements(self, temperature: float, humidity: float, pressure: float):
        self._temperature = temperature
        self._humidity = humidity
        self._pressure = pressure
        self.notify()  # Auto-notify observers

class PhoneDisplay(Observer):
    def update(self, temperature: float, humidity: float, pressure: float):
        print(f"Phone: {temperature}°F, {humidity}% humidity")

class WebDisplay(Observer):
    def update(self, temperature: float, humidity: float, pressure: float):
        print(f"Web: Temperature: {temperature}°F, Pressure: {pressure} inHg")

# Usage
station = WeatherStation()

phone = PhoneDisplay()
web = WebDisplay()

station.attach(phone)
station.attach(web)

station.set_measurements(75.0, 65.0, 30.4)
# Output:
# Phone: 75.0°F, 65% humidity
# Web: Temperature: 75.0°F, Pressure: 30.4 inHg

station.detach(phone)
station.set_measurements(80.0, 70.0, 29.8)
# Output:
# Web: Temperature: 80.0°F, Pressure: 29.8 inHg
```

**Pros:**
- Loose coupling between subject and observers
- Follows Open/Closed Principle
- Dynamic relationships at runtime

**Cons:**
- Observers notified in random order
- Can cause memory leaks if not detached

---

### 14. Strategy Pattern

**Purpose:** Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.

**When to Use:**
- Many related classes differ only in behavior
- You need different variants of an algorithm
- Algorithm uses data that clients shouldn't know about

**Real-World Analogy:** Navigation app - different routing strategies (fastest, shortest, scenic)

**Example:**
```python
from abc import ABC, abstractmethod

class PaymentStrategy(ABC):
    @abstractmethod
    def pay(self, amount: float) -> str:
        pass

class CreditCardStrategy(PaymentStrategy):
    def __init__(self, card_number: str, cvv: str):
        self.card_number = card_number
        self.cvv = cvv

    def pay(self, amount: float) -> str:
        return f"Paid ${amount} using Credit Card ending in {self.card_number[-4:]}"

class PayPalStrategy(PaymentStrategy):
    def __init__(self, email: str):
        self.email = email

    def pay(self, amount: float) -> str:
        return f"Paid ${amount} using PayPal account {self.email}"

class CryptoStrategy(PaymentStrategy):
    def __init__(self, wallet_address: str):
        self.wallet_address = wallet_address

    def pay(self, amount: float) -> str:
        return f"Paid ${amount} using Crypto wallet {self.wallet_address[:10]}..."

class ShoppingCart:
    def __init__(self):
        self.amount: float = 0
        self._payment_strategy: PaymentStrategy | None = None

    def set_payment_strategy(self, strategy: PaymentStrategy):
        self._payment_strategy = strategy

    def checkout(self) -> str:
        if self._payment_strategy is None:
            return "Please select a payment method"
        return self._payment_strategy.pay(self.amount)

# Usage - swap strategies at runtime
cart = ShoppingCart()
cart.amount = 150.00

# Use credit card
cart.set_payment_strategy(CreditCardStrategy("1234567890123456", "123"))
print(cart.checkout())  # Paid $150.0 using Credit Card ending in 3456

# Switch to PayPal
cart.set_payment_strategy(PayPalStrategy("user@example.com"))
print(cart.checkout())  # Paid $150.0 using PayPal account user@example.com

# Switch to Crypto
cart.set_payment_strategy(CryptoStrategy("0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb"))
print(cart.checkout())  # Paid $150.0 using Crypto wallet 0x742d35Cc...
```

**Pros:**
- Swap algorithms at runtime
- Isolates algorithm details from client
- Follows Open/Closed Principle

**Cons:**
- Clients must be aware of strategies
- Increased number of objects

---

### 15. Command Pattern

**Purpose:** Encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations.

**When to Use:**
- Parameterize objects with operations
- Queue operations, schedule execution
- Support undo/redo
- Logging changes

**Real-World Analogy:** Restaurant order - waiter takes order (command) to kitchen

**Example:**
```python
from abc import ABC, abstractmethod

class Command(ABC):
    @abstractmethod
    def execute(self):
        pass

    @abstractmethod
    def undo(self):
        pass

class Light:
    def __init__(self, location: str):
        self.location = location
        self.is_on = False

    def turn_on(self):
        self.is_on = True
        print(f"{self.location} light is ON")

    def turn_off(self):
        self.is_on = False
        print(f"{self.location} light is OFF")

class LightOnCommand(Command):
    def __init__(self, light: Light):
        self.light = light

    def execute(self):
        self.light.turn_on()

    def undo(self):
        self.light.turn_off()

class LightOffCommand(Command):
    def __init__(self, light: Light):
        self.light = light

    def execute(self):
        self.light.turn_off()

    def undo(self):
        self.light.turn_on()

class RemoteControl:
    def __init__(self):
        self.history: list[Command] = []

    def execute_command(self, command: Command):
        command.execute()
        self.history.append(command)

    def undo_last(self):
        if self.history:
            command = self.history.pop()
            command.undo()

# Usage
living_room = Light("Living Room")
bedroom = Light("Bedroom")

remote = RemoteControl()

# Execute commands
remote.execute_command(LightOnCommand(living_room))   # Living Room light is ON
remote.execute_command(LightOnCommand(bedroom))       # Bedroom light is ON
remote.execute_command(LightOffCommand(living_room))  # Living Room light is OFF

# Undo last command
remote.undo_last()  # Living Room light is ON (undo the turn off)
remote.undo_last()  # Bedroom light is OFF (undo the turn on)
```

**Pros:**
- Decouples sender and receiver
- Easy to add new commands
- Supports undo/redo
- Supports command queuing

**Cons:**
- Increases number of classes
- Can become complex

---

### 16-23. Other Behavioral Patterns (Brief Overview)

Due to length constraints, here's a brief overview of the remaining behavioral patterns:

**16. State Pattern**
- **Purpose:** Allow object to alter behavior when internal state changes
- **Use Case:** TCP connection states, vending machine states
- **Key Benefit:** Eliminates large conditional statements

**17. Iterator Pattern**
- **Purpose:** Access elements of collection without exposing representation
- **Use Case:** Iterating through lists, trees, graphs
- **Key Benefit:** Multiple traversal algorithms

**18. Template Method Pattern**
- **Purpose:** Define skeleton of algorithm, let subclasses override steps
- **Use Case:** Data parsing frameworks, game AI
- **Key Benefit:** Code reuse through inheritance

**19. Chain of Responsibility Pattern**
- **Purpose:** Pass request along chain of handlers
- **Use Case:** Event handling, middleware, logging
- **Key Benefit:** Decouples sender and receiver

**20. Mediator Pattern**
- **Purpose:** Define object that encapsulates how objects interact
- **Use Case:** Chat rooms, air traffic control
- **Key Benefit:** Reduces coupling between components

**21. Memento Pattern**
- **Purpose:** Capture and restore object state
- **Use Case:** Undo mechanisms, save points
- **Key Benefit:** Preserves encapsulation

**22. Visitor Pattern**
- **Purpose:** Separate algorithm from object structure
- **Use Case:** Compilers, report generation
- **Key Benefit:** Add operations without changing classes

**23. Interpreter Pattern**
- **Purpose:** Define grammar and interpreter for language
- **Use Case:** SQL parsing, regular expressions
- **Key Benefit:** Easy to change and extend grammar

---

## Design Patterns Comparison

### Creational Patterns Quick Reference

| Pattern | Problem Solved | When to Use |
|---------|---------------|-------------|
| **Singleton** | Need exactly one instance | Database connections, config managers |
| **Factory Method** | Defer instantiation to subclasses | Unknown product types at compile time |
| **Abstract Factory** | Create families of related objects | Multiple product families, platform-specific UI |
| **Builder** | Construct complex objects step-by-step | Many optional parameters, telescoping constructors |
| **Prototype** | Copy existing objects | Expensive object creation, runtime configuration |

### Structural Patterns Quick Reference

| Pattern | Problem Solved | When to Use |
|---------|---------------|-------------|
| **Adapter** | Make incompatible interfaces work together | Legacy system integration, third-party libraries |
| **Decorator** | Add responsibilities dynamically | Extend functionality without subclassing |
| **Facade** | Simplify complex subsystem | Hide complexity, provide unified interface |
| **Proxy** | Control access to object | Lazy loading, access control, remote objects |
| **Composite** | Treat individual and composite objects uniformly | Tree structures, hierarchies |
| **Bridge** | Separate abstraction from implementation | Both abstraction and implementation vary |
| **Flyweight** | Share objects to reduce memory | Large numbers of similar objects |

### Behavioral Patterns Quick Reference

| Pattern | Problem Solved | When to Use |
|---------|---------------|-------------|
| **Observer** | Notify multiple objects of state changes | Event systems, MVC, pub-sub |
| **Strategy** | Swap algorithms at runtime | Multiple algorithm variants, conditional logic |
| **Command** | Encapsulate requests as objects | Undo/redo, queuing, logging |
| **State** | Change behavior based on state | State machines, workflow |
| **Iterator** | Access collection elements sequentially | Hide collection implementation |
| **Template Method** | Define algorithm skeleton | Algorithm variants with common structure |
| **Chain of Responsibility** | Pass request through handler chain | Event bubbling, middleware |
| **Mediator** | Reduce coupling between components | Complex object interactions |
| **Memento** | Save and restore object state | Undo, snapshots |
| **Visitor** | Add operations without changing classes | Operations on complex structures |
| **Interpreter** | Interpret language/grammar | DSLs, expression evaluation |

---

## How to Learn Design Patterns

### 1. Start with the Fundamentals
- Understand SOLID principles first
- Master object-oriented concepts (inheritance, polymorphism, composition)
- Learn UML basics for visualizing patterns

### 2. Learn Patterns by Category
- **Begin with Creational:** Singleton, Factory Method, Builder
- **Move to Structural:** Decorator, Adapter, Facade
- **Then Behavioral:** Observer, Strategy, Command

### 3. Practice with Real Problems
- Implement each pattern from scratch
- Solve OOD interview problems using patterns
- Refactor existing code to use patterns

### 4. Recognize When NOT to Use Patterns
- Don't over-engineer simple solutions
- Patterns add complexity - use when benefit outweighs cost
- YAGNI principle: "You Aren't Gonna Need It"

### 5. Study Real-World Examples
- Examine frameworks (Django, Flask, React)
- Read open-source code
- Identify patterns in production systems

---

## Interview Strategy for Design Patterns

### What Interviewers Assess

When asked about design patterns in interviews, interviewers evaluate:

- **Understanding:** Do you know what the pattern does?
- **Application:** Can you identify when to use it?
- **Implementation:** Can you code it?
- **Trade-offs:** Do you know pros/cons?
- **Alternatives:** Can you compare with other solutions?

### Common Interview Questions

1. **"Explain the Singleton pattern and implement it in Python."**
   - Show thread-safe implementation
   - Discuss pros/cons
   - Mention alternatives (dependency injection)

2. **"Design a notification system." (Observer pattern)**
   - Identify publisher-subscriber relationship
   - Show loose coupling
   - Handle edge cases

3. **"How would you add features to objects at runtime?" (Decorator)**
   - Compare with inheritance
   - Show method chaining
   - Discuss order of decoration

4. **"Design a payment processing system." (Strategy pattern)**
   - Multiple payment methods
   - Runtime selection
   - Easy to extend

### Interview Tips

- **Always start with clarifying questions**
- **Explain your thinking out loud**
- **Draw diagrams** (UML, class relationships)
- **Code iteratively** (start simple, add complexity)
- **Discuss trade-offs** (time/space, flexibility/simplicity)
- **Mention SOLID principles** applied
- **Provide real-world examples**
- **Know when NOT to use a pattern**

---

## Anti-Patterns to Avoid

Understanding what NOT to do is equally important:

### 1. Pattern Overuse
- Don't use patterns for the sake of using patterns
- Keep it simple when possible
- "Design patterns are solutions to problems, not solutions looking for problems"

### 2. God Object
- One class that knows/does too much
- Violates Single Responsibility Principle
- Hard to test and maintain

### 3. Premature Optimization
- Don't optimize before profiling
- Don't add complexity for hypothetical future needs
- Follow YAGNI

### 4. Cargo Cult Programming
- Don't copy patterns without understanding them
- Understand the "why" not just the "how"
- Adapt patterns to your specific context

---

## Resources for Further Learning

### Books
1. **"Design Patterns: Elements of Reusable Object-Oriented Software"** (Gang of Four)
   - The original, authoritative source

2. **"Head First Design Patterns"** by Freeman & Robson
   - Beginner-friendly, visual approach

3. **"Refactoring to Patterns"** by Joshua Kerievsky
   - Shows when and how to apply patterns

### Online Resources
- **Refactoring.Guru** - Visual explanations with code examples
- **SourceMaking.com** - Patterns, anti-patterns, refactoring
- **Python Design Patterns** - Python-specific implementations

### Practice Platforms
- **LeetCode OOD Section** - Design problems
- **System Design Primer** - Real-world applications
- **Educative.io** - Interactive courses

---

## Summary

Design patterns are:

- **Proven solutions** to recurring design problems
- **Communication tools** for developers
- **Essential knowledge** for technical interviews
- **Guidelines, not rules** - adapt to your context

### Key Takeaways

1. **23 GoF patterns** organized in 3 categories (Creational, Structural, Behavioral)
2. **Learn gradually:** Start with common patterns (Singleton, Factory, Observer, Strategy)
3. **Understand trade-offs:** Every pattern has pros and cons
4. **Practice coding:** Implement patterns from scratch
5. **Recognize contexts:** Know when to apply (and when not to)
6. **Interview success:** Combine patterns with SOLID principles
7. **Real-world focus:** Study how frameworks use patterns

### Next Steps

1. Read the individual pattern files in `01-design-patterns/`
2. Practice OOD problems in `02-beginner-problems/`
3. Review SOLID principles in `solid-principles.md`
4. Study UML diagrams in `uml-diagrams.md`
5. Learn interview approach in `interview-approach.md`

Remember: **Understanding the problem a pattern solves is more important than memorizing its implementation.**

Good luck with your learning journey!
