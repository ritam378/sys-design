# Mediator Pattern

## Intent
Define an object that encapsulates how a set of objects interact. Mediator promotes loose coupling by keeping objects from referring to each other explicitly.

## Problem
Complex communication between many objects creates tight coupling.

## Solution
Centralize complex communications in a mediator object.

## Python Implementation

```python
from abc import ABC, abstractmethod

class ChatMediator(ABC):
    @abstractmethod
    def send_message(self, message: str, user: 'User'):
        pass

    @abstractmethod
    def add_user(self, user: 'User'):
        pass

class ChatRoom(ChatMediator):
    def __init__(self):
        self._users = []

    def add_user(self, user: 'User'):
        self._users.append(user)

    def send_message(self, message: str, sender: 'User'):
        for user in self._users:
            if user != sender:  # Don't send to sender
                user.receive(message, sender.name)

class User:
    def __init__(self, name: str, mediator: ChatMediator):
        self.name = name
        self.mediator = mediator

    def send(self, message: str):
        print(f"{self.name} sends: {message}")
        self.mediator.send_message(message, self)

    def receive(self, message: str, sender: str):
        print(f"{self.name} receives from {sender}: {message}")

# Usage
chatroom = ChatRoom()

alice = User("Alice", chatroom)
bob = User("Bob", chatroom)
charlie = User("Charlie", chatroom)

chatroom.add_user(alice)
chatroom.add_user(bob)
chatroom.add_user(charlie)

alice.send("Hello everyone!")
# Bob receives: Hello everyone!
# Charlie receives: Hello everyone!
```

## When to Use
- ✅ Complex object interactions
- ✅ Reduce coupling between objects
- ✅ Centralize control logic

## Summary
Mediator centralizes complex communications, reducing coupling between objects. Common in chat systems, GUI components, air traffic control.
