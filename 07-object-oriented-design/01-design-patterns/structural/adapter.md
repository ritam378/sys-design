# Adapter Pattern

## Intent
Convert the interface of a class into another interface clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces.

## Problem
Need to use an existing class with an incompatible interface.

## Solution
Create an adapter class that translates one interface to another.

## Python Implementation

```python
# Legacy/Third-party class with incompatible interface
class OldPaymentSystem:
    def make_payment(self, account_num: str, amount: float):
        print(f"Old system: ${amount} from account {account_num}")

# Target interface expected by client
class PaymentProcessor:
    def process(self, user_id: int, amount: float):
        pass

# Adapter
class PaymentAdapter(PaymentProcessor):
    def __init__(self, old_system: OldPaymentSystem):
        self.old_system = old_system

    def process(self, user_id: int, amount: float):
        account_num = f"ACC{user_id:06d}"
        self.old_system.make_payment(account_num, amount)

# Usage
old_sys = OldPaymentSystem()
adapter = PaymentAdapter(old_sys)
adapter.process(12345, 99.99)  # Translates interface
```

## When to Use
- ✅ Use existing class with incompatible interface
- ✅ Create reusable class with unrelated classes
- ✅ Integrate third-party libraries

## Summary
Adapter translates one interface to another, enabling compatibility without modifying existing code.
