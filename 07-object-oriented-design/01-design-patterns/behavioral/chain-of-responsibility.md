# Chain of Responsibility Pattern

## Intent
Avoid coupling sender of request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects and pass the request along the chain until an object handles it.

## Problem
Multiple objects might handle a request, but handler is not known beforehand.

## Solution
Create chain of handler objects, each deciding whether to process or pass along.

## Python Implementation

```python
from abc import ABC, abstractmethod

class Handler(ABC):
    def __init__(self):
        self._next_handler = None

    def set_next(self, handler: 'Handler') -> 'Handler':
        self._next_handler = handler
        return handler

    @abstractmethod
    def handle(self, request: dict) -> str:
        pass

class AuthenticationHandler(Handler):
    def handle(self, request: dict) -> str:
        if not request.get("authenticated"):
            return "Authentication failed"

        print("Authentication passed")
        if self._next_handler:
            return self._next_handler.handle(request)
        return "Request processed"

class AuthorizationHandler(Handler):
    def handle(self, request: dict) -> str:
        if request.get("role") != "admin":
            return "Authorization failed: insufficient permissions"

        print("Authorization passed")
        if self._next_handler:
            return self._next_handler.handle(request)
        return "Request processed"

class ValidationHandler(Handler):
    def handle(self, request: dict) -> str:
        if not request.get("data"):
            return "Validation failed: missing data"

        print("Validation passed")
        if self._next_handler:
            return self._next_handler.handle(request)
        return "Request processed"

# Build chain
auth = AuthenticationHandler()
authz = AuthorizationHandler()
valid = ValidationHandler()

auth.set_next(authz).set_next(valid)

# Usage
request1 = {"authenticated": True, "role": "admin", "data": "test"}
print(auth.handle(request1))  # Passes all checks

print()

request2 = {"authenticated": True, "role": "user", "data": "test"}
print(auth.handle(request2))  # Fails at authorization
```

## When to Use
- ✅ Multiple objects might handle request
- ✅ Handler not known beforehand
- ✅ Decouple sender and receiver

## Summary
Chain of Responsibility passes request along chain of handlers until one handles it. Common in middleware, logging, event handling.
