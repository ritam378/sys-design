# Decorator Pattern

## Intent
Attach additional responsibilities to an object dynamically. Decorators provide flexible alternative to subclassing for extending functionality.

## Problem
Need to add functionality to objects without affecting other objects of same class.

## Solution
Wrap objects with decorator classes that add new behavior.

## Python Implementation

```python
from abc import ABC, abstractmethod

class Coffee(ABC):
    @abstractmethod
    def cost(self) -> float:
        pass

    @abstractmethod
    def description(self) -> str:
        pass

class SimpleCoffee(Coffee):
    def cost(self) -> float:
        return 2.0

    def description(self) -> str:
        return "Simple coffee"

# Decorators
class CoffeeDecorator(Coffee):
    def __init__(self, coffee: Coffee):
        self._coffee = coffee

    def cost(self) -> float:
        return self._coffee.cost()

    def description(self) -> str:
        return self._coffee.description()

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

# Usage - wrap decorators
coffee = SimpleCoffee()
coffee = MilkDecorator(coffee)
coffee = SugarDecorator(coffee)
print(f"{coffee.description()}: ${coffee.cost()}")  # Simple coffee, milk, sugar: $2.7
```

## When to Use
- ✅ Add responsibilities dynamically
- ✅ Extend functionality without subclassing
- ✅ Combine behaviors flexibly

## Summary
Decorator wraps objects to add new functionality dynamically, following Open/Closed Principle.
