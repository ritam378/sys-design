# Prototype Pattern

## Intent
Specify the kinds of objects to create using a prototypical instance, and create new objects by copying this prototype.

## Problem
- Object creation is expensive (database loading, complex initialization)
- Want to avoid subclassing
- Need runtime object configuration

## Solution
Clone existing objects instead of creating from scratch. Use shallow or deep copying based on requirements.

## Python Implementation

```python
import copy
from abc import ABC, abstractmethod

class Prototype(ABC):
    @abstractmethod
    def clone(self) -> 'Prototype':
        pass

# Example 1: Document Cloning
class Document(Prototype):
    def __init__(self, title: str, content: str, metadata: dict):
        self.title = title
        self.content = content
        self.metadata = metadata  # Mutable object

    def clone(self) -> 'Document':
        # Deep copy to avoid shared references
        return copy.deepcopy(self)

    def __str__(self) -> str:
        return f"Document('{self.title}', metadata={self.metadata})"

# Usage
original = Document("Report", "Content here", {"author": "Alice", "version": 1})
clone1 = original.clone()
clone1.title = "Report Copy 1"
clone1.metadata["version"] = 2

clone2 = original.clone()
clone2.title = "Report Copy 2"
clone2.metadata["version"] = 3

print(original)  # version: 1 (unchanged)
print(clone1)    # version: 2
print(clone2)    # version: 3
```

### Example 2: Shape Cloning

```python
class Shape(ABC):
    def __init__(self, color: str):
        self.color = color

    @abstractmethod
    def clone(self) -> 'Shape':
        pass

    @abstractmethod
    def draw(self) -> str:
        pass

class Circle(Shape):
    def __init__(self, color: str, radius: float):
        super().__init__(color)
        self.radius = radius

    def clone(self) -> 'Circle':
        return copy.copy(self)  # Shallow copy sufficient

    def draw(self) -> str:
        return f"Drawing {self.color} circle (radius={self.radius})"

class Rectangle(Shape):
    def __init__(self, color: str, width: float, height: float):
        super().__init__(color)
        self.width = width
        self.height = height

    def clone(self) -> 'Rectangle':
        return copy.copy(self)

    def draw(self) -> str:
        return f"Drawing {self.color} rectangle ({self.width}x{self.height})"

# Usage - clone and modify
original_circle = Circle("red", 5.0)
cloned_circle = original_circle.clone()
cloned_circle.color = "blue"
cloned_circle.radius = 10.0

print(original_circle.draw())  # red circle, radius=5.0
print(cloned_circle.draw())    # blue circle, radius=10.0
```

### Example 3: Configuration Templates

```python
class ServerConfig(Prototype):
    def __init__(self):
        self.host = "localhost"
        self.port = 8080
        self.ssl_enabled = False
        self.workers = 4
        self.settings = {}

    def clone(self) -> 'ServerConfig':
        return copy.deepcopy(self)

    def __str__(self) -> str:
        return f"Server {self.host}:{self.port}, SSL={self.ssl_enabled}, workers={self.workers}"

# Create templates
dev_template = ServerConfig()
dev_template.host = "dev.example.com"
dev_template.port = 8000
dev_template.ssl_enabled = False

prod_template = ServerConfig()
prod_template.host = "prod.example.com"
prod_template.port = 443
prod_template.ssl_enabled = True
prod_template.workers = 8

# Clone and customize
server1 = dev_template.clone()
server1.port = 8001

server2 = prod_template.clone()
server2.host = "prod2.example.com"

print(server1)
print(server2)
```

### Example 4: Prototype Registry

```python
class PrototypeRegistry:
    """Registry to manage prototypes"""

    def __init__(self):
        self._prototypes = {}

    def register(self, name: str, prototype: Prototype):
        self._prototypes[name] = prototype

    def unregister(self, name: str):
        del self._prototypes[name]

    def clone(self, name: str) -> Prototype:
        prototype = self._prototypes.get(name)
        if prototype:
            return prototype.clone()
        raise ValueError(f"Prototype '{name}' not found")

# Usage
registry = PrototypeRegistry()

# Register templates
registry.register("small_circle", Circle("red", 5.0))
registry.register("large_circle", Circle("blue", 20.0))
registry.register("small_rect", Rectangle("green", 10.0, 5.0))

# Clone from registry
shape1 = registry.clone("small_circle")
shape2 = registry.clone("large_circle")

print(shape1.draw())
print(shape2.draw())
```

## Shallow vs Deep Copy

```python
import copy

class Address:
    def __init__(self, street: str, city: str):
        self.street = street
        self.city = city

class Person:
    def __init__(self, name: str, address: Address):
        self.name = name
        self.address = address  # Mutable reference

# Shallow copy - shares address reference
person1 = Person("Alice", Address("123 Main St", "NYC"))
person2 = copy.copy(person1)
person2.name = "Bob"
person2.address.city = "LA"  # Affects person1 too!

print(f"{person1.name}: {person1.address.city}")  # Alice: LA (changed!)
print(f"{person2.name}: {person2.address.city}")  # Bob: LA

# Deep copy - independent address
person3 = Person("Charlie", Address("456 Oak Ave", "SF"))
person4 = copy.deepcopy(person3)
person4.name = "Diana"
person4.address.city = "Seattle"  # Doesn't affect person3

print(f"{person3.name}: {person3.address.city}")  # Charlie: SF (unchanged)
print(f"{person4.name}: {person4.address.city}")  # Diana: Seattle
```

## When to Use
- ✅ Object creation is expensive
- ✅ Want to avoid subclass explosion
- ✅ Objects configured at runtime
- ✅ Hide complexity of creating objects

## Advantages
- Avoids expensive initialization
- Reduces subclassing
- Runtime configuration
- Hides construction complexity

## Disadvantages
- Deep copying can be complex (circular references)
- Cloning objects with private fields can be tricky
- Must implement clone for all types

## Real-World Examples

### Game Development
```python
class Enemy(Prototype):
    def __init__(self, name: str, health: int, damage: int):
        self.name = name
        self.health = health
        self.damage = damage

    def clone(self) -> 'Enemy':
        return copy.copy(self)

# Create prototype enemies
goblin_template = Enemy("Goblin", 50, 10)
orc_template = Enemy("Orc", 100, 20)

# Spawn many enemies from templates
enemies = []
for i in range(10):
    enemies.append(goblin_template.clone())
for i in range(5):
    enemies.append(orc_template.clone())
```

## Interview Tips

**Q: When would you use Prototype over Factory?**
A: When object creation is expensive or complex, and you want to clone pre-configured instances.

**Q: What's the difference between shallow and deep copy?**
A: Shallow copies share references to mutable objects; deep copies create independent copies of all nested objects.

**Q: How do you handle circular references in cloning?**
A: Use `copy.deepcopy()` which tracks already-copied objects, or implement custom clone logic.

## Summary
Prototype pattern is ideal when object creation is expensive. Use `copy.copy()` for shallow copies and `copy.deepcopy()` for deep copies. Common in games, document editors, and configuration systems.
