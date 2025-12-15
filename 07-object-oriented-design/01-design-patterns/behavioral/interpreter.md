# Interpreter Pattern

## Intent
Given a language, define a representation for its grammar along with an interpreter that uses the representation to interpret sentences in the language.

## Problem
Need to interpret or evaluate language sentences or expressions.

## Solution
Create classes for each grammar rule; build expression tree and interpret.

## Python Implementation

```python
from abc import ABC, abstractmethod

class Expression(ABC):
    @abstractmethod
    def interpret(self, context: dict) -> int:
        pass

class Number(Expression):
    def __init__(self, value: int):
        self.value = value

    def interpret(self, context: dict) -> int:
        return self.value

class Variable(Expression):
    def __init__(self, name: str):
        self.name = name

    def interpret(self, context: dict) -> int:
        return context.get(self.name, 0)

class Add(Expression):
    def __init__(self, left: Expression, right: Expression):
        self.left = left
        self.right = right

    def interpret(self, context: dict) -> int:
        return self.left.interpret(context) + self.right.interpret(context)

class Subtract(Expression):
    def __init__(self, left: Expression, right: Expression):
        self.left = left
        self.right = right

    def interpret(self, context: dict) -> int:
        return self.left.interpret(context) - self.right.interpret(context)

class Multiply(Expression):
    def __init__(self, left: Expression, right: Expression):
        self.left = left
        self.right = right

    def interpret(self, context: dict) -> int:
        return self.left.interpret(context) * self.right.interpret(context)

# Usage
# Expression: (x + y) * z where x=5, y=3, z=2
context = {"x": 5, "y": 3, "z": 2}

# Build expression tree: (x + y) * z
x = Variable("x")
y = Variable("y")
z = Variable("z")
add_xy = Add(x, y)
expression = Multiply(add_xy, z)

result = expression.interpret(context)
print(f"(x + y) * z = {result}")  # (5 + 3) * 2 = 16

# Another expression: 10 - (4 + 2)
expression2 = Subtract(Number(10), Add(Number(4), Number(2)))
result2 = expression2.interpret({})
print(f"10 - (4 + 2) = {result2}")  # 10 - 6 = 4
```

## When to Use
- ✅ Simple grammar to interpret
- ✅ Efficiency not critical
- ✅ DSL (Domain Specific Language)

## Summary
Interpreter defines grammar and evaluates expressions. Useful for DSLs, expression evaluators, but can become complex for large grammars.
