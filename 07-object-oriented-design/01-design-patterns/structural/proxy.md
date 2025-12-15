# Proxy Pattern

## Intent
Provide a surrogate or placeholder for another object to control access to it.

## Problem
Need to control access to an object (lazy loading, access control, logging, caching).

## Solution
Create proxy class with same interface that controls access to real object.

## Types
1. **Virtual Proxy:** Lazy initialization
2. **Protection Proxy:** Access control
3. **Remote Proxy:** Represents remote object
4. **Cache Proxy:** Caches results

## Python Implementation

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
        print(f"Loading {self.filename} from disk...")  # Expensive

    def display(self) -> str:
        return f"Displaying {self.filename}"

# Virtual Proxy - lazy loading
class ImageProxy(Image):
    def __init__(self, filename: str):
        self.filename = filename
        self._real_image = None

    def display(self) -> str:
        if self._real_image is None:
            self._real_image = RealImage(self.filename)  # Load on first access
        return self._real_image.display()

# Usage
print("Creating proxy...")
image = ImageProxy("photo.jpg")  # No loading yet
print("Displaying first time:")
print(image.display())  # Loads now
print("Displaying second time:")
print(image.display())  # No loading
```

### Protection Proxy Example

```python
class BankAccount:
    def __init__(self, balance: float):
        self.balance = balance

    def withdraw(self, amount: float):
        self.balance -= amount
        return f"Withdrew ${amount}"

class AccountProxy:
    def __init__(self, account: BankAccount, user_role: str):
        self.account = account
        self.user_role = user_role

    def withdraw(self, amount: float):
        if self.user_role != "admin":
            return "Access denied"
        return self.account.withdraw(amount)

# Usage
account = BankAccount(1000)
user_proxy = AccountProxy(account, "user")
admin_proxy = AccountProxy(account, "admin")

print(user_proxy.withdraw(100))   # Access denied
print(admin_proxy.withdraw(100))  # Withdrew $100
```

## When to Use
- ✅ Lazy initialization (virtual proxy)
- ✅ Access control (protection proxy)
- ✅ Logging/caching
- ✅ Remote object access

## Summary
Proxy controls access to objects, useful for lazy loading, access control, and adding functionality without changing the object.
