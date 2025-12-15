# Visitor Pattern

## Intent
Represent an operation to be performed on elements of an object structure. Visitor lets you define a new operation without changing the classes of the elements on which it operates.

## Problem
Need to perform operations on object structure without modifying classes.

## Solution
Move operations into visitor classes; elements accept visitors.

## Python Implementation

```python
from abc import ABC, abstractmethod

class ShapeVisitor(ABC):
    @abstractmethod
    def visit_circle(self, circle: 'Circle'):
        pass

    @abstractmethod
    def visit_rectangle(self, rectangle: 'Rectangle'):
        pass

class Shape(ABC):
    @abstractmethod
    def accept(self, visitor: ShapeVisitor):
        pass

class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius

    def accept(self, visitor: ShapeVisitor):
        visitor.visit_circle(self)

class Rectangle(Shape):
    def __init__(self, width: float, height: float):
        self.width = width
        self.height = height

    def accept(self, visitor: ShapeVisitor):
        visitor.visit_rectangle(self)

# Concrete visitors
class AreaCalculator(ShapeVisitor):
    def visit_circle(self, circle: Circle):
        area = 3.14159 * circle.radius ** 2
        print(f"Circle area: {area:.2f}")

    def visit_rectangle(self, rectangle: Rectangle):
        area = rectangle.width * rectangle.height
        print(f"Rectangle area: {area:.2f}")

class PerimeterCalculator(ShapeVisitor):
    def visit_circle(self, circle: Circle):
        perimeter = 2 * 3.14159 * circle.radius
        print(f"Circle perimeter: {perimeter:.2f}")

    def visit_rectangle(self, rectangle: Rectangle):
        perimeter = 2 * (rectangle.width + rectangle.height)
        print(f"Rectangle perimeter: {perimeter:.2f}")

# Usage
shapes = [Circle(5), Rectangle(4, 6), Circle(3)]

area_calc = AreaCalculator()
perimeter_calc = PerimeterCalculator()

print("Calculating areas:")
for shape in shapes:
    shape.accept(area_calc)

print("\nCalculating perimeters:")
for shape in shapes:
    shape.accept(perimeter_calc)
```

## When to Use
- ✅ Add operations without changing classes
- ✅ Operations need different element types
- ✅ Object structure stable, operations change

## Summary
Visitor separates algorithm from object structure, enabling new operations without modifying classes. Common in compilers, reporting systems.
