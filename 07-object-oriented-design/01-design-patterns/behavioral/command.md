# Command Pattern

## Intent
Encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations.

## Problem
Need to parameterize objects with operations, queue operations, or support undo.

## Solution
Encapsulate requests as command objects with execute() and undo() methods.

## Python Implementation

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
        print(f"{self.location} light ON")

    def turn_off(self):
        self.is_on = False
        print(f"{self.location} light OFF")

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
        self.history = []

    def execute_command(self, command: Command):
        command.execute()
        self.history.append(command)

    def undo_last(self):
        if self.history:
            command = self.history.pop()
            command.undo()

# Usage
living_room = Light("Living Room")
remote = RemoteControl()

remote.execute_command(LightOnCommand(living_room))
remote.execute_command(LightOffCommand(living_room))
remote.undo_last()  # Undo turn off (turns on)
```

## When to Use
- ✅ Parameterize objects with operations
- ✅ Queue operations
- ✅ Support undo/redo
- ✅ Log changes

## Summary
Command encapsulates requests as objects, enabling queuing, logging, and undo functionality.
