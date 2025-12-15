# Strategy Pattern

## Intent
Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients.

## Problem
Need different variants of an algorithm and want to switch between them at runtime.

## Solution
Encapsulate algorithms in separate classes with common interface.

## Python Implementation

```python
from abc import ABC, abstractmethod

class PaymentStrategy(ABC):
    @abstractmethod
    def pay(self, amount: float) -> str:
        pass

class CreditCardStrategy(PaymentStrategy):
    def __init__(self, card_number: str):
        self.card_number = card_number

    def pay(self, amount: float) -> str:
        return f"Paid ${amount} using Credit Card ending in {self.card_number[-4:]}"

class PayPalStrategy(PaymentStrategy):
    def __init__(self, email: str):
        self.email = email

    def pay(self, amount: float) -> str:
        return f"Paid ${amount} using PayPal ({self.email})"

class CryptoStrategy(PaymentStrategy):
    def __init__(self, wallet: str):
        self.wallet = wallet

    def pay(self, amount: float) -> str:
        return f"Paid ${amount} using Crypto wallet {self.wallet[:10]}..."

class ShoppingCart:
    def __init__(self):
        self.amount = 0.0
        self._payment_strategy = None

    def set_payment_strategy(self, strategy: PaymentStrategy):
        self._payment_strategy = strategy

    def checkout(self) -> str:
        if not self._payment_strategy:
            return "Please select payment method"
        return self._payment_strategy.pay(self.amount)

# Usage - swap strategies at runtime
cart = ShoppingCart()
cart.amount = 150.00

cart.set_payment_strategy(CreditCardStrategy("1234567890123456"))
print(cart.checkout())

cart.set_payment_strategy(PayPalStrategy("user@example.com"))
print(cart.checkout())
```

## When to Use
- ✅ Multiple algorithm variants
- ✅ Eliminate conditional logic
- ✅ Switch algorithms at runtime

## Summary
Strategy enables selecting algorithm at runtime, promoting Open/Closed Principle and eliminating conditionals.
